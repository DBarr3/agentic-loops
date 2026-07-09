---
name: loop-sparring-partners
loop-id: LOOP-17
description: Standing red-team/blue-team co-evolution loop — a persistent breaker and a persistent builder run concurrently against the same artifact, coupled by a shared findings/fix queue, reporting independently to the human
domain: Continuous Adversarial Hardening
risk-class: branch-mutating
default-debate: FREE-MAD
model-tiers: {scan: cheap, audit: mid, verdict: reasoning}
---

# Mission

Keep an artifact under permanent adversarial pressure instead of auditing it once and
walking away. LOOP-17 stands up two long-lived agent roles — a **breaker** that never
stops looking for the next way to break the artifact, and a **builder** that never stops
hardening it — and lets them run concurrently, each on its own retry/iteration cadence,
coupled only by a shared findings/fix queue: the breaker's latest finding becomes the
builder's next input, the builder's latest fix becomes the breaker's next target. This is
**not** [PROTOCOL] step 4 or LOOP-11's inline debate-and-verdict engine — LOOP-11 is a
bounded, one-shot generator-vs-reviewer debate that terminates in a verdict; LOOP-17 is an
unbounded, standing loop that keeps running rounds until a convergence condition or a
round budget or the human pulls the kill-switch. LOOP-17 calls LOOP-11 inline wherever a
single round needs a scored verdict (e.g. "is this fix real or cosmetic"), but the outer
loop itself has no single terminal verdict — it has a trend. The human is not a
synchronization point inside the loop; both agents report to the human asynchronously, and
the human's job is oversight, difficulty-raising, and the kill-switch — never per-round
approval.

# Trigger (when the operator runs this)

- Standing background process against a stable artifact you want continuously hardened
  (a security-sensitive module, a public API surface, a spec, a running service) — start it
  and let it run rounds until convergence or budget, not a single invocation per bug.
- On demand: `Run LOOP-17 on <target>` to start a fresh session, or `Run LOOP-17 --resume <run-id>`
  to continue a suspended one.
- After LOOP-01/LOOP-05/etc. clear their backlog of known findings and you want ongoing
  pressure instead of waiting for the next scheduled sweep.

# Inputs (target repo/dir, scope flags)

- `target`: the artifact under standing adversarial pressure — a repo/module, a running
  service endpoint, or a spec/contract document.
- `--round-budget <N>`: hard cap on total rounds (default 20, required if `--converge-on`
  is not set).
- `--converge-on <no-new-finding-streak>`: stop after this many consecutive breaker rounds
  with zero new findings (default 5).
- `--difficulty <tier>`: starting attack-difficulty tier the breaker must clear before the
  loop can report convergence at that tier (default: reuse the calling scope's existing
  severity taxonomy).
- `--report-interval <rounds|time>`: cadence for the human-facing findings/fix feed (default
  every round).

# Preconditions

- Git worktree clean; builder's mutations land only on `loop/LOOP-17-<YYYY-MM-DD>`, created
  from origin/main before round 1 — the breaker never commits to this branch or any branch.
- Breaker has read/attack access to the target (sandboxed execution environment for anything
  beyond static analysis — never attack a shared production instance without an explicit
  isolated-target confirmation).
- A code-graph/knowledge-graph tool if available, so both roles trace structure instead of
  pattern-matching; ast-grep/Semgrep as the structural-scan fallback.
- A shared, append-only findings/fix queue artifact both roles read and write to
  (`_loopstate/LOOP-17/<run-id>/queue.md`) — this is the only coupling channel between them.

# Execution DAG (numbered nodes, [P] = parallelizable)

This DAG is **not** a single linear pipeline. It is two concurrent, independently-paced
loops — the breaker's cycle (nodes 1-3) and the builder's cycle (nodes 4-6) — coupled only
through the shared queue (node 7), plus two nodes that sit outside both cycles: the
human-report node (8, fed by both continuously) and the convergence-check node (9, which
decides whether either cycle gets to keep running). Represent it as two self-looping tracks
joined at the queue, not as node-1-then-node-2-then...-then-node-9:

```
   [Breaker track]              [Builder track]
   1 → 2 → 3 ─┐              ┌─ 4 → 5 → 6
   ↑          │              │          ↓
   └──────────┤   7 (queue)  ├──────────┘
              └──────┬───────┘
                      ├──→ 8  (human report, continuous, both tracks feed it)
                      └──→ 9  (convergence check, gates both tracks' next round)
```

- Nodes 1-3 (find attack → attempt → report) repeat on the breaker's own cadence, never idle.
- Nodes 4-6 (read latest findings → fix/build → ship) repeat on the builder's own cadence,
  never idle, and never lockstepped node-for-node with the breaker.
- Node 7 is the only synchronization surface: it is where each track reads the other's
  latest output, not a merge or barrier — either track may be several internal iterations
  ahead of the other at any moment.
- Node 8 runs continuously alongside both tracks, not after them.
- Node 9 runs after every queue write and can halt either or both tracks.

# Node Specs

