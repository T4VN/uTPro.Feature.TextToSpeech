> **Rendered documentation:** [docs.utpro.dev](https://docs.utpro.dev/uTPro.Feature.TextToSpeech/) is the canonical, always-current home of this page.

# Rendering the player

The player is a ViewComponent. Use it in any view, partial or layout:

```cshtml
@await Component.InvokeAsync("TextToSpeechPlayer", new { content = Model, text = bodyHtml })
```

## Parameters

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `content` | `IPublishedContent` | Yes | The page. Provides the culture, the node key (cache grouping and backoffice links) and the per-node voice override. |
| `text` | `string` | No | What to read. HTML is allowed and stripped. Wins over `TextPropertyAliases`. |
| `voiceId` | `string` | No | Override as `Provider:VoiceId`, for example `Piper:vi_VN-vais1000-medium`. |
| `culture` | `string` | No | Override the culture. Defaults to the request culture. |
| `slot` | `string` | No | Name for a second player on the same page (see below). |

## Why the template passes the text

Article bodies are usually Block List or Block Grid. A block can be prose, a call to action, a caption or a teaser, and only your template knows which is which. So the template builds exactly what a reader sees and hands it over. For plain rich-text sites you can skip `text` and rely on `Rendering.TextPropertyAliases`.

### Block List example

```cshtml
@using System.Text
@{
    var sb = new StringBuilder();
    sb.Append(Model.Name).Append(". ");
    foreach (var block in Model.Blocks ?? [])
    {
        // Read only the blocks that are prose. RichTextBlock is your own ModelsBuilder element type.
        if (block.Content is RichTextBlock rich)
        {
            sb.Append(rich.Text).Append(' ');
        }
    }
}
@await Component.InvokeAsync("TextToSpeechPlayer", new { content = Model, text = sb.ToString() })
```

Headings and list items get a pause: block-level tags become sentence breaks, and a line with no closing punctuation gets a full stop.

## What happens during the render

Rendering never calls a speech engine.

1. The text is normalised (tags, scripts and styles removed, entities decoded, whitespace collapsed) and cut at `MaxTextLength` on a sentence boundary.
2. Your [pronunciation rules](voices.md#pronunciation) are applied.
3. The voice is resolved: the `voiceId` parameter, then the page's override property, then the best culture mapping, then Browser.
4. **Browser voice:** the player is emitted with the text in a `data-text` attribute. Nothing is stored.
5. **Server voice:** a hash of the text, voice, speed, rules, page and culture is computed. The package registers it (a database write only the first time), queues generation if the file is missing, and emits a player that streams `/api/text-to-speech/audio/{hash}`.

If the database is not ready (for example the migration has not run yet) the render falls back to the Browser voice instead of failing the page.

## Several players on one page

Audio is tracked per page, culture and *slot*. A second player with different text needs its own slot, otherwise each render would replace the other's audio:

```cshtml
@await Component.InvokeAsync("TextToSpeechPlayer", new { content = Model, text = intro, slot = "intro" })
@await Component.InvokeAsync("TextToSpeechPlayer", new { content = Model, text = body })
```

Only one player plays at a time; starting another pauses the first.

## Per-page voice override

Add a text property whose alias matches `Rendering.VoicePropertyAlias` (`ttsVoice` by default) and enter `Provider:VoiceId`, for example `Kokoro:am_michael`. It wins over the culture mapping but not over the `voiceId` parameter.

## The player

| Feature | Browser voice | Server voice |
|---|---|---|
| Play, pause, progress, seek | Yes (sentence by sentence) | Yes |
| Speed 0.75x - 1.5x | Yes | Yes (on top of the mapping's speed) |
| Voice picker | Yes, when the browser has two or more voices for the language | No - the mapped voice is fixed |
| Resume where you stopped | Yes (`localStorage`) | Yes (`localStorage`) |
| Download | No | Yes |
| Estimated listening time | Yes | Yes (exact once loaded) |

Server audio is **not fetched until the visitor presses play**, so pages cost nothing until someone listens.

Browser mode lists neural/online voices first (Edge ships very good Vietnamese ones) and remembers the visitor's choice per language. Where the browser has no voice for the language the player still plays with the browser default.

Labels ship in English and Vietnamese. For other languages override the view (below).

## Styling

The CSS uses custom properties with fallbacks. Set any of these on `.utpro-tts` or an ancestor:

| Property | Default | Controls |
|---|---|---|
| `--utpro-tts-accent` | `var(--color-primary, #1b264f)` | Play button, slider, focus ring |
| `--utpro-tts-accent-contrast` | `#fff` | Icon colour on the accent |
| `--utpro-tts-bg` | `var(--color-surface, #f4f6fa)` | Background |
| `--utpro-tts-border` | `var(--color-border, #d9dee8)` | Borders |
| `--utpro-tts-text` | `var(--color-text, #1f2430)` | Text |
| `--utpro-tts-muted` | `var(--color-text-muted, #5b6475)` | Secondary text |
| `--utpro-tts-radius` | `12px` | Corner radius |

Set `Rendering.PlayerStyle` to `sticky` to pin the player to the bottom of the viewport while the article scrolls.

## Overriding the markup

Copy the package view to your site at the same path and edit it:

```
Views/Shared/Components/TextToSpeechPlayer/Default.cshtml
```

Keep the `data-utpro-tts` root, the `data-role` hooks (`toggle`, `progress`, `time`, `status`, `speed`, `voice`, `download`) and the `data-*` attributes; `player.js` binds to them. The view receives a `PlayerView` (`Player` and `Labels`).

## Accessibility

The player is a labelled `group`; the play button is a real `<button>` with `aria-pressed` and a label that switches between Play and Pause; the progress bar is a native range input (arrow keys seek); status text such as "Preparing audio..." is announced through an `aria-live="polite"` region; focus rings use the accent colour. Full conformance still needs manual testing with assistive technologies on your own theme.
