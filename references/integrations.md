# integrations.md — Listing connected vendor integrations

```
GET /api/v1/workspaces/{workspaceId}/integrations
```

Scope: `integrations:read`. Returns the list of vendor toolkits the workspace has connected — Slack, Jira, ClickUp, Airtable, Notion, Gmail, Google Calendar, GitHub, Linear, Confluence, Atlassian, etc.

Read-only via API key. Connect / disconnect / OAuth flows are intentionally JWT-only because they require interactive consent screens that don't fit non-interactive callers.

## Response shape

A bare array of `AuthorizedServiceDto` — verified against prod, only two fields per entry:

```json
[
  {"integrationSystem": "JIRA",     "status": "CONNECTED"},
  {"integrationSystem": "CLICKUP",  "status": "CONNECTED"},
  {"integrationSystem": "AIRTABLE", "status": "CONNECTED"},
  {"integrationSystem": "PM",       "status": "ERROR"}
]
```

Field-level rules:

- **`integrationSystem`** — IntegrationSystem enum value. Common: `JIRA`, `CLICKUP`, `AIRTABLE`, `SLACK`, `NOTION`, `GMAIL`, `GCAL`, `GITHUB`, `LINEAR`, `CONFLUENCE`, `ATLASSIAN`, `PM`. Pull the authoritative full list from the catalog (`Workspace Settings` tag → `IntegrationSystem` schema).
- **`status`** — usually `CONNECTED`. Other values (`REVOKED`, `INITIATED`, `ERROR`) appear when the connection is in a bad state.

That's it. **Vendor-specific identifiers (`cloudId`, `selectedListId`, `baseId`, etc.) are NOT exposed via this endpoint** — Hub deliberately doesn't reveal them to API-key callers. To use a `send_to_jira/clickup/airtable` action you need those identifiers from the user (or from the vendor's own UI). The `list_integrations` endpoint only tells you which integrations exist + whether they're healthy; pushing actions requires the user to supply the cloudId / list-id / base-id explicitly.

## What this is for

Two main use cases:

### 1. Pre-flight check before attempting a "send to" call

```python
integrations = requests.get(
    f"https://dutify.ai/api/v1/workspaces/{ws}/integrations",
    headers={"X-API-Key": key},
).json()

if not any(i["integrationSystem"] == "JIRA" and i["status"] == "CONNECTED" for i in integrations):
    raise RuntimeError("No Jira integration on this workspace — connect one in Hub UI.")

# Then ask the USER for the cloudId + projectIds and call:
# POST /usercall/selection/jira  with  {callUUID, cloudId, projectIds}
```

### 2. Knowing what's available before suggesting a target

If the user says "send the action items somewhere", list integrations first and offer them the choice:

```python
options = [i["integrationSystem"] for i in integrations if i["status"] == "CONNECTED"]
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
    return any(i["integrationSystem"] == system and i["status"] == "CONNECTED" for i in integrations)

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

if preferred and not any(i["integrationSystem"] == preferred and i["status"] == "CONNECTED" for i in integrations):
    print(f"Workspace prefers {preferred} but it's not connected. The workspace settings are stale.")
```

## Footgun: `integrationSystem` is case-sensitive

The IntegrationSystem enum values are uppercase (`JIRA`, not `jira` or `Jira`). String comparison against the response field is byte-for-byte. If you're matching against user input, normalize with `.upper()` first.
