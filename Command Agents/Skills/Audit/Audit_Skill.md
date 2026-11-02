---
name: audit-prove-versaterm
description: 'Try to fool the prove-versaterm verifier. Plants one deliberate, user-visible bug at a time in the files the current change touched (or along a flow the user names), has a fresh blind agent prove the change with prove-versaterm without knowing what was planted, scores caught / caught-by-luck / missed, reverts byte-exactly, and reports every miss with a fix to the skill and to the answer key. Use for "audit the verifier", "can prove be fooled", "mutation test the verifier", "is our verification any good", or after creating or heavily editing a check in prove-versaterm.'
argument-hint: 'Plan path, feature slug, or a named flow (e.g., "patients triage manual step")'
---

# Audit Prove Versaterm

A verifier that says PROVED on a broken app is worse than none — in an incident-command product it is false confidence during an active emergency. The only way to know it works is to break the app on purpose and see if it notices.

## Safety first

You are editing product code on purpose:

- **Local only.** Never against a shared, staging or production instance, and never with `STORE_DRIVER=pg` pointed at a non-disposable database.
- **Back up before you plant.** Touched files usually hold uncommitted work, so `git checkout` would destroy it. Copy each file byte for byte to `.prove-it/audits/<UTC timestamp>/originals/<path>` and revert by copying it back.
- **One planted bug at a time.** Plant → check → revert → confirm → next.
- **Confirm every revert**: `(Get-FileHash $backup).Hash -eq (Get-FileHash $file).Hash` must be `True`. If not, stop everything and restore from the backup.
- **Never commit, push or stash a planted bug.** At the end, `git status --porcelain` must match the start, plus receipts under `.prove-it/`.
- **Never plant in** `uploads/`, `react-native/db/migrations/*.sql` (applied migrations are immutable), secrets or CI config.

## 1. Baseline

1. Record `git status --porcelain` and `git rev-parse HEAD`.
2. Work out the target: by default, the files in uncommitted work plus the commits on the branch since `main`. If the user names a flow, the target is that flow and the files it runs through. If the change touches no product code, ask for a flow.
3. Run `prove-versaterm` once on the untouched target. **It must come back `passed`.** If the clean change already fails, any later "catch" means nothing: exclude that check, or run `maintain-prove-versaterm` first.
4. Note from the baseline receipts the flows, surfaces, checks and answer keys used. That is what you hand the blind checker.

## 2. Plan the bugs

Plan **4–8** bugs from [references/mutation-catalogue.md](references/mutation-catalogue.md), spread across layers so the audit finds blind spots rather than hitting one check eight times. Prefer lines the change itself added. Include at minimum:

- one bug the UI hides (API or stored state is wrong, the screen looks fine),
- one bug the API hides (the screen lies, the data is fine),
- one in shared code that an **existing** flow runs through,
- one parity bug (web and RN diverge, or both break the same way via shared code),
- one compliance bug (PHI to localStorage, PII in a log or realtime payload, tenant leak, or a permission check loosened), if the target touches that area.

Each bug must be **real** (a plausible code change, not a syntax error), **user-visible** (a user or an API caller would notice) and **small** (one or two lines). At most one startup-breaking bug — it only tests the health check.

Write the plan as a table before touching anything: id · flow · layer · file · change · what a user would see.

## 3. Plant, check blind, revert

For each bug:

1. **Back up and plant.** Restart the backend via `command-activate` if the change is server-side; Metro hot-reloads web/RN.
2. **Check blind.** Start a fresh subagent (`test-automation-engineer` as verifier, a fresh `senior-engineer` as checker) and give it only: *"Use the `prove-versaterm` skill to prove these flows: `<flows from baseline>`, with these checks and answer keys: `<from baseline>`. Report verdicts with evidence. Do not read `git diff`, `git status`, `git log`, or `.prove-it/audits/`. Keep scratch files inside your run folder."* No hint about the bug, file or layer. If you cannot start a subagent, run only the scripted checks and mark the result **not blind**.
3. **Score**: **caught** (BROKEN, evidence names the planted symptom), **caught by luck** (BROKEN for an unrelated reason), or **missed** (PROVED, or BROKEN with unrelated evidence). Note whether the scripted steps or the agent going off-script caught it. If unsure, have a second fresh agent read the evidence and decide.
4. **Revert** from the backup, confirm the hash, and restart if needed so no planted instance stays alive.

## 4. Report

1. Confirm `git status --porcelain` matches the start.
2. Run `prove-versaterm` once more on the clean target. Any BROKEN now is a **false alarm** — report it too; a verifier that cries wolf gets ignored.
3. Write `.prove-it/audits/<UTC timestamp>/report.md`:

```
# Prove audit: <slug>, <date>
Target: <base sha>..<head sha> (+ uncommitted: yes|no) | flow "<name>"
Score: <caught>/<planted> caught (<n> by scripted steps alone), <n> false alarms

| # | Flow | Layer | Planted bug | Result | Evidence |
|---|---|---|---|---|---|

## Misses
### <id>: <bug>
Why it slipped through: <the check that should have seen it and why it didn't>
Fix to the skill: <exact step, probe or check to add to prove-versaterm>
Fix to the answer key: <plan decision, ADR line, parity-matrix row or CLAUDE.md rule that would name this behaviour, or "none needed">
```

Say plainly what the score means: a handful of plants shows this run's misses, not total coverage. Recommend a different layer mix next time.

Fixes are to the **skill**, never the app. Typical ones: check stored state after the UI says success, read the error body not just the status code, check the second item and the count, list localStorage keys after every PHI flow, verify both themes, scan the backend log for swallowed errors.

## 5. Close the gaps (with the user's OK)

Ask the user whether to apply the fixes. If yes:

1. Edit only `.github/skills/prove-versaterm/`.
2. Re-plant just the missed bugs and re-run the blind check. A fix counts only when the re-run catches the bug. Revert and confirm as before.
3. A miss often means no answer key named the behaviour. Propose the missing line (a plan decision, an ADR addendum, a parity-matrix row or a CLAUDE.md rule) — **never** add or change an answer key without the user's OK.
