---
name: framedash-cli
description: Use the Framedash CLI (@framedash/cli, bin `framedash`) for auth verification, dashboard/retention/funnel/query analytics, builds and perf-diff, alerts, maps, and content. Use when a user asks to run Framedash commands from a shell or CI, pick an output format, handle API keys safely, or manage alerts/maps/content from the terminal.
---

# Framedash CLI

`@framedash/cli` (binary `framedash`) drives Framedash from a shell, CI, or a
coding agent. Install with `npx -y @framedash/cli <command>` or a global install.

## Commands

| Command | Purpose |
|---------|---------|
| `framedash auth` | Verify an API key (only needs the key, no project) |
| `framedash projects list` | List accessible projects (id, name, createdAt) to discover a project id |
| `framedash status` | Check project ingestion status |
| `framedash dashboard` | Fetch dashboard KPI summary (DAU, MAU, sessions, events) |
| `framedash retention` | Retention cohort data |
| `framedash funnel` | Funnel conversion data |
| `framedash query` | Run read-only SQL analytics (see the query-cookbook skill) |
| `framedash builds` | List builds seen for the project (input to `perf-diff`) |
| `framedash perf-diff` | Compare two builds and gate CI on a perf regression |
| `framedash run-profile-test` | Launch a profiling build, wait for ingest, then gate |
| `framedash alerts` | Manage alert rules and channels |
| `framedash maps` | List and manage map overlays |
| `framedash content` | Manage content registry entries |
| `framedash map-capture` | Upload captured map images |

Run `framedash <command> --help` for per-command flags.

## Environment

| Variable | Meaning |
|----------|---------|
| `FRAMEDASH_API_KEY` | API key (`analytics:read`; `query` needs `data:admin` too) |
| `FRAMEDASH_PROJECT_ID` | Default project UUID for project-scoped commands |
| `FRAMEDASH_BASE_URL` | API base URL (default `https://app.framedash.dev`) |
| `FRAMEDASH_FORMAT` | Default output format: `json`, `table`, or `csv` |

`auth` only needs the API key. Project-scoped commands such as `dashboard`,
`retention`, and `funnel` also need a project id from `--project-id` or
`FRAMEDASH_PROJECT_ID`. `framedash projects list` is the first-class discovery command: it prints each
accessible project's `id`, `name`, and `createdAt`. It requires a credential
with `analytics:read` (an API key or an OAuth login) and needs no `--project-id`;
an `events:write`-only ingest key cannot list projects. `framedash auth` also
lists the projects a credential can read while validating that credential, so it
is a fine alternative for discovering a project id.

### API-key scopes

Different commands need different key scopes. Create a key with the least scope
the task requires:

- **`analytics:read`** -- read commands: `dashboard`, `retention`, `funnel`,
  `builds`, `perf-diff`, `status`, and listing `alerts` / `maps` / `content`.
- **`resources:write`** -- mutating resources: `alerts` create/update/delete,
  `maps` delete, `map-capture` uploads, and `content` import.
- **`data:admin`** -- `query` (raw SQL) and data export/erase operations.

A single key can carry multiple scopes; a read-only integration should stay on
`analytics:read` alone.

## API-key handling (best practice)

Key precedence is: `--api-key` > `--api-key-file` (with `-` meaning stdin) >
`FRAMEDASH_API_KEY` > OAuth tokens stored by `framedash login`. Prefer a file or
piped stdin over `--api-key` so the raw key never lands in shell history or the
process list. Because `--api-key` wins, passing it alongside a file silently
overrides the file rather than acting as a fallback -- supply exactly one source.

```bash
# From a file
framedash auth --api-key-file ./framedash.key

# Piped via stdin (nothing written to disk or history)
cat ./framedash.key | framedash auth --api-key-file -

# From the environment (fine for CI where the runner injects a secret)
export FRAMEDASH_API_KEY="fd_..."
framedash status
```

Never paste a raw key on the command line where a coding agent or shell logs it.

## Verify auth first

Before running analytics, confirm the key works:

```bash
framedash auth --api-key-file ./framedash.key
```

A successful `auth` means the key is valid. If project-scoped commands still
fail afterward, the project id is missing or the key lacks a scope. Run
`framedash projects list` to find a valid project id.

## Output formats

Set `--format` per call or `FRAMEDASH_FORMAT` as a default:

```bash
framedash dashboard --format json    # full nested structure, best for agents
framedash dashboard --format csv     # flat rows for spreadsheets
framedash dashboard --format table   # human-readable terminal table
```

Caveat: for `dashboard`, `--format table` renders nested JSON values (the KPI
sub-objects) as a single blob cell, which is hard to read. Use `--format json`
(or `csv`) when you need the nested dashboard fields, and reserve `table` for
flat command output.

## Alerts: dashboard prerequisites

Creating an alert requires two UUIDs -- `--map-id` and
`--threshold-profile-id` -- that are currently discoverable only from the
Framedash dashboard; the CLI has no command to list them yet. Before running
`framedash alerts create ...`, get both IDs from the dashboard:

- the map id from the project's Maps page, and
- the threshold-profile id from the alert/threshold configuration in the
  dashboard.

Then pass them to the create command. Listing existing alerts and history works
without those IDs.

## CI performance gating

`run-profile-test` ties the SDK automated-session API to the `perf-diff` gate in
one step (launch build, wait for ingest, gate on regression). See the
`framedash-perf-ci` skill for full recipes and the exported `FRAMEDASH_*`
session contract.
