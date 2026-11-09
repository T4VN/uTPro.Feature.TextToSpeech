> **Rendered documentation:** [docs.utpro.dev](https://docs.utpro.dev/uTPro.Feature.TextToSpeech/) is the canonical, always-current home of this page.

# Getting started

## Which package do I need?

Start with the core package. Add an offline engine only when you want a server voice.

| You want | Install | Cost |
|---|---|---|
| Read articles aloud, zero setup | `uTPro.Feature.TextToSpeech` only (Browser voice) | Free |
| One consistent offline voice for everyone, incl. Vietnamese | core + `uTPro.Feature.TextToSpeech.Piper` | Free |
| The most natural offline English | core + `uTPro.Feature.TextToSpeech.Kokoro` | Free |
| Broadcast quality, many languages | core + an OpenAI-compatible API key (no add-on) | Paid API |

> The Browser voice and the OpenAI-compatible provider are built into the core, so the core works on its own. [Piper and Kokoro](offline-voices.md) are optional add-ons. Each add-on depends on the core, so installing `uTPro.Feature.TextToSpeech.Piper` pulls in `uTPro.Feature.TextToSpeech` automatically at the same version - you never install the core separately first.

## 1. Install

```bash
dotnet add package uTPro.Feature.TextToSpeech
```

No `Program.cs` changes are required. The package self-wires its services, migration, background workers and backoffice UI through an Umbraco composer, and uninstalling it leaves nothing running. (Generated audio and the three `uTProTextToSpeech*` tables stay behind as inert data; delete them if you want a fully clean removal.)

Requires Umbraco 16, 17 or 18.

## 2. First boot

On the first start the package creates its tables with an idempotent Umbraco migration (SQL Server, SQLite and PostgreSQL are supported):

| Table | Holds |
|---|---|
| `uTProTextToSpeechAudio` | One row per generated or pending audio file |
| `uTProTextToSpeechRule` | Pronunciation rules |
| `uTProTextToSpeechVoice` | Voice mappings saved in the backoffice |

Open **Settings → Advanced Settings → Text To Speech**. You should see the **Voices** tab with a Browser card marked *Ready* and an OpenAI-compatible card marked *Not configured*.

## 3. Add the player to a template

Pass the text you want read:

```cshtml
@using Umbraco.Cms.Web.Common.PublishedModels
@inherits Umbraco.Cms.Web.Common.Views.UmbracoViewPage<Article>
@{
    var body = Model.BodyText?.ToString();   // rich text renders as HTML; the package strips the tags
}

<h1>@Model.Name</h1>

@await Component.InvokeAsync("TextToSpeechPlayer", new { content = Model, text = Model.Name + ". " + body })

@Model.BodyText
```

Publish a page and open it. The player appears, and **Play** reads the article with the visitor's browser voice. That is the whole zero-configuration setup: no keys, no files, no outbound traffic.

If you leave `text` out, the player reads the first non-empty property listed in `Rendering.TextPropertyAliases` (`bodyText`, `content` by default). See [Rendering](rendering.md) for Block List / Block Grid pages and the other parameters.

## 4. Use a server voice (optional)

Server voices are generated once, cached, and sound the same for every visitor. Pick one:

- **Offline, free:** install the [Piper or Kokoro add-on](offline-voices.md).
- **AI API:** configure an [OpenAI-compatible endpoint](ai-api.md).

Then map a culture to it in **Voices → Add mapping**:

| Culture | Engine | Voice |
|---|---|---|
| `vi-VN` | Piper | `vi_VN-vais1000-medium` |
| `en` | Kokoro | `af_heart` |
| `*` | OpenAiCompatible | `alloy` |

Use **Preview** to hear a sample before you save. Reload a published page: the player now points at `/api/text-to-speech/audio/{hash}`, and the first **Play** may show "Preparing audio..." for a few seconds while the file is generated in the background. Every later play is served straight from the cache.

## 5. See what was generated

Open the **Audio cache** tab. Each row shows the page, culture, voice, size and a status:

| Status | Meaning |
|---|---|
| OK | The file exists and matches the page text |
| Pending | Registered, not generated yet (or the last attempt failed - the error is shown) |
| Stale | The page was published after this audio was made; the next render replaces it |
| Orphan | Superseded or the page was removed; purged after `OrphanRetentionDays` |

See [Caching](caching.md) for the details, and [Configuration](configuration.md) for every option.
