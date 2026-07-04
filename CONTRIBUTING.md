# Contributing

Loops are just markdown files — no build step, no dependencies.

## Adding a new loop

1. Copy the Standard Loop File Template from `PROTOCOL.md` §14.
2. Follow the numbering convention: pick the next free `LOOP-XX` id, or propose a sub-variant of an existing domain.
3. Every loop needs: a Mission, an Execution DAG, per-node tool/failure specs, an Adversarial Check with a real hostile persona, quantitative Exit Criteria, and a copy-pasteable `RUN PROMPT`.
4. Keep it general — no project-specific paths, tool names, or infrastructure assumptions. Anyone should be able to run your loop on their own repo with zero edits beyond the target.

## Improving an existing loop

PRs that tighten exit criteria, add a missed bug class as a new node, or sharpen an adversarial persona are welcome. Explain what failure mode motivated the change.

## Reporting an issue

If a `RUN PROMPT` doesn't work as written against a real agent, open an issue with the agent/tool you used and what broke.
