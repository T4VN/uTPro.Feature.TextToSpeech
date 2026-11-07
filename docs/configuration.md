> **Rendered documentation:** [docs.utpro.dev](https://docs.utpro.dev/uTPro.Feature.TextToSpeech/) is the canonical, always-current home of this page.

# Configuration

Every setting has a working default. With no configuration at all the package uses the Browser voice and stores nothing.

Settings live under `uTPro:Feature:TextToSpeech`:

```json
{
  "uTPro": {
    "Feature": {
      "TextToSpeech": {
        "Enabled": true,
        "Provider": "Auto",
        "DefaultSpeed": 1.0,
        "CachePath": "",
        "OrphanRetentionDays": 7,
        "MaxTextLength": 30000,
        "VoicesByCulture": [
          { "Culture": "vi-VN", "Provider": "Piper", "VoiceId": "vi_VN-vais1000-medium", "Speed": 1.0, "Label": "Female - Vais1000 (local)" },
          { "Culture": "en", "Provider": "Kokoro", "VoiceId": "af_heart", "Speed": 1.0, "Label": "Female - Heart (local)" }
        ],
        "OpenAi": {
          "Endpoint": "",
          "ApiKey": "",
          "Model": "gpt-4o-mini-tts",
          "DefaultVoice": "alloy",
          "ResponseFormat": "mp3",
          "AllowLocalEndpoint": false,
          "TimeoutSeconds": 60
        },
        "Rendering": {
          "TextPropertyAliases": [ "bodyText", "content" ],
          "VoicePropertyAlias": "ttsVoice",
          "PlayerStyle": "inline",
          "AutoGenerateOnPublish": true
        },
        "Queue": {
          "MaxParallelism": 1,
          "SynthesisTimeoutSeconds": 60
        },
        "Piper": { "ExecutablePath": "", "ModelsRoot": "", "TimeoutSeconds": 120 },
        "Kokoro": { "ModelPath": "", "VoicesPath": "", "WarmUp": true, "DefaultVoice": "af_heart" }
      }
    }
  }
}
```

## General

| Option | Default | Description |
|---|---|---|
| `Enabled` | `true` | Master switch. When `false` the player renders nothing. |
| `Provider` | `Auto` | `Auto`: use a mapped server voice when its engine is ready, otherwise Browser. `Browser`: never generate server audio (mappings are ignored). `Server`: same as `Auto`; Browser stays the fallback so a language is never silent. |
| `DefaultSpeed` | `1.0` | Speed used when a mapping has none. Clamped to 0.5 - 2.0. |
| `CachePath` | *(empty)* | Audio folder. Empty = `{ContentRoot}/umbraco/Data/uTProTextToSpeechCache`. Relative values resolve against the content root; absolute values let load-balanced servers share one folder. |
| `OrphanRetentionDays` | `7` | Superseded audio is deleted after this many days. `0` purges on the next pass (every 6 hours). |
| `MaxTextLength` | `30000` | Characters per player. Longer text is cut at a sentence boundary. Caps the cost of a paid API and the time of a local engine. |
| `VoicesByCulture` | *(empty)* | Seed mappings. A mapping saved in the backoffice for the same culture wins. |

## Voice mappings

A mapping has `Culture`, `Provider`, `VoiceId`, `Speed` and an optional `Label`.

- `Culture` is a full culture (`vi-VN`), a bare language (`vi`, matches `vi-VN`) or `*` (anything). The most specific match wins, and backoffice mappings beat appsettings seeds at the same level.
- `Provider` is `Browser`, `OpenAiCompatible`, `Piper`, `Kokoro`, or the `Name` of a [custom engine](extensibility.md).
- For `Browser`, `VoiceId` is an optional name hint the player looks for (for example `HoaiMy` or `NamMinh` on Edge).

## OpenAI-compatible API

See [AI API voices](ai-api.md).

| Option | Default | Description |
|---|---|---|
| `OpenAi:Endpoint` | *(empty)* | Base URL, for example `https://api.openai.com`. `/v1/audio/speech` is appended (once). Empty = inactive. |
| `OpenAi:ApiKey` | *(empty)* | Sent as a Bearer token. Store it in user-secrets or an environment variable. |
| `OpenAi:Model` | `gpt-4o-mini-tts` | Model name sent in the request. |
| `OpenAi:DefaultVoice` | `alloy` | Voice used when a mapping has no `VoiceId`. |
| `OpenAi:ResponseFormat` | `mp3` | `mp3` or `wav`. Long text is split into requests and joined, which needs one of these two. |
| `OpenAi:AllowLocalEndpoint` | `false` | Allow loopback / private addresses (self-hosted gateway). Off by default for SSRF safety. |
| `OpenAi:TimeoutSeconds` | `60` | Per-request timeout. |

## Rendering

| Option | Default | Description |
|---|---|---|
| `Rendering:TextPropertyAliases` | `["bodyText","content"]` | Used only when the template passes no `text`. First non-empty string or rich-text property wins. |
| `Rendering:VoicePropertyAlias` | `ttsVoice` | Optional per-page override property holding `Provider:VoiceId`. |
| `Rendering:PlayerStyle` | `inline` | `inline` or `sticky` (pinned to the bottom of the viewport). |
| `Rendering:AutoGenerateOnPublish` | `true` | Regenerate property-sourced audio in the background on publish. Template-sourced audio is always refreshed by the next render. |

## Queue

| Option | Default | Description |
|---|---|---|
| `Queue:MaxParallelism` | `1` | Concurrent syntheses. Keep `1` on small servers so a bulk publish cannot starve the site. Read at start-up. |
| `Queue:SynthesisTimeoutSeconds` | `60` | How long the audio endpoint waits for an on-demand synthesis before answering `503` (the player retries). |

## Piper add-on

| Option | Default | Description |
|---|---|---|
| `Piper:ExecutablePath` | *(empty)* | Piper binary. Empty = `{ContentRoot}/umbraco/Data/uTProTextToSpeechPiper/piper/piper` (`piper.exe` on Windows). |
| `Piper:ModelsRoot` | *(empty)* | Folder of `*.onnx` + `*.onnx.json` voices. Empty = `{ContentRoot}/umbraco/Data/uTProTextToSpeechPiper/models`. |
| `Piper:TimeoutSeconds` | `120` | The process is killed after this. |

## Kokoro add-on

| Option | Default | Description |
|---|---|---|
| `Kokoro:ModelPath` | *(empty)* | `.onnx` model file. Empty = KokoroSharp downloads the default model on first use (about 300 MB). |
| `Kokoro:VoicesPath` | *(empty)* | Folder of `*.npy` voices. Empty = `{app}/voices`, where the package copies them on build and publish. |
| `Kokoro:WarmUp` | `true` | Load the model 30 s after start-up, but only when a Kokoro voice is mapped. |
| `Kokoro:DefaultVoice` | `af_heart` | Voice used when a mapping has no `VoiceId`. |

> Options are read live (`IOptionsMonitor`) except `Queue:MaxParallelism`, which needs a restart.
