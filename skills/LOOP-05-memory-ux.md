---
name: loop-memory-ux
loop-id: LOOP-05
description: CoALA four-tier memory audit (working/episodic/semantic/procedural), tenant isolation, temporal decay, intentional forgetting, GDPR erasure UI
domain: Memory & Cognitive Architecture
risk-class: branch-mutating
default-debate: FREE-MAD
model-tiers: {scan: cheap, audit: mid, verdict: reasoning}
---

# Mission

Audit and harden the application's agentic memory architecture against the CoALA
four-tier model, then remediate defects on a `loop/loop-05-<date>` branch. Unbounded
memory produces retrieval noise, behavioral drift, runaway token cost, and — worst —
cross-tenant confidentiality breaches. This loop verifies each memory tier is
segmented, bounded, correctly retrieved, and privacy-compliant, and that end users
have granular control over what the system remembers about them. It is
branch-mutating: it fixes what it finds, but only through the checkpoint → modify →
test → verify → commit-on-branch discipline (PROTOCOL.md §6), and it hands every bug
a permanent regression test (LOOP-12 handoff).

# Trigger (when the operator runs this)

- Monthly deep sweep of memory + cognitive architecture (recommended baseline cadence).
- After any change to the memory store schema, retrieval pipeline, or summarizer.
- After a report of contradictory recall, stale facts, cross-user bleed, or timeouts.
- Before shipping any new user-facing memory/erasure UI.

# Inputs (target repo/dir, scope flags)

- `target`: repo/dir (default: your target repository).
- `scope`: e.g. `memory/`, `agent/memory/`, retrieval + summarizer modules, memory UI.
- `--tier <working|episodic|semantic|procedural>`: audit a single tier.
- `--no-fix`: audit-only, emit artifact without opening a branch.

# Preconditions

- Clean worktree on `origin/main` truth; `git status` sampled (PROTOCOL.md §2).
- Code-graph/knowledge-graph tool loaded if your project has one (MCP server, IDE
  semantic index, or equivalent); memory-store and retrieval modules located via that
  tool or an architecture/codebase map.
- Read any existing architecture docs or ADRs for the memory store, retrieval
  pipeline, and user memory UI before touching code.
- DB/schema changes require explicit human approval (PROTOCOL.md §10) — this loop
  proposes migrations as artifacts; it does not apply them.
- Never write to any directory your project designates off-limits for automated
  agents (e.g., a memory/context vault or secrets store).

# Execution DAG (numbered nodes, [P] = parallelizable)

1. Scope & memory-map — locate each of the four tiers + retrieval + isolation filters.
2. [P] Working-memory audit — volatility + aggressive end-of-session pruning.
3. [P] Episodic-memory audit — compressed summaries + similarity×temporal-decay retrieval.
4. [P] Semantic-memory audit — entity/graph retrieval + explicit fact overwrite.
5. [P] Procedural-memory audit — version-controlled prompts/skills + failure-driven evolution.
6. Security audit — cryptographic segmentation + multi-tenant isolation (user_id/tenant_id).
7. Privacy audit — intentional forgetting/temporal-threshold pruning + GDPR erasure UI.
8. Consolidate + confidence block; open `loop/loop-05-<date>`, buffer fixes, checkpoint.
9. Remediate + adversarial verify (LOOP-11 inline); regression test per bug; commit on branch.

# Node Specs

### Node 1 — Scope & memory-map
- **Action:** Enumerate the four tiers, the retrieval pipeline, the tenant-isolation filter
  layer, and the user memory UI. Build a memory topology map.
- **Tools:** Code-graph/knowledge-graph tool if available (MCP server, IDE semantic index)
  for structural lookup, otherwise `ast-grep`; Read on schema/migrations.
- **Failure conditions:** a tier is absent or unidentifiable → finding (Architecture Failure).
- **Output artifact:** `_loopstate/loop-05/<run-id>/memory-map.json` (`<run-id>` is an
  ISO-8601 timestamp).

### Node 2 — Working-memory audit [P]
- **Action:** Verify working memory (active state, current task, recent turns / context window)
  is volatile and **aggressively pruned at end of session**. Flag any path where working memory
  grows unbounded (→ state explosion, application timeouts) or persists past session end.
- **Tools:** `ast-grep` for session lifecycle + context assembly; a code-graph tool for
  data-flow tracing if available.
- **Failure conditions:** unbounded working memory = HIGH (Timeout risk class).
- **Output artifact:** `working-mem.md`.

### Node 3 — Episodic-memory audit [P]
- **Action:** Verify episodes are stored as **compressed summaries, not raw transcripts**, and
  that retrieval combines vector similarity **with a temporal-decay function** so older events
  fade in relevance. Flag raw-transcript storage and pure-recency or pure-similarity retrieval.
- **Tools:** Read on summarizer + retrieval ranking; a code-graph/knowledge-graph tool if
  available.
- **Failure conditions:** raw transcript storage = MEDIUM; missing temporal decay = MEDIUM.
- **Output artifact:** `episodic-mem.md`.

### Node 4 — Semantic-memory audit [P]
- **Action:** Verify durable facts/entities/preferences are retrieved by **entity identity /
  graph traversal**, not vector similarity alone, and that facts are **explicitly updatable and
  overwritable** to prevent contradictory hallucinations. Flag append-only fact stores that
  accumulate conflicting values with no overwrite path.