**Node 1 — Attack surface scan (breaker track).**
Action: from the current state of the target (post-latest builder fix per the queue),
enumerate candidate attack vectors not yet exhausted this session — reuse OWASP-class
categories, fuzz targets, contract-violation probes, or domain-specific attack classes
relevant to the target type. Prioritize variants of previously-successful attack classes
first (cheapest signal on whether the builder fixed root cause vs. symptom).
Tools: a code-graph/knowledge-graph tool if available, ast-grep/Semgrep, Read, fuzzing/probe
tooling appropriate to the target type.
Failure conditions: no candidate vectors remain untried this round (Logic Failure — this is
not an error, it feeds directly into node 9 as a possible no-new-finding round).
Output artifact: `breaker-round-<N>-candidates.md`.

**Node 2 — Attempt (breaker track).**
Action: execute the highest-priority candidate against a sandboxed instance of the target.
Never against a shared production target without explicit isolated-target confirmation.
Tools: Bash (sandbox exec), the target's own test/exploit harness, Read.
Failure conditions: sandbox unavailable (Tool Failure — halt breaker track only, builder
track continues; escalate to operator).
Output artifact: `breaker-round-<N>-attempt.md` (vector | steps | result: broke it | held).

**Node 3 — Report & loop (breaker track).**
Action: write the result to the shared queue (node 7) with severity, evidence, and whether
this is a **novel** finding or a **variant of a previously-reported bug class** — the
variant flag is what feeds the root-cause-vs-symptom check. Immediately return to node 1;
the breaker never waits for the builder to finish before starting its next attack.
Tools: Write (queue only).
Failure conditions: cannot classify novel-vs-variant with confidence ≥0.70 → mark `unknown`,
route to reasoning-tier classification before the round counts toward convergence.
Output artifact: queue entry (breaker-authored), confidence block attached.

**Node 4 — Read latest findings (builder track).**
Action: pull unresolved entries from the shared queue, ranked by severity then recency.
Distinguish an entry that is a variant of an already-"fixed" bug class — that is itself a
signal the previous fix addressed a symptom, not the root cause, and must be flagged before
attempting another symptom patch.
Tools: Read (queue).
Failure conditions: queue empty and no capability-extension backlog exists → builder idles
this round (not a failure, logged as such).
Output artifact: `builder-round-<N>-intake.md`.

**Node 5 — Fix or extend (builder track).**
Action: on the loop branch, fix the highest-priority finding at root cause (trace to the
actual defect class, not just the reproduction steps), or — if the queue is clear — extend
the artifact's capability and invite the breaker at the new surface. Run the existing test
suite plus the breaker's reproduction as a new regression test before shipping.
Tools: Edit, Bash (tests, git), a code-graph/knowledge-graph tool if available to confirm
the fix addresses the actual call path, not a superficial symptom.
Failure conditions: fix cannot be verified against the breaker's exact reproduction (Logic
Failure — do not ship; re-diagnose root cause).
Output artifact: commit on `loop/LOOP-17-<date>` + `builder-round-<N>-fix.md`.

**Node 6 — Ship & signal readiness (builder track).**
Action: mark the queue entry resolved with the commit reference and a root-cause summary;
signal the target is ready for the next breaker round at (at minimum) the same difficulty.
Immediately return to node 4 — the builder never waits idle for an explicit go-ahead.
Tools: Write (queue).
Failure conditions: none halting; a resolved-but-unverified entry is not permitted — route
back to node 5.
Output artifact: queue entry (builder-authored), confidence block attached.

**Node 7 — Shared queue (coupling surface).**
Action: append-only log both tracks read/write; never edited retroactively (corrections are
new entries referencing the old one, preserving the audit trail). This is the sole channel
of coupling — breaker and builder never call each other directly or share execution state.
Tools: Write/Read (`_loopstate/LOOP-17/<run-id>/queue.md` only).
Failure conditions: write conflict between simultaneous track writes → last-write-wins is
NOT acceptable; both entries persist, ordered by timestamp.
Output artifact: `queue.md` (append-only).

**Node 8 — Human report (continuous, both tracks feed it).**
Action: emit a rolling status feed (dashboard/log/digest per `--report-interval`) showing:
breaker win-rate trend over the session, open vs. resolved queue entries, root-cause vs.
symptom-fix ratio, and any HALT conditions. The human is a consumer of this feed, never a
per-round approval gate — the loop does not block on the human reading it.
Tools: Write (report artifact); no mutation authority.
Failure conditions: none halting; a stale feed (no update within `--report-interval`) is
itself logged as a Tool Failure against node 8, not against either track.
Output artifact: `_loopstate/LOOP-17/<run-id>/human-report.md`, updated continuously.

