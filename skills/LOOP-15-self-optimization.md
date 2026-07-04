---
name: loop-self-optimization
loop-id: LOOP-15
description: Meta-loop: failure→root-cause→fix→generalized-pattern store; audits and improves the loop MDs + procedural memory themselves
domain: Continuous Evaluation
risk-class: branch-mutating (loops only)
default-debate: RA-CR
model-tiers: {scan: cheap, audit: mid, verdict: reasoning}
---

# Mission

The recursive improvement cycle: **Workflow → Audit Workflow → Improve Workflow → Test Workflow → Deploy Workflow.**

LOOP-15 is the only loop whose mutation target is the loop system itself — the `LOOP-XX-*.md` files in this collection's `skills/` directory and the procedural-memory patterns they encode. It:

1. Ingests LOOP-14's GOVERNANCE-REPORT and the fleet's failure records.
2. Runs the long-term learning chain per failure: **Failure → Root Cause → Successful Fix → Generalized Pattern → Future Workflow Update**.
3. Turns generalized patterns into concrete diffs against loop MDs: tighter exit criteria, new nodes for missed bug classes, retired dead nodes, better failure routing.
4. Dry-run tests each updated loop on a small scoped target before adoption.
5. Versions every change and hands the whole package to the operator for approval.

Loops are procedural memory; per CoALA, procedural memory is version-controlled and evolves — but only under human sign-off. **Hard boundary: LOOP-15 NEVER touches application code, and NEVER weakens protocol safety rails** (approval classes, security halts, rollback rules) — any proposal in that direction is quarantined and surfaced to the operator as a discussion item, never a diff.

# Trigger (when the operator runs this)

- After each LOOP-14 run whose GOVERNANCE-REPORT carries `for-LOOP-15` recommendations (a common cadence: after your governance/telemetry loop runs).
- When any loop exhausts retries on the same node across 2+ consecutive runs (the loop is fighting its own spec).
- When a bug class escapes the fleet entirely (found in production, no loop caught it) — a missing-node signal.
- On operator demand: `Run LOOP-15`.

# Inputs (target repo/dir, scope flags)

- `governance_report`: latest `_loopstate/LOOP-14/<run-id>/GOVERNANCE-REPORT.md` (REQUIRED, ≤ 7 days old — stale report → run LOOP-14 first).
- `failure_records`: all FAIL/HALTED artifacts + retry-exhaustion checkpoints in `_loopstate/*/*/` within the window; escaped-bug reports supplied by the operator.
- `pattern_store`: `_loopstate/LOOP-15/pattern-store.md` — the durable generalized-pattern ledger (created first run; append/update only).
- `dry_run_target`: small scoped target for node 6 (a single low-risk module/dir the operator names; never a production surface).
- Flags: `--loop=<LOOP-ID>` (restrict proposals to one loop), `--patterns-only` (learning chain without diff proposals).

# Preconditions

- [PROTOCOL] read. This collection is a git repo (or your fork of it) — versioning is non-negotiable for procedural memory.
- Loop branch `loop/LOOP-15-<date>` created in this repo.
- **Mutation surface is EXCLUSIVELY this collection's `skills/LOOP-*.md` files + `_loopstate/LOOP-15/`.** `PROTOCOL.md` is READ-ONLY to this loop — protocol changes are operator-only, by hand.
- Fresh GOVERNANCE-REPORT present (see Inputs).
- No other LOOP-15 run mid-flight (the meta-loop is strictly serialized).

# Execution DAG (numbered nodes, [P] = parallelizable)

1. Ingest GOVERNANCE-REPORT + failure-record corpus
2. Long-term learning chain (per-failure root-cause analysis)
3. Pattern generalization + pattern-store update
4. Loop-MD diff proposals (the Improve step)
5. Safety-rail guard screen (quarantine gate)
6. Dry-run test of updated loops on scoped target
7. Versioning + changelog + proposal package
8. Adversarial verification (RA-CR) + operator approval handoff

Order: 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8. No parallel nodes: each step's output is the next step's input, and the safety screen (5) must see every diff before anything is tested (6).

# Node Specs

### Node 1 — Ingest
- **Action:** parse the GOVERNANCE-REPORT (per-loop verdicts, detector flags, `for-LOOP-15` recommendations) and collect the failure corpus: every FAIL-with-artifact, HALTED-* artifact, retry-exhaustion checkpoint, benchmark regression, and operator-supplied escaped bug in the window; deduplicate into distinct failure cases (same loop + node + failure-class + signature = one case with occurrence count).
- **Tools:** Read, Glob over `_loopstate/`, cheap tier.
- **Failure conditions:** GOVERNANCE-REPORT missing/stale → HALT, instruct operator to run LOOP-14; zero failure cases AND zero recommendations → terminate early as PASS "nothing to learn", governance row written (do not invent work).
- **Output artifact:** `failure-corpus.md` (case id, loop, node, class, occurrences, evidence links).

