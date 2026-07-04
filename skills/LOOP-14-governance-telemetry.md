---
name: loop-governance-telemetry
loop-id: LOOP-14
description: Audits the autonomous system itself — governance ledger trends, retry/rollback rates, token efficiency, benchmark suite runs, agent trace quality
domain: Runtime Observability
risk-class: read-only
default-debate: FREE-MAD
model-tiers: {scan: cheap, audit: mid, verdict: reasoning}
---

# Mission

Every other loop audits application code. LOOP-14 audits the AUTONOMOUS SYSTEM — the loops, the agents, the debate machinery — and continuously answers five questions:

- **Is the agent improving?**
- **Is it looping?**
- **Is it degrading?**
- **Is it wasting tokens?**
- **Is it becoming less reliable?**

It parses the governance ledger and every run's checkpoint traces, computes trends for every PROTOCOL.md §8 metric, audits trace quality against PROTOCOL.md §9, runs infinite-loop and degradation detectors with explicit thresholds, re-executes a fixed benchmark suite of historical bugs to measure whether the agents themselves have regressed, and reviews whether agent effort follows the priority ladder (critical security > production bug > feature > docs > cleanup). Output: a GOVERNANCE-REPORT with a verdict per loop and concrete recommendations — the primary input to LOOP-15. Without this loop, we are auditing code but not the autonomous system itself.

# Trigger (when the operator runs this)

- Continuous cadence: after every N loop runs (default N = 5) or weekly, whichever comes first (a common pattern: run this continuously, then your self-optimization loop after).
- Immediately after any loop terminates HALTED-Security or exhausts its retry budget.
- Before any LOOP-15 self-optimization run (LOOP-15 requires a fresh GOVERNANCE-REPORT ≤ 7 days old).
- On operator demand: `Run LOOP-14`.

# Inputs (target repo/dir, scope flags)

- `ledger`: `_loopstate/governance-ledger.md` (one row per loop run, PROTOCOL.md §8 columns).
- `traces`: all `_loopstate/<loop-id>/<run-id>/` checkpoint dirs (node JSONs + trace records per PROTOCOL.md §9).
- `benchmark_suite`: `_loopstate/LOOP-14/benchmark-suite.md` — the fixed suite (see node 5); created on first run, thereafter append-only.
- `window`: analysis window (default: last 20 runs or 30 days).
- Flags: `--no-benchmark` (trend-only quick pass), `--loop=<LOOP-ID>` (single-loop deep dive).

# Preconditions

- [PROTOCOL] read. Read-only loop; writes only under `_loopstate/LOOP-14/`.
- Governance ledger exists with ≥ 3 rows (fewer → loop runs in "bootstrap" mode: trace audit + benchmark seeding only, trends marked `insufficient-n`).
- Benchmark suite exists or this run seeds it (node 5).
- No other loop currently mid-run against the same `_loopstate` tree (avoid reading half-written checkpoints; check for lock/mtime churn).

# Execution DAG (numbered nodes, [P] = parallelizable)

1. Ledger + checkpoint ingest and normalization
2. [P] Metric trend computation (all PROTOCOL.md §8 metrics)
3. [P] Trace-quality audit (PROTOCOL.md §9 completeness)
4. Infinite-loop / degradation detectors (thresholded flags)
5. Benchmark suite run (agent regression test)
6. Agent resource scheduling review (priority ladder)
7. Per-loop verdicts + GOVERNANCE-REPORT assembly
8. Adversarial verification + AUDIT-ARTIFACT

Order: 1 → (2 ‖ 3) → 4 → 5 → 6 → 7 → 8.

# Node Specs

### Node 1 — Ledger + checkpoint ingest
- **Action:** parse every governance-ledger row in `window` into a normalized table (loop-id, run-id, ISO date, all §8 metrics); walk every checkpoint dir, indexing node JSONs and trace records; reconcile — every ledger row must have a checkpoint dir and vice versa. Orphans are themselves findings: a loop that ran without emitting governance data is a governance failure.
- **Tools:** Read, Glob over `_loopstate/`, cheap tier.
- **Failure conditions:** malformed rows → quarantine + parse the rest (never let one bad row kill the audit); > 20% of rows malformed → HIGH finding "ledger discipline collapse".
- **Output artifact:** `normalized-runs.json` + reconciliation table.

