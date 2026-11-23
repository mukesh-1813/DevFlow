# DevFlow --- Entity Relationship Diagram (ERD)

**Document:** `docs/erd.md`\
**Version:** 1.0\
**Status:** Conceptual ERD draft for review\
**Source baseline:** *DevFlow Product Requirements, Technical Blueprint
& Execution Plan*, Version 1.0, Planning --- Phase 1

------------------------------------------------------------------------

## 1. Purpose

This document records the conceptual data model for DevFlow. It
identifies the core entities, their relationships, tenant boundaries and
integrity rules.

The source blueprint provides a conceptual relationship map and a set of
relationship and constraint rules. It does not define every column or
exact foreign-key implementation. Therefore, this document does not
invent a complete physical schema. Exact attributes, nullability,
deletion behavior and database-level enforcement must be confirmed
during implementation.

## 2. Core entities

  -----------------------------------------------------------------------
  Entity                  Purpose                 Scope
  ----------------------- ----------------------- -----------------------
  User                    Stores user accounts    Global identity

  Organization            Represents a            Tenant root
                          tenant/workspace        

  Membership              Connects a user to an   Organization-scoped
                          organization            relationship

  Invitation              Tracks invitations to   Organization-scoped
                          join an organization    

  Project                 Stores an               Organization-owned
                          organization's projects 

  ProjectMember           Connects users to       Project relationship
                          projects                

  Sprint                  Organizes project work  Project-owned
                          into sprints            

  Task                    Stores tasks and bugs   Project-owned; may
                                                  belong to a sprint

  Label                   Stores labels used to   Organization/project
                          categorize tasks        scope to be confirmed

  TaskLabel               Connects tasks and      Many-to-many join
                          labels                  entity

  Comment                 Stores discussion on a  Task-owned
                          task                    

  Attachment              Stores task-related     Task-owned
                          attachment metadata     

  Notification            Stores notifications    User and organization
                          for users               scoped

  ActivityLog             Records organization    Organization-owned
                          activity                
  -----------------------------------------------------------------------

## 3. Conceptual ERD

``` mermaid
erDiagram
    USER ||--o{ MEMBERSHIP : has
    ORGANIZATION ||--o{ MEMBERSHIP : includes

    ORGANIZATION ||--o{ INVITATION : issues
    ORGANIZATION ||--o{ PROJECT : owns

    PROJECT ||--o{ PROJECT_MEMBER : has
    USER ||--o{ PROJECT_MEMBER : joins

    PROJECT ||--o{ SPRINT : contains
    PROJECT ||--o{ TASK : contains
    SPRINT o|--o{ TASK : groups

    TASK ||--o{ COMMENT : has
    TASK ||--o{ ATTACHMENT : has

    TASK ||--o{ TASK_LABEL : tagged_by
    LABEL ||--o{ TASK_LABEL : applied_to

    USER ||--o{ NOTIFICATION : receives
    ORGANIZATION ||--o{ NOTIFICATION : scopes

    ORGANIZATION ||--o{ ACTIVITY_LOG : records
```

**Notation:** `||` means exactly one; `o{` means zero or many; `o|`
means zero or one. The diagram is conceptual and should be checked
against the implemented foreign keys and constraints.

## 4. Relationship rules

1.  User ↔ Organization is many-to-many through Membership.
2.  Organization → Projects is one-to-many.
3.  Project ↔ Users is many-to-many through ProjectMember.
4.  Project → Tasks and Project → Sprints are one-to-many.
5.  Sprint → Tasks is one-to-many; a task may have no sprint when it is
    in the backlog.
6.  Task → Comments and Task → Attachments are one-to-many.
7.  Task ↔ Labels is many-to-many through TaskLabel.
8.  User → Notifications is one-to-many.
9.  Organization → ActivityLogs is one-to-many.

## 5. Integrity constraints and indexes

The blueprint specifies these constraints and indexing rules:

  -----------------------------------------------------------------------------------------
  Area                                Rule
  ----------------------------------- -----------------------------------------------------
  Organization                        Organization slug must be unique

  Membership                          Unique membership for each
                                      `(organization_id, user_id)` pair

  Project                             Project key must be unique within an organization

  Invitation                          Invitation token hash must be unique

  Invitation lookup                   Index expiry and pending-invitation lookup fields

  Task listing                        Consider an index on
                                      `(organization_id, project_id, status, created_at)`
                                      where query patterns justify it

  Task filtering                      Consider indexes for assignee, sprint, priority and
                                      due date after inspecting real queries

  Data invariants                     Use database constraints for rules that must hold
                                      even if application code has a defect

  Cross-tenant references             Ensure project, sprint, assignee and label references
                                      belong to the same organization through service
                                      validation and, where feasible, database constraints
  -----------------------------------------------------------------------------------------

Do not add indexes without a query reason. Indexes improve some reads
but add storage and write overhead.

## 6. Multi-tenant data integrity

Organization is the tenant root. Tenant-owned data must not leak across
organizations.

-   Resolve the active organization from the authenticated user's
    validated membership.
-   Scope organization-owned queries to the active organization.
-   Do not treat a client-supplied `organization_id` as proof of access.
-   Validate that referenced projects, sprints, assignees and labels
    belong to the same organization.
-   Test attempts to read or modify another organization's records by
    changing IDs, request payloads or nested resource URLs.

## 7. Decisions to finalize during physical schema design

The conceptual blueprint does not fully specify the following. Confirm
them before treating the physical schema as final:

-   Exact fields and data types for every entity.
-   Whether Label is organization-wide or project-specific.
-   Exact foreign-key placement for tenant ownership where it is not
    explicit in the conceptual diagram.
-   Nullability and deletion behavior for each relationship.
-   Whether database-level composite constraints are feasible for
    cross-tenant references.
-   Required indexes based on actual query patterns.

## 8. Review checklist

-   [ ] All 14 core entities are represented.
-   [ ] Membership models User--Organization many-to-many.
-   [ ] ProjectMember models Project--User many-to-many.
-   [ ] Task can be in a project backlog without a sprint.
-   [ ] Task--Label many-to-many uses TaskLabel.
-   [ ] Organization boundaries are explicit.
-   [ ] Uniqueness constraints are documented.
-   [ ] Indexes are justified by lookup/query patterns.
-   [ ] Exact columns and foreign keys are reviewed before migrations.

------------------------------------------------------------------------

**Document status:** Conceptual ERD draft based on the planning
blueprint. It is not yet a final physical database schema and does not
claim that migrations or tables have been implemented.
