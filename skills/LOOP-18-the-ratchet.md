---
name: loop-the-ratchet
loop-id: LOOP-18
description: Single-variable verified tuning primitive — measure, one isolated change, re-measure, weld (permanent, irreversible) or retry a different change
domain: Verified Incremental Tuning
risk-class: branch-mutating
default-debate: FREE-MAD
model-tiers: {scan: cheap, audit: mid, verdict: reasoning}
---

# Mission

A mechanical ratchet: a gear that turns one direction only, one tooth at a time, and once a tooth is pinned it cannot slip back. LOOP-18 is that mechanism applied to tuning a single parameter, threshold, or config value: **measure → one change → re-measure → weld-or-retry**. It exists to make incremental improvement to a metric provably monotonic — every welded turn is a verified, irreversible gain (or verified no-regression), never a guess and never a batch of guesses stacked on top of each other.

This loop is deliberately narrower and more mechanical than LOOP-15 (Self-Optimization). LOOP-15 is the meta-loop that decides *what* to improve about the loop system itself and *why*; LOOP-18 is the primitive any loop reaches for when it already knows *which single lever* to turn (a retry count, a cache TTL, a timeout, a threshold, a rate limit, a prompt-cache boundary) and needs a safe, verified way to turn it. LOOP-18 has no opinion about strategy — it only guarantees that whatever single change is attempted gets measured honestly, and that nothing is kept unless proven.

# Trigger (when the operator runs this)

- Invoked BY another loop as an inner sub-routine, not run standalone in the common case: any loop in this collection that needs to safely tune one parameter/threshold/config value calls out to LOOP-18 rather than hand-rolling its own measure-and-commit logic (e.g. LOOP-15 turning a generalized pattern into a tighter exit-criterion value; LOOP-08 tuning a CI timeout; LOOP-12 tuning a fuzz corpus size; any loop tuning a retry budget or cache TTL against PROTOCOL.md §5/§7 defaults).
- On direct operator demand when a single well-defined metric and a single well-defined lever are already known: `Run LOOP-18 on <target> (metric: <metric>, lever: <parameter>)`.
- Never triggered when the "improvement" isn't reducible to one variable — that's a design or refactor task, not a ratchet turn.

# Inputs (target repo/dir, scope flags)

- `target`: repo root or module the metric is measured against.
- `metric`: the single measurable quantity to improve (latency, token spend, cache-hit rate, error rate, pass rate — anything with a reproducible measurement method). Must be named by the caller; LOOP-18 does not invent metrics.
- `lever`: the single parameter/threshold/config value this run is allowed to touch. Exactly one, named up front.
- `direction`: which way is "better" (lower/higher) and the minimum effect size that counts as real (see Exit Criteria).
- `harness`: the exact measurement command/method to use for both baseline and every re-measure (same script, same inputs, same environment — non-negotiable, see Preconditions).
- Flags: `--max-attempts=N` (default 5, see diminishing-returns halt), `--tolerance=<value>` (noise band; caller-supplied or computed in node 1).

# Preconditions

- [PROTOCOL] read. Loop branch `loop/LOOP-18-<date>` created before node 2 (§6).
- A `harness` that is reproducible: same measurement method, same fixture inputs, same environment for baseline and every subsequent re-measure. If no such harness exists, the loop HALTS at node 1 rather than substituting an estimate, a badge value, or a prior run's number — a baseline that wasn't just re-derived under identical conditions is not trustworthy, and this loop refuses to ratchet against a number it cannot personally reproduce.
- `lever` scope confirmed as exactly one variable — if the caller hands LOOP-18 more than one thing to change at once, HALT and return to caller: this loop turns one tooth per call, not several.
- Working tree clean at node 1 (no uncommitted drift that could contaminate the baseline measurement).

# Execution DAG (numbered nodes, [P] = parallelizable)

1. Baseline measurement (re-derive, never trust a stale/estimated number)
2. One isolated change
3. Re-measurement (identical harness)
4. Decision fork: verified improvement/no-regression → node 5; regression or inconclusive → node 6
5. Weld (permanent commit, point of no return)
6. Diminishing-returns / halt check → loop back to node 2 with a DIFFERENT change, or halt

No parallel nodes — the ratchet is inherently sequential: each turn's outcome gates whether and how the next turn happens. Node 4 branches; node 6 either routes back to node 2 or terminates the loop.

# Node Specs

**Node 1 — Baseline measurement.**
Action: run `harness` against `target` exactly as specified, with no changes applied, and record the baseline value plus a computed or supplied `tolerance` (a statistical/noise band — e.g. run the harness N≥3 times if it has any variance and take the tolerance from the observed spread; a single-run "eyeball" number is not a baseline). Refuse to proceed on any baseline sourced from a badge, a dashboard snapshot, a prior loop's cached number, or an estimate — it must be re-derived here, now, under the stated conditions.
Tools: Bash (harness execution), Read (harness definition).
Failure conditions: harness missing/non-reproducible → HALT (Dependency Failure, per PROTOCOL.md §4) — this is a hard stop, not a degrade-and-continue; baseline variance so high that no realistic single change could clear the noise floor → HALT and return to caller with a `metric-not-tunable-at-this-resolution` finding.
Output artifact: `node-1-baseline.json` (value, tolerance, harness invocation, N runs, confidence block).

