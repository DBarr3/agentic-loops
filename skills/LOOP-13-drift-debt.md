---
name: loop-drift-debt
loop-id: LOOP-13
description: Architectural drift vs target (layer violations, circular deps, dead APIs), complexity metrics, technical-debt scoring trended over time
domain: Continuous Evaluation
risk-class: read-only
default-debate: FREE-MAD
model-tiers: {scan: cheap, audit: mid, verdict: reasoning}
---

# Mission

Applications drift. Services quietly fuse into monoliths, layers start calling upward, dependencies form cycles, features get reimplemented twice, and APIs die without being buried. This loop:

1. **Captures the target-architecture baseline** — the reference is your own architecture map: e.g. a code-graph/knowledge-graph tool's community structure plus a god-node/high-fan-in inventory, if you maintain one; otherwise your architecture docs/ADRs.
2. **Detects drift** against that reference: microservice→monolith creep, layer violations, circular dependencies, feature duplication, dead APIs, dead services.
3. **Runs complexity analysis** — cyclomatic, cognitive, fan-in/fan-out, dependency depth, inheritance depth, call-graph size — the metrics that predict future bugs — and maintains the god-file watchlist (files > 800 lines or top-decile graph degree).
4. **Emits a numeric technical-debt scorecard**, 0–100 per axis: **Security, Performance, Maintainability, Complexity, Testability, Accessibility**, appended to `_loopstate/LOOP-13/debt-trend.md` so scores trend over time, with a delta report against the previous run.

This loop is strictly read-only: it produces findings and scores, never patches — remediation is routed to the owning loops (LOOP-01/02/etc.) via the artifact's "Recommended next loops".

# Trigger (when the operator runs this)

- Weekly sweep (or your own cadence, commonly alongside your backend and frontend audit loops).
- After any large merge or multi-module refactor lands on main.
- When PROTOCOL.md §4 routes an Architecture Failure from another loop here for follow-up.
- On operator demand: `Run LOOP-13 on <repo>`.

# Inputs (target repo/dir, scope flags)

- `target`: repo (default: your primary repo) + optional scope dir.
- `baseline_ref`: prior run id in `_loopstate/LOOP-13/` to delta against (default: most recent).
- Flags: `--axes=security,performance,maintainability,complexity,testability,accessibility` (default all), `--no-trend` (dry run, no append).

# Preconditions

- [PROTOCOL] read. No branch needed — read-only loop; runs against clean origin/main state.
- If you use a code-graph/knowledge-graph tool, make sure its graph is fresh for the target before this run; refresh it first if stale.
- Your architecture/complexity report (god-node + community/module structure) readable, if you maintain one — this is the drift reference.
- `_loopstate/LOOP-13/debt-trend.md` exists or is created on first run (columns: ISO date, run_id, target, six axis scores, composite).

# Execution DAG (numbered nodes, [P] = parallelizable)

1. Target-architecture baseline capture
2. [P] Structural drift detection (boundaries, layers, cycles)
3. [P] Duplication & dead-surface detection (feature dupes, dead APIs, dead services)
4. [P] Complexity analysis + god-file watchlist
5. Technical-debt scorecard (0–100 per axis)
6. Trend append + delta report vs previous run
7. Adversarial verification + AUDIT-ARTIFACT

Order: 1 → (2 ‖ 3 ‖ 4) → 5 → 6 → 7.

# Node Specs

### Node 1 — Target-architecture baseline capture
- **Action:** build the reference model this run audits against: your code-graph tool's module/community list + inter-module coupling (if you have one), a god-node/high-fan-in inventory (from your architecture report, if you maintain one), and your intended layering/service boundaries (your own runtime topology docs for service boundaries; documented layer rules where present). Snapshot as the run's `target-architecture.json` (modules, allowed inter-module edges, god nodes with degree, declared service boundaries).
- **Tools:** a code-graph/knowledge-graph tool if available, Read on your architecture report and docs.
- **Failure conditions:** graph stale or unreadable → Dependency Failure, refresh or HALT; no prior target doc and graph ambiguous → record baseline as "descriptive, first-run" (drift is then measured from THIS snapshot onward).
- **Output artifact:** `target-architecture.json` + summary table.

