---
name: loop-agent-runtime
loop-id: LOOP-04
description: Audits the agentic infra itself — MCP/tool schema strictness, least-privilege agency, prompt-injection surfaces, untrusted-data boundaries
domain: Agent Runtime & Tooling
risk-class: read-only→branch
default-debate: FREE-MAD
model-tiers: {scan: cheap, audit: mid, verdict: reasoning}
---

# Mission

Audit the operator's own agentic runtime — the tools, MCP servers, skills, agent
definitions, and context pipelines that make the agents act — and certify it against
the OWASP Agentic Top 10 and a schema-gated least-privilege doctrine (enforced in
Node 3 below). The loop is **read-only by default**: it produces an
AUDIT-ARTIFACT of findings. Only when the operator approves a fix pass does it
open a `loop/loop-04-<date>` branch and buffer typed remediations (tightened
schemas, permission down-scopes, sanitization shims) as structured artifacts that
pass through the LOOP-11 adversarial gate before any commit. The system audits
its own hands, and it may never grant itself more reach in the process.

# Trigger (when the operator runs this)

- Monthly deep sweep of agent runtime + tooling (or per your team's own audit cadence).
- After any change to the MCP catalog, agent tool definitions, or your skills/tool-config
  directory (e.g. `~/.claude/skills/` for Claude Code, or the equivalent in your agent framework).
- Whenever a new tool with a shell, filesystem, network, or DB surface is added.
- Before shipping a new agent or MCP server to a production host.

# Inputs (target repo/dir, scope flags)

- `target`: repo or dir (default: the current repository).
- `scope`: e.g. `mcp/`, `agents/`, `skills/`, `agent/tools/` — restricts the DAG.
- `--fix` (optional): after read-only audit, open branch and buffer remediations.
- `--tool <name>`: audit a single tool/MCP schema end-to-end.

# Preconditions

- Clean worktree on your trunk branch (`main`/`master`) — standing rule; `git status`
  sampled per PROTOCOL.md §2.
- If you maintain a code-graph or knowledge-graph tool for this repo (an MCP server, an
  IDE's semantic index), load it and identify the MCP-catalog and agent-tool
  modules/communities; otherwise use `ast-grep`/Semgrep/LSP "find references" to map the
  same territory by hand.
- Read any existing architecture docs/ADRs for the MCP catalog, agent tool registry, and
  skills loader before reading any code — highest-fan-in modules carry the highest ripple risk.
- No write action executes until node 8, and only under `--fix` + operator approval.

# Execution DAG (numbered nodes, [P] = parallelizable)

1. Scope & inventory — enumerate every tool/MCP/skill/agent-definition in scope.
2. [P] Tool/MCP schema-strictness audit — typed inputs/outputs; flag open-ended tools.
3. [P] Excessive-Agency / least-privilege audit — read-only defaults, write buffering.
4. [P] Context-management audit — typed AST tools mandated, grep-for-comprehension forbidden.
5. [P] Prompt-injection surface audit — every external-context pipeline into the LLM.
6. Untrusted-data boundary audit — data-vs-instruction separation, supply-chain provenance.
7. Consolidate findings → severity-rank → confidence block per finding.
8. Adversarial Check (LOOP-11 inline, FREE-MAD) → verdict; under `--fix`, buffer + gate remediations.

# Node Specs

### Node 1 — Scope & inventory
- **Action:** Build the runtime manifest: for each tool, record name, transport (MCP/bash/http),
  declared schema location, permission grants, and which agents/skills mount it.
- **Tools:** your code-graph/knowledge-graph tool's query/community lookup (if you have one),
  `ast-grep`, Read on schema/registry files.
- **Failure conditions:** tool with no discoverable definition (Tool Failure → log, continue);
  scope resolves to zero tools (Workflow Failure → re-scope with operator).
- **Output artifact:** `_loopstate/loop-04/<run-id>/inventory.json` (`run-id` is an ISO 8601 timestamp).

### Node 2 — Tool/MCP schema-strictness audit [P]
- **Action:** Verify every tool declares explicit strongly-typed inputs AND outputs
  (JSON Schema / typed signature). Flag ambiguous, open-ended tools — an unconstrained
  shell, a raw `exec`, a `query(sql: string)` with no allow-list, a "do anything" catch-all —
  as HIGH minimum. Enum/pattern/maxLength constraints on free-text fields are required.
- **Tools:** `ast-grep`/Semgrep over tool registries, Read on MCP manifests.
- **Failure conditions:** untyped I/O = finding; unconstrained shell = CRITICAL candidate.
- **Output artifact:** `schema-findings.md` (id | tool | typed? | constraints | severity).

### Node 3 — Excessive-Agency / least-privilege audit [P]
- **Action:** Map each agent's granted capabilities against its actual task (OWASP Agentic
  Top 10 — Excessive Agency). Enforce read-only defaults. Any write/delete/network/DB
  capability must be (a) justified, (b) buffered as a structured artifact, and (c) routed
  through an independent adversarial threat-detection gate before commit. Flag standing
  write grants, ambient credentials, and auto-commit paths.
- **Tools:** call-graph traversal from agent → tool → side-effect sink (your code-graph/
  knowledge-graph tool if you have one, otherwise manual or LSP-assisted tracing); Read on
  permission config.
- **Failure conditions:** unbuffered write path = Security Failure → HALT per PROTOCOL.md §4.
- **Output artifact:** `agency-map.md` (agent | capability | justified? | buffered? | gate?).

### Node 4 — Context-management audit [P]
- **Action:** Verify the runtime mandates typed programmatic comprehension tools
  (`ast-grep`, Semgrep) and forbids string `grep` for logic comprehension in agent prompts
  and skill instructions. Flag any pipeline that bulk-cats files into context (window flooding
  → degraded reasoning). Confirm compression/summarization of inter-node handoffs.
- **Tools:** Read on skill/agent prompt bodies; `ast-grep` for tool-call patterns.
- **Failure conditions:** grep-for-comprehension mandated in a prompt = MEDIUM finding.
- **Output artifact:** `context-hygiene.md`.

### Node 5 — Prompt-injection surface audit [P]
- **Action:** Enumerate EVERY pipeline feeding external context to the LLM: user uploads,
  scraped web pages, third-party API responses, file contents, tool outputs, MCP payloads.
  For each, verify sanitization exists and that untrusted content is fenced as *data*, never
  interpreted as instructions. Probe for Agent Goal Hijacking (injected directives override
  legitimate goal) and Agentic Supply-Chain vulnerabilities (poisoned upstream tool/data).
- **Tools:** data-flow traversal from external source → prompt assembly (your code-graph/
  knowledge-graph tool if available, otherwise manual trace); Read.
- **Failure conditions:** unsanitized external source reaching the prompt = HIGH/CRITICAL.
- **Output artifact:** `injection-surfaces.md` (source | sink | sanitized? | fenced? | severity).

### Node 6 — Untrusted-data boundary audit
- **Action:** Assert the hard invariant: the agent NEVER treats untrusted data as executable
  instructions. Verify a structural data/instruction boundary (delimiters, role separation,
  content-type tagging) rather than a stylistic hope. Check tool-result provenance and that
  no tool re-injects attacker-controlled text into a privileged instruction slot.
- **Tools:** your code-graph/knowledge-graph tool (if available) + manual read of
  prompt-assembly code.
- **Failure conditions:** missing structural boundary on a reachable untrusted source = CRITICAL.
- **Output artifact:** `boundary-audit.md`.

### Node 7 — Consolidate & score
- **Action:** Merge node 2–6 findings, dedupe, severity-rank (CRITICAL/HIGH/MEDIUM/LOW),
  attach a confidence block per finding and a loop-level confidence block.
- **Tools:** none (synthesis).
- **Failure conditions:** loop confidence < 0.85 → cannot PASS (PROTOCOL.md §3).
- **Output artifact:** `findings.md`.

### Node 8 — Adversarial Check & optional remediation
- **Action:** Invoke LOOP-11 inline (FREE-MAD, hostile senior security architect persona)
  to attack the `unknown` list and challenge every "sanitized/typed/least-privilege" claim.
  If `--fix`: open `loop/loop-04-<date>`, buffer typed remediations, route each through the
  same adversarial gate, generate a permanent regression test per fix (LOOP-12 handoff),
  commit on branch only. Never merge — hand findings to operator.
- **Tools:** LOOP-11 engine, Edit/Write on branch (only under `--fix`), a pre-commit
  secret-scanning guard (e.g. gitleaks, trufflehog, or an equivalent skill).
- **Failure conditions:** reviewer rejects twice → rollback branch (PROTOCOL.md §6); Security
  Failure anywhere → HALT + human.
- **Output artifact:** `AUDIT-ARTIFACT.md` (PROTOCOL.md §13 shape).

# Adversarial Check (reviewer persona + what it attacks)

Reviewer = **hostile pentester + skeptical senior security architect**. It attacks, in order:
the confidence block's `unknown` list; every tool claimed "strongly typed" (demands the
schema); every "read-only" claim (hunts a hidden write/exec path); every "sanitized" external
source (crafts a benign injection string and asks where it lands); and any data/instruction
boundary that is stylistic rather than structural. Silent-agreement guard: if reviewers agree
round 1 with no cited evidence, force one hostile extra round (PROTOCOL.md §11).

