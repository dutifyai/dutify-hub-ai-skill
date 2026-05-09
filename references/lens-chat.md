# lens-chat.md — Programmatic Lens chat

`POST /api/lens/chat` is Hub's programmatic Lens entrypoint — the **same conversational AI** that powers the Slack and Teams bots, but as a synchronous HTTP call that returns the answer in the response body.

Scope: `lens:chat`. Hub's `ApiKeyScopeFilter` only allows the **exact path** `/lens/chat` for API-key callers — `/lens/chat/stream` (SSE), `/v1/lens/chat/query` (async with callbacks), and any other lens path is denied.

## Request shape

```
POST /api/lens/chat
Content-Type: application/json
X-API-Key: dh_live_…

{
  "lastMessage": {
    "user": "your-script",
    "datetime": "2026-05-09T18:00:00Z",
    "content": "What action items came out of yesterday's Acme call?"
  },
  "history": [
    {
      "user": "your-script",
      "datetime": "2026-05-09T17:55:00Z",
      "content": "Summarize the Acme renewal call"
    },
    {
      "user": "assistant",
      "datetime": "2026-05-09T17:55:30Z",
      "content": "The Acme call covered..."
    }
  ]
}
```

Field rules:

- **`lastMessage` (required)** — `RagChatMessage` shape. `user` is your script's identifier (any string — used for display attribution). `datetime` is ISO-8601 UTC. `content` is the user's actual question.
- **`history` (optional)** — list of prior turns in **chronological order** (oldest first). Max 20 messages. Include both user and assistant turns so Lens can resolve references like "the call we just discussed" / "based on what you said before".
- **`tenantUuid` and `channelUuid`** — required for Slack/Teams chat-bot callers, **ignored for API-key callers**. Hub's `LensRequestValidator` detects the API-key principal and short-circuits the workspace lookup to use the bound workspace from the key. Send empty strings (or omit the fields entirely; they're no longer `@NotBlank`).

## Response shape

The response is the **synthesized answer as a string**, wrapped in JSON depending on what the agent decided to return:

- For most queries: a plain string with the answer text — possibly markdown-formatted.
- For task-creation queries: the response can include embedded `<tasks>...</tasks>` JSON tags listing structured tasks the agent generated. Strip + parse separately if you want to render them as cards.
- For confirmation flows (e.g. "send this email"): the agent may emit a proposal envelope. The exact shape is documented under the `Lens` tag in the catalog; for API-key flows you generally won't trigger this path because Composio actions need the async `/v1/lens/chat/query` endpoint (which API keys can't reach).

The HTTP status is `200 OK` for any successful synthesis (including "no relevant data" answers). Error statuses:

| Status | Why |
|---|---|
| 400 | Validation failure (e.g. lastMessage missing), or input guardrail blocked the request |
| 403 | Missing `lens:chat` scope, or rate-limited |
| 429 | Lens rate limiter triggered |
| 504 | Synthesis took longer than the backend deadline (~60s default; bump client timeout to 90s for safety) |

## Latency expectations

- Simple cached queries: ~0.5-2s
- Single-tool queries (e.g. "list my recordings"): ~3-8s
- Multi-source synthesis (fan-out across recordings + calls + transcripts): ~15-30s
- Worst-case agent-loop queries with multiple iterations: up to 60s

Set your client read timeout to **90s** to avoid false 504s. If a query consistently times out, break it into smaller questions.

## Pattern: ask Lens what action items came out of yesterday's calls

```python
import requests, os
from datetime import datetime, timezone

KEY = os.environ["DUTIFY_HUB_API_KEY"]

resp = requests.post(
    "https://dutify.ai/api/lens/chat",
    headers={"X-API-Key": KEY, "Content-Type": "application/json"},
    json={
        "lastMessage": {
            "user": "morning-digest",
            "datetime": datetime.now(timezone.utc).isoformat(),
            "content": "What action items came out of yesterday's calls? Group by call.",
        },
    },
    timeout=90,
)
resp.raise_for_status()
print(resp.text)   # the answer (often markdown)
```

## Pattern: multi-turn conversation

Pass the prior turns as `history` so Lens can resolve "the call we just discussed":

```python
history = []

def ask(question):
    global history
    payload = {
        "lastMessage": {
            "user": "scripted-bot",
            "datetime": datetime.now(timezone.utc).isoformat(),
            "content": question,
        },
        "history": history,   # full chronological transcript so far
    }
    resp = requests.post(
        "https://dutify.ai/api/lens/chat",
        headers={"X-API-Key": KEY, "Content-Type": "application/json"},
        json=payload, timeout=90,
    )
    resp.raise_for_status()
    answer = resp.text.strip().strip('"')   # response is a JSON-encoded string in some shapes
    history.append({"user": "scripted-bot", "datetime": payload["lastMessage"]["datetime"], "content": question})
    history.append({"user": "assistant", "datetime": datetime.now(timezone.utc).isoformat(), "content": answer})
    return answer

print(ask("Summarize the Acme renewal call"))
print(ask("What were the open questions?"))   # "the" call = the Acme call from history
```

**Don't accumulate more than 20 turns** — Hub's `@Size(max=20)` validator rejects bigger histories. Keep a sliding window.

## What Lens has access to (workspace-scoped)

Lens runs against the API key's bound workspace's data:

- All call recordings + transcripts the user has access to
- Workspace's custom prompt (if set — see [settings.md](settings.md))
- Connected integrations (read-only — Lens can list them but can't take action via the API-key path)
- Workspace metadata + members

It does NOT have access to:

- Other workspaces (workspace boundary is enforced at the data-source layer)
- Composio "act" capabilities (sending emails, creating Notion pages etc.) — those need the async endpoint that API keys can't reach
- The user's calendar (separate microservice with its own auth)

## When Lens can't answer

Common patterns:

- "I don't have access to that information" — usually means the data isn't in this workspace's recordings, or the user isn't a member of the source.
- "I couldn't find any recordings about X" — the search across transcripts came up empty. Try broader keywords or check whether the recording exists via `/usercall/search`.
- The response references "based on the recordings I can see" — Lens is being conservative when the data is sparse. Often worth asking a more specific follow-up.

## Why not use the async `/v1/lens/chat/query` endpoint?

The async endpoint is **not** in the API-key allowlist (`ApiKeyScopeFilter` denies anything under `/v1/lens/...`). It exists for the Slack and Teams bots which use a callback delivery flow. From an API-key script, the synchronous `/lens/chat` is the only Lens path you can hit — accept the synchronous wait and design around the 60-90s upper bound.
