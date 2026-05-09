# auth.md — Hub API key authentication

Every Hub data-access call needs `X-API-Key: dh_live_<rest>`. Keys are issued from Hub UI → workspace → **API Keys** → Create.

## Key format

`dh_live_<32-base62>` — 40 chars total. Hub stores only the SHA-256 hash; the raw value is shown once at creation time and unrecoverable after. Treat it like a password.

The PM/Wiki/FR keys (`dk_live_…`) are NOT interchangeable — different backend, different scopes, different filter. If the user gives you a `dk_live_` key for Hub work, ask them to provision a Hub key.

## Workspace binding

Each key is bound to **exactly one workspace** at creation. Hub's `ApiKeyScopeFilter` rejects:

- Any path with `/v1/workspaces/{id}/...` where `{id}` ≠ the key's bound workspace UUID — returns 403 "API key cannot access this workspace"
- Even if the user is a member of multiple workspaces; you can't switch between them with one key. Provision a separate key per workspace.

`GET /api/v1/workspaces` from an API-key caller returns ONLY the bound workspace (other workspaces in the user's membership list are filtered out at the resource layer). To discover the bound workspace from a stored key, hit `GET /api/v1/workspaces` and read the `id` of the only entry returned.

## Scope catalog (9 scopes)

Scopes are granted at key creation; you cannot add scopes to an existing key. The filter maps the request path's "resource" segment to a scope prefix and the HTTP method to `:read` or `:write`.

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
| `recordings:write` | `POST /recording/{id}/reprocess`, `POST /recording/{id}/regenerate-summary`, `DELETE /usercall/{id}`, `POST /usercall/selection/{jira|clickup|airtable}`, `POST /usercall/fill` |
| `lens:chat` | `POST /lens/chat` (exact path only — `/lens/chat/stream`, `/lens/chat/anything-else`, `/v1/lens/...` are denied) |

## What's NOT exposed to API keys

Defense-in-depth blocks at the path filter, regardless of scopes granted:

- `/internal/*` — microservice-only endpoints
- `/webhooks/*` — Composio + vendor inbound webhooks (signed differently, not API-key-auth)
- `/v1/workspaces/{id}/api-keys/*` — creating/managing API keys (otherwise a leaked key could provision more keys for itself)
- `/user/credentials`, `/user/reset-password`, `/user/init` — account-takeover-class operations
- `/v1/workspaces/{id}/integrations/{service}/oauth`, `/internal-connect`, `DELETE` — OAuth flows need interactive consent, JWT-only

## Error responses for auth failures

| Code | Status | Meaning |
|---|---|---|
| 401 | Missing `X-API-Key` header, or value doesn't start with `dh_live_`, or key revoked / not found |
| 403 | "Insufficient scope. Required: <scope>" — your key doesn't have the scope this endpoint needs |
| 403 | "API key cannot access this workspace" — path workspace doesn't match the key's bound workspace |
| 403 | "API keys cannot manage other API keys" — you tried to hit `/v1/workspaces/{id}/api-keys/...` |
| 403 | "Endpoint is not exposed to API keys" — the path isn't in the filter's allowlist |
| 403 | `WORKSPACE_OUT_OF_SCOPE` — `setDefaultWorkspace` body had a workspaceId that wasn't the key's bound one |

For full error envelope shape see [errors.md](errors.md).

## Bearer-token alias

Hub also accepts `Authorization: Bearer dh_live_<rest>` as an equivalent to the `X-API-Key` header — useful when an HTTP client doesn't make custom headers easy. Either works; `X-API-Key` is the documented canonical form.
