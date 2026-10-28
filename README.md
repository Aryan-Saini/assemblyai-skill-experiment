# AssemblyAI skill, slimmed down

I rewrote [AssemblyAI's coding-agent skill](https://github.com/AssemblyAI/assemblyai-skill/tree/ef37bb3/skills/assemblyai) to be 95% smaller, then benchmarked both versions with Claude Code (Opus 5.5) and Codex (GPT 6.1 Sol). The smaller one passed more tasks and used about half the tokens.

> **This is an experiment and a first shot.** I don't know which use cases, models or prompts AssemblyAI's customers actually use. I picked tasks from the use cases on assemblyai.com and tested on two frontier models. I just like to move fast and directly show an example of what I did. I'd change this after seeing how customers use their agents and what they send to the models.

## The idea

AI has gotten really good. Frontier models don't need a 35,000-word manual loaded on every task. They need the few things that change the code they write: the right endpoint, the auth header, the current models and the traps. Simple beats bloat.

## Results

Score is quality times token savings, with the original skill set to 100.

| Both agents | Score | Tasks fully passed | Best practices | Median tokens | Skill size |
| --- | ---: | ---: | ---: | ---: | ---: |
| No skill | 125.7 | 94% | 96.8% | 218,870 | 0 |
| Original skill | 100 | 92% | 97.1% | 305,836 | 34,773 words |
| **This skill** | **171.8** | **100%** | **100%** | **149,420** | **1,803 words** |

Opus 5.5 scored 164.3 and Sol 6.1 scored 179.3. On these two models, no skill at all did better than the original. The original's size cost more than it helped.

## What changed

**Removed:** parameter lists an agent can look up when it needs them, a "common mistakes" section that repeated the gotchas, pinned SDK versions that had already gone stale, and the Voice Agent API, which AssemblyAI's own `models.md` says is being retired. Nine reference files became five.

**Added:** text-to-speech (launched Oct 27, not in the original), the voice agent cascade (streaming STT, then LLM Gateway, then TTS), an explicit dictation cleanup instruction, LLM Gateway model access handling, and one table of the purpose-built features (PII redaction, captions, translation, medical, key terms).

## Bugs I found

Running agent-written code against the live API turned up three bugs:

1. **Dictation cleanup leaves fillers, silently.** With no `llm_instruction`, 3 of 16 responses still had filler words, with no error. An explicit instruction was clean 13 of 13 times.
2. **Live dictation hangs without a config part.** `/v1/transcribe/live` never returns or errors if `config` is missing.
3. **Free accounts see models they can't use.** `GET /v1/models` lists 58 models, but only 2 work on a free account, and the response doesn't say which.

Reading the docs, skill, SDKs and site side by side turned up more, mostly places where they've drifted apart. Those are in [BUGS.md](BUGS.md) with screenshots.

## Use it

Copy `skills/assemblyai` into `.claude/skills/` (Claude Code) or `.agents/skills/` (Codex).

## Limits

- 9 tasks, 2 runs each, on a free account.
- Sol was only benchmarked on the final version. Opus went through three rounds.
- Frontier models only. Smaller models might need more guidance than this skill gives.
- Not affiliated with AssemblyAI. Everything here was checked on Oct 28, 2026.

## How I did this

1. **Picked the tasks.** Nine tasks from the use cases on assemblyai.com: a meeting notetaker, real-time streaming, a voice agent, dictation, PII redaction, a medical scribe, captions, translation and key terms. Every run got the same audio, prompt and expected output.
2. **Built a clean sandbox.** A fresh Docker container per run, with Claude Code or Codex and nothing else loaded: no CLAUDE.md, no MCP servers, no other skills.
3. **Graded by running the code.** The grader runs whatever the agent built against the real API and checks the output against a known script: the transcript, speakers, redaction and medical terms. Spoken output gets transcribed back to text and compared.
4. **Set the score before starting.** I wrote down the scoring rule and the target (125, original = 100) before the first run, so I couldn't move the goalposts.
5. **Looped.** Edit the skill, test every code snippet in it against the live API, rerun the benchmark, read the failures, fix the cause. Three rounds got it from 100 to 171.8.
