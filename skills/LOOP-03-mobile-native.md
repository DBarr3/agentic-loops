---
name: loop-mobile-native
loop-id: LOOP-03
description: Dual-layer (JS/Dart + native) audit — bridge traffic, ANR hunters, list virtualization, manifest hardening, keystore/keychain, pinning
domain: Mobile & Native Platforms
risk-class: branch-mutating
default-debate: FREE-MAD
model-tiers: {scan: cheap, audit: mid, verdict: reasoning}
---

# Mission

Audit a mobile codebase as the dual-layer system it actually is: the application-logic layer (JavaScript/TypeScript for React Native, Dart for Flutter) AND the native OS layer (Java/Kotlin on Android, Swift/Objective-C on iOS). Users judge mobile software by battery drain, heat, memory pressure, frame stutter, and crash behavior — so this loop hunts the causes: heavy work crossing the async bridge, animations running on the JS thread, unbounded listeners and un-cleared timers, unvirtualized lists, oversized bitmaps, and synchronous work blocking Time to Initial Display. It then hardens the native security posture: manifests, secure storage, TLS pinning, deep links, and tamper resilience. Mutations land on a loop branch; every bug leaves a permanent regression test (hand-off to LOOP-12).

# Trigger (when the operator runs this)

- Before any store submission or mobile release candidate.
- After adding a native module, SDK, animation-heavy screen, or any deep-link/URL-scheme change.
- On field reports of ANR, jank, battery complaints, or crash-rate movement.
- On demand: `Run LOOP-03 on <repo> (scope: mobile/)`.

# Inputs (target repo/dir, scope flags)

- `target`: repo root containing the mobile app.
- `scope`: app directories (e.g. `mobile/`, `apps/ios/`, `android/`, `ios/`).
- `--platform`: `android` | `ios` | `both` (default `both`; nodes degrade gracefully if one platform is absent).
- `--security-only`: run only nodes 6–7 (manifest + security table).
- `--no-mutate`: findings only.

# Preconditions

- Loop branch `loop/LOOP-03-<YYYY-MM-DD>` created from origin/main before any mutation.
- A code-graph/knowledge-graph tool for the JS/Dart layer if your project has one (MCP server, IDE semantic index, or similar) — used for architecture/community comprehension; ast-grep available with language grammars for TS/Dart/Kotlin/Java/Swift as applicable (required — it performs the structural enumeration every node relies on).
- Build toolchain for at least one platform present (Gradle / Xcode CLI) so release-variant configuration can be verified, not assumed. If neither toolchain exists, security nodes run in static-only mode and flag it in the confidence block's `unknown` list.
- `AndroidManifest.xml` and/or `Info.plist` locatable in scope.

# Execution DAG (numbered nodes, [P] = parallelizable)

1. Dual-layer graph construction — map JS/Dart modules, native modules, and every bridge crossing.
2. [P] Bridge & animation audit — heavy traffic and JS-thread animations → native-driven.
3. [P] Memory-leak hunt — unbounded listeners, un-cleared timers, large-dataset handling.
4. [P] List virtualization & image downsampling audit.
5. Main-thread / TTID audit — defer synchronous SQLite reads, SDK init, heavy parsing (ANR/frame-stutter hunt).
6. Platform manifest hardening (AndroidManifest.xml / Info.plist).
7. Native security table — secure storage, TLS pinning, deep links, obfuscation, root/jailbreak.
8. Fix application on loop branch (severity-ordered).
9. Regression-test generation + verification (unit + platform-native tests; LOOP-12 hand-off).

