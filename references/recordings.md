# recordings.md — Recording-level operations

A **Recording** is the underlying media + transcript for a UserCall (see [calls.md](calls.md) for the metadata layer above it). Recording IDs are **integers** (`Long`); they're surfaced as `recordings[].id` on `GET /usercall/{id}` responses.

Two scopes:
- `recordings:read` — progress + signed URLs to download the media
- `recordings:write` — reprocess + regenerate-summary

## Read endpoints

### Processing progress

```
GET /api/recording/{recordingId}/progress
```

Returns a bare integer 0-100. Useful to poll after `reprocess` or while waiting for a freshly-uploaded recording to be analysed. There's no terminal "done" sentinel; 100 means done.

### Signed audio URL

```
GET /api/recording/{recordingId}/audio-url?expiryMinutes=30
```

Returns a JSON-quoted string — the full Azure Blob SAS URL to download the audio (typically MP3 / WAV / M4A depending on source). The URL is valid for `expiryMinutes` minutes (default 30; Hub may cap it). **Don't cache** — request a fresh URL each time you need to download.

```python
url = requests.get(
    f"https://dutify.ai/api/recording/{rid}/audio-url?expiryMinutes=15",
    headers={"X-API-Key": key},
).json()   # bare quoted string — a vendor-blob SAS URL ending in ?sv=…&sig=…

# Then download with no auth (the SAS URL is the auth):
audio = requests.get(url).content
```

### Signed media URL (video + audio)

```
GET /api/recording/{recordingId}/media-url?expiryMinutes=30
```

Same shape as `/audio-url` but returns the FULL recording (video track when present, falls back to audio-only). Use this when the user wants to "see" the call, not just hear it.

### Signed preview URL

```
GET /api/recording/{recordingId}/preview-url?expiryMinutes=30
```

Same shape; returns a thumbnail / short looping clip suitable for embedding as a card preview.

## Write endpoints

### Re-generate summary only

```
POST /api/recording/{recordingId}/regenerate-summary
Content-Type: application/json

{"saveToDb": true}
```

Re-runs summary generation against the **existing transcript** without re-transcribing. Cheaper + faster than full reprocess. Body `saveToDb` (default `true`) — when `false`, returns the new summary without persisting it (preview mode, useful for "show me what a fresh summary would look like" UX).

Returns the new summary text wrapped in a `ServiceResult` envelope. Read `data` (the summary) when `success: true`.

### Full reprocess

```
POST /api/recording/{recordingId}/reprocess
Content-Type: application/json

{"regenerate": true}
```

Re-runs the **whole pipeline**: transcription → summary → key points → action items. Slow (minutes) and expensive. Use when:

- A different language was detected on first pass and the transcript is wrong
- Speaker diarization needs re-running because two participants got merged
- Workspace's custom prompt was updated and the user wants existing summaries to reflect it

Returns the updated `RecordingDTO` synchronously, but the actual processing is async — use `/progress` to track completion if you need to know when it's done.

`regenerate` (boolean) — when true, also regenerates downstream artifacts (key points, action items) after the new transcription. Backend default is `true`.

## Patterns

### Get audio for the most recent Acme call

```python
calls = requests.get("https://dutify.ai/api/usercall/search?query=Acme",
                     headers={"X-API-Key": key}).json()
call = calls[0]
recording_id = call["recordings"][0]["id"]   # integer

audio_url = requests.get(
    f"https://dutify.ai/api/recording/{recording_id}/audio-url",
    headers={"X-API-Key": key},
).json()

with open("acme.mp3", "wb") as f:
    f.write(requests.get(audio_url).content)
```

### Poll until reprocessing is done

```python
import time

requests.post(
    f"https://dutify.ai/api/recording/{rid}/reprocess",
    headers={"X-API-Key": key, "Content-Type": "application/json"},
    json={"regenerate": True},
)

while True:
    progress = requests.get(
        f"https://dutify.ai/api/recording/{rid}/progress",
        headers={"X-API-Key": key},
    ).json()
    if progress >= 100:
        break
    time.sleep(15)

# Re-fetch the call to see the updated summary
updated = requests.get(
    f"https://dutify.ai/api/usercall/{call_id}",
    headers={"X-API-Key": key},
).json()
print(updated["summary"])
```

## Gotchas

- The `/audio-url` etc. responses are **JSON-encoded strings** (quoted), not raw URLs. `response.json()` gives you the URL; `response.text` gives you `"https://..."` with the surrounding quotes still in.
- A recording ID for a call that's been deleted returns 404 — even if the underlying blob still exists momentarily.
- Reprocess is workspace-billable; don't loop on it. If `/progress` doesn't reach 100 in 10 minutes, surface as "reprocessing taking longer than expected" and stop polling.
- The signed URLs are tied to Azure Blob Storage. If they 403 on download, the SAS token already expired — request a fresh URL with a longer `expiryMinutes` (or refetch right before the download).
