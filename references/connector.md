# Connector compatibility

The combined connector is served at `https://mcp.dutify.ai/unified` after its release. Verify MCP initialization and `tools/list` before adopting it. Existing installations may continue using `https://mcp.dutify.ai/mcp` for Suite and `https://mcp-hub.dutify.ai/mcp` for Hub.

The unified connector accepts personal `du_live_` keys across granted products. The new adapter also supports `dw_live_` keys after the authority and consumers are deployed; see [integration keys](integration-keys.md). Existing `dk_live_` and `dh_live_` keys remain Suite-only and Hub-only. The connector does not combine old keys automatically. Select named accounts explicitly.

Hub business tools use `hub_` prefixes on the combined connector: `hub_list_calls`, `hub_get_call`, `hub_get_custom_prompt`. Existing Hub connections keep `list_calls`, `get_call`, `get_custom_prompt`. Confirm names in the endpoint's tool list. Shared discovery tools retain `whoami`, `list_accounts`, `list_workspaces` and `find_my_workspace`.

Use `workspace` per call or an explicit `X-Dutify-Workspace` default. Unified discovery returns UUID and Suite identifier for personal keys; legacy discovery may provide only its product identifier. Bound keys cannot switch workspaces. Personal account-owned calendar tools do not require selection; requested processing targets still require current access.

`get_api_catalog` accepts `service` (`pm`, `wiki`, `roadmarq`, `hub`) and optional `tag`. It uses existing catalogs; tags shared by Suite products return `AMBIGUOUS_CATALOG_TAG`. A missing product catalog is unavailable, not empty. These errors never justify substituting credentials.

Existing HTTP callers and this Hub skill do not depend on the combined MCP. The all-product `dutify-api` skill is optional for Hub-only installations. Installing a skill does not enable workspace-key issuance or change quotas.
