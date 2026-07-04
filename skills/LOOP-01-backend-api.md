---
name: loop-backend-api
loop-id: LOOP-01
description: AST-driven backend audit: OWASP API Top 10 (BOLA/BOPLA/SSRF/auth/resource limits), N+1 queries, dead code, logging hygiene
domain: Backend & API Architecture
risk-class: branch-mutating
default-debate: FREE-MAD
model-tiers: {scan: cheap, audit: mid, verdict: reasoning}
---

# Mission

Audit and harden the backend and API layer of a target repository as a semantic graph, not a pile of text. This loop builds an AST-level understanding of the router-to-ORM data flow, hunts structural performance defects (N+1 queries, missing indexes, blocking synchronous calls, unbounded fetches), prunes dead code and duplicate helpers, validates the service against the OWASP API Security Top 10 under a Zero-Trust posture, and enforces structured logging with correlation IDs. All mutations land on an isolated loop branch; every confirmed bug leaves behind a permanent regression test (handed to LOOP-12).

# Trigger (when the operator runs this)

- Weekly sweep cadence (or your own audit cadence).
- Before any backend release, after a major router/ORM refactor, or after new endpoints ship.
- On demand: `Run LOOP-01 on <repo> (scope: <api dir>)`.

# Inputs (target repo/dir, scope flags)

- `target`: repo root (your primary repo, judged from a clean origin/main worktree).
- `scope`: backend directories (e.g. `api/`, `server/`, `services/`). Default: all server-side code.
- `--security-only`: run only nodes 5–6 (OWASP + logging) when time-boxed.
- `--no-mutate`: audit-and-report mode; skip nodes 7–8 fixes, emit findings only.

# Preconditions

- Git worktree clean; loop branch `loop/LOOP-01-<YYYY-MM-DD>` created from origin/main before node 3.
- A code-graph/knowledge-graph tool if your project has one (MCP server, IDE semantic index) — otherwise `ast-grep`/Semgrep/LSP "find references".
- `ast-grep` or Semgrep available on PATH (verify with `Bash: ast-grep --version`); if neither exists, HALT with Tool Failure — this loop never falls back to grep for comprehension.
- Test suite runnable locally (`pytest` / `npm test` per repo) so fixes can be verified.

# Execution DAG (numbered nodes, [P] = parallelizable)

1. Semantic graph construction — build/refresh the AST (and code-graph tool, if available) picture of the scope.
2. [P] ORM & query audit — N+1, missing indexes, blocking sync queries.
3. [P] Boundedness audit — pagination, rate limiting, caching layers.
4. [P] Dead-code & duplicate-helper prune plan.
5. OWASP API Top 10 verification (depends on node 1 graph).
6. Structured-logging & error-handling audit (depends on node 1 graph).
7. Fix application on loop branch (consumes nodes 2–6 findings; ordered by severity).
8. Regression-test generation + full test run (every confirmed bug → permanent test; hand-off manifest for LOOP-12).

Nodes 2, 3, 4 run in parallel after node 1. Nodes 5 and 6 run sequentially after node 1 (they share the router→handler→ORM trace). Node 7 waits for all findings; node 8 closes the loop.

# Node Specs

**Node 1 — Semantic graph construction.**
Action: if a code-graph/knowledge-graph tool is available, query it for every module touching the scope and identify high-fan-in modules (highest ripple risk); otherwise read existing architecture docs/ADRs for the scope. Then run `ast-grep` structural queries to enumerate all routers, handlers, middleware chains, and ORM entry points, producing a typed route table: `method | path | handler | auth middleware | ORM models touched`.
Tools: a code-graph/knowledge-graph tool if available, ast-grep, Read (targeted files only), Bash. Grep is permitted solely to locate literal strings (route prefixes, error codes) — never to trace logic.
Failure conditions: knowledge-graph tool unreachable (Tool Failure — retry once, then continue in ast-grep-only mode); route table empty for a scope known to contain endpoints (Logic Failure — re-scope).
Output artifact: `_loopstate/LOOP-01/<run-id>/node-1-route-graph.json` (route table + high-fan-in list) with confidence block.

