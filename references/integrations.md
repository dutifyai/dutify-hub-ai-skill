# integrations.md — Listing connected vendor integrations

```
GET /api/v1/workspaces/{workspaceId}/integrations
```

Scope: `integrations:read`. Returns the list of vendor toolkits the workspace has connected — Slack, Jira, ClickUp, Airtable, Notion, Gmail, Google Calendar, GitHub, Linear, Confluence, Atlassian, etc.

Read-only via API key. Connect / disconnect / OAuth flows are intentionally JWT-only because they require interactive consent screens that don't fit non-interactive callers.

## Response shape

A list of `AuthorizedServiceDto`:

```json
[
  {
    "system": "JIRA",
    "status": "CONNECTED",
    "cloudId": "abc-123",
    "lastSyncAt": "2026-05-09T18:00:00Z",
    "metadata": { ... vendor-specific fields ... }
  },
  {
    "system": "SLACK",
    "status": "CONNECTED",
    "teamId": "T091H4LUM6H",
    "lastSyncAt": "...",
    "metadata": { ... }
  }
]
```

Field-level rules:

- **`system`** — IntegrationSystem enum value. Common: `JIRA`, `CLICKUP`, `AIRTABLE`, `SLACK`, `NOTION`, `GMAIL`, `GCAL`, `GITHUB`, `LINEAR`, `CONFLUENCE`, `ATLASSIAN`, `PM`. Pull the authoritative full list from the catalog (`Workspace Settings` tag → `IntegrationSystem` schema).
- **`status`** — usually `CONNECTED`. Other values (`REVOKED`, `INITIATED`, `ERROR`) appear when the connection is in a bad state.
- **Vendor-specific identifiers** — `cloudId` (Jira), `teamId` (Slack), `baseId` (Airtable), `composioConnectedAccount` (Composio toolkits) — these are what you pass to the call-action push endpoints (`POST /usercall/selection/jira` etc.).

The exact field set varies per `system` because each vendor has different metadata. Treat the fields beyond `system` and `status` as optional and read them defensively.

## What this is for

Two main use cases:

### 1. Resolving the right `cloudId` / `selectedListId` / `baseId` for a "send to" call

```python
integrations = requests.get(
    f"https://dutify.ai/api/v1/workspaces/{ws}/integrations",
    headers={"X-API-Key": key},
).json()

jira = next((i for i in integrations if i["system"] == "JIRA"), None)
if jira is None:
    raise RuntimeError("No Jira integration on this workspace — connect one in Hub UI.")

# Now you have jira["cloudId"] for POST /usercall/selection/jira
```

### 2. Knowing what's available before suggesting a target

If the user says "send the action items somewhere", list integrations first and offer them the choice:

```python
options = [i["system"] for i in integrations if i["status"] == "CONNECTED"]
print(f"Connected: {', '.join(options)}. Which do you want?")
```

## What you can't do via API key

- **Connect a new integration** — needs OAuth consent. Direct the user to Hub UI → Integrations.
- **Disconnect an integration** — `DELETE /v1/workspaces/{id}/integrations/{system}` is JWT-only (`integrations:write` is intentionally not in the API-key scope catalog). User must do it from the UI.
- **Refresh tokens** — handled automatically by Hub's scheduled refresher; nothing to do from your side.
- **Read tokens** — Hub never exposes vendor access tokens via the API. Use the integration through Hub's wrapping endpoints (e.g. `/usercall/selection/jira` for pushing tasks).

## Common patterns

### Check whether a specific integration is connected before attempting to use it

```python
def has_integration(system: str) -> bool:
    integrations = requests.get(
        f"https://dutify.ai/api/v1/workspaces/{ws}/integrations",
        headers={"X-API-Key": key},
    ).json()
    return any(i["system"] == system and i["status"] == "CONNECTED" for i in integrations)

if not has_integration("JIRA"):
    print("Jira not connected — please connect via Hub UI then retry.")
    sys.exit(1)
```

### Cross-reference with workspace's preferred integration

```python
preferred = requests.get(
    f"https://dutify.ai/api/v1/workspaces/{ws}/settings/preferred-integration",
    headers={"X-API-Key": key},
).json()   # "JIRA" or null

integrations = requests.get(
    f"https://dutify.ai/api/v1/workspaces/{ws}/integrations",
    headers={"X-API-Key": key},
).json()

if preferred and not any(i["system"] == preferred and i["status"] == "CONNECTED" for i in integrations):
    print(f"Workspace prefers {preferred} but it's not connected. The workspace settings are stale.")
```

## Footgun: `system` is case-sensitive

The IntegrationSystem enum values are uppercase (`JIRA`, not `jira` or `Jira`). String comparison against the response field is byte-for-byte. If you're matching against user input, normalize with `.upper()` first.
