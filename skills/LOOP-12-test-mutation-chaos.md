---
name: loop-test-mutation-chaos
loop-id: LOOP-12
description: Test generation (regression/property/fuzz/snapshot/integration), mutation testing of weak suites, chaos injection (DB down, Redis off, LLM garbage, MCP gone)
domain: Continuous Evaluation
risk-class: branch-mutating
default-debate: FREE-MAD
model-tiers: {scan: cheap, audit: mid, verdict: reasoning}
---

# Mission

Harden the target's test suite until it is an adversary, not a formality. This loop does three things in sequence:

1. **Test generation** — closes coverage gaps across ALL test types (regression, property, fuzz, snapshot, integration) and enforces the collection-wide rule that **every bug found by any other loop must leave a permanent regression test**. LOOP-12 is the enforcement point for that rule: it sweeps other loops' artifacts and backfills any missing tests.
2. **Mutation testing** — attacks the suite itself. Instead of asking "does the code work?", it asks "what if this code was intentionally broken?", mutates operators and conditionals, and scores how many deliberate breaks the suite actually catches. Tests that stay green under mutation are weak tests.
3. **Chaos engineering** — injects controlled dependency failures into a sandbox/dev environment, observes recovery, and grades graceful degradation.

Output: a stronger suite on a loop branch, a mutation-strength score, and a graded chaos-resilience report.

# Trigger (when the operator runs this)

- Pre-release sweep (commonly run alongside LOOP-08 and LOOP-06 as part of a pre-release gate).
- After any mutating loop (LOOP-01/02/03/05/06/08) closes findings — to verify each fixed bug left a regression test.
- On operator demand: `Run LOOP-12 on <repo> (scope: <dir>)`.

# Inputs (target repo/dir, scope flags)

- `target`: repo + optional scope dir (e.g. `<your-repo>`, scope `desktop/` or `api/`).
- `bug_sources`: prior `_loopstate/*/*/AUDIT-ARTIFACT.md` files whose findings tables list fixed bugs (defaults to all loops, last 30 days).
- `chaos_env`: REQUIRED explicit name of the sandbox/dev environment for node 7 (e.g. local docker-compose, a scoped dev/staging deployment). If absent, chaos nodes are SKIPPED, never guessed.
- Flags: `--no-chaos`, `--mutation-only`, `--types=regression,property,fuzz,snapshot,integration`.

# Preconditions

- [PROTOCOL] read; loop branch `loop/LOOP-12-<date>` created from clean origin/main (per standing rule: origin/main is truth).
- Test runner identified and green at baseline. A red baseline suite → HALT, route to operator; this loop hardens suites, it does not repair broken builds.
- Context gathered for every file in scope per [PROTOCOL] §2 — a code-graph/knowledge-graph tool if your project has one (MCP server, IDE semantic index), otherwise `ast-grep`/Semgrep/LSP "find references".
- `chaos_env` verified reachable AND verified NOT to be a production host (production hostnames/IPs are a deny-list checked before any injection).

# Execution DAG (numbered nodes, [P] = parallelizable)

1. Scope & coverage-gap map
2. Regression backfill sweep (bug-to-test enforcement)
3. [P] Test generation — behavioral types (regression, integration, snapshot)
4. [P] Test generation — generative types (property, fuzz)
5. Mutation testing run + suite-strength score
6. Weak-test remediation (kill surviving mutants)
7. Chaos injection campaign (sandbox/dev ONLY — approval-gated)
8. Graceful-degradation grading & recovery report
9. Adversarial verification + AUDIT-ARTIFACT

Order: 1 → 2 → (3 ‖ 4) → 5 → 6 → 7 → 8 → 9. Nodes 7–8 skip cleanly if `chaos_env` is unset.

# Node Specs

### Node 1 — Scope & coverage-gap map
- **Action:** run the project's coverage tooling (jest/vitest `--coverage`, pytest `--cov`, cargo tarpaulin as applicable) on the scoped dir; cross-reference uncovered files against your code-graph tool's fan-in/centrality ranking if available — high-fan-in (god-node) uncovered files rank first, architecture docs/ADRs otherwise.
- **Tools:** coverage runner, a code-graph/knowledge-graph MCP tool if available, `ast-grep` for entrypoint discovery.
- **Failure conditions:** coverage tool absent → Dependency Failure (install on branch or emit finding); baseline suite red → HALT.
- **Output artifact:** `node-1.json` + `coverage-gap-map.md` (file, current %, module/cluster, priority).

