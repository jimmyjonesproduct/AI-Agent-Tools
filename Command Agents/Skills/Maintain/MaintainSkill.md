---
name: maintain-prove-versaterm
description: 'Improve the prove-versaterm verifier from what its past runs learned. Reads the receipts in .prove-it/runs/ (failed and blocked steps, retries, commands that finally worked, failures unrelated to the change, recurring UNPROVEN items) and any bug that slipped past a PROVED run, then fixes drifted steps, writes in working commands, adds the check that would have caught a slipped bug and covers new surfaces. Never rewrites an answer key and never edits product code. Proves every edit live. Use for "update the verifier", "prove is out of date", "prove missed this bug", "prove keeps failing on unrelated steps", or after a big merge from main.'
argument-hint: 'Optional: a bug that slipped past prove, or a run slug to start from'
---

# Maintain Prove Versaterm

A verifier goes stale the day the app changes: a route moves, a testID is renamed, a dock leaf is gated, a new module ships with nothing in the skill that reaches it. A stale verifier is worse than none, because the next agent trusts it. Every prove run leaves receipts that show where it struggled; this pass folds them back into the skill.

## Rules for the whole pass

- **Edit only `.github/skills/prove-versaterm/`.** Never change product code here.
- **When app and skill disagree, decide which is wrong.** The skill describing something the app no longer does is either **drift** (the app changed on purpose — fix the skill) or a **product bug** (the app broke — report it). Read the commit that changed it (`git log -p -- <file>`). If still unsure, ask the user.
- **Answer keys are not steps.** Plan locked decisions, ADRs, `docs/parity-matrix.md`, `.versaterm/CONTEXT.md` invariants and `CLAUDE.md` hard rules say what right looks like. An answer key that disagrees with the app is **reported**, never rewritten.
- **Prove every edit live** before finishing. An unproven edit is more drift.
- **Old verdicts are leads, not evidence.** Only this pass's live runs count.
- **Never loosen a check to make a flaky step pass.** Record the failing run, then decide whether the app has a race or the check is ambiguous.

## 1. Ask first

In your first message, ask whether anyone reported a bug that prove should have caught. For each, find the receipt of the run that said PROVED on that change and work out why: the flow was not in scope, the step looked at the wrong thing, or seed data hid it.

## 2. Read the past runs

List `.prove-it/runs/` newest first. Start from runs since the skill last changed (`git log -1 --format=%cI -- .github/skills/prove-versaterm`). For each receipt note:

- **Failed or BLOCKED steps** and their stated cause.
- **Retries that worked** — the port, wait, token, selector or testID that finally succeeded.
- **Failures unrelated to the change** — the clearest sign of drift.
- **Recurring UNPROVEN** lines (e.g. RN-only behaviour, real CAD feed, Keycloak) and the unlock each names.
- **Scope gaps** — a changed path or dock leaf the run reached with no recipe in the skill.
- **Answer keys** — flows repeatedly marked "no answer key" mean plans are not recording locked decisions; say so.

Also read the latest `.prove-it/audits/*/report.md` — its misses are the highest-priority inputs.

If there are no receipts, say so; then the only inputs are reported bugs and a fresh run on the current change.

## 3. Sort what you found

Each edit must cite the receipt or bug that calls for it:

| Finding | Action |
|---|---|
| **Drifted step** (route, testID, dock path, theme toggle moved on purpose) | Fix the step. |
| **Working command** found on retry | Write it into the skill now. |
| **Slipped bug** | Add the check that would have caught it. If the flow had no answer key, propose one (plan decision, ADR line, parity-matrix row) for the user to approve. |
| **New surface** (new module, dock leaf, route, modal) | Add a drive recipe under the surfaces table. Only call it new if you can name the file that adds it. |
| **Removed surface** | Remove its recipe, citing the commit. |
| **Launch change** (env var, port, `STORE_DRIVER`, migration) | Update Boot and health checks; coordinate with `command-activate`. |
| **Answer key disagrees with the app** | Do not touch. Report with evidence and the commit. |
| **Product bug** | Record with evidence for the user. |

## 4. Edit and prove live

Make the edits, then use `prove-versaterm` exactly as written: boot, health check, and drive every step you touched. When a step fails:

1. Re-run the health check. If the stack is broken, fix Boot first and restart.
2. App works but the step is wrong → drift: fix and re-drive.
3. App is really broken → product bug: record and move on.
4. Surface unreachable → record the exact prerequisite and route tried.

A new check for a slipped bug counts only if it **fails on the broken code**. If the broken version is in history, check it out in a scratch worktree (`git worktree add $env:TEMP\prove-wt <sha>`) and run the check there; remove the worktree after. Otherwise recommend `audit-prove-versaterm` plant that bug next time.

Stop every process you started and confirm the receipts are still present.

## 5. Report

- **Outcome:** `current` (nothing needed changing), `updated` (every change proved live) or `blocked` (what stopped you).
- Each skill change, one line, with the receipt or bug that called for it.
- Product bugs found, with evidence.
- Answer keys that disagree with the app, and proposed answer keys, for the user to approve.
- Surfaces left UNPROVEN and why.

Leave the skill changes uncommitted unless the user asks.
