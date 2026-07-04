---
name: loop-adversarial-review
loop-id: LOOP-11
description: The debate engine — generator vs adversarial reviewer(s), FREE-MAD default, silent-agreement detection — callable standalone or inline by every other loop
domain: Cross-cutting engine
risk-class: read-only
default-debate: FREE-MAD
model-tiers: {scan: cheap, audit: mid, verdict: reasoning}
---

# Mission

Be the adversarial refinement engine for the whole loop system. A single agent drifts
toward confident wrongness; this loop forces a **generator vs adversarial reviewer**
debate that hunts edge cases, logic flaws, vulnerabilities, and contradictions until
termination conditions are met. It is **read-only**: it produces verdicts, critiques,
and scores — it never mutates code itself (the calling loop applies fixes on its own
branch). LOOP-11 runs two ways: **inline mode**, as [PROTOCOL] step 4 (the Adversarial
Check) for every other loop; and **standalone mode**, pointed by the operator at any
artifact, PR, plan, or diff. Confidence-block rule: reviewers attack the `unknown` list first.

# Trigger (when the operator runs this)

- **Inline:** automatically, as [PROTOCOL] §1 step 4 of every LOOP-01…LOOP-15 node/verdict.
- **Standalone:** `Run LOOP-11 on <artifact|PR|plan|diff>` when the operator wants an
  independent hostile review of anything — a design doc, a PR, an audit artifact, a plan.

# Inputs (target repo/dir, scope flags)

- `target`: the artifact under review — a file, PR number, plan, diff, or a calling loop's
  node output + its declared completion criteria.
- `--protocol <FREE-MAD|MAD|MoE|RA-CR>`: override the debate protocol (default FREE-MAD).
- `--rounds <N>`: round budget override (default 2, max 4).
- `--persona <security|architecture|correctness>`: bias the reviewer's hostile lens.
- `--mode <inline|standalone>`: defaults to inline when called by a loop, standalone otherwise.

# Preconditions

- The generator's output + explicit completion criteria are available (inline: passed by the
  caller; standalone: the operator names the artifact and the bar it must clear).
- For code targets: a code-graph/knowledge-graph tool if available, plus relevant architecture
  docs, loaded so the reviewer can trace claims rather than pattern-match.
- Read-only: no Edit/Write to target code. The engine emits a verdict + critique artifact only.

# Debate Protocols (from source §"Architecting the Multi-Agent Debate")

| Protocol | When to use |
|----------|-------------|
| **Consensus (standard MAD)** | Low-stakes: labeling, lint-class findings, cheap code gen — rapid convergence via majority vote is acceptable. Prone to Silent Agreement. |
| **FREE-MAD** (consensus-free, score-based) — **DEFAULT** | High-stakes: architecture audits, security reviews, complex problems. Reviewers flag flaws independently with no forced agreement; a score-based mechanism evaluates all intermediate results across rounds. Diversity of thought prevents systemic blind spots. |
| **MoE self-debate** | Cheap rapid passes: parameter-efficient self-debate inside one backbone model — self-critique before handoff, low latency/token cost. |
| **RA-CR** (rank-adaptive cross-round) | Long-horizon refactors: an external judge dynamically reorders and selectively silences reviewer nodes per round to force higher peer-referencing and argument diversity. |

# Execution DAG (numbered nodes, [P] = parallelizable)

1. Ingest — generator output + completion criteria + confidence block.
2. Select protocol & personas — pick from the table; spin up reviewer(s) with hostile persona(s).
3. [P] Reviewer critique round — attack `unknown` list, then edge cases/logic/security/contradictions.
4. Silent-agreement check — unanimous + no cited evidence → force a hostile extra round.
5. Handoff to generator — generator self-reviews the critique and revises (revision text only).
6. Reviewer re-attack — score the revision; continue rounds until termination (budget/threshold).
7. Score-based verdict — aggregate round scores → PASS | REVISE | REJECT + final confidence block.
8. Emit — verdict artifact (inline: return to caller as step-4 result; standalone: to operator).

# Node Specs

### Node 1 — Ingest
- **Action:** Load the generator output, its stated completion criteria, and its confidence block.
- **Tools:** Read; a code-graph/knowledge-graph tool if available, for code targets.
- **Failure conditions:** missing completion criteria → cannot judge (Workflow Failure → ask caller/operator).
- **Output artifact:** `_loopstate/loop-11/<run-id>/round-0-intake.md` (`<run-id>` is an ISO-8601 timestamp).

### Node 2 — Select protocol & personas
- **Action:** Choose protocol (default FREE-MAD; MAD for low-stakes; MoE for cheap self-pass;
  RA-CR for long-horizon refactors). Instantiate reviewer(s) with a hostile persona — **skeptical
  senior security architect** and/or **hostile penetration tester** whose sole objective is to find
  edge cases, logic flaws, vulns, and contradictions.
- **Tools:** none (configuration).
- **Failure conditions:** protocol/persona mismatch to stakes → re-select.
- **Output artifact:** `config.md` (protocol | personas | round budget | score threshold).

