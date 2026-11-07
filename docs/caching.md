> **Rendered documentation:** [docs.utpro.dev](https://docs.utpro.dev/uTPro.Feature.TextToSpeech/) is the canonical, always-current home of this page.

# Caching and generation

Server audio is made once and reused. This page explains when it is made, where it lives and how it is cleaned up.

## Identity

Each audio file is identified by a SHA-256 hash of:

- the page key, culture and player slot
- the normalised text (after pronunciation rules)
- the engine, voice id and speed
- a fingerprint of the rules that applied

Change any of these and you get a new hash, so the cache can never serve stale audio for changed text. Identical text on two different pages gets two rows, so deleting one page's audio never breaks the other.

## When audio is generated

| Trigger | What happens |
|---|---|
| A page with a server voice is rendered and its hash is new | The hash is registered as **Pending** and queued. The page does not wait. |
| A visitor presses play before the queue got to it | The audio endpoint generates it now (single-flight) and streams it. |
| A page is published (property-sourced audio) | The text is re-read from the published page; if it changed, new audio is queued when `AutoGenerateOnPublish` is on. |
| **Regenerate** in the backoffice | The row is reset to Pending and queued. |
| The site restarts with Pending rows | They are re-queued about 45 seconds after start-up. |

A page render never calls an engine. Piper takes seconds and Kokoro longer for an article; doing that inside the render would block every first view.

**Single-flight:** however many visitors (and the queue) ask for the same hash at once, one synthesis runs and everyone shares the result. `Queue:MaxParallelism` (default 1) caps how many different syntheses run at the same time.

After a failed generation the error is stored on the row, retries back off (30 seconds for the endpoint, 5 minutes for re-registration), and the player keeps showing "Preparing audio..." while it retries.

## Where files live

```
{ContentRoot}/umbraco/Data/uTProTextToSpeechCache/{pageKey}/{hash}.wav|mp3
```

Override with `CachePath`. The folder is outside `wwwroot`, so files are served only through the audio endpoint. Files are written to a temporary name and moved into place, so a reader never sees half a file.

## Serving

`GET /api/text-to-speech/audio/{hash}` streams the file with:

- `Cache-Control: public, max-age=86400` (safe: a content change produces a new URL)
- `ETag` and `Last-Modified`, so repeat requests get `304 Not Modified`
- HTTP range support (`206 Partial Content`), so seeking works and nothing is buffered in memory

Add `?download=true` for a download with a readable file name.

## Lifecycle

| Status | Set when | Served? |
|---|---|---|
| **Pending** | Registered, not generated yet | Generated on first request |
| **OK** | Generated | Yes |
| **Stale** | The page was published after a template-sourced file was made | Yes, until the next render replaces it |
| **Orphan** | A newer hash replaced it for the same page, culture and slot; or the page was unpublished, trashed or deleted | Yes while the file exists, never regenerated |

Why Stale? For template-sourced text the server cannot know the new text until your template renders the page again, so it keeps the old audio playable and lets the next render decide. If the text did not change, the row goes back to OK.

Orphans are deleted, file and row, by a background job every 6 hours once they are older than `OrphanRetentionDays` (default 7). You can also delete them from the **Audio cache** tab.

## Disk use

WAV is about 2.5 - 3.5 MB per minute of speech (22 - 24 kHz mono). An API voice with `ResponseFormat: "mp3"` is roughly ten times smaller. The **Audio cache** tab shows the total.

## Load-balanced hosting

Point `CachePath` at a shared folder so every server sees the same files. Each server runs its own queue; single-flight is per server, so two servers can occasionally generate the same new file at the same moment. The result is identical and the second write simply replaces the first.
