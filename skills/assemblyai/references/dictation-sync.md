# Dictation API and Sync STT (short clips, one request, no polling)

## Dictation: spoken note -> verbatim text + cleaned text

```python
import json, os, requests
H = {"Authorization": os.environ["ASSEMBLYAI_API_KEY"]}
r = requests.post("https://dictation.assemblyai.com/v1/transcribe", headers=H, timeout=90, files={
    "config": (None, json.dumps({"llm_instruction": "Rewrite as clean text: remove filler words, false starts and repetitions. Keep every fact."}), "application/json"),   # config first
    "audio": ("note.wav", open(path, "rb").read(), "audio/wav")})
r.raise_for_status()
res = r.json()
verbatim = res["text"]
cleaned = res["llm_response"]   # best effort: null when the rewrite timed out (see res["llm_error"])
```

The rewrite has a 5 s internal deadline and is sometimes `null`. If you need cleaned text, retry the request once, then clean `text` yourself with the LLM Gateway rather than returning the verbatim text with fillers.

- Without `llm_instruction` the default cleanup runs, but about 1 in 5 times it returned the verbatim text unchanged (fillers kept, `llm_error` null). An explicit instruction was reliable. `config` keys: `llm_instruction` (replace the cleanup with your own rewrite, e.g. "bulleted action items"), `keyterms_prompt`, `stt_prompt`, `language_codes` (list), `sample_rate` + `channels` for raw PCM. Unknown keys return 400.
- Audio: WAV or raw 16-bit PCM, up to 120 s. Compressed formats return 415; convert with ffmpeg first.
- `/v1/transcribe/live` streams the body while recording; it needs the `config` part first and waits forever without it.
- SDK: Python `DictationTranscriber`, JS `client.dictation`.

## Sync STT: verbatim transcript of a short clip

`POST https://sync.assemblyai.com/v1/transcribe`, header `X-AAI-Model: universal-3-5-pro`, multipart `audio` part (WAV or PCM, 16-bit, <= 120 s, 30 s server deadline) plus optional `config` JSON (`prompt`, `keyterms_prompt`, `language_codes`, `timestamps`). Response: `{text, words, ...}`. Unknown config keys return 400.
