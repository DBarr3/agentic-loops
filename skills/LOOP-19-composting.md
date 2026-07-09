---
name: loop-composting
loop-id: LOOP-19
description: Consolidation/dedup loop for any pile of near-duplicate artifacts — clusters raw noisy items, distills a canonical component set, decides group-vs-recurse, and stamps a canonical-content signature so future near-duplicates are matched regardless of surface wording
domain: Knowledge & Finding Consolidation
risk-class: read-only→branch-mutating (mutates only a findings/memory store, never application code)
default-debate: FREE-MAD
model-tiers: {scan: cheap, audit: mid, verdict: reasoning}
---

# Mission

Any system that accumulates artifacts over repeated runs — audit findings across sweeps, memory/log entries, code patterns flagged by more than one loop, ideas from repeated brainstorm sessions, error/log clusters — eventually piles up near-duplicates: the same underlying issue reported 20 different ways because surface wording, ordering, or formatting differs each time it's rediscovered. Left alone, this either floods the store with noise or, worse, lets a "loop-until-dry" discovery process treat every rephrased duplicate as new work forever, so it never converges.

This loop formalizes **composting**: turning a batch of noisy raw variants into clean, reusable, matchable material. Four stages, always in this order:

1. **Raw intake** — collect a batch of items suspected to be near-duplicates. Nothing arrives pre-grouped; this loop runs its own similarity/clustering pass to decide which raw items even belong in the same compost batch.
2. **Compost (distill to canonical component set)** — extract the minimal set of true underlying components/claims from the batch, discarding surface noise (wording, ordering, formatting) while preserving every distinct real component. Composting is lossy by design; what gets discarded and why is recorded, never silently dropped.
3. **Proposals (group-or-recurse)** — decide, per distilled set, whether it is ready to file as one consolidated entry or still contains sub-clusters that need another composting pass first.
4. **Signature** — compute a content fingerprint from the CANONICAL component set only, never from the raw noisy input, so any future incoming item can be composted and its signature checked against the existing index to detect "this is the same thing again" even when its raw wording looks nothing like the earlier raw wording.

The payoff: this is what keeps an audit/finding/memory system from re-reporting the same root cause forever under different phrasings, and what lets any "run loops until nothing new turns up" discovery process actually terminate instead of mistaking every rephrasing for new work.

Three failure modes this loop is explicitly designed against:
- **False-merge** — two genuinely different issues composted into one, silently hiding one of them.
- **Signature collision** — two different canonical sets producing the same fingerprint.
- **Signature drift** — the canonicalization method changes over time so the same underlying issue gets a different signature on a later run, breaking match continuity.

# Trigger (when the operator runs this)

- Whenever a findings/memory/log store shows suspected duplicate buildup (recurring near-identical entries across runs).
- Routinely after any batch of loop runs that append to a shared findings or memory store (e.g. after several LOOP-13-class sweeps, or after a run of repeated brainstorm/ideation sessions).
- Before closing out a "run until dry" discovery loop, to confirm remaining "new" items are actually new and not rephrasings of composted ones.
- On demand: `Run LOOP-19 on <findings/memory store>`.

# Inputs (target repo/dir, scope flags)

