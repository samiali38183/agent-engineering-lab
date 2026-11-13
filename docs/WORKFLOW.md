# Workflow: how agent work is delegated, claimed, integrated and stopped

This describes, in general terms, how I run agents on a private codebase. I describe the private files without naming or quoting them, because they contain business detail. The agent rules file (`AGENTS.md`) also exists in the public snapshot. The labels (DEPLOYED, TESTED, IMPLEMENTED, PLANNED) are defined in the [README](../README.md).

## Principles

1. Agents coordinate through files in the repo, not chat. If a decision isn't in a file, the next session doesn't know it.
2. Rules are constraints an agent can check a draft against ("price text may only come from the plan file", "never print keys"), not goals like "be careful with pricing".
3. A lane is done when a command output or a commit shows it, not when the agent says so.
4. Each file has one writer and each release has one integrator. Running agents in parallel is only safe where their write sets don't overlap.
5. Anything involving money, outside contact or policy waits for me, and the agent keeps working on something else.

## The artifacts

| Artifact | Purpose | Who writes it |
|---|---|---|
| `AGENTS.md` | Binding rules and "where things are". First file any agent reads. | Owner and coordinator; edits are commits |
| State document | Current state in fixed sections (production, verified, not deployed, blockers, security, next action). Older sections are marked HISTORICAL when a newer one supersedes them. | Coordinator after each verified change |
| Owner directives | Time-boxed instructions with explicit hard stops, saved verbatim in `docs/`. | Owner |
| Ops state | Control flags (`PAUSE`, `LOCK`), delegate limits, retry limits, notification rules. | Coordinator |
| Work board | Lane and file ownership per machine, plus a claims table edited only through a helper. | Helper script |
| Controller log | Dated, evidence-based reconciliation of every lane. | Coordinator |
| Claims registry | Every public claim with a state, source and re-verify date. | Any lane; audited |

## Delegation

The coordinator is Hermes Agent. It reads the ops state, then the project state, decides the next lane or lanes and delegates. Its operating rules:

- Single coordinator; do not create another scheduler or spawn recursive agents.
- At most 8 delegates per run, each with a named output file and no two writers on the same file.
- If the `PAUSE` flag file exists in the ops state, do nothing. If the `LOCK` flag file exists and is younger than three hours, exit.
- A failing action is tried at most twice per run; record it as a blocker and move on. Do not rerun tests with unchanged conditions.
- Notify the owner only for an interested reply, an action only the owner can take, a failure or a verified milestone. Everything else goes in the file.

A lane is a scoped job with a short id that appears in commit messages and checkpoint files. Typical scopes are a portal UI change, call summaries, CRM linking, a competitive-gap study, a cost model, a red-team pass, a documentation audit and a deploy. Implementation lanes use Claude Code against the existing test suite, and research and QA lanes produce documents and failing tests.

I work from two machines. As documented in October, Machine A builds, tests, deploys and is the only one that holds the outbound-email sender. Machine B, a second workstation, takes outreach and coordination tasks. It is forbidden from editing application code, documents owned by A, or running the sender. A later update gave B its own coordinator and a small set of directories it owns. The ownership table lives in the work board, and the starter prompt for B repeats the same constraints.

A lane is claimed by running a small helper script with a lane id and a machine name. The helper pulls with rebase, edits only the board file, commits and pushes. If git refuses, it stops and the claim isn't taken. It rejects a claim on a lane that belongs to the other machine, and its check command fails on a claim older than 24 hours with no progress note. On a rebase conflict the rule is to abort, never force-push, and leave the resolution to Machine A. The helper has a test that creates two clones and races them (TESTED).

## Integration

- Lane branches end with a commit whose message says the lane is ready (a "checkpoint" commit). Evidence in the controller log is the commit hash and age.
- One integrator lane merges ready branches into a release branch, runs the full gates and deploys. Lanes that are finished but not merged are reported as such, not as shipped.
- Before the deploy, the previous release image is noted as the rollback target. After it, the live self-check runs.
- Source files may use mixed line endings across machines, so edits are made with line-ending-safe scripts and byte-checked afterwards.

## Claim levels

The private notes grade every statement with a claim level, lowest to highest: proposed, implemented, unit tested, integration tested, deployed, production verified, proven with real customers. A document may not state a level it hasn't reached. Call Kettle hasn't reached the highest level, and nothing in this repo claims it.

## When documents disagree

State documents drift. The newest dated section at the top of the state document wins over older text, and an older "current" statement is marked HISTORICAL when it's superseded instead of being deleted. When a document and the code disagree, the code and the audit win, and I correct the document.

## Rules that came from something going wrong

- An interrupted run left orphaned test processes behind, and they caused database-lock errors that looked like regressions. The rule is to check for leftover processes before and after every full suite run.
- A token-budget test failed once (7,301 against a limit of 7,300) and passed on re-run. I didn't accept the re-run. The cause was the wall-clock date and the open/closed text in the prompt, which I reproduced by sweeping 730 clocks over five scripted scenarios. The fix pins the clock.
- After the plan ladder changed, older documents still showed retired figures. Now one plan file is the source, the audit fails on any price-looking amount that isn't in it, and superseded documents are removed with a recorded recovery list.
- A second email sender would re-contact people and blow the shared daily sending limit. Exactly one machine holds the sender and its suppression state, enforced by file ownership and the starter prompt for the other machine.
- Two findings from a live end-to-end run (no operator alert on an unbound paid checkout, and a tenant that was never bound) became regression tests the same day, in a named test file.

## What is not in place

- Agents have no direct access to production secrets. That is a rule and a habit, not a technical sandbox. The main protection is that secrets sit in platform secrets and never in the repo.
- Nothing automatically checks that an agent obeyed a rule, except where a test or audit encodes it (price text, retired brand, secret leakage, route guards). The rest depends on my review.
- Cloud coding sessions are covered by a written guide: one repo, no production secrets, work on a branch, review before merge. I'm not claiming a running cloud pipeline.