Nodes 2–4 parallelize after node 1. Node 5 follows node 2 (it consumes the bridge map). Nodes 6–7 are sequential (7 builds on 6's manifest facts). Node 8 waits for all findings.

# Node Specs

**Node 1 — Dual-layer graph construction.**
Action: if your project has a code-graph/knowledge-graph tool, query it for the modules/communities covering the scope and read any associated architecture notes. Use ast-grep to enumerate: JS/Dart screens and services; native modules (NativeModules/TurboModules registrations, MethodChannels, Swift/Kotlin bridge classes); and every crossing point between the layers. Produce a bridge table: `crossing | JS/Dart side file:line | native side file:line | payload shape | frequency class (per-frame / per-event / one-shot)`.
Tools: code-graph/knowledge-graph tool (if available), ast-grep, Read (targeted). Grep only for literal channel/module names.
Failure conditions: knowledge-graph tool unreachable (Tool Failure — retry once, then continue in ast-grep-only mode and record the reduced-context degradation in the confidence block); no bridge crossings found in a cross-platform app (Logic Failure — re-derive from framework registration conventions).
Output artifact: `_loopstate/LOOP-03/<run-id>/node-1-bridge-graph.json` + confidence block.

**Node 2 — Bridge & animation audit [P].**
Action: for every per-frame or high-frequency crossing in the node-1 table, flag serialization-heavy payloads and chatty round-trips. Detect animations driven on the JS thread (Animated without `useNativeDriver`, JS-timer-driven transforms, setState-per-frame patterns) and mandate native-driven equivalents (native driver, Reanimated worklets, platform animators). Rank by screen visibility and frequency.
Tools: ast-grep (animation API patterns), node-1 bridge table, Read.
Failure conditions: driver eligibility undecidable (layout-affecting properties) → record fix as "restructure animation", not a mechanical flag flip; flag `unknown` in the confidence block if untestable.
Output artifact: `node-2-bridge-anim.md` — `id | severity | screen | file:line | pattern | native-driven fix`.

**Node 3 — Memory-leak hunt [P].**
Action: detect (a) event-listener/subscription registrations (emitters, device sensors, sockets, Firebase, navigation listeners) with no matching removal on unmount/dispose, (b) `setInterval`/`setTimeout`/Timer instances not cleared on unmount, (c) large datasets held in component state or module singletons instead of paginated/windowed access, (d) native-side leaks: Kotlin/Java listeners not unregistered in lifecycle callbacks, Swift closures capturing `self` strongly in long-lived handlers.
Tools: ast-grep pair-matching (register/unregister, schedule/clear), code-graph/knowledge-graph tool if available (singleton identification), Read.
Failure conditions: >150 raw candidates (Context Overflow — compress per-module); lifecycle pairing undecidable → flag `unknown` in the confidence block, never PASS.
Output artifact: `node-3-leaks.md`.

**Node 4 — List virtualization & image downsampling [P].**
Action: (a) find every list rendering that can exceed ~50 items rendered via plain map/ScrollView/Column iteration; mandate virtualization — FlashList (or equivalent) preferred, or `getItemLayout` + windowing for FlatList to eliminate dynamic height measurement overhead; verify stable keys and memoized row renderers. (b) Trace image rendering paths: flag full-resolution assets decoded into small views; require downsampling/resize-to-view (resizeMode + explicit dimensions, Glide/Coil/SDWebImage downsampling, or an equivalent view-sized image-caching helper) before render so bitmap allocations match display size.
Tools: ast-grep, Read, Bash (asset dimension inspection where assets are local).
Failure conditions: item-count bound unknowable statically → flag as candidate with confidence block `unknown` and a runtime-verification note.
Output artifact: `node-4-lists-images.md`.

**Node 5 — Main-thread / TTID audit (ANR & frame-stutter hunt).**
Action: identify synchronous operations on the startup and interaction paths: synchronous SQLite/storage reads, heavyweight SDK initialization in app entry, synchronous JSON parsing of large payloads, synchronous file I/O in render/lifecycle methods. Require each to be deferred to background threads/isolates/coroutines with an explicit startup ordering, prioritizing Time to Initial Display; interactive-path blockers are ANR candidates on Android (main thread > 5s) and frame-stutter sources everywhere (budget: 16ms/frame).
Tools: ast-grep, node-1/node-2 maps, Read; Bash (build-time startup trace if toolchain present).
Failure conditions: deferral would change observable init order for a dependency (Architecture Failure — emit finding, require operator decision rather than silent reorder).
Output artifact: `node-5-ttid-anr.md` — blocker table with thread, path (startup vs interaction), and deferral plan.

**Node 6 — Platform manifest hardening.**
Action: parse `AndroidManifest.xml` and `Info.plist` and verify:
- No component (activity/service/receiver/provider) exported (`android:exported="true"`) without a strict permission guard or intent-filter justification — each exported component gets an explicit verdict.
- Debug flags (`android:debuggable`, `usesCleartextTraffic`) disabled for release variants; verify against the actual release build config (Gradle variant merge), not just the source manifest.
- Every requested runtime permission has a matching, accurate `Info.plist` usage description (and Android rationale where applicable); flag permissions requested but unused in code.
Tools: Read (manifests, Gradle/Xcode configs), ast-grep (permission usage cross-check), Bash (merged-manifest generation if Gradle available).
Failure conditions: merged manifest unobtainable → static-only verdicts recorded with `unknown: release-variant merge not verified` in the confidence block.
Output artifact: `node-6-manifest.md` — per-component / per-permission verdict table.

**Node 7 — Native security table.**
Action: verify each row, with file:line evidence:

| Vector | Directive |
|--------|-----------|
| Secure storage | Trace all local data writes. Any sensitive value (JWT, refresh token, PII, API key) written in plaintext to SharedPreferences (Android) or NSUserDefaults (iOS) = CRITICAL; refactor to hardware-backed Android Keystore / iOS Keychain via encrypted wrappers (EncryptedSharedPreferences, Keychain Services). |
| Cleartext & pinning | Cleartext traffic disabled (networkSecurityConfig / ATS). TLS public-key pinning implemented WITH valid backup pins (a single pin is a self-DoS foot-gun and is flagged); app must reject intercepting proxies. |
| Deep links & intents | Every deep link / custom URL scheme / App Link handler sanitizes its parameters before use; authenticated workflows cannot be bypassed by crafting an external intent or URL directly into an inner screen — trace each handler to its auth check. |
| Tamper resilience | R8/ProGuard enabled for Android release (symbols stripped, resources shrunk); root/jailbreak environment checks present AND degrade features gracefully — a check that hard-crashes is itself a finding (predictable crash = trivially patchable by an attacker). |

Tools: ast-grep, Read, Grep for literal key names only, node-6 manifest facts, Bash (proguard config inspection).
Failure conditions: any plaintext-credential CRITICAL = Security Failure → HALT per PROTOCOL.md §4, operator approval required before node 8.
Output artifact: `node-7-security.md` — table verdicts + evidence.

**Node 8 — Fix application (loop branch only).**
Action: severity-ordered fixes, one commit per finding (`loop(LOOP-03): <node> — <change>` + confidence block in body): keystore/keychain migrations (with data-migration path for existing installs — never strand stored tokens), listener/timer cleanup, virtualization swaps, native-driver conversions, manifest tightening, deferral refactors from node 5. Build both release variants after security fixes to prove config lands. Run a secrets scanner (gitleaks, trufflehog, or equivalent) before every commit.
Tools: Edit, Bash (gradle/xcodebuild, git), ast-grep (fix verification).
Failure conditions: release build breaks (Dependency Failure — fix or rollback that commit per PROTOCOL.md §6); storage migration cannot be made safe → operator-deferred with plan.
Output artifact: commit list + `node-8-fixes.md`.

**Node 9 — Regression tests + verification.**
Action: every fixed bug gets a permanent test: unit tests for leak/cleanup logic (mount/unmount cycles assert zero residual listeners/timers), list-virtualization render tests, manifest assertions (test that parses the merged manifest and fails on exported-unguarded or debuggable-release), storage tests asserting sensitive keys never appear in SharedPreferences/NSUserDefaults. Run the full suite. Emit LOOP-12 hand-off manifest (candidates for fuzzing deep-link handlers).
Tools: Write (tests), Bash.
Failure conditions: test cannot reproduce original defect (Logic Failure — rewrite the test).
Output artifact: `node-9-tests.md` + LOOP-12 manifest + final `AUDIT-ARTIFACT.md` per PROTOCOL.md §13.

# Adversarial Check (reviewer persona + what it attacks)

Persona: **hostile mobile penetration tester with a rooted device and a Frida script**, running FREE-MAD. It attacks:

1. Every deep-link "sanitized" verdict — attempts to construct a bypass URL/intent on paper from the handler code and demands the auth-check trace.
2. The pinning implementation — checks for backup pins, pin-failure handling, and whether the pin set actually covers all API hosts.
3. Keystore/keychain migrations — hunts for the old plaintext value left behind un-deleted after migration.
4. Confidence block `unknown` lists first (PROTOCOL.md §3), especially static-only manifest verdicts and untested release variants.
5. Node-2/5 performance fixes — verifies a "deferral" didn't introduce a race on first-use of the deferred resource.
6. Silent Agreement without file:line evidence in round 1 forces a hostile second round (PROTOCOL.md §11).

Maximum 2 debate rounds; survivors become operator-deferred findings.

# Exit Criteria (quantitative, overrides PROTOCOL.md §12)

- Zero plaintext sensitive values (JWT/PII/keys) in SharedPreferences/NSUserDefaults; 100% migrated to Keystore/Keychain with old values purged.
- Zero exported components without permission guards; zero debuggable/cleartext release variants; 100% of runtime permissions with accurate usage descriptions.
- TLS pinning present with ≥1 valid backup pin per pinned host, or explicitly operator-deferred with rationale.
- Zero unguarded deep-link/URL-scheme handlers; all handlers trace to an auth check where the target screen requires auth.
- All lists over the virtualization threshold virtualized; zero per-frame JS-thread animations remaining; all node-5 startup blockers deferred or operator-deferred.
- Zero unresolved CRITICAL/HIGH leak findings; release builds green on both targeted platforms.
- Test pass rate ≥ 99% on touched surface; every fixed bug has a named regression test; retry ≤ 3/node; final confidence block score ≥ 0.85; governance row written.

# Failure Routing (only deviations from PROTOCOL.md §4)

- Missing platform toolchain downgrades affected nodes to static-only and records it in the confidence block's `unknown` list — it does not fail the loop, but the final verdict cannot exceed PASS-with-unknowns and says so.
- Storage-migration hazards route to operator decision (Architecture Failure), never to silent skip.

# Approval Gates (only deviations from PROTOCOL.md §10)

- PROTOCOL.md defaults apply. Additionally: keystore/keychain data migrations and any change to pinning configuration require explicit operator approval BEFORE commit (a wrong pin set bricks connectivity for shipped installs).

# RUN PROMPT

```
Run LOOP-03 on <repo> (scope: mobile/, platform: both). Follow LOOP-03-mobile-native.md under [PROTOCOL] (see PROTOCOL.md in this repo). Audit BOTH layers (JS/Dart logic + native Java/Kotlin/Swift/ObjC): bridge traffic, leaks, virtualization, TTID blockers, then the manifest + security table (keystore/keychain, pinning with backup pins, deep links, R8/ProGuard, graceful root checks). Mutate only on loop/LOOP-03-<date>, regression test every bug, and deliver AUDIT-ARTIFACT + loop branch. No merge without my approval.
```
