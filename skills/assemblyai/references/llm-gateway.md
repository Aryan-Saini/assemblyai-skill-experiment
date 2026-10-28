# LLM Gateway

OpenAI-compatible chat completions over many models, billed to the AssemblyAI key. LeMUR is gone: send the transcript text yourself.

```python
import os, requests
H = {"Authorization": os.environ["ASSEMBLYAI_API_KEY"]}
r = requests.post("https://llm-gateway.assemblyai.com/v1/chat/completions", headers=H, timeout=120, json={
    "model": MODEL,
    "max_tokens": 800,
    "messages": [{"role": "system", "content": "Summarize this meeting in 2 sentences."},
                 {"role": "user", "content": transcript_text}]})
if r.status_code == 400 and "access" in r.text: ...  # account can't use this model, try another
r.raise_for_status()
answer = r.json()["choices"][0]["message"]["content"]
```

## Choosing `model`

- IDs are exact strings. List them with `GET https://llm-gateway.assemblyai.com/v1/models` (same header).
- Not every account can use every model. A 400 `"Your account does not have access to this LLM Gateway model"` means pick another. `qwen3.5-4b-32k-fast` (hosted by AssemblyAI) worked on a free account when the Claude, GPT and Gemini models did not, so it is a safe fallback. Try a stronger model first and fall back in code.
- EU endpoint (`llm-gateway.eu.assemblyai.com`) serves Claude and Gemini only.

## Notes

- Long transcripts: send `t["text"]`, or paragraphs from `GET /v2/transcript/{id}/paragraphs` for chaptering.
- Tools and `response_format` follow the OpenAI shape, but support varies by model; when you need JSON, ask for it in the prompt and parse defensively.
- Ask for exactly the shape you need ("2 to 3 sentences", "headings: Subjective, Objective, Assessment, Plan").
