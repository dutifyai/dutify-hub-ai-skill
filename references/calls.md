# calls.md — UserCalls (the call/meeting metadata layer)

A **UserCall** is the user-facing wrapper around a recording: name, attendees, summary, key points, action items. The recording (audio/video file + transcript) is the underlying media; the call is everything around it.

Two different identifiers per call — easy to confuse:

- **`id`** — integer (`Long`). Used in URL paths: `GET /usercall/{id}`, `DELETE /usercall/{id}`.
- **`uuid`** — string (UUID format). Used in **request bodies** for the "send to Jira/ClickUp/Airtable" endpoints. Get it from `GET /usercall/{id}.uuid`.

Always read both fields when fetching a call — you'll need the integer for follow-up gets and the UUID for action pushes.

## Listing + searching (read)

All require `recordings:read` scope.

### List most-recent calls

```
GET /api/usercall/all?page=0&size=20
```

Returns `[]` of `UserCallDTO`. Page + size pagination; defaults documented in the catalog. Most-recent first.

### Total count

```
GET /api/usercall/all/count
```

Returns a bare integer (e.g. `123`). Pair with `/all` for paginated UIs.

### Free-text search

```
GET /api/usercall/search?query=acme&page=0&size=20
```

Searches across call name, description, attendees, AND recording transcript content. Returns `[]` of matching calls. Empty `query=` returns all calls (same as `/all`).

### Search count

```
GET /api/usercall/search/count?query=acme
```

Bare integer. Pair with `/search` for paginated UIs.

### Get one

```
GET /api/usercall/{id}
```

Returns full `UserCallDTO` — name, description, attendees (list of `{email, name, role}`), recording IDs (`[{id, ...}]`), summary text, key points (list of strings), action items (list of `{description, assignee?, dueDate?}`), uuid, timestamps.

This is the primary "what did this call cover?" call. Read `summary` for an executive overview, `actionItems` for what to push to a task system, `recordings[].id` if you need to fetch audio/video URLs (see [recordings.md](recordings.md)).

## Mutating (write)

All require `recordings:write` scope.

### Delete a call

```
DELETE /api/usercall/{id}
```

**Hard-deletes both the UserCall metadata AND the underlying recording media. Not reversible.** Returns `Response.ok(true)` on success. Confirm with the user before calling this — even when the user says "delete the call about X", consider listing first and showing them what will be deleted.

### Send action items to Jira

```
POST /api/usercall/selection/jira
Content-Type: application/json

{
  "callUUID": "<uuid from get_call.uuid>",
  "cloudId": "<from list_integrations Jira entry>",
  "projectIds": ["ENG", "OPS"]
}
```

Hub maps each action item to a Jira issue under the listed projects. `projectIds` is a `Set<String>` — pass an array even when there's only one project. Body field is `projectIds` (plural), NOT `projectId`.

Requires:
- `recordings:write` scope
- A Jira integration on the workspace (`GET /v1/workspaces/{id}/integrations` shows it)

Returns boolean — `true` for success, `false` for soft failure (vendor tokens expired etc.). Surface `false` as "couldn't push, please reconnect Jira".

### Send action items to ClickUp

```
POST /api/usercall/selection/clickup
Content-Type: application/json

{
  "callUUID": "<uuid>",
  "selectedListId": "<ClickUp list id>"
}
```

Single list per call. Same auth + return semantics as Jira.

### Send action items to Airtable

```
POST /api/usercall/selection/airtable
Content-Type: application/json

{
  "callUUID": "<uuid>",
  "baseId": "<Airtable base id>",
  "tableIds": ["tbl_xxx", "tbl_yyy"]
}
```

`tableIds` is a `Set<String>` — Hub fans the action items out across the listed tables in the same base. Same auth + return semantics.

### Backfill metadata

```
POST /api/usercall/fill
```

Internal-style endpoint used by Hub's meeting-processing pipeline to enrich metadata. Rarely needed by external callers; the catalog lists it but you generally won't use it. If you do, the request shape is `UserCallFillRequest` — fetch the catalog detail.

## Common patterns

### Find a call by description and push its actions to Jira

```python
import requests, os

KEY = os.environ["DUTIFY_HUB_API_KEY"]
WS  = os.environ["DUTIFY_HUB_WORKSPACE_ID"]
H   = {"X-API-Key": KEY}

# 1. Find the call
calls = requests.get(
    "https://dutify.ai/api/usercall/search",
    headers=H, params={"query": "acme renewal"},
).json()
call = calls[0]   # most-recent match

# 2. Resolve cloudId from connected Jira integration
integrations = requests.get(
    f"https://dutify.ai/api/v1/workspaces/{WS}/integrations",
    headers=H,
).json()
jira = next(i for i in integrations if i["system"] == "JIRA")
cloud_id = jira["cloudId"]

# 3. Push action items
resp = requests.post(
    "https://dutify.ai/api/usercall/selection/jira",
    headers={**H, "Content-Type": "application/json"},
    json={
        "callUUID": call["uuid"],          # NOTE: uuid, not id
        "cloudId": cloud_id,
        "projectIds": ["ENG"],             # array, even for one project
    },
)
print(resp.json())   # true / false
```

### Iterate every visible call

Cursor pagination doesn't exist here — page through with `page` + `size` until you get fewer results than `size`:

```python
page = 0
size = 100
while True:
    batch = requests.get(
        "https://dutify.ai/api/usercall/all",
        headers=H, params={"page": page, "size": size},
    ).json()
    if not batch:
        break
    for c in batch:
        process(c)
    if len(batch) < size:
        break
    page += 1
```

For a planning UI where you want a total before iterating, hit `/api/usercall/all/count` first.
