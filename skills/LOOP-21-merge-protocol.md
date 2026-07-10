---
name: loop-merge-protocol
loop-id: LOOP-21
description: Mega-loop integration - anchor the real remote trunk, capture an incoming PR and its goals immutably, adversarially simplify and verify it, merge only from a fresh CI-proven state, replay the original contract over the entire post-merge trunk, and open a scoped follow-up PR for verified gaps
domain: Merge Orchestration & Post-Merge Assurance
risk-class: branch-mutating
default-debate: RA-CR
model-tiers: {scan: cheap, audit: mid, verdict: reasoning}
composed-of: [LOOP-08, LOOP-11, LOOP-15, LOOP-16, LOOP-17, LOOP-18, LOOP-19, LOOP-20]
---

# Mission

Turn an incoming pull request into a closed-loop merge, not a one-time green button.
The Merge Protocol first anchors itself to the repository's **real remote default branch**
and exact commit SHA; it never assumes the branch is named `main`, never treats a stale
local checkout as truth, and never invents missing pull-request state. It then captures the
PR's complete observable state and goal contract before anything mutates. That immutable
capture becomes the test oracle for both the pre-merge work and the post-merge replay.

The protocol composes the five meta-loops without collapsing their separate guarantees:

| Loop | Responsibility inside the Merge Protocol |
|------|------------------------------------------|
| LOOP-16 | Fan out only evidence-gathering or proposal work proven independent; never parallelize overlapping writers. |
| LOOP-17 | Run a bounded breaker/builder sparring session over the PR; the breaker never mutates, and proposed fixes flow through the ratchet before they land. |
| LOOP-18 | Apply accepted changes one coherent tooth at a time; test, adversarially verify, then weld or revert. |
| LOOP-19 | When a fix or simplification has multiple credible shapes, generate genuinely different candidates and converge on one provenance-tracked proposal. |
| LOOP-20 | Compile the captured goals into a cycle-checked task DAG and continuously re-verify it as the PR, trunk, and findings change. |

Three base loops provide hard gates: LOOP-11 supplies hostile review, LOOP-08 proves that
CI actually executed and gates the merge, and LOOP-15 converts failures of the *process*
into generalized improvements for the loop system. LOOP-15's existing hard boundary still
applies: **it never edits application code**. Application changes are executed by the PR's
builder path and welded by LOOP-18; procedural-memory improvements are routed to a separate
LOOP-15 proposal package.

The governing implementation goal is:

> Satisfy the captured PR goal with the smallest understandable working change. Simplify,
> label, and organize where evidence supports it; remove unnecessary code and dependencies;
> keep control flow obvious enough for a junior developer to trace; never trade correctness,
> safety, or an explicit requirement for a prettier diff.

After the operator-approved merge, the protocol checks out the exact resulting default-branch
SHA and replays the **entire initial capture** against the **entire integrated default branch**,
not merely the merged diff. Regressions, missing requirements, integration seams, cleanup,
and follow-ups become a structured gap register and, when a real diff exists and the operator
approves, a new draft follow-up PR. An empty follow-up PR is never created to manufacture motion.

# Trigger

- An open PR is believed ready for final hardening and merge.
- A high-impact PR needs a complete before/after contract and post-merge regression sweep.
- A previously merged PR has a saved Merge Protocol capture and needs the post-merge replay resumed.
- On demand: `Run LOOP-21 (Merge Protocol) on <repo> PR #<N>`.

Example input notation: `[PR #79]`. The number is an example only; node 2 must prove that
the referenced PR exists in the named repository before proceeding.

# Inputs (target repo/dir, scope flags)

- `repository`: canonical `owner/repo` or an unambiguous repository URL.
- `pr`: pull-request URL or number. Required unless resuming from a merge receipt.
- `goal_override`: optional operator clarification. It augments the PR's recorded goals but
  never silently replaces contradictory issue, review, or acceptance-criteria text.
- `merge_method`: `merge|squash|rebase`; default is the repository's allowed/preferred policy.
- `followup_mode`: `approval|off`; default `approval`. An actionable verified gap prepares
  a follow-up diff, but opening it still follows the approval gate below.
