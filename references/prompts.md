# Custom prompts — workspace, series, occurrence, call

A custom prompt is extra instruction text handed to the LLM when it analyses a
meeting. There are four places to set one. They are **not** an override chain
at all — every level that is set is applied. Read the combination rules below
before assuming the most specific one silences the others.

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
- to **clear** it, `PUT` an **empty body** — not `""` and not `null`, both of
  which are stored verbatim as the prompt. `DELETE` on this route is a 405
- `GET` returns the prompt as **plain text**, not JSON. The class says
  `@Produces("application/json")` but the method returns a bare `String`, which
  is written verbatim — `"Summarise in bullet points"` comes back as 26 bytes of
  text, not a quoted JSON string. Read `r.text`; `.json()` raises on any prompt
  that isn't itself valid JSON
- three read shapes, none of them a 404: **200 + text** (set), **200 + empty
  body** (set once, then cleared), **204 No Content** (never set — the service
  returns Java `null` and JAX-RS turns that into a 204). The four-character
  token `null` is never on the wire

Sending `{"customPrompt": "…"}` here does not fail. It is accepted and the
literal text `{"customPrompt": "…"}` becomes the workspace prompt, which is then
prepended to every recording's analysis until someone notices. **Read the current
value before writing it** — there is no history, and the previous text is not
recoverable through the API.

## How the levels combine

Nothing overrides anything. At bot-launch the calendar side reads the occurrence's
own prompt **and** its series prompt, and applies **both**:

- both set → the call receives them together, labelled:

  ```
  series custom prompt: <series prompt>

  occurrence custom prompt: <occurrence prompt>
  ```

- only one set → that one, as-is, without a label
- neither → no call-specific prompt

The result is copied onto the resulting call as its call-specific prompt. Clearing
an occurrence prompt therefore never "falls back" to anything — the series prompt
was already applying; you have only removed the date-specific part. Never advise
clearing a series prompt to "avoid a conflict" with an occurrence prompt: there is
no conflict, and clearing it deletes instructions every other occurrence relies on.

The **workspace** prompt is the third input of the same set: a participant's version
of the summary, key points, action items and enhanced notes is generated from
their workspace prompt + their call-specific prompt (the series/occurrence text
above, or whatever they set on the call) + their own notes from the desktop app.
Nothing is aggregated across participants: two attendees with identical inputs
share one generated version, an attendee with none of them gets the default
version, and no attendee's prompt or notes ever shapes — or is visible in —
another attendee's version. A participant with only a workspace prompt is
therefore affected by the workspace prompt alone.

`GET /usercall/{id}` returns the caller's version in `recordings[0].summary`,
`keyPoints`, `actionItems`, `enhancedNotes`, and only the caller's `userNotes`.
`POST /recording/{id}/regenerate-summary` regenerates the caller's version from
their CURRENT inputs and touches nobody else's.

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
- **Assuming another participant's prompt changes what you see.** It does not:
  versions are per instruction set, and a call prompt only ever reaches the
  version of the user who set it.
- **Calling `/v1/calendar/events` without both dates.** 400, not an empty list.
- **Assuming an occurrence prompt replaces the series prompt for that date.** It
  does not — both are applied, labelled, in one call-specific prompt.
