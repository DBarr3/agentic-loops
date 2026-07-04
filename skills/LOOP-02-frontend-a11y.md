---
name: loop-frontend-a11y
loop-id: LOOP-02
description: Component-tree + bundle audit; Playwright accessibility-tree interrogation (roles, focus traps, dynamic ARIA, light/dark contrast)
domain: Frontend & Accessibility
risk-class: branch-mutating
default-debate: FREE-MAD
model-tiers: {scan: cheap, audit: mid, verdict: reasoning}
---

# Mission

Treat the frontend as a state machine, not a visual layer. This loop parses the component tree for rendering defects (unnecessary re-renders, missing memoization, broken effect dependency arrays, infinite render loops), plans the separation of state logic from presentation, audits the bundler for lazy-loading and dependency hygiene, and then performs the audit static scanners cannot: a live Playwright interrogation of the browser's Accessibility Tree. Automated scanners (Lighthouse, stock Axe passes) miss 60–70% of real-world WCAG violations because they check attribute presence, not behavior — this loop interacts with the running UI to verify roles, dynamic ARIA state, keyboard focus trapping, and contrast in both themes. Mutations land on a loop branch; every bug found generates a permanent regression test (hand-off to LOOP-12).

# Trigger (when the operator runs this)

- Weekly sweep cadence, or whatever cadence your team's audit schedule defines.
- After any significant component refactor, new route, design-system change, or dependency bump.
- On demand: `Run LOOP-02 on <repo> (scope: <frontend-dir>)`.

# Inputs (target repo/dir, scope flags)

- `target`: repo root of the repository being audited (e.g., the monorepo root, or a specific frontend package root).
- `scope`: frontend directories to audit (e.g. `src/`, `app/`, `client/`, `packages/web/` — wherever your components, routes, and hooks live).
- `--a11y-only`: run only nodes 5–8 (Playwright battery) against an already-running dev server.
- `--no-mutate`: findings only, no fixes.
- `dev-server`: how to launch the app for the Playwright nodes (script name or URL of a running instance).

# Preconditions

- Loop branch `loop/LOOP-02-<YYYY-MM-DD>` created from origin/main before any mutation.
- Code-graph/knowledge-graph tool reachable, if your project has one (MCP server, IDE semantic index) — with component modules identified in its index or docs. If you don't have one, ast-grep alone covers node 1.
- ast-grep available; Playwright installed and able to launch a browser (`Bash: npx playwright --version`).
- Dev server buildable locally — confirm required local env vars/config are present first; a common gotcha is a missing `.env.local` (or equivalent) causing the app to silently boot against placeholder backend config (e.g., a stub Supabase or Firebase project ref) instead of real values.
- Node/npm toolchain present for bundle analysis (`npm run build` or equivalent must succeed on the base commit before the loop mutates anything — establish the baseline bundle size first).

# Execution DAG (numbered nodes, [P] = parallelizable)

1. Component-graph construction — code-graph tool (if available) + ast-grep map of components, hooks, and state flow.
2. [P] Render-efficiency audit — re-renders, memoization, effect dependency arrays, infinite loops.
3. [P] State/presentation separation — refactor plan for entangled logic.
4. [P] Bundle & dependency audit — lazy loading, TTI, deprecated/unused packages, size drift.
5. Launch app + Playwright accessibility-tree snapshot (depends on 1; blocks 6–8).
6. [P] Semantic-role and dynamic-ARIA interaction battery.
7. [P] Keyboard-navigation and focus-trap battery.
8. [P] Theme battery — light AND dark mode contrast + focus visibility.
9. Fix application on loop branch (consumes 2–8 findings, severity-ordered).
10. Regression-test generation (unit + Playwright a11y tests) + full verification run.

Nodes 2–4 parallelize after node 1. Node 5 gates the Playwright batteries 6–8, which then run in parallel against the same app instance.

# Node Specs

**Node 1 — Component-graph construction.**
Action: if your project has a code-graph/knowledge-graph tool, query it for the modules covering the scope and review any associated docs; then use ast-grep to enumerate components, custom hooks, context providers, and state containers, producing a component table: `component | file | hooks used | context deps | children count | route`.
Tools: code-graph/knowledge-graph tool (MCP server, IDE semantic index — if available), ast-grep, Read (targeted). Grep only for literal class names/test ids.
Failure conditions: code-graph tool unreachable, if in use (Tool Failure — retry once, then degrade to ast-grep-only enumeration); framework undetectable (Dependency Failure — resolve from package.json, retry).
Output artifact: `_loopstate/LOOP-02/<run-id>/node-1-component-graph.json` + confidence block.