**Node 2 — One isolated change.**
Action: apply EXACTLY ONE change to `lever` on the loop branch. Never batch multiple changes into one turn — the entire guarantee of the ratchet depends on there being exactly one suspect if the next measurement regresses. If a prior turn in this same run failed (routed from node 6), the change here MUST be different from every previously-attempted change in this run; retrying an identical failed change is a Logic Failure, not a retry.
Tools: Edit, Bash (git), a code-graph/knowledge-graph tool if available to confirm the lever's blast radius before touching it.
Failure conditions: change touches more than the declared `lever` (scope violation) → reject and re-scope; identical change to a prior failed attempt in this run → reject (Logic Failure per PROTOCOL.md §4), pick a different change.
Output artifact: `node-2-change-<attempt>.diff` + rationale (what value, what was tried before, why this one is expected to move the metric).

**Node 3 — Re-measurement.**
Action: run `harness` again, byte-for-byte identical method to node 1 — same script, same inputs, same environment, same N-run protocol if variance-sampled. Any deviation in method invalidates the comparison; if the environment drifted (different machine load, different dataset snapshot, different dependency versions) since node 1, re-run node 1's baseline fresh rather than compare across mismatched conditions.
Tools: Bash (harness execution, identical invocation to node 1).
Failure conditions: harness invocation differs from node 1 in any recorded parameter → Tool Failure, redo with corrected invocation before any decision is made; environment drift detected → re-baseline (back to node 1), do not compare stale-vs-fresh.
Output artifact: `node-3-remeasure-<attempt>.json` (same schema as node-1-baseline.json, delta vs baseline, delta vs tolerance).

