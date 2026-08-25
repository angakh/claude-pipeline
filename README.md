# delivery-pipeline

A Claude Code plugin that runs a Jira ticket through a full delivery cycle — confirm the problem is real, plan it, build it, QA it, hand it to a human — with Jira itself as the state machine so the process survives between runs without any external database.

**Status: local, unpublished.** Not on a marketplace yet — install it from this local path in each project until it's stable enough to push to a real git remote.

## The idea

A cron job sweeps a Jira project on an interval. Every ticket it touches gets re-evaluated from scratch each time, based only on its current Jira status, labels, and comments, plus whether a git branch already exists for it — there's no separate database tracking pipeline state, because Jira already **is** the state, and that means you can always intervene by hand (move a card, leave a comment) and the next sweep will respect it.

**Each sweep caps itself** at `sweep.maxTicketsPerRun` (default 5) — it does not try to process an entire backlog at once. Tickets already mid-pipeline (awaiting QA, then awaiting build) are always finished before new tickets are started, and anything left over just waits for the next sweep. A project's first few sweeps are automatically capped to 1 ticket regardless of that setting, so you can watch one go through the whole pipeline before trusting it with a full batch.

Five specialized agents do the actual work; the orchestrator (`pipeline-sweep`) never writes code or Jira comments itself, only reads state and dispatches:

| Agent | Job | Stops for a human when |
|---|---|---|
| `pm-decompose` | Epic + PRD → child Story issues | never — always finishes on its own |
| `research-triage` | Confirms a reported bug is real, and real for the reason claimed, using DB/log/browser evidence — **not** the reporter's word alone | it can't confirm the issue, or the real cause is different from what's described |
| `tech-planner` | Turns a confirmed/scoped ticket into a concrete implementation plan, posted as a Jira comment | always — every plan waits for a human to approve it |
| `engineer-build` | Implements the approved plan on `feature/<ticket-key>`, one commit, pushed | only if the plan turns out fundamentally unworkable |
| `qa-verify` | Runs the test suite + drives the app in a real browser + reads server/console logs | on QA pass, hands off to a human for merge review; after `qa.maxCycles` failed loops on one ticket, stops looping and escalates instead |

## Why there's a triage step at all

The obvious version of this pipeline would go straight from "ticket in To Do" to "plan a fix." That's wrong for bug reports specifically, because the symptom described and the actual problem are often different things — "this user's profile isn't working" might mean the profile system is broken, or it might mean this one profile is simply incomplete. `research-triage` exists to tell those apart *with evidence* (a real query, real logs, a real repro) before anything gets planned, so the pipeline never confidently builds the wrong fix.

## The state machine

Jira status + two kinds of marker carry all the state:
- **Labels** distinguish stages that share a status (see below).
- **Comment prefixes** (`[TRIAGE]`, `[TECH PLAN]`, `[BUILD]`, `[QA REPORT — PASS/FAIL]`) make the ticket's history legible to a human skimming it, and let agents count prior QA cycles without any external tracking.

```
toDoStatus
  │
  ├─ Epic+PRD ──────────────────► pm-decompose ─► doneStatus (label: stories-generated)
  │
  └─ Bug / ambiguous ──► research-triage
        │                    │
        │            not confirmed / wrong premise
        │                    └─► needs-triage-decision label, STOP, notify human
        │
        confirmed (or clearly-scoped net-new work)
        │
        ▼
     tech-planner ──► planReviewStatus (label: plan-drafted)
        │
        │   human reviews the [TECH PLAN] comment, approves by
        │   dragging the card to buildStatus themselves — that's the whole gate
        ▼
     buildStatus
        │
        ▼
     engineer-build ──► branch pushed ──► qaStatus
        │                                    │
        │                                    ▼
        │                               qa-verify
        │                                 │      │
        │                              FAIL      PASS
        │                                 │      │
        └─────────── buildStatus ◄────────┘      └─► planReviewStatus + PR opened
                     (retry loop, capped                  + push notification:
                      at qa.maxCycles, then                "ready — here's how to
                      needs-human label + notify)           test/merge"
```

