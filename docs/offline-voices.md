> **Rendered documentation:** [docs.utpro.dev](https://docs.utpro.dev/uTPro.Feature.TextToSpeech/) is the canonical, always-current home of this page.

# Offline voices (Piper and Kokoro)

Both engines run on your own server: no API key, no per-request cost, and no outbound traffic once their files are in place. They ship as separate packages so the core stays small.

| | Piper | Kokoro |
|---|---|---|
| Package | `uTPro.Feature.TextToSpeech.Piper` | `uTPro.Feature.TextToSpeech.Kokoro` |
| Runs | The Piper binary, one process per synthesis | In-process (ONNX Runtime, CPU) |
| You provide | The binary + `.onnx` voice files | Nothing (model downloads on first use) or a model file |
| Vietnamese | Yes (`vi_VN-vais1000-medium`, female) | No |
| Speed (measured, CPU) | About 14x real time | About 1.3 - 2x real time |
| Memory | About 200 MB per run, released after | Model stays loaded; about 1 GB while synthesizing |
| Output | WAV, 22 kHz mono | WAV, 24 kHz mono |

## Piper

### 1. Install

```bash
dotnet add package uTPro.Feature.TextToSpeech.Piper
```

### 2. Download Piper and a voice

- Piper releases: <https://github.com/rhasspy/piper/releases> (Windows amd64, Linux amd64 / arm64 / armv7).
- Voices: <https://huggingface.co/rhasspy/piper-voices>. Download **both** the `.onnx` and the matching `.onnx.json`.

Put them in the default folders (or set `Piper:ExecutablePath` / `Piper:ModelsRoot`):

```
{ContentRoot}/umbraco/Data/uTProTextToSpeechPiper/
â”œâ”€â”€ piper/                       # the extracted release (piper or piper.exe + espeak-ng-data, libs)
â””â”€â”€ models/
    â”œâ”€â”€ vi_VN-vais1000-medium.onnx
    â””â”€â”€ vi_VN-vais1000-medium.onnx.json
```

On Linux make the binary executable:

```bash
chmod +x umbraco/Data/uTProTextToSpeechPiper/piper/piper
```

Use a Debian/Ubuntu based image. The official binaries are built for glibc and do not run on Alpine (musl).

### 3. Map a culture

The Piper card turns *Ready* and lists every model it found. Map `vi-VN` to `Piper` / `vi_VN-vais1000-medium` and preview it.

Piper maps speed to its `--length_scale` setting. How much that changes the length depends on the voice model: `vi_VN-vais1000-medium` responds only mildly. The player's own speed control works on top of it for every engine.

## Kokoro

### 1. Install

```bash
dotnet add package uTPro.Feature.TextToSpeech.Kokoro
```

The package uses the CPU build of KokoroSharp, which runs on Windows, Linux and macOS. Its voice files are copied to `{app}/voices` when you build or publish.

### 2. Model

Leave `Kokoro:ModelPath` empty and the model (about 300 MB) downloads on first use, then stays cached. On servers without internet access, or to bake it into a Docker image, download `kokoro.onnx` yourself and point `Kokoro:ModelPath` at it.

The first load takes a while (about 18 s measured). With `Kokoro:WarmUp` on, the model loads 30 seconds after start-up, but only if a Kokoro voice is mapped, so sites that do not use Kokoro never download anything.

### 3. Map a culture

Map `en` (or `en-US`, `en-GB`, `es`, `fr`, `hi`, `it`, `ja`, `pt-BR`, `zh`) to `Kokoro` and a preset such as `af_heart` or `am_michael`. The Voice field suggests every installed preset with its gender.

## Docker

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish MySite.csproj -c Release -o /app

FROM mcr.microsoft.com/dotnet/aspnet:10.0
WORKDIR /app
COPY --from=build /app .

# Piper (optional): binary + Vietnamese voice
ADD https://github.com/rhasspy/piper/releases/download/2023.11.14-2/piper_linux_x86_64.tar.gz /tmp/piper.tgz
RUN mkdir -p umbraco/Data/uTProTextToSpeechPiper/models \
 && tar -xzf /tmp/piper.tgz -C umbraco/Data/uTProTextToSpeechPiper \
 && chmod +x umbraco/Data/uTProTextToSpeechPiper/piper/piper \
 && rm /tmp/piper.tgz
ADD https://huggingface.co/rhasspy/piper-voices/resolve/main/vi/vi_VN/vais1000/medium/vi_VN-vais1000-medium.onnx umbraco/Data/uTProTextToSpeechPiper/models/
ADD https://huggingface.co/rhasspy/piper-voices/resolve/main/vi/vi_VN/vais1000/medium/vi_VN-vais1000-medium.onnx.json umbraco/Data/uTProTextToSpeechPiper/models/

# Keep generated audio across container rebuilds
VOLUME ["/app/umbraco/Data/uTProTextToSpeechCache"]
ENTRYPOINT ["dotnet", "MySite.dll"]
```

Alternatively mount the Piper folder as a volume (`-v ./tts:/tts`) and set `Piper:ExecutablePath=/tts/piper/piper` and `Piper:ModelsRoot=/tts/models`, which lets you add voices without rebuilding the image.

Give a Kokoro container at least 1.5 GB of memory.