- `--capture-only`: run nodes 1-4 and stop with the immutable intake package.
- `--no-merge`: run through the pre-merge decision packet but do not execute node 10.
- `--resume-from <merge-receipt>`: begin at node 11 after validating the receipt and remote state.
- `--sparring-rounds <N>`: bounded LOOP-17 rounds before the ratchet, default 3, maximum 8.
- `--max-ratchet-teeth <N>`: maximum verified changes added to the incoming PR, default 8.
- `--followup-depth <N>`: maximum automatic parent/child lineage depth, default 1. A follow-up
  PR is never recursively merged by the same run.

# Preconditions

1. [PROTOCOL] read; `_loopstate/LOOP-21/<run-id>/` is writable and outside the application diff.
2. GitHub/repository API access can read the repository, PR, reviews, comments, checks, and
   branch metadata. Git access can materialize the exact SHAs returned by the API. A cached
   local branch alone is insufficient evidence.
3. The remote's default branch name and current commit SHA are resolvable. The protocol must
   record the actual name (`main`, `master`, or another configured trunk); it never guesses.
4. The incoming PR is open and its base repository matches `repository`. A closed/merged PR
   must use `--resume-from` or start a post-merge-only run.
5. The local worktree is clean before any mutation. PR changes occur only on the PR branch or
   an isolated child branch; the default branch is never edited directly.
6. The operator has write access to the PR head or explicitly authorizes an isolated maintainer
   branch. Without either, node 7 stops at a proposal packet and no mutation is attempted.
7. At least one reproducible validation harness exists for the PR's core behavior. If the
   repository has no tests, node 3 must define an executable acceptance probe before node 7.
8. The operator remains the merge authority. Passing checks, reviewer approval, or invoking
   this loop is not itself permission to merge.

# Non-Negotiable Rails

1. **Remote truth before local reasoning.** Resolve the remote default branch and SHA before
   reading the local checkout as the baseline.
2. **Capture before mutation.** Nodes 1-3 finish and their manifest is sealed before any code edit.
3. **SHA-pinned decisions.** Every diff, check, review, approval, and merge receipt names the
   exact base and head SHAs it evaluated.
4. **No guessed state.** Inaccessible PR fields become named `unknown` values. Any unknown that
   could change a merge verdict blocks the merge instead of being filled from memory.
5. **One writer path.** Parallel workers may inspect or propose. Accepted application mutations
   serialize through LOOP-18; LOOP-15 never touches application code.
6. **No stale merge.** Any material base, head, goal, review, or required-check drift resets the
   affected gates before merge.
7. **CI execution, not green decoration.** A required check must have a real run for the locked
   head SHA and a successful conclusion; missing, skipped, neutral, stale, or quota-blocked is not green.
8. **Post-merge replay is mandatory.** Merge success is an intermediate state, never loop success.
9. **No empty follow-up PR.** A follow-up opens only when a verified actionable diff exists.

# Execution DAG

```text
1 Remote truth anchor
  -> 2 Immutable full-PR capture
  -> 3 Goal contract + simplicity bar (LOOP-20 intake)
  -> 4 Cycle-checked composition plan (LOOP-20)
  -> 5 Independent evidence/proposal fan-out (LOOP-16 + conditional LOOP-19)
  -> 6 Hostile debate + bounded sparring (LOOP-11 + LOOP-17 + LOOP-15 learning track)
  -> 7 Verified one-change ratchet (LOOP-18)
  -> 8 Pre-merge CI/CD proof (LOOP-08)
  -> 9 Fresh-state / drift gate (LOOP-20 re-verify)
  -> 10 Operator approval + SHA-locked merge
  -> 11 Post-merge remote snapshot
  -> 12 Replay initial capture over the entire default branch
  -> 13 Gap classification + follow-up ratchet
  -> 14 Follow-up PR or evidence-backed NO-CHANGE close-out
```

