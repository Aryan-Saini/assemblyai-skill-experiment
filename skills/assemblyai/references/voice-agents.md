# Voice agents

The Voice Agent API (`wss://agents.assemblyai.com/v1/ws`) is being retired. Build the cascade instead: streaming STT for the caller's turn, LLM Gateway for the reply, TTS for the voice. All three use the same key.

## One turn, no framework

1. Stream caller audio to `wss://streaming.assemblyai.com/v3/ws` (see `streaming.md`). Take the `Turn` message with `end_of_turn: true` as the caller's utterance.
2. Send it to the LLM Gateway with your system prompt (see `llm-gateway.md`). Keep replies short and speakable: no markdown, no lists.
3. Speak the reply with TTS: HTTP `/v1/speech` for a file, or the TTS WebSocket for lowest latency (see `tts.md`).
4. Send `{"type":"Terminate"}` on the STT socket when the call ends. For the next turn, pass the agent's last reply as `agent_context` via `UpdateConfiguration` so the STT model has context.

## Frameworks

- **LiveKit:** `livekit-plugins-assemblyai` / `@livekit/agents-plugin-assemblyai`, `assemblyai.STT(model="universal-3-6-pro")`. Set the model explicitly; some plugin versions default to 3.5.
- **Pipecat:** `AssemblyAISTTService(api_key=..., settings=AssemblyAISTTService.Settings(model="universal-3-6-pro"))`. The older `connection_params=` form is deprecated; the default model is still 3.5.
- Turn detection: tune `min_turn_silence` / `max_turn_silence` (ms) or `mode`. `end_of_turn_confidence_threshold` has no effect on Universal-3.6 Pro.
- Telephony (Twilio): 8 kHz `pcm_mulaw` straight through; base64-decode media payloads before sending.
