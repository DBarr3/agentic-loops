---
name: loop-infra-devops
loop-id: LOOP-07
description: Docker/K8s/Terraform/secrets/IAM/SBOM/supply-chain audit across your deployment mesh
domain: Infrastructure, Cloud & DevOps
risk-class: infra-touching
default-debate: FREE-MAD
model-tiers: {scan: cheap, audit: mid, verdict: reasoning}
---

# Mission

Certify the infrastructure layer of your platform — container images, compose/systemd service units, IaC state, secrets posture, service-account privilege, resource limits, ingress path integrity, supply chain, and backups — across your live fleet of nodes. This loop is **infra-touching**: it operates READ+PLAN by default. It inspects live nodes over SSH (read-only, via your documented jump hosts), diffs reality against your documented topology, and produces a remediation plan as an artifact. **Any live infra mutation (unit edits, restarts, firewall changes, secret rotation) requires an explicit human approval gate per PROTOCOL.md §10 — the loop halts and hands back.**

# Trigger (when the operator runs this)

- Monthly deep sweep (or your team's own cadence), or
- After any node provisioning change, service migration between nodes, or new systemd unit ships, or
- On operator command: `Run LOOP-07 on <repo|node|mesh>` — e.g. `Run LOOP-07 on mesh (scope: node-A,node-B)`.

# Inputs

- **Target:** one or more of {repo infra dirs, individual node, full mesh}. Any node you've formally decommissioned is a *negative target*: the loop verifies nothing still references it.
- **Scope flags:** `--images-only`, `--secrets-only`, `--no-ssh` (repo-side static audit only), `--node=<name>`.
- **Reference topology:** your own documented infra map (whatever you use — an architecture doc, a runbook, an internal wiki page) naming each node's role and the network path between them (e.g. edge/CDN → VPN overlay → internal mesh → service).

# Preconditions

1. [PROTOCOL] loaded; checkpoint dir `_loopstate/LOOP-07/<run-id>/` created.
2. SSH reachability confirmed read-only to every in-scope node, via your documented jump-host paths. If a node is unreachable → Tool Failure route (PROTOCOL.md §4), audit continues on remaining nodes.
3. A code-graph/knowledge-graph tool if your project has one (MCP server, IDE semantic index) — otherwise ast-grep/Semgrep/LSP — for repo-side comprehension; read any existing architecture docs/ADRs for an infra file before analyzing it.
4. Confirm the target repo worktree is clean and judged from origin/main (or your trunk branch). No loop branch is created unless a repo-side remediation is planned (branch `loop/LOOP-07-<date>`, never live edits).

# Execution DAG

1. **N1 — Topology inventory & drift check** (anchor node; everything else keys off its service map)
2. **N2 [P] — Docker image audit** (pinned digests, non-root, minimal base)
3. **N3 [P] — Compose & systemd unit audit** (resource limits, restart policy, hardening directives)
4. **N4 [P] — IaC drift audit** (Terraform/other IaC if present; else document its absence as a finding)
5. **N5 — Secrets handling audit** (depends on N2+N3 file inventory)
6. **N6 [P] — IAM & least-privilege audit** (service accounts, tokens, SSH keys, DB roles used by services)
7. **N7 — Ingress & network policy audit** (edge→overlay→mesh path integrity; depends on N1)
8. **N8 [P] — Supply chain & SBOM audit**
9. **N9 — Backup existence audit** (existence only; restore *testing* for the data layer belongs to your data-layer audit process — flagged here for everything else)

# Node Specs

**N1 — Topology inventory & drift check**
- *Action:* Build the ground-truth service map: for each reachable node run read-only `systemctl list-units --type=service --state=running`, `docker ps --format`, `ss -tlnp` (via Bash/ssh through your documented jump hosts). Diff against your documented topology. Explicitly grep repo configs, unit files, and DNS/overlay-network references for any node you've formally decommissioned — it must have zero inbound references (e.g., confirm any traffic once routed to it was cleanly rerouted, not left dangling).
- *Tools:* Bash (ssh read-only), Read, a code-graph/knowledge-graph tool if available (query for services referencing hostnames/IPs), Grep for literal host strings only.
- *Failure conditions:* any service running on an undocumented node; any live reference to a decommissioned node; a previously-disabled daemon found re-enabled (must stay disabled per your own standing policy) → severity HIGH, Security Failure route if it faces the public internet.
- *Output:* `node-1.json` + `topology-drift.md` table: service | documented node | actual node | drift | severity.

**N2 [P] — Docker image audit**
- *Action:* Enumerate every image in Dockerfiles, compose files, and `docker ps` output. Verify: (a) base images pinned by digest (`@sha256:`) not floating tags (`:latest` = automatic finding); (b) containers run as non-root (`USER` directive or compose `user:`); (c) minimal base (alpine/distroless preferred; full OS images flagged with justification required); (d) no build-time secrets in layers (`docker history` scan read-only).
- *Tools:* Read (Dockerfiles/compose in repo), Bash/ssh read-only (`docker inspect`, `docker history --no-trunc`).
- *Failure conditions:* unpinned tag on an internet-facing service = HIGH; root container with host mounts = HIGH; secret in image layer = CRITICAL → HALT per PROTOCOL.md §4 Security Failure.
- *Output:* `node-2.json` image matrix: image | pinned | user | base class | findings.

**N3 [P] — Compose & systemd unit audit**
- *Action:* Read every unit file (`/etc/systemd/system/*.service`, read-only cat over ssh) and compose file for your services. Verify resource limits exist — systemd `MemoryMax` (or your platform's equivalent) on any long-running service on a swapless host; flag any long-running service with no memory ceiling on such a host. Check `Restart=`, `WantedBy`, sandboxing directives (`ProtectSystem`, `NoNewPrivileges`, `PrivateTmp`), and port-vs-unit drift (a unit file hardcoding a port the service no longer actually listens on is a classic silent-breakage pattern — verify each unit's configured port against what the service really binds).
- *Tools:* Bash/ssh read-only, Read, Grep (literals: ports, paths).
- *Failure conditions:* no memory ceiling on swapless node = HIGH; unit runs as root without justification = MEDIUM; unit/port drift vs documented = MEDIUM.
- *Output:* `node-3.json` unit matrix + proposed hardened unit diffs (PLAN only, not applied).

**N4 [P] — IaC drift audit**
- *Action:* Locate Terraform/Pulumi/CloudFormation state in the repo (a code-graph/knowledge-graph tool if available, plus Glob). If present: `terraform plan -refresh-only` in read-only mode to surface drift between state and the live mesh. If absent (common for hand-provisioned infra): record "no IaC" as a standing MEDIUM finding with a recommendation to codify the N1 service map as the seed inventory.
- *Tools:* Glob, Read, Bash (terraform read-only ops only; never `apply`), a code-graph/knowledge-graph tool if available.
- *Failure conditions:* `terraform apply` is forbidden in this loop — attempting it is a Permission Failure (stop, surface to operator). Detected drift on security-relevant resources = HIGH.
- *Output:* `node-4.json` drift list or documented-absence finding.

**N5 — Secrets handling audit**
- *Action:* Sweep the N2/N3 file inventory plus repo history for plaintext secrets: env files committed to git, `Environment=` lines in units carrying API keys, tokens in compose files, credentials in URLs. Apply standard credential-shape detection (common provider key shapes, JWTs, high-entropy strings) — this is the same class of check a secrets-scanning pre-commit guard (gitleaks, trufflehog, or equivalent) should enforce on every future commit. Verify live services load secrets from protected paths (correct file permissions, e.g. `600`) rather than world-readable locations. **Never print a discovered secret value into any artifact — record path + shape + hash prefix only.**
- *Tools:* Read, Grep (credential shapes — literal matching is the sanctioned use), Bash/ssh read-only (`stat -c '%a %U'` on secret files).
- *Failure conditions:* any plaintext secret in a committed file = CRITICAL → HALT LOOP, emit artifact, human approval required to continue (rotation is a multi-step human approval action, PROTOCOL.md §10).
- *Output:* `node-5.json`: location | secret shape | exposure class | rotation-required flag.

**N6 [P] — IAM & least-privilege audit**
- *Action:* Enumerate identities services actually use: SSH authorized_keys per node, service-role vs anon/public key usage in server code (trace key imports with a code-graph tool if available), CI deploy tokens, CDN/DNS API token scopes, VPN/overlay-network ACLs. Verify each service holds the minimum privilege it needs — e.g. a read-only dashboard should hold a read-only DB role; confirm it hasn't quietly gained write grants over time. Cross-reference your data-layer audit's role-grant sweep if you have one; deep DB-grant work belongs there, not here.
- *Tools:* Bash/ssh read-only, Read, a code-graph/knowledge-graph tool if available, DB MCP read paths (`list_tables`, advisors) if applicable — no SQL writes.
- *Failure conditions:* a service holding elevated/service-role credentials where a scoped role would suffice = HIGH; shared SSH key across nodes = HIGH; token with wildcard scope = MEDIUM.
- *Output:* `node-6.json` identity matrix: identity | used by | scope held | scope needed | delta.

**N7 — Ingress & network policy audit**
- *Action:* Verify your documented path (e.g. **edge/CDN → VPN overlay → internal mesh → service**) holds for every service from N1. From outside-in: enumerate publicly-proxied hostnames, then check each node's listening sockets (`ss -tlnp`) against firewall rules (`iptables -L -n` / `nft list ruleset`, read-only) — **anything bound to a public interface that your topology says should be mesh-only is a HIGH finding minimum**. Confirm any internal monitoring/security tooling can actually see the traffic its role depends on, and that any prior network-hardening work (VPN re-keying, jump-host-only SSH, etc.) is still in effect.
- *Tools:* Bash (ssh read-only; external `curl -sI`/DNS checks from the workstation for the public surface), Read.
- *Failure conditions:* mesh-only service reachable from public internet = CRITICAL → HALT (Security Failure); VPN/overlay peer missing for a node that services depend on = HIGH.
- *Output:* `node-7.json`: hostname/port | intended exposure | actual exposure | verdict.

**N8 [P] — Supply chain & SBOM audit**
- *Action:* Generate/refresh an SBOM per deployable (syft or `docker sbom` if available; fallback: lockfile inventory via Read). Check: base image provenance, dependencies of services on your production nodes against known-vulnerable versions (`pip list --outdated`, `npm audit --omit=dev` read-only), install scripts fetched over plain HTTP or `curl | bash` patterns in provisioning docs, and unpinned CI Actions in any deploy path (overlap with your CI/CD loop — record and hand off, do not duplicate).
- *Tools:* Bash (read-only scanners), Read, Glob, a code-graph/knowledge-graph tool if available.
- *Failure conditions:* known-critical CVE in an internet-facing service dependency = CRITICAL; missing SBOM = MEDIUM (generate one as an artifact, which is an automatic-approval write).
- *Output:* `node-8.json` + `sbom/<service>.json` per deployable.

**N9 — Backup existence audit**
- *Action:* For each stateful thing found in N1 (database, object/file storage, any application-specific persistent data, release/download artifacts, unit files themselves): verify a backup mechanism *exists* — cron/systemd timers, snapshot config, replication target — and record last-run timestamp. This node proves existence and freshness only; **restore verification for the data layer belongs to your data-layer audit process**, and any non-DB restore test is emitted as a recommended follow-up, not executed here.
- *Tools:* Bash/ssh read-only (`systemctl list-timers`, `crontab -l`, `ls -la` on backup dirs), DB MCP read paths if applicable, Read.
- *Failure conditions:* stateful service with no backup mechanism = HIGH; backup older than 2× its declared cadence = HIGH.
- *Output:* `node-9.json`: asset | backup mechanism | last success | age | verdict.

# Adversarial Check

- *Persona:* **a hostile SRE who assumes every backup is broken, every "mesh-only" port is secretly public, and every documented topology line is six months stale.** Secondary persona on N2/N8: a supply-chain attacker who already controls one upstream image tag.
- *Protocol:* FREE-MAD (PROTOCOL.md §11) — reviewers attack independently, score-based verdict, no forced consensus. The reviewer receives every node's confidence block and attacks the `unknown` lists first (typically: nodes skipped via `--no-ssh`, sockets not probed from outside, backup jobs whose logs weren't read).
- *Specific attacks:* demand external proof for every "mesh-only" claim (an actual failed connection attempt from a public vantage, not just firewall-rule reading); demand a timestamp for every backup claim; re-derive the N1 service map from `ss` output alone and diff against the loop's own map.
- *Silent Agreement rule:* if all reviewers pass round 1 with no cited command output, force a hostile second round.