**Node 9 — Convergence check.**
Action: after every queue write, evaluate: (a) has the breaker gone `--converge-on` (default
5) consecutive rounds with zero novel findings at the current difficulty tier — if so, either
raise the difficulty tier (operator-gated, see Approval Gates) or converge; (b) has the
round budget been hit; (c) has the breaker's win-rate trend gone flat or risen over the last
5 rounds — a rising/flat trend is itself a finding, not silence, and forces a HALT for
human review rather than a quiet continuation; (d) has the symptom-fix ratio (variant
re-findings ÷ total findings) crossed 0.3 — if so, HALT and route to root-cause review
before either track continues.
Tools: none (control logic over the queue's structured data).
Failure conditions: any of (c) or (d) → HALT-<class>, not silent continuation; this is the
loop's own meta-adversarial guard as much as a control node (see next section).
Output artifact: `convergence-round-<N>.md` (verdict: CONTINUE | RAISE-DIFFICULTY | HALT | CONVERGED).

# Adversarial Check (reviewer persona + what it attacks)

Persona: **skeptical embedded referee** — a role distinct from both breaker and builder,
running FREE-MAD, whose sole job is to police whether the two tracks are actually improving
the artifact rather than performing motion. It attacks, in order:

1. The breaker's "novel" classifications — re-derives whether each "new" finding is
   actually a fresh vector or an unlabeled variant of a bug class already in the queue
   (cross-checks node-3's confidence block first).
2. The builder's fixes — for every resolved queue entry, demands the root-cause trace, not
   the passing reproduction test alone; a fix that only defeats the breaker's exact
   reproduction steps without addressing the underlying defect class is REJECTed back to
   node 5, and the resolved-count does not advance.
3. The breaker win-rate trend itself — if it is flat or rising over a 5-round window while
   the builder keeps marking entries "resolved," that is direct evidence of symptom-chasing
   or a breaker that has stalled at the current difficulty tier; either is a finding against
   the loop, not just the artifact.
4. Isolation discipline — confirms the breaker made zero commits and zero edits to the loop
   branch this session (breaker-as-attacker must never also be breaker-as-silent-fixer).
5. Silent Agreement: if a round produces no disagreement between breaker and builder framing
   of a finding with no cited evidence either way, force a hostile extra pass before letting
   node 9 count it toward convergence.

Maximum 2 debate rounds per queue entry under review; disagreements surviving round 2 are
recorded as operator-deferred findings against the loop's own health, never silently dropped.

# Exit Criteria (quantitative, overrides PROTOCOL.md §12)

- CONVERGED requires: `--converge-on` consecutive breaker rounds (default 5) with zero
  novel findings at the current difficulty tier, AND breaker win-rate trend strictly
  decreasing or flat-at-zero over the same window (a flat-nonzero trend does not converge —
  see node 9c).
- Symptom-fix ratio (variant re-findings ÷ total findings) must stay below 0.3 for the run
  to report CONVERGED; at or above 0.3 the run reports HALT-ROOT-CAUSE, never CONVERGED.
- Isolation discipline: zero breaker commits/edits on the loop branch for the entire run —
  any violation is an immediate CRITICAL and HALTs the loop regardless of round count.
- Every resolved queue entry carries a permanent regression test built from the breaker's
  exact reproduction; test pass rate ≥ 99% on the touched surface.
- Round budget respected (default 20, operator override); a run that hits budget without
  converging reports FAIL-with-artifact carrying the full trend data, never silent success.
- Final confidence ≥ 0.85 on the convergence verdict; governance row written per run, plus
  one summary row per session covering the full round history.

# Failure Routing (only deviations from PROTOCOL.md §4)

- Rising/flat breaker win-rate over a 5-round window (node 9c) → HALT-TREND, human review
  required before either track resumes; not classified as a normal Logic Failure because the
  defect is in the loop's dynamics, not a single node's output.
- Symptom-fix ratio ≥ 0.3 (node 9d) → HALT-ROOT-CAUSE; builder track suspended, breaker
  track may continue read-only cataloguing but no further fixes ship until an operator or a
  reasoning-tier pass re-diagnoses the underlying bug class.
- Breaker touches the loop branch (isolation violation) → Security Failure per PROTOCOL.md
  §4, HALT LOOP entirely, CRITICAL artifact, human approval required to resume.

# Approval Gates (only deviations from PROTOCOL.md §10)

- Raising the attack-difficulty tier (node 9's RAISE-DIFFICULTY path) requires explicit
  operator approval — the loop proposes the next tier and waits; it never self-escalates
  difficulty silently.
- Starting a session against a shared/production instance (rather than a sandboxed target)
  requires explicit operator confirmation before node 1 of round 1, every session.
- Merge of the loop branch to main follows PROTOCOL.md §10 defaults: operator approval,
  findings handed back first — LOOP-17 itself never merges.

# RUN PROMPT

```
Run LOOP-17 on <target> (sandboxed). Follow LOOP-17-sparring-partners.md under [PROTOCOL]
(see PROTOCOL.md in this repo). Start breaker and builder tracks concurrently, coupled only
through _loopstate/LOOP-17/<run-id>/queue.md; breaker never mutates, builder mutates only on
loop/LOOP-17-<date>. Report to me continuously via human-report.md — do not wait for my
approval each round. Converge after 5 consecutive no-new-finding breaker rounds or a 20-round
budget, whichever first; HALT and ping me on a flat/rising win-rate trend or a symptom-fix
ratio ≥ 0.3. Ask me before raising the difficulty tier or targeting anything non-sandboxed.
```