**Node 2 — Render-efficiency audit [P].**
Action: with ast-grep patterns over the node-1 table, detect: (a) expensive computations inside render bodies lacking memoization (`useMemo`/`useCallback`/`React.memo` or framework equivalent), (b) effect hooks with missing, over-broad, or object-identity-unstable dependency arrays, (c) state updates inside effects that re-trigger the same effect (infinite render loop candidates), (d) props drilling of new object/array/lambda literals into memoized children (defeats memoization), (e) blocking synchronous work on the render path. Rank by component fan-out from the graph (god components first).
Tools: ast-grep, code-graph tool (fan-out ranking, if available), Read.
Failure conditions: >150 raw matches (Context Overflow — compress to per-component summaries); loop-candidate undecidable statically → mark for runtime confirmation in node 5 (React DevTools profile or console warning check), not guessed.
Output artifact: `node-2-render-findings.md` — `id | severity | component | file:line | defect class | fix sketch`.

**Node 3 — State/presentation separation plan [P].**
Action: identify components where business logic (fetching, derivation, validation, side effects) is entangled with presentation markup. For each, propose a concrete refactor: extract logic into a custom hook or isolated state container, leave the component as a pure presenter. Output is a PLAN with per-component effort/risk scores; only low-risk extractions (no behavioral branching, covered by existing tests) are executed in node 9 — the rest are deliverables for the operator.
Tools: ast-grep, code-graph tool (if available), Read.
Failure conditions: none halting; ambiguous ownership goes to the confidence block's `unknown` list.
Output artifact: `node-3-separation-plan.md`.

**Node 4 — Bundle & dependency audit [P].**
Action: (a) verify route-level code splitting — every route and heavy third-party dependency (charting, editor, animation libs) is lazy-loaded rather than in the initial chunk blocking Time to Interactive; (b) run the production build and record chunk sizes against the pre-loop baseline — flag drift; (c) audit `package.json` + lockfile for deprecated libraries, known-vulnerable versions, and unused packages (depcheck/knip class tooling via Bash), producing a prune list to shrink attack surface and memory footprint.
Tools: Bash (build, `npx depcheck`/`knip`, bundle analyzer), Read (package.json/lockfile), ast-grep (dynamic-import verification).
Failure conditions: baseline build fails (Dependency Failure — fix build first or HALT with finding); analyzer unavailable (Tool Failure — degrade to raw chunk-size comparison from build output).
Output artifact: `node-4-bundle.md` — chunk table, drift %, prune list, lazy-load gaps.

**Node 5 — App launch + accessibility-tree snapshot.**
Action: start the dev server (or attach to `dev-server` URL); for each key route, capture the Playwright accessibility snapshot (`page.accessibility.snapshot()` / aria snapshot) and an interactive-element inventory. This is the ground truth the batteries in 6–8 interrogate. Also confirm any node-2 infinite-loop candidates via console warnings/profiling here.
Tools: Playwright (via Bash), Bash.
Failure conditions: app fails to boot (Dependency Failure — HALT, emit finding; the Playwright battery cannot be skipped silently); route inventory empty (Logic Failure — re-derive routes from node 1).
Output artifact: `node-5-a11y-snapshots/` (one snapshot per route) + inventory table.

**Node 6 — Semantic-role and dynamic-ARIA battery [P].**
Action: static scanners confirm an attribute exists; this node confirms it behaves. (a) Every interactive element in the snapshot must expose a valid semantic role and accessible name — flag and queue refactors for generic `<div>`/`<span>` click targets that lack native keyboard support (div-buttons). (b) Drive dynamic states through Playwright: open every modal, expand every accordion/dropdown/disclosure, and assert that `aria-expanded`, `aria-hidden`, `aria-selected` and equivalents actually update in the accessibility tree at each transition — an aria-label that never changes state is a violation even though a scanner passes it.
Tools: Playwright, node-5 snapshots, ast-grep (to locate the offending component source for each finding).
Failure conditions: flaky interaction (Tool Failure — retry once with waits, then record as `unknown`, never as pass).
Output artifact: `node-6-roles-aria.md` — `element | route | violation | WCAG criterion | source file:line`.

**Node 7 — Keyboard-navigation and focus-trap battery [P].**
Action: simulate keyboard-only usage per route: Tab/Shift+Tab through the full tab order and record it; open each dialog and verify focus moves into it, is trapped inside while open, Escape closes it, and focus returns to the invoking element; verify no keyboard dead-ends (focusable but inescapable regions) and no focus loss to `<body>` after dynamic content changes.
Tools: Playwright (keyboard API), node-5 inventory.
Failure conditions: as node 6.
Output artifact: `node-7-keyboard.md` — tab-order maps + trap/escape matrix per dialog.