Nodes 1-4 are strictly read-only and sequential. Node 5 may fan out only the units whose
read/write-set proof passes LOOP-16. Nodes 6-10 are sequential because each changes the merge
decision. Nodes 11-14 are a mandatory post-merge phase. A drift edge from node 9 routes to the
earliest invalidated node; it never skips forward with a patched timestamp.

# Node Specs

## Node 1 - Remote truth anchor

- **Action:** Resolve canonical repository metadata from the remote; read its configured default
  branch; resolve that branch to an exact commit SHA; fetch/materialize that object locally; and
  verify the local commit and tree hashes match the remote. Capture repository merge policies,
  default-branch protection summary, remote URL, branch name, SHA, tree hash, and UTC timestamp.
  Record both API and git evidence when both are available. If only one channel is available,
  name the missing cross-check in the confidence block.
- **Tools:** repository API/connector, git (`ls-remote`/`fetch`/`rev-parse` or equivalent), Read.
- **Failure conditions:** default branch or SHA cannot be resolved; local object differs from the
  remote; permission/tool failure prevents a current read. HALT - never substitute a cached ref.
- **Output:** `node-1-remote-anchor.json` and `node-1-remote-anchor.md`.

Required identity fields:

```yaml
repository: owner/repo
default_branch: <resolved name>
baseline_sha: <40-char commit>
baseline_tree: <tree hash>
captured_at: <UTC ISO-8601>
evidence_channels: [github-api, git-remote]
unknown: []
```

## Node 2 - Immutable full-PR capture

- **Action:** Prove the PR exists in `repository`, then capture all observable state: number, URL,
  title, body, author, draft state, base/head branches and SHAs, mergeability, merge method options,
  commit list, full diff/patch, changed-file manifest, labels, assignees, milestone, linked issues,
  requested reviewers, review submissions, inline review threads and resolution state, conversation
  comments, check suites/runs, required-check mapping, deployments/environments relevant to the PR,
  and branch-protection requirements. Extract every explicit goal, non-goal, acceptance criterion,
  and operator decision with a source link/reference. Hash each captured artifact and seal the list
  in `initial-state-manifest.json`; later nodes append new capture versions but never overwrite v1.
- **Tools:** PR/review/check APIs, git diff/patch, Read, hashing tool.
- **Failure conditions:** PR not found; base repository mismatch; base/head SHA unavailable; diff
  unavailable; reviews/checks required for the merge decision are unreadable. HALT with the exact
  missing surface. A 404 is evidence that the example/reference is not a real input, not permission
  to infer its content.
- **Output:** `node-2-pr-capture.json`, `node-2-pr.patch`, `node-2-source-goals.md`, and
  `initial-state-manifest.json` containing path, content hash, source, and capture time.

## Node 3 - Goal contract + simplicity bar (LOOP-20 intake)

- **Action:** Feed node 2's source goals into LOOP-20's clarity validation. Produce one traceable
  goal contract with: target, in-scope behavior, explicit non-goals, acceptance criteria, protected
  invariants, test/probe commands, and unresolved questions. Add the Merge Protocol's default
  simplicity objective without overriding product requirements:
  1. minimum code and dependency surface that satisfies the goal;
  2. clear names/labels and obvious ownership boundaries;
  3. related code organized together, with no speculative abstraction;
  4. control flow a junior developer can trace without hidden magic;
  5. comments explain *why*, not restate *what*;
  6. behavior, security, compatibility, and operability remain intact.
  Establish repeatable baseline measures where the repo supports them: tests, non-comment LOC on
  the touched surface, cyclomatic/cognitive complexity, dependency count, public API surface, and
  a structured junior-reader walkthrough. These are evidence, not code-golf targets.
- **Tools:** LOOP-20 node 1, Read, test/analysis commands, reasoning tier.
- **Failure conditions:** missing target, scope, or success condition; conflicting source goals;
  no executable acceptance probe; an unknown capable of reversing the desired behavior. Route a
  specific clarification request to the operator; do not start mutation.
- **Output:** `node-3-goal-contract.md`, `node-3-traceability.csv`, and `node-3-baseline.json`.