### Node 2 — Long-term learning chain
- **Action:** for EACH failure case, build the full chain: **Failure** (what observably went wrong) → **Root Cause** (why — interrogate the trace, the node spec, and the inputs; distinguish spec defect, missing node, wrong tool, wrong tier, bad exit criterion, bad failure routing, environmental issue) → **Successful Fix** (what eventually worked: the retry that passed, the human override, the manual fix — or NONE YET, which is itself data). A case whose root cause is "the target code was genuinely hard" is NOT a loop defect — mark `no-action:inherent` and exclude from proposals.
- **Tools:** Read on traces + loop MDs, mid tier reasoning, reasoning tier for ambiguous root causes.
- **Failure conditions:** root cause undeterminable from evidence → mark `insufficient-trace`, and THAT becomes a proposal input (the loop's §9 trace spec needs strengthening).
- **Output artifact:** chain table (case → failure → root cause → fix → classification).

### Node 3 — Pattern generalization + store update
- **Action:** cluster chains into **Generalized Patterns** — statements that transcend the single case (e.g. "fuzz nodes on parser-heavy modules need corpus seeding or they time out"; "read-only loops touching git worktrees must verify detached state first"). Append new patterns to `_loopstate/LOOP-15/pattern-store.md` (fields: pattern-id, ISO date, source cases, statement, affected loops, status: proposed|adopted|retired); update occurrence counts on existing patterns; retire patterns contradicted by newer evidence (status→retired with reason — never delete rows). This store IS the fleet's long-term learning; it must remain human-readable.
- **Tools:** mid tier clustering, Write to the pattern store.
- **Failure conditions:** pattern supported by a single occurrence → record as `weak-signal`, do NOT generate a diff from it (small-n discipline: one instance is an anecdote, not a pattern).
- **Output artifact:** updated pattern-store + new/updated pattern list.

### Node 4 — Loop-MD diff proposals
- **Action:** translate adopted-candidate patterns + `for-LOOP-15` recommendations into CONCRETE unified diffs against specific loop MDs. Legal proposal classes:
  - **Tighter exit criteria** — raise thresholds where loops pass too easily per LOOP-14 Goodhart findings.
  - **New nodes for missed bug classes** — escaped bugs become a detection node in the owning loop.
  - **Retired dead nodes** — nodes producing zero findings and zero value across ≥ 5 runs, with the evidence.
  - **Better failure routing** — reroute failure classes that repeatedly dead-ended.
  - **Tool/tier corrections** — persistent Tool Failures → fallback tool baked in; wasteful tier mixes → tier reassignment.
  Every diff carries: motivating pattern-id(s), expected metric improvement (which §8 metric, which direction), and a rollback note.
- **Tools:** Read loop MDs, mid tier drafting, reasoning tier review.
- **Failure conditions:** a recommendation cannot be expressed as a bounded MD diff → escalate to operator as a design question, not a diff.
- **Output artifact:** `proposals/<n>-<loop-id>.diff` + rationale block per proposal.

### Node 5 — Safety-rail guard screen (QUARANTINE GATE)
- **Action:** screen EVERY proposal against the hard boundary. Auto-quarantine any diff that: weakens or removes an Approval Gate; lowers a security-related halt (Security Failure handling, chaos production prohibition, standing rails such as never self-escalating privileges or never mutating outside `_loopstate`); loosens rollback rules or checkpoint requirements; expands a loop's risk-class or write surface; touches `PROTOCOL.md`; or edits LOOP-15's own guard sections (this node, Approval Gates, the mission boundary). Quarantined items move to `SURFACED-TO-OPERATOR.md` with full reasoning — never tested, never merged, never "just this once". The screen is a deny-list PLUS a semantic check by the reasoning tier: "does this diff make any safety behavior less likely to fire?"
- **Tools:** diff analysis, reasoning tier.
- **Failure conditions:** ambiguity about whether a diff weakens a rail → quarantine it (fail-closed).
- **Output artifact:** proposals split into `cleared/` and `SURFACED-TO-OPERATOR.md`.

### Node 6 — Dry-run test of updated loops
- **Action:** for each cleared proposal, apply the diff on the loop branch and execute the updated loop against `dry_run_target` in scoped/degraded mode (scan + audit nodes; any mutating nodes of the target loop run propose-only — the dry run must not itself mutate app code). Verify: the updated loop still parses per PROTOCOL.md §14 template; runs end-to-end; its exit criteria remain reachable; and the change moves in the promised direction (e.g. a new detection node actually fires on a seeded example of the missed bug class — seed one from the original failure case). A dry run that makes the loop slower/noisier with no detection gain = proposal rejected with evidence.
- **Tools:** the updated loop MD itself, dry-run harness, mid tier.
- **Failure conditions:** dry run reveals the diff breaks the loop → return to node 4 for revision (max 2 revision cycles per proposal, then drop with record).
- **Output artifact:** `dryrun-report.md` per proposal (parse OK, run OK, promised-improvement evidence, verdict KEEP/REVISE/DROP).

### Node 7 — Versioning + changelog + proposal package
- **Action:** for every KEEP proposal, commit the diff on the loop branch (`loop(LOOP-15): <target-loop> — <change summary>` + pattern-ids + confidence block in body) AND append a dated entry to a `## Changelog` section at the bottom of the modified loop MD (`- YYYY-MM-DD (LOOP-15 run <run-id>): <change> — motivated by <pattern-id>, dry-run: PASS`). Assemble the operator package: per-proposal diff, rationale, pattern lineage, dry-run evidence, expected metric delta — plus the quarantine file and the pattern-store delta.
- **Tools:** git, Write.
- **Failure conditions:** git unavailable → HALT (versioning is a precondition, not a nicety).
- **Output artifact:** committed branch + `PROPOSAL-PACKAGE.md`.

### Node 8 — Adversarial verification + operator handoff
- **Action:** run the Adversarial Check below (RA-CR) over the full package; on survival, write AUDIT-ARTIFACT (PROTOCOL.md §13) + governance row, and hand the package to the operator. **Nothing merges without operator approval — the loop ends at the handoff, always.**
- **Tools:** LOOP-11 in RA-CR mode, reasoning tier.
- **Failure conditions:** reviewer rejects a proposal twice → drop it from the package (recorded), do not relitigate.
- **Output artifact:** `AUDIT-ARTIFACT.md` + handoff message listing each proposal for individual approve/reject.

# Adversarial Check (reviewer persona + what it attacks)

Protocol: **RA-CR** (rank-adaptive cross-round — the judge reorders and silences reviewers per round to force argument diversity; self-modification proposals deserve the maximum-scrutiny protocol, not FREE-MAD). Persona: **the Reward-Hacking Hunter** — a reviewer who assumes every self-modification exists to make the system's numbers look better rather than make the system better. It attacks:

1. **Exit-criteria edits** — does a "tighter" criterion actually loosen something in disguise, or shift measurement to a friendlier denominator?
2. **Retired nodes** — is a node "dead" because it is useless, or because it keeps flagging the fleet's own failures?
3. **New nodes** — do they detect the escaped bug CLASS, or merely the one historical instance (overfit to the anecdote)?
4. **Self-serving learning** — do patterns systematically blame the environment and never the loop specs?
5. **Guard-screen evasion** — re-runs its own independent rail check on every cleared diff, hunting anything node 5 missed; a single miss fails the entire package.
6. **Dry-run theater** — was the seeded example trivially easy? It demands the dry-run inputs and re-judges them.

# Exit Criteria (quantitative, overrides PROTOCOL.md §12)

- 100% of failure cases from node 1 carry a completed learning chain (or explicit `insufficient-trace` / `no-action:inherent` classification).
- Pattern store updated; zero diffs generated from single-occurrence patterns.
- **Every proposed diff: cleared the safety-rail screen, dry-run tested with recorded evidence, committed on the loop branch, and changelogged in the target MD** — or dropped/quarantined with reasons.
- **Zero rail-weakening diffs in the cleared set** (verified independently by the adversarial reviewer); all such items in `SURFACED-TO-OPERATOR.md`.
- Zero writes outside this collection's `skills/LOOP-*.md` files (branch) + `_loopstate/LOOP-15/`; `PROTOCOL.md` untouched (verified via `git status` + diff).
- Operator handoff package delivered; **zero merges performed by the loop itself.**
- Final confidence ≥ 0.85; governance row written.

# Failure Routing (only deviations from PROTOCOL.md §4)

- Any indication that a prior LOOP-15 run merged or weakened something without approval → **Security Failure**: HALT, CRITICAL artifact, full stop pending operator review of this repo's git history.
- Dry-run harness failure → Tool Failure, but the affected proposal is marked UNTESTED and automatically excluded from the KEEP set (untested self-modifications never ship).

# Approval Gates (only deviations from PROTOCOL.md §10)

- **Every loop-MD change requires explicit operator approval before it goes live** — merge of the loop branch is operator-only, per-proposal (the operator may approve some diffs and reject others).
- `PROTOCOL.md`: NOT editable by this loop under any circumstances; proposals affecting it go to `SURFACED-TO-OPERATOR.md` as prose, never diffs.
- LOOP-15 may not modify its own Approval Gates, safety-rail screen (node 5), or mission boundary — such changes are operator-by-hand only.

# RUN PROMPT

```
Run LOOP-15 (Self-Optimization Meta-Loop).
Read PROTOCOL.md (this collection's shared protocol), then
LOOP-15-self-optimization.md, and execute its Execution DAG end-to-end under
protocol rules.
governance_report: latest _loopstate/LOOP-14/<run-id>/GOVERNANCE-REPORT.md
dry_run_target: <small low-risk dir/module I name here>
Mutation surface: ONLY the LOOP-*.md files in this collection's skills/
directory, on branch loop/LOOP-15-<date>, plus _loopstate/LOOP-15/. NEVER app
code, NEVER PROTOCOL.md, NEVER any safety rail — quarantine those to
SURFACED-TO-OPERATOR.md. Dry-run every diff before including it. Hand me the
PROPOSAL-PACKAGE for per-proposal approval. Merge nothing yourself.
```
