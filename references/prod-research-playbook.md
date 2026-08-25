# Production research playbook

Shared by the `research-triage` and `qa-verify` agents whenever a claim needs to be checked against real data or real logs instead of assumed. Read this before running any query.

## Ground rule

**Read-only, always.** Never run INSERT/UPDATE/DELETE/DDL against production, in any `db.mode`. If confirming something requires changing data, that's out of scope for this agent — stop and describe what a human would need to do instead.

## Picking a method, based on `db.mode`

- **`direct-readonly`**: Run SELECT-only queries straight against `db.renderPostgresId` via `mcp__render__query_render_postgres`. This is the default and covers almost everything: does a row exist, what are its field values, how many rows match a pattern, etc.
- **`single-record-only`**: Direct queries are allowed only when the query targets one specific record (one user id, one space id, one ticket id — a `WHERE` clause pinned to a specific primary/foreign key, not a broad scan). Anything else (aggregate counts, pattern analysis across many rows, anything that needs app code to run against the data) requires a local restore — follow `db.restoreRunbookPath`.
- **`local-restore-only`**: Never query prod directly. Always restore a dump locally first per `db.restoreRunbookPath`, then query the local copy.

Regardless of mode, if a question needs you to run actual application code against realistic data (not just inspect rows), that always means a local restore — direct queries only ever return rows, they don't exercise app logic.

## Logs

Use `mcp__render__list_logs` (and `mcp__render__get_deploy` / `mcp__render__list_deploys` if the timing matters — e.g. "did this start after a specific deploy") scoped to the reported time window. Cross-reference the timestamp in the ticket/screenshot against log entries, not just "recent errors in general."

## Browser reproduction

When reproducing a reported symptom in a live environment (local dev or staging) with Chrome MCP:

- **Never use the real reporting user's account or credentials.** Use the project's `qa.testAccountStrategy` to get a safe test/quicklogin account instead.
- If the report is user- or record-specific (e.g. "this user's profile"), reproduce the *shape* of their situation with a test account/record in a similar state, rather than logging in as them.
- Capture browser console output and network errors, not just a visual screenshot — the visual symptom and the actual error are often different things.

## Writing up findings

Every findings comment should let a reader (human or the next agent) tell the difference between "I confirmed this with evidence" and "I couldn't fully confirm this." Always include:

- What you checked and how (query run, logs viewed, repro steps taken) — enough for someone to redo it.
- What you found, in plain terms.
- Your conclusion: confirmed root cause / not reproducible / symptom has a different cause than reported — and why.

Never write a conclusion you don't have evidence for. "Probably X" without a query or log to back it is not a finding — either go get the evidence or say plainly that you couldn't confirm it.
