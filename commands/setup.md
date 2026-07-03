---
description: Guided setup for the Framedash plugin. Use when a user installs the plugin, asks how to connect Framedash, or the MCP tools report a missing or invalid API key.
---

# Framedash setup

Walk the user through connecting the Framedash MCP server. Do the checks in
order and stop at the first one that fails, telling the user exactly how to fix
it before continuing.

## Critical rule: never handle the API key

Do NOT ask the user to paste their API key into the chat, and never echo, print,
or store it. The key is a secret that must live only in the user's environment.
Your job is to tell the user to set it themselves and then verify the connection
through the MCP tools.

## Step 1 -- Check npx is available

The MCP server runs via `npx -y @framedash/mcp-server`. Confirm `npx` is on the
user's PATH:

```bash
npx --version
```

If this fails, tell the user to install Node.js (which provides `npx`) from
https://nodejs.org and re-run setup.

## Step 2 -- Confirm the API key is set (do not read its value)

Tell the user to set `FRAMEDASH_API_KEY` in their own shell or environment
before starting Claude Code, for example:

```bash
export FRAMEDASH_API_KEY="fd_..."   # run this yourself; do not share the value
```

The key needs the `analytics:read` scope; the raw `query` tool additionally
requires `data:admin`. Keys are created (and their value shown only once) in the
Framedash dashboard.

To check that a key is present WITHOUT revealing it, you may confirm only that
the variable is non-empty. Use the check that matches the user's shell:

```bash
# bash / zsh / POSIX shells
test -n "$FRAMEDASH_API_KEY" && echo "FRAMEDASH_API_KEY is set" || echo "FRAMEDASH_API_KEY is NOT set"
```

```powershell
# PowerShell (common on Windows)
if ($env:FRAMEDASH_API_KEY) { "FRAMEDASH_API_KEY is set" } else { "FRAMEDASH_API_KEY is NOT set" }
```

Both print only a fixed status string, never the key value.

If it is not set, ask the user to set it and restart Claude Code (or run
`/reload-plugins`) so the MCP server starts with the key in its environment.

## Step 3 -- Verify the connection

Call the `list_projects` MCP tool. On success it returns the project bound to
the API key, which confirms the key is valid and the server is reachable. If it
returns an auth error, the key is missing, wrong, or lacks `analytics:read` --
send the user back to Step 2.

## Step 4 -- Optional default project

If the user works with a single project, suggest setting `FRAMEDASH_PROJECT_ID`
to that project's UUID (visible from `list_projects` output) so project-scoped
tools do not need an explicit `project_id` argument each call:

```bash
export FRAMEDASH_PROJECT_ID="<project-uuid>"
```

Without it, project-scoped tools still work but require a `project_id` argument
per call. An optional `FRAMEDASH_BASE_URL` overrides the default API host
(`https://app.framedash.dev`); most users should leave it unset.

## Done

Once `list_projects` succeeds, the user can ask Claude to explore dashboards,
run retention and funnel analyses, query events with SQL, or gate CI on perf
regressions. Point them at the `framedash-query-cookbook`, `framedash-cli`, and
`framedash-perf-ci` skills for concrete patterns.