### Node 2 — [P] Structural drift detection
- **Action:** compare current dependency edges (a code-graph/knowledge-graph tool if available, plus ast-grep import/call analysis) against the node-1 reference. Detect: **microservice→monolith creep** (modules whose inter-module edge count or shared-state coupling grew past baseline — services merging in practice while separate on paper); **layer violations** (edges pointing the wrong direction: UI→DB direct, handler→handler, lower layer importing upper); **circular dependencies** (SCC detection over the import graph; report each cycle with member files and the cheapest edge to cut).
- **Tools:** a code-graph/knowledge-graph tool if available, ast-grep, `madge`/`pydeps`-class tools where available; cheap tier triage → mid tier confirmation.
- **Failure conditions:** edge data incomplete for a language → mark that surface `unaudited`, list in the confidence block's `unknown`; never extrapolate.
- **Output artifact:** drift findings table (type, from→to, severity, baseline delta).

### Node 3 — [P] Duplication & dead-surface detection
- **Action:** hunt **feature duplication** (near-identical functions/components across modules — a code-graph tool's similar-node queries if available, plus AST shape comparison; confirm same runtime context before flagging true duplication — near-identical code that actually runs in different runtimes, e.g. browser vs. server, may need to diverge and isn't true duplication); **dead APIs** (exported endpoints/routes with zero inbound edges in the call graph and no external-contract annotation); **dead services** (declared services/processes with no live callers or traffic references).
- **Tools:** a code-graph/knowledge-graph tool if available, ast-grep, route-table extraction; Grep only for literal route strings.
- **Failure conditions:** cannot distinguish "dead" from "externally called" → classify `suspected-dead`, require operator confirmation, never assert dead without inbound-edge evidence.
- **Output artifact:** duplication table + dead-surface table with evidence per row.