**Node 2 — ORM & query audit [P].**
Action: for each ORM entry point in the node-1 table, use ast-grep patterns to detect (a) query calls inside loops or per-item serializers (N+1), (b) filters/sorts on columns with no corresponding index in migrations or schema files, (c) synchronous/blocking query calls inside async handlers or event-loop threads. Rank each by request-path heat (routes with auth middleware and list semantics first).
Tools: ast-grep, Read (migration/schema files), a code-graph/knowledge-graph tool if available (model→migration linkage), Bash.
Failure conditions: cannot resolve ORM dialect (Dependency Failure — pin, retry); >200 raw matches (Context Overflow — compress to per-file summaries, re-scope).
Output artifact: `node-2-orm-findings.md` — findings table `id | severity | file:line | pattern | suggested fix`.

**Node 3 — Boundedness audit [P].**
Action: trace every list-returning endpoint recursively from router to data source and verify: pagination logic present (limit/offset or cursor), rate-limiting middleware attached (per-route or global), and caching layer (Redis/Memcached/in-proc) used for expensive repeated reads. Flag any endpoint that can load an unbounded dataset into memory.
Tools: ast-grep (structural trace), a code-graph/knowledge-graph tool if available (path queries between router and ORM), Read.
Failure conditions: ambiguous middleware resolution order (Logic Failure — escalate to mid tier and re-trace).
Output artifact: `node-3-boundedness.md` — per-endpoint matrix `endpoint | paginated? | rate-limited? | cached? | unbounded-risk`.

**Node 4 — Dead-code & duplicate-helper prune plan [P].**
Action: from the call graph (a code-graph/knowledge-graph tool if available, or ast-grep reachability tracing otherwise), list functions/modules with zero inbound edges (excluding entry points, CLI hooks, scheduled jobs, and dynamically dispatched handlers — verify each exclusion explicitly). Detect near-duplicate helper functions via ast-grep structural similarity and propose a single consolidated home for each cluster. Output is a prune PLAN, not a deletion — deletions happen only in node 7 after adversarial review, and never delete anything reachable via reflection/dynamic import without a manual-read confirmation.
Tools: a code-graph/knowledge-graph tool if available (inbound-edge queries), ast-grep, Read.
Failure conditions: dynamic dispatch makes reachability undecidable for a candidate (mark `unknown` in the confidence block, exclude from prune).
Output artifact: `node-4-prune-plan.md`.

**Node 5 — OWASP API Top 10 verification.**
Action: validate the scope against this table, one row at a time, using AST traces from node 1:

| Vector | Auditing directive |
|--------|--------------------|
| BOLA (Broken Object Level Authorization) | Trace the AST from each router returning user-owned data down to the ORM query. Verify an explicit check of authenticated user ID against the resource owner ID (or an RLS-equivalent) exists on the path. No check found = CRITICAL. |
| BOPLA / Mass Assignment | Audit every data-binding/deserialization site that writes to models. Require explicit allow-lists of writable fields; flag any spread/`**kwargs`/whole-body binding that could let a client set privileged fields (e.g. an admin/role/tier flag). |
| SSRF | Trace all backend functions accepting URLs or external identifiers as input to any outbound request. Require protocol enforcement, host allow-listing or network-level filtering, and input sanitization before the request executes. |
| Broken Authentication | Inspect JWT/OAuth2 config: short access-token expiry, refresh-token rotation implemented (theft detection), and zero plaintext token logging anywhere on the logging path. |
| Unrestricted Resource Consumption | Verify quotas, timeouts, and concurrency limits on computationally expensive operations; expensive routes must sit behind role checks and rate limiters (cross-check node-3 matrix). |

Also flag unsafe consumption of downstream third-party APIs (responses used without validation).
Tools: ast-grep, a code-graph/knowledge-graph tool if available, Read, Grep for literal secrets/token strings only.
Failure conditions: any CRITICAL finding = Security Failure → per PROTOCOL.md §4, HALT the loop, emit CRITICAL artifact, require operator approval to continue into node 7.
Output artifact: `node-5-owasp.md` — findings keyed to the table above, each with file:line evidence and an AST trace snippet.

**Node 6 — Structured-logging & error-handling audit.**
Action: sweep all try/catch (or except) blocks in scope. Flag: (a) silent catch blocks that swallow errors without logging, (b) handlers that return raw stack traces or internal error detail to the client, (c) log calls that are unstructured strings where the service standard is structured JSON, (d) absence of correlation/request IDs threading through the request path for distributed tracing. Propose the standardized error-envelope replacement for each.
Tools: ast-grep (catch-block patterns), Read, a code-graph/knowledge-graph tool if available (logger-module identification).
Failure conditions: none halting; low-confidence classifications go to `unknown` in the confidence block.
Output artifact: `node-6-logging.md`.

