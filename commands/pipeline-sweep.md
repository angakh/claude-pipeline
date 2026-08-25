---
description: One pass of the delivery pipeline over the configured Jira project — triage, plan, build, and QA-loop tickets, dispatching to specialized agents. This is what the scheduled cron job runs.
argument-hint: "[list | <count> | <TICKET-1>,<TICKET-2>,...]  (no arguments = default capped/prioritized run)"
allowed-tools: [Read, Bash, Agent, EnterWorktree, ExitWorktree]
---

# Pipeline Sweep

One pass over the configured Jira project. You are the orchestrator: you never write code or Jira comments yourself — you read state, decide what stage each ticket is in, and dispatch to the right subagent. This command is stateless between runs; **Jira status + labels + comments and git branch existence are the only source of truth**, so always re-derive state from them rather than assuming anything from a previous run.

## 0. Load config

Read `.claude/delivery-pipeline.config.json` in the project root. If it doesn't exist or fails schema validation against `${CLAUDE_PLUGIN_ROOT}/config.schema.json`, stop immediately and tell the user to run `/pipeline-init` first — do not guess at values.

All status/branch/command names below refer to the config fields (`jira.toDoStatus`, `git.branchPrefix`, etc.), not literal strings — different projects configure these differently.

## 0.5. Parse `$ARGUMENTS` — pick a mode

