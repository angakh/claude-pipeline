---
description: Set up the delivery-pipeline plugin for the current project by generating and explaining .claude/delivery-pipeline.config.json
argument-hint: (no arguments)
allowed-tools: [Read, Write, Bash, Glob, Grep]
---

# Pipeline Init

You are setting up the delivery-pipeline plugin for **this** project. Nothing runs automatically until this finishes — your only job here is to produce a valid, well-understood `.claude/delivery-pipeline.config.json`.

Read `${CLAUDE_PLUGIN_ROOT}/config.schema.json` first. That file is the source of truth for every field: what it's called, whether it's required, and — critically — its `description`, which explains *why the pipeline needs it*. Do not invent fields that aren't in the schema, and do not skip required ones.

## Process

1. **Check for an existing config.** If `.claude/delivery-pipeline.config.json` already exists in this project, read it, show the user its current values grouped by schema section, and ask whether they want to redo setup or just patch specific fields. Don't blindly overwrite.

2. **Auto-detect what you can, then confirm rather than guess silently:**
   - `git.trunkBranch` — check `git remote show origin` / default branch, or ask if ambiguous.
   - `git.branchPrefix` — look at recent branch names (`git branch -a --sort=-committerdate | head -20`) for an existing convention before inventing one.
   - `dev.startCommand` / `dev.testCommand` — check `package.json` scripts (`dev`, `start`, `test`) or the equivalent for the project's stack.
   - `jira.cloudId` / `jira.projectKey` — call `mcp__atlassian__getAccessibleAtlassianResources` and `mcp__atlassian__getVisibleJiraProjects`, then ask the user to confirm which project.
   - `jira.toDoStatus` / `planReviewStatus` / `buildStatus` / `qaStatus` / `doneStatus` — call `mcp__atlassian__getTransitionsForJiraIssue` on any real issue in the target project to enumerate the workflow's actual status names. **Do not assume the schema's defaults match this project's workflow** — confirm against the real statuses returned.
   - `db.renderPostgresId` — call `mcp__render__list_postgres_instances` and ask the user to pick.
   - Everything else (`db.mode`, `qa.maxCycles`, `qa.testAccountStrategy`, `pr.autoCreate`) has no safe auto-detection — ask directly.

3. **For every field you ask the user about, state the schema's `description` in your own words before asking**, so they understand *why* they're being asked, not just what format to answer in. Group the questions by schema section (jira / git / dev / db / qa / pr) rather than firing them one at a time.

4. **`db.mode` deserves an explicit explanation, not just a pick-one prompt** — this is the field that decides whether an unattended cron job gets direct read access to production data. Explain the three options plainly (direct read-only always, direct only for single-record lookups, or local-restore-only) before asking which the user wants.

5. **Write the file** to `.claude/delivery-pipeline.config.json` in the project root, validate it mentally against every `required` field in the schema, and if `.claude/.gitignore` or the project's root `.gitignore` would exclude it, warn the user — this file is meant to be checked in (it holds identifiers, never secrets).

6. **Finish with a summary table** of every field set and its value, plus a short "what's still manual" note:
   - Scheduling the cron sweep is a separate step (`schedule` skill / CronCreate) — this command only writes config, it doesn't schedule anything.
   - Nothing runs against this project until a human schedules `pipeline-sweep` or runs it manually.

## Guardrails while running this command

- Never write actual credentials, tokens, or passwords into the config file — if a field seems to want one, that's a sign it belongs in existing secret storage instead, not this file. Flag it to the user rather than writing it.
- Don't guess Jira status names — get them from the live workflow via the MCP tools, not from the schema's `default` values, which are only a fallback documentation aid.
