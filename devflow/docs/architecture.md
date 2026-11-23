# DevFlow --- System Architecture

**Document:** `docs/architecture.md`\
**Version:** 1.0\
**Status:** Architecture draft for review\
**Source baseline:** *DevFlow Product Requirements, Technical Blueprint
& Execution Plan*, Version 1.0, Planning --- Phase 1

------------------------------------------------------------------------

## 1. Architecture decision

DevFlow will use a **modular monolith**: one deployable Django REST
Framework backend, organized into clearly separated domain modules, with
a React + TypeScript frontend and a shared PostgreSQL database.

This keeps deployment and operational complexity manageable while
preserving boundaries between product domains. Services should not be
split into microservices merely to make the architecture appear more
advanced. A future extraction should be considered only when independent
scaling or team ownership makes it necessary.

## 2. High-level architecture

``` mermaid
flowchart TB
    U[Users: Owner / Admin / Project Manager / Developer / Viewer]

    subgraph CLIENT["Frontend"]
        UI["React + TypeScript + Vite"]
        FE["Pages • Components • Router"]
        Q["TanStack Query • Forms • Client validation"]
        UI --- FE
        UI --- Q
    end

    API["HTTPS / JSON REST API<br/>Versioned endpoints • validation • consistent errors"]

    subgraph BACKEND["Django REST Framework — Modular Monolith"]
        AUTH["Accounts & Authentication"]
        ORG["Organizations & Memberships"]
        PROJ["Projects"]
        TASK["Tasks & Bugs"]
        SPRINT["Sprints"]
        COLLAB["Comments • Attachments"]
        SUPPORT["Notifications • Audit Logs • Analytics"]
        COMMON["Common: permissions • pagination • exceptions • middleware • utilities"]
        SERVICES["Domain Services<br/>Tenant context • RBAC • business rules • transactions • scoped queries"]
        AUTH --> SERVICES
        ORG --> SERVICES
        PROJ --> SERVICES
        TASK --> SERVICES
        SPRINT --> SERVICES
        COLLAB --> SERVICES
        SUPPORT --> SERVICES
        COMMON --> SERVICES
    end

    DB[("PostgreSQL<br/>Persistent relational data")]
    REDIS[("Redis<br/>Cache + Celery message transport")]
    WORKER["Celery Workers<br/>Email • notifications • scheduled jobs"]

    U --> UI
    UI -->|"HTTPS requests"| API
    API --> AUTH
    API --> ORG
    API --> PROJ
    API --> TASK
    API --> SPRINT
    API --> COLLAB
    API --> SUPPORT
    SERVICES <-->|"Read / write • transactions"| DB
    SERVICES <-->|"Cache read / invalidation"| REDIS
    SERVICES -->|"Queue background work"| REDIS
    REDIS -->|"Deliver queued jobs"| WORKER
    WORKER -->|"Read / update application data as needed"| DB

    INFRA["Docker Compose • GitHub Actions • Hosting<br/>Build • test • deploy • HTTPS • logs • backups"]
    INFRA -. "supports deployment and operations" .-> UI
    INFRA -.-> BACKEND
    INFRA -.-> WORKER
```

### Component responsibilities

  -----------------------------------------------------------------------
  Component                           Responsibility
  ----------------------------------- -----------------------------------
  React + TypeScript + Vite           User interface, navigation, forms
                                      and client-side interaction

  TanStack Query                      Frontend server-state caching,
                                      retries and invalidation

  Django REST Framework               Versioned REST API, request
                                      handling, serializers and API
                                      responses

  Domain services and permissions     Business rules, tenant context,
                                      role checks, transactions and
                                      organization-scoped queries

  PostgreSQL                          Permanent relational storage,
                                      constraints, indexes and
                                      transactions

  Redis                               Cache and message transport for
                                      Celery

  Celery                              Executes background email,
                                      notification and scheduled jobs

  Docker Compose                      Repeatable local services and
                                      containerized deployment

  GitHub Actions                      Automated lint, test and build
                                      checks in CI

  OpenAPI / Swagger                   API contract and integration
                                      reference
  -----------------------------------------------------------------------

## 3. Request lifecycle

A normal authenticated API request follows this sequence:

1.  The React interface sends an HTTPS request to a versioned API
    endpoint.
2.  Authentication identifies the requesting user.
3.  The backend resolves the active organization through a **validated
    membership**. It must not blindly trust an `organization_id`
    supplied in the request body.
4.  Permission checks authorize the action against the organization and
    requested resource.
5.  Serializer and domain/service logic validate the input and business
    operation.
6.  PostgreSQL reads or writes the required data. A database transaction
    protects operations that must succeed or fail together.
7.  Cache data is read or invalidated where appropriate.
8.  Non-blocking background work, such as email or notifications, is
    queued when required.
9.  The API returns a consistent status code and response body.
10. The frontend updates server state and displays success, loading or
    error feedback.

``` mermaid
sequenceDiagram
    actor User
    participant UI as React Frontend
    participant API as Django REST API
    participant Auth as Authentication
    participant Guard as Tenant & Permission Checks
    participant Service as Domain Service
    participant DB as PostgreSQL
    participant Redis
    participant Worker as Celery Worker

    User->>UI: Perform an action
    UI->>API: HTTPS request (JSON)
    API->>Auth: Identify user
    Auth-->>API: Authenticated user
    API->>Guard: Resolve organization membership + authorize
    Guard-->>API: Access decision
    API->>Service: Validate and execute business operation
    Service->>DB: Read / write in transaction
    DB-->>Service: Persisted result
    opt Non-blocking background work required
        Service->>Redis: Enqueue job
        Redis-->>Worker: Deliver job
        Worker->>Worker: Execute email / notification task
    end
    Service-->>API: Operation result
    API-->>UI: Consistent response
    UI-->>User: Updated interface / feedback
```