- **Empty** → `default` mode: use `sweep.maxTicketsPerRun` and the priority ordering in step 2 as-is, including the first-few-sweeps auto-cap-to-1 safety net.
- **The literal word `list`** → `list` mode: build the full prioritized candidate list (step 2) but **do not dispatch anything at all** — no agents, no Jira writes, nothing. Print the list as a table (ticket key, type, current status, what stage/agent it would go to, and why — or why it'd be skipped, e.g. `needs-human`) and stop. This is the "show me what's there so I can choose" mode.
- **A bare integer** (e.g. `3`) → `count-override` mode: same as `default`, except use this number in place of `sweep.maxTicketsPerRun` for this run only — **and this explicit number also overrides the first-few-sweeps auto-cap-to-1 safety net**, since typing a specific count is a deliberate choice that supersedes the automatic caution default.
- **A comma-separated list of ticket keys** (e.g. `SCR-144,SCR-163`) → `explicit` mode: skip the querying/ordering/capping in step 2 entirely. Fetch exactly these tickets, validate each belongs to `jira.projectKey` (reject and report any that don't), and run each through the normal categorization logic in step 3 as if it had been selected — including a ticket labeled `needs-human`, since naming it explicitly here **is** the human clearing it for another pass. Still respects `sweep.maxConcurrentAgents` for dispatch pacing.

## 1. Query Jira

Using `mcp__atlassian__searchJiraIssuesUsingJql` against `jira.cloudId` / `jira.projectKey`, pull tickets in each of these statuses (include labels, comments, issuetype, parent, priority, updated):

- `jira.toDoStatus` — new/unprocessed tickets
- `jira.buildStatus` — approved plans and in-flight builds
- `jira.qaStatus` — built branches awaiting QA

Skip anything already carrying the label `needs-human` — those are parked for you until a human clears the label.

## 2. Cap and order the run — do not process everything you found

This step applies to `default` and `count-override` modes. (`list` mode does everything here except the final cut, then stops instead of dispatching. `explicit` mode skips this step entirely — go straight to step 3 with exactly the named tickets.)

A real backlog can easily return more matching tickets than one sweep should touch at once. **Before dispatching anything**, build one prioritized list across all three statuses and cut it to the effective cap (`sweep.maxTicketsPerRun`, or the number from `count-override` mode) — this is a hard cap on the whole run, not per-status.

Order the candidates like this, then take the top N:

1. **qaStatus tickets first** (finish what's already built before starting anything new).
2. **buildStatus tickets second** (finish what's already approved/mid-build next).
3. **toDoStatus tickets last** (only start new triage/planning work with whatever budget remains).
4. Within each group, order by Jira priority (highest first), then by `updated` ascending (oldest-untouched first) as a tiebreak.

In `default` mode only: if this project has had **fewer than 3 completed sweeps so far** (check: are there any tickets anywhere in this project carrying a pipeline comment marker like `[TRIAGE]`, `[TECH PLAN]`, `[BUILD]`, or `[QA REPORT`?), treat this as an early run and cap the list to **1** regardless of `sweep.maxTicketsPerRun`, so the first real end-to-end pass is easy to watch and sanity-check before trusting it with a full batch. Say explicitly in your final summary that you did this and why. `count-override` mode never applies this — an explicit number is already the user overriding the default.

In `list` mode: keep the whole ordered list (don't cut it), and mark each entry with the cap line — i.e. show which ones would run in a `default`-mode pass and which wouldn't, so the count is informative context, not just a raw dump.

Anything that didn't make the cut this pass (`default`/`count-override` modes) is simply left for the next sweep — no action needed on it, nothing to flag.

## 3. Categorize and dispatch

In `list` mode, work out which agent each ticket *would* go to using the rules below, note it in the table, and stop — do not call the `Agent` tool at all in this mode.

For each ticket in **toDoStatus**:

- **Epic with a PRD-shaped description, no child stories, no `stories-generated` label** → dispatch to the `pm-decompose` agent.
- **Bug, or a Task/Story whose description reports a symptom rather than specifying net-new work, no `research-confirmed` or `needs-triage-decision` label yet** → dispatch to the `research-triage` agent first. Do not send these straight to planning.
- **Story/Task with `research-confirmed` label (or a clearly-scoped net-new feature with nothing to verify), no plan comment yet** → dispatch to the `tech-planner` agent.
- Anything already carrying `needs-triage-decision` → skip; it's parked for a human decision, already notified.

For each ticket in **buildStatus** with a `plan-drafted` label and no existing `feature/<key>`-prefixed branch (check via `git ls-remote --heads <remote>` using `git.branchPrefix`) → this is the human's approval signal (they moved it here themselves). Dispatch to the `engineer-build` agent. If there IS already a branch and a fresh QA-failure comment newer than the last engineer comment, this is a retry loop — dispatch to `engineer-build` again with the QA failure report attached.

For each ticket in **qaStatus** with a pushed branch → dispatch to the `qa-verify` agent.
- Before dispatching, count prior QA-failure comments (marked `[QA REPORT — FAIL]`) on the ticket. If the count has already reached `qa.maxCycles`, do NOT dispatch again — instead label `needs-human`, leave status as-is, and push-notify the user with a summary of every failed cycle and why. This cap exists so a genuinely hard bug doesn't silently burn cycles forever.

## 4. Dispatching mechanics

- Use the `Agent` tool with the matching `subagent_type` (`pm-decompose`, `research-triage`, `tech-planner`, `engineer-build`, `qa-verify`) — these are defined in this plugin's `agents/` directory.
- **Always brief the agent from scratch** — it has no memory of this conversation. Pass: the ticket key, full description/comments (fetched via `mcp__atlassian__getJiraIssue` with `comment` in fields), the relevant slice of the project config, and — for engineer/QA — the branch name to use.
- **Multiple tickets in flight**: give each ticket its own git worktree via `EnterWorktree`/`ExitWorktree` before dispatching engineer/QA agents, so parallel builds never collide on branch checkout. Dispatch independent tickets' agents in parallel, but never more than `sweep.maxConcurrentAgents` (default 2) at once — queue the rest of this run's capped list behind the first batch rather than firing them all simultaneously. Don't parallelize stages *within* one ticket (triage before plan before build before QA is a strict sequence per ticket).
- Agents report back what they did (comment posted, label set, status transitioned, branch pushed) — trust their report but spot-check anything surprising before moving to the next ticket.

## 5. What you never do directly

- Never write a Jira comment, transition a status, create a branch, or run app code yourself in this command — that's always a subagent's job, so the division of responsibility (and the audit trail in Jira comments) stays clean.
- Never push to `git.trunkBranch`.
- Never touch a ticket labeled `needs-human` — that label means the loop stopped on purpose and a human needs to look.
- Never exceed `sweep.maxTicketsPerRun` or `sweep.maxConcurrentAgents` — the cap from step 2 is a hard ceiling for this run, not a suggestion.

## 6. End of sweep

**`list` mode** ends with just the table from steps 2-3 — ticket key, type, summary, current status, the agent it would go to (or the reason it'd be skipped), and whether it falls inside or outside the default cap. Nothing was dispatched, so there's nothing else to report. Tell the user how to act on it: `/pipeline-sweep <count>` to run the top N, or `/pipeline-sweep TICKET-1,TICKET-2` to run specific ones.

**`default`, `count-override`, and `explicit` modes** end with a summary in your final message (to whoever/whatever invoked this — a human or the cron log): how many tickets matched vs. how many were actually touched (i.e. how many were left capped for next run — not applicable in `explicit` mode, where the named list *is* the run), what stage each touched ticket moved to, and anything that hit `needs-human` or `needs-triage-decision` this pass. Push-notifications for individual ready-for-review or stuck tickets happen from within the relevant agent (qa-verify, research-triage) as they complete — this summary is the sweep-level roundup, not a duplicate notification.
