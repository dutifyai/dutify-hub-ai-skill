# dutify-hub-api — Claude Code skill

A Claude Code (and Claude.ai) skill that teaches an LLM how to use the **Dutify Hub** HTTP API directly: discover endpoints via the catalog at `https://dutify.ai/api/v1/api-catalog`, call the right tag with an `X-API-Key`, and self-correct on structured errors instead of guessing endpoint shapes from memory.

Covers the Hub API surface in one skill: workspace settings, integrations, members, call recordings + transcripts, the user's default workspace for events, and Lens chat.

This is the **second** of two Dutify HTTP skills:

- [`dutify-api`](../dutify-api) — Project Management, Wiki, Feature Requests. PM API keys (`dk_live_…`).
- **`dutify-hub-api` (this skill)** — Hub. Hub API keys (`dh_live_…`).

The two skills are independent and use different keys; pick whichever matches the data the user wants to work with.

## Install

### User-level (any Claude Code session, any project)

```bash
git clone https://github.com/dutifyai/dutify-hub-ai-skill.git ~/.claude/skills/dutify-hub-api
```

After install, restart your Claude Code session (or `/clear`). The skill auto-loads when a prompt mentions Hub-side resources ("list my call recordings", "set my custom prompt", "send action items to Jira", "ask Lens about…", "set my default workspace", etc.) — no need to invoke it explicitly.

### Project-level (only when working inside one project)

```bash
git clone https://github.com/dutifyai/dutify-hub-ai-skill.git <your-project>/.claude/skills/dutify-hub-api
```

### Verify

After install:

```bash
ls ~/.claude/skills/dutify-hub-api/SKILL.md         # should exist
ls ~/.claude/skills/dutify-hub-api/references/      # 7 topic files
```

Then in any Claude Code session, ask "what's the URL for downloading the audio of a Dutify recording?" — Claude should pick up the skill, load `references/recordings.md`, and answer.

## How the skill is laid out

`SKILL.md` is the orientation file an LLM always sees; the 7 reference files in `references/` are loaded on-demand based on the topic map.

| File | Topic |
|---|---|
| `SKILL.md` | Orientation: discover→call flow, MCP-vs-skill choice, lite-vs-non-lite (n/a — Hub doesn't split), pagination, calling pattern, worked example |
| `references/auth.md` | API-key header, the 9 scopes, bound-workspace constraint, what's NOT exposed to API keys, error responses for auth failures |
| `references/errors.md` | Hub error envelope shape, status code vocabulary, error code vocabulary (`WORKSPACE_OUT_OF_SCOPE` etc.), silent-failure modes |
| `references/calls.md` | UserCalls — list, search, get, count, delete; integer ID vs UUID distinction; send-to-Jira/ClickUp/Airtable per-vendor body shapes |
| `references/recordings.md` | `/recording/...` — progress, reprocess, regenerate-summary, signed audio/media/preview URLs; expiry semantics |
| `references/lens-chat.md` | `/lens/chat` — programmatic equivalent of Slack/Teams bot; `ConversationalRagRequest` body, history shape, the 90s timeout |
| `references/settings.md` | Workspace custom prompt, preferred integration, AND `/user/settings/default-workspace` with the bound-workspace constraint |
| `references/integrations.md` | `GET /v1/workspaces/{id}/integrations` — read-only listing of connected vendors |
| `references/workspace.md` | `/v1/workspaces` (returns ONLY the bound workspace for API-key callers); members |

## Authentication

Every Hub data-access call needs `X-API-Key: dh_live_<rest>`. Generate keys via the Hub UI under workspace settings → API Keys. One key is bound to one workspace; the scope filter rejects path-workspace mismatches with `403`. See `references/auth.md` for the full scope vocabulary (9 scopes).

## Why a topic-indexed skill rather than one long doc

Skills load fully whenever they're triggered — a 1000-line single doc would burn that much context per task. Splitting orientation in `SKILL.md` and detail per topic means a Lens-only task only loads `lens-chat.md`; a recordings task loads `recordings.md` + maybe `auth.md`; nothing else. SKILL.md is ~140 lines; each reference averages ~150 lines.

## When to use this skill vs the dutify-hub-mcp server

Both wrap the same Hub API surface. Pick one:

- **Skill (this) — direct HTTP** — best when writing scripts / one-shot automation outside an MCP-speaking host, when you want maximum control (custom retry, batching, custom auth flows), or when using less-common HTTP idioms (curl, `xh`, Postman).
- **MCP** — best when you're a host that speaks MCP (Claude Desktop, Claude Code with `mcp_servers.json`) and the user wired the server in. Pre-baked tool schemas and per-tool descriptions; workspace-name resolution done for free.

Both consume the same Hub backend. Both honour the same key + scope model. The MCP server lives at `https://mcp-hub.dutify.ai/mcp` (project source: [`mcp/dutify-hub-mcp`](../../mcp/dutify-hub-mcp)).

## Contributing

The canonical source for this skill lives at `Dutify-suite/skills/dutify-hub-api/` inside the Dutify suite monorepo. Edit at the source.

## License

Internal Dutify documentation. Use of the API requires a valid Dutify Hub API key.
