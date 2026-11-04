---
name: mcp-retention-flow
tags: [mcp]
runs: 2
max_turns: 8
allowed_tools: [Read, Glob, Grep, Skill]
expected_outcome: Claude reaches Framedash data through the mocked MCP server -- get_retention, the query tool, or both -- and reports the mocked D7 retention of 18.4 percent.
---

Using Framedash, what was the D7 retention for my project over the last week, as of 2026-09-08? My project id is 7f3a1c2e-9b44-4d18-9a51-6c2f0d8e1b37.
