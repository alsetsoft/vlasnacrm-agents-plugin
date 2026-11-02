---
description: Apply a UI fix or behaviour change across the whole app — find every place with the same kind of data/interaction, unify it behind one shared primitive, record the rule in conventions.md. Dispatches the orchestrator agent.
argument-hint: <what is wrong / what should change, e.g. "archived records show up in the customer picker" or "the item picker should show available quantity">
---

Invoke the `vlasnacrm-agents:orchestrator` agent with the request below and the instruction to follow its **Consistency Rule** (fix the pattern, not the instance). Reply in the user's language. Dispatch it in the **foreground** (`run_in_background: false`) and don't reply until it returns its final report; if it ever hands back an interim "still running" message, resume it (SendMessage) until the pipeline is done.

Request: $ARGUMENTS

Expected flow:
1. Read `.vlasnacrm/policy.json`, `.vlasnacrm/conventions.md` (canon + `## Patterns`) and the project's CLAUDE.md / AGENTS.md if present.
2. Classify: pattern-wide (default) or explicitly screen-specific.
3. Inventory every occurrence of the same data shape / interaction (`file:line — current behaviour`) and show it in 3–10 lines; ask once only if it's very large or needs a product decision.
4. `e2e-writer` (Mode B): a `@regression` Playwright test reproducing the problem — written first, it must fail now (re-run after step 5: it must pass). DB-level bug → a rollback DB test; test data is cleaned up and the residue check must pass. For a production hotfix the `@prod-safe` test verifies it on production — only when `policy.e2e.prod` is on and the user runs/approves it.
5. `component-builder`: one shared primitive, migrate all occurrences, delete local variants.
6. `design-reviewer`: section 13 must find zero stragglers; `tsc --noEmit` passes.
7. Add/update the pattern's entry in `conventions.md` → `## Patterns` and a dated line in the Decision log.
8. Summary: pattern, N sites unified, primitive path, regression test name.
