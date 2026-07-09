---
name: loop-fanout-orchestrator
loop-id: LOOP-16
description: Decompose one incoming task into independent units, dispatch each to its own autonomous worker loop with no mid-flight barrier, and merge at collection time
domain: Multi-Agent Orchestration
risk-class: branch-mutating
default-debate: FREE-MAD
model-tiers: {scan: cheap, audit: mid, verdict: reasoning}
---

# Mission

Formalize the fan-out pattern: one orchestrator receives a task, decomposes it into N units of work that are provably independent (no shared mutable state, no ordering dependency), and dispatches each unit to its own worker loop that runs to completion on its own schedule — retrying, refining, and verifying internally, with no synchronization barrier back to the orchestrator mid-flight. The orchestrator's real job is proving independence before it splits work, isolating each worker's mutations so parallel writers never collide, and running a collection/merge strategy that tolerates workers finishing at different times. This loop governs the split and the merge; it does not micromanage what happens inside a worker's own loop.

# Trigger (when the operator runs this)

- A task naturally decomposes into multiple independent subtasks (e.g. "audit these 5 modules," "generate variants for these 3 targets," "port this feature across 4 services") and the operator wants them run concurrently instead of one after another.
- Before committing to `parallel()` over `pipeline()` in any multi-agent orchestration system — this loop is the judgment call that decides which is correct for the task at hand.
- On demand: `Run LOOP-16 on <task description> (target: <repo/dir>)`.

# Inputs (target repo/dir, scope flags)

- `task`: the incoming unit of work to decompose (a feature, an audit, a migration, a generation batch).
- `target`: repo root or working directory the workers will operate against.
- `--max-workers <N>`: cap on concurrent worker loops (default: number of decomposed units, no artificial ceiling below that).
- `--collection-mode <streaming|barrier>`: streaming = orchestrator ingests each worker's result as it lands; barrier = orchestrator waits for all workers before merging. Default: streaming.
- `--no-mutate`: decomposition-and-plan mode only; do not dispatch workers or touch the target.

# Preconditions

- The task must survive the independence proof in Node 1 before any dispatch happens — if it fails, this loop terminates as FAIL-with-artifact and recommends a sequential pipeline instead of forcing a fan-out.
- Git worktree clean; loop branch `loop/LOOP-16-<YYYY-MM-DD>` created from origin/main before any worker is dispatched.
- If workers will mutate shared files, a per-worker isolation mechanism (isolated git worktree or branch per worker) must be available — verify with `Bash: git worktree list` support before Node 4.
- Each worker's own loop file (or task spec) must declare its own local completion criteria; the orchestrator does not accept a worker dispatch that has no defined "done."

# Execution DAG (numbered nodes, [P] = parallel markers)

1. Task ingestion & candidate decomposition — split the incoming task into candidate units.
2. Independence proof — verify no shared mutable state, no ordering dependency between candidate units; reject/re-merge units that fail.
3. Isolation assignment — allocate a collision-free workspace (worktree/branch/namespace) per surviving unit.
4. [P] Worker dispatch — launch one autonomous worker loop per unit, each running its own local retry/refine/verify cycle to its own completion criteria, no barrier.
5. Streaming collection — ingest each worker's result artifact as it lands; track per-worker state (pending/running/done/failed) without blocking on siblings.
6. Failure-policy resolution — for any worker that fails permanently, apply the declared policy (retry with a fresh worker / block the merge / ship partial with the gap flagged).
7. Merge — reconcile all landed (and any accepted-partial) results into the loop branch, resolving cross-worker conflicts the independence proof didn't fully rule out.
8. Adversarial verification + close-out — verify the merged result against the original task, emit the audit artifact.

Node 4 is the fan-out point: every dispatched worker runs [P] relative to its siblings, with no join until Node 5 begins consuming results. Nodes 1–3 are strictly sequential gatekeeping for the fan-out; nodes 5–8 are strictly sequential collection/close-out.

# Node Specs

**Node 1 — Task ingestion & candidate decomposition.**
Action: parse the incoming task and propose a candidate split into N units, each stated as a self-contained deliverable (input, expected output, done-condition). Decomposition axis should follow the task's natural seams (per-module, per-file-set, per-target-variant, per-service) — never split arbitrarily just to create parallelism.
Tools: Read (task spec / issue), a code-graph/knowledge-graph tool if available (module boundaries), Bash (repo layout scan).
Failure conditions: task cannot be split into more than one genuinely separable unit (Logic Failure — recommend a single-worker or sequential pipeline loop instead, do not force a fan-out); candidate count exceeds `--max-workers` with no natural grouping (Context Overflow — cluster into batches, re-propose).
Output artifact: `_loopstate/LOOP-16/<run-id>/node-1-candidates.md` — table `unit-id | description | inputs | done-condition`.

