---
name: dutify-hub-api
description: Use the Dutify Hub HTTP API directly. Discover endpoints via the catalog at https://dutify.ai/api/v1/api-catalog, drill into a tag with /api-catalog/{tag} for full operation + schema detail, then call the endpoint with an X-API-Key header. Use this skill whenever the user wants to query, search, or act on Hub-side resources — call recordings, transcripts, summaries, signed audio/video URLs, workspace integrations, the workspace's custom AI prompt, the user's default workspace for events, sending call action items to Jira/ClickUp/Airtable, or asking Lens (programmatic chat) — even when they don't say "API" explicitly. Hub keys (dh_live_…) are NOT interchangeable with PM keys (dk_live_…); for PM/Wiki/Feature-Requests work, use the dutify-api skill instead.
---

# Dutify Hub API

Direct HTTP access to **Dutify Hub** — workspace settings, integrations, members, call recordings + transcripts, the user's default workspace, and Lens chat — via the API-key surface (`dh_live_…`).

This is the **second** of two Dutify HTTP skills. The other one — `dutify-api` — covers Project Management, Wiki, and Feature Requests with PM keys (`dk_live_…`). They're independent products with separate keys; pick whichever matches the data the user wants.

## When to use this skill

Typical signals from the user:

- "list my call recordings", "search calls about X"
- "what's in the call from yesterday?" / "what action items came out of the Acme call?"
- "set the workspace's custom prompt to X" / "what's our current Lens prompt?"
- "what integrations are connected to my workspace?"
- "send the action items from this call to our Jira project"
- "set my default workspace for events" / "where do new calendar events route?"
- "ask Lens what we discussed in last week's calls" (programmatic Lens chat)
- "give me a download link for the audio of recording 42"

## Topic map — load only what you need

This SKILL.md covers the orientation. Each focused reference below is loaded on demand:

| Reference | When to read |
|---|---|
| [auth.md](references/auth.md) | API-key header (`dh_live_…`), the 9 scopes, bound-workspace constraint, how to discover the bound workspace |
| [errors.md](references/errors.md) | Hub error envelope shape, common codes (401/403/404/`WORKSPACE_OUT_OF_SCOPE`), what `validOptions` looks like when present |
| [calls.md](references/calls.md) | `/usercall/...` — list, search, get, count, delete; integer ID vs UUID distinction; the `send_to_jira/clickup/airtable` "selection" endpoints with their per-vendor body shapes |
| [recordings.md](references/recordings.md) | `/recording/...` — progress, reprocess, regenerate-summary, signed audio/media/preview URLs; expiry semantics; integer recording IDs |
| [lens-chat.md](references/lens-chat.md) | `/lens/chat` — programmatic equivalent of Hub's Slack/Teams bot. The `ConversationalRagRequest` body, history shape, and the Hub-side short-circuit that lets API-key callers skip `tenantUuid`/`channelUuid` |
| [settings.md](references/settings.md) | `/v1/workspaces/{id}/settings/...` (workspace-settings) AND `/user/settings/default-workspace` (per-user). The "API keys can only set defaultWorkspace to their bound workspace" constraint |
| [integrations.md](references/integrations.md) | `GET /v1/workspaces/{id}/integrations` — read-only listing of connected vendor toolkits (Slack, Jira, ClickUp, Airtable, Notion, Gmail, GCal, GitHub, ...). Why connect/OAuth flows are JWT-only |
| [workspace.md](references/workspace.md) | `/v1/workspaces` listing (returns ONLY the bound workspace for API-key callers); `/v1/workspaces/{id}/members` |

When in doubt, fetch the catalog tag detail first — it's the deployed contract.

## Core idea: discover, then call

Hub exposes its OpenAPI catalog at the same path shape PM uses. Always start there — never assume an endpoint shape from memory, because the catalog is what's actually deployed.

Three steps for any task:

1. **List tags** — `GET https://dutify.ai/api/v1/api-catalog`. Returns Hub's tags with descriptions and operation counts. Pick the tag closest to what the user wants.
2. **Get tag detail** — `GET https://dutify.ai/api/v1/api-catalog/{tag}`. URL-encode tag names with spaces (`Workspace Settings` → `Workspace%20Settings`). Returns `{tag, description, service, baseUrl, operations[], schemas{}}` — operations have `method`, `path`, `summary`, `description`, `parameters`, `requestBody`, `responses`; schemas resolve `$ref` values.
3. **Call the endpoint.** Send `X-API-Key: dh_live_…` and (for write endpoints) `Content-Type: application/json`.

There's also `GET /api-catalog/detailed` if you want the full thing in one shot, but it's larger — prefer the per-tag endpoint when you know which tag.

### Footgun: catalog `path` is host-absolute

The catalog's `path` field is **absolute from the host root** — it already includes the `/api` prefix that's in `baseUrl`. So `path` looks like `/api/v1/workspaces/{id}/settings/custom-prompt` and `baseUrl` is `https://dutify.ai/api`; naive `{baseUrl}{path}` concatenation produces a duplicate prefix and 404s. Either use `https://dutify.ai` + `path`, or strip the duplicated prefix from `path` before concatenating with `baseUrl`. The `baseUrl` field is mostly informational.

## What the catalog covers

The Hub catalog at `https://dutify.ai/api/v1/api-catalog` covers ONLY Hub. (PM/Wiki/Feature-Requests are aggregated separately at `https://dutify.ai/mp/api/v1/api-catalog` — covered by the `dutify-api` skill.)

