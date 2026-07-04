---
name: loop-cicd-release
loop-id: LOOP-08
description: Actions/pipelines audit — caching, test gating, artifact signing, rollback automation, canary/blue-green, feature flags
domain: CI/CD & Release Engineering
risk-class: branch-mutating
default-debate: FREE-MAD
model-tiers: {scan: cheap, audit: mid, verdict: reasoning}
---

# Mission

Audit and harden the release pipeline end-to-end: pipeline inventory, whether CI is *actually running* (not silently billing/quota-blocked), cache and parallelism efficiency, test gating before deploy, human approval points, artifact signing (verify whatever signing pattern you already use — Ed25519, GPG, Sigstore/cosign — and generalize it to every deployable), rollback automation, progressive-deployment readiness, feature-flag hygiene, and secret exposure in workflow logs. This loop is **branch-mutating for workflow files only**: fixes to `.github/workflows/**` and pipeline config land on a `loop/LOOP-08-<date>` branch as a PR. It never triggers deploys, never touches live infrastructure, and never merges — the operator does (PROTOCOL.md §10).

# Trigger (when the operator runs this)

- Pre-release sweep (commonly run alongside your test/mutation loop and UX loop before a release), or
- After any workflow file changes, a failed/suspicious ship, or a CI billing/quota event, or
- On operator command: `Run LOOP-08 on <repo>`.

# Inputs

- **Target:** your repo (judge from origin/main, or your trunk branch).
- **Scope flags:** `--workflows-only` (static audit, no gh API calls), `--include-desktop-ship` (audit a desktop release path if you have one: build → sign → publish artifact+manifest+signature to your distribution origin), `--since=<ISO date>` for run-history windows.
- **Known-risk context:** CI billing/quota blocks are a real failure mode — some accounts have had PR merges proceed with CI silently not executing. "Green" in the merge UI is therefore NOT proof that checks ran.

# Preconditions

1. [PROTOCOL] loaded; checkpoint dir `_loopstate/LOOP-08/<run-id>/` created.
2. `gh` CLI authenticated; repo resolvable (note: `gh` may need an explicit `-R <org>/<repo>` flag when run from a symlinked or non-standard worktree).
3. Clean worktree from origin/main; loop branch `loop/LOOP-08-<date>` created only when N-node output requires a workflow edit.
4. Read any existing architecture docs/ADRs for a workflow-adjacent script before proposing edits — a code-graph/knowledge-graph tool if available.
5. A secrets-scanning guard (gitleaks, trufflehog, or equivalent) runs before ANY commit on the loop branch.

# Execution DAG

1. **N1 — Pipeline inventory** (anchor: enumerate every workflow, trigger, and deploy path)
2. **N2 — CI liveness verification** (is CI actually executing, or billing/quota-blocked/skipped?) — depends on N1
3. **N3 [P] — Build caching effectiveness**
4. **N4 [P] — Parallel job structure**
5. **N5 — Test gating audit** (no deploy without green) — depends on N1+N2
6. **N6 [P] — Deployment approvals audit**
7. **N7 — Artifact signing audit** (verify whatever signing pattern you use; generalize) — depends on N1
8. **N8 [P] — Rollback automation & progressive-deployment readiness** (canary / blue-green)
9. **N9 [P] — Feature-flag hygiene**
10. **N10 — Secret exposure in workflow logs** — depends on N1 run inventory

# Node Specs

**N1 — Pipeline inventory**
- *Action:* Enumerate `.github/workflows/*.yml` plus any out-of-band release scripts (some release paths live partly outside CI — e.g. a local build + sign + upload step run by hand or via a separate script). For each pipeline record: triggers, jobs, permissions block, secrets consumed, deploy targets, and whether it gates anything. Build the deploy-path map: code → build → test → sign → publish, per deployable (e.g. desktop app, web app, backend services — whatever your platform ships).
- *Tools:* Glob, Read, Bash (`gh workflow list`, `gh api repos/.../actions/workflows`), a code-graph/knowledge-graph tool if available for scripts referenced by workflows.
- *Failure conditions:* a deploy path with no workflow or script representation (undocumented manual deploy) = HIGH finding; workflow with `permissions: write-all` = HIGH.
- *Output:* `node-1.json` pipeline matrix: workflow | trigger | jobs | secrets | deploy target | gate role.

