# vlasnacrm-agents

Claude Code plugin that builds Next.js 16 + React 19 + Tailwind v4 + shadcn + Supabase apps — landing pages (`marketing`) and CRM/ERP back-offices (`backoffice`) — and keeps every screen consistent, tested and reviewed.

## Install

```text
/plugin marketplace add alsetsoft/vlasnacrm-agents-plugin
/plugin install vlasnacrm-agents@alsetsoft
```

Update: `/plugin marketplace update alsetsoft` → `/plugin update vlasnacrm-agents@alsetsoft` → `/reload-plugins`.

For the whole team on a project, commit to its `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": { "alsetsoft": { "source": { "source": "github", "repo": "alsetsoft/vlasnacrm-agents-plugin" } } },
  "enabledPlugins": { "vlasnacrm-agents@alsetsoft": true }
}
```

## Use

| You want to… | Run |
|--------------|-----|
| build a new app | `/vlasnacrm backoffice CRM for a furniture store: clients, orders, payments` |
| connect an existing project (once) | `/vlasnacrm-adopt` — writes `.vlasnacrm/policy.json` + `conventions.md` from your CLAUDE.md and code; review and commit them |
| add a screen / entity | `/vlasnacrm add a suppliers list and form` |
| fix UI or a bug | `/vlasnacrm-fix archived records still appear in pickers` — fixed app-wide, `@regression` e2e test first |
| make a client/QA remark a rule | `/vlasnacrm-lesson after saving, stay on the record` |

Describe **what** you want — not which rules to follow. Your `CLAUDE.md` / `AGENTS.md` always win over `.vlasnacrm/policy.json`, which wins over the agents' defaults. Commit `.vlasnacrm/` with the code.

**Skills are loaded by the agents, not by you.** The orchestrator maps each task to skills (an order form → `crud-form` + `document-lines`), builders read them before writing code, the reviewer runs their audit tables, the e2e writer uses `e2e-crud` and regenerates the testers' PDF (`test-docs`) whenever test cases change. Skills: `crud-list`, `crud-form`, `document-lines`, `bulk-actions`, `filters`, `dashboard-charts`, `master-detail-tabs`, `export-print`, `e2e-crud`, `test-docs`. Calling one by hand (`/vlasnacrm-agents:<skill> <task>`) is only for one-off tasks outside the pipeline.

**Tests:** e2e and visual QA run against a test environment (local stack or a separate test DB). For logged-in screens set `E2E_EMAIL` / `E2E_PASSWORD` (e2e) and `VLASNACRM_QA_EMAIL` / `VLASNACRM_QA_PASSWORD` (visual QA). Test data never stays behind: every record the dedicated e2e account creates is tracked and deleted after the run, and the run fails if anything is left (`e2e_cleanup` / `e2e_residue`).

**Bug fixes on production:** every fix gets a `@regression` test — a DB test that always rolls back when the database decides the bug, a UI test otherwise. With `"e2e": { "prod": true }` in `.vlasnacrm/policy.json`, tests tagged `@prod-safe` can prove a hotfix on production with zero residue: `E2E_TARGET=prod E2E_ALLOW_PROD=1 npx playwright test --grep @prod-safe`. Agents never set `E2E_ALLOW_PROD` themselves. Details: `skills/e2e-crud` → *Bug-fix verification on production*.

## Structure

```
.claude-plugin/marketplace.json          marketplace "alsetsoft"
plugins/vlasnacrm-agents/
  .claude-plugin/plugin.json             name + version
  agents/     orchestrator → project-init, design-system, component-builder, page-assembler,
              db-snapshot, design-reviewer, e2e-writer, visual-qa
  commands/   /vlasnacrm, /vlasnacrm-adopt, /vlasnacrm-fix, /vlasnacrm-lesson
  skills/     <name>/SKILL.md — rules for data screens and tests (+ an Audit table for the reviewer)
  scripts/    visual-qa.mjs, test-docs.mjs
```

Agents read rules from skills by name; change a data-screen rule in its skill, not in the agents. Keep everything generic — no project-specific names or business rules (those go into the project's CLAUDE.md or `.claude/skills/`).

## Contribute

1. Branch off `main`, edit under `plugins/vlasnacrm-agents/`.
2. **Bump `version`** in `plugin.json` (SemVer) — without it `/plugin update` keeps serving the old version.
3. Check locally:
   ```bash
   claude plugin validate ./plugins/vlasnacrm-agents
   claude --plugin-dir ./plugins/vlasnacrm-agents   # try it without installing
   ```
4. Open a PR to `main`; merge after one approval from another team member.
