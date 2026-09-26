# Service-qualified API discovery

Prefer `GET /mp/api/v1/integration-catalog` on the configured Dutify origin when it returns `contractVersion: 1`. This is public discovery: omit API keys and workspace headers. Each service (`pm`, `wiki`, `roadmarq`, `hub`) reports `available` or `unavailable` independently. An outage is not an empty operation list or an authentication failure.

Use `/{service}` for that service's tags and `/{service}/{url-encoded-tag}` for details. Always retain the service identifier: identical tag and schema names in two services are distinct. A detail response includes operation paths, schemas and a trusted-origin `url`; paths are absolute from the HTTP origin, not relative to a prefixed API base URL. Verify the configured origin before any credentialed data request.

Only fall back to the existing Suite `/mp/api/v1/api-catalog` or Hub `/api/v1/api-catalog` when the new index is explicitly unsupported (404/501 or unsupported contract version). A missing tag in a supported catalog is a missing tag, not a reason to use a merged legacy detail. Do not fall back after 401, 403, 429 or transient failure. The legacy Suite catalog can merge shared tag/schema names; do not guess when it is ambiguous.

Catalogs describe operation availability, not caller authority. Existing upstream specifications do not consistently declare required API-key scopes and ownership. Missing authorization metadata is **unknown**, not unrestricted access. Read the product reference and operation description; never invent a scope or infer that a listed operation accepts a workspace credential. In particular, workspace integration keys exclude account calendars and credential administration.

This route is deployed separately from the skill. An updated skill alone does not make it available.
