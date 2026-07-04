---
name: loop-ux-ui-visual
loop-id: LOOP-06
description: Visual regression, stateful UI states, graceful degradation pathways, memory-transparency panel presence, heuristic UX evaluation
domain: UX/UI & Human Factors
risk-class: branch-mutating
default-debate: FREE-MAD
model-tiers: {scan: cheap, audit: mid, verdict: reasoning}
---

# Mission

Bridge the gap between static code validation and genuine user experience. Backend logic can be proven by tests; whether the product feels stable, honest, and humane cannot — it must be observed. This loop runs disciplined visual regression testing (signal, not sub-pixel noise), exercises every stateful UI condition (hover/focus/disabled/active, loading skeletons, both themes), audits behavioral heuristics under failure — error pathways and network degradation must produce coherent messages, retry mechanisms, and preserved user input, never crashes or cleared forms — and verifies the trust-critical memory-transparency surface: a dedicated, visible UI panel where users can view, edit, and delete what the AI remembers about them. An application that remembers invisibly reads as surveillance; this loop treats the absence of that panel as a HIGH finding, not a nice-to-have. Mutations land on a loop branch; every bug found generates a permanent regression test (hand-off to LOOP-12).

# Trigger (when the operator runs this)

- Pre-release cadence — run LOOP-08, LOOP-12, LOOP-06 before releases (see PROTOCOL.md for suggested sequencing).
- After any design-system change, CSS refactor, breakpoint work, or theme change.
- After any change to AI memory features or their UI surface.
- On demand: `Run LOOP-06 on <repo> (scope: web/)`.

# Inputs (target repo/dir, scope flags)

- `target`: repo root (a web app, marketing site, or desktop app renderer are typical targets).
- `scope`: UI directories + the route list to audit (default: all user-facing routes).
- `baseline`: path to the approved visual-baseline set (`_loopstate/LOOP-06/baselines/<scope>/`); first run on a scope creates baselines and terminates as BASELINE-ESTABLISHED, not PASS.
- `--viewports`: default `375x812, 768x1024, 1440x900` (mobile / tablet / desktop).
- `--no-mutate`: findings only.

# Preconditions

- Loop branch `loop/LOOP-06-<YYYY-MM-DD>` created from origin/main before any mutation.
- Playwright installed with screenshot comparison available; app bootable locally (dev server script or URL provided).
- A code-graph or knowledge-graph tool, if your project has one (MCP server, IDE semantic index — otherwise ast-grep/Semgrep/LSP), reachable to map findings back to owning components/modules.
- Deterministic rendering prepared: animations frozen (`prefers-reduced-motion`), clock/network fixtures for dynamic content where feasible — flaky pixels are the enemy of this loop.

# Execution DAG (numbered nodes, [P] = parallelizable)

1. Route + component inventory and baseline load (code-graph map, boot app, verify determinism).
2. Visual regression sweep across viewports and both themes — filtered diffing against baselines.
3. [P] Stateful UI battery — hover/focus/disabled/active, loading skeletons, async feedback.
4. [P] Error-pathway and network-degradation battery — graceful failure heuristics.
5. [P] Memory-transparency panel audit — view/edit/delete surface for AI memory.
6. Heuristic UX evaluation — synthesis pass over 2–5 evidence with severity scoring.
7. Fix application on loop branch (severity-ordered).
8. Regression-test generation (visual + behavioral specs) + verification re-run.

Node 2 must complete before 3–5 (it establishes the healthy-state visual ground truth those batteries mutate away from). Nodes 3, 4, 5 run in parallel against the same app instance. Node 6 synthesizes; 7–8 close the loop.

# Node Specs

**Node 1 — Inventory, boot, baseline load.**
Action: derive the route/screen list from the router config (ast-grep) and your code-graph tool's module/component map, if available; boot the app; verify deterministic rendering (two consecutive screenshots of a static route must be pixel-identical — if not, identify and freeze the noise source before proceeding). Load or initialize the visual baseline set.
Tools: code-graph/knowledge-graph tool (if available), ast-grep, Playwright, Bash, Read (router config).
Failure conditions: app fails to boot (Dependency Failure — HALT, finding emitted; this loop cannot run headless-static); determinism unachievable on a route (mark route `unknown`, exclude from pixel diffing, keep it in the behavioral batteries).
Output artifact: `_loopstate/LOOP-06/<run-id>/node-1-inventory.json` (routes × viewports × themes matrix) + confidence block.

