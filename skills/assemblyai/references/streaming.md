# Streaming STT (v3)

`wss://streaming.assemblyai.com/v3/ws?sample_rate=16000&encoding=pcm_s16le&speech_model=universal-3-6-pro`
Header `Authorization: <key>` (server side), or `?token=` from `GET https://streaming.assemblyai.com/v3/token?expires_in_seconds=60` for browsers. Never ship the key to a client.

## Stream a 16 kHz mono PCM16 WAV at real-time pace

```python
import asyncio, json, os, time, wave, websockets
KEY = os.environ["ASSEMBLYAI_API_KEY"]
URL = "wss://streaming.assemblyai.com/v3/ws?sample_rate=16000&encoding=pcm_s16le&speech_model=universal-3-6-pro"

async def stream(path, chunk_ms=50):
    with wave.open(path) as w:  # convert other formats first (ffmpeg -ar 16000 -ac 1)
        pcm = w.readframes(w.getnframes())
    step, turns = 16000 * 2 * chunk_ms // 1000, {}
    async with websockets.connect(URL, additional_headers={"Authorization": KEY}) as ws:
        async def send():
            t0 = time.monotonic()
            for i, off in enumerate(range(0, len(pcm), step)):
                await ws.send(pcm[off:off + step])
                await asyncio.sleep(max(0, t0 + (i + 1) * chunk_ms / 1000 - time.monotonic()))
            await ws.send(json.dumps({"type": "Terminate"}))
        async def recv():
            async for raw in ws:
                m = json.loads(raw)
                if m["type"] == "Turn" and m.get("end_of_turn"):
                    turns[m["turn_order"]] = m["transcript"]
                elif m["type"] == "Termination":
                    return
        await asyncio.gather(send(), recv())
    return [turns[k] for k in sorted(turns) if turns[k].strip()]
```

`websockets` >= 14 uses `additional_headers` (older: `extra_headers`). `Error` messages and close codes: 1008 = auth or too many concurrent sessions, 3007 = bad chunk size.

## Messages

- Server: `Begin` (session id, echoed config), `Turn` (`transcript`, `end_of_turn`, `turn_order`, `words`), `SpeechStarted`, `Termination`, `Error`. Each `Turn` re-sends the whole turn so far: replace, don't append. Keep only `end_of_turn: true` for finals.
- Client: binary audio frames of 50 to 1000 ms (outside that closes with 3007), `{"type":"UpdateConfiguration", ...}`, `{"type":"ForceEndpoint"}`, `{"type":"Terminate"}`.

## Useful params (query string)

`speaker_labels=true` (adds `speaker_label` per Turn), `keyterms_prompt=` (JSON list, up to 100), `prompt=` (context description), `language_codes=` (list), `domain=medical-v1`, `mode=min_latency|balanced|max_accuracy`, `min_turn_silence`/`max_turn_silence` (ms) for turn detection. Phone audio: keep 8 kHz `encoding=pcm_mulaw`, don't upsample.

Streaming uses the singular `speech_model`; pre-recorded uses the plural list `speech_models`.

SDK: Python `from assemblyai.streaming.v3 import RealTimeTranscriber, RealTimeParameters` (older name `StreamingClient` still works), JS `client.streaming.transcriber({...})`. Pass `speech_model="universal-3-6-pro"` and close the session with terminate so it stops billing.
