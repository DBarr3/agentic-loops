---
name: loop-data-layer
loop-id: LOOP-09
description: Schema evolution, migration safety, FK/orphan/cascade audit, indexes, isolation/deadlocks, backup verification (Supabase-aware)
domain: Data Architecture & Persistence
risk-class: infra-touching
default-debate: FREE-MAD
model-tiers: {scan: cheap, audit: mid, verdict: reasoning}
---

# Mission

Audit the persistence layer of your target application or platform — schema evolution and migration safety, referential integrity (foreign keys, cascades, orphan rows), index coverage against real query patterns, partitioning/replication posture, backup *restore* verification, transaction isolation and deadlock exposure, per-role RLS/grant posture, and DB-level pagination — with full Supabase awareness (used here as the concrete example backend; the same nodes generalize to any relational store via your own read-only DB tooling). This loop is **infra-touching**: it reads live database structure and data via read-only DB/MCP paths and read-only SQL, and it produces migration/remediation plans as artifacts. **Any schema change, migration application, or grant modification is a human-approval action ([PROTOCOL] §10) — the loop halts and hands the migration back for the operator to apply, and it NEVER applies to the wrong migration tree.**

# Trigger (when the operator runs this)

- A recurring cadence you define (e.g. a monthly deep sweep), or
- Before any migration ships, after an N+1 escalation from LOOP-01, or after any RLS/grant change, or
- On operator command: `Run LOOP-09 on <target>` — e.g. `Run LOOP-09 on <your-repo> (project: <supabase-project-ref>)`.

# Inputs

- **Target:** a Supabase project (or any relational database your platform runs on). If you operate multiple projects/repos, give each its own DB connection / MCP binding.
- **Scope flags:** `--rls-only`, `--migrations-only`, `--no-live` (repo migration files only, no live DB reads), `--tables=<list>`.
- **Migration-tree facts (load-bearing):** confirm which migration directory is LIVE before touching anything (commonly `supabase/migrations/` for Supabase projects). Many repos accumulate a second, non-canonical migration tree (e.g. a stale `legacy/` folder left over from a prior framework or an abandoned migration attempt) that is NOT applied and must never be edited or targeted. Two migration trees is a known hazard; every migration finding must name which tree it belongs to.
- **RLS / privilege-escalation history:** if your project has ever had a security review or incident around row-level-security bypass, treat that class of finding as CRITICAL and actively re-test it every run — don't just re-cite the old evidence. A common real-world pattern: an `anon`/unauthenticated role able to `UPDATE` a privilege-bearing column directly — e.g. an `is_admin`/`is_staff` flag, or a `plan`/`tier` billing column — self-escalation. Treat this class as CRITICAL, not just theoretical.

# Preconditions

1. [PROTOCOL] loaded; checkpoint dir `_loopstate/LOOP-09/<run-id>/` created.
2. DB MCP (or equivalent read-only DB access) reachable for the target project. Note: on machines where a local AV/TLS-interception proxy intercepts certs, MCP/DB calls may need `NODE_OPTIONS=--use-system-ca` to succeed. All MCP/SQL in this loop uses read-only paths (`list_tables`, `list_migrations`, `list_extensions`, `get_advisors`, `get_logs`, and `execute_sql` restricted to SELECT/EXPLAIN).
3. Review architecture docs/ADRs for any server code whose query patterns this loop analyzes — use a code-graph/knowledge-graph tool if your project has one (MCP server, IDE semantic index), otherwise trace with ast-grep/Semgrep/LSP ([PROTOCOL] §2).
4. Clean repo worktree from origin/main (or your trunk branch). Proposed migrations are written as files under the LIVE tree path on a `loop/LOOP-09-<date>` branch — staged, never applied.
5. **Never write into your project's own internal docs/notes vault, if it has one** — treat it as read-only reference material, not loop output space.

# Execution DAG

1. **N1 — Schema & migration inventory** (anchor: which tree, which applied, drift live-vs-files)
2. **N2 — Migration safety & reversibility** — depends on N1
3. **N3 [P] — Foreign keys, cascades & orphan-row detection**
4. **N4 [P] — Index coverage vs query patterns** (N+1 escalations from LOOP-01 land here)
5. **N5 [P] — Backup verification (restore test, not existence)**
6. **N6 [P] — Partitioning & replication posture**
7. **N7 — RLS / grant audit per role** (anon/authenticated/PUBLIC self-escalation sweep) — CRITICAL-priority node
8. **N8 [P] — Transaction isolation & deadlock analysis**
9. **N9 [P] — Unbounded queries & DB-level pagination**

# Node Specs

