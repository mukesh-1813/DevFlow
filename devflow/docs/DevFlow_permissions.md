# DevFlow --- Role and Permission Matrix

**Document:** `docs/permissions.md`\
**Version:** 1.0\
**Status:** Baseline permission draft for review\
**Source baseline:** *DevFlow Product Requirements, Technical Blueprint
& Execution Plan*, Version 1.0, Planning --- Phase 1

------------------------------------------------------------------------

## 1. Purpose

This document records the baseline organization-role permissions for
DevFlow and the security rules that govern project-level access.

The matrix follows the permission baseline in the product blueprint.
Where the blueprint says `Limited` or `Policy-based`, the exact
permitted operations must be defined before implementation; this
document does not silently expand those permissions.

## 2. Roles

  -----------------------------------------------------------------------
  Role                                Responsibility
  ----------------------------------- -----------------------------------
  Organization Owner                  Owns and controls the organization

  Administrator                       Manages permitted organization
                                      operations and members, subject to
                                      owner-protection rules

  Project Manager                     Manages projects and coordinates
                                      project work

  Developer                           Works with tasks and permitted
                                      project data

  Viewer                              Reads permitted project data
  -----------------------------------------------------------------------

## 3. Baseline permission matrix

  -------------------------------------------------------------------------------------
  Action          Owner       Administrator   Project     Developer      Viewer
                                              Manager                    
  --------------- ----------- --------------- ----------- -------------- --------------
  Manage          Yes         Limited         No          No             No
  organization                                                           
  settings                                                               

  Invite/remove   Yes         Yes             No          No             No
  organization                                                           
  members                                                                

  Manage          Yes         Yes\*           No          No             No
  organization                                                           
  roles                                                                  

  Create/manage   Yes         Yes             Yes         Policy-based   No
  projects                                                               

  Create/update   Yes         Yes             Yes         Yes            No
  tasks                                                                  

  Comment on      Yes         Yes             Yes         Yes            Policy-based
  tasks                                                                  

  View permitted  Yes         Yes             Yes         Yes            Yes
  project data                                                           
  -------------------------------------------------------------------------------------

`*` An Administrator must not be able to promote themselves to Owner or
remove the last Owner.

**Permission labels** - **Yes:** The baseline explicitly allows the
action. - **No:** The baseline explicitly disallows the action. -
**Limited:** The role has restricted access; exact allowed settings must
be specified. - **Policy-based:** Access depends on additional
project-level policy and membership rules; it is not an unconditional
grant.

## 4. Owner-protection rules

The following rules are explicit in the blueprint:

1.  An Administrator cannot promote themselves to Owner.
2.  An Administrator cannot remove the organization's last Owner.
3.  These rules must be encoded in automated tests.
4.  Organization role changes and member removal must be performed only
    by authorized actors.

The blueprint does not fully define every Owner-transfer, multi-owner
removal or organization-deletion scenario. Those policies must be
specified separately before implementation.

## 5. Project-level permission boundary

Project-level permissions may **narrow** organization-level permissions,
but must never widen them.

A project policy can restrict an organization member's access within a
particular project. It cannot grant a user an action that the user's
organization role does not permit.

For actions marked `Policy-based`, define exact rules before
implementation. In particular: - Developer access to creating or
managing projects is not fully specified by the baseline. - Viewer
access to commenting is not fully specified by the baseline. - The exact
scope of `Limited` organization settings for Administrators is not fully
specified by the baseline.

Do not infer that a Developer can delete projects from the
`Policy-based` entry. Do not infer that every task update or status
transition is automatically permitted; task transitions and resource
permissions require backend validation.

## 6. Backend authorization requirements

-   Enforce role and resource permissions on every protected operation.
-   Authenticate the user before evaluating permissions.
-   Resolve the active organization through a validated membership.
-   Check project membership and project-level policy where applicable.
-   Validate that the requested resource belongs to the active
    organization.
-   Never rely on frontend button visibility as an authorization
    control.
-   Reject unauthorized requests consistently.
-   Record important actions in activity/audit history when that
    capability is implemented.

## 7. Security test scenarios

  -----------------------------------------------------------------------
  ID                      Scenario                Expected result
  ----------------------- ----------------------- -----------------------
  PERM-01                 Unauthenticated user    Reject
                          requests a protected    
                          operation               

  PERM-02                 User from Organization  Reject
                          B accesses Organization 
                          A's project             

  PERM-03                 Administrator attempts  Reject
                          to promote themselves   
                          to Owner                

  PERM-04                 Administrator attempts  Reject
                          to remove the last      
                          Owner                   

  PERM-05                 Viewer attempts to      Reject
                          create or update a task 

  PERM-06                 Developer attempts a    Reject
                          project-management      
                          action without an       
                          explicit grant          

  PERM-07                 Project-level policy    Reject
                          attempts to grant more  
                          than the organization   
                          role allows             

  PERM-08                 User attempts an action Reject
                          on a project they       
                          cannot access           

  PERM-09                 Authorized Developer    Allow
                          creates or updates a    
                          task within permitted   
                          scope                   

  PERM-10                 Viewer reads project    Allow
                          data they are permitted 
                          to view                 
  -----------------------------------------------------------------------

These are acceptance scenarios for the intended policy. Exact API
endpoints, response codes and test fixtures will be defined during
implementation.

## 8. Decisions required before implementation

Confirm the following policy details before treating this matrix as
final:

-   Which organization settings Administrators may change.
-   Whether Developers may create projects, and under what conditions.
-   Whether Developers may edit project metadata or archive projects.
-   Whether Viewers may comment, and under what conditions.
-   Which task fields and status transitions each role may change.
-   Whether and how ownership can be transferred.
-   Whether an organization may have multiple Owners and how Owner
    removal works beyond the last-Owner safeguard.

------------------------------------------------------------------------

**Document status:** Baseline permission draft derived from the DevFlow
planning blueprint. It is not an implementation claim. `Limited` and
`Policy-based` entries remain open policy decisions until explicitly
resolved.