## Node 4 - Cycle-checked composition plan (LOOP-20)

- **Action:** Decompose the goal contract into a task DAG and map each task to the actual loop
  capability that owns it. Run cycle detection and write-set/read-set analysis. Separate two
  authority domains: (a) application/PR work; (b) procedural-memory learning for LOOP-15. Decide
  where each meta-loop participates and record why. LOOP-16 receives only proven-independent
  units; LOOP-19 runs only where multiple credible implementations exist; LOOP-17 is bounded for
  this merge run; LOOP-18 owns every accepted mutation; LOOP-20 continuously re-validates the DAG.
  Run LOOP-11 inline against the plan itself before dispatch.
- **Tools:** LOOP-20 nodes 2-5, LOOP-11 inline, code/knowledge graph if available, Read.
- **Failure conditions:** cycle or contradiction; vibes-based task->loop match; unproven parallel
  write safety; plan review REJECT. Route to the exact invalidated decomposition/goal node.
- **Output:** `node-4-plan-dag.md`, `node-4-loop-map.md`, `node-4-plan-review.md`.

## Node 5 - Independent evidence/proposal fan-out (LOOP-16 + conditional LOOP-19)

- **Action:** Use LOOP-16 to prove independence, isolate, and fan out read-only evidence work such
  as: requirement coverage, behavioral/test surface, structural simplicity/duplication, naming and
  organization, dependency/API blast radius, security boundaries, documentation/integration seams,
  and CI/release-path liveness. Workers may write only their `_loopstate` artifacts. If a material
  finding has more than one credible remedy, run LOOP-19 with at least three forced angles:
  smallest safe patch, clearest structural cleanup, and lowest-risk no-code/config/documentation
  alternative. Preserve provenance and rejected strengths. No candidate touches the PR branch yet.
- **Tools:** LOOP-16 nodes 1-6 in evidence mode, conditional LOOP-19, static analysis, test discovery.
- **Failure conditions:** failed independence proof -> sequence the units; worker output lacks cited
  evidence/confidence; candidate diversity fails; any worker mutates the PR branch. Mutation is a
  Security Failure and halts the run.
- **Output:** `node-5-independence-proof.md`, worker evidence artifacts, and
  `node-5-candidate-register.md`.

## Node 6 - Hostile debate + bounded sparring (LOOP-11 + LOOP-17 + LOOP-15 track)

- **Action:** Run LOOP-11 over the goal contract, PR diff, evidence, and candidate register. Use
  FREE-MAD for ordinary correctness and RA-CR when the proposal changes architecture or a long
  control path. Then run a bounded LOOP-17 session: the breaker attacks the current PR and proposed
  remedies; the builder supplies root-cause explanations and candidate diffs **as proposals only**;
  both communicate through an append-only queue. Stop on the configured round budget or two
  consecutive rounds with no new HIGH/CRITICAL finding, whichever occurs first. Independently feed
  process failures (missed review class, weak gate, bad routing, absent trace) into a LOOP-15 intake
  using its failure->root-cause->successful-fix->generalized-pattern chain. Invoke LOOP-15 only when
  its own preconditions pass, including a fresh LOOP-14 governance report; otherwise emit a handoff
  for a later LOOP-15 run. Its outputs go to a separate procedural-memory package, never the app branch.
- **Tools:** LOOP-11, bounded LOOP-17, LOOP-15 handoff or fully preconditioned invocation, reasoning tier.
- **Failure conditions:** cited CRITICAL ignored; builder proposes only symptom patches; breaker
  mutates; debate reaches silent agreement without evidence; LOOP-15 proposal targets app code or
  weakens a rail. Reject/quarantine rather than continue.
- **Output:** `node-6-debate-verdict.md`, `node-6-sparring-queue.md`,
  `node-6-accepted-candidates.md`, and `node-6-protocol-learning.md`.

## Node 7 - Verified one-change ratchet (LOOP-18)