### Node 2 — [P] Metric trend computation
- **Action:** for EVERY PROTOCOL.md §8 metric — **success_rate, retry_count, debate_rounds, tool_success_pct, human_overrides, tokens_spent/tier_mix, hallucinations_caught, false_positive_rate, mttr, rollbacks** — compute per-loop and fleet-wide: current value, window mean, direction over last 3/5/10 runs, and variance. Token efficiency additionally normalized per finding produced (tokens-per-verified-finding is the real cost metric; raw tokens alone rewards loops that do nothing). Tier-mix drift flagged when reasoning-tier share grows without a matching confidence-rejection rate — escalation without cause, i.e. waste.
- **Tools:** computation over the node-1 table, cheap tier.
- **Failure conditions:** metric absent from older rows → compute on the available subset, mark `partial-n`.
- **Output artifact:** `trend-matrix.md` (metric × loop × direction, with run-series values).

### Node 3 — [P] Trace-quality audit
- **Action:** sample ≥ 30% of trace records per loop (100% for loops flagged by node 2) and grade against PROTOCOL.md §9: task_id → node → agent-role chain present? reasoning_summary present and ≤ 5 lines (present but vacuous — "did the task" — counts as MISSING)? tools_used, duration, tokens, confidence_block.confidence, result all recorded? Compute a trace-completeness score per loop (fields present / fields required). Incomplete traces are how a degrading system hides — sustained incompleteness is a reliability signal, not paperwork.
- **Tools:** Read on checkpoint dirs, cheap tier grading, mid tier for vacuousness calls.
- **Failure conditions:** a loop's checkpoint dir empty despite ledger rows → HIGH finding "flying blind".
- **Output artifact:** trace-quality table (loop × completeness % × worst-missing-field).

### Node 4 — Infinite-loop / degradation detectors
- **Action:** apply thresholded detectors over node 2+3 outputs; every trip = a flag with evidence:
  - **LOOPING** — retry_count trending up 3 consecutive runs for the same loop; OR any run hitting max retries on ≥ 2 nodes; OR identical failure-class on the same node across 3 runs (it is re-fighting the same battle).
  - **DEGRADING** — success_rate declining over 5 runs; OR false_positive_rate up 3 runs; OR trace completeness falling.
  - **WASTING** — tokens-per-verified-finding up ≥ 50% vs window mean; OR debate_rounds pegged at budget every run (debate theater); OR reasoning-tier share up without rejection-rate cause.
  - **LESS RELIABLE** — rollbacks up 3 runs; OR human_overrides rising; OR mttr worsening 3 runs; OR hallucinations_caught rising while false_positive_rate rises (generator AND verifier drifting together).
- **Tools:** rules over the trend matrix, mid tier for causal notes.
- **Failure conditions:** conflicting signals (improving on one axis, degrading on another) → report both, never average away a warning.
- **Output artifact:** `detector-flags.md` (flag, loop, evidence rows, threshold tripped).

### Node 5 — Benchmark suite run
- **Action:** maintain and execute the fixed benchmark: N historical bugs (target N ≥ 10, grown over time toward a 100-bug pattern) drawn from past session records and closed loop findings — each entry: bug description, repo+SHA where it existed, the verified fix, and which loop/node should catch it. Re-run the relevant loop's detection nodes (read-only scan portions only) against the historical state (detached worktree at the recorded SHA, or preserved fixture) and score: found? correctly classified? fix proposal matches the verified fix's substance? Compare against previous benchmark runs — **any bug previously caught but now missed = an agent regression, severity HIGH**. First run: seed the suite from `_loopstate` artifacts + memory records, no scoring.
- **Tools:** git worktree (detached, read-only), the target loops' scan nodes, mid tier scoring.
- **Failure conditions:** historical state unreconstructable for an entry → mark `stale-benchmark-entry`, propose a replacement, never silently shrink the suite.
- **Output artifact:** `benchmark-report.md` (per-bug found/missed/classified, score vs previous runs, regression list).

### Node 6 — Agent resource scheduling review
- **Action:** reconstruct from ledger timestamps + run targets what the fleet actually spent effort on in `window`, and compare against the priority ladder: **critical security > production bug > feature > docs > cleanup**. Flag inversions (e.g. cleanup loops consuming reasoning-tier tokens while a CRITICAL security finding sat unaddressed > 7 days; observability runs starving pre-release sweeps). Recommend a concrete next-cadence schedule (which loops, what order, which tier) consistent with the ladder.
- **Tools:** node-1 table, open-findings scan across artifacts, mid tier.
- **Failure conditions:** cannot determine finding-open durations → mark the scheduling verdict `low-confidence`.
- **Output artifact:** scheduling review (actual vs ladder, inversions, recommended cadence).