## 4. Backend domain modules

The backend remains one Django deployment, but code is grouped by
responsibility.

``` text
backend/
├── config/
│   ├── settings/
│   │   ├── base.py
│   │   ├── development.py
│   │   └── production.py
│   ├── urls.py
│   ├── asgi.py
│   ├── wsgi.py
│   └── celery.py
├── apps/
│   ├── accounts/
│   ├── organizations/
│   ├── memberships/
│   ├── invitations/
│   ├── projects/
│   ├── tasks/
│   ├── sprints/
│   ├── comments/
│   ├── attachments/
│   ├── notifications/
│   ├── audit_logs/
│   └── analytics/
├── common/
│   ├── permissions/
│   ├── pagination/
│   ├── exceptions/
│   ├── middleware/
│   └── utilities/
├── tests/
└── manage.py
```

The frontend is organized by shared application structure and product
features:

``` text
frontend/
├── public/
└── src/
    ├── app/
    ├── components/
    ├── features/
    ├── hooks/
    ├── lib/
    ├── routes/
    ├── services/
    ├── types/
    └── styles/
```

## 5. Tenant isolation and authorization boundary

DevFlow is a multi-tenant application. Organization data must be
isolated at the backend, not merely hidden in the frontend.

-   Resolve organization context from the authenticated user's valid
    membership.
-   Scope organization-owned querysets to the active organization.
-   Check authorization on every protected endpoint and resource action.
-   Validate that related objects, such as project, task and assignee,
    belong to the same organization.
-   Do not trust client-supplied organization identifiers as proof of
    access.
-   Test attempts to access another organization's records by changing
    IDs, payloads or nested URLs.
-   Project-level permissions may narrow organization-level permissions,
    but must not widen them.

## 6. Data and background-work responsibilities

### PostgreSQL

PostgreSQL is the system of record for persistent application data such
as users, organizations, memberships, projects and tasks. It provides
relational constraints, indexes and transactions.

### Redis

Redis is used for caching and as the message transport used by Celery.
It is not the permanent source of truth for project or task records.

### Celery

Celery executes work outside the normal request/response path, such as
email, notifications and scheduled jobs. Redis transports queued
messages; Celery workers execute the jobs.

**Failure-handling principle:** If a task has been successfully saved in
PostgreSQL but a non-critical notification worker is temporarily
unavailable, the saved task should remain intact. The notification
should be retried or recovered through a reliable retry mechanism. A
background notification failure must not be treated as a reason to
delete valid core application data.

> Implementation note: when queueing a background job depends on a
> database write, design the ordering and retry behavior carefully so a
> committed database change is not silently separated from its required
> job. The exact reliability mechanism will be finalized during
> implementation.

## 7. Technology stack

  -----------------------------------------------------------------------
  Layer                               Selected technology
  ----------------------------------- -----------------------------------
  Frontend                            React + TypeScript + Vite

  Styling                             Tailwind CSS

  Routing                             React Router

  Server state                        TanStack Query

  Forms and validation                React Hook Form + Zod

  Backend                             Python + Django + Django REST
                                      Framework

  Authentication                      Simple JWT or an equivalent
                                      maintained JWT package

  Database                            PostgreSQL

  Cache / message broker              Redis

  Background jobs                     Celery

  Testing                             Pytest / Django tests, Vitest,
                                      React Testing Library, Playwright

  Infrastructure                      Docker Compose

  CI/CD                               GitHub Actions

  API documentation                   OpenAPI / Swagger
  -----------------------------------------------------------------------

## 8. Architecture quality and security rules

1.  Keep domain boundaries clear inside the modular monolith.
2.  Keep authorization and tenant isolation on the server.
3.  Use database constraints and transactions to protect data integrity.
4.  Keep secrets outside source control and use environment-specific
    settings.
5.  Use HTTPS in deployed environments.
6.  Configure authentication, refresh-token rotation and revocation
    carefully.
7.  Validate inputs and related-object ownership on the backend.
8.  Paginate list endpoints and bound page sizes.
9.  Add automated tests for tenant isolation, permissions and critical
    workflows.
10. Use logs and backups as part of deployment and recovery planning.

## 9. Deployment and operations view

Docker Compose provides repeatable local services and supports
containerized deployment. GitHub Actions runs automated linting, tests
and build checks. The deployed application must use HTTPS and have
documented logging and backup practices.

The deployment topology and hosting provider are intentionally not fixed
in this architecture document; they should be selected and documented
when the deployment environment is configured.

## 10. Architecture decisions and items to confirm

  -----------------------------------------------------------------------
  Decision                            Current position
  ----------------------------------- -----------------------------------
  Backend architecture                Modular monolith

  API style                           Versioned JSON REST API

  Persistent database                 PostgreSQL

  Tenant model                        Shared schema with tenant-aware
                                      access control

  Cache and task transport            Redis

  Background job executor             Celery

  Frontend                            React + TypeScript + Vite

  Microservices                       Explicitly excluded from the first
                                      release

  Hosting topology                    To be finalized during deployment
                                      planning

  Background-job recovery mechanism   To be finalized during
                                      implementation
  -----------------------------------------------------------------------

------------------------------------------------------------------------

**Document status:** This is the architecture documentation draft based
on the DevFlow planning blueprint. It records the intended design; it
does not claim that these components have already been implemented,
deployed or tested.
