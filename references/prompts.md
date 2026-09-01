# Custom prompts — workspace, series, occurrence, call

A custom prompt is extra instruction text handed to the LLM when it analyses a
meeting. There are four places to set one. They are **not** a single override
chain — read the precedence rules below before assuming the most specific one
silences the others.

## The four levels

| Level | Endpoint | Scope |
|---|---|---|
| Workspace | `GET`/`PUT /api/v1/workspaces/{workspaceId}/settings/custom-prompt` | `workspace-settings:read` / `:write` |
| Series (recurring meeting) | `PUT`/`DELETE /api/v1/calendar/series/{seriesMasterId}/custom-prompt` | `recordings:write` |
| Occurrence (one date) | `PUT`/`DELETE /api/v1/calendar/events/{eventId}/custom-prompt` | `recordings:write` |
| Call (already recorded) | `PUT`/`DELETE /api/usercall/{id}/custom-prompt` | `recordings:write` |

Body for the last three: `{"customPrompt": "…"}`, and `DELETE` clears without a
body.

**The workspace one behaves differently in both directions and there is no
`DELETE` on it:**

```
PUT /api/v1/workspaces/{workspaceId}/settings/custom-prompt
Content-Type: text/plain

Summarise in bullet points and flag budget risks.
```

- the body is a **bare string**, not `{"customPrompt": …}`
- to **clear** it, `PUT` an **empty string** — `DELETE` on this route is a 405
- `GET` returns JSON `null` when a prompt was never set, and an empty body after
  one has been cleared. Both mean "no prompt configured" — neither is a 404

Sending `{"customPrompt": "…"}` here does not fail. It is accepted and the
literal text `{"customPrompt": "…"}` becomes the workspace prompt, which is then
prepended to every recording's analysis until someone notices. **Read the current
value before writing it** — there is no history, and the previous text is not
recoverable through the API.

## Precedence

At bot-launch the calendar side resolves **occurrence → series**: an event's own
prompt wins, and only if it has none does the series prompt apply. The winner is
copied onto the resulting call as its call-specific prompt.

The **workspace** prompt applies to every meeting *in addition* to whatever
call-specific prompt exists. It is not overridden by the more specific levels —
think "always-on house style" plus "instructions for this meeting".

Call-specific prompts **aggregate across participants**: if three attendees each
set a different prompt on their own copy of the same call, the analysis receives
all three distinct prompts. Setting yours adds your voice; it does not silence
anyone else's.

## Finding the ids

`GET /api/v1/calendar/events?startDate=…&endDate=…` (scope `calendar:read`) lists
the caller's events. Both dates are **required** ISO-8601 timestamps — omitting
either is a 400, not an empty list. Each event carries:

- `eventId` — for the occurrence-level prompt
- `seriesMasterId` — for the series-level prompt (null on a one-off meeting)
- `eventCustomPrompt` / `seriesCustomPrompt` — whatever is already set, so you can
  read back what you wrote
- `subject`, `startDateTime`, `endDateTime`, `isSubscribed`, `callId`, `workspaceId`

For the call level, use `GET /api/usercall/all` or `/api/usercall/search` and take
the outer integer `id`. The prompt comes back as `callSpecificCustomPrompt` on the
`UserCallDTO`.

These calendar routes are **actor-scoped, not workspace-scoped**: they act on the
calendar of the user who owns the key, which is why they live at
`/v1/calendar/...` rather than under `/v1/workspaces/{id}/...`, and why the
bound-workspace path rule does not apply to them.

## When each one takes effect

Series and occurrence prompts are **pre-meeting configuration**: they reach the
model the next time that meeting is recorded and processed.

A call-level prompt is **stored only**. Setting it on a call that has already been
analysed changes nothing by itself — trigger the re-run explicitly:

```
PUT  /api/usercall/{id}/custom-prompt   {"customPrompt": "Summarise as a decision log"}
POST /api/recording/{recordingId}/regenerate-summary   {"saveToDb": true}
```

Prefer `regenerate-summary` (cheap, reuses the transcript) over `/reprocess`
(re-transcribes, slow and expensive) unless the transcript itself is the problem.

## Setting a series prompt creates its stored row

The series prompt lives on a per-(series, user) row that nothing in the subscribe
path creates — historically only assigning a workspace inserted one, so a series
you had subscribed to but never given a workspace answered `404 Series not found`
forever. Setting a series prompt now creates that row when it is missing.

The row needs a workspace. It is resolved as: the **API key's bound workspace**,
falling back to the **actor's own default workspace**. You may name one explicitly:

```json
PUT /api/v1/calendar/series/{seriesMasterId}/custom-prompt
{"customPrompt": "Track blockers", "workspaceId": "…"}
```

`workspaceId` is optional and only consulted when the row must be created. It may
be either the key's bound workspace or the actor's own default — **either one,
even when they differ**. Any other workspace is refused with 403.

Two cases still answer 404: a series with no synced occurrences, and one where
neither a key workspace nor a default workspace resolves — a row with no workspace
would only move the problem into the data.

`DELETE` (clearing) also goes through this path, so clearing a prompt on a series
that has no row creates an empty row. Harmless, but worth knowing if you are
counting rows.

## The origin service

These calendar routes are a proxy. The service that owns the data,
`calendar-events-api` at `https://calendar.dutify.ai`, accepts the same
`dh_live_…` key directly (as `X-API-Key` or `Authorization: Bearer`), with the
same scopes, on slightly different paths:

```
POST /api/calendar/event-custom-prompt    {"eventId": "…", "customPrompt": "…"}
POST /api/calendar/series-custom-prompt   {"seriesMasterId": "…", "customPrompt": "…", "workspaceId": "…"}
GET  /api/calendar/events?startDate=…&endDate=…
```

Prefer the Hub paths — one host, one key surface, and they are in the catalog.
The direct host is documented here because the web UI uses it and it stays
available.

## Footguns

- **Wrapping the workspace prompt in an object.** `/settings/custom-prompt` takes a
  bare string; the three newer endpoints take `{"customPrompt": …}`. They differ.
- **Expecting a per-call prompt to re-analyse the call.** It does not. Follow it
  with `regenerate-summary`.
- **Assuming the most specific prompt is the only one applied.** The workspace
  prompt is always in play as well.
- **Calling `/v1/calendar/events` without both dates.** 400, not an empty list.
- **Expecting a series prompt to reach occurrences that carry their own.** An
  occurrence-level prompt wins for that date.