| Tag | Scope | What lives there |
|---|---|---|
| `Workspaces` | `workspaces:read`, `members:read` | List workspaces, list members, member ops |
| `Workspace Settings` | `workspace-settings:read/write` | Custom AI prompt, preferred integration target |
| `Integrations` | `integrations:read` | List connected vendor toolkits |
| `Recordings` | `recordings:read/write` | Recording progress, reprocess, regenerate summary, signed audio/media/preview URLs |
| `Calls` | `recordings:read/write` | UserCall list/get/search/count/delete, send-to-Jira/ClickUp/Airtable |
| `Lens` | `lens:chat` | Programmatic Lens chat (full pipeline against the workspace's knowledge) |
| `Reference Data` | (public) | The catalog endpoints themselves |

User-level settings (`get/set_default_workspace`) live under `Workspaces` in the catalog but use a separate `user-settings:*` scope — see [settings.md](references/settings.md).

## Auth model — quick

Every data-access call needs `X-API-Key: dh_live_<rest>`. Keys are **workspace-bound** (one key, one workspace) and **scope-gated** (per resource family — see `auth.md`). Hub's `ApiKeyScopeFilter` rejects:

- Path-workspace mismatches (`/v1/workspaces/{id}/...` where `{id}` ≠ the key's bound workspace)
- Endpoints not on the API-key allowlist (e.g. anything under `/internal`, `/webhooks`, OAuth flows, `/user/credentials`, `/user/reset-password`)
- Calls without the right scope (returns the required scope in the error message)

Full table of scopes + auth failure modes in [auth.md](references/auth.md).

## Hub vs Lite-API conventions (different from PM)

Hub is **not** organized into "lite" vs "non-lite" tags like PM is. There's only one set of endpoints; they generally take **identifiers** (UUIDs / integer IDs), not names:

- Workspace identifiers are UUIDs (e.g. `00000000-0000-0000-0000-000000000000`).
- Recording IDs are **integers** (`Long`).
- Call IDs are also integers; call **UUIDs** (used by `send-to-*` endpoints) are strings — get them from `GET /usercall/{id}.uuid`.
- Integration system names (e.g. `JIRA`, `CLICKUP`, `AIRTABLE`) are enum values — case matters; pull the catalog of valid values from `GET /v1/workspaces/{id}/integrations`.

When the user only knows a workspace name or a call by description, you'll need to list first and resolve.

## Pagination

UserCalls list/search use **page + size** (NOT cursor pagination like PM). Defaults vary; the catalog `parameters` block has the authoritative numbers. To know how many total results exist for a search, hit the matching `/count` endpoint. There's no `cursor`, no `nextCursor`.

## Calling pattern (Python)

Pseudocode that works in any language with an HTTP client. **Don't trust the field names below verbatim — get them from the catalog tag detail.**

```python
import requests, urllib.parse, os

API_KEY = os.environ["DUTIFY_HUB_API_KEY"]
WORKSPACE_ID = os.environ["DUTIFY_HUB_WORKSPACE_ID"]   # the UUID this key is bound to
CATALOG = "https://dutify.ai/api/v1/api-catalog"
HEADERS = {"X-API-Key": API_KEY}

# 1. (optional) Discover the right tag
tags = requests.get(CATALOG).json()

# 2. Drill in to confirm the endpoint contract
detail = requests.get(f"{CATALOG}/Calls").json()

# 3. Call the endpoint. Note: 'path' from the catalog is host-absolute,
#    so prefix with https://dutify.ai (NOT baseUrl).
resp = requests.get(
    "https://dutify.ai/api/usercall/search",
    headers=HEADERS,
    params={"query": "acme", "page": 0, "size": 20},
)
resp.raise_for_status()
calls = resp.json()
for c in calls:
    print(c["id"], c["name"], c.get("summary"))
```

If the host language already has an HTTP idiom (Node `fetch`, `curl`, `axios`, `httpx`), use that — the structure is the same.

## Putting it together — a worked example

"Find action items from yesterday's Acme call and push them to Jira project ENG":

1. `GET /api/usercall/search?query=Acme&page=0&size=10` → find the call. Pick the integer `id` of the most recent matching one. Pull its `uuid` field too.
2. `GET /api/usercall/{id}` → confirm it's the right call, read `summary` / `actionItems` to verify there's something worth pushing.
3. `GET /api/v1/workspaces/{wsId}/integrations` → confirm Jira is connected; pull the `cloudId` from the Jira entry.
4. `POST /api/usercall/selection/jira` with body `{"callUUID": "<uuid>", "cloudId": "<from-step-3>", "projectIds": ["ENG"]}` (note: `projectIds` is an **array** of one or more Jira project ids, not a single string — the DTO is `Set<String>`).
5. The response is a boolean indicating whether the apply succeeded. Surface to the user with the call name + count of action items pushed.

If any step fails with a structured error body, read `code` and `message` (and `validOptions` when present) — see [errors.md](references/errors.md) for the contract.

## When to use the dutify-hub-mcp server instead

The `dutify-hub-mcp` MCP server (https://hub-mcp.dutify.ai/mcp) wraps the same API surface as named MCP tools (`list_calls`, `lens_chat`, `set_default_workspace`, etc.). Use the MCP when:

- You're a host that speaks MCP and the user wired the server in (Claude Desktop, Claude Code with mcp_servers.json, etc.)
- You want pre-baked argument schemas + per-tool descriptions
- You want the workspace-name → UUID resolver done for free per request

Use this skill (direct HTTP) when:

- You're writing a script / one-shot automation outside an MCP-speaking host
- You want maximum control (custom retry logic, batching, custom auth flows)
- You're using a less-common HTTP idiom (curl, `xh`, Postman) where MCP isn't an option

Both consume the same Hub. Both honour the same key + scope model.
