# Evidence map

Each major claim in the [README](../README.md), the source it rests on, and its status. Sources are in the private codebase unless marked public. Private files are described rather than named, so each claim stays traceable by description in a walkthrough. Dates are when the source was written or the command was run.

| Claim | Source | Status |
|---|---|---|
| One rule file for any tool; "CONTINUE" resumes the recorded next action | The agent rules file (`AGENTS.md`, opening section; a version is in the public snapshot) | IMPLEMENTED |
| Coordinator caps: one coordinator, up to 8 delegates per run, named outputs, 2 retries, `PAUSE`, 3-hour `LOCK` | Private operating notes for the coordinator (2026-10-15) | IMPLEMENTED |
| Work-board claim protocol and file ownership | The private work board, its claim helper script and the helper's test file (includes a two-clone race test) | TESTED |
| Lanes with checkpoint commits, one integrator, status reconciled against evidence | The private controller log (2026-10-28 run) and commit messages | IMPLEMENTED |
| Claim-level ladder (proposed through proven with real customers) | A private operating-notes file that defines the claim levels | IMPLEMENTED |
| Fallback model order | The agent rules file, fallback-model rule | IMPLEMENTED as policy; not claimed as exercised |
| 5,342 backend tests collected | `pytest --collect-only -q` in the backend virtualenv, 2026-11-05 | Command output; collection only |
| 7,516 passed, 5 skipped, 1 xfailed | Commit message of the DEMO8 checkpoint, 2026-10-29 | Recorded lane result; not re-run for this write-up |
| 4,134 passed, 1 skipped, 1 xfailed | The private project state document (2026-10-15) | Recorded result |
| About 1,550 public test items | `pytest --collect-only -q` in the public snapshot, 2026-11-05 (2 modules fail to import without private files) | Command output; collection only |
| CI runs builds, audits and both suites on push and PR | The private repo's CI workflow; the public snapshot has its own `.github/workflows/ci.yml` | IMPLEMENTED. I do not state the current pass or fail status of either |
| Heartbeat against API health and the public site: every 15 minutes in the private repo, every 6 hours in the public snapshot | `.github/workflows/heartbeat.yml` in each repo (cron `*/15 * * * *` private, `0 */6 * * *` public) | DEPLOYED (private); public workflow IMPLEMENTED |
| Live self-check of 27-28 checks | Private project state notes (deploy notes, October); `backend/scripts/selfcheck.py` (public); I did not re-run the count for this write-up | DEPLOYED |
| Flaky test root-caused to wall-clock dependence | A private write-up of the flaky-test investigation (2026-10-15) | TESTED |
| 75 resilience tests, 18 failing before fixes | A private resilience audit and `backend/tests/test_resilience_chaos.py` (public). The audit was written 2026-10-04 and the test file was first committed the same day; the public snapshot also contains `test_resilience_chaos.py` | TESTED |
| 40-scenario red team with failing-before, passing-after tests | A private red-team report (2026-10-13; offline tests with a scripted model; live part was 16 single-turn lines) | TESTED |
| Production end-to-end run with PASS, FAIL-then-FIXED and NOT DONE rows | A private end-to-end proof record (2026-10-22) | Recorded run |
| Tool-use loop, 4-iteration cap, reply cap, 6 s timeout with one retry, claim guard, ownership checks, `BEGIN IMMEDIATE` booking | `backend/app/agent.py`, `cost_guard.py`, `tools.py`, `storage.py` in the public snapshot | IMPLEMENTED, TESTED (public) |
| Strict `Policy` schema and load-time validators | `backend/app/config.py` (public). Only the `Policy` model is `extra: forbid`; the tenant config as a whole is not, and hours, timezone, webhook URL and AI-disclosure validators run at load | TESTED (public) |
| Concurrency race: 42 admitted against a ceiling of 24 at 64 callers; exact ceiling after the fix; two lost transcript writes in an earlier run | A private concurrency analysis (2026-10-06), its load-test script and the regression test it names | MEASURED locally with a stubbed model, not on the Fly VM |
| Single VM, SQLite on one volume, one process | Same analysis, section 2, from the repo's Fly and Docker configuration | DEPLOYED |
| Daily backup, restore test, volume snapshots | Project notes of 2026-09-29 and 2026-09-30; retention constants cited in the claims registry | IMPLEMENTED and TESTED (restore test); I have not re-run it for this write-up |
| Security controls C1-C19 and their evidence types (production probe, test, code) | A private security self-review (2026-09-30; self-review, not an audit) | Mixed, labeled per control in the source |
| Adversarial persona run: 17 of 18 passed before a fix | The same security self-review, control C10; the source says a rerun on final code was still owed | TESTED, rerun not recorded in that document |
| Worst-case-loss model | A private worst-case-loss write-up and `backend/tests/test_worst_case_cap.py` (public) | MODELED only; no customer usage exists to measure |
| Outbound compliance policy | A private outbound-compliance policy note (2026-10-01; states it is not legal advice) | IMPLEMENTED as policy |
| Written guide for cloud coding sessions with least privilege | A private guide for cloud coding sessions (2026-10-02; says no cloud session had been launched) | PLANNED / guide only |
| Scheduled agent jobs (health alert, weekday brief, weekly watchdog) | The private controller log (2026-10-28) | DEPLOYED as of that date |
| About 870 commits | `git log --oneline` in the private repo, 868 on 2026-11-05 | Command output |

## Not claimed, and why

- **Customers, revenue, uptime, call volume.** None are documented as proven, so none are stated.
- **A dedicated role for Codex.** It is installed and has local session records, but no written lane or decision records what it did on this project, so I list it as used and describe nothing more.
- **Public CI status.** The public snapshot's CI result changes over time and is not part of the evidence here.