### Node 2 — Regression backfill sweep
- **Action:** parse every `bug_sources` artifact's findings table; for each finding marked fixed, search the suite (`ast-grep` on test names/assertions referencing the fixed symbol or issue id) for a corresponding regression test. Any fixed bug with no test = a MISSING-REGRESSION finding, and this node writes the test immediately — reproducing the original failure condition, then asserting the fix.
- **Tools:** Read on `_loopstate`, `ast-grep`, test runner.
- **Failure conditions:** cannot reproduce original bug context → mark `unverifiable-regression`, emit finding, do NOT fabricate a vacuous test.
- **Output artifact:** backfill table (bug id → test file:name → status) + commits `loop(LOOP-12): node-2 — regression backfill <bug-id>`.

### Node 3 — [P] Behavioral test generation
- **Action:** for each coverage gap from node 1, generate regression tests (pin current correct behavior), integration tests (API endpoint / IPC channel / DB boundary crossings, per your code-graph tool's dependency edges, or manual boundary tracing otherwise), and snapshot tests (serialized outputs, UI component trees) — AAA structure, descriptive names.
- **Tools:** mid-tier codegen, test runner, a code-graph tool (or manual dependency tracing) for boundary discovery.
- **Failure conditions:** generated test flaky over 3 runs → discard and reclassify the gap; test passes trivially without exercising target lines (verified via per-test coverage) → discard.
- **Output artifact:** committed tests + per-type counts.

### Node 4 — [P] Generative test generation
- **Action:** property tests (fast-check / hypothesis / proptest) for pure-ish functions with inferable invariants (round-trips, idempotence, ordering, bounds); fuzz tests for parsers, input validators, and any function your code-graph tool (or manual review) shows receiving external/untrusted data. Seed corpora from real fixture data where present.
- **Tools:** property/fuzz frameworks, `ast-grep` for candidate signatures.
- **Failure conditions:** property counterexample found → that is a REAL BUG: emit HIGH finding, write the minimized counterexample as a permanent regression test (feeding node 2's rule), do NOT weaken the property.
- **Output artifact:** committed tests + any counterexample findings.

### Node 5 — Mutation testing run
- **Action:** run a mutation tool (Stryker for JS/TS, mutmut/cosmic-ray for Python, cargo-mutants for Rust) or, where no tool fits, perform agentic mutation: systematically flip operators (`<`→`<=`, `+`→`-`), negate conditionals, delete statements, swap return values on the scoped files, running the suite per mutant. A mutant the suite fails to kill = a **weak test region** (tests that stay green under intentional breakage). Compute mutation score = killed / (killed + survived).
- **Tools:** mutation framework or scripted AST mutation, test runner, cheap tier for triage.
- **Failure conditions:** runtime exceeds budget → sample mutants stratified by file priority (node 1 ranking), never silently truncate; report the sampling rate.
- **Output artifact:** `mutation-report.md` (score, surviving mutants with file:line and mutation applied).

### Node 6 — Weak-test remediation
- **Action:** for each surviving mutant, either strengthen the nearest test's assertions or write a new targeted test that kills it; re-run mutation on remediated files to confirm kills. Equivalent mutants (behavior-identical) are documented and excluded from the denominator with justification.
- **Tools:** mid tier, test runner, mutation re-run.
- **Failure conditions:** 3 attempts fail to kill a non-equivalent mutant → escalate to reasoning tier once; still surviving → emit MEDIUM finding "untestable region", include in artifact.
- **Output artifact:** remediation table + updated mutation score.

### Node 7 — Chaos injection campaign (APPROVAL-GATED, sandbox/dev only)
- **Action:** inject, one scenario at a time with clean restoration between runs, the full catalog: **database unavailable, cache (e.g. Redis) offline, API timeout, network latency, disk full, memory pressure, token limit hit, LLM unavailable, model returns garbage, MCP server unavailable**. Injection mechanics: stop/pause containers, toxiproxy or firewall rules for latency/timeout, tmpfs fill for disk, cgroup limits for memory, mock endpoints returning malformed payloads for "model returns garbage", env-var API-key invalidation for LLM/MCP loss.
- **Tools:** docker/compose, toxiproxy or `tc`, mock servers, Bash (sandbox only).
- **Failure conditions:** target of injection resolves to any production host → **HALT LOOP immediately, Security-class event, human approval required**; environment fails to restore clean between scenarios → stop campaign, report partial.
- **Output artifact:** per-scenario raw observation log in the checkpoint dir.

### Node 8 — Graceful-degradation grading & recovery report
- **Action:** grade each scenario A–F: A = degraded gracefully (user-coherent error, retry/backoff, state preserved, auto-recovery on restoration); C = survived but ugly (silent failures, lost input, no retry); F = crash, corruption, hang, or infinite retry loop. Record time-to-recovery per scenario. Every C or worse → finding + (where testable) a permanent automated chaos regression test committed to the branch.
- **Tools:** log analysis, mid tier.
- **Failure conditions:** ungradeable scenario (no observable signal) → grade `INCONCLUSIVE`, list required instrumentation as a finding.
- **Output artifact:** `chaos-grade-matrix.md` (scenario × grade × TTR × finding-id).

### Node 9 — Adversarial verification + artifact
- **Action:** run the Adversarial Check (below) over nodes 2–8 outputs; on acceptance, assemble AUDIT-ARTIFACT per [PROTOCOL] §13 and write the governance row ([PROTOCOL] §8).
- **Tools:** LOOP-11 inline, reasoning tier.
- **Failure conditions:** reviewer rejects twice → rollback per [PROTOCOL] §6, terminate FAIL-with-artifact.
- **Output artifact:** `AUDIT-ARTIFACT.md`.

# Adversarial Check (reviewer persona + what it attacks)

Persona: **the Mutation Adversary** — a hostile QA architect who believes every green test is a lie until proven otherwise, and who is personally offended by assertion-free tests. Protocol: FREE-MAD ([PROTOCOL] §11), max 2 rounds. It attacks:

1. **Vacuous tests** — re-runs a sample of generated tests with the asserted code path stubbed out; if they still pass, the test is rejected.
2. **The mutation score's denominator** — audits every "equivalent mutant" justification, assuming they are excuses.
3. **Chaos grades** — demands log evidence for every A grade ("show me the retry, show me the preserved state").
4. **The confidence block's `unknown` list first**, per [PROTOCOL] §3.
5. **Sweep completeness** — whether node 2 actually swept ALL bug sources or quietly narrowed scope.

# Exit Criteria (quantitative, overrides protocol §12)

- Mutation score **≥ 70%** on scoped files (post-remediation, equivalent-mutant-adjusted, sampling rate reported if < 100%).
- Line coverage on scoped files ≥ 80%; every node-1 priority-1 gap has at least one non-vacuous test.
- **Zero fixed bugs from `bug_sources` without a regression test** (or explicit `unverifiable-regression` finding).
- All 10 chaos scenarios executed and graded (or loop ran with `--no-chaos` / unset `chaos_env`, recorded in artifact); zero scenarios executed against production hosts.
- Full suite green on the loop branch; no flaky test committed (3× stability runs).
- Final confidence ≥ 0.85; governance row written.

# Failure Routing (only deviations from protocol §4)

- Property counterexample or chaos F-grade revealing exploitable behavior → treat as **Security Failure**: HALT, CRITICAL artifact, human approval to continue.
- Mutation tool crash → Tool Failure, but the fallback is agentic AST mutation (node 5), not skip.
- Chaos environment contamination (state not restored) → Workflow Failure; resume only after clean-environment re-verification, never mid-campaign.

# Approval Gates (only deviations from protocol §10)

- **Node 7 chaos injection: even in sandbox/dev, the operator approves the scenario list + target environment before the first injection.**
- Chaos NEVER runs against production infrastructure without explicit multi-step human approval — and this loop's default answer to that request is to refuse and hand back a plan instead.
- Committing generated tests to the loop branch: automatic (per [PROTOCOL] §10 — tests are auto-writable). Merging: operator, as always.

# RUN PROMPT

```
Run LOOP-12 (Test Generation, Mutation & Chaos).
Read PROTOCOL.md (this collection's shared protocol), then
LOOP-12-test-mutation-chaos.md, and execute its Execution DAG
end-to-end under protocol rules.
target: <repo> (scope: <dir>)
chaos_env: <sandbox/dev env name, or omit to skip chaos>
bug_sources: default (all _loopstate artifacts, 30 days)
Work on branch loop/LOOP-12-<date>. Checkpoint after every node. Hand me the
AUDIT-ARTIFACT and the chaos scenario list for approval BEFORE node 7 runs.
Do not merge anything.
```
