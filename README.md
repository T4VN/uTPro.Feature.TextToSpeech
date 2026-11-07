# uTPro.Feature.TextToSpeech

**Documentation:** <https://docs.utpro.dev/uTPro.Feature.TextToSpeech/>

Add a **"listen to this article"** player to any Umbraco 16, 17 or 18 page with one line in a template. You pass the text to read, visitors get an accessible audio player (play/pause, speed, progress, resume, download), and you choose the voice from a backoffice workspace. It is **free and offline by default**: the Browser voice needs no API key, no files and no outbound traffic. When you want consistent, higher-quality audio, map a culture to a **server voice** (Piper or Kokoro, fully local, or any OpenAI-compatible speech API). Server audio is generated in the background, cached on disk by content hash and reused, so a page render never waits for a synthesizer.

Part of the uTPro feature family (alongside Simple Cart, Simple Form Builder, SEO Audit and others).

[![NuGet](https://img.shields.io/nuget/v/uTPro.Feature.TextToSpeech.svg)](https://www.nuget.org/packages/uTPro.Feature.TextToSpeech)
[![NuGet Downloads](https://img.shields.io/nuget/dt/uTPro.Feature.TextToSpeech.svg)](https://www.nuget.org/packages/uTPro.Feature.TextToSpeech)
[![Umbraco Marketplace](https://img.shields.io/badge/Umbraco-Marketplace-blue)](https://marketplace.umbraco.com/package/utpro.feature.texttospeech)

## Which package do I need?

Start with the core package. Add an offline engine only when you want a server voice.

| You want | Install | Cost |
|---|---|---|
| Read articles aloud, zero setup | `uTPro.Feature.TextToSpeech` only (Browser voice) | Free |
| One consistent offline voice for everyone, incl. Vietnamese | core + `uTPro.Feature.TextToSpeech.Piper` | Free |
| The most natural offline English | core + `uTPro.Feature.TextToSpeech.Kokoro` | Free |
| Broadcast quality, many languages | core + an OpenAI-compatible API key (no add-on) | Paid API |

The core works on its own: the Browser voice and the OpenAI-compatible provider are both built in. Piper and Kokoro are optional add-ons for local server voices. You install an add-on **instead of**, not before, the core: each add-on depends on the core package, so `dotnet add package uTPro.Feature.TextToSpeech.Piper` pulls in `uTPro.Feature.TextToSpeech` automatically at the same version. If a mapped engine is missing or misconfigured, pages fall back to the Browser voice, so a culture is never silent.

## Install

```bash
dotnet add package uTPro.Feature.TextToSpeech
```

That's it. The package **self-wires** its services, migration and backoffice UI through an Umbraco composer, so there are **no changes to your `Program.cs`**. Requires Umbraco 16, 17 or 18 (net9.0 / net10.0).

Optional offline voices are separate packages, so the core stays small:

```bash
dotnet add package uTPro.Feature.TextToSpeech.Piper    # fast, local, includes Vietnamese
dotnet add package uTPro.Feature.TextToSpeech.Kokoro   # natural English and a few other languages, local
```

## Use it

In any template, pass the text you want read:

```cshtml
@await Component.InvokeAsync("TextToSpeechPlayer", new { content = Model, text = bodyHtml })
```

`content` is the page (for its culture and key), `text` is what to read (HTML is allowed and stripped). Leave `text` out and the player falls back to the first non-empty property in `Rendering.TextPropertyAliases` (`bodyText`, `content` by default).

The template decides what is read because article bodies are usually Block List / Block Grid, and only your template knows which blocks are prose.

## Voices

| Engine | Where it runs | Needs | Languages | Audio file |
|---|---|---|---|---|
| **Browser** (default) | Visitor's browser (Web Speech API) | Nothing | Whatever the visitor's browser offers; Edge has neural Vietnamese voices | None |
| **Piper** (add-on) | Your server, offline | Piper binary + `.onnx` voice files | ~40 locales, including `vi_VN` | Cached WAV |
| **Kokoro** (add-on) | Your server, offline | ~300 MB model (auto-download or file) | English, Spanish, French, Hindi, Italian, Japanese, Chinese, Portuguese | Cached WAV |
| **OpenAI-compatible API** | Your provider | Endpoint + API key | Provider's | Cached MP3/WAV |

Pick the voice per culture in **Settings → Advanced Settings → Text To Speech → Voices**. If a mapped engine is missing or not configured, the page falls back to the Browser voice, so a language is never silent.

## How server audio works

1. The template renders the player. The package normalises the text, applies your pronunciation rules and computes a content hash.
2. It registers that hash in a database table and queues generation in the background. **The page does not wait.**
3. The player streams `/api/text-to-speech/audio/{hash}`. The file is generated once, stored under `umbraco/Data/uTProTextToSpeechCache`, and served with ETag and range support.
4. Publishing a page marks its audio stale; the next render produces new audio and the old file is purged after `OrphanRetentionDays` (7).

## Backoffice

Under **Settings → Advanced Settings → Text To Speech**:

- **Voices** - engine status cards, a voice per culture, preview.
- **Pronunciation** - replacement rules ("TP.HCM" becomes "Thành phố Hồ Chí Minh") with a before/after test box.
- **Audio cache** - every generated file with search, filters, play, download, regenerate and delete.

Every document also gets an **Audio** content app showing the audio of that page.

## Configuration

Everything has a working default. Settings live under `uTPro:Feature:TextToSpeech`:

```json
{
  "uTPro": {
    "Feature": {
      "TextToSpeech": {
        "Provider": "Auto",
        "OpenAi": { "Endpoint": "https://api.openai.com", "ApiKey": "", "Model": "gpt-4o-mini-tts" },
        "Piper": { "ExecutablePath": "", "ModelsRoot": "" },
        "Kokoro": { "ModelPath": "", "WarmUp": true }
      }
    }
  }
}
```

Keep `ApiKey` in user-secrets or environment variables, not in a committed file. The full reference is in the [documentation](https://docs.utpro.dev/uTPro.Feature.TextToSpeech/configuration/).

## Security

- The public audio endpoint accepts **only a hash your own template registered**. Visitors cannot send text, so the site cannot be used as a free text-to-speech service.
- Calls to the speech API are **SSRF-guarded** at connect time: loopback, private, link-local and reserved addresses are refused unless you opt in with `OpenAi:AllowLocalEndpoint`.
- The API key is never returned by any endpoint; the backoffice shows only that one is set.
- Piper is started with an argument list (never a shell), the voice id must be a bare file name inside the models folder, and every run has a timeout.
- Backoffice endpoints are authenticated; the Voices, Pronunciation and Audio cache tabs need Settings access, the Audio content app needs Content access.

## Building / packing

```bash
dotnet build uTPro.Feature.TextToSpeech.sln -c Release
dotnet test tests/uTPro.Feature.TextToSpeech.Tests/uTPro.Feature.TextToSpeech.Tests.csproj -c Release
pwsh ./pack.ps1
```

`pack.ps1` wipes `bin`/`obj`, packs the core and both add-ons into `Build/`, evicts the version from the local NuGet cache and mirrors the docs to the public repo.

## Test site

`src/uTPro.Feature.TextToSpeech.TestSite` is a minimal Umbraco 18 host (SQLite, unattended install as `admin@example.com`). Run it yourself with `dotnet run --project src/uTPro.Feature.TextToSpeech.TestSite`, create a document type with a `bodyText` rich text property and a template that calls the player, then publish a page.

## License

See [LICENSE.txt](LICENSE.txt). Free to use in your own Umbraco sites, including commercial ones.
