# Guardrails: from model suggestion to consequence

The runtime agent in Call Kettle answers phone calls. Anything a caller says is untrusted input, and anything the model says is untrusted output. This note lists the layers in between, the threat each one addresses, and the evidence behind it. Labels are defined in the [README](../README.md). Items marked "public" can be read in [call-kettle](https://github.com/samiali38183/call-kettle); the rest are in the private codebase.

The rule behind all of it: a prompt instruction is a request, a line of code is a guarantee. Anything with consequences (booking, cancelling, transferring, spending money, telling a caller something is true) is enforced in code.

## 1. Input layer

| Control | Threat | Evidence |
|---|---|---|
| Telephony webhook signature check on every voice route; fail closed if no token or signature | Forged webhooks driving the agent or running up cost | TESTED, probed against production |
| Tenant resolved from the signed URL, never from model or caller text | Cross-tenant access | TESTED (`test_isolation`, public) |
| Caller utterance length cap; per-number call-rate limit; per-IP limits on public endpoints | Flooding, token-stuffing | TESTED |
| Strict tenant config schema (`extra: forbid`, validators) | A typo or hostile edit surfacing mid-call | TESTED (public) |
| Route guard test that enumerates every route and asserts it is either deliberately public or refuses anonymous requests | A future route shipped without authentication | TESTED |
| Email header and calendar-invite field sanitization | Header injection through caller-supplied names | TESTED |

## 2. Model layer

| Control | Threat | Evidence |
|---|---|---|
| Per-tenant tool list; disabled tools are not offered and are rejected if called | Model invoking a capability the tenant did not buy or allow | TESTED (public) |
| Iteration cap per turn, reply token cap, short API timeout, one retry | Runaway tool loops and cost | IMPLEMENTED and TESTED (public) |
| Conversation history trimmed to a fixed window | Token growth on long calls | IMPLEMENTED |
| Emergency phrases bypass the model and trigger a fixed emergency message plus owner alert | A model improvising on gas, fire or CO calls | TESTED (offline, scripted model) |
| "Ask for a person" phrases transfer without a model call | Model stalling a caller who wants a human | TESTED |

## 3. Output layer

| Control | Threat | Evidence |
|---|---|---|
| Claim guard: the reply may not say booked, cancelled, moved or transferred unless the matching tool call succeeded this call | The model confirming something that did not happen | TESTED (23 truth-invariant tests and 22 failure-injection tests at the time of review; public code) |
| Date correction on spoken dates | Wrong weekday or ordinal read aloud | IMPLEMENTED (public) |
| No price quotes unless the exact figure is in the tenant's FAQ; no repair or safety advice | Liability from improvised advice | PROMPT-LEVEL ONLY: a prompt rule plus live spot checks, not a code guarantee. This is the weakest row in the table and I treat it as such |

The offline red-team tests use a scripted model that says the worst plausible thing, so they prove the code around the model, not the model's own behavior. The live-model spot check was 16 lines on one model and says so.

## 4. Action layer (tools)

| Control | Threat | Evidence |
|---|---|---|
| Hours, slot grid, lead time and service-area rules enforced in the tool, not the prompt | Booking outside what the owner allows | TESTED (public) |
| `BEGIN IMMEDIATE` plus overlap query plus a unique constraint on tenant and slot | Double booking under concurrency | TESTED (`test_frontdesk_concurrency`, public) |
| Ownership check on change and cancel: only bookings made on this call or by this caller ID | A prompt-injected model touching someone else's booking | TESTED (public) |
| Webhook delivery to customer URLs: https only, every resolved address must be public, no redirects, size and time caps, signed with a timestamp and rejected after 5 minutes of skew | SSRF and replay through customer-configured webhooks | TESTED (URL and resolver cases) |

## 5. Cost and capacity layer

| Control | Threat | Evidence |
|---|---|---|
| Cost-based monthly ceiling per tenant; once reached, calls drop to a message-taking mode that does not call the model | One tenant or attacker running up spend | TESTED |
| Cap on degraded calls after the ceiling | Spend continuing after the ceiling | TESTED |
| Concurrent-call ceiling with an atomic admit step (hold between check and row) | Burst of simultaneous calls all passing a count-then-act check | TESTED; found by load test (42 admitted against a ceiling of 24 at 64 simultaneous callers, before the fix) |
| Over-capacity callers go to a staff phone, then to message-taking, then to a plain reject, never to the model | Capacity failures turning into lost leads or a loop | TESTED |
| Worst-case-loss model with a pinned test | A single heavy customer costing more than a set bound | MODELED, not measured; pinned in `test_worst_case_cap.py` |
| Duplicate request replay cache keyed by call and the carrier's idempotency token; per-call turn lock | Carrier retries doubling model cost; overlapping turns corrupting history | TESTED |
| Telephony usage guard script | Carrier-side spend surprises | IMPLEMENTED |

Capacity numbers in the notes are labeled MEASURED (produced by a harness or test), MODELED (arithmetic from repo models) or UNVERIFIED (a vendor limit from memory or config). I keep those labels on purpose: it stops a modeled number from being repeated later as a measured one.

## 6. Data layer

| Control | Threat | Evidence |
|---|---|---|
| Card-number (Luhn-valid) and SSN-shaped scrubbing on stored transcripts and escalation summaries; phone numbers are kept deliberately | Sensitive data at rest, and in the emails that carry summaries | TESTED (25 cases, including phone numbers that must survive) |
| Log redaction: keys, emails, all but the last four phone digits | Secrets and PII in logs | TESTED |
| Automatic transcript deletion after 90 days; rolling backups and volume snapshots | Data retained longer than needed | IMPLEMENTED |
| Owner-portal changes written to an audit table; portal pages are server-rendered with no scripts and a strict content security policy | Silent configuration changes; cross-site scripting | IMPLEMENTED |
| Password hashing with scrypt, per-account and per-IP lockouts, server-side sessions, CSRF tokens on every POST | Account takeover | TESTED |
| API docs and schema endpoints not published; security headers on every response | Reconnaissance, clickjacking | TESTED, probed against production (the docs endpoints were exposed until the review found them) |

## 7. Secrets

- Keys live in platform secrets. The environment file is git-ignored.
- A test fails if a secret value from the environment appears in any tracked file; a history scan was run once during the review.
- The master key is checked never to appear in any admin page, link or log.
- Rule for agents: never paste a key into a prompt, chat or document. Cloud coding sessions are specified to receive no production credentials.
- Dependency audit (`pip-audit`) was clean on 2026-09-30 and is meant to be rerun per release; I have not recorded a later run in this write-up.

## 8. Outbound compliance (sales, not the product)

A written policy, built from reading public rules on 2026-10-01. It states that it is not legal advice and has not been reviewed by a lawyer.

- Calls are placed live by a person from a stable business number. No AI, prerecorded or synthesized voice places an outbound sales call. No autodialer. No caller-ID spoofing.
- Published business numbers only. Identify yourself and the company first, accurately. No pretext calls.
- Call within a stated local-time window.
- "Do not call again" is permanent, recorded immediately, and checked before every call.
- Email: truthful sender, commercial-message identification, a working opt-out honored the same day, and a separate sender from transactional mail so a complaint cannot hurt product deliverability.
- Texting to prospects is not done. Product texting stays off until carrier registration passes.

## 9. What these layers do not cover

- A model that is manipulated into saying something false that is not a booking, cancellation or transfer claim. The claim guard covers consequences; it does not make every sentence true.
- A compromised platform account or a leaked key. Controls above reduce exposure, they do not replace key rotation and account hygiene.
- Behavior under real customer load, which is unmeasured.
- Anything an independent reviewer would find. The security review was written by the builder.