# Exit Criteria (quantitative, overrides PROTOCOL.md §12)

- Zero unresolved CRITICAL findings (unbuffered write path, unsanitized injection sink,
  missing data/instruction boundary) — any CRITICAL → FAIL-with-artifact.
- 100% of in-scope tools have typed inputs AND outputs, or are explicitly operator-deferred.
- 100% of external-context pipelines have a documented sanitization + fencing status.
- Final loop confidence ≥ 0.85; governance row written (PROTOCOL.md §8).
- Under `--fix`: every applied remediation carries a permanent regression test; branch only.

# Failure Routing (only deviations from PROTOCOL.md §4)

- Any unbuffered write capability or unsanitized reachable injection sink = **Security Failure**
  → HALT LOOP, emit CRITICAL artifact, require human approval to continue (PROTOCOL.md §4).
- Tool with no schema = Tool Failure → log, continue, flag in artifact (do not self-repair silently).

# Approval Gates (only deviations from PROTOCOL.md §10)

- Read-only audit + artifact: automatic.
- `--fix` remediations: adversarial reviewer sign-off + operator approval before any commit;
  merge to main = operator only. Permission/scope widenings are NEVER self-applied.

# RUN PROMPT (verbatim block the operator pastes or invokes)

```
Run LOOP-04 on <your-repo> (scope: mcp/ + agents/ + skills/).
Follow skills/LOOP-04-agent-runtime.md under PROTOCOL.md in this repo.
Read-only first; deliver AUDIT-ARTIFACT. Do not open a fix branch unless I add --fix.
Any Security Failure: halt and surface to me.
```