# Exit Criteria (overrides PROTOCOL.md §12 values)

- **Zero plaintext secrets** in committed files or world-readable live paths (CRITICAL class must be empty or operator-deferred with rotation ticket).
- **Zero mesh-only services reachable from the public internet** (verified by external probe, not config reading).
- **Zero live references to any decommissioned node** (configs, units, DNS, overlay-network routes).
- 100% of running containers accounted for: pinned-digest + non-root, or explicitly flagged with a finding ID.
- 100% of long-running services on swapless nodes carry a memory ceiling, or a HIGH finding exists per exception.
- Every stateful asset has a backup mechanism with a last-success timestamp ≤ 2× cadence, or a HIGH finding exists.
- Topology drift table complete for every reachable node; unreachable nodes explicitly listed under confidence-block `unknown`.
- Final confidence ≥ 0.85; governance row written; all retries ≤ budget.
- FAIL-with-artifact if any criterion cannot be met — never silent success.

# Failure Routing (deviations from PROTOCOL.md §4)

- **Unreachable node:** Tool Failure, but do NOT suspend the loop — mark the node's audits `unknown` in the confidence block and continue; the exit criteria then force FAIL-with-artifact or operator deferral.
- **Live-mutation temptation:** any node concluding "just restart the unit / patch the firewall now" is a Permission Failure by definition — plan it, never do it.
- **Discovered active intrusion indicators:** immediate HALT, emit CRITICAL artifact, page the operator; do not touch the affected node further (preserve forensics for whatever incident-response process you run).

# Approval Gates (deviations from PROTOCOL.md §10)

- All SSH activity in this loop is read-only: **automatic**.
- Writing proposed unit-file/Dockerfile diffs into `_loopstate` or onto a repo loop branch: **automatic / reviewer sign-off** respectively.
- Applying ANY change to a live node (unit edit, `systemctl` state change, firewall rule, secret rotation, package upgrade): **multi-step human approval — the loop halts and hands the plan back. No exceptions, including "trivial" restarts.**

# RUN PROMPT

```
Run LOOP-07 (Infrastructure, Cloud & DevOps audit).

Read skills/LOOP-07-infra-devops.md and PROTOCOL.md (this collection's shared
protocol), then execute the LOOP-07 Execution DAG end-to-end under protocol
rules: checkpoints per node, a confidence block on every output, FREE-MAD
adversarial check, governance row at the end.

Target: <mesh | node list | repo infra dirs>   Scope flags: <optional>

Posture: READ + PLAN only. SSH via your documented jump hosts, read-only
commands only. Produce _loopstate/LOOP-07/<run-id>/AUDIT-ARTIFACT.md with the
topology-drift table, findings, and a remediation plan for my sign-off.

Hard rail: no live infra/schema mutation without my explicit approval.
```
