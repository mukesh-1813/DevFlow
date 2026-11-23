# DevFlow API Contract

**Version:** 1.0  
**Status:** Planning draft  
**Base path:** `/api/v1/`

> This document records the API design defined in the DevFlow Product Requirements, Technical Blueprint & Execution Plan. Request and response schemas will be refined during implementation and documented in OpenAPI.

## 1. API design principles

- Use a versioned REST API under `/api/v1/`.
- Keep endpoint names resource-oriented.
- Use standard HTTP methods and status codes.
- Validate all input on the server.
- Document request and response schemas in OpenAPI.
- Scope tenant-owned data to the authenticated user's validated organization membership.
- Never treat a client-supplied `organization_id` as proof of access.
- Validate that related objects belong to the same organization.
- Apply object-level permission checks to sensitive actions.

## 2. Initial endpoint inventory

| Method | Endpoint | Purpose |
|---|---|---|
| POST | `/api/v1/auth/register/` | Register user |
| POST | `/api/v1/auth/login/` | Obtain tokens |
| POST | `/api/v1/auth/refresh/` | Refresh access token |
| POST | `/api/v1/auth/logout/` | Revoke/blacklist refresh token if configured |
| GET, POST | `/api/v1/organizations/` | List/create organizations |
| GET, PATCH | `/api/v1/organizations/{org_id}/` | Read/update organization |
| GET, POST | `/api/v1/organizations/{org_id}/members/` | List/add or invite members |
| PATCH, DELETE | `/api/v1/organizations/{org_id}/members/{member_id}/` | Change role/remove member |
| GET, POST | `/api/v1/projects/` | List/create projects in authorized context |
| GET, PATCH, DELETE | `/api/v1/projects/{project_id}/` | Read/update/archive project |
| GET, POST | `/api/v1/tasks/` | List/create tasks |
| GET, PATCH, DELETE | `/api/v1/tasks/{task_id}/` | Read/update/archive task |
| POST | `/api/v1/tasks/{task_id}/comments/` | Create comment |
| GET, POST | `/api/v1/projects/{project_id}/sprints/` | List/create sprints |
| GET | `/api/v1/analytics/overview/` | Organization/project analytics |
| GET | `/api/v1/notifications/` | List current user's notifications |
| PATCH | `/api/v1/notifications/{id}/read/` | Mark notification read |

## 3. HTTP method conventions

| Method | Use |
|---|---|
| GET | Read resources |
| POST | Create resources or perform actions |
| PATCH | Partially update resources |
| DELETE | Delete or archive according to the resource's lifecycle rules |

## 4. Status-code conventions

| Status | Meaning |
|---|---|
| 200 | Successful read or update |
| 201 | Resource created |
| 204 | Successful operation with no response body |
| 400 | Validation error |
| 401 | Unauthenticated request |
| 403 | Authenticated request denied |
| 404 | Resource not found; denied resources may also use 404 according to the information-disclosure policy |
| 429 | Rate limit exceeded |

## 5. List endpoint conventions

- Use consistent pagination metadata.
- Enforce a maximum page size.
- Document supported filter names.
- Document allowed ordering fields.
- Document allowed status transitions.
- Keep list responses bounded; do not return an unbounded collection.

Illustrative query parameters:

```http
GET /api/v1/tasks/?status=in_progress&priority=high&ordering=-created_at
```

The supported filters and ordering fields must be explicitly defined for each endpoint during implementation.

## 6. Error response convention

Use stable machine-readable error codes with human-readable messages. Include field-level details when useful. Do not expose stack traces or internal implementation details.

Illustrative validation error:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Please correct the submitted data.",
    "details": {
      "title": ["This field is required."]
    }
  }
}
```

Illustrative resource error:

```json
{
  "error": {
    "code": "TASK_NOT_FOUND",
    "message": "The requested task does not exist."
  }
}
```

Error codes should remain stable when human-readable messages are edited or translated.

## 7. Illustrative task response

```json
{
  "id": "task_uuid",
  "key": "DEV-142",
  "title": "Add organization-scoped task listing",
  "type": "task",
  "status": "in_progress",
  "priority": "high",
  "project_id": "project_uuid",
  "assignee": {
    "id": "user_uuid",
    "display_name": "Alex"
  },
  "due_date": "2026-12-10T18:00:00Z",
  "created_at": "2026-11-04T10:00:00Z"
}
```

This is an illustrative response shape from the blueprint, not a complete serializer specification.

## 8. Retry-sensitive operations

Make retry-sensitive operations idempotent where practical. For example, a repeated task-creation attempt caused by a network timeout should not unintentionally create duplicate tasks. The exact mechanism and endpoints requiring it will be decided during implementation.

## 9. Contract workflow

1. Agree on endpoint purpose and permission rules.
2. Define serializer/request and response schemas.
3. Document filters, ordering, pagination and errors.
4. Implement the backend and tests.
5. Publish or update the OpenAPI schema.
6. Integrate the frontend against the documented contract.
7. Make contract changes through reviewed pull requests.

## 10. Implementation decisions still to make

The blueprint establishes the conventions above but does not fully specify every endpoint. Resolve these details while implementing:

- Exact request and response fields for each endpoint.
- Exact pagination format and maximum page size.
- Supported filters and ordering fields per list endpoint.
- Exact status-transition rules for tasks.
- Which operations require idempotency keys and how keys are stored.
- Whether denied resources return 403 or 404 in each relevant case.
- Archive/soft-delete behaviour for projects and tasks.

---
**Source:** DevFlow Product Requirements, Technical Blueprint & Execution Plan, API design and contract sections (pages 14–15).
