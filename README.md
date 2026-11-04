# Agent Engineering Lab

Notes on how I build software using AI coding agents, and how I check their work. The code itself is
private; this repo is the summary.

Tools I've used: Hermes Agent, Claude Code, Codex, Claude tool use, Deepgram, ElevenLabs, Fly.io,
SQLite, FastAPI.

## What I've worked on

- Running several AI agents in parallel on separate parts of a project, then merging and reviewing
  their changes. Nothing ships unless the automated tests pass (3,400+ tests).
- Phone agents that use tools, with live speech-to-text and text-to-speech. I tested different
  speech, voice and model vendors on real calls.
- Deployment on Fly.io with SQLite, plus load tests for concurrent calls. I keep measured numbers
  separate from estimates.
- Scheduling and CRM integrations using OAuth and scoped API keys.
- Small benchmark scripts so I can compare options with data instead of guessing.

## Security

An AI agent that answers phone calls is an internet-facing system, so I treat it like one:
authenticated inputs, tenant isolation, cost limits, escaped output, minimal data retention, no
secrets in prompts or code, and audit logs. Load testing caught a real concurrency race, which I
fixed before it reached customers.

Portfolio write-up: https://samiali38183.github.io/projects/agent-lab.html