- **Action:** Order accepted application candidates by requirement impact, risk, and dependency.
  For each candidate, declare one coherent lever and one reproducible harness, then run LOOP-18:
  re-derive baseline, apply exactly one change, re-run the identical harness, compare against the
  tolerance/invariants, and ask LOOP-11's Weld Auditor to verify the evidence. Weld the change as a
  commit only on verified improvement or verified no-regression; otherwise revert the unwelded
  change and try a genuinely different candidate within budget. A simplification may weld only if
  required behavior remains green and it makes at least one named dimension simpler without making
  another materially worse. Never batch unrelated cleanup into a tooth.
- **Tools:** LOOP-18, git on the PR/isolated branch, test/static-analysis harnesses, LOOP-11 inline.
- **Failure conditions:** no authorized writable branch; non-reproducible harness; change touches
  multiple undeclared levers; test, security, API, or operability regression; attempt budget exhausted;
  welded evidence rejected. Without write authority, emit proposals only. Otherwise produce
  FAIL-with-artifact for the candidate, not an optimistic commit.
- **Output:** `node-7-ratchet-ledger.md`, per-tooth baseline/diff/remeasure/decision artifacts,
  and welded commit SHAs. Update the *current* head capture while preserving initial-state v1.

## Node 8 - Pre-merge CI/CD proof (LOOP-08)

- **Action:** Run LOOP-08 in non-deploying gate mode against the actual remote default branch and
  the current PR head. Inventory workflows and required checks; prove every required check maps to
  a real workflow/run for the current head SHA; detect skipped, action-required, startup-failure,
  billing/quota, stale, or nonexistent runs; trace deploy paths for test bypass; verify mergeability,
  approvals, secret-log risk, and branch-protection requirements. Run the repository's relevant
  local/full test matrix from the same locked head. Workflow defects discovered here are findings
  unless workflow repair was already part of the captured goal; the gate never triggers a deploy.
- **Tools:** LOOP-08 nodes 1, 2, 5, 6, and 10 plus repository-specific required nodes; GitHub API;
  local CI-equivalent commands.
- **Failure conditions:** any required check did not really execute and pass; quota/billing block;
  deploy path bypasses tests; unresolved review request; merge conflict; live secret; local/remote
  result mismatch. The merge is blocked.
- **Output:** `node-8-ci-proof.md`, check-run matrix keyed by head SHA, and `node-8-gate-verdict.md`.

## Node 9 - Fresh-state / drift gate (LOOP-20 re-verify)

- **Action:** Immediately before presenting the merge decision, re-read the remote default branch
  and the entire PR state. Compare against node 1, node 2 v1, and the expected ratchet head. Classify
  drift: base SHA, head SHA, diff, body/goals, reviews/threads, required checks, labels, protection,
  and mergeability. Re-run LOOP-20's DAG and contradiction checks with current state. Base movement
  requires integrating/rebasing according to repository policy and re-running every invalidated
  harness and node 8. Unexpected head or goal movement resets to node 2. Review/check-only movement
  resets at least node 8. Seal a `merge-candidate` capture with expected head SHA.
- **Tools:** repository/PR/check APIs, git, LOOP-20 continuous re-verification.
- **Failure conditions:** material drift is unexplained; head cannot be locked; new commits arrive
  during re-verification; conflict with the new base; invalidated gate not rerun. Never wave through
  drift because the earlier capture was green.
- **Output:** `node-9-drift-report.md` and `node-9-merge-candidate.json`.

## Node 10 - Operator approval + SHA-locked merge

- **Action:** Hand the operator a decision packet: repository/default branch, original baseline SHA,
  current base/head SHAs, captured goals, final diff summary, ratchet ledger, adversarial findings,
  real CI proof, unresolved unknowns, merge method, rollback note, and exact post-merge replay plan.
  After explicit approval, re-check the expected head SHA and required checks one final time, then
  invoke the provider's merge operation using the approved method and expected-head protection if
  available. Capture the provider response, actor, timestamp, method, original PR number, merged
  head SHA, and resulting merge commit SHA. The loop never pushes directly to the default branch.