- `target`: a findings store, memory log, or artifact directory containing accumulated raw items (default: your project's primary findings/memory store).
- `scope`: optional filter (date range, source loop/tool, tag) to bound the intake batch.
- `signature_index`: path to the existing canonical-signature index (created on first run if absent).
- Flags: `--batch-size=N` (cap raw items per compost pass), `--recursion-depth-max=N` (default 3), `--no-mutate` (report proposed compost/signatures only, write nothing).

# Preconditions

- [PROTOCOL] read.
- Loop branch `loop/LOOP-19-<date>` created before any write to the findings/memory store; this loop never touches application source code — that is out of scope by construction.
- A similarity/embedding tool or structural-diff tool available for clustering raw items (falls back to normalized string/edit-distance similarity if none is configured — flag this in the confidence block as a lower-precision mode).
- `signature_index` readable, or created fresh and labeled first-run.
- The canonicalization method (how raw items resolve to canonical components) is versioned; the current version string is known before node 2 runs.

# Execution DAG (numbered nodes, [P] = parallelizable)

1. Raw intake & similarity clustering
2. Compost — canonical component distillation + discard ledger (per cluster)
3. [P] Discard ledger audit
4. [P] Proposals — group-or-recurse decision
5. Recursion / convergence gate
6. Signature computation (canonical set only, versioned scheme)
7. Signature collision check against index
8. Adversarial verification + AUDIT-ARTIFACT

Order: 1 → 2 → (3 ‖ 4) → 5 → 6 → 7 → 8. Node 5 may loop back to node 2 for any sub-cluster flagged "needs another pass," bounded by `--recursion-depth-max`.

# Node Specs

### Node 1 — Raw intake & similarity clustering
- **Action:** pull the raw item batch from `target` under `scope`. Run a similarity pass (embedding distance, structural-shape comparison, or normalized-string distance as fallback) to group raw items into candidate compost batches. Do not assume any pre-existing grouping — items may arrive in arbitrary order with scrambled/reworded surface form.
- **Tools:** an embedding/similarity tool if available, structural-diff tooling, Read on the store, cheap tier for the first clustering pass.
- **Failure conditions:** no similarity signal above threshold for an item → route it to its own singleton batch, never force it into a cluster; batch size exceeds `--batch-size` → split by sub-similarity, re-scope (Context Overflow per PROTOCOL §4).
- **Output artifact:** `node-1-clusters.json` (cluster id → member raw items + pairwise similarity scores) with confidence block.

### Node 2 — Compost: canonical component distillation + discard ledger
- **Action:** for each cluster, extract the minimal set of distinct true underlying components/claims. Preserve every component that is genuinely distinct; merge only surface variation (wording, ordering, formatting) of the same component. For every piece of raw content NOT carried into the canonical set, log it in a discard ledger with the specific reason (e.g. "reworded duplicate of component X," "formatting-only variant," "subsumed by component Y").
- **Tools:** mid tier for component extraction, Read on raw items, structural comparison to justify each merge decision with cited evidence (not "looks similar").
- **Failure conditions:** a raw item cannot be confidently mapped to an existing component or a new one → keep it as its own component rather than guessing (never silently drop ambiguous content); discard reason cannot be stated concretely → do not discard, carry forward instead.
- **Output artifact:** `node-2-canonical-sets.md` (cluster id → canonical component list) + `node-2-discard-ledger.md` (raw item → discard reason, per cluster).

### Node 3 — [P] Discard ledger audit
- **Action:** independently re-check every discard-ledger entry from node 2: does the cited reason hold (is the discarded item truly redundant with a retained component, not a distinct claim that got dropped)? Flag any discard lacking a concrete justification as a potential under-compost failure (true content lost).
- **Tools:** mid tier, Read on node-2 outputs, no new raw-content extraction.
- **Failure conditions:** any discard without a defensible reason → reinstate the item as its own component, do not let the ledger stand unchallenged.
- **Output artifact:** `node-3-discard-audit.md` (pass/fail per discard entry, reinstated items if any).

### Node 4 — [P] Proposals: group-or-recurse decision
- **Action:** for each canonical component set from node 2, decide: file as one consolidated entry, or does it still contain sub-clusters that need another composting pass? Evidence required for "group": every component pair in the set shares the same underlying root cause/claim, not just topical adjacency. Evidence required for "recurse": at least two components in the set are independently verifiable as distinct issues that got bundled together in this pass.
- **Tools:** mid tier for the first pass, reasoning tier for borderline sets (ambiguous grouping is exactly where false-merges happen).
- **Failure conditions:** cannot produce concrete evidence for "group" → default to "recurse," never default to grouping on ambiguity (grouping is the higher-cost mistake — it hides information); a set that recursed on the immediately prior pass with no change in composition → do not recurse again, escalate to reasoning tier (stability check, see node 5).
- **Output artifact:** `node-4-proposals.md` (set id → verdict: `group` | `recurse`, evidence per verdict).

### Node 5 — Recursion / convergence gate
- **Action:** for every set verdicted `recurse` in node 4, feed its member components back into node 2 as a new sub-batch, incrementing recursion depth. For every set verdicted `group`, pass through to node 6. Track depth per lineage; if a set would exceed `--recursion-depth-max`, or if two consecutive passes over the same set produce an identical component partition (no new splitting or merging), halt recursion on that set and force a `group` verdict with an explicit "recursion-exhausted, human review recommended" flag rather than looping indefinitely.
- **Tools:** Read/Write on `_loopstate/LOOP-19/<run-id>/` lineage tracking, cheap tier bookkeeping.
- **Failure conditions:** depth limit hit with genuine unresolved sub-clustering → emit as operator-deferred finding, never silently force-group without the flag.
- **Output artifact:** `node-5-lineage.md` (set id → recursion depth, convergence status, final partition).

### Node 6 — Signature computation
- **Action:** for every set that reached `group` (via node 4 directly or node 5's convergence), compute a content fingerprint derived strictly from the canonical component set — never from the original raw wording, ordering, or formatting of the intake items. Tag the signature with the canonicalization method's version string so a later change in method is detectable rather than silently producing a different signature for the same underlying issue (signature drift).
- **Tools:** mid tier for canonical-set normalization before hashing, Bash/Write for the fingerprint function.
- **Failure conditions:** canonicalization version unset or stale relative to node 2's actual method → HALT this node, fix the version tag first (a versioning gap is exactly how drift goes undetected).
- **Output artifact:** `node-6-signatures.md` (set id → canonical component list → signature → scheme version).

### Node 7 — Signature collision check
- **Action:** cross-reference every new signature from node 6 against `signature_index`. Exact match on signature with a materially different canonical component set = collision, not a legitimate re-match — flag CRITICAL. Exact match with an equivalent canonical set = legitimate dedup against a prior run; link the new occurrence to the existing entry instead of creating a duplicate.
- **Tools:** Read on `signature_index`, mid tier for component-set equivalence comparison on any signature match.
- **Failure conditions:** any collision (same signature, different canonical content) → HALT the affected set, do not write it to the index; two DIFFERENT signatures that a manual spot-check shows represent the same canonical set → route as signature-scheme Logic Failure, not a data issue (the hash function or normalization is at fault).
- **Output artifact:** `node-7-collision-report.md` (new-vs-index comparison, collisions flagged, legitimate matches linked).

### Node 8 — Adversarial verification + artifact
- **Action:** run the Adversarial Check below over nodes 2–7; on PASS, commit the consolidated entries + discard ledger + signature index update to the loop branch, one commit per resolved set: `loop(LOOP-19): compost — <set id> consolidated`. Assemble `AUDIT-ARTIFACT.md` per PROTOCOL §13 with the discard ledger, recursion lineage, signature collisions, and "Recommended next loops" for anything operator-deferred. Write the governance row.
- **Tools:** LOOP-11 inline (or this loop's own adversarial protocol below), reasoning tier, Edit/Bash/git for the branch commit.
- **Failure conditions:** reviewer rejects a `group` verdict twice → revert that set to `recurse` and re-run from node 2 for it only; unresolved collision after review → HALT that set, require operator tie-break (Approval Gates below).
- **Output artifact:** `AUDIT-ARTIFACT.md` + loop branch commits.

# Adversarial Check

Persona: **the Compost Auditor** — a skeptical archivist who assumes every "these are the same thing" claim is a lazy pattern-match until proven otherwise. Protocol: FREE-MAD (PROTOCOL §11), max 2 rounds. It attacks, in order:

1. **Every `group` verdict from node 4/5** — demands proof the merged components share the same underlying root cause, not just surface topical similarity. "They look similar" is rejected outright; the reviewer requires a cited mechanism, claim, or evidence trail showing they are the same thing.
2. **Every discard-ledger entry** — re-derives whether the discarded content was truly redundant; any discard that reads like content loss dressed up as a formatting note gets reinstated.
3. **Every signature match in node 7** — for legitimate-match links, verifies the linked canonical sets are actually equivalent, not merely signature-identical by coincidence (collision masquerading as match).
4. **Recursion-exhausted flags from node 5** — confirms the stability check fired correctly and isn't hiding a set that would have kept splitting productively.
5. **Silent Agreement** (PROTOCOL §11): if all reviewers concur in round 1 with no cited component-level evidence, force a second hostile round before accepting any `group` verdict.

# Exit Criteria (quantitative, overrides PROTOCOL §12)

- Zero unresolved signature collisions (every collision either explained as a scheme bug and fixed, or escalated to operator tie-break).
- Every discard-ledger entry carries a concrete reason; zero entries audited as "content loss" remain unreinstated.
- Recursion depth bounded per `--recursion-depth-max`; every set that hit the limit is explicitly flagged, never silently force-grouped.
- Zero `group` verdicts standing without cited same-root-cause evidence surviving the Adversarial Check.
- Signature scheme version recorded on every signature in `node-6-signatures.md`; no signature written without a version tag.
- Final confidence ≥ 0.85; governance row written to `_loopstate/governance-ledger.md`.

# Failure Routing (only deviations from PROTOCOL §4)

- A confirmed false-merge (two distinct issues wrongly grouped) is a Logic Failure: un-merge, re-run the affected set from node 2 with the false-merge critique as context — never patch it by editing the consolidated entry in place.
- A signature collision (same fingerprint, different canonical content) escalates like a Security Failure: HALT that set's write, emit a CRITICAL artifact entry, require operator resolution before the set enters the index.
- Recursion exceeding `--recursion-depth-max` without convergence is a Workflow Failure: checkpoint the lineage, escalate the set to reasoning tier once; if still unresolved, mark operator-deferred rather than looping further.

# Approval Gates (only deviations from PROTOCOL §10)

- Writing new or updated entries to `_loopstate/LOOP-19/` and to the findings/memory store's consolidated entries: automatic once the Adversarial Check passes (branch-only, per PROTOCOL §10's "write docs/artifacts" class).
- Any action that supersedes or removes a previously standalone entry in the findings/memory store (because it was merged into a consolidated one) requires adversarial reviewer sign-off before the commit — this is data-continuity-sensitive even though it isn't application code.
- If the loop is configured to prune raw duplicate source items after successful composting (not just the derived entries), that deletion requires explicit operator approval, treated as PROTOCOL §10's "production delete" class.
- Merge to main / opening a PR: operator approval, findings handed back first, per protocol default.

# RUN PROMPT

```
Run LOOP-19 (Composting — Near-Duplicate Consolidation & Canonical
Signatures). Read PROTOCOL.md (this collection's shared protocol), then
LOOP-19-composting.md, and execute its Execution DAG end-to-end under
protocol rules.
target: <findings/memory store or artifact dir> (scope: <filter, optional>)
signature_index: default (existing index, or create fresh and label first-run)
recursion_depth_max: 3 (or your own bound)
This loop mutates ONLY the findings/memory store on loop/LOOP-19-<date> —
never application code. Cluster raw items first (don't assume pre-grouping),
distill to canonical component sets with an explicit discard ledger, make a
group-or-recurse call per set with cited same-root-cause evidence, compute
every signature from the canonical set only (never raw wording), and run the
collision check against the existing index before writing anything. Hand me
the AUDIT-ARTIFACT with the discard ledger, recursion lineage, and any
unresolved collisions. Do not force a `group` verdict without evidence.
```
