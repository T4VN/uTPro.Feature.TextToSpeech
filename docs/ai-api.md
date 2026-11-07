> **Rendered documentation:** [docs.utpro.dev](https://docs.utpro.dev/uTPro.Feature.TextToSpeech/) is the canonical, always-current home of this page.

# AI API voices

The core package includes an engine for any service that implements the OpenAI speech endpoint, `POST /v1/audio/speech`. That covers OpenAI itself, Azure OpenAI deployments, Groq, and gateways that put other vendors (for example Google Gemini or ElevenLabs voices) behind the same shape.

It is inactive until you configure it, and generated audio is cached, so you pay once per text and voice, not once per listen.

## Configure

```json
{
  "uTPro": {
    "Feature": {
      "TextToSpeech": {
        "OpenAi": {
          "Endpoint": "https://api.openai.com",
          "Model": "gpt-4o-mini-tts",
          "DefaultVoice": "alloy",
          "ResponseFormat": "mp3"
        }
      }
    }
  }
}
```

Keep the key out of committed files:

```bash
dotnet user-secrets set "uTPro:Feature:TextToSpeech:OpenAi:ApiKey" "sk-..."
# or, in production
export uTPro__Feature__TextToSpeech__OpenAi__ApiKey="sk-..."
```

The **OpenAiCompatible** card in **Voices** turns *Ready* and shows the host and model. Map a culture to it and pick a voice (`alloy`, `ash`, `ballad`, `coral`, `echo`, `fable`, `nova`, `onyx`, `sage`, `shimmer`, `verse`, or any voice id your provider accepts).

## What is sent

```json
{ "model": "gpt-4o-mini-tts", "voice": "alloy", "input": "...", "speed": 1.0, "response_format": "mp3" }
```

with `Authorization: Bearer {ApiKey}`. Text longer than 4,000 characters is split on sentence boundaries into several requests and the results are joined, which is why `ResponseFormat` must be `mp3` or `wav`.

The endpoint may already end in `/v1`: `https://gw.example.com/openai/v1` becomes `https://gw.example.com/openai/v1/audio/speech`.

## Self-hosted gateways

Requests to loopback, private (10.x, 172.16-31.x, 192.168.x), link-local, carrier-grade NAT and other reserved addresses are refused, which protects the server against SSRF. The check runs on the address actually being connected to, so a DNS record that changes between check and request cannot get around it. Redirects are not followed and system proxies are not used.

If you run a gateway on the same machine or network (for example Kokoro-FastAPI), opt in:

```json
"OpenAi": { "Endpoint": "http://127.0.0.1:8880", "AllowLocalEndpoint": true }
```

Only do this for an endpoint you control.

## Errors

A failed request is logged with the host and status code (never the key or the text) and recorded on the cache row, where the **Audio cache** tab shows it.

A configured API that fails (wrong key, quota, outage) does **not** switch pages to the Browser voice: the server player keeps showing "Preparing audio..." and retries. Fix the cause, then press **Regenerate**. Pages fall back to Browser only when the engine is not configured at all, so remove `Endpoint` or the mapping if you need that during an outage.