- **Tools:** GitHub merge API/connector, read-only recheck, Write receipt.
- **Failure conditions:** no explicit approval; expected head mismatch; gate state changed; provider
  cannot enforce/verify the expected head; merge fails or produces an unknown commit. Return to node 9.
- **Output:** `node-10-decision-packet.md` and immutable `node-10-merge-receipt.json`.

## Node 11 - Post-merge remote snapshot

- **Action:** Resolve the remote default branch again until the merge receipt's resulting commit is
  reachable from it (within a declared timeout). Capture the new default-branch SHA/tree, merge-base,
  post-merge checks, and any commits that landed concurrently. Materialize a clean read-only checkout
  of that exact SHA. If concurrent commits exist, keep them in scope: the next node intentionally
  evaluates the integrated branch users now receive, not a synthetic branch containing only the PR.
- **Tools:** repository API, git fetch/rev-list/checkout in isolation, check APIs.
- **Failure conditions:** merge commit never becomes reachable; default branch advances in a way the
  checkout cannot reproduce; post-merge checks are unreadable. HALT with receipt preserved.
- **Output:** `node-11-postmerge-snapshot.json` and `node-11-concurrent-commits.md`.

## Node 12 - Replay initial capture over the entire default branch

- **Action:** Load the immutable node-2/node-3 v1 artifacts from disk, verify their hashes, and replay
  every captured goal, non-goal, invariant, acceptance probe, review finding, CI expectation, and
  named unknown against node 11's exact **whole default-branch tree**. Re-run the full relevant test
  suite plus integration checks across callers/consumers outside the original diff. Re-run LOOP-20's
  plan verification against current architecture; use LOOP-16 only for independent read-only replay
  units; use the LOOP-17 breaker track to probe regressions while its builder track remains proposal-
  only; finish with LOOP-11's hostile whole-state verdict. Compare baseline->PR->post-merge for behavior,
  requirements coverage, code/dependency/complexity measures, readability, CI liveness, and unknowns.
- **Tools:** sealed initial manifest, git, tests/static analysis, LOOP-20, LOOP-16, bounded LOOP-17,
  LOOP-11, applicable domain loops named by node 4.
- **Failure conditions:** manifest hash mismatch; replay silently narrows to changed files; test or
  requirement regression; review finding reappears; integration seam omitted; unknown is dropped
  instead of resolved/carried forward.
- **Output:** `node-12-replay-matrix.md`, `node-12-regressions.md`,
  `node-12-whole-state-verdict.md`, and comparison metrics.

## Node 13 - Gap classification + follow-up ratchet

- **Action:** Deduplicate node 12 findings into a gap register with classes: `regression`,
  `missing-requirement`, `integration-gap`, `simplification`, `documentation`, `process-learning`,
  `operator-deferred`, or `false-positive`. Assign severity, evidence, owner, source goal, and whether
  it blocks a PASS. CRITICAL/HIGH regressions produce FAIL-with-artifact and an urgent remediation or
  revert proposal; a follow-up PR does not launder them into success. For actionable application gaps,
  create an isolated branch from node 11's exact default-branch SHA and apply fixes through LOOP-18,
  one verified tooth at a time. Route process-learning items to the separate LOOP-15 package.
- **Tools:** reasoning tier, LOOP-11 dedup/verdict, LOOP-18, git isolated branch, LOOP-15 package path.
- **Failure conditions:** finding lacks evidence; follow-up scope mixes unrelated debt; branch is not
  based on the captured post-merge SHA; fixes are batched or unverified; critical regression has no
  explicit operator escalation.
- **Output:** `node-13-gap-register.md`, follow-up ratchet ledger/commits if any, and
  `node-13-protocol-learning-package.md` if needed.

## Node 14 - Follow-up PR or evidence-backed NO-CHANGE close-out

