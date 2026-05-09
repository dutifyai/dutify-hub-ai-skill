# errors.md — Hub error responses

Hub error envelopes are simpler than PM's nested `{errorCode, message, details:{field, providedName, validOptions}}` shape. Most Hub endpoints return one of:

```json
{"error": "Insufficient scope. Required: recordings:read"}
```

or

```json
{"message": "Recording not found: 42"}
```

Some newer endpoints (the `/v1/lens/proposals/*` family, the `setDefaultWorkspace` constraint check) return a structured form:

```json
{"error": "API keys can only set the default workspace to the workspace they are bound to. ...", "code": "WORKSPACE_OUT_OF_SCOPE"}
```

Bean-validation failures (e.g. missing required fields) return RestEasy's default violation list — typically a JSON object with `parameterViolations` or a plain text body. Treat as `400 BAD_REQUEST`.

## Status code vocabulary

| Status | Meaning | Typical fix |
|---|---|---|
| 400 | Bad request — malformed body, validation failure, missing query param | Read the message; fix the input |
| 401 | API key missing or wrong format (must start with `dh_live_`), or key revoked | Reissue a key from Hub UI |
| 403 | Scope or workspace boundary violation | See [auth.md](auth.md) — message tells you which |
| 404 | Endpoint not exposed to API keys, or referenced resource doesn't exist | If "Endpoint is not exposed to API keys" — the path isn't in the API-key allowlist; this can't be fixed without provisioning a different scope/auth |
| 409 | Conflict (rare on Hub — mostly for DB unique-constraint conflicts) | Body usually has actionable detail |
| 429 | Rate limited | Retry with backoff |
| 500 | Server error | Surface to user; not safely retryable |
| 504 | Backend timeout (most likely on `/lens/chat` for synthesis-heavy queries) | Retry once; if it persists, break the query into smaller pieces |

## Error code vocabulary (when present)

Not every Hub error has a `code` field. The ones that do:

| Code | Where it appears | What to do |
|---|---|---|
| `WORKSPACE_OUT_OF_SCOPE` | `PUT /user/settings/default-workspace` from API key with mismatched workspaceId | Use the bound workspace UUID (or null to clear). See `auth.md` |
| `PROPOSAL_NOT_FOUND` / `PROPOSAL_EXPIRED` | Composio proposal resume/reject | Proposal already processed or TTL'd out (5 min default) |
| `INVALID_CONFIRMATION_TOKEN` | Composio proposal resume/reject | Token doesn't match what was issued; usually means the user clicked an old card |

## When the body is just a string

Some Hub endpoints (mostly older `/recording/...` ones) return a plain text body for errors — `Response.serverError().entity("Some error message").build()` style. Read the response status first; if it's not 2xx and the content-type isn't JSON, treat the body as the error message.

## What to do in code

- Parse the response body as JSON.
- If parsing fails, fall back to the raw text.
- Read `error` first, then `message`, then `detail`. Use whichever is present; they're never both populated on the same response.
- If `code` is present, log it — agents will need it to self-correct (e.g. on `WORKSPACE_OUT_OF_SCOPE`, retry with the correct workspace).
- Don't retry on 400 / 401 / 403 / 404. Retry once on 429 / 504. Retry with backoff on 5xx (other than 504).

## Network-layer errors

A `502 Bad Gateway` or `503 Service Unavailable` from Hub's ingress means the backend pod is restarting or hasn't come up yet — retry with exponential backoff (initial 1s, max ~30s, give up after a minute). Don't surface these to the user as Hub failures; they're transient infrastructure events.

A connection timeout (no response within 30s for non-Lens endpoints, 90s for Lens) means the request never reached the backend or the backend hung. Same retry policy.

## Common silent-failure modes

These don't error — they return 2xx but with empty/null payloads:

- `GET /v1/workspaces/{id}/settings/custom-prompt` returns the literal string `null` (JSON null, not the absence of a key) when no prompt has been set. Treat as "no prompt configured".
- `GET /usercall/search?query=` with no matching results returns `[]`, not 404.
- `GET /usercall/{id}` for a deleted call returns 404, not 200 with `{deleted: true}` — check status, not body.
- `POST /usercall/selection/jira` returns the boolean `true` on success, `false` on a soft failure (e.g. integration tokens were valid at scope-check time but expired by the time the actual Jira API was called). Surface false to the user as "couldn't push" — don't treat it as success.
