> **Rendered documentation:** [docs.utpro.dev](https://docs.utpro.dev/uTPro.Feature.TextToSpeech/) is the canonical, always-current home of this page.

# API

## Public audio endpoint

Anonymous; used by the player.

| Method | Route | Result |
|---|---|---|
| `GET` | `/api/text-to-speech/audio/{hash}` | The audio (`200` / `206` with range). `304` when the ETag matches. `404` for an unknown or superseded hash. `503` with `Retry-After` while the audio is still being generated. |
| `GET` | `/api/text-to-speech/audio/{hash}?download=true` | The same file as an attachment. |

The hash is the only input and it must have been registered by one of your templates. The endpoint never accepts text, so it cannot be used to synthesize arbitrary speech.

## Management API

Authenticated backoffice endpoints under `/umbraco/management/api/v1/utpro/text-to-speech`. All are `POST` with a JSON body (the Management API redirects unknown `GET` routes, which would surface as a confusing 404). They appear in Swagger under **uTPro Text To Speech API** when Swashbuckle is present.

### Settings section access

| Route | Body | Result |
|---|---|---|
| `voices` | `{}` | Engine statuses (with voices), saved mappings, appsettings seeds, cache folder |
| `voice-save` | `{ "id": 0, "culture": "vi-VN", "provider": "Piper", "voiceId": "...", "speed": 1.0, "label": "..." }` | The saved mapping (one per culture) |
| `voice-delete` | `{ "id": 1 }` | `{ "removed": true }` |
| `preview` | `{ "text": "...", "provider": "Piper", "voiceId": "...", "speed": 1.0, "culture": "vi-VN" }` | Audio bytes; `400` with a message when the engine is not ready |
| `rules` | `{}` | Pronunciation rules |
| `rules-save` | `{ "id": 0, "culture": "vi", "pattern": "TP.HCM", "replacement": "...", "isRegex": false, "sortOrder": 10 }` | The saved rule; `400` for an invalid regex |
| `rules-delete` | `{ "id": 1 }` | `{ "removed": true }` |
| `rules-test` | `{ "culture": "vi-VN", "text": "..." }` | `{ "before", "after", "version" }` |
| `cache` | `{ "search": "", "filter": "all", "nodeKey": null, "fromDate": null, "toDate": null, "page": 1, "pageSize": 50 }` | `{ "items": [...], "total", "summary" }` |
| `regenerate` | `{ "hash": "..." }` | `{ "queued": 1 }` |
| `cache-bulk` | `{ "action": "delete" \| "regenerate", "hashes": [...] }` | `{ "affected": n }` (max 500 per call) |
| `clear-cache` | `{}` | Deletes every file and row: `{ "affected": n }` |
| `audio-file` | `{ "hash": "...", "inline": true }` | The file (inline or as an attachment) |

`filter` is `all`, `pending`, `ok`, `stale` or `orphan`.

### Content section access (Audio content app)

Under `.../utpro/text-to-speech/node`:

| Route | Body | Result |
|---|---|---|
| `status` | `{ "nodeKey": "guid" }` | `{ "items": [...] }` for that page |
| `regenerate` | `{ "nodeKey": "guid" }` | Queues every live row of the page: `{ "queued": n }` |
| `clear` | `{ "nodeKey": "guid" }` | Deletes the page's audio: `{ "affected": n }` |
| `audio-file` | `{ "hash": "...", "nodeKey": "guid" }` | The file, only if it belongs to that page |

### Cache item

```json
{
  "hash": "9f2c...", "nodeKey": "guid", "nodeId": 1234, "nodeName": "My article",
  "culture": "vi-VN", "slot": "", "provider": "Piper", "voiceId": "vi_VN-vais1000-medium",
  "voiceLabel": "Female - local", "speed": 1.0, "textSource": "Template",
  "textPreview": "First 160 characters...", "characters": 4210,
  "format": "wav", "sizeBytes": 1987654, "durationSeconds": 44.5,
  "status": "OK", "error": null, "createdUtc": "...", "generatedUtc": "..."
}
```

The full text is never returned.

## Services

Inject these from your own code:

| Interface | Use |
|---|---|
| `ITextToSpeechService` | `Prepare(PlayerRequest)` (what the ViewComponent calls), `EnsureGeneratedAsync(hash)`, `RegenerateAsync(hash)`, `PreviewAsync(...)`, `RefreshNode(key)` |
| `IAudioCacheRegistry` | Query and manage registry rows |
| `IAudioAdminService` | Regenerate / delete / clear with file clean-up |
| `ISpeechProviderResolver` | `Resolve(culture, override)` and engine statuses |
| `IPronunciationRuleEngine` | Apply rules to a text |
| `ISpeechAudioCache` | Resolve file paths inside the cache root |