**N1 — Schema & migration inventory**
- *Action:* Establish ground truth. `list_migrations` (live applied set) vs the files in the LIVE `supabase/migrations/` tree — surface any applied-but-missing-file or file-but-unapplied drift. Explicitly confirm any LEGACY/secondary migration tree (e.g. a stale `legacy/` directory) is not being read as authoritative, and flag any tooling/config that still points at it. `list_tables` + `list_extensions` to snapshot the schema. Every downstream finding cites the tree by name.
- *Tools:* DB MCP (`list_migrations`, `list_tables`, `list_extensions`), Read (migration files), Glob (locate both trees), a code-graph/knowledge-graph tool if available (otherwise ast-grep/Semgrep/LSP).
- *Failure conditions:* migration applied to live with no file in the LIVE tree = HIGH (provenance gap); any config referencing the LEGACY tree as a write target = HIGH.
- *Output:* `node-1.json`: applied migrations | file migrations | drift | tree | schema snapshot ref.

**N2 — Migration safety & reversibility**
- *Action:* For each migration file (LIVE tree), statically assess safety: (a) reversibility — is there a down/rollback path, or is it explicitly flagged irreversible with justification? (b) locking risk — `ALTER TABLE` rewrites, index creation without `CONCURRENTLY`, `NOT NULL` adds without defaults on large tables (blocking-lock hazards); (c) destructive ops — `DROP COLUMN`/`DROP TABLE`/`TRUNCATE` without a data-preservation step; (d) ordering hazards vs applied state from N1. Produce a per-migration reversibility verdict. Any *new* migration this loop proposes is written reversible-by-default under the LIVE tree, staged only.
- *Tools:* Read, DB MCP (`list_migrations`, EXPLAIN via read-only `execute_sql` on a shadow/branch if available), a code-graph/knowledge-graph tool if available (otherwise ast-grep/Semgrep/LSP).
- *Failure conditions:* irreversible migration with no explicit flag = HIGH; unqualified `DROP`/`TRUNCATE` on a table with rows = **CRITICAL** (data-loss risk) → require human approval before it could ever run.
- *Output:* `node-2.json`: migration | reversible? | lock risk | destructive? | verdict.

**N3 [P] — Foreign keys, cascades & orphan-row detection**
- *Action:* Map declared foreign keys across all tables (from schema snapshot). For each parent→child relationship: is the FK actually enforced (constraint present, not just a naming convention)? Is `ON DELETE` behavior intentional — `CASCADE` where cascade is desired, `RESTRICT`/`SET NULL` elsewhere? Run read-only orphan-detection SELECTs (child rows whose FK target no longer exists) on relationships lacking enforced constraints. Flag "implicit FKs" (columns named `*_id` with no constraint) as integrity risks.
- *Tools:* DB MCP (`execute_sql` — SELECT-only orphan counts, e.g. `LEFT JOIN ... WHERE parent.id IS NULL`), Read, `list_tables`.
- *Failure conditions:* orphan rows found on an implicit FK = HIGH; `ON DELETE CASCADE` on a chain that could mass-delete user data unexpectedly = HIGH (flag for explicit operator review).
- *Output:* `node-3.json`: relationship | FK enforced? | on-delete | orphan count | verdict.

**N4 [P] — Index coverage vs query patterns**
- *Action:* This is where **N+1 escalations from LOOP-01 land.** Take the hot query patterns (from LOOP-01 findings if present, else derive from server code via a code-graph/knowledge-graph tool or static tracing) and check each against actual indexes. Use `get_advisors` (Supabase's own index/performance advisories, or your DB's equivalent) plus `EXPLAIN` (read-only) on representative queries to find sequential scans on large tables, missing composite indexes for common WHERE+ORDER BY combos, and redundant/unused indexes (write-amplification cost). Propose `CREATE INDEX CONCURRENTLY` migrations, staged.
- *Tools:* DB MCP (`get_advisors`, `execute_sql` EXPLAIN-only), a code-graph/knowledge-graph tool if available (otherwise ast-grep/Semgrep/LSP) to trace query sites, Read.
- *Failure conditions:* seq scan on a table > ~100k rows in a hot path = HIGH; advisor-reported missing index on a foreign key = MEDIUM.
- *Output:* `node-4.json`: query pattern | index used | scan type | advisor note | proposed index.

