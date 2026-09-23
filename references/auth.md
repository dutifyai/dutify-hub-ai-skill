# auth.md — Hub API key authentication

## Account personal keys

`du_live_…` keys work across Hub and the suite. Create/manage them in account settings. Discover workspaces and select one per request using `X-Dutify-Workspace`; see [personal-keys.md](personal-keys.md). The workspace-binding rules below describe existing `dh_live_…` workspace keys. For personal keys, workspace resources follow the workspace selected for this request. Calendar events remain account-owned across processing-workspace assignments; see [prompts.md](prompts.md).


Every Hub data-access call needs `X-API-Key` with either a `dh_live_…` workspace key or a `du_live_…` account personal key. Workspace keys are issued from Hub workspace settings → **API Keys**; personal keys from account settings.

## Key format

`dh_live_<32-base62>` — 40 chars total. Hub stores only the SHA-256 hash; the raw value is shown once at creation time and unrecoverable after. Treat it like a password.

The PM/Wiki/FR workspace keys (`dk_live_…`) are NOT interchangeable — different backend, different scopes, different filter. If the user gives you a `dk_live_` key for Hub work, ask them to provision a Hub workspace key or an account personal key (`du_live_…`).

## Workspace binding

Each `dh_live_…` workspace key is bound to **exactly one workspace** at creation. Hub's `ApiKeyScopeFilter` rejects:

- Any path with `/v1/workspaces/{id}/...` where `{id}` ≠ the key's bound workspace UUID — returns 403 "API key cannot access this workspace"
- Even if the user is a member of multiple workspaces; you can't switch between them with one key. Provision a separate key per workspace.

`GET /api/v1/workspaces` from an API-key caller returns ONLY the bound workspace (other workspaces in the user's membership list are filtered out at the resource layer). To discover the bound workspace from a stored key, hit `GET /api/v1/workspaces` and read the `id` of the only entry returned.

## Scope catalog (11 scopes)

Scopes are set at creation and may be edited by an authorized owner. The filter maps the request path's "resource" segment to a scope prefix and the HTTP method to `:read` or `:write`.

| Scope | Powers |
|---|---|
| `workspaces:read` | `GET /v1/workspaces`, `GET /v1/workspaces/{id}` |
| `members:read` | `GET /v1/workspaces/{id}/members`, `GET /v1/workspaces/{id}/members/{userId}` |
| `workspace-settings:read` | `GET /v1/workspaces/{id}/settings/custom-prompt`, `GET /v1/workspaces/{id}/settings/preferred-integration` |
| `workspace-settings:write` | `PUT` versions of the above |
| `user-settings:read` | `GET /user/settings/default-workspace` (per-user default workspace for events) |
| `user-settings:write` | `PUT /user/settings/default-workspace` — **with the constraint that the workspaceId in the body MUST match the key's bound workspace** (or be null to clear). 403 `WORKSPACE_OUT_OF_SCOPE` otherwise |
| `integrations:read` | `GET /v1/workspaces/{id}/integrations` (list connected toolkits) |
| `recordings:read` | `GET /usercall/all`, `/usercall/search`, `/usercall/{id}`, `/usercall/all/count`, `/usercall/search/count`; all `GET /recording/{id}/...` |
| `recordings:write` | `POST /recording/{id}/reprocess`, `POST /recording/{id}/regenerate-summary`, `DELETE /usercall/{id}`, `POST /usercall/selection/{jira|clickup|airtable}`, `POST /usercall/fill`; the custom-prompt writes — `PUT`/`DELETE` on `/v1/calendar/events/{eventId}/custom-prompt`, `/v1/calendar/series/{seriesMasterId}/custom-prompt` and `/usercall/{id}/custom-prompt` (see [prompts.md](prompts.md)) |
| `calendar:read` | `GET /v1/calendar/events` (the actor's calendar events, with any prompts already set) |
| `lens:chat` | `POST /lens/chat` (exact path only — `/lens/chat/stream`, `/lens/chat/anything-else`, `/v1/lens/...` are denied) |

## What's NOT exposed to API keys

Defense-in-depth blocks at the path filter, regardless of scopes granted:

- `/internal/*` — microservice-only endpoints
- `/webhooks/*` — Composio + vendor inbound webhooks (signed differently, not API-key-auth)
- `/v1/workspaces/{id}/api-keys/*` — blocked for workspace keys. Personal keys require `hub:api-keys:read/write` and the user’s current management permission; they cannot grant broader scopes than their own.
- `/user/credentials`, `/user/reset-password`, `/user/init` — account-takeover-class operations
- `/v1/workspaces/{id}/integrations/{service}/oauth`, `/internal-connect`, `DELETE` — OAuth flows need interactive consent, JWT-only

## Error responses for auth failures

| Code | Status | Meaning |
|---|---|---|
| 401 | Missing `X-API-Key` header, or value starts with neither `dh_live_` nor `du_live_`, or key revoked / not found |
| 403 | "Insufficient scope. Required: <scope>" — your key doesn't have the scope this endpoint needs |
| 403 | "API key cannot access this workspace" — path workspace doesn't match the key's bound workspace |
| 403 | "API keys cannot manage other API keys" — you tried to hit `/v1/workspaces/{id}/api-keys/...` |
| 403 | "Endpoint is not exposed to API keys" — the path isn't in the filter's allowlist |
| 403 | `WORKSPACE_OUT_OF_SCOPE` — `setDefaultWorkspace` body had a workspaceId that wasn't the key's bound one |

For full error envelope shape see [errors.md](errors.md).

## Bearer-token alias

Hub also accepts `Authorization: Bearer dh_live_<rest>` as an equivalent to the `X-API-Key` header — useful when an HTTP client doesn't make custom headers easy. Either works; `X-API-Key` is the documented canonical form.

### Personal-key denials

Personal-key authentication uses a flat `{code, message}` response. Check the status and code before suggesting a different key.

| Status | Code | Action |
| --- | --- | --- |
| 400 | `WORKSPACE_REQUIRED` | Discover accessible workspaces and send the selected canonical identifier in `X-Dutify-Workspace`. The key format is valid. |
| 401 | authentication failure | Check for a missing, invalid, expired, or revoked key, including a revoked or expired delegating parent. Both the product's workspace-key prefix and `du_live_` are supported. |
| 402 | `PERSONAL_API_KEY_LIMIT_REACHED` | Workspace keys have priority. Ask an administrator to upgrade capacity or disable personal-key access in workspace Security settings; do not switch workspaces to bypass the limit. |
| 403 | `PERSONAL_API_KEYS_DISABLED` / `PERSONAL_API_KEY_ACCESS_DENIED` / `ACCESS_DENIED` | Check workspace opt-out, current membership, product access, selected workspace, and scopes. Retry only after the relevant condition changes. Downstream products may normalize the code to `PERSONAL_API_KEY_ACCESS_DENIED`. |
| 503 | `PERSONAL_API_KEY_AUTHORITY_UNAVAILABLE` | Authorization could not reach its authority. Retry a read with bounded backoff; report a persistent outage. Never substitute cached authorization or repeat a mutation whose outcome is unknown. |
