# More bugs

These are places where AssemblyAI's skill, docs, site and SDKs disagree, so a coding agent gets a different answer depending on which one it reads. Each was checked against the live source on Oct 28, 2026.

## 1. The skill has no TTS and pins old SDKs

**Problem.** The skill's last commit is Oct 14. TTS launched Oct 27. The skill calls Python `1.6.1` and Node `4.41.5` "the current releases", but TTS needs 1.8.0 and 4.42.0.

**Fix.** Add TTS. Say "install the latest" and list the minimum version per feature instead of pinning.

![Skill pins 1.6.1 and 4.41.5](images/05-skill-stale-sdk-versions.png)
![Last commit Oct 14](images/07-skill-last-commit.png)
![npm 4.42.0](images/08-npm-latest.png)
![PyPI 1.8.2](images/09-pypi-latest.png)

## 2. The skill teaches the Voice Agent API, which is being retired

**Problem.** `models.md` says "The Voice Agent API is being retired. Don't recommend it for new builds." The skill still has a full section on it.

**Fix.** Teach the cascade (streaming STT, LLM Gateway, TTS) or LiveKit and Pipecat instead.

![Skill Voice Agent API section](images/06-skill-voice-agent-api.png)

## 3. The TTS page says "Coming soon" after launch

**Problem.** `models.md` says TTS launched Oct 27 and the quickstart is live. The product page still says "Coming soon" with a waitlist.

**Fix.** Update the product page and changelog with the launch.

![TTS page: Coming soon](images/01-tts-coming-soon.png)
![models.md: launched October 27](images/02-models-md-tts-launched.png)

## 4. `speech_model` is "deprecated" with no scope

**Problem.** `models.md` calls `speech_model` deprecated three times without saying that only applies to pre-recorded audio. Its streaming section then says switching models is "a one-line change to `speech_model`".

**Fix.** Add "for pre-recorded audio" to the deprecation lines.

![Deprecated, no scope](images/03-models-md-speech-model-deprecated.png)
![Streaming uses speech_model](images/04-models-md-speech-model-one-line.png)

## 5. The SDK READMEs point to a dead MCP URL

**Problem.** Both SDK READMEs (also on npm and PyPI) say to add `https://mcp.assemblyai.com/docs`. It returns 503. The working docs MCP is `www.assemblyai.com/docs/mcp`.

**Fix.** Update the READMEs or redirect the old URL.

![mcp.assemblyai.com/docs returns 503](images/10-mcp-paused.png)

## 6. The API MCP has no host, and uses a different auth format

**Problem.** The API MCP README says to connect to `https://<host>/mcp` with `Authorization: Bearer <key>`. The REST API and the skill say never to use `Bearer`.

**Fix.** Host the MCP, and either accept both formats or call out the exception in the skill.

![API MCP README](images/11-mcp-readme-host-bearer.png)
![Skill says Bearer is a mistake](images/12-skill-readme-bearer-warning.png)

## 7. The Node SDK repo still advertises LeMUR

**Problem.** LeMUR shut down Mar 31 and was removed from the Node SDK in v4.37.0 (issue #171). The repo description still mentions "the latest LeMUR models".

**Fix.** Update the repo description.

![Repo description](images/13-node-sdk-lemur-description.png)
![Issue #171](images/14-node-sdk-issue-171.png)

## 8. The published skill eval doesn't run code

**Problem.** The eval in the skill repo has 3 tasks, 1 run each, Claude only, graded by reading answers rather than running them. `feedback.json` has been empty and "in_progress" since March.

**Fix.** Run the code agents write, across several agents, on every docs or SDK release. That's what this repo tries.

![feedback.json](images/15-skill-eval-feedback.png)
![benchmark.json](images/16-skill-eval-benchmark.png)