### Node 3 — Reviewer critique round [P]
- **Action:** Each reviewer attacks the confidence block's `unknown` list first, then hunts edge
  cases, logic flaws, security vulnerabilities, and architectural contradictions. Every critique
  must cite evidence (AST, trace, test, a code-graph tool if available) — unsupported assertions are discarded.
- **Tools:** a code-graph/knowledge-graph tool if available, `ast-grep`, Read, sandbox reasoning.
- **Failure conditions:** reviewer produces no cited evidence → its critique is void this round.
- **Output artifact:** `round-<N>-critique.md` (issue | severity | cited evidence | score delta).

### Node 4 — Silent-agreement check
- **Action:** If all reviewers agree in round 1 with **no cited evidence**, treat it as conformity,
  not consensus (PROTOCOL.md §11) — inject a fresh hostile persona and force one additional round.
- **Tools:** none (control logic).
- **Failure conditions:** repeated unevidenced agreement → escalate reviewer to reasoning tier.
- **Output artifact:** appended to `config.md` (silent_agreement: true|false).

### Node 5 — Handoff to generator
- **Action:** Generator takes the full critique, self-reviews it, and produces a revision
  (proposed changes as text/diff description — LOOP-11 does not commit anything).
- **Tools:** none from LOOP-11 (generator is the caller's executor).
- **Failure conditions:** generator ignores a cited CRITICAL → reviewer auto-rejects next round.
- **Output artifact:** `round-<N>-revision.md`.

### Node 6 — Reviewer re-attack
- **Action:** Reviewer scores the revision against completion criteria; loop rounds 3→8 until a
  termination condition: score ≥ threshold (PASS), score plateau/regression (REJECT), or round
  budget hit (default 2, max 4).
- **Tools:** a code-graph/knowledge-graph tool if available, Read.
- **Failure conditions:** round budget exhausted without PASS → verdict = REVISE or REJECT with artifact.
- **Output artifact:** `round-<N>-score.md`.

### Node 7 — Score-based verdict
- **Action:** Aggregate per-round scores → **PASS** (criteria met, confidence ≥ caller's threshold),
  **REVISE** (fixable gaps remain, under budget), or **REJECT** (fundamental flaw / budget exhausted).
  Emit final confidence block. FREE-MAD uses the score history across all rounds, not a single final vote.
- **Tools:** none (synthesis).
- **Failure conditions:** verdict confidence < caller's exit threshold → REVISE/REJECT, never PASS.
- **Output artifact:** `verdict.md`.

### Node 8 — Emit
- **Action:** **Inline mode:** return the verdict + critique as the caller's [PROTOCOL] step-4 result;
  on REJECT the caller resets to its failed node with this critique as new context ([PROTOCOL] §1).
  **Standalone mode:** hand the operator `AUDIT-ARTIFACT.md` with verdict, findings, and scores.
- **Tools:** Write on `_loopstate` artifacts only.
- **Output artifact:** `AUDIT-ARTIFACT.md` (PROTOCOL.md §13; findings table + verdict + score history).

# Adversarial Check (reviewer persona + what it attacks)

LOOP-11 *is* the adversarial check — so its self-check is the meta-guard: a second-order verifier
confirms the reviewer actually cited evidence, actually attacked the `unknown` list first, and did
not rubber-stamp. If the reviewer itself shows silent agreement or uncited verdicts, the meta-guard
forces protocol escalation (MAD→FREE-MAD→RA-CR) and a reasoning-tier reviewer.

# Exit Criteria (quantitative, overrides PROTOCOL.md §12)

- Verdict ∈ {PASS, REVISE, REJECT} with a numeric score history across all rounds.
- Round budget respected: default 2, hard max 4 rounds.
- PASS requires: all reviewer-cited CRITICAL/HIGH issues resolved AND final confidence ≥ caller's
  declared threshold (default ≥ 0.85).
- Silent-agreement guard fired whenever round-1 was unanimous with no cited evidence.
- Every critique that drove the verdict carries cited evidence; governance row written.

# Failure Routing (only deviations from PROTOCOL.md §4)

- Generator repeatedly ignores a cited CRITICAL → verdict REJECT; inline caller routes to its own
  Security/Architecture failure handling (PROTOCOL.md §4). LOOP-11 never patches.
- Reviewer degeneration (uncited/rubber-stamp) → protocol + tier escalation, not a PASS.

# Approval Gates (only deviations from PROTOCOL.md §10)

- Read-only + verdict = automatic. LOOP-11 produces no branch and no code changes.
- Acting on the verdict (applying fixes, merging) belongs to the calling loop or the operator —
  never to LOOP-11.

# RUN PROMPT (verbatim block the operator pastes or invokes)

```
Run LOOP-11 on <artifact|PR#|plan|diff> (standalone).
Follow LOOP-11-adversarial-review.md under [PROTOCOL] (see PROTOCOL.md in this repo).
Use FREE-MAD, hostile senior-security-architect persona, round budget 2 (max 4).
Read-only; attack the unknown list first; deliver verdict + AUDIT-ARTIFACT. Do not modify code.
```

Inline invocation (used by every other loop as [PROTOCOL] step 4):

```
LOOP-11 inline: protocol=FREE-MAD persona=<security|architecture|correctness>
input=<node output + completion criteria + confidence block>  → return PASS|REVISE|REJECT + critique.
```