- **Action:** Present the gap register and follow-up diff to the operator. When an actionable verified
  diff exists and approval is given, push the isolated branch and open a **draft** follow-up PR. Its
  title/body links the original PR, baseline SHA, merged head, merge commit, post-merge snapshot SHA,
  replay verdict, gap IDs, tests, and parent `run-id`; it states scope and non-goals and carries no
  unrelated cleanup. The follow-up may be run through a new Merge Protocol invocation, but this run
  never auto-merges it and never exceeds `--followup-depth`. If no actionable diff exists, record
  `NO-CHANGE` with the evidence that every captured criterion passed; do not create an empty branch/PR.
- **Tools:** git push, PR API/connector, Write.
- **Failure conditions:** PR would be empty; branch contains unrelated commits; lineage/evidence is
  missing; operator has not approved opening it; recursion depth exceeded.
- **Output:** `node-14-followup-pr.md` with URL/number, or `node-14-no-change.md`, plus the standard
  `AUDIT-ARTIFACT.md` and governance row.

# Initial-State Replay Contract

The post-merge replay is a deterministic comparison, not a fresh interpretation of the PR:

| Captured before mutation | Replayed after merge |
|--------------------------|-----------------------|
| Remote default branch name/SHA/tree | Exact integrated default-branch name/SHA/tree and reachability of merge receipt |
| PR goals/non-goals/acceptance criteria | Requirement-by-requirement PASS/FAIL with evidence on the whole tree |
| Original diff, commits, files, API/dependency blast radius | Integrated behavior across touched code plus all known callers/consumers |
| Reviews, comments, unresolved threads, explicit decisions | Every finding remains resolved or is carried into the gap register |
| Required checks and real run history | Post-merge checks actually ran; no stale or skipped green state |
| Baseline tests/probes and simplicity measures | Same harnesses plus full integration suite; baseline->PR->post-merge deltas |
| Unknown/missing-evidence lists | Each unknown resolved, still explicit, or promoted to a blocking gap |

# Adversarial Check

Protocol: **RA-CR** for the final merge decision, with FREE-MAD allowed inside low-risk evidence
units. Personas:

1. **The Merge Saboteur** - tries to land an unreviewed commit between capture and merge, pass a
   skipped/stale check as green, hide a merge conflict, exploit a branch-protection mismatch, or
   make the merge receipt point at a different head than the one reviewed.
2. **The Junior Maintainer** - assumes they inherit this code tomorrow and attacks cleverness,
   vague names, scattered ownership, undocumented invariants, unnecessary abstraction, and cleanup
   that reduced LOC while making the behavior harder to trace.
3. **The Follow-Up Launderer** - hunts cases where a regression is downgraded to "future work" so
   the run can claim PASS, or where unrelated debt is smuggled into the follow-up PR.

They attack, in order: remote/base authenticity; completeness and hash integrity of the initial
capture; goal traceability; single-tooth mutation evidence; CI liveness at the locked head;
last-moment drift; merge-receipt identity; whole-tree post-merge replay coverage; and follow-up
lineage/scope. Round-1 agreement without cited SHAs, run IDs, file/goal references, and replay rows
is Silent Agreement and forces another hostile round.

# Exit Criteria (quantitative, overrides [PROTOCOL] section 12)

- Remote default branch and baseline are recorded by exact name, commit SHA, tree hash, timestamp,
  and evidence channel; no assumed `main`/`master` string.
- Initial PR capture includes every readable surface listed in node 2, all omissions appear in the
  confidence block, and 100% of capture artifacts match `initial-state-manifest.json` hashes.
- 100% of goal-contract statements trace to PR/issue/review/operator evidence; zero unresolved
  contradictions at merge time.
- Plan DAG is cycle-free; every parallel unit has a passing independence proof; zero parallel writers.
- Every welded application change has one declared lever, reproducible before/after harness, passing
  invariants, LOOP-11 approval, and a commit SHA; zero unwelded changes remain.
- Every required check has a real successful run for the locked head SHA; zero skipped, stale,
  missing, quota-blocked, or unresolved required checks.
- Operator explicitly approves the exact merge candidate; merge receipt's merged head equals the
  approved head; resulting commit is reachable from the post-merge default branch.
- Post-merge replay verifies 100% of initial goals, findings, tests/probes, CI expectations, and
  unknowns against the whole integrated tree; no diff-only shortcut.
