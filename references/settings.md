# settings.md — Workspace settings + per-user default workspace

Two distinct setting layers, two distinct scopes, two distinct path prefixes:

| Layer | Scope | Path | What it controls |
|---|---|---|---|
| Workspace | `workspace-settings:read/write` | `/v1/workspaces/{id}/settings/...` | Custom AI prompt for recordings, preferred integration target |
| User (the api key's actor user) | `user-settings:read/write` | `/user/settings/default-workspace` | Where new calendar events route by default |

Workspace settings affect every user of the workspace. User settings affect only the api-key creator.

## Workspace setting: custom prompt

The custom prompt here is the **workspace-wide** one — the least specific of four levels. A single meeting, a recurring series, and an individual call can each carry their own; see [prompts.md](prompts.md) for the precedence (the workspace prompt applies *in addition* to those, not instead of them).

The custom prompt is **system instructions prepended to every recording's LLM analysis** — the workspace's "house style" for summaries, key points, and action items. Setting it well is one of the headline use cases for an API key (a sync script can keep the prompt locked to an external source of truth).

### Read

```
GET /api/v1/workspaces/{workspaceId}/settings/custom-prompt
```

Returns the prompt as **plain text**. The class carries `@Produces("application/json")`, but the method returns a bare `String` and that is written verbatim — so a prompt of `Summarise in bullet points` comes back as those 26 characters, *not* as a quoted JSON string.

Three shapes, all valid, none of them a 404:

| Status | Body | Means |
|---|---|---|
| `200` | the prompt as text | set |
| `200` | empty | set once, then cleared (`PUT` of an empty body) |
| `204` | empty | never set — the service returns `null` and JAX-RS makes that a 204 |

```python
r = requests.get(
    f"https://dutify.ai/api/v1/workspaces/{ws}/settings/custom-prompt",
    headers={"X-API-Key": key},
)
prompt = r.text or None          # str | None
# NOT r.json() — that raises unless the prompt happens to be valid JSON
```

### Write

```
PUT /api/v1/workspaces/{workspaceId}/settings/custom-prompt
Content-Type: text/plain

Always summarize calls in the style of a sales executive...
```

**The body is stored verbatim.** There is no `@Consumes` on the resource and the
parameter is a plain `String`, so whatever bytes you send become the prompt —
nothing is parsed, nothing is rejected, and there is no shape that fails.

That makes every "encode it as JSON" instinct a way to corrupt the setting.
Measured against the live endpoint:

| Sent | Stored as the prompt |
|---|---|
| `Summarise in bullet points` | `Summarise in bullet points` ✔ |
| `"Summarise in bullet points"` (JSON string) | the text **including both quote characters** |
| `{"customPrompt": "..."}` | that literal JSON text |
| `null` | the 4-character word `null` |
| `""` | two quote characters |
| *(empty body)* | cleared ✔ |

So **the only way to clear it is an empty body** — not `null`, not `""`. Each of
those stores itself and is then prepended to every recording's analysis.

```python
# Set — send raw bytes, never json=
requests.put(
    f"https://dutify.ai/api/v1/workspaces/{ws}/settings/custom-prompt",
    headers={"X-API-Key": key, "Content-Type": "text/plain"},
    data="Always summarize in third person and call out customer pain points.".encode(),
)

# Clear — an empty body
requests.put(
    f"https://dutify.ai/api/v1/workspaces/{ws}/settings/custom-prompt",
    headers={"X-API-Key": key, "Content-Type": "text/plain"},
    data=b"",
)
```

`json=` in `requests` is the trap: `json="text"` sends `"text"` with quotes and
`json=None` sends `null`, and both are stored literally.

**Read the current value before writing.** This route has no history and no
audit; an overwrite is unrecoverable through the API.

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

- **Wrapping the body in an object for `set_custom_prompt`** — Hub takes a `@RequestBody String`, so `{"customPrompt": "..."}` is not rejected; the literal JSON text is stored *as the prompt* and prepended to every recording's analysis. Bare string only, and read the current value before overwriting it — there is no history.
- **Reaching for `DELETE` to clear it** — there is no `DELETE` on this route (405). Clear it by `PUT`ing an empty **body**.
- **Treating "no prompt set" as an error** — there are two empty states and neither is a failure: **204 No Content** when a prompt was never set, and **200 with an empty body** once one has been cleared. The literal token `null` is never returned, so do not compare against it.
- **Clearing with `null` or `""`** — both are stored verbatim as the prompt. Only an empty body clears it.
- **Calling `.json()` on the read at all** — the response is plain text, so `.json()` raises for any prompt that isn't itself valid JSON, and again on the empty body of a cleared prompt. Use `r.text`.
- **Trying to set defaultWorkspace to a different workspace's UUID** — always 403. Set to the bound workspace or null.
- **PUT-ing `{workspaceId: ""}`** — empty string isn't a valid UUID, returns 400. Use `null` (JSON null) to clear.
