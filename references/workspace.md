# workspace.md — Workspace listing + members

## List workspaces

```
GET /api/v1/workspaces
```

Scope: `workspaces:read`.

For an API-key caller, the response is filtered to **only the workspace this key is bound to** — even if the underlying user is a member of multiple workspaces. This is the simplest way to discover the bound workspace UUID from a stored key:

```python
ws = requests.get(
    "https://dutify.ai/api/v1/workspaces",
    headers={"X-API-Key": key},
).json()
bound_uuid = ws[0]["id"]   # always exactly one entry for API-key callers
```

For JWT callers (interactive Hub UI), the response includes every workspace the user belongs to.

Each entry shape:

```json
{
  "id": "00000000-0000-0000-0000-000000000000",
  "name": "Acme Engineering",
  "description": "Engineering team's workspace",
  "isOwner": true,
  "createdAt": "2026-01-01T00:00:00Z",
  ...
}
```

## Create workspace — JWT only

```
POST /api/v1/workspaces
```

Scope: would be `workspaces:write` but **not exposed to API keys** — `workspaces:write` is intentionally absent from the scope catalog. Provisioning a new workspace is an interactive flow (user picks name, color, plan, etc.); a script shouldn't create workspaces on a user's behalf without explicit consent.

If you need to create a workspace programmatically, the user has to do it via Hub UI first, then provision an API key for the new workspace.

## Members

```
GET /api/v1/workspaces/{workspaceId}/members
```

Scope: `members:read`.

Returns the full member directory:

```json
{
  "members": [
    {
      "uuid": "<user-uuid>",
      "email": "alex@dutify.ai",
      "firstName": "Alex",
      "lastName": "Smith",
      "role": "OWNER",
      "joinedAt": "2026-01-01T00:00:00Z",
      ...
    },
    ...
  ]
}
```

Useful for:

- Resolving "who said X in this call?" — cross-reference attendee email from `GET /usercall/{id}` with member directory
- "Tag this person" flows in agent responses
- Showing the user a list of workspace teammates before they pick one to do something with

## Single member by user UUID

```
GET /api/v1/workspaces/{workspaceId}/members/{userId}
```

Scope: `members:read`. Returns one entry from the same shape as above. Useful when you have a UUID from somewhere (e.g. a call's attendee record) and need to look up the full profile.

## Member writes — JWT only

`DELETE /v1/workspaces/{id}/members/{userId}`, `POST /v1/workspaces/{id}/owner`, `POST /v1/workspaces/{id}/leave` are NOT exposed to API keys. They require `workspaces:write` (which the scope catalog doesn't grant) AND owner-only permission checks at the resource layer.

If a script needs to remove members, transfer ownership, or leave a workspace, the user must do it via Hub UI.

## Patterns

### Resolve email → user UUID via the directory

```python
def email_to_uuid(email: str) -> str | None:
    members = requests.get(
        f"https://dutify.ai/api/v1/workspaces/{ws}/members",
        headers={"X-API-Key": key},
    ).json()["members"]
    for m in members:
        if m["email"].lower() == email.lower():
            return m["uuid"]
    return None
```

### Discover bound workspace + member count in one go

```python
ws_list = requests.get(
    "https://dutify.ai/api/v1/workspaces",
    headers={"X-API-Key": key},
).json()
bound = ws_list[0]
print(f"Bound workspace: {bound['name']} ({bound['id']})")

members = requests.get(
    f"https://dutify.ai/api/v1/workspaces/{bound['id']}/members",
    headers={"X-API-Key": key},
).json()["members"]
print(f"Members: {len(members)}")
```

## Footguns

- **`/v1/workspaces` returning a single entry from an API key is by design**, not a bug. If you assumed it would list all the user's workspaces, you'll think the user only has one — they don't, the key does.
- **`isOwner` on the bound workspace tells you whether the api-key creator is an owner**, not whether the API key has owner-level permissions (it doesn't — API keys never have admin-equivalent rights regardless of who created them).
- **Member roles are workspace-scoped** — `OWNER` here is different from `OWNER` of another workspace. Don't carry roles across workspace boundaries.
