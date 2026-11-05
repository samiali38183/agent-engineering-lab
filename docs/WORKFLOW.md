# Workflow: how agent work is delegated, claimed, integrated and stopped

A generalized description of the mechanisms I use on a private codebase. Names of files are real; contents are described, not quoted, because they include business detail. Labels (DEPLOYED, TESTED, IMPLEMENTED, PLANNED) are defined in the [README](../README.md).

## Design principles

1. **Files, not chat, are the coordination layer.** Agents read and write documents in the repo. If a decision is not in a file, the next session does not know it.
2. **Constraints beat goals.** Rules are phrased so an agent can check a draft against them ("price text may only come from the plan file", "never print keys") rather than "be careful with pricing".
3. **Evidence before status.** A lane is "done" when a command output or commit shows it, not when the agent says so.
4. **One writer per file, one integrator per release.** Parallelism is only safe where write sets do not overlap.
5. **A human-owned stop.** Anything involving money, outside contact or policy waits for me, and the agent keeps working elsewhere.

## The artifacts

| Artifact | Purpose | Who writes it |
|---|---|---|
| `AGENTS.md` | Binding rules and "where things are". First file any agent reads. | Owner and coordinator; edits are commits |
| `PROJECT_STATE.md` | Current state in fixed sections (production, verified, not deployed, blockers, security, next action). Older sections are marked HISTORICAL when a newer one supersedes them. | Coordinator after each verified change |
| Owner directives | Time-boxed instructions with explicit hard stops, saved verbatim in `docs/`. | Owner |
| Ops state (`ops/`) | Control flags (`PAUSE`, `LOCK`), delegate limits, retry limits, notification rules. | Coordinator |
| Work board (`docs/WORK_BOARD.md`) | Lane and file ownership per machine, plus a claims table edited only through a helper. | Helper script |
| Controller log | Dated, evidence-based reconciliation of every lane. | Coordinator |
| Claims registry | Every public claim with a state, source and re-verify date. | Any lane; audited |

## Delegation

**Coordinator.** Hermes Agent reads the ops state, then the project state, decides the next lane or lanes and delegates. The operating rules it follows:

- Single coordinator; do not create another scheduler or spawn recursive agents.
- At most 8 delegates per run, each with a named output file and no two writers on the same file.
- If `ops/PAUSE` exists, do nothing. If `ops/LOCK` exists and is younger than three hours, exit.
- A failing action is tried at most twice per run; record it as a blocker and move on. Do not rerun tests with unchanged conditions.
- Notify the owner only for an interested reply, an owner-required action, a failure, or a verified milestone. Everything else goes in the file.

**Lanes.** A lane is a scoped job with a short id that appears in commit messages and checkpoint files. Typical scopes: a portal UI change, call summaries, CRM linking, a competitive-gap study, a cost model, a red-team pass, a documentation audit, a deploy. Implementation lanes use Claude Code against the existing test suite; research and QA lanes produce documents and failing tests.

**Two machines.** As documented in October, Machine A builds, tests, deploys and is the only one that holds the outbound-email sender; Machine B (a second workstation) takes outreach and coordination tasks and is forbidden from editing application code, documents owned by A, or running the sender. A later update gave B its own coordinator and a small set of directories it owns. The ownership table lives in the work board, and the same constraints are repeated in the starter prompt for B.

**Claims.** A lane is claimed with `python marketing/work_board.py claim <id> <machine> --push`. The helper pulls with rebase, edits only the board file, commits and pushes; if git refuses, it stops and the claim is not taken. It rejects a claim whose lane belongs to the other machine, and `check` fails on a claim older than 24 hours with no progress note. On a rebase conflict the rule is: abort, never force-push, leave the resolution to Machine A. The helper has a test that creates two clones and races them. **TESTED.**

## Integration

- Lane branches end with a commit whose message says the lane is ready (a "checkpoint" commit). Evidence in the controller log is the commit hash and age.
- One integrator lane merges ready branches into a release branch, runs the full gates and deploys. Lanes that are finished but not merged are reported as such, not as shipped.
- Before the deploy, the previous release image is noted as the rollback target. After it, the live self-check runs.
- Source files may use mixed line endings across machines, so edits are made with line-ending-safe scripts and byte-checked afterwards.

## Verification ladder

The private notes grade every statement with a claim level, lowest to highest: proposed, implemented, unit tested, integration tested, deployed, production verified, proven with real customers. The rule is that a document may not state a level it has not reached. The highest level has not been reached for Call Kettle, and nothing in this repo claims it.

## Handling disagreement between documents

State documents drift. The rule is that the newest dated section at the top of `PROJECT_STATE.md` supersedes older text, and every older "current" statement is marked HISTORICAL when it is superseded rather than deleted. When a document and the code disagree, the code and the audit win, and the document is corrected.

## Rules that exist because something went wrong

- **Orphaned test processes** left behind by an interrupted run caused database-lock errors that looked like regressions. Rule: check for leftover processes before and after every full suite run.
- **A flaky token-budget test** failed once (7,301 against a limit of 7,300) and passed on re-run. I did not accept the re-run: the cause was the wall-clock date and open/closed text in the prompt, reproduced by sweeping 730 clocks over five scripted scenarios, and the fix pins the clock.
- **Price drift.** After the plan ladder changed, older documents still showed retired figures. Rule: one plan file is the source, the audit fails on any price-looking amount not in it, and superseded documents are removed with a recorded recovery list.
- **A second email sender** would re-contact people and blow the shared daily sending limit. Rule: exactly one machine holds the sender and its suppression state, enforced by file ownership and the starter prompt for the other machine.
- **Findings from a live end-to-end run** (no operator alert on an unbound paid checkout, a tenant that was never bound) became regression tests the same day, in a named test file.

## What is not in place

- Agents have no direct access to production secrets. That is a rule and a habit, not a technical sandbox; the main protection is that secrets are held in platform secrets and are never in the repo.
- There is no automated check that an agent obeyed a rule other than where a test or audit encodes it (price text, retired brand, secret leakage, route guards). The rest depends on review.
- Cloud coding sessions are covered by a written guide (one repo, no production secrets, work on a branch, review before merge). I do not claim a running cloud pipeline.