**N2 — CI liveness verification**
- *Action:* For each workflow from N1, pull the actual run history (`gh run list --workflow=<w> --limit 50 --json status,conclusion,createdAt`). Detect the three silent-death modes: (a) zero runs in a window where matching triggers fired (compare against `gh api .../commits` on trigger branches); (b) runs with conclusion `skipped`/`action_required`/`startup_failure` — the billing/quota-block signature; (c) required checks configured in branch protection that no live workflow produces. Cross-check billing/quota state via `gh api /repos/.../actions/permissions` and recent PRs merged with no check runs attached.
- *Tools:* Bash (`gh run list`, `gh api`), Read.
- *Failure conditions:* any protected branch merging with checks not executed = **CRITICAL** (the pipeline is theater); billing/quota-blocked workflows = HIGH with explicit operator escalation (only the operator can fix billing).
- *Output:* `node-2.json` liveness table: workflow | last real run (ISO) | trigger events since | executed? | death mode.

**N3 [P] — Build caching effectiveness**
- *Action:* Audit `actions/cache` / `setup-node` cache usage: are lockfile-keyed caches present for npm/pip? Measure hit rate from run logs (`gh run view --log` on recent runs, grep cache-hit lines). Flag rebuild-the-world patterns (no cache, or cache key on `github.sha`), and native-module rebuild cost (Electron/node native-addon rebuilds are a known expensive step — verify a documented avoidance/cache strategy is encoded in the workflow, not tribal knowledge).
- *Tools:* Bash (`gh run view --log`), Read, Grep (literal cache-key strings).
- *Failure conditions:* cache hit rate < 50% on a hot workflow = MEDIUM; no caching on a >5-minute install step = MEDIUM.
- *Output:* `node-3.json`: workflow | cache steps | key strategy | measured hit rate | est. minutes wasted/run.

**N4 [P] — Parallel job structure**
- *Action:* Analyze job graphs (`needs:` chains) for false serialization: independent jobs chained sequentially, matrix opportunities not taken (multi-platform builds in series), and monolithic jobs that bundle lint+test+build into one runner. Propose a reshaped DAG per workflow with estimated wall-clock savings. Also flag the inverse: unbounded matrix fan-out with no `max-parallel` on a billing-constrained account.
- *Tools:* Read, Bash (`gh run view` timing data).
- *Failure conditions:* critical-path wall clock > 2× the sum of its longest independent chain = MEDIUM; unbounded fan-out = MEDIUM (cost risk given billing history).
- *Output:* `node-4.json` + proposed workflow diffs (staged for loop branch).

**N5 — Test gating audit**
- *Action:* Verify the invariant **no deploy without green**: every deploy/publish job must have `needs:` on test jobs AND branch protection must mark those checks required. Trace each deploy path from N1 for bypass routes: `workflow_dispatch` deploys that skip tests, `continue-on-error: true` on test steps, `if: always()` on publish steps, and out-of-band ship scripts (if you have one) that have no test gate at all — for out-of-band paths, specify the minimal pre-ship check script as a proposed addition. Combined with N2: a required check that never executes is a bypass, not a gate.
- *Tools:* Read, Bash (`gh api repos/.../branches/<b>/protection`), a code-graph/knowledge-graph tool if available.
- *Failure conditions:* any deploy path reachable with failing or unexecuted tests = **CRITICAL**; `continue-on-error` on a gating test = HIGH.
- *Output:* `node-5.json` gate map: deploy path | gates present | bypass routes | verdict.

