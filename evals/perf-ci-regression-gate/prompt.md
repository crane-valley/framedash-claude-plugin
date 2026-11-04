---
name: perf-ci-regression-gate
tags: [trigger]
runs: 2
max_turns: 12
allowed_tools: [Read, Glob, Grep, Skill]
expected_outcome: The perf-CI skill fires and the answer gates on perf-diff or run-profile-test with a baseline build and a threshold or fail-on-regression flag.
---

I want our CI to fail a pull request when the game's frame time gets measurably worse than the last known-good build. We already send Framedash telemetry from the game. How do I wire that gate up?