- **Tools:** A code-graph/knowledge-graph tool for entity traversal if available (otherwise
  manual search); Read on fact store + upsert logic.
- **Failure conditions:** no explicit fact-overwrite path = HIGH (contradiction/hallucination risk).
- **Output artifact:** `semantic-mem.md`.

### Node 5 — Procedural-memory audit [P]
- **Action:** Verify prompts, skills, and workflows are **version-controlled** and that the system
  supports dynamic evolution — refining procedures automatically from analysis of past failures.
  Flag unversioned prompt edits and any procedural change with no failure-analysis provenance.
- **Tools:** A code-graph tool if available, plus `git log` on prompt/skill files; Read.
- **Failure conditions:** unversioned procedural memory = MEDIUM.
- **Output artifact:** `procedural-mem.md`.

### Node 6 — Security: segmentation & multi-tenant isolation
- **Action:** Verify cryptographic segmentation and that **every** memory query explicitly filters
  by `user_id` / `tenant_id`. Trace each retrieval path from request → store; any path that can
  return another tenant's memory is a **critical confidentiality breach**. Probe with a crafted
  two-tenant read in a sandbox to confirm isolation.
- **Tools:** A code-graph/data-flow tracing tool if available (otherwise manual trace) from
  request → query; Read on RLS/filter middleware.
- **Failure conditions:** any unfiltered cross-tenant read path = **Security Failure** → HALT
  (PROTOCOL.md §4).
- **Output artifact:** `isolation-audit.md` (path | filters user_id? | filters tenant_id? | severity).

### Node 7 — Privacy: forgetting & GDPR erasure
- **Action:** Verify intentional-forgetting / temporal-threshold pruning exists (memories unaccessed
  beyond a threshold are pruned). Verify the UI gives users **granular view / edit / delete** of their
  semantic AND episodic memories (GDPR right-to-erasure). Flag missing erasure endpoints and any
  "delete" that only hides rather than removes.
- **Tools:** Read on pruning job + memory-UI components; a code-graph tool for UI→store trace
  if available.
- **Failure conditions:** no user-facing erasure of semantic+episodic memory = HIGH (compliance).
- **Output artifact:** `privacy-audit.md`.

### Node 8 — Consolidate, branch, checkpoint
- **Action:** Merge findings, severity-rank, attach a confidence block. Open `loop/loop-05-<date>`,
  buffer fixes as structured diffs; propose (never apply) any DB migration as an artifact for
  human approval.
- **Tools:** git branch/checkpoint; a secret-scanning guard (e.g., gitleaks, trufflehog, or an
  equivalent pre-commit check) before any commit.
- **Failure conditions:** loop confidence < 0.85 → cannot PASS.
- **Output artifact:** `findings.md` + `proposed-migrations/` (unapplied).

### Node 9 — Remediate & adversarial verify
- **Action:** Apply code-level fixes on branch, generate a permanent regression test per bug
  (LOOP-12), run tests, then LOOP-11 inline (FREE-MAD) verifies. Isolation fixes get a dedicated
  cross-tenant negative test. Rollback if reviewer rejects twice (PROTOCOL.md §6).
- **Tools:** Edit/Write on branch, test runner, LOOP-11 engine.
- **Failure conditions:** Security Failure → HALT + human; Migration change → human approval only.
- **Output artifact:** `AUDIT-ARTIFACT.md` (PROTOCOL.md §13).

# Adversarial Check (reviewer persona + what it attacks)

Reviewer = **hostile privacy/security architect**. It attacks the confidence block's `unknown`
list first, then: every "isolated" query (demands the user_id/tenant_id filter and a two-tenant
negative test); every "pruned" claim (asks for the temporal threshold and proof unbounded growth
can't recur); every "overwritable fact" claim (constructs a contradiction and asks which value
wins); and the erasure UI (asks whether delete truly removes from semantic AND episodic stores).
Silent-agreement guard per PROTOCOL.md §11.

# Exit Criteria (quantitative, overrides protocol §12)

- Zero cross-tenant memory-bleed paths — any such path → FAIL/HALT, no exceptions.
- 100% of retrieval paths provably filter user_id/tenant_id.
- All four tiers present and conformant (or defects operator-deferred with rationale).
- User-facing view/edit/delete for semantic AND episodic memory exists or is remediated.
- Every applied fix carries a permanent regression test; test pass ≥ 99% on touched surface.
- Final loop confidence ≥ 0.85; governance row written.

# Failure Routing (only deviations from protocol §4)

- Cross-tenant read path = **Security Failure** → HALT LOOP + human approval (PROTOCOL.md §4).
- Any schema/migration need = surfaced as a proposed artifact; **Human explicit approval** required
  (PROTOCOL.md §10) — the loop never self-applies a DB migration.

# Approval Gates (only deviations from protocol §10)

- Code fixes on branch: adversarial reviewer sign-off, then operator merges.
- DB migration / schema change: human explicit approval (proposed as unapplied artifact).
- Memory data deletion during testing: use synthetic tenants only; never delete real user memory.

# RUN PROMPT (verbatim block the operator pastes or invokes)

```
Run LOOP-05 on <your-repo> (scope: memory/ + retrieval + memory UI).
Follow LOOP-05-memory-ux.md under the protocol defined in PROTOCOL.md (same repo).
Branch-only (loop/loop-05-<date>); propose migrations as artifacts, do not apply them.
Any cross-tenant memory bleed: halt and surface to me. Deliver AUDIT-ARTIFACT.
```