**Node 2 — Visual regression sweep (filtered).**
Action: capture every route at every viewport in light AND dark themes; diff against baselines with a noise filter that ignores sub-pixel rendering anomalies and OS font-variation deltas (anti-aliasing tolerance, small per-pixel color threshold), and focuses strictly on critical regressions: component occlusion (elements overlapping/hidden), unintended layout shifts, missing responsive breakpoints (content overflow, horizontal scroll at declared viewports), and CSS breakage (unstyled flashes, collapsed containers, broken grids). Every diff above threshold is classified into one of those four classes or explicitly dismissed as accepted-noise with justification.
Tools: Playwright screenshot compare, Bash, node-1 matrix.
Failure conditions: >30% of routes diffing (Logic Failure — suspect environmental cause, e.g. font load or theme default, before filing 50 findings); baseline missing for a route → capture and mark BASELINE-NEW.
Output artifact: `node-2-vrt.md` — `route | viewport | theme | diff class | screenshot pair | verdict` + updated-baseline proposals (baselines only update on operator approval).

**Node 3 — Stateful UI battery [P].**
Action: for every interactive element in the inventory, drive and screenshot each dynamic state: hover, focus, disabled, active/pressed. Verify (a) visual feedback is immediate and distinguishable per state, (b) every asynchronous operation renders a loading skeleton or explicit progress affordance — a frozen UI during a fetch is a finding, (c) color contrast of each state meets WCAG AA (4.5:1 text, 3:1 UI components) in BOTH light and dark themes — state styling that passes in light and vanishes in dark is a classic failure this node exists to catch.
Tools: Playwright (state forcing, colorScheme emulation, computed styles), node-1 inventory.
Failure conditions: state not reachable programmatically (record `unknown`, never PASS); flaky state render → one retry then `unknown`.
Output artifact: `node-3-states.md` — element × state × theme matrix with violations.

**Node 4 — Error-pathway & network-degradation battery [P].**
Action: using Playwright network interception, simulate per key user flow: hard API failure (500), timeout, offline, and slow network (throttled). For each degraded condition verify the UI degrades gracefully: (a) a coherent, user-friendly error message appears — no raw exception text, no stack traces, no silent nothing-happens; (b) a retry mechanism is offered where the operation is retryable; (c) user input is preserved — forms must NOT clear, in-progress compositions must survive the failure and the retry; (d) the app never white-screens or crashes. Exercise the highest-value flows first (auth, checkout/billing, primary creation flows).
Tools: Playwright (route interception, offline emulation, throttling), node-1 inventory, ast-grep (locating the owning error-boundary/handler for each finding).
Failure conditions: any crash or form-wipe = HIGH minimum, CRITICAL on auth/payment flows; flow not automatable → `unknown` with manual-test note.
Output artifact: `node-4-degradation.md` — `flow | condition | message coherent? | retry? | input preserved? | crash? | evidence`.

**Node 5 — Memory-transparency panel audit [P].**
Action: verify the application exposes a dedicated, highly visible UI surface (e.g. "Memory Overview" / "Custom Instructions" panel) where end-users can transparently interact with what the AI system remembers about them. Audit four requirements: (1) discoverability — reachable from primary settings/navigation within two interactions, not buried; (2) view — remembered items are legible, human-readable, and attributable (what is stored, when it was learned); (3) edit — users can correct or amend a remembered item; (4) delete — users can remove individual memories AND clear all, and deletion verifiably propagates (the UI reflects removal; pair with LOOP-05 for backend erasure verification). Absence of the panel entirely = HIGH finding (trust/anti-surveillance requirement); a view-only panel without edit/delete = MEDIUM. Where the target app has no AI memory features, record N/A with evidence, not silence.
Tools: Playwright (navigation + interaction), code-graph tool (locate memory-feature modules, if available), ast-grep, Read.
Failure conditions: panel exists but delete action errors (Security-adjacent finding — escalate severity to HIGH and flag for LOOP-05 follow-up).
Output artifact: `node-5-memory-ux.md` — four-requirement scorecard + navigation-path evidence.

**Node 6 — Heuristic UX evaluation (synthesis).**
Action: reasoning-tier pass over the combined evidence of nodes 2–5, scoring each user flow against usability heuristics: visibility of system status (skeletons, progress, confirmations), error prevention and recovery (node-4 evidence), consistency (node-2/3 cross-route comparisons — same control, same look, same behavior), and user control/trust (node-5 evidence). Produces the unified findings table with severities and a per-flow UX verdict; anything scored from a single screenshot without interaction evidence must say so.
Tools: reasoning-tier model over prior artifacts; no new browsing (evidence must already exist — this node synthesizes, it does not collect).
Failure conditions: evidence gaps discovered here route BACK to the owning battery node (workflow recovery per PROTOCOL.md §7), never patched by assumption.
Output artifact: `node-6-heuristics.md` — unified severity-ranked findings table.