- PASS requires zero unresolved CRITICAL/HIGH regressions or missing requirements. A draft follow-up
  may carry bounded non-blocking work, but cannot convert a failing core goal into PASS.
- Follow-up decision is explicit: approved draft PR with non-empty verified diff and full lineage,
  or evidence-backed `NO-CHANGE`. Final confidence >= 0.90; governance row written.

# Failure Routing (deviations from [PROTOCOL] section 4)

- **Remote/API disagreement or stale-only access:** HALT-REMOTE-TRUTH. No cached-baseline fallback.
- **PR reference not found:** HALT-INPUT-NOT-FOUND. Preserve the literal reference; do not synthesize a PR.
- **Capture hash mismatch:** Security Failure; quarantine the run state and recapture from node 1.
- **Goal ambiguity/contradiction:** route to LOOP-20 node 1 with the exact conflicting sources.
- **Base/head/goal drift:** reset to the earliest invalidated node; never edit timestamps in place.
- **CI quota/billing/skipped required check:** Dependency Failure that blocks merge and escalates to
  the operator; retrying the same invisible check does not prove liveness.
- **Merge race or expected-head mismatch:** abort merge and return to node 9.
- **Post-merge CRITICAL/HIGH regression:** final verdict FAIL-with-artifact; prepare urgent remediation
  or revert PR from the exact post-merge SHA and alert the operator.
- **No actionable post-merge gap:** emit `NO-CHANGE`; an empty PR is a Workflow Failure, not success.
- **Follow-up recursion beyond budget:** halt lineage and require a new operator-scoped run.

# Approval Gates (deviations from [PROTOCOL] section 10)

- Read-only capture, planning, debate, and replay are automatic.
- Each refactor tooth on the incoming or follow-up branch requires LOOP-11 Weld Auditor sign-off.
- Any change outside the captured goal, any security/cost/permission threshold, any destructive
  migration, and any live-infrastructure action requires separate explicit operator approval.
- **Node 10 merge always requires just-in-time operator approval of the exact base/head/method packet.**
- **Node 14 opening a follow-up PR requires the gap register and diff to be handed to the operator
  first.** Approval opens a draft only; merging it is a separate future decision.
- LOOP-15 procedural-memory proposals retain LOOP-15's per-proposal approval and separate branch;
  they are never bundled into the application PR or its follow-up.

# RUN PROMPT

```text
Run LOOP-21 (Merge Protocol) on <owner/repo> PR #<N>.
Read PROTOCOL.md and skills/LOOP-21-merge-protocol.md, then execute the DAG
end-to-end.

First resolve the repository's REAL remote default branch and exact SHA; do
not assume it is named main and do not trust a stale local checkout. Capture
the full PR state, goals, diff, reviews/comments/threads, checks, labels,
linked issues, base/head SHAs, branch protection, and unknowns. Hash and seal
that initial capture before any mutation.

Compile the goal with LOOP-20. Default implementation objective: satisfy the
captured goal with the smallest working, well-labeled, well-organized code a
junior developer can trace. Use LOOP-16 only for proven-independent evidence
work, LOOP-19 for genuinely different remediation candidates, LOOP-11 plus a
bounded LOOP-17 session to attack them, and LOOP-18 to apply one verified
change at a time. LOOP-15 may propose improvements to the loop system only;
it must never edit application code.

Run LOOP-08's real CI-liveness and merge gates against the current locked head.
Immediately re-capture remote/PR state before the decision. Hand me the exact
base/head/method/evidence packet and wait for explicit approval before merging.

After merge, resolve and check out the exact integrated remote default-branch
SHA. Verify the sealed initial capture, then replay every original goal,
finding, test/probe, CI expectation, and unknown against the ENTIRE default
branch, including callers and integration seams outside the original diff.
Classify regressions, gaps, simplifications, and process learning. Prepare a
scoped ratcheted follow-up branch for actionable gaps; show me its gap register
and diff before opening a draft PR. Never create an empty PR and never merge
the follow-up automatically.
```
