> **Rendered documentation:** [docs.utpro.dev](https://docs.utpro.dev/uTPro.Feature.TextToSpeech/) is the canonical, always-current home of this page.

# Extensibility

## Add your own engine

Implement `ISpeechSynthesizer` and append it from your own composer. No edits to the package are needed.

```csharp
using Umbraco.Cms.Core.Composing;
using Umbraco.Cms.Core.DependencyInjection;
using uTPro.Feature.TextToSpeech.Composing;
using uTPro.Feature.TextToSpeech.Models;
using uTPro.Feature.TextToSpeech.Providers;

public sealed class MyCloudVoice : ISpeechSynthesizer
{
    private readonly IHttpClientFactory _http;

    public MyCloudVoice(IHttpClientFactory http) => _http = http;

    public string Name => "MyCloud";                    // what mappings refer to
    public bool IsConfigured => true;                   // false = pages fall back to Browser
    public string? StatusMessage => "Ready.";           // shown on the Voices card; never include secrets

    public Task<IReadOnlyList<SpeechVoice>> GetVoicesAsync(CancellationToken ct)
        => Task.FromResult<IReadOnlyList<SpeechVoice>>([new("anna", "Anna (female)", "vi-VN")]);

    public async Task<SpeechResult> SynthesizeAsync(SpeechRequest request, CancellationToken ct)
    {
        byte[] mp3 = await CallMyServiceAsync(request.Text, request.VoiceId, request.Speed, ct);
        return new SpeechResult { Audio = mp3, Format = "mp3" };
    }

    private Task<byte[]> CallMyServiceAsync(string text, string voice, double speed, CancellationToken ct)
        => throw new NotImplementedException();
}

[ComposeAfter(typeof(TextToSpeechComposer))]
public sealed class MyVoicesComposer : IComposer
{
    public void Compose(IUmbracoBuilder builder)
        => builder.TextToSpeechSynthesizers().Append<MyCloudVoice>();
}
```

The engine appears as a card in **Voices**, can be mapped to a culture, previewed, and its output is cached exactly like the built-in ones.

### Contract

| Member | Notes |
|---|---|
| `Name` | Unique, stable. Stored on every cache row; renaming it orphans existing audio. |
| `IsConfigured` | Checked on every render. Keep it cheap (no network calls). |
| `GetVoicesAsync` | Feeds the Voice suggestions. May do I/O; failures are logged and the list shows empty. |
| `SynthesizeAsync` | Receives normalised text with pronunciation rules already applied, the voice id from the mapping and a speed between 0.5 and 2.0. Return bytes plus a short lowercase extension (`wav`, `mp3`, `ogg`, ...). Set `DurationSeconds` when you know it. Throw on failure; the error is stored on the cache row. |

Engines are singletons. The core already limits how many syntheses run at once (`Queue:MaxParallelism`), and may call `SynthesizeAsync` with up to `MaxTextLength` characters, so split long text yourself if your service has a limit (`WavUtil.Concat` joins WAV parts).

### Remove a built-in engine

You rarely need to: the OpenAI-compatible engine is inert until `Endpoint`, `ApiKey` and `Model` are set, and the add-ons do nothing unless installed. The built-in types are internal, so remove one by type name:

```csharp
var openAi = Type.GetType(
    "uTPro.Feature.TextToSpeech.Providers.OpenAiCompatibleSpeechProvider, uTPro.Feature.TextToSpeech", throwOnError: true)!;
builder.TextToSpeechSynthesizers().Remove(openAi);
```

To replace one, remove it and append your own engine with the same `Name`; existing cache rows keep working.

## Override the player markup

Copy `Views/Shared/Components/TextToSpeechPlayer/Default.cshtml` into your site at the same path. See [Rendering](rendering.md#overriding-the-markup) for the hooks `player.js` needs.

## Prepare audio from your own code

To warm the cache from a script or a custom job, call the same entry point the ViewComponent uses:

```csharp
var model = textToSpeechService.Prepare(new PlayerRequest { Content = page, Text = html, Culture = "vi-VN" });
if (model?.Hash is { } hash)
{
    await textToSpeechService.EnsureGeneratedAsync(hash, ct);
}
```

`Prepare` registers and queues; `EnsureGeneratedAsync` waits for the file.