**Node 7 — Fix application (loop branch only).**
Action: severity-ordered fixes, one commit per finding (`loop(LOOP-06): <node> — <change>` + confidence block in body): CSS/breakpoint repairs, state-style corrections in both themes, skeleton insertion for async ops, error-boundary and retry implementations, form-state preservation (controlled persistence across failure), memory-panel gaps (in coordination with LOOP-05 if backend surface is missing). Visual fixes are re-screenshotted against intent immediately. Run a secrets/credential scan before every commit; never write into vault, credentials, or other protected state directories reserved for other tooling in the repo.
Tools: Edit, Playwright (per-fix visual verification), Bash (build/tests/git).
Failure conditions: a fix moves pixels on unrelated routes (regression — rollback that commit per PROTOCOL.md §6, re-scope); 3 retries → operator-deferred.
Output artifact: commit list + `node-7-fixes.md`.

**Node 8 — Regression tests + verification.**
Action: every fixed bug becomes a permanent spec: visual regression baselines updated ONLY for intentional fixes (operator-approved), Playwright specs asserting state styles per theme, degradation specs asserting message/retry/input-preservation per flow, and a memory-panel spec asserting view/edit/delete affordances exist and function. Re-run nodes 2–5 batteries on the fixed branch. Emit LOOP-12 hand-off manifest.
Tools: Write (tests/specs), Playwright, Bash.
Failure conditions: a regression spec cannot reproduce the original defect (Logic Failure — rewrite the spec).
Output artifact: `node-8-tests.md` + LOOP-12 manifest + final `AUDIT-ARTIFACT.md` per PROTOCOL.md §13.

# Adversarial Check (reviewer persona + what it attacks)

Persona: **ruthless UX researcher who has watched a thousand users rage-quit**, running FREE-MAD. It attacks:

1. Every "accepted-noise" dismissal in node 2 — demands the justification and re-inspects the screenshot pair; noise-filtering is the easiest place for this loop to hide real regressions.
2. Node-4 "graceful" verdicts — replays the evidence asking: would a non-technical user understand this message? Is the retry actually wired, or decorative? Was the form input REALLY still there after retry?
3. Node-5 scoring — attempts to argue the panel is not discoverable in two interactions, and that "delete" is cosmetic (UI removal without propagation).
4. Confidence block `unknown` lists first (PROTOCOL.md §3): unreachable states, non-automatable flows, non-deterministic routes.
5. Theme parity — rejects any state-styling verdict backed by light-mode evidence only.
6. Silent Agreement without screenshot/transcript citations in round 1 forces a hostile second round (PROTOCOL.md §11).

Maximum 2 debate rounds; survivors become operator-deferred findings.

# Exit Criteria (quantitative, overrides protocol §12)

- Zero unresolved CRITICAL/HIGH visual regressions (occlusion, layout shift, missing breakpoint, CSS breakage) across all declared viewports in both themes.
- Stateful battery: 100% of reachable interactive elements show distinguishable hover/focus/disabled/active feedback; 100% of async operations have loading affordances; all state contrast ≥ WCAG AA (4.5:1 / 3:1) in BOTH themes.
- Degradation battery: zero crashes and zero form-wipes across all tested flows; 100% of failed flows show a coherent message; 100% of retryable operations offer retry with input preserved.
- Memory transparency: panel present, discoverable ≤2 interactions, with functioning view + edit + delete (or documented N/A) — no unresolved HIGH on this surface.
- Baseline hygiene: every baseline update operator-approved; zero unexplained diffs remaining.
- Test pass rate ≥ 99% on touched surface; every fixed bug has a named regression spec; retry ≤ 3/node; final confidence ≥ 0.85; governance row written.

# Failure Routing (only deviations from protocol §4)

- Non-deterministic route rendering routes to `unknown` + exclusion from pixel diffing, not to loop failure — but more than 25% of routes excluded forces FAIL-with-artifact (the audit surface is too degraded to certify).
- Evidence gaps found in node 6 re-open the owning battery node (resume per PROTOCOL.md §7), never restart node 1.

# Approval Gates (only deviations from protocol §10)

- Protocol defaults apply. Additionally: visual-baseline updates are an explicit operator approval item (approving a bad baseline poisons every future run), and any UX copy rewrite in error messages is presented to the operator as before/after pairs before commit.

# RUN PROMPT

```
Run LOOP-06 on <repo> (scope: web/). Follow LOOP-06-ux-ui-visual.md under PROTOCOL.md in this repo. Run the filtered visual-regression sweep (both themes, all viewports), the stateful UI battery, the error/network-degradation battery (no crash, no cleared forms, coherent message + retry), and the memory-transparency panel audit (view/edit/delete). Mutate only on loop/LOOP-06-<date>, regression spec for every bug, and deliver AUDIT-ARTIFACT + loop branch. No merge without my approval.
```
