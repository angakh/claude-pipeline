---
description: One pass of the delivery pipeline over the configured Jira project — triage, plan, build, and QA-loop tickets, dispatching to specialized agents. This is what the scheduled cron job runs.
argument-hint: (no arguments — reads .claude/delivery-pipeline.config.json)
allowed-tools: [Read, Bash, Agent, EnterWorktree, ExitWorktree]
---

# Pipeline Sweep

One pass over the configured Jira project. You are the orchestrator: you never write code or Jira comments yourself — you read state, decide what stage each ticket is in, and dispatch to the right subagent. This command is stateless between runs; **Jira status + labels + comments and git branch existence are the only source of truth**, so always re-derive state from them rather than assuming anything from a previous run.

## 0. Load config

Read `.claude/delivery-pipeline.config.json` in the project root. If it doesn't exist or fails schema validation against `${CLAUDE_PLUGIN_ROOT}/config.schema.json`, stop immediately and tell the user to run `/pipeline-init` first — do not guess at values.

All status/branch/command names below refer to the config fields (`jira.toDoStatus`, `git.branchPrefix`, etc.), not literal strings — different projects configure these differently.

## 1. Query Jira

Using `mcp__atlassian__searchJiraIssuesUsingJql` against `jira.cloudId` / `jira.projectKey`, pull tickets in each of these statuses (include labels, comments, issuetype, parent):

- `jira.toDoStatus` — new/unprocessed tickets
- `jira.buildStatus` — approved plans and in-flight builds
- `jira.qaStatus` — built branches awaiting QA

Skip anything already carrying the label `needs-human` — those are parked for you until a human clears the label.

## 2. Categorize and dispatch

For each ticket in **toDoStatus**:

- **Epic with a PRD-shaped description, no child stories, no `stories-generated` label** → dispatch to the `pm-decompose` agent.
- **Bug, or a Task/Story whose description reports a symptom rather than specifying net-new work, no `research-confirmed` or `needs-triage-decision` label yet** → dispatch to the `research-triage` agent first. Do not send these straight to planning.
- **Story/Task with `research-confirmed` label (or a clearly-scoped net-new feature with nothing to verify), no plan comment yet** → dispatch to the `tech-planner` agent.
- Anything already carrying `needs-triage-decision` → skip; it's parked for a human decision, already notified.

For each ticket in **buildStatus** with a `plan-drafted` label and no existing `feature/<key>`-prefixed branch (check via `git ls-remote --heads <remote>` using `git.branchPrefix`) → this is the human's approval signal (they moved it here themselves). Dispatch to the `engineer-build` agent. If there IS already a branch and a fresh QA-failure comment newer than the last engineer comment, this is a retry loop — dispatch to `engineer-build` again with the QA failure report attached.

For each ticket in **qaStatus** with a pushed branch → dispatch to the `qa-verify` agent.
- Before dispatching, count prior QA-failure comments (marked `[QA REPORT — FAIL]`) on the ticket. If the count has already reached `qa.maxCycles`, do NOT dispatch again — instead label `needs-human`, leave status as-is, and push-notify the user with a summary of every failed cycle and why. This cap exists so a genuinely hard bug doesn't silently burn cycles forever.

## 3. Dispatching mechanics

- Use the `Agent` tool with the matching `subagent_type` (`pm-decompose`, `research-triage`, `tech-planner`, `engineer-build`, `qa-verify`) — these are defined in this plugin's `agents/` directory.
- **Always brief the agent from scratch** — it has no memory of this conversation. Pass: the ticket key, full description/comments (fetched via `mcp__atlassian__getJiraIssue` with `comment` in fields), the relevant slice of the project config, and — for engineer/QA — the branch name to use.
- **Multiple tickets in flight**: give each ticket its own git worktree via `EnterWorktree`/`ExitWorktree` before dispatching engineer/QA agents, so parallel builds never collide on branch checkout. Run independent tickets' agent dispatches in parallel; don't parallelize stages *within* one ticket (triage before plan before build before QA is a strict sequence per ticket).
- Agents report back what they did (comment posted, label set, status transitioned, branch pushed) — trust their report but spot-check anything surprising before moving to the next ticket.

## 4. What you never do directly

- Never write a Jira comment, transition a status, create a branch, or run app code yourself in this command — that's always a subagent's job, so the division of responsibility (and the audit trail in Jira comments) stays clean.
- Never push to `git.trunkBranch`.
- Never touch a ticket labeled `needs-human` — that label means the loop stopped on purpose and a human needs to look.

## 5. End of sweep

Summarize in your final message (to whoever/whatever invoked this — a human or the cron log): how many tickets were touched, what stage each moved to, and anything that hit `needs-human` or `needs-triage-decision` this pass. Push-notifications for individual ready-for-review or stuck tickets happen from within the relevant agent (qa-verify, research-triage) as they complete — this summary is the sweep-level roundup, not a duplicate notification.