### Node 7 — Per-loop verdicts + GOVERNANCE-REPORT
- **Action:** assign each loop a verdict — **IMPROVING / STABLE / LOOPING / DEGRADING / WASTING / INSUFFICIENT-DATA** — justified by specific trend rows, detector flags, trace scores, and benchmark results; answer the five mission questions fleet-wide in one summary block; attach ranked recommendations, each tagged `for-LOOP-15` (loop-MD/procedural change proposals) or `for-operator` (cadence, budget, rail decisions).
- **Tools:** reasoning tier.
- **Failure conditions:** verdict unsupported by at least 2 independent evidence sources → downgrade to INSUFFICIENT-DATA rather than guess.
- **Output artifact:** `_loopstate/LOOP-14/<run-id>/GOVERNANCE-REPORT.md` — the primary input artifact for LOOP-15.

### Node 8 — Adversarial verification + artifact
- **Action:** run the Adversarial Check below over the report; assemble AUDIT-ARTIFACT (PROTOCOL.md §13); write this run's own governance row — LOOP-14 measures itself and appears in its own next ledger window.
- **Tools:** LOOP-11 inline, reasoning tier.
- **Failure conditions:** reviewer rejects twice → FAIL-with-artifact.
- **Output artifact:** `AUDIT-ARTIFACT.md`.

# Adversarial Check (reviewer persona + what it attacks)

Persona: **the Metrics Skeptic** — an auditor who assumes every metric is gamed, every trend is survivorship bias, and the fleet grades its own homework. Protocol: FREE-MAD (PROTOCOL.md §11), max 2 rounds. It attacks:

1. **Goodhart effects** — are loops optimizing the ledger instead of the work (e.g. success_rate high because exit criteria were quietly loosened — cross-checks exit-criteria text in loop MDs against git history)?
2. **Denominator games** — tokens-per-finding improved by inflating trivial findings; false_positive_rate improved by finding less.
3. **Benchmark contamination** — were benchmark bugs or their fixes present in agent context during scoring?
4. **Verdict charity** — re-derives one loop's verdict from raw rows and rejects the report if it disagrees.
5. **Missing-data laundering** — `INSUFFICIENT-DATA` used to avoid an unflattering DEGRADING call.
6. **The confidence block's `unknown` list first**, per PROTOCOL.md §3.

# Exit Criteria (quantitative, overrides PROTOCOL.md §12)

- 100% of ledger rows in window ingested or explicitly quarantined; ledger↔checkpoint reconciliation table complete.
- **Trend computed for all 10 PROTOCOL.md §8 metrics** per loop and fleet-wide (or marked `partial-n`/`insufficient-n` with row counts).
- Trace-quality score emitted per loop with ≥ 30% sampling (100% for flagged loops).
- All four detector families (LOOPING / DEGRADING / WASTING / LESS-RELIABLE) evaluated; every tripped threshold documented with evidence.
- **Benchmark suite executed (or seeded on first run): benchmark regression count = 0, or every regression is a HIGH finding in the report.**
- Scheduling review delivered with explicit ladder comparison.
- GOVERNANCE-REPORT written with a verdict for every loop that ran in window + recommendations tagged for-LOOP-15 vs for-operator.
- Zero writes outside `_loopstate/LOOP-14/`; final confidence ≥ 0.85; governance row written.

# Failure Routing (only deviations from PROTOCOL.md §4)

- Evidence of a loop violating protocol safety rails (e.g. a mutation committed by a read-only loop, chaos touching production hosts) → treat as **Security Failure**: HALT, CRITICAL artifact, surface to operator immediately — do not wait for the report.
- Benchmark node worktree checkout failure → Tool Failure, retry once, then run remaining entries and report the gap; never fabricate a benchmark score.

# Approval Gates (only deviations from PROTOCOL.md §10)

- None beyond protocol defaults — fully read-only + `_loopstate/LOOP-14/` writes.
- LOOP-14 recommends; it never changes loops (that is LOOP-15, operator-gated) and never changes cadence on its own.

# RUN PROMPT

```
Run LOOP-14 (Runtime Governance & Telemetry).
Read PROTOCOL.md (this collection's shared protocol), then
LOOP-14-governance-telemetry.md, and execute its Execution DAG end-to-end
under protocol rules.
window: default (last 20 runs / 30 days)
This loop audits the AUTONOMOUS SYSTEM, not app code. READ-ONLY: writes only
under _loopstate/LOOP-14/. Answer the five questions (improving? looping?
degrading? wasting tokens? less reliable?) with evidence, run the benchmark
suite, and hand me the GOVERNANCE-REPORT with per-loop verdicts and the
recommendations split for-LOOP-15 vs for-operator. Change nothing.
```