### Node 4 — [P] Complexity analysis + god-file watchlist
- **Action:** compute per-file and per-function: **cyclomatic complexity, cognitive complexity, fan-in, fan-out, dependency depth, inheritance depth, call-graph size** (fan-in/out and call-graph size from your code-graph tool's degree metric, if available). These metrics predict future bugs — rank the top-20 riskiest files. Maintain the **god-file watchlist**: every file > 800 lines (house limit) or in the top decile of your code-graph tool's degree metric (or an equivalent fan-in ranking); flag watchlist entries that GREW since baseline.
- **Tools:** lizard/radon/complexity-report/eslint-complexity per language, a code-graph/knowledge-graph tool if available, cheap tier.
- **Failure conditions:** analyzer unsupported for a language → compute the graph-derivable subset (fan-in/out, call-graph size) and mark the rest `unmeasured`.
- **Output artifact:** `complexity-report.md` (metrics table, top-20 risk ranking, god-file watchlist with line counts + degree + delta).

### Node 5 — Technical-debt scorecard
- **Action:** score each axis 0–100 (100 = clean) from concrete evidence, each score citing its inputs:
  - **Security** — open findings from LOOP-01/07/10 artifacts, secret-scan status, dependency CVEs.
  - **Performance** — N+1/bundle/perf findings, regression budget status.
  - **Maintainability** — duplication rate from node 3, dead surface count, doc coverage, file-size discipline.
  - **Complexity** — node 4 aggregates vs thresholds (e.g. share of functions with cyclomatic > 10, watchlist growth).
  - **Testability** — coverage %, mutation score from latest LOOP-12 run, untestable-region findings.
  - **Accessibility** — latest LOOP-02/06 a11y findings, WCAG status.
  The score formula per axis is written into the artifact so it is reproducible; missing upstream data lowers confidence, never the score silently.
- **Tools:** Read on `_loopstate` artifacts, mid tier synthesis, reasoning tier for borderline calls.
- **Failure conditions:** an axis has zero upstream evidence → score `N/A` with reason, never invent a number.
- **Output artifact:** scorecard block (axis, score, evidence citations, formula inputs).

### Node 6 — Trend append + delta report
- **Action:** append one row to `_loopstate/LOOP-13/debt-trend.md`, then compute deltas vs `baseline_ref` for every axis, every complexity aggregate, drift-finding counts, and watchlist membership. Any axis dropping ≥ 10 points or any NEW circular dependency = a HIGH finding in its own right. Trend row format (dates ISO, scores 0–100, values below illustrative):

  ```markdown
  | date       | run_id   | target       | Sec | Perf | Maint | Cplx | Test | A11y | composite |
  |------------|----------|--------------|-----|------|-------|------|------|------|-----------|
  | YYYY-MM-DD | r-000001 | your-repo    | 91  | 84   | 72    | 69   | 77   | 88   | 80.2      |
  ```
- **Tools:** Read/Write on `_loopstate/LOOP-13/` only (the sole write surface of this read-only loop; PROTOCOL.md §10: `_loopstate` artifacts are auto-writable).
- **Failure conditions:** prior row schema mismatch → migrate the trend file forward with both schemas preserved, note it.
- **Output artifact:** `delta-report.md` (per-axis Δ, new/resolved drift findings, watchlist joins/leaves, trend direction over last 5 runs).

### Node 7 — Adversarial verification + artifact
- **Action:** run the Adversarial Check below over nodes 2–6; assemble AUDIT-ARTIFACT (PROTOCOL.md §13) with verdict, scorecard, delta report, and "Recommended next loops" routing each finding class to its owning loop (cycles/layer violations → LOOP-01; a11y axis → LOOP-02/06; testability axis → LOOP-12; infra drift → LOOP-07). Write the governance row.
- **Tools:** LOOP-11 inline, reasoning tier.
- **Failure conditions:** reviewer rejects twice → FAIL-with-artifact.
- **Output artifact:** `AUDIT-ARTIFACT.md`.

# Adversarial Check (reviewer persona + what it attacks)

Persona: **the Forensic Architect** — a skeptical principal engineer who assumes the reported architecture is fiction and the scores are flattery. Protocol: FREE-MAD (PROTOCOL.md §11), max 2 rounds. It attacks:

1. **Baseline circularity** — is drift being measured against a reference that was itself captured from the drifted code? First-run descriptive baselines must be labeled as such.
2. **Every `dead` verdict** — demands the zero-inbound-edge evidence and an external-caller check.
3. **Score inflation** — recomputes one axis independently from the cited evidence and rejects the scorecard if it deviates > 5 points.
4. **Delta laundering** — checks that axis formulas did not change between runs to manufacture an improvement (formula changes must be flagged in the delta report).
5. **The confidence block's `unknown` list** and every `unaudited`/`unmeasured` surface.

# Exit Criteria (quantitative, overrides PROTOCOL.md §12)

- Target-architecture baseline captured and snapshotted for the run.
- All three parallel analysis nodes complete; unaudited surfaces explicitly enumerated (not silently skipped).
- **Debt scorecard emitted with all six axes scored 0–100 (or justified N/A), each with cited evidence and reproducible formula.**
- **Trend row appended to `_loopstate/LOOP-13/debt-trend.md` and delta report vs previous run produced** (first run: baseline row, delta marked `initial`).
- Every circular dependency and layer violation reported with file-level evidence and a suggested cut point.
- Zero mutations outside `_loopstate/LOOP-13/` (read-only guarantee verified via `git status` at exit).
- Final confidence ≥ 0.85; governance row written.

# Failure Routing (only deviations from PROTOCOL.md §4)

- Architecture Failure findings do not halt this loop (finding them IS the loop); they are recorded and routed onward in the artifact.
- Security-relevant discoveries (e.g. a dead-but-still-exposed API endpoint) → escalate to Security Failure handling: CRITICAL finding, flag for LOOP-01/LOOP-10, but the loop may complete since it mutates nothing.

# Approval Gates (only deviations from PROTOCOL.md §10)

- None beyond protocol defaults. Entirely read-only + `_loopstate` writes = automatic class.
- All remediation requires the operator to launch the owning loop; this loop never fixes.

# RUN PROMPT

```
Run LOOP-13 (Architectural Drift & Tech-Debt Scoring).
Read PROTOCOL.md (this collection's shared protocol), then
LOOP-13-drift-debt.md, and execute its Execution DAG end-to-end under
protocol rules.
target: <repo> (scope: <dir, optional>)
baseline_ref: default (most recent run in _loopstate/LOOP-13/)
This loop is READ-ONLY: no code changes, no branches; writes only under
_loopstate/LOOP-13/. Refresh your code-graph tool first if stale. Hand me the
AUDIT-ARTIFACT with the six-axis scorecard, the delta report, and the
recommended next loops. Do not fix anything.
```
