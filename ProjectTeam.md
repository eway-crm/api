# Project Team API

API for managing a project's team (members, their roles and the project manager) from the
web frontend. Exposed by the WCF service on `API.svc` alongside all other endpoints.

> **Availability:** These endpoints are available only from eWay-CRM **10.1**. Earlier
> versions of the API do not expose them.

> A team is **not** a standalone entity. It is the set of `TEAM` relations between a project
> and its users, plus `TeamRoles` records that assign roles (groups with `IsRole = 1`) to a
> member. A member can hold **several roles at once**. The project manager is the member holding
> the PM role (`IsPM = 1`); they also have a `SUPERVISOR` relation. The frontend does not need to
> know this — it only works with the DTOs below.

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

Membership and roles are managed by **separate calls**: the membership calls put users on the team
or take them off it, the role calls only add or take away roles.

| Method | Parameters | Returns | Purpose |
|--------|-----------|---------|---------|
| `GetProjectTeamMembers` | `sessionId`, `projectGuid` | `DataResponse<TeamMember>` | List the project's team members with their roles. |
| `GetTeamRoles` | `sessionId` | `DataResponse<TeamRoleInfo>` | List all assignable roles. |
| `AddProjectTeamMembers` | `sessionId`, `projectGuid`, `userGuids: Guid[]` | `ResponseBase` | Put one or more users on the team **without any role**. |
| `RemoveProjectTeamMembers` | `sessionId`, `projectGuid`, `userGuids: Guid[]` | `ResponseBase` | Remove one or more users from the team. |
| `AddProjectTeamMemberRoles` | `sessionId`, `projectGuid`, `members: ProjectTeamMemberAssignment[]` | `ResponseBase` | Give members the listed roles **on top of** the ones they already hold (and put them on the team if they are not there yet). |
| `RemoveProjectTeamMemberRoles` | `sessionId`, `projectGuid`, `members: ProjectTeamMemberAssignment[]` | `ResponseBase` | Take the listed roles away, keeping membership and the remaining roles. |
| `SetProjectManager` | `sessionId`, `projectGuid`, `userGuid` | `ResponseBase` | Set the project manager (dedicated action). |

## DTOs

**`TeamMember`** (read)
| Field | Type | Notes |
|-------|------|-------|
| `UserGuid` | `Guid` | The member user. |
| `FileAs` | `string` | Display name of the user. |
| `Roles` | `TeamRoleInfo[]` | All roles the member holds. **Empty array** when the member has no role. |
| `IsProjectManager` | `bool` | True when one of the member's roles is the PM role. Read-only (set via `SetProjectManager`). |

**`TeamRoleInfo`** (read)
| Field | Type | Notes |
|-------|------|-------|
| `RoleGuid` | `Guid` | The role (group) id — use it in `RoleGuids` when adding or removing roles. |
| `RoleName` | `string` | Role name. |
| `IsPM` | `bool` | True for the project manager role. Cannot be used in the role calls — see below. |

**`ProjectTeamMemberAssignment`** (request, for the two role calls)
| Field | Type | Notes |
|-------|------|-------|
| `UserGuid` | `Guid` | The user. |
| `RoleGuids` | `Guid[]` | The roles to add / take away. **Required and non-empty** in both role calls. |

## Behavior to know

- **A member can hold several roles at once.** `GetProjectTeamMembers` returns all of them in
  `Roles`, and one request can grant several roles to the same user.
- **The role calls are a delta, not a replace.** `AddProjectTeamMemberRoles` only adds the roles you
  list and `RemoveProjectTeamMemberRoles` only takes away the roles you list — the member's other
  roles are left alone. The caller therefore does **not** have to know, or send, the roles a member
  already holds. Granting a role a member already has, or removing one they do not have, is a no-op.
- **Membership and roles are separate calls.** Use `AddProjectTeamMembers` to put a user on the team
  with no role; use `AddProjectTeamMemberRoles` when you want to grant roles (it puts the user on the
  team too, so there is no need to call both). Taking the last role away with
  `RemoveProjectTeamMemberRoles` **keeps the user on the team** — use `RemoveProjectTeamMembers` to
  take them off it.