**Node 7 — Fix application (loop branch only).**
Action: apply fixes in severity order (CRITICAL → HIGH → MEDIUM), one commit per finding: `loop(LOOP-01): <node> — <change>` with confidence-block score in the body. Prunes from node 4 execute last and only for zero-inbound-edge items with confirmed reachability analysis. Run the test suite after each severity band. Respect any standing rails your project defines (e.g. never write into a protected docs/notes vault directory); run a secrets scan (gitleaks, trufflehog, or equivalent) before every commit.
Tools: Edit, Bash (tests, git), ast-grep to verify the fix pattern landed.
Failure conditions: test regression after a fix (rollback that commit per PROTOCOL.md §6, reclassify); 3 retries exhausted on a finding → mark operator-deferred.
Output artifact: commit list + `node-7-fixes.md`.

**Node 8 — Regression tests + verification run.**
Action: for every bug fixed in node 7, write a permanent regression test that fails on the pre-fix code and passes post-fix (verify both directions where cheap). Run the full suite. Emit a hand-off manifest for LOOP-12 listing new tests and any endpoints that deserve property/fuzz coverage.
Tools: Write (tests), Bash (test runner).
Failure conditions: any new test cannot demonstrate the bug (Logic Failure — rewrite test, not the assertion).
Output artifact: `node-8-tests.md` + LOOP-12 hand-off manifest + final `AUDIT-ARTIFACT.md` per PROTOCOL.md §13.

# Adversarial Check (reviewer persona + what it attacks)

Persona: **skeptical senior security architect** running FREE-MAD (no forced consensus, score-based verdict). The reviewer attacks, in order:

1. Every BOLA "pass" — demands the actual AST trace showing the owner check, not the claim that one exists.
2. The confidence block's `unknown` lists of nodes 4 and 5 first (per PROTOCOL.md §3).
3. Node-4 prune candidates — hunts for dynamic dispatch, reflection, scheduled-job registration, or serialization hooks that make "dead" code live.
4. Node-2 N+1 fixes — verifies the fix didn't trade N+1 for an unbounded JOIN or a stale cache.
5. Silent Agreement: if it concurs with everything in round 1 without citing file:line evidence, a second hostile round is forced (PROTOCOL.md §11).

Maximum 2 debate rounds; disagreements surviving round 2 are recorded as operator-deferred findings, never silently dropped.

# Exit Criteria (quantitative, overrides PROTOCOL.md §12)

- Zero unresolved OWASP-critical findings (BOLA/BOPLA/SSRF/auth CRITICALs fixed or explicitly operator-deferred with rationale).
- Zero endpoints in the node-3 matrix with `unbounded-risk = yes` remaining unaddressed.
- Zero silent catch blocks and zero raw-stack-trace-to-client paths remaining in scope.
- N+1 findings: 100% of CRITICAL/HIGH fixed; MEDIUM fixed or deferred.
- Test pass rate ≥ 99% on touched surface; every fixed bug has a named regression test; coverage on touched code ≥ 80%.
- Retry count ≤ 3 per node; final confidence ≥ 0.85; governance row written to `_loopstate/governance-ledger.md`.

# Failure Routing (only deviations from PROTOCOL.md §4)

- Security Failure in node 5 halts nodes 7–8 for the affected finding class only; non-security fixes may proceed on the branch while the operator reviews the CRITICAL artifact.
- Prune-plan reachability doubt is never a failure: route to `unknown` and exclude from mutation.

# Approval Gates (only deviations from PROTOCOL.md §10)

- PROTOCOL.md defaults apply. Additionally: any fix touching authentication middleware, token issuance, or database row-level-security/permission-adjacent code requires explicit operator approval BEFORE the commit, not just before merge.

# RUN PROMPT

```
Run LOOP-01 on <repo> (scope: <backend dir>). Follow LOOP-01-backend-api.md under [PROTOCOL] (see PROTOCOL.md in this repo). Build the semantic graph first (a code-graph/knowledge-graph tool if available, plus ast-grep — no grep-for-comprehension), run the OWASP table in full, mutate only on loop/LOOP-01-<date>, generate a regression test for every bug, and deliver AUDIT-ARTIFACT + loop branch. No merge without my approval.
```
