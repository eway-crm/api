# Project Team API

API for managing a project's team (members, their roles and the project manager) from the
web frontend. Exposed by the WCF service on `API.svc` alongside all other endpoints.

> **Availability:** These endpoints are available only from eWay-CRM **10.1**. Earlier
> versions of the API do not expose them.

> A team is **not** a standalone entity. It is the set of `TEAM` relations between a project
> and its users, plus `TeamRoles` records that assign a role (a group with `IsRole = 1`) to a
> member. The project manager is the member whose role is the PM group (`IsPM = 1`); it also
> has a `SUPERVISOR` relation. The frontend does not need to know this — it only works with the
> DTOs below.

## Calling convention

Same as every other `API.svc` method:

- `POST /API.svc/<MethodName>`, `Content-Type: application/json`.
- Body is the **wrapped request**: a JSON object whose properties are the method parameters
  (e.g. `sessionId`, `projectGuid`, `members`).
- `sessionId` comes from `LogIn` (standard session handling, identical to the rest of the API).
- Every method returns the **standard response envelope** (the same one the FE already handles
  for other endpoints): `ReturnCode` (`"rcSuccess"` on success, otherwise an error code),
  `Description` (error message when not successful), and for read operations `Data` (an array).

## Endpoints

| Method | Parameters | Returns | Purpose |
|--------|-----------|---------|---------|
| `GetProjectTeamMembers` | `sessionId`, `projectGuid` | `DataResponse<TeamMember>` | List the project's team members with their roles. |
| `GetTeamRoles` | `sessionId` | `DataResponse<TeamRoleInfo>` | List all assignable roles. |
| `AddProjectTeamMembers` | `sessionId`, `projectGuid`, `members: ProjectTeamAssignment[]` | `ResponseBase` | Add one or more users (each with an optional role). |
| `RemoveProjectTeamMembers` | `sessionId`, `projectGuid`, `userGuids: Guid[]` | `ResponseBase` | Remove one or more users from the team. |
| `ChangeProjectTeamMemberRoles` | `sessionId`, `projectGuid`, `members: ProjectTeamAssignment[]` | `ResponseBase` | Change the role of one or more existing members. |
| `SetProjectManager` | `sessionId`, `projectGuid`, `userGuid` | `ResponseBase` | Set the project manager (dedicated action). |

## DTOs

**`TeamMember`** (read)
| Field | Type | Notes |
|-------|------|-------|
| `UserGuid` | `Guid` | The member user. |
| `UserName` | `string` | Display name (FileAs). |
| `RoleGuid` | `Guid?` | Assigned role, or `null` when the member has no role. |
| `RoleName` | `string` | Role name, or `null`. |
| `IsProjectManager` | `bool` | True for the project manager. Read-only (set via `SetProjectManager`). |

**`TeamRoleInfo`** (read)
| Field | Type | Notes |
|-------|------|-------|
| `RoleGuid` | `Guid` | The role (group) id — use as `RoleGuid` when adding/changing. |
| `RoleName` | `string` | Role name. |
| `IsPM` | `bool` | True for the project manager role. |

**`ProjectTeamAssignment`** (request, for add / change role)
| Field | Type | Notes |
|-------|------|-------|
| `UserGuid` | `Guid` | The user. |
| `RoleGuid` | `Guid?` | Role to assign. **Optional** for add; **required** for change role. |

## Behavior to know

- **Batches are idempotent, not all-or-nothing.** `AddProjectTeamMembers` and
  `RemoveProjectTeamMembers` process items one by one and are safe to re-send: adding a user who is
  already a member (or listed twice in the same request) silently skips them, and removing a user who
  is not on the team is a no-op. Items are **not** wrapped in one transaction, so if an item does fail
  the items before it stay saved and `ReturnCode` is an error — just re-send the full desired list to
  recover. `ChangeProjectTeamMemberRoles` is the exception: it still **fails on a non-member** (see
  below), so send it only for users you know are on the team.
- **Role is optional when adding.** `RoleGuid = null` adds a plain member without a role. (Roles
  are a licensed feature; on editions without it, members exist without roles.) Changing a role
  requires a non-null `RoleGuid`.
- **Setting the project manager is a separate action.** `SetProjectManager` is the only call that
  creates the `SUPERVISOR` relation; the generic add / change-role calls never do. To read who the PM
  is, use `IsProjectManager` from `GetProjectTeamMembers`.
- **A project has exactly one project manager, and switching is automatic.** Calling
  `SetProjectManager` for a new user **demotes the previous project manager** (they stay a team member
  but lose the PM role, and the supervisor relation moves to the new PM). The frontend does not need to
  demote the old PM first — just call `SetProjectManager` with the new user. Setting the same user
  again is a no-op.
- **Removing a member** cleans everything up (TEAM relation, role record and the supervisor link if present).
- **Adding an existing member is skipped** (silently ignored, no error). Adding does not change the
  role of someone already on the team — use `ChangeProjectTeamMemberRoles` for that; setting the same
  role again is a no-op.

## Examples

Add two members (one with a role, one without):

```
POST /API.svc/AddProjectTeamMembers
{
  "sessionId": "00000000-0000-0000-0000-000000000000",
  "projectGuid": "11111111-1111-1111-1111-111111111111",
  "members": [
    { "UserGuid": "22222222-2222-2222-2222-222222222222", "RoleGuid": "33333333-3333-3333-3333-333333333333" },
    { "UserGuid": "44444444-4444-4444-4444-444444444444", "RoleGuid": null }
  ]
}
```

List the team:

```
POST /API.svc/GetProjectTeamMembers
{ "sessionId": "...", "projectGuid": "11111111-1111-1111-1111-111111111111" }

→ {
  "ReturnCode": "rcSuccess",
  "Data": [
    { "UserGuid": "2222...", "UserName": "Jane Doe", "RoleGuid": "3333...", "RoleName": "Developer", "IsProjectManager": false },
    { "UserGuid": "4444...", "UserName": "John Roe", "RoleGuid": null, "RoleName": null, "IsProjectManager": false }
  ]
}
```

Remove members:

```
POST /API.svc/RemoveProjectTeamMembers
{ "sessionId": "...", "projectGuid": "1111...", "userGuids": ["2222...", "4444..."] }
```

## Error handling

On failure, `ReturnCode` is not `"rcSuccess"` and `Description` carries the message — handle it the
same way as for the other API endpoints. Typical causes: missing/zero `projectGuid` or `userGuid`,
empty arrays, or changing the role of a non-member. Adding an already-member user or removing a
non-member is **not** an error (both are silently ignored).