- **`RoleGuids` may not be empty.** Both role calls reject a member entry with an empty or missing
  `RoleGuids` — such a request would do nothing, and putting a user on the team without a role is what
  `AddProjectTeamMembers` is for.
- **The PM role is rejected by the role calls.** Passing the role with `IsPM = true` to
  `AddProjectTeamMemberRoles` or `RemoveProjectTeamMemberRoles` is an error, because granting or
  taking it away also has to demote the previous project manager and move the `SUPERVISOR` relation.
  Use `SetProjectManager` instead.
- **Role calls are validated up front.** An unknown role id or the PM role rejects the **whole
  request before anything is written** — a rejected request never touches the team.
- **Batches are idempotent, but not all-or-nothing.** All four batch calls process items one by one
  and are safe to re-send: adding a user who is already a member (or listed twice in the same request)
  silently skips them, and removing a user who is not on the team is a no-op. Past the up-front
  validation the items are **not** wrapped in one transaction, so if an item fails, the items before it
  stay saved and `ReturnCode` is an error — just re-send the full desired list to recover.
- **Setting the project manager is a separate action.** `SetProjectManager` is the only call that
  creates the `SUPERVISOR` relation; the generic role calls never do. To read who the PM is, use
  `IsProjectManager` from `GetProjectTeamMembers`.
- **A project has exactly one project manager, and switching is automatic.** Calling
  `SetProjectManager` for a new user **demotes the previous project manager** (they stay a team member
  but lose the PM role, and the supervisor relation moves to the new PM). The frontend does not need to
  demote the old PM first — just call `SetProjectManager` with the new user. Setting the same user
  again is a no-op. Unlike the batch calls, this one runs in a **transaction**, so a mid-way failure
  cannot leave the team with a broken project manager.
- **Becoming the project manager keeps the member's other roles.** `SetProjectManager` adds the PM
  role, it does not replace what the member already holds.
- **Removing a member** cleans everything up (the `TEAM` relation, all their role records and the
  supervisor link if present).

## Examples

Put two users on the team without any role:

```
POST /API.svc/AddProjectTeamMembers
{
  "sessionId": "00000000-0000-0000-0000-000000000000",
  "projectGuid": "11111111-1111-1111-1111-111111111111",
  "userGuids": [
    "22222222-2222-2222-2222-222222222222",
    "44444444-4444-4444-4444-444444444444"
  ]
}
```

Give one member two roles at once (this also puts them on the team if they are not there yet):

```
POST /API.svc/AddProjectTeamMemberRoles
{
  "sessionId": "...",
  "projectGuid": "11111111-1111-1111-1111-111111111111",
  "members": [
    {
      "UserGuid": "22222222-2222-2222-2222-222222222222",
      "RoleGuids": [
        "33333333-3333-3333-3333-333333333333",
        "55555555-5555-5555-5555-555555555555"
      ]
    }
  ]
}
```

Take one role away — the member keeps their membership and their other roles:

```
POST /API.svc/RemoveProjectTeamMemberRoles
{
  "sessionId": "...",
  "projectGuid": "1111...",
  "members": [
    { "UserGuid": "2222...", "RoleGuids": ["55555555-5555-5555-5555-555555555555"] }
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
    {
      "UserGuid": "2222...",
      "FileAs": "Jane Doe",
      "Roles": [
        { "RoleGuid": "3333...", "RoleName": "Developer", "IsPM": false }
      ],
      "IsProjectManager": false
    },
    {
      "UserGuid": "4444...",
      "FileAs": "John Roe",
      "Roles": [],
      "IsProjectManager": false
    }
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
same way as for the other API endpoints. Typical causes:

- missing or zero `projectGuid`, `userGuid`, or a zero guid inside `userGuids` / `RoleGuids`,
- an empty `userGuids` / `members` array, or a member entry with empty `RoleGuids`,
- a `RoleGuids` entry that is not an existing role,
- the PM role passed to `AddProjectTeamMemberRoles` or `RemoveProjectTeamMemberRoles`.

Adding an already-member user, granting a role a member already holds, removing a non-member and
removing a role a member does not hold are **not** errors (all are silently ignored).
