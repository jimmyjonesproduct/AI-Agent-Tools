---
name: prove-versaterm
description: 'Prove a Versaterm Command change works by running it, not by reading it. Drives the real surfaces (mock-backend API on :4000, web canvas on :8081 at iPad 1366x1024 in both themes, and the web/RN parity scans), compares what happened against answer keys the builder did not write, and writes receipts. Used by the command-orchestrator verify round (Stage 5b) and directly for "prove it", "verify it works", "did it actually work", "check it end to end". Audits and maintenance live in audit-prove-versaterm and maintain-prove-versaterm.'
argument-hint: 'Plan path or feature slug (e.g., ".versaterm/plans/2026-09-07_patients-maximized.md")'
---

# Prove Versaterm

A test passing means the code agrees with the test's author. A **proof** means the running app agrees with something the author did not write. Every check has three parts:

1. **Surface** — where you observe the behaviour (an HTTP response, a rendered screen, a scan result).
2. **Answer key** — what *should* happen, from a source the builder did not write this round.
3. **Comparison** — a mechanical rule that turns the two into a verdict.

If you cannot name all three, the check is **UNPROVEN** — say so; never round it up to a pass.

## Verdicts (per check)

| Verdict | Meaning |
|---|---|
| **PROVED** | Surface matched the answer key. |
| **BROKEN** | Surface reached, did not match. A real defect. |
| **BLOCKED** | Could not reach the surface (boot failure, missing permission, no data). Not a defect claim. |
| **UNPROVEN** | No answer key or no comparison exists for this behaviour. |

The **run verdict** the checker returns: `passed` (all PROVED, or UNPROVEN by design), `passed+defects` (BROKEN only outside the plan's scope), `ran_wrong` (a BROKEN inside scope), `could_not_boot`, or `unreachable_state` (in-scope checks BLOCKED because the state cannot be produced locally — e.g. a real CAD feed, Keycloak, KMS).

## Roles — never let one agent hold two

- **Setup** boots the stack and proves it is healthy. Nothing else.
- **Verifier** drives the surfaces and writes raw receipts. It does not judge them.
- **Checker** reads receipts + answer keys and assigns verdicts. It did not write the code, the tests or the answer keys.

## 1. Boot (setup)

Run `pwsh -NoProfile -File scripts/boot-check.ps1` — it checks both surfaces and exits 0 when healthy. Only if it fails, boot via `command-activate` and re-check. Healthy means:

- Backend: `GET http://localhost:4000/v1.0.1/health` returns 200.
- Web: `http://localhost:8081` loads with no red-box or console error on first paint.

Boot failure after one retry → `could_not_boot` with the last 40 log lines. Do not attempt a code fix.

## 2. Answer keys (in priority order)

1. The plan's **locked decisions** (`D1…Dn`) and its coverage map.
2. The ADRs the plan names (`docs/adr/`).
3. `docs/parity-matrix.md` and the Standing invariants in `.versaterm/CONTEXT.md`.
4. Hard rules in `CLAUDE.md` (HIPAA no-persist, sentence case, 12px floor, 44px targets, both themes).

**Never** use the new tests in the diff, the impl notes or the implementer's summary as an answer key — the builder wrote them.

## 3. Surfaces and how to drive them (verifier)

| Surface | Drive with | Receipt |
|---|---|---|
| API contract | `Invoke-RestMethod` / `curl` against `:4000` with the dev token, including the If-Match / 409 / 422 / 428 paths the plan names | request + status + body (redact names, DOB, tag-to-identity) |
| Web UI | browser tools at **1366×1024**, in **light and dark**; reach the plan's entrypoints from the dock, not by deep link | `readPage` excerpt + one screenshot per state |
| Accessibility | `readPage` roles/names, tab order, Escape closes, live-region text | the accessible names and the announced text |
| HIPAA no-persist | after the flow, list `localStorage` keys via `runPlaywrightCode` | the key list (must hold no PHI/PII) |
| Parity | quote the post-impl gate report's parity result; rerun `node scripts/test-run.js parity` only if `git diff` changed since the gate (e.g. a verify-round fix) — the runner's cache makes an unchanged rerun free anyway | the summary line; KNOWN RED is expected, anything else is BROKEN |
| RN | not driveable here — mark RN-only behaviour **UNPROVEN (no device)** unless a parity suite proves the mount | — |

Write each receipt to `.prove-it/runs/<slug>/<NN>-<check>.md` with: check id, surface, exact steps, raw observation, timestamp and `git rev-parse HEAD`. Receipts are PII-scrubbed — ids and versions only.

## 4. Compare (checker)

For each check, quote the answer-key line, quote the receipt line, and give the verdict. Write `.prove-it/runs/<slug>/verdict.md` with the per-check table and the single run verdict.

## 5. Pick the checks

From the plan's coverage map, pick at most **8** checks: every locked decision with an observable effect, every new web entrypoint, every new API error path, one both-themes check per new screen, and the HIPAA no-persist check whenever patient or crew data is touched. Prefer the check that would catch the most expensive failure during an active incident.

## Keeping the checks honest

- **Audit** — to test whether these checks can be fooled, use the `audit-prove-versaterm` skill (blind planted-bug audit).
- **Maintain** — when checks drift or a bug slipped past a PROVED run, use the `maintain-prove-versaterm` skill. Never rewrite an answer key to make a check pass.