**N5 [P] — Backup verification (restore test, not existence)**
- *Action:* Existence of a backup is NOT the deliverable — a proven restore is. Verify Supabase PITR/backup configuration is enabled and current (`get_project`/project settings via MCP, or your provider's equivalent), then verify restorability *without touching production*: prefer a Supabase branch (`create_branch` is a human-approval action — request it) or a scratch project restore, and confirm the restored copy has expected table counts and a recent row. If no non-prod restore target can be created without approval, this node emits a HIGH finding and a ready-to-run restore-test plan for the operator rather than asserting backups work.
- *Tools:* DB MCP (`get_project`, `list_branches`; branch creation requires approval — request, do not self-execute), Bash/SSH read-only for any file-level backups on your infrastructure (hand cross-check to LOOP-07 N9), Read.
- *Failure conditions:* backups configured but never restore-tested = HIGH; PITR disabled on a production project = **CRITICAL**.
- *Output:* `node-5.json`: backup mechanism | last backup | restore test result | RTO/RPO | verdict.

**N6 [P] — Partitioning & replication posture**
- *Action:* Identify tables that should be partitioned (large append-only: logs, usage/accounting rows, telemetry) and whether they are. Assess replication/read-replica posture for read-heavy tables and the failover story — verify no data-layer component still assumes a node or environment that has since been decommissioned or retired; failover expectations must route to your current infrastructure topology. This is posture assessment + recommendation, not live reconfiguration.
- *Tools:* DB MCP (`list_tables` with row estimates, `get_advisors`), a code-graph/knowledge-graph tool if available (otherwise ast-grep/Semgrep/LSP), Read.
- *Failure conditions:* unpartitioned append-only table growing unbounded = MEDIUM; any data-layer failover config referencing a decommissioned/retired node = HIGH (hand topology drift to LOOP-07 N1).
- *Output:* `node-6.json`: table | size/growth | partitioned? | replication | failover target.

**N7 — RLS / grant audit per role (CRITICAL-priority node)**
- *Action:* The self-escalation re-test. For every table, sweep effective privileges for **anon**, **authenticated**, and **PUBLIC** roles using `has_column_privilege`/`has_table_privilege` across SELECT/INSERT/UPDATE/DELETE — column-granular. **Actively re-test any known finding from prior security reviews:** confirm an anon/unauthenticated role cannot `UPDATE` an admin/staff flag, a billing/plan-tier column, or any other privilege-bearing column. Verify RLS is ENABLED (not just policies written) on every table holding user or tenant data, and that policies actually filter by `auth.uid()`/tenant, closing cross-tenant reads. Treat any anon/authenticated write to a privileged column as **CRITICAL**. Cross-reference `get_advisors` security lints.
- *Tools:* DB MCP (`execute_sql` — `has_column_privilege(...)` sweeps and RLS-enabled checks, all read-only; `get_advisors` security category), a code-graph/knowledge-graph tool if available (trace which policies protect which routes — otherwise ast-grep/Semgrep/LSP), Read.
- *Failure conditions:* anon/authenticated writable privileged column = **CRITICAL** → HALT LOOP, emit artifact, human approval + REVOKE migration staged (grant change = human approval, [PROTOCOL] §10); RLS disabled on a user-data table = CRITICAL; policy not filtering by identity = HIGH.
- *Output:* `node-7.json`: table.column | role | SELECT/INS/UPD/DEL | RLS enabled | policy filters identity? | verdict.

**N8 [P] — Transaction isolation & deadlock analysis**
- *Action:* Review multi-statement operations in server code (traced via a code-graph/knowledge-graph tool if available, otherwise ast-grep/Semgrep/LSP): are they wrapped in transactions where atomicity matters (e.g. billing/usage-metering updates — watch for the classic bug where a metering write still fires even though the underlying operation aborted partway through)? Assess isolation levels for read-modify-write races — a common pattern is an unlocked check-then-spend sequence on a quota or balance (an "overspend race") — flag places needing `SELECT ... FOR UPDATE` or serializable isolation. Review `get_logs` for deadlock/lock-timeout signatures. Recommend fixes; do not alter live transactions.
- *Tools:* a code-graph/knowledge-graph tool if available (otherwise ast-grep/Semgrep/LSP) to trace transaction boundaries, DB MCP (`get_logs` for deadlock signatures), Read.
- *Failure conditions:* money/quota read-modify-write without proper isolation = HIGH (financial correctness); recurring deadlock signature in logs = MEDIUM.
- *Output:* `node-8.json`: operation | txn wrapped? | isolation | race risk | log evidence.

**N9 [P] — Unbounded queries & DB-level pagination**
- *Action:* At the DB boundary, find queries that can return unbounded result sets (no `LIMIT`, no keyset/offset pagination) — the DB-side complement to LOOP-01's endpoint pagination check. Trace high-cardinality-table reads from server code; flag `SELECT *` on wide tables and any endpoint that materializes a whole table into memory. Recommend keyset pagination and sensible default LIMITs.
- *Tools:* a code-graph/knowledge-graph tool if available (otherwise ast-grep/Semgrep/LSP) to find query sites, DB MCP (`execute_sql` EXPLAIN to confirm scan breadth), Read.
- *Failure conditions:* unbounded read on a table > ~10k rows reachable from a request path = HIGH.
- *Output:* `node-9.json`: query site | table | bounded? | pagination style | proposed fix.

# Adversarial Check

- *Persona:* **an attacker holding only the anon (publishable) key**, probing for one writable privileged column, one table with RLS off, one cross-tenant read, one cascade that mass-deletes, and one backup that has never actually been restored. Secondary persona: a careless migration author about to `DROP COLUMN` on prod during peak load.
- *Protocol:* FREE-MAD ([PROTOCOL] §11). Reviewer attacks the confidence block's `unknown` lists first — typically: tables not swept in N7, relationships assumed-enforced but unverified, and the restore test that was "planned" but not run.
- *Specific attacks:* re-run the privilege-column self-escalation probe (admin/staff flags, billing/tier columns) independently and demand the exact `has_column_privilege` output; demand a restored-copy row count for any "backups OK" claim; attempt to name one table where RLS policies exist but RLS is not actually ENABLED (policies-without-enable is a classic bypass).
- *Silent Agreement rule:* unanimous round-1 pass on N7 with no cited privilege-check output → forced hostile round 2 (this node's confidence must be earned, not assumed).

# Exit Criteria (overrides [PROTOCOL] §12 values)

- **Zero anon/authenticated-writable privileged columns** (any admin/staff flag, billing/plan-tier column, or other role/permission column) — re-tested and proven, not assumed.
- **RLS ENABLED on 100% of user/tenant-data tables**, each with an identity-filtering policy, or a CRITICAL finding exists.
- **100% of LIVE-tree migrations reversible or explicitly flagged irreversible** with justification; zero unqualified destructive ops staged.
- **Backup restore test PASSES** (restored copy verified with counts + recent row) OR a HIGH finding + ready-to-run restore plan is handed back.
- Zero orphan rows on enforced FKs; every implicit FK either constrained (staged migration) or flagged.
- Zero migrations targeting or referencing the LEGACY migration tree.
- Zero unbounded request-reachable reads on large tables, or a HIGH finding each.
- Final confidence ≥ 0.85 (N7 sub-confidence must independently be ≥ 0.85); governance row written; retries ≤ budget. FAIL-with-artifact otherwise.

# Failure Routing (deviations from [PROTOCOL] §4)

- **Wrong-tree temptation:** proposing any edit to the LEGACY migration tree is a hard stop (Architecture Failure) — re-scope to the LIVE tree and log it.
- **Live write attempt:** any MCP call outside the read-only allowlist (`apply_migration`, non-SELECT `execute_sql`, `create_branch`, `reset_branch`, etc.) is a Permission Failure → stop, surface to operator. This loop reads; the operator applies.
- **CRITICAL RLS finding:** immediate HALT + CRITICAL artifact; stage the REVOKE/RLS-enable migration but do NOT apply.

# Approval Gates (deviations from [PROTOCOL] §10)

- All read-only MCP/SQL and repo-side migration authoring: **automatic** (writes to `_loopstate` and staged migration files on the loop branch).
- Any `apply_migration`, live schema/grant change, Supabase branch creation, or backup restore against a real target: **human explicit approval — the loop halts and hands the migration/plan back.**
- Production data delete or destructive migration: **multi-step human approval** — loop MUST halt.

# RUN PROMPT

```
Run LOOP-09 (Data Architecture & Persistence audit).

Read skills/LOOP-09-data-layer.md and PROTOCOL.md (the loop protocol), then execute
the LOOP-09 Execution DAG under protocol rules: checkpoints per node, a confidence
block on every output, FREE-MAD adversarial check, governance row at the end.

Target project: <your Supabase project ref, or other DB connection>
Scope flags: <optional>

Posture: READ-ONLY on the live DB (DB MCP read paths + SELECT/EXPLAIN only).
LIVE migration tree is supabase/migrations/ — NEVER touch any legacy/secondary
migration tree still lingering in the repo.
Actively re-test anon/unauthenticated self-escalation on every admin/staff-flag
and billing/tier column. Prove backups by restore test, not existence. Stage any
migration/REVOKE on loop/LOOP-09-<date>; do not apply.
Produce _loopstate/LOOP-09/<run-id>/AUDIT-ARTIFACT.md for my sign-off.

Hard rail: no live infra/schema mutation without my explicit approval.
```
