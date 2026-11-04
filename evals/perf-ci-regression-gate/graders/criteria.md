---
type: llm
weight: 1
---

PASS requires all of the following:

1. The response gates CI with a Framedash CLI perf command, either framedash perf-diff or framedash run-profile-test.
2. The command compares against a known-good build with --baseline and carries --fail-on-regression. Without --fail-on-regression the command reports the comparison and still exits 0, so the job would never fail.
3. The candidate side of the comparison is supplied: --candidate <build id> for perf-diff, which requires both build ids, or the --command that launches the profiling build for run-profile-test.

Any one of these supporting points strengthens a PASS but is not required on its own: --threshold <pct> to tolerate a regression percentage (default 0 fails on any worsening); the candidate build id reaches the game only through the exported FRAMEDASH_BUILD_ID and must not be hardcoded in the build; the gate needs an analytics:read key while the game needs a separate events:write ingest key; framedash builds lists the build ids that perf-diff compares.

FAIL if any of the three requirements is missing, if the response proposes a home-grown frame-time comparison, if it invents CLI commands or flags that are not Framedash's, or if it never names a framedash command.
