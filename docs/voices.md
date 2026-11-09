> **Rendered documentation:** [docs.utpro.dev](https://docs.utpro.dev/uTPro.Feature.TextToSpeech/) is the canonical, always-current home of this page.

# Voices and pronunciation

Open **Settings → Advanced Settings → Text To Speech**. The workspace has three tabs: **Voices**, **Pronunciation** and **Audio cache**. They need Settings section access.

## Engines

The **Voices** tab starts with one card per engine:

| Engine | Ready when | Audio |
|---|---|---|
| Browser | Always | Played by the visitor's browser; nothing stored |
| OpenAiCompatible | `Endpoint`, `ApiKey` and `Model` are set | Generated on the server, cached |
| Piper | The add-on is installed, the binary exists and at least one model is in the models folder | Generated on the server, cached |
| Kokoro | The add-on is installed (and `ModelPath`, if set, exists) | Generated on the server, cached |

A card that is not ready explains why, for example "Piper binary not found at ...". The API key is never shown, only that one is set.

### Which engine to pick

| You want | Use |
|---|---|
| Zero setup, zero cost | Browser. Quality depends on the visitor: Edge has very good neural voices (including Vietnamese HoaiMy and NamMinh), Chrome is good, other browsers fall back to the operating system's voices. |
| The same good voice for everyone, free, offline, Vietnamese | Piper with `vi_VN-vais1000-medium` |
| The most natural free English | Kokoro |
| Broadcast quality, male and female voices in many languages | An OpenAI-compatible API (paid, bring your own key) |

## Voice per culture

**Add mapping** opens a form:

| Field | Example | Notes |
|---|---|---|
| Culture | `vi-VN`, `vi`, `*` | Most specific wins |
| Engine | `Piper` | Engines that are not ready are marked |
| Voice | `vi_VN-vais1000-medium` | Suggestions come from the engine. For Browser this is an optional name hint. |
| Speed | `1.0` | 0.5 - 2.0. Visitors can still change speed in the player. |
| Label | `Female - local` | Shown in the backoffice and the cache table |

Write a sentence in **Sample text** and press **Preview** to hear it. Server engines synthesize a short sample (up to 600 characters) without touching the cache; Browser previews in your own browser.

Mappings from `VoicesByCulture` in appsettings are listed read-only underneath. A mapping saved here for the same culture overrides them.

> If a mapped engine is not installed or not ready, pages fall back to the Browser voice. A misconfiguration never silences a language.

Changing a mapping changes the audio hash, so the next render of each page registers new audio and the old file becomes an orphan. Nothing is regenerated in bulk behind your back.

## Pronunciation

The **Pronunciation** tab holds find-and-replace rules applied to the text before **every** engine speaks it, including Browser.

| Field | Example |
|---|---|
| Culture | `vi` (or `*` for all) |
| Find | `TP.HCM` |
| Say instead | `Thành phố Hồ Chí Minh` |
| Order | `10` (ascending) |
| Regular expression | Off for plain text. On for patterns such as `\b(\d+)%` → `$1 phần trăm` |

Plain rules are case-sensitive exact matches. Regular expressions run with a 250 ms timeout, and an invalid or slow pattern is skipped instead of breaking the page.

Use **Try it** to see the text before and after the rules for a culture.

The rules that apply to a culture are fingerprinted into the audio hash. Editing a Vietnamese rule refreshes Vietnamese audio on the next render and leaves other languages alone.
