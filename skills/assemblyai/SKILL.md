---
name: assemblyai
description: Use for any AssemblyAI work - speech-to-text (files or real-time streaming), speaker diarization, captions/subtitles, PII redaction, translation, medical transcription, dictation, text-to-speech, voice agents (including LiveKit/Pipecat), and running LLMs over transcripts with the LLM Gateway. Use it whenever code calls an AssemblyAI API or SDK, even if the user only says "transcribe" or "voice agent".
---

# AssemblyAI

Tested patterns. Training data is stale for this API; prefer what is written here.

## Rules

1. **Auth header is the raw key:** `Authorization: <API_KEY>`. No `Bearer` (REST, streaming, LLM Gateway, TTS, Dictation).
2. **Raw HTTP is the most reliable path** (recipes use only `requests`/`websockets`). SDKs are fine too: `pip install -U assemblyai`, `npm i assemblyai@latest` (TTS needs >= 1.8.0 / 4.42.0).
3. **Use the purpose-built feature, not a workaround:** redaction via `redact_pii`, captions via the SRT export, translation via Speech Understanding, medical via `domain`, cleanup of a spoken note via the Dictation API. Only reach for the LLM Gateway when no feature fits.
4. **Leave models at their defaults** unless asked. Pre-recorded picks the best model automatically; streaming should set `speech_model=universal-3-6-pro`.
5. **Always end streaming sessions** with `{"type": "Terminate"}`. Open sockets bill until a 3 hour cap.
6. **Never use:** LeMUR (shut down), `word_boost` (use `keyterms_prompt`), top-level `summarization`/`auto_chapters` (removed Oct 20 2026; use the LLM Gateway), the v2 realtime socket, model names `best`/`nano`/`slam-1`/`universal-3-pro`/`u3-rt-pro`, and the Voice Agent API (`agents.assemblyai.com`, being retired; build the cascade in `references/voice-agents.md`).
7. Errors come back as JSON with an `error` field; read it. Free accounts cap concurrent streaming sessions (`1008 Too many concurrent sessions`): retry with backoff.

## Endpoints (US; add `.eu` for EU, e.g. `api.eu.assemblyai.com`)

| Product | Endpoint |
|---|---|
| Pre-recorded STT | `https://api.assemblyai.com/v2/upload`, `/v2/transcript` |
| Streaming STT | `wss://streaming.assemblyai.com/v3/ws` |
| Sync STT (clips <= 120 s) | `POST https://sync.assemblyai.com/v1/transcribe` |
| Dictation (clip -> verbatim + cleaned text) | `POST https://dictation.assemblyai.com/v1/transcribe` |
| LLM Gateway (OpenAI-compatible) | `https://llm-gateway.assemblyai.com/v1/chat/completions`, `GET /v1/models` |
| Text-to-speech | `POST https://tts.us.assemblyai.com/v1/speech`, `wss://tts.us.assemblyai.com/v1/ws` |

## Pick the product

| Need | Use | Details |
|---|---|---|
| Transcribe a file, speakers, captions, PII, translation, medical, keyterms | Pre-recorded | below |
| Live mic / phone audio, agent assist | Streaming v3 | `references/streaming.md` |
| Voice agent (listen, think, speak) | Streaming + LLM Gateway + TTS | `references/voice-agents.md` |
| Speak text | TTS | `references/tts.md` |
| Summaries, notes, Q&A, extraction over a transcript | LLM Gateway | `references/llm-gateway.md` |
| Short voice note to clean text; short clip, no polling | Dictation / Sync | `references/dictation-sync.md`. Pass an explicit `llm_instruction`: the default cleanup sometimes returns the verbatim text, fillers included, with no error |

## Pre-recorded recipe

```python
import os, time, requests
API, H = "https://api.assemblyai.com", {"Authorization": os.environ["ASSEMBLYAI_API_KEY"]}

def transcribe(path, **params):
    with open(path, "rb") as f:  # raw bytes body, not multipart
        url = requests.post(f"{API}/v2/upload", headers=H, data=f).json()["upload_url"]
    tid = requests.post(f"{API}/v2/transcript", headers=H, json={"audio_url": url, **params}).json()["id"]
    while True:
        t = requests.get(f"{API}/v2/transcript/{tid}", headers=H).json()
        if t["status"] == "completed": return t
        if t["status"] == "error": raise RuntimeError(t["error"])
        time.sleep(2)
```

`t["words"]` and `t["utterances"]` carry `start`/`end` in ms; speakers are `A`, `B`, ... In production use `webhook_url` instead of polling.

| Want | Params on `/v2/transcript` | Result |
|---|---|---|
| Speakers | `speaker_labels=True` (optional `speakers_expected`) | `t["utterances"]` |
| Rare names, jargon | `keyterms_prompt=["Supabase", "Aryan Bhalla"]` (exact spelling) | better spelling |
| Context | `prompt="Cardiology consult about chest pain."` (a description, not instructions) | |
| Medical terms | `domain="medical-v1"` (en, es, de, fr) | drug names, dosages |
| PII redaction | `redact_pii=True, redact_pii_policies=["person_name","phone_number","email_address"], redact_pii_sub="entity_name"` | redacted `text`; also `redact_pii_audio=True` |
| Translation | `speech_understanding={"request": {"translation": {"target_languages": ["es"]}}}` | `t["translated_texts"]["es"]` |
| Language | `language_code="es"` or `language_detection=True` | |
| Captions | after completion `GET /v2/transcript/{id}/srt` (or `/vtt`), optional `?chars_per_caption=64` | SRT text |

Summaries, chapters, SOAP notes, action items: transcribe, then send `t["text"]` to the LLM Gateway.