**Node 2 — Independence proof.**
Action: for every pair of candidate units, check for (a) shared mutable state (same file, same DB row/table, same config key written by both), (b) ordering dependency (unit B's correctness depends on unit A having already run), (c) shared external resource contention (same rate-limited API key, same lock). Any positive hit forces either a merge of the two units into one, or an explicit sequencing edge that moves one unit out of the fan-out and into a prerequisite step before Node 4. This is the orchestrator's real job — not convenience-splitting.
Tools: a code-graph/knowledge-graph tool if available (write-set overlap), ast-grep/Grep (literal resource-key collisions), Read.
Failure conditions: any unresolved dependency between two units that cannot be cleanly severed (Architecture Failure — halt fan-out for that pair, emit finding, route to LOOP-13 drift/debt or a sequential loop); ambiguous shared-state verdict (mark `unknown`, treat conservatively as dependent, exclude from parallel dispatch).
Output artifact: `node-2-independence-proof.md` — pairwise matrix + final surviving unit list with confidence block.

**Node 3 — Isolation assignment.**
Action: for every surviving unit whose worker will mutate shared resources (files, a database, a shared branch), allocate a collision-free workspace: an isolated git worktree or branch per unit (mirroring how this repo's other mutating loops isolate concurrent work), a scoped DB migration branch, or a namespaced output directory. Record the mapping `unit-id → isolation target` so Node 7 knows what to reconcile.
Tools: Bash (`git worktree add`, branch creation), Write (mapping file).
Failure conditions: isolation mechanism unavailable for the target resource type (Tool Failure — retry once, then HALT that unit's dispatch and route it to sequential execution after the fan-out, never dispatch it unisolated).
Output artifact: `node-3-isolation-map.md`.

**Node 4 — Worker dispatch [P].**
Action: launch one autonomous worker loop per surviving unit, each in its assigned isolation target, each carrying its own input, done-condition, and local retry/refine/verify cycle (per PROTOCOL.md §1 steps 3–4, scoped to that unit only). The orchestrator passes the unit spec and isolation target and then steps back — it does not inspect or gate the worker's internal steps, and a worker finishing early does not wait on a slower sibling. Each worker emits its own checkpointed output artifact on completion or permanent failure.
Tools: whatever agent/task-dispatch mechanism the host environment provides (subagent spawn, background job, CI matrix job); Bash for process/job tracking.
Failure conditions: dispatch mechanism cannot actually run units concurrently (Tool Failure — degrade to sequential dispatch, log to governance, continue); a worker never reports back within a declared timeout (Timeout — per PROTOCOL.md §4, checkpoint and treat as a permanent failure for Node 6 to route).
Output artifact: one `node-4-worker-<unit-id>.json` per dispatched unit (dispatch record: start time, isolation target, worker loop reference).

**Node 5 — Streaming collection.**
Action: as each worker lands a result (success or permanent failure), ingest its output artifact and update a live status table (`pending|running|done|failed` per unit). Do not block on any unit still running — this node's job is to consume results as they arrive, not to wait for the slowest worker before doing anything. If `--collection-mode barrier` was set, this node instead polls until all units reach a terminal state before proceeding, and that deviation must be logged.
Tools: Bash (poll/watch worker outputs), Read (per-worker artifacts), Write (status table).
Failure conditions: a worker's output artifact is malformed or missing its confidence block (Model Hallucination — discard, treat as permanent failure for that unit, route to Node 6).
Output artifact: `node-5-status-table.md`, updated live, final snapshot retained.

**Node 6 — Failure-policy resolution.**
Action: for every unit in a terminal `failed` state, apply the declared failure policy in this order unless the loop invocation overrides it: (1) retry once with a fresh worker in a fresh isolation target if the failure class is retryable per PROTOCOL.md §4 (Syntax/Tool/Rate-Limit/Timeout); (2) if retry is exhausted or the failure class is Logic/Architecture/Security, mark the unit operator-deferred and decide whether it blocks the merge (default: Security Failure always blocks; non-security gaps ship as partial results with the gap explicitly flagged in the merge artifact). Never silently drop a failed unit from the final report.
Tools: Bash (re-dispatch), Read (failure classification from worker artifact).
Failure conditions: a Security Failure in any unit — per PROTOCOL.md §4, HALT the merge for the whole run until operator approval, even if other units succeeded.
Output artifact: `node-6-failure-resolution.md` — table `unit-id | failure class | action taken | blocks merge?`.

**Node 7 — Merge.**
Action: reconcile every `done` unit's output (and any accepted-partial units from Node 6) from its isolated workspace back onto the loop branch. Resolve any conflict the independence proof didn't fully anticipate (e.g. two units both touched a shared file the proof missed) by treating it as a Node 2 failure retroactively — halt, do not silently overwrite. Run the full test suite once after all merges land, not once per unit.
Tools: Bash (`git merge`/`git cherry-pick` from isolated branches, tests), Edit (conflict resolution only when unambiguous).
Failure conditions: merge conflict with no unambiguous resolution (Logic Failure — halt, surface both versions to operator, do not guess); post-merge test regression (rollback per PROTOCOL.md §6, isolate to the offending unit, re-open that unit only).
Output artifact: merge commit list + `node-7-merge.md`.

**Node 8 — Adversarial verification + close-out.**
Action: validate the merged result against the original task from Node 1 — does the sum of unit outputs actually satisfy the task, or did decomposition lose something at the seams? Verify every flagged partial-result gap from Node 6 is visible in the final artifact, not buried. Emit the standard audit artifact.
Tools: Read, a code-graph/knowledge-graph tool if available (cross-unit consistency), Bash (final test run).
Failure conditions: merged result does not cover the full original task scope (Logic Failure — identify the missing seam, spin a follow-up unit, do not mark PASS with a silent gap).
Output artifact: `node-8-closeout.md` + final `AUDIT-ARTIFACT.md` per PROTOCOL.md §13.

# Adversarial Check (reviewer persona + what it attacks)

Persona: **skeptical distributed-systems reviewer** running FREE-MAD (no forced consensus, score-based verdict). The reviewer attacks, in order:

1. Node 2's independence proof first — demands the actual write-set/read-set evidence for every "independent" verdict, not the claim that units don't overlap.
2. Every unit that got merged or resequenced out of the fan-out — checks whether the merge masked a real dependency instead of resolving it.
3. Node 6's failure-policy application — verifies a "shipped as partial" decision was actually flagged in the final artifact, not quietly absorbed.
4. Node 3's isolation map — hunts for any unit dispatched to a shared, unisolated resource despite Node 3's assignment.
5. Silent Agreement: if it concurs with everything in round 1 without citing unit-level evidence, a second hostile round is forced (PROTOCOL.md §11).

Maximum 2 debate rounds; disagreements surviving round 2 are recorded as operator-deferred findings, never silently dropped.

# Exit Criteria (quantitative, overrides PROTOCOL.md §12)

- 100% of dispatched units reach a terminal state (`done` or `failed`-with-resolution) before Node 7 begins — no unit left `running` indefinitely.
- Zero units dispatched without a passing independence proof (Node 2) or an explicit isolation assignment (Node 3) if they mutate shared resources.
- Zero unresolved Security Failures in any unit blocking the merge.
- Every `failed` unit has a recorded resolution action (retry / operator-deferred / partial-flagged) — zero silently dropped units.
- Test pass rate ≥ 99% on the merged surface; every unit-level bug fixed inside a worker loop still leaves a regression test per that worker's own loop contract.
- Retry count ≤ 3 per node (orchestrator nodes) — per-worker retry budgets are governed by the worker's own loop, not this one.
- Final confidence ≥ 0.85; governance row written to `_loopstate/governance-ledger.md`, including `worker_count`, `parallel_wallclock_vs_sequential_estimate`, and `units_shipped_partial`.

# Failure Routing (only deviations from PROTOCOL.md §4)

- A failure inside a worker's own loop is classified and handled by that worker's loop contract first; it only surfaces to this loop's Node 6 as a terminal `failed` state with a failure class attached — the orchestrator does not re-diagnose a worker's internal failure from scratch.
- Independence-proof failures (Node 2) are never silently downgraded to "run it anyway" — they always route to either a merge of the two units or an explicit sequencing edge.
- A Timeout on one worker (Node 4) never halts sibling workers already dispatched; it only blocks that unit's path into Node 7 until Node 6 resolves it.

# Approval Gates (only deviations from PROTOCOL.md §10)

- PROTOCOL.md defaults apply. Additionally: dispatching more than `--max-workers` concurrent units against the same target repo requires explicit operator approval before Node 4, since it raises the collision surface Node 2/3 have to have gotten right.
- Shipping any unit as "partial results, gap flagged" (Node 6) requires the gap to be named explicitly in the RUN PROMPT response to the operator — it is never bundled into a PASS verdict silently.

# RUN PROMPT

```
Run LOOP-16 on <task description> (target: <repo/dir>). Follow LOOP-16-fanout-orchestrator.md under [PROTOCOL] (see PROTOCOL.md in this repo). Prove independence before dispatching anything — if the task doesn't survive Node 2, tell me to run a sequential loop instead of forcing a fan-out. Isolate every mutating worker in its own worktree/branch, stream-collect results with no artificial barrier, and flag any partial-result gaps explicitly in the final artifact. Mutate only on loop/LOOP-16-<date>. No merge without my approval.
```