**N6 [P] — Deployment approvals audit**
- *Action:* Audit GitHub Environments and protection rules: do production-facing workflows target an environment with required reviewers? Map the repo's real approval surface against PROTOCOL.md §10 classes (merge = operator; production deploy = multi-step human). Flag deploys that run on bare `push` to main with no environment gate, and self-approval possibilities (author == sole required reviewer).
- *Tools:* Bash (`gh api repos/.../environments`), Read.
- *Failure conditions:* production deploy with zero human approval step = HIGH; approval satisfiable by the triggering actor alone = MEDIUM.
- *Output:* `node-6.json`: deploy target | environment | required reviewers | gaps.

**N7 — Artifact signing audit**
- *Action:* If you already have an artifact-signing pattern proven somewhere in your release process (e.g. Ed25519 manifest signing), verify it end-to-end: artifact + manifest + signature published together, and the consumer (updater/installer/package manager) actually *verifies* the signature before applying it — signing without client-side verification is decoration; trace the verify call in the consuming code (a code-graph/knowledge-graph tool if available). Then generalize: enumerate every other artifact class from N1 (web bundles, npm packages, container images, release zips) and score each as signed/unsigned; propose extending the manifest-sign pattern or adopting provenance attestation (SLSA/`gh attestation`) where it fits. Confirm signing keys are NOT in the repo (hand N5-class evidence to LOOP-07 N5 if found).
- *Tools:* Read, a code-graph/knowledge-graph tool if available (trace signature verification in the consuming code), Bash (`gh api` release assets; ssh read-only to your distribution origin to confirm signature files present for the latest version).
- *Failure conditions:* published manifest without matching signature = HIGH; updater applies unverified payloads = **CRITICAL**; signing key material in repo = CRITICAL → HALT (Security Failure).
- *Output:* `node-7.json`: artifact class | signed | verified-on-consume | key custody | gap.