**Node 8 — Theme battery: light AND dark contrast + focus states [P].**
Action: emulate `prefers-color-scheme` light and dark (plus any in-app theme toggle). In BOTH themes, for every text element and every interactive element in default/hover/focus/disabled states, compute contrast ratios against WCAG AA (4.5:1 normal text, 3:1 large text and UI components) and verify focus indicators remain visible. A single-theme snapshot is never sufficient evidence.
Tools: Playwright (colorScheme emulation, computed styles), axe-core run per theme as a floor (not a ceiling).
Failure conditions: theme toggle not discoverable (record as `unknown` + finding: theme not testable).
Output artifact: `node-8-contrast.md` — per-theme violation table with measured ratios.

**Node 9 — Fix application (loop branch only).**
Action: apply fixes severity-ordered, one commit per finding (`loop(LOOP-02): <node> — <change>` + confidence score in body): memoization and dependency-array corrections, div-button → semantic `<button>` refactors, ARIA state wiring, focus-trap implementations, lazy-load conversions, safe package prunes, low-risk state extractions from node 3. Re-run the production build after bundle-affecting fixes to confirm size improvement, not regression. Run a secrets/credential scan on the diff before every commit — no keys, tokens, or credentials committed.
Tools: Edit, Bash (build/tests/git), Playwright (spot re-verification of each a11y fix).
Failure conditions: any fix regresses tests or re-introduces a battery failure (rollback that commit per protocol §6); 3 retries → operator-deferred.
Output artifact: commit list + `node-9-fixes.md`.

**Node 10 — Regression tests + verification.**
Action: every fixed bug gets a permanent test — unit tests for render/logic fixes, Playwright a11y specs for battery findings (role assertions, focus-trap specs, per-theme contrast checks) so violations cannot silently return. Re-run nodes 6–8 batteries end-to-end on the fixed branch. Emit LOOP-12 hand-off manifest.
Tools: Write (tests), Bash, Playwright.
Failure conditions: a regression test cannot reproduce the original violation (Logic Failure — rewrite the test).
Output artifact: `node-10-tests.md` + LOOP-12 manifest + final `AUDIT-ARTIFACT.md` per protocol §13.

# Adversarial Check (reviewer persona + what it attacks)

Persona: **hostile accessibility auditor with a screen reader and a stopwatch**, running FREE-MAD. It attacks:

1. Every a11y "pass" that rests on attribute presence rather than an interaction transcript — demands the Playwright evidence of the state change.
2. The confidence block's `unknown` lists first, especially untestable themes and flaky interactions recorded as unknown.
3. Node-9 memoization fixes — hunts for stale-closure bugs and correctness regressions introduced by over-memoization.
4. The bundle prune list — verifies no "unused" package is actually consumed via dynamic import, CSS side effects, or a build plugin.
5. Dark-mode evidence — rejects any contrast verdict backed by a light-mode-only measurement.
6. Silent Agreement in round 1 without cited element/route evidence forces a second round (protocol §11).

Maximum 2 debate rounds; surviving disagreements become operator-deferred findings.

# Exit Criteria (quantitative, overrides protocol §12)

- WCAG 2.1 AA: zero unresolved violations from nodes 6–8 batteries (serious/critical axe impact and all interaction-battery failures fixed or operator-deferred with rationale).
- Zero div-buttons remaining on interactive paths in scope; 100% of dialogs pass trap+escape+focus-return.
- Contrast: 100% of measured elements ≥ 4.5:1 (normal text) / 3:1 (large text, UI components) in BOTH light and dark themes.
- Bundle: initial-chunk size growth ≤ +2% vs pre-loop baseline (reductions expected); all routes and flagged heavy deps lazy-loaded; zero deprecated packages with known vulnerabilities remaining.
- Zero confirmed infinite render loops; all CRITICAL/HIGH render findings fixed.
- Test pass rate ≥ 99% on touched surface; every fixed bug has a named regression test; retry ≤ 3/node; final confidence ≥ 0.85; governance row written.

# Failure Routing (only deviations from protocol §4)

- App-boot failure in node 5 is a first-class finding, not a skip: the loop terminates FAIL-with-artifact if the Playwright battery cannot run, because static results alone cannot certify this domain.
- Flaky Playwright interactions route to `unknown` after one retry — never to PASS.

# Approval Gates (only deviations from protocol §10)

- Protocol defaults apply. Additionally: node-3 state-extraction refactors beyond the low-risk set, and any package removal that changes the lockfile, are listed for operator approval before commit.

# RUN PROMPT

```
Run LOOP-02 on <repo> (scope: <frontend-dir>). Follow LOOP-02-frontend-a11y.md under PROTOCOL.md in this repo. Static scanners miss 60-70% of WCAG violations — you must run the live Playwright accessibility-tree batteries (roles, dynamic ARIA, focus traps, light+dark contrast), plus the render/bundle audit. Mutate only on loop/LOOP-02-<date>, regression test every bug, and deliver AUDIT-ARTIFACT + loop branch. No merge without my approval.
```
