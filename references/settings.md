# settings.md — Workspace settings + per-user default workspace

Two distinct setting layers, two distinct scopes, two distinct path prefixes:

| Layer | Scope | Path | What it controls |
|---|---|---|---|
| Workspace | `workspace-settings:read/write` | `/v1/workspaces/{id}/settings/...` | Custom AI prompt for recordings, preferred integration target |
| User (the api key's actor user) | `user-settings:read/write` | `/user/settings/default-workspace` | Where new calendar events route by default |

Workspace settings affect every user of the workspace. User settings affect only the api-key creator.

## Workspace setting: custom prompt

The custom prompt is **system instructions prepended to every recording's LLM analysis** — the workspace's "house style" for summaries, key points, and action items. Setting it well is one of the headline use cases for an API key (a sync script can keep the prompt locked to an external source of truth).

### Read

```
GET /api/v1/workspaces/{workspaceId}/settings/custom-prompt
```

Returns a JSON-encoded **string** (the prompt) or JSON `null` when none has been set. Both shapes are valid; check for null before using.

```python
prompt = requests.get(
    f"https://dutify.ai/api/v1/workspaces/{ws}/settings/custom-prompt",
    headers={"X-API-Key": key},
).json()    # might be a string OR None
```

### Write

```
PUT /api/v1/workspaces/{workspaceId}/settings/custom-prompt
Content-Type: application/json

"Always summarize calls in the style of a sales executive..."
```

The body is a **bare JSON string** (with surrounding quotes) — NOT an object wrapping it. The frontend sends with `Content-Type: text/plain` (raw string, no quotes). Either works because the backend's `@RequestBody String` accepts both shapes; pick whichever your HTTP client makes natural.

To clear the prompt, PUT a JSON `null` (4 chars literal) or a JSON empty string `""`.

```python
# Set
requests.put(
    f"https://dutify.ai/api/v1/workspaces/{ws}/settings/custom-prompt",
    headers={"X-API-Key": key, "Content-Type": "application/json"},
    json="Always summarize in third person and call out customer pain points.",   # bare string
)

# Clear
requests.put(
    f"https://dutify.ai/api/v1/workspaces/{ws}/settings/custom-prompt",
    headers={"X-API-Key": key, "Content-Type": "application/json"},
    json=None,
)
```

Takes effect on the **next recording processed**. Existing recordings are not re-analysed automatically; trigger `/recording/{id}/regenerate-summary` to apply the new prompt to a specific recording (see [recordings.md](recordings.md)).

## Workspace setting: preferred integration

Which task system new actions route to by default — Jira / ClickUp / Airtable / PM. Used by the call-action-item flows when the user doesn't pick a target explicitly.

### Read

```
GET /api/v1/workspaces/{workspaceId}/settings/preferred-integration
```

Returns the IntegrationSystem enum value as a JSON-quoted string: `"JIRA"`, `"CLICKUP"`, `"AIRTABLE"`, `"PM"`, etc. Get the full list of valid values from `GET /v1/workspaces/{id}/integrations` (see [integrations.md](integrations.md)).

### Write

```
PUT /api/v1/workspaces/{workspaceId}/settings/preferred-integration
Content-Type: application/json

"JIRA"
```

Body is the **bare enum value as a JSON-quoted string**. NOT `{preferredIntegration: "JIRA"}`. Hub deserializes the JSON string directly into the `IntegrationSystem` enum.

If the value isn't a known enum, you get a 400 — Quarkus's enum deserializer doesn't fall back. Pull the valid values from the catalog or from `/integrations`.

## User setting: default workspace (for events)

This is the per-USER setting that controls **where new calendar events route by default** when they don't have an explicit workspace assignment. It's a property of the user, not the workspace — but it affects which workspace ends up owning new content.

### Read

```
GET /api/user/settings/default-workspace
```

Returns `{workspaceId: "<uuid>"}` or `{workspaceId: null}` when none is set.

### Write — with a critical constraint

```
PUT /api/user/settings/default-workspace
Content-Type: application/json

{"workspaceId": "<uuid>"}
```

For an API-key caller, **the only `workspaceId` value Hub will accept is the key's bound workspace UUID** (or `null` to clear). Any other UUID returns:

```
HTTP/1.1 403 Forbidden
{
  "error": "API keys can only set the default workspace to the workspace they are bound to. ...",
  "code": "WORKSPACE_OUT_OF_SCOPE"
}
```

Why: if a user has 3 workspaces (A, B, C) and an API key bound to A, that key MUST NOT be able to redirect the user's new events to B or C. The constraint is enforced at the resource layer (in addition to the path-allowlist done by `ApiKeyScopeFilter`).

In practice, a script using an API key only ever wants to "set my default to THIS workspace" anyway — the key is bound to it.

```python
# Set default to the bound workspace
requests.put(
    "https://dutify.ai/api/user/settings/default-workspace",
    headers={"X-API-Key": key, "Content-Type": "application/json"},
    json={"workspaceId": bound_workspace_uuid},
)

# Clear default
requests.put(
    "https://dutify.ai/api/user/settings/default-workspace",
    headers={"X-API-Key": key, "Content-Type": "application/json"},
    json={"workspaceId": None},
)
```

JWT callers (interactive Hub UI users) can set the default to any workspace they're a member of — the constraint above only applies when the principal is an API key.

## Anti-patterns

- **Wrapping the body in an object for `set_custom_prompt`** — Hub's `@RequestBody String` won't deserialize `{"customPrompt": "..."}`. Bare string only.
- **Treating "no prompt set" as an error** — `GET .../custom-prompt` returning `null` is a valid state, not a 404. Default-empty.
- **Trying to set defaultWorkspace to a different workspace's UUID** — always 403. Set to the bound workspace or null.
- **PUT-ing `{workspaceId: ""}`** — empty string isn't a valid UUID, returns 400. Use `null` (JSON null) to clear.