**N8 [P] — Rollback automation & progressive-deployment readiness**
- *Action:* For each deploy target, answer: "how do we get back to N-1 in under 5 minutes, and is it scripted?" Inventory rollback mechanisms (previous-artifact retention on your distribution origin, git revert + redeploy for web services, versioned manifests for desktop/installer builds). Score progressive-deployment readiness: canary (percentage rollout via staged manifests/flags), blue-green (parallel service + load-balancer/DNS cutover). This node produces a readiness scorecard and scripts-as-proposals — it does NOT build live canary infra (that's a live-infra change, gated by your infra-audit loop/process).
- *Tools:* Read, Bash (ssh read-only artifact-retention checks), a code-graph/knowledge-graph tool if available.
- *Failure conditions:* any production target with no documented, tested rollback path = HIGH; rollback that requires rebuilding from source = MEDIUM.
- *Output:* `node-8.json` scorecard: target | rollback (scripted? tested? RTO) | canary-ready | blue-green-ready.

**N9 [P] — Feature-flag hygiene**
- *Action:* Inventory feature flags across your codebase (env-driven toggles, DB-driven flags such as a boolean feature/tier column). For each: owner, default state, kill-switch behavior, and age. Flag zombie flags (fully rolled out but never removed), flags whose off-path is untested, and **flags that gate security/billing behavior stored in user-mutable columns** — hand DB-side verification to your data-layer audit loop, but flag the CI/CD-side risk here: tests must run with such flags OFF.
- *Tools:* a code-graph/knowledge-graph tool if available (query for flag identifiers), Grep (literal flag names only), Read.
- *Failure conditions:* security-gating flag with an untested off-path = HIGH; flag older than 2 release cycles at 100% rollout = LOW (cleanup finding → LOOP-13).
- *Output:* `node-9.json` flag register: flag | location | default | gates what | age | verdict.

**N10 — Secret exposure in workflow logs**
- *Action:* Sweep recent run logs (`gh run view --log`, sampled across workflows) for credential shapes using standard credential-shape detection (common provider key shapes, JWTs, high-entropy strings, URLs with embedded credentials). Audit workflow YAML for the leak vectors: `echo`/`set -x` around secret env vars, secrets passed as CLI args (visible in process lines), secrets interpolated into build artifacts, and missing `::add-mask::` on derived values. Never copy a matched value into artifacts — path + shape + line ref only.
- *Tools:* Bash (`gh run view --log`), Read, Grep (credential shapes — sanctioned literal use).
- *Failure conditions:* live secret visible in any retained log = **CRITICAL** → HALT, rotation required (multi-step human approval, PROTOCOL.md §10); secret passed as CLI arg = HIGH.
- *Output:* `node-10.json`: workflow | run id | leak vector | shape | rotation-required.

# Adversarial Check

- *Persona:* **a release saboteur who wants to ship a malicious build through this exact pipeline without any human noticing** — they hunt for the unexecuted required check, the unsigned artifact path, the `workflow_dispatch` backdoor, and the log line that hands them a token. Secondary persona: a bored on-call engineer at 3am who must execute the rollback docs exactly as written.
- *Protocol:* FREE-MAD (PROTOCOL.md §11). Reviewers attack confidence-block `unknown` lists first — typically: runs not sampled in N10, out-of-band ship steps nobody traced, and N2 windows where trigger history was incomplete.
- *Specific attacks:* attempt to construct (on paper) a commit-to-production path that never executes a test; attempt to name one artifact a user consumes that is not signature-verified; replay the billing/quota-block scenario and check whether N2's detection would actually have fired.
- *Silent Agreement rule:* unanimous round-1 pass with no cited run IDs or file:line evidence → forced hostile round 2.

# Exit Criteria (overrides PROTOCOL.md §12 values)

- **CI liveness proven:** 100% of required checks have ≥1 real (non-skipped) execution in the audit window, or a CRITICAL finding exists naming the dead check.
- **Zero deploy paths reachable without green tests** (including out-of-band ship scripts, or an explicit operator-deferred finding per path).
- **Zero secrets in sampled workflow logs**; zero secrets passed as CLI args in workflow YAML.
- 100% of user-consumed artifact classes either signature-verified end-to-end or carrying an explicit HIGH finding.
- Every production target has a scripted rollback path, or a HIGH finding exists per target.
- Cache hit rate ≥ 50% on hot workflows or a staged fix exists on the loop branch.
- All workflow-file fixes staged on `loop/LOOP-08-<date>` with passing YAML lint; PR opened only after operator sign-off.
- Final confidence ≥ 0.85; governance row written; retries ≤ budget. FAIL-with-artifact otherwise.

# Failure Routing (deviations from PROTOCOL.md §4)

- **Billing/quota-blocked CI (N2):** classify as Dependency Failure but escalate immediately to the operator — no retry loop can fix billing; the finding blocks PASS.
- **`gh` rate limits:** Rate Limit route with backoff; if run-log sampling stays incomplete, record the unsampled set in the confidence block's `unknown` — the adversarial check will attack it.
- **Workflow edit breaks YAML/actionlint:** Syntax Failure, auto-fix on cheap tier, re-lint before checkpoint.

# Approval Gates (deviations from PROTOCOL.md §10)

- Workflow-file edits on the loop branch: **adversarial reviewer sign-off** (standard refactor class).
- Opening the PR: **operator approval** — findings artifact handed back first.
- Anything that would *trigger* a pipeline run with deploy side effects, modify branch protection, change GitHub environments/secrets, or touch your distribution origin: **human explicit approval — loop halts.** This loop audits the release system; it never operates it.

# RUN PROMPT

```
Run LOOP-08 (CI/CD & Release Engineering audit).

Read skills/LOOP-08-cicd-release.md and PROTOCOL.md (this collection's shared
protocol), then execute the LOOP-08 Execution DAG under protocol rules:
checkpoints per node, a confidence block on every output, FREE-MAD adversarial
check, governance row at the end.

Target repo: <repo>   Scope flags: <optional>

Posture: branch-mutating for workflow files ONLY, on loop/LOOP-08-<date>.
Verify CI is actually executing (billing/quota-block history) — do not trust
green UI. Do not trigger deploys, modify branch protection, or touch
environments/secrets.
Produce _loopstate/LOOP-08/<run-id>/AUDIT-ARTIFACT.md with the liveness table,
gate map, signing scorecard, and staged fixes for my sign-off.

Hard rail: no live infra/schema mutation without my explicit approval.
```
