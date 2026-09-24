# Account personal API keys

Personal credentials (`du_live_…`) belong to the signed-in account, work with Hub and the suite, and are available on all plans. Existing `dh_live_…` Hub and `dk_live_…` suite credentials remain workspace-bound. A key secret is shown only once; never put it in source control, logs, or tool output.

## Discover and select

1. Call `GET https://dutify.ai/api/v1/api-keys/current` without a workspace header. `keyType: PERSONAL` identifies an account key; `personalScopes` lists its full namespaced scopes across products, while `scopes` contains the Hub projection.
2. Call `GET https://dutify.ai/api/v1/personal-api-keys/workspaces` with `X-API-Key` and no workspace header. Each entry contains `identifier`, UUID `id`, `name`, `suiteAccess`, and `hubAccess`. Filter by `hubAccess`.
3. Choose the workspace from the user's request and current context. Disambiguate duplicate names before any mutation. For a cross-workspace search, issue separate reads for each relevant discovered workspace and label results with their workspace.
4. Send `X-Dutify-Workspace: <id>` on every data request, including direct calendar calls. URL/body workspace references must agree with that selection. Discovery and current-key introspection do not require a workspace.

With either MCP, use `list_workspaces` / `find_my_workspace` to discover and pass the `workspace` argument on data tools. A per-call selection overrides the configured default and is isolated from concurrent calls. A default never grants access. Confirm tool names in the connected server's tool list.

## Scopes and permissions

Personal scopes are product-qualified: `suite:tasks:read`, `hub:recordings:read`, `suite:*`, `hub:*`. Product wildcards grant the user's current permissions for that product, including administrative actions where supported. Read-only keys contain read scopes only. Scopes never elevate the user’s workspace role or product access. Account key administration uses `account:api-keys:read/write` separately. A key creating or expanding another credential cannot grant scopes it lacks.

Manage keys through the PM authority at `https://dutify.ai/mp/api/v1/personal-api-keys`: GET lists, POST creates, PUT `/{id}` edits name/scopes/expiry, DELETE `/{id}` revokes, and GET `/scopes` lists the accepted scope vocabulary. Mutation body: `{"name":"Agent","scopes":["hub:*"],"expiresAt":null}`. Prefer the account UI unless the user asks for API administration. Check the deployed catalog before relying on new endpoints during rollout.

## Workspace policy and limits

PM workspace Security settings include **Allow personal API keys**, enabled by default and controlled by the workspace settings/security permission. Disabling it denies personal-key access to both products. Membership loss, product-access loss, expiry, revocation, and policy changes are checked afresh on subsequent requests. Webhooks created or modified by a personal key are checked again before delivery. Personal keys do not add API-key authentication to the JWT-only WebSocket ticket flow.

A personal key consumes one remaining slot in the workspace's API-key count on first successful use, shared across products. Workspace keys have priority; remaining personal slots follow first-use order. A limit denial (`PERSONAL_API_KEY_LIMIT_REACHED`, 402) suggests upgrading the workspace or disabling personal-key access in Security settings. A 403 policy/access denial (`PERSONAL_API_KEYS_DISABLED` from PM or `PERSONAL_API_KEY_ACCESS_DENIED` from downstream services) requires checking the workspace policy, current membership, product access, and key scopes. Missing selection is 400. Do not retry these failures by switching workspaces or credentials without user direction.

The server can revoke access at any time. Re-discover as needed; never retain positive authorization decisions across requests. Keep discovery separate from data calls so an unavailable configured default does not prevent selecting another allowed workspace.

Keys created by a personal key inherit an expiry no later than their parent's. An omitted child expiry uses the parent's expiry; a later explicit expiry is rejected. Revoking a parent revokes its descendants. Key-management scopes remain in the full-permission default; read-only keys may list key metadata with `account:api-keys:read` but cannot mint or revoke keys.

Calendar events belong to the user’s account. Personal keys can list and manage the owner’s events across all workspace assignments, including unassigned events. An event’s workspace assignment supplies processing context; it does not determine event ownership or visibility. Calendar access still requires the relevant scopes and successful key authorization. Recordings remain subject to workspace access and opt-out checks.

Processing assignments must target a workspace the user belongs to. For personal keys, the target must also allow personal-key access and pass current product, scope, revocation, and quota checks. This restriction applies to assigning event/series context and creating a series prompt row; it does not filter account-owned event listings. Publish these instructions together with or after the Calendar target-access fix.
