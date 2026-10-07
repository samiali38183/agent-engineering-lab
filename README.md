# Agent Engineering Lab

How I build real software with AI coding agents (Hermes Agent, Claude Code, Codex) and keep it trustworthy: parallel sub-agents, tests as the contract, benchmarks before opinions, and security reviewed like any internet-facing system.

This is a sanitized engineering write-up from a live voice-AI project. It contains no customer data, pricing or secrets.

## Stack

| Layer | What |
|---|---|
| Runtime | FastAPI on Fly.io, persistent volume, **SQLite (WAL)** as system of record |
| Telephony / speech | Twilio (production), Deepgram streaming STT, Amazon Polly; **ElevenLabs, Telnyx, Retell** under benchmark |
| Reasoning | Claude tool-use loop, schema-validated tools ("the model proposes, code disposes") |
| Scheduling | Google Calendar, Cal.com, Calendly, iCal busy feeds behind one provider interface |
| Agent tooling | Hermes Agent orchestrating Claude Code and Codex builders and parallel sub-agents |

## The agent workflow

1. **Fan out by stream.** Independent sub-agents (voice audit, call-routing modes, usage metering, calendar audit, portal UI, concurrency, field-service research) get disjoint files and explicit do-not-touch lists.
2. **Tests are the contract.** The backend suite is 3,400+ tests. After each fan-out: re-run the affected slice, diff against a clean baseline worktree, accept only regressions I can explain.
3. **Claim levels.** Every feature is tagged `BUILT < UNIT TESTED < INTEGRATION TESTED < DEPLOYED < PRODUCTION VERIFIED`. Nothing is described above its level.
4. **Measured vs modeled.** Cost and latency numbers are labeled so a model never passes as a measurement.
5. **No secrets in prompts.** Keys live in gitignored env files or platform secrets and are checked with read-only calls.

## Voice-stack benchmark harness

Instead of debating vendors, a reproducible harness runs 28 scenarios (book, reschedule, cancel, transfer, emergency, barge-in, noisy audio, tool failure, business timezone different from the server) through the real conversation engine. A deterministic stub model keeps CI free; `--live` runs real models. Scoring: booking correctness, tool-call correctness, hallucination checks, transfer correctness, recovery, plus slots for latency p50/p95 and speech accuracy from real-call runs. Compared: Twilio baseline, Deepgram streaming, Telnyx managed and custom, Retell, ElevenLabs voices, native audio models.

## A real concurrency bug

A per-client concurrent-call ceiling looked right but was not atomic: a burst of 64 simultaneous callers admitted 42 against a ceiling of 24 (check-then-insert race). Fix: an admission lock around count-and-claim, a hard global cap, and a rollback-safe default. A local load test now holds the ceiling exactly at 8, 16, 32 and 64. SQLite WAL, one Fly machine and a shared model client were analysed as the true scaling limits, with measured and modeled claims separated.

## Security engineering for an agent system

- Signature-verified telephony webhooks with duplicate-event handling
- Tenant isolation on every portal query, with cross-tenant leak tests
- Output escaping and CSRF on all write routes
- Booking-claim guard: the agent cannot confirm what the tool did not do; tool args validated first
- Worst-case cost modeling so abuse cannot create an unbounded bill
- Data minimisation (caller emails purged after appointments); secrets never committed
- Persona that sounds natural but never claims to be human and answers truthfully if sincerely asked

## Takeaway

Agents are fast; verification is the bottleneck. The skills that mattered: scoping sub-agents so they cannot collide, demanding evidence before trusting a summary, and treating a voice agent like any internet-facing service.

Portfolio: https://samiali38183.github.io/projects/agent-lab.html
