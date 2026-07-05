# Framedash plugin for Claude Code

Analyze your game telemetry with Claude Code in a one-step setup. This plugin
bundles:

- The **Framedash MCP server** (`@framedash/mcp-server`) -- 12 read-only tools
  and 4 resources for querying analytics (dashboard KPIs, retention, funnels,
  insights, heatmaps, maps, content, alerts, and raw SQL).
- A **query cookbook** skill -- the events schema plus ready-to-run SQL for DAU,
  retention, funnels, perf percentiles per build, and automated-vs-organic
  splits.
- A **CLI usage** skill -- `@framedash/cli` patterns for auth, analytics, output
  formats, and safe API-key handling.
- A **perf-CI** skill -- `run-profile-test` and `builds` + `perf-diff` recipes
  with a GitHub Actions example for build-over-build regression gating.
- A **Unity integration** skill -- installing `com.framedash.sdk`, initializing
  `TelemetrySDK`, the batchmode/PlayMode CI pattern, and verifying ingest.
- A **UE5 integration** skill -- the C++ plugin install, headless null-RHI runs,
  the authoritative `Saved/Logs` log, and the map-path/Git-Bash pitfalls.
- A **Godot integration** skill -- the C# addon install, the headless
  build-solutions fallback, the flush-then-wait pattern, and ingest verification.
- A **`/framedash:setup`** command that walks you through configuration.

## Prerequisites

- Node.js with `npx` available on your `PATH` (the MCP server runs via
  `npx -y @framedash/mcp-server`).
- A Framedash API key with the `analytics:read` scope (the raw `query` tool
  additionally needs `data:admin`). Create one in the Framedash dashboard.

## Install

Add the marketplace, then install the plugin:

```bash
claude plugin marketplace add crane-valley/framedash-claude-plugin
claude plugin install framedash@framedash
```

Then set your API key in your environment (never paste it into chat):

```bash
export FRAMEDASH_API_KEY="fd_..."           # required
export FRAMEDASH_PROJECT_ID="<project-uuid>" # optional default project
```

Run `/framedash:setup` inside Claude Code for a guided walkthrough, or restart
the session so the MCP server picks up the environment.

## MCP-only fallback

If you only want the MCP tools without the plugin, register the server directly:

```bash
claude mcp add framedash -- npx -y @framedash/mcp-server
```

The server reads `FRAMEDASH_API_KEY` (required), `FRAMEDASH_PROJECT_ID`
(optional), and `FRAMEDASH_BASE_URL` (optional, default
`https://app.framedash.dev`) from the environment.

## Links

- Documentation: https://docs.framedash.dev
- Product: https://framedash.dev

## License

MIT