**Node 4 — Decision fork.**
Action: compare the node-3 value against the node-1 baseline using the node-1 tolerance band. Classify as: **verified improvement** (delta beyond tolerance, in the declared `direction`), **verified no-regression** (delta within tolerance, effectively flat — still weldable if the caller's exit criteria accept neutral turns, e.g. a safety/clarity change with no expected metric cost), **regression** (delta beyond tolerance in the wrong direction), or **inconclusive** (delta smaller than tolerance but ambiguous sign, or measurement instability detected). Improvement/no-regression routes to node 5; regression/inconclusive routes to node 6. Noise vs real signal is decided ONLY by the tolerance band computed in node 1 — never by eyeballing one run's number.
Tools: none beyond arithmetic on node-1/node-3 artifacts; reasoning tier only if the tolerance band itself is disputed.
Failure conditions: tolerance band undefined or zero-width (measurement had no variance sampling) → treat as Context Overflow, go back and re-derive tolerance from a proper multi-run baseline before deciding anything.
Output artifact: `node-4-decision.md` (classification + numeric evidence).

**Node 5 — Weld (permanent commit).**
Action: commit the change on the loop branch as PERMANENT: `loop(LOOP-18): weld — <lever>=<value>, metric <metric> Δ<delta> (tolerance <tol>)` with the full confidence block and the node-1/node-3 evidence in the commit body. A weld is the point of no return for THIS loop — once welded, a later turn of the same ratchet run (or a future LOOP-18 run) MUST NOT silently reopen, retry, or revert it. If a welded change is later discovered to have been wrong, that is explicitly not this loop's problem to auto-correct: the fix is a brand-new ratchet cycle where the single "one change" is the revert itself, remeasured and welded on its own merits. The ratchet never reaches backward into its own history to undo a weld — it only ever adds forward turns, including a turn that happens to be a revert.
Tools: git (commit only — no `git reset`/force-push against a welded commit from within this loop), Write (changelog entry).
Failure conditions: commit fails (dirty tree, hook rejection) → Tool Failure, retry once per PROTOCOL.md §5, then HALT rather than force past a hook.
Output artifact: welded commit SHA + `node-5-weld.md` (lever, value, verified delta, evidence links) + loop terminates as PASS for this turn.

**Node 6 — Diminishing-returns / halt check.**
Action: on regression or inconclusive result, increment the attempt counter and check it against `--max-attempts` (default 5). If under budget: discard the failed change (revert on the loop branch — this is a plain working-tree revert of an UN-welded change, not a violation of the weld-is-permanent rule, since nothing was welded), route back to node 2 requiring a genuinely different lever value/strategy than every prior attempt this run. If at or over budget: HALT the loop as `FAIL-with-artifact — diminishing returns`, on the theory that a metric that resists N distinct honest attempts is itself a finding (the lever may be maxed out, the metric may be bottlenecked elsewhere, or the harness may be under-sensitive) — burning unlimited attempts on an unmovable metric is a failure mode of the loop, not evidence to keep trying.
Tools: Bash (git revert on loop branch), Read (attempt history).
Failure conditions: none beyond the halt itself, which is a normal terminal state, not an error to suppress.
Output artifact: `node-6-attempt-log.md` (every attempt this run, its change, its verified delta, its classification) — on halt, this becomes the loop's final artifact.

# Adversarial Check (reviewer persona + what it attacks)

Persona: **the Weld Auditor** — a reviewer whose only job is to attack whether a weld was actually earned, running FREE-MAD (no forced consensus, independent scoring). It never reviews unwelded attempts; it exists solely to re-litigate node 5 before the loop is allowed to report success. It attacks:

1. **Apples-to-apples**: were node 1 and node 3 run with the literal same harness invocation, same inputs, same environment — demands the two invocation records side by side, not a claim that they matched.
2. **Tolerance legitimacy**: was the tolerance band derived from real multi-run variance, or invented/assumed after the fact to make a marginal result look like a clean pass?
3. **Single-suspect discipline**: does the node-2 diff touch anything beyond the declared `lever`? Any scope creep invalidates the weld regardless of the measured delta.
4. **Cherry-picked re-measure**: was node 3 run once, or did the loop quietly re-run until a favorable number appeared? Demands the full attempt log, not just the winning run.
5. **Silent Agreement**: if it concurs with the weld in round 1 without citing the node-1/node-3 numeric evidence directly, a second hostile round is forced (PROTOCOL.md §11).

A weld the Auditor rejects is NOT reverted automatically (reverting a weld is itself a new ratchet turn per node 5's own rule) — rejection blocks this run from reporting PASS and escalates to the operator with the Auditor's findings attached.

# Exit Criteria (quantitative, overrides PROTOCOL.md §12)

- Weld is permitted only when the node-4 classification is verified improvement or verified no-regression, with delta measured against a tolerance band derived from ≥3 sampled runs (or a caller-supplied, justified tolerance for harnesses too expensive to sample 3x).
- Minimum effect size before weld: the delta must exceed the tolerance band by a non-zero margin in the declared `direction`; a delta inside the tolerance band welds only under the explicit "no-regression, neutral change accepted" case, never silently.
- Max consecutive failed/inconclusive single-change attempts before halt: `--max-attempts` (default 5); each attempt's change must differ from every prior attempt this run (verified by the Weld Auditor's single-suspect check applied retroactively to the attempt log).
- Zero un-welded changes left applied on the loop branch at loop end — every turn ends either welded or reverted, never dangling.
- Adversarial Check passed (or escalated to operator) before the run reports PASS.
- Final confidence ≥ 0.85; governance row written to `_loopstate/governance-ledger.md`.

# Failure Routing (only deviations from PROTOCOL.md §4)

- Non-reproducible or missing harness at node 1 is always a hard HALT — never degrade to an estimated baseline (this is a standing rail for this loop specifically, stricter than the general Dependency Failure default of "pin/resolve and retry").
- A weld rejected by the Adversarial Check is a Security-Failure-shaped HALT only if the weld touches cost/security-adjacent config (see Approval Gates); otherwise it routes as a Logic Failure — the run reports FAIL-with-artifact and the operator decides whether to re-run node 2 by hand.
- Diminishing-returns halt (node 6 over budget) is a normal terminal state, not routed through the escalation ladder in PROTOCOL.md §5 — it does not retry into a higher model tier, because the problem is the metric's movability, not reasoning quality.

# Approval Gates (only deviations from PROTOCOL.md §10)

- PROTOCOL.md defaults apply to the loop branch itself (operator approval to merge/PR).
- Additionally, and stricter than the general "Refactor on loop branch → Adversarial reviewer sign-off" default: any weld whose `lever` touches cost limits, rate limits, auth/permission thresholds, secret rotation cadence, or any other cost- or security-adjacent config value requires explicit HUMAN sign-off BEFORE node 5 commits — the Adversarial Check alone is not sufficient authority for that class of weld, even though this loop is otherwise fully mechanical and would normally weld without pausing.

# RUN PROMPT

```
Run LOOP-18 (The Ratchet) on <target>.
Follow LOOP-18-the-ratchet.md under [PROTOCOL] (see PROTOCOL.md in this repo).
metric: <the single measurable quantity>
lever: <the single parameter/threshold/config value this run may touch>
direction: <lower-is-better | higher-is-better>, min effect size: <value or "tolerance-derived">
harness: <exact reproducible measurement command>
max-attempts: <N, default 5>
Re-derive the baseline yourself — never accept a cached/estimated number.
One change per turn, identical re-measure method every time, weld only on
verified improvement or verified no-regression. Halt and report on
diminishing returns rather than looping indefinitely. If the lever touches
cost/security-adjacent config, stop before welding and get my sign-off.
```
