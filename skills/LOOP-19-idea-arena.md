---
name: loop-idea-arena
loop-id: LOOP-19
description: Convergent ideation engine — diverse candidate ideas generated in parallel enter one shared adversarial-debate-and-mutation arena, converge to a single provenance-tracked synthesized idea, and get packaged for execution-agent handoff
domain: Ideation & Convergent Synthesis
risk-class: read-only (produces a spec/handoff artifact, never mutates code itself)
default-debate: FREE-MAD
model-tiers: {scan: cheap, audit: mid, verdict: reasoning}
---

# Mission

Turn one seed request into a single, defensible idea — without the two failure modes that plague ad-hoc brainstorming: **accidental convergence** (three "independent" generations that are secretly the same idea in different words) and **elimination bias** (picking one candidate wholesale and quietly throwing away its rivals' good parts). LOOP-19 forces genuine diversity at generation time, puts all candidates into **one shared arena simultaneously** — every idea can attack every other idea and propose mutations that graft strengths across them, in the same rounds, not a bracket of pairwise duels — and converges on a synthesized idea whose every element is traceable back to the candidate(s) that earned it. Losing candidates' unused strengths are never silently discarded; they land in a rejected-idea register. The loop's only mutation surface is documentation: it emits a spec-level handoff packet for an execution agent to build. It never touches application code itself.

This is distinct from [PROTOCOL]'s inline adversarial check and from LOOP-11's standalone debate engine: LOOP-11 is one generator vs one (or more) reviewer(s) refining a single artifact. LOOP-19's arena is N candidates co-present, all mutating and attacking each other at once — LOOP-11 is invoked here only as the final gate over the synthesized handoff packet (§ Adversarial Check).

# Trigger (when the operator runs this)

- A seed request needs more than one take before committing engineering time to it — new feature direction, a build-vs-buy call, a redesign, a go-to-market angle.
- Before handing a fuzzy "figure out how to do X" ask to an execution agent — run the idea through the arena first so the execution agent gets a spec, not a vibe.
- On demand: `Run LOOP-19 on <seed request>`.

# Inputs (target repo/dir, scope flags)

- `seed`: the incoming request/brainstorm prompt — REQUIRED, and must be concrete enough to extract a goal and hard constraints (budget, timeline, non-negotiables). A one-line vague ask is not sufficient input (see node 1 failure condition).
- `context`: any existing architecture docs/ADRs/product context the synthesized idea must stay compatible with (used by node 8's coherence check).
- `--candidates <N>`: number of parallel idea-generation nodes, default 3 (this loop's node numbering assumes 3; more candidates fan the arena out further under the same protocol).
- `--diversity-threshold <0.0-1.0>`: max allowed pairwise similarity before a regenerate is forced, default 0.75 (lower = stricter).
- `--rounds <N>`: arena round budget, default 3 (protocol max 4, per [PROTOCOL] §5).
- `--execution-target`: the agent/team the handoff packet is destined for (informs node 9's spec level).

# Preconditions

- Seed request carries an extractable goal and at least one hard constraint; otherwise node 1 halts and asks the operator, rather than inventing scope.
- `_loopstate/LOOP-19/<run-id>/` writable for arena logs, provenance table, and the rejected-idea register.
- LOOP-11 available (inline) for the final adversarial check over the synthesized idea and handoff packet.
- Read-only to any codebase: this loop may Read existing architecture docs/context for the coherence check, but never Edits application code.

# Execution DAG (numbered nodes, [P] = parallelizable)

1. Ingest & scope — extract goal, hard constraints, candidate count, forced-angle assignments.
2. [P] Idea 1 generation — forced angle: safest incremental option.
3. [P] Idea 2 generation — forced angle: most ambitious / highest-ceiling option.
4. [P] Idea 3 generation — forced angle: cheapest / fastest option.
5. Diversity gate — reject/regenerate any candidate that's a restatement, not a reframe.
6. Shared arena: debate + mutate — all three candidates co-present, multi-round attack + graft.
7. Convergence & synthesis — build the winning idea with a full provenance trail; stock the rejected-idea register.
8. Coherence check — verify the grafted synthesis doesn't Frankenstein incompatible assumptions together.
9. Handoff packaging — spec-level packet for the execution agent; final [PROTOCOL] adversarial check (LOOP-11 inline) before emit.

Nodes 2, 3, 4 run in parallel after node 1, each seeded with its own forced angle so divergence is structural, not accidental. Node 5 gates entry to the arena. Nodes 6 → 7 → 8 → 9 run strictly sequentially — each is the next node's only input.

# Node Specs

### Node 1 — Ingest & scope
- **Action:** Parse the seed request into a goal statement, explicit hard constraints (budget, timeline, non-negotiables), and any existing product/architecture context relevant to later coherence checking. Confirm candidate count (default 3) and lock in each candidate's forced differentiator (angle, constraint profile, risk tolerance, or audience) so generation cannot drift into near-duplicates by default.
- **Tools:** Read (context docs if provided).
- **Failure conditions:** goal or constraints not extractable from the seed → do not invent scope; halt and request the missing detail from the operator (Workflow Failure).
- **Output artifact:** `node-1-scope.md` (goal, constraints, candidate angle assignments).

### Node 2/3/4 — Idea generation [P]
- **Action:** Each node generates one candidate under its assigned forced angle from node 1 (e.g. idea1 = safest incremental option respecting existing architecture; idea2 = most ambitious rewrite with the highest ceiling; idea3 = cheapest/fastest option that ships minimum viable value) as a structured proposal: problem framing, approach, key risks, rough cost/time estimate, and what it explicitly does **not** do.
- **Tools:** reasoning (mid tier), Read for existing context.
- **Failure conditions:** candidate silently violates a node-1 hard constraint → regenerate with the violation named; a candidate omits a required field → regenerate.
- **Output artifact:** `node-2-idea-1.md`, `node-3-idea-2.md`, `node-4-idea-3.md`.

### Node 5 — Diversity gate
- **Action:** Score every pair of candidates against the diversity threshold — via semantic-similarity scoring if available, otherwise an explicit checklist requiring the pair to differ on at least 2 of 3 axes (approach, cost/risk profile, scope). A candidate that is a restatement of another (not a genuinely different angle) is sent back to node 2/3/4 for regeneration with a sharper forced angle, not just a reword instruction.
- **Tools:** reasoning, similarity scoring if available.
- **Failure conditions:** same pair fails diversity twice in a row → Logic Failure, escalate to reasoning tier with a harder-forced constraint on the weaker candidate.
- **Output artifact:** `node-5-diversity.md` (pairwise scores, verdicts, regeneration log).

### Node 6 — Shared arena: debate + mutate
- **Action:** Instantiate the arena — all three candidates present in the SAME rounds (never a pairwise elimination bracket). Turn order per round: each candidate's advocate attacks weaknesses in **both** other candidates, then proposes at least one mutation grafting a strength from another candidate onto its own (e.g. "idea 3's cost model bolted onto idea 1's architecture"). A neutral arbiter scores and accepts/rejects each attack and each mutation proposal with cited rationale. Round budget default 3 ([PROTOCOL] §5, hard max 4). Track, per candidate, a running count of attacks-received and mutation-attempts — this count is the premature-convergence guard.
- **Tools:** reasoning tier, FREE-MAD arena mechanics (multi-party, not single generator/reviewer).
- **Failure conditions:** any candidate reaches round-budget exhaustion with zero attacks-received or zero mutation-attempts logged against/from it → Logic Failure; force one additional hostile round targeting specifically the neglected candidate before the arena is allowed to close.
- **Output artifact:** `node-6-arena-log.md` (per round: attacks, mutation proposals, accept/reject + rationale, arbiter scores, running attack/mutation counts per candidate).

### Node 7 — Convergence & synthesis
- **Action:** Build ONE synthesized idea from the arena log — not "candidate 2 wins, discard 1 and 3," but a constructed idea whose every element (architecture choice, cost model, scope boundary, risk mitigation, etc.) is traceable to the specific candidate(s) and arena round that earned it. Produce a provenance table (`element → source candidate → accepting arena round`). Any accepted-but-unused strength from a non-winning candidate is written to a rejected-idea register — preserved as a future option, never silently dropped.
- **Tools:** reasoning tier, Write.
- **Failure conditions:** an element appears in the synthesis with no corresponding accepted arena entry → Logic Failure; strip the untraceable element and resynthesize.
- **Output artifact:** `node-7-synthesis.md`, `node-7-provenance-table.md`, `node-7-rejected-register.md`.

### Node 8 — Coherence check (Frankenstein guard)
- **Action:** Before handoff, verify the synthesized idea's grafted elements are compatible with **each other**, not just individually sound — check for conflicting assumptions between grafts (e.g. one graft assumes synchronous flow, another assumes async batching), conflicting non-goals, and any dependency the combined synthesis silently requires that no single source candidate actually validated. This is distinct from node 6's mutation-acceptance scoring, which judges "is this graft good on its own" — coherence judges "do all the accepted grafts still fit together once combined."
- **Tools:** reasoning tier, Read on existing architecture/context if provided.
- **Failure conditions:** incompatible graft found → route back to node 7 with the conflict named explicitly (max 2 resynthesis cycles); if still incoherent, fall back to the single strongest candidate wholesale, labeled explicitly as "no synthesis achieved — fallback to candidate <N>," rather than shipping an incoherent hybrid.
- **Output artifact:** `node-8-coherence.md` (conflicts found, resolution or fallback verdict).

### Node 9 — Handoff packaging
- **Action:** Package the converged idea (synthesis or, on coherence failure, the labeled fallback) into a spec-level handoff contract for an execution agent: scope, explicit non-goals (including "do not silently reintroduce the discarded elements of candidate X" wherever the arena rejected them for a reason), acceptance criteria, and a provenance summary linking back to node 7's table and the rejected-idea register. Run the [PROTOCOL] §1 step-4 adversarial check (LOOP-11 inline, FREE-MAD) over the packet before emit.
- **Tools:** Write, LOOP-11 inline.
- **Failure conditions:** packet missing scope, non-goals, acceptance criteria, or provenance → incomplete, not emitted as final; LOOP-11 verdict REJECT → return to node 7/8 with the critique as new context ([PROTOCOL] §1 step 4).
- **Output artifact:** `HANDOFF-PACKET.md` + `AUDIT-ARTIFACT.md` ([PROTOCOL] §13).

# Adversarial Check (reviewer persona + what it attacks)

Protocol: FREE-MAD, persona **the Arena Referee** — a hostile reviewer whose sole job is catching the two failure modes this loop is designed against:

1. **Premature convergence** — pulls node 6's per-candidate attack/mutation counts and rejects any run where one candidate was treated as a strawman (near-zero attacks received, no mutation attempts drawn from it) while another was crowned early. Demands evidence the neglected candidate got a genuine hostile round, not a token one.
2. **Frankenstein synthesis** — independently re-checks node 8's coherence verdict rather than trusting it; hunts for grafted elements whose combined assumptions were never actually tested against each other, and for any provenance-table entry whose "accepting arena round" doesn't actually show an accept.
3. **Silent Agreement** — if the arena's own arbiter accepted every mutation proposal in round 1 with no cited rationale, treats that as conformity, not consensus ([PROTOCOL] §11), and forces one additional hostile round.
4. **Register integrity** — confirms the rejected-idea register actually captured the losing candidates' distinct strengths rather than being left empty because everything "lost cleanly."

Maximum 2 rounds; disagreements surviving round 2 are recorded as operator-deferred findings on the handoff packet, never silently dropped.

# Exit Criteria (quantitative, overrides [PROTOCOL] §12)

- Diversity gate passed for all 3 candidates before arena entry (no pair above the diversity threshold).
- Every candidate shows ≥1 attack received AND ≥1 mutation attempt (accepted or rejected) logged in the arena log — the premature-convergence guard's minimum bar.
- Synthesized idea's provenance table is complete: 100% of its elements trace to an accepted arena entry.
- Coherence check passed with zero unresolved incompatible grafts (or an explicit, labeled fallback-to-single-candidate verdict).
- Rejected-idea register non-empty whenever the arena ran ≥2 rounds (a losing candidate that produced literally nothing worth preserving is itself a finding, not silence).
- Handoff packet carries all four required fields: scope, non-goals, acceptance criteria, provenance summary.
- LOOP-11 inline verdict on the packet = PASS; final confidence ≥ 0.85; governance row written.

# Failure Routing (only deviations from [PROTOCOL] §4)

- Diversity gate fails the same pair twice → Logic Failure escalates to reasoning tier with a harder-forced angle constraint (§ node 5), not an indefinite retry loop.
- Arena round-budget exhausted with a neglected candidate (zero attacks/mutations) → treated as a Logic Failure requiring one forced extra round targeting that candidate, never allowed to silently close as PASS.
- Coherence check finds an unresolvable incompatible graft after 2 resynthesis cycles → Architecture Failure; the loop does not halt — it degrades to the labeled single-candidate fallback (§ node 8) and surfaces the degradation explicitly in the handoff packet and AUDIT-ARTIFACT, never presenting a patched-over Frankenstein idea as a clean synthesis.
- LOOP-11 REJECT on the handoff packet → resets to node 7 (resynthesis) with the critique as new context, per [PROTOCOL] §1 step 4 — never node 1.

# Approval Gates (only deviations from [PROTOCOL] §10)

- Nodes 1–8 (generation, arena, synthesis, coherence) are read/report work — automatic, per [PROTOCOL] §10's "Read / audit / report" class.
- **Node 9's dispatch of the handoff packet to an execution agent requires explicit operator review before dispatch**, even though LOOP-19 itself never mutates code — the packet is about to hand write authority to a downstream agent, so the human sees scope, non-goals, and provenance before that agent starts building. This is stricter than the default automatic class for read-only artifacts specifically because of what the artifact triggers next.

# RUN PROMPT

```
Run LOOP-19 on <seed request> (candidates: 3, execution-target: <who receives the handoff>).
Follow LOOP-19-idea-arena.md under [PROTOCOL] (see PROTOCOL.md in this repo).
Force genuine diversity across the 3 generated ideas (distinct angle/risk/cost profile
per candidate) — reject near-duplicates before they enter the arena. Run one shared
arena (all 3 candidates co-present, not pairwise elimination): every candidate must
show at least one attack received and one mutation attempt logged. Converge to a
single synthesized idea with a complete provenance trail; keep unused strengths from
losing candidates in a rejected-idea register, never silently drop them. Run the
coherence check before handoff — degrade to a labeled single-candidate fallback rather
than ship an incoherent graft. Package the result as a spec-level HANDOFF-PACKET
(scope, non-goals, acceptance criteria, provenance) and run the LOOP-11 inline
adversarial check over it. Read-only to any codebase throughout. Hand me the packet
for review before it goes to the execution agent — do not dispatch it yourself.
```
