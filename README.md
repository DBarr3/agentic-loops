<div align="center">

# Agentic Loops

**15 autonomous, self-contained "loop" systems that turn Claude Code (or any AI coding agent) into a full audit-and-fix pipeline — with built-in adversarial review, confidence scoring, and safe rollback.**

[![GitHub stars](https://img.shields.io/github/stars/DBarr3/agentic-loops?style=social)](https://github.com/DBarr3/agentic-loops/stargazers)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg?style=flat-square)](LICENSE)
[![Made by Aether AI](https://img.shields.io/badge/made%20by-Aether%20AI-6f42c1?style=flat-square)](https://github.com/DBarr3)
[![Claude Code Compatible](https://img.shields.io/badge/Claude%20Code-compatible-CC785C?style=flat-square)](https://claude.com/claude-code)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=flat-square)](CONTRIBUTING.md)

</div>

---

## What is this?

A **loop** is a complete, runnable audit-and-fix system in a single markdown file — not a one-off prompt. Each loop defines a mission, a multi-step execution DAG, per-node tool specs, a hostile adversarial reviewer, quantitative exit criteria, and a `RUN PROMPT` you can paste straight into Claude Code (or any capable coding agent) right now.

Every loop shares one contract, defined once in [`PROTOCOL.md`](PROTOCOL.md): confidence scoring on every output, a failure taxonomy that routes instead of just failing, retry/escalation budgets, checkpointed rollback on a dedicated branch, and a FREE-MAD adversarial debate step before anything is called "done." That's what makes these *loops* — self-correcting systems, not single-shot prompts.

15 loops cover backend/API security, frontend a11y, mobile, agent-runtime security, memory architecture, UX/UI, infra/DevOps, CI/CD, data layer, AI red-teaming, adversarial review, test/mutation/chaos engineering, architectural drift, governance telemetry, and self-optimization.

## Table of Contents

- [Quick Start](#quick-start)
- [The Loop Catalog](#the-loop-catalog)
- [How It Works](#how-it-works)
- [Installing as a Claude Code Skill](#installing-as-a-claude-code-skill)
- [Suggested Cadence](#suggested-cadence)
- [Contributing](#contributing)
- [License](#license)

## Quick Start

Pick a loop, paste its `RUN PROMPT` into your AI coding agent, done. No install required.

```
Run LOOP-01 on <your-repo> (scope: <api dir>)
```

The agent reads the loop file, reads [`PROTOCOL.md`](PROTOCOL.md) for the shared execution rules, and runs the whole audit end-to-end — handing you back a branch, a findings artifact, and nothing merged without your sign-off.

Three ways to use this repo:

1. **Copy-paste** — open any file in [`skills/`](skills/), copy the `RUN PROMPT` block at the bottom, paste it into Claude Code (or ChatGPT, Cursor, Windsurf, etc.) with your target repo.
2. **Install as a skill** — each loop's frontmatter is already skill-compatible (see [Installing as a Claude Code Skill](#installing-as-a-claude-code-skill)).
3. **Clone the whole set** — `git clone` this repo and point your agent at the `skills/` folder for the full catalog.

## The Loop Catalog

| # | Loop | Domain | Risk | What it does |
|---|------|--------|------|---------------|
| [01](skills/LOOP-01-backend-api.md) | Backend & API Hardening | Backend & API Architecture | branch-mutating | AST-driven audit: OWASP API Top 10 (BOLA/BOPLA/SSRF/auth/resource limits), N+1 queries, dead code, logging hygiene. |
| [02](skills/LOOP-02-frontend-a11y.md) | Frontend & Accessibility | Frontend & Accessibility | branch-mutating | Component/bundle audit plus a live Playwright accessibility-tree interrogation (roles, ARIA, focus traps, contrast) that static scanners miss. |
| [03](skills/LOOP-03-mobile-native.md) | Mobile & Native Hardening | Mobile & Native Platforms | branch-mutating | Dual-layer (JS/Dart + native) audit: bridge traffic, ANR hunters, list virtualization, manifest hardening, keystore/keychain, pinning. |
| [04](skills/LOOP-04-agent-runtime.md) | Agent Runtime & Tooling | Agent Runtime & Tooling | read-only→branch | Audits your own agent stack against the OWASP Agentic Top 10: MCP/tool schema strictness, least-privilege agency, prompt-injection surfaces. |
| [05](skills/LOOP-05-memory-ux.md) | Agentic Memory Hardening | Memory & Cognitive Architecture | branch-mutating | Four-tier memory audit (working/episodic/semantic/procedural), tenant isolation, unbounded growth, GDPR erasure UI. |
| [06](skills/LOOP-06-ux-ui-visual.md) | UX/UI Visual Integrity | UX/UI & Human Factors | branch-mutating | Visual regression, stateful UI battery, graceful-degradation checks, memory-transparency panel audit. |
| [07](skills/LOOP-07-infra-devops.md) | Infrastructure & DevOps | Infrastructure, Cloud & DevOps | infra-touching | Docker/K8s/Terraform/secrets/IAM/SBOM/supply-chain audit across your deployment mesh — read+plan, never live-mutates without approval. |
| [08](skills/LOOP-08-cicd-release.md) | CI/CD & Release Engineering | CI/CD & Release Engineering | branch-mutating | Pipeline audit: is CI *actually* running, caching, test gating, artifact signing, rollback automation, canary/blue-green, feature flags. |
| [09](skills/LOOP-09-data-layer.md) | Data Layer Integrity | Data Architecture & Persistence | infra-touching | Schema/migration safety, FK/orphan/cascade audit, index coverage, RLS/grant self-escalation sweep, backup **restore** verification. |
| [10](skills/LOOP-10-ai-security.md) | AI Security Red-Team | Governance, Compliance & Risk | read-only | Red-teams your own AI stack: prompt injection, RAG/memory/embedding poisoning, model supply chain, jailbreaks, goal drift, reward hacking. |
| [11](skills/LOOP-11-adversarial-review.md) | Adversarial Review Engine | Cross-cutting engine | read-only | The debate engine every other loop calls inline: generator vs. hostile reviewer, FREE-MAD default, silent-agreement detection. |
| [12](skills/LOOP-12-test-mutation-chaos.md) | Test, Mutation & Chaos | Continuous Evaluation | branch-mutating | Closes coverage gaps, mutation-tests your suite to find tests that don't actually test anything, injects chaos (DB/cache/LLM/MCP failures). |
| [13](skills/LOOP-13-drift-debt.md) | Architectural Drift & Debt | Continuous Evaluation | read-only | Detects layer violations, circular deps, dead APIs/services; scores technical debt 0–100 across six axes, trended over time. |
| [14](skills/LOOP-14-governance-telemetry.md) | Governance & Telemetry | Runtime Observability | read-only | Audits the autonomous system itself: is it improving, looping, degrading, wasting tokens, or getting less reliable? |
| [15](skills/LOOP-15-self-optimization.md) | Self-Optimization Meta-Loop | Continuous Evaluation | branch-mutating (loops only) | The recursive-improvement loop: failure → root cause → fix → generalized pattern → proposed diff to the loops themselves. Human-gated always. |

**Risk classes:** `read-only` never touches your code · `branch-mutating` works only on an isolated loop branch, never main · `infra-touching` reads live infrastructure and proposes plans, never applies changes without explicit approval.

## How It Works

Every loop shares one execution contract, defined once in [`PROTOCOL.md`](PROTOCOL.md) so the loop files themselves stay focused on their domain:

- **4-step protocol** — parse & ingest → scope context → monitored execution → adversarial check, before anything is marked done.
- **Confidence blocks** — every output carries a calibrated confidence score, a risk level, and an explicit `unknown` list the reviewer has to attack first.
- **Failure taxonomy** — 13 named failure classes (syntax, logic, security, dependency, permission, hallucination...) each with a defined route, so nothing just silently "fails."
- **Rollback engineering** — every mutation happens on a checkpointed branch; rollback is `git reset --hard` plus a governance event, never a scramble.
- **Adversarial debate (FREE-MAD)** — a hostile reviewer persona attacks the work before it's accepted; silent unanimous agreement with no cited evidence forces a second round.
- **Quantitative exit criteria** — every loop defines measurable PASS conditions; a loop that can't meet them terminates as `FAIL-with-artifact`, never a silent success.

Read [`PROTOCOL.md`](PROTOCOL.md) for the full spec — it's the shortest path to understanding all 15 loops at once.

## Installing as a Claude Code Skill

Each loop's YAML frontmatter (`name`, `description`, etc.) is already skill-shaped. To install one:

```bash
mkdir -p ~/.claude/skills/loop-backend-api
cp skills/LOOP-01-backend-api.md ~/.claude/skills/loop-backend-api/SKILL.md
```

Repeat per loop, or copy the whole `skills/` folder and adapt the frontmatter `name` field per file. Once installed, invoke with `/loop-backend-api` (or your agent's skill-invocation syntax) instead of pasting the `RUN PROMPT` manually.

## Suggested Cadence

A reasonable starting rhythm — tune it to your team:

| Cadence | Loops |
|---|---|
| Weekly sweep | 01 (backend), 02 (frontend), 13 (drift/debt) |
| Pre-release | 08 (CI/CD), 12 (test/mutation/chaos), 06 (UX/UI) |
| Monthly deep audit | 07 (infra), 09 (data layer), 10 (AI security), 05 (memory) |
| Continuous / every N runs | 14 (governance), then 15 (self-optimization) |

## Contributing

New loops, sharper adversarial personas, better exit criteria — all welcome. See [`CONTRIBUTING.md`](CONTRIBUTING.md).

## License

[MIT](LICENSE) — use it, fork it, ship it.

---

<div align="center">

Built by **[Aether AI](https://github.com/DBarr3)**

</div>
