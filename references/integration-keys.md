# Workspace integration keys

`dw_live_` is PM-issued, bound to one workspace and a named member (initially its creator), with explicit Hub and/or Suite grants. It cannot switch workspaces. This contract rolls out separately from this skill; do not assume the deployment accepts or issues these credentials.

With a supplied `dw_live_`, use PM `GET /mp/api/v1/api-keys/current` without workspace selection and require `keyType: INTEGRATION`. It returns `workspaceUuid`, Suite `workspaceIdentifier`, `workspaceName`, `qualifiedScopes`, `availableProducts`, projected Suite `scopes`, and Suite `resourceAccess`. Hub's own `GET /api/v1/api-keys/current` uses `workspaceIdentifier` for the UUID and `suiteWorkspaceIdentifier` for the Suite identifier; it returns `contractVersion: 1`. Do not interchange those fields. Prefixes select validation; they do not establish identity.

Send the same `X-API-Key` to granted products. Suite calls use the Suite identifier; Hub calls use the UUID. Headers, paths, queries and bodies must agree with the fixed binding. Current membership, product access, member permissions, explicit scopes, expiry/revocation and resource ACLs all apply. Discover scopes rather than inventing grants. `resourceAccess` restricts the Suite hierarchy; Hub enforces recording ownership/visibility. The personal-key opt-out does not disable workspace keys.

These keys cannot access account-owned calendars, calendar subscriptions/prompts, account preferences, Hub billing or credential administration. A recording write grant does not authorize a calendar prompt write. Never silently switch credentials after denial.

## Management and deployment

The additive namespace is `/mp/api/v1/workspaces/{identifier}/integration-api-keys`. `/capabilities` reports `contractVersion`, `issuanceEnabled`, `actorMode`, and `delegation`. Version 1 requires an interactive user token (`delegation: false`); personal API keys cannot mint these keys. Do not call internal validators or request service secrets.

When management is requested and interactive authentication is available, inspect capabilities and the deployed catalog. Contract v1 reports `issuanceEnabled: true` and needs no activation flag. A missing or unsupported contract means this workflow is not deployed; report that rather than retrying through a legacy issuer. Unsupported versions may need a supported workflow; 401/403 and outages do not authorize fallback.

POST accepts `name`, concrete `scopes`, optional future `expiresAt`, and Suite `resourceAccess`, returning `rawKey` once. Lists omit secrets/hashes and distinguish revoked, expired and suspended actor access. `/scopes` returns scopes the current signed-in member can manage. PUT edits metadata/grants; DELETE revokes. Actor/workspace are immutable, and management requires permission for every granted product. A multi-product key consumes one PM API-key slot, with priority over personal-key slots. Legacy Hub quota remains separate. Creating a replacement never implies revoking the old key.

The unified Workspace API keys page aggregates integration, Suite-only and Hub-only credentials using the signed-in user's authorized API calls. Row identity is `(issuer, id)`; equal IDs from different issuers are not the same key. Each legacy update/revoke goes to its original service. An unavailable issuer does not mean its keys were deleted. A successful create/update with failed refresh remains successful; preserve the one-time secret and do not repeat issuance.

For requested rotation, `POST /{id}/rotate` takes `{"mode":"OVERLAP"}` or `{"mode":"REPLACE"}`. OVERLAP needs a spare PM slot and leaves the previous key active until explicitly revoked. REPLACE revokes the previous secret in the same transaction as issuing the replacement and works without a spare slot; callers cut over immediately. Confirm the user's cutover choice when it was not specified. Both modes preserve the named actor, permissions, restrictions and expiry. Actor departure requires an authorized new key, not silent reassignment. Rotation returns its secret once and must not be retried blindly after an uncertain response.

Read [qualified catalog discovery](catalog.md) before discovering new management operations.

On 403, check binding, membership, product access and scope. On 503, report authority availability and retry safe reads with bounded backoff. Do not label an outage as revocation or repeat mutations with unknown outcomes. Existing credential and connector contracts stay supported.