`planReviewStatus` is reused for two different human gates — plan approval, and final merge-readiness — deliberately: it always means "a human looks at this next." Labels + comment prefixes tell you which gate you're at, not the status name.

## Setting this up in a project

1. Install the plugin locally (path-based, until it has a real remote):
   ```bash
   claude plugin marketplace add /home/batch/claude-delivery-pipeline
   claude plugin install delivery-pipeline
   ```
2. Inside the target project, run:
   ```
   /pipeline-init
   ```
   This reads `config.schema.json`, explains every field as it asks for it (auto-detecting what it safely can — trunk branch, test/dev commands, live Jira workflow status names — and confirming the rest with you), and writes `.claude/delivery-pipeline.config.json`. That file is meant to be committed — it holds identifiers (Jira project key, branch prefix, Render service IDs, commands), never secrets. See `config.schema.json` for the full field reference and `config.example.json` for a filled-in example (Scroobious).
3. Try a sweep manually first, before scheduling anything:
   ```
   /pipeline-sweep list
   ```
   Shows every eligible ticket, what agent it would go to (or why it'd be skipped), and whether it falls inside the default cap — without touching Jira or dispatching anything. Then choose:
   ```
   /pipeline-sweep            # default: capped + prioritized (see below)
   /pipeline-sweep 3          # process the top 3 from the prioritized list, this run only
   /pipeline-sweep SCR-144,SCR-163   # process exactly these tickets, ignoring the cap
   ```
4. Schedule the sweep (separate step — `pipeline-init` only writes config, it doesn't schedule anything):
   ```
   /schedule
   ```
   and point it at `/pipeline-sweep` on whatever interval fits (a 15–30 min cron is a reasonable default). You can also just run `/pipeline-sweep` manually any time.

## Guardrails, by design

- **Never pushes to the trunk branch.** Every build lives on its own `feature/<ticket-key>` branch; a human merges.
- **Read-only against production, always**, regardless of `db.mode` — the mode only controls *how* (direct query vs. local restore), never *whether* writes are allowed. See `references/prod-research-playbook.md`.
- **Never uses a real user's account or credentials** to reproduce an issue — always a configured test/quicklogin account (`qa.testAccountStrategy`).
- **Every plan waits for explicit human approval** before a single line of code gets written — approval is just moving the Jira card, no extra tooling needed.
- **QA failures loop with a cap** (`qa.maxCycles`, default 3) — after that, the pipeline stops spending cycles on a ticket and escalates to a human with the full failure history instead of looping forever.
- **Findings need evidence.** Both `research-triage` and `qa-verify` are instructed to say "inconclusive" rather than assert a conclusion they can't back with a query, log line, or repro.

## Repo layout

```
.claude-plugin/plugin.json     # plugin manifest
config.schema.json             # every config field, documented — the source of truth
config.example.json            # filled example (Scroobious)
commands/
  pipeline-init.md             # sets up .claude/delivery-pipeline.config.json for a project
  pipeline-sweep.md            # the orchestrator — what the cron job runs
agents/
  pm-decompose.md
  research-triage.md
  tech-planner.md
  engineer-build.md
  qa-verify.md
references/
  prod-research-playbook.md    # shared DB/log/browser research rules (triage + QA)
```

## Known limitations / not yet built

- No remote yet — local marketplace path only, per design (not ready to share broadly).
- Agent `tools` are currently unrestricted (`"*"`) for all five agents to avoid under-provisioning during early iteration; worth tightening per-agent once the flows are proven out (e.g. `pm-decompose` and `tech-planner` don't need write/edit access to code at all).
- Parallel-ticket worktree isolation is specified in `pipeline-sweep.md` but hasn't been exercised under real concurrent load yet.
