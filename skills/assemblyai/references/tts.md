# Text-to-speech (AssemblyAI-TTS-1, launched Oct 27 2026)

## Whole file over HTTP

```python
import os, requests
H = {"Authorization": os.environ["ASSEMBLYAI_API_KEY"]}
r = requests.post("https://tts.us.assemblyai.com/v1/speech", headers=H, timeout=120,
                  json={"text": text, "container": "wav", "stream": False})
r.raise_for_status()
open("out.wav", "wb").write(r.content)  # complete WAV, 24 kHz 16-bit by default
```

- Only `text` is required. Optional: `voice_id` (default voice `aai-jane`, English), `language`, `sample_rate`, `encoding`, `container` (`wav` for a playable file; omit for raw PCM), `stream` (omit/true streams bytes as they are made).
- Response headers say what ran: `x-tts-voice-id`, `x-tts-sample-rate`, `x-tts-encoding`, `x-tts-container`. Read the sample rate from there when writing your own WAV header.
- EU: `tts.eu.assemblyai.com`.

## SDKs (Python >= 1.8.0, JS >= 4.42.0)

```python
import assemblyai as aai
tts = aai.TextToSpeech(api_key=os.environ["ASSEMBLYAI_API_KEY"])
resp = tts.synthesize("Hello there.", aai.TTSConfig(container=aai.TTSContainer.wav))
open("out.wav", "wb").write(resp.audio)
```
JS: `client.tts.synthesize({ text, container: "wav" })`, `client.tts.stream(...)`, `client.tts.session({ voiceId })` (session options are camelCase).

## Low latency (voice agents)

`wss://tts.us.assemblyai.com/v1/ws?voice_id=...&sample_rate=24000&encoding=pcm_s16le` with the same header. Wait for `Begin`, send `{"type":"Generate","text":...}` then `{"type":"Flush"}`, collect `Audio` frames until `FlushDone`, then `{"type":"Terminate"}`. You can stream LLM tokens into `Generate` as they arrive.
