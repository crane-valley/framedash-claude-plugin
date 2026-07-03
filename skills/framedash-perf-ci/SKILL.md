---
name: framedash-perf-ci
description: Recipes for gating CI on game performance regressions with the Framedash CLI -- run-profile-test end-to-end, the builds + perf-diff two-step, threshold/baseline flags, a GitHub Actions example, and the exported FRAMEDASH_* automated-session env contract. Use when a user wants to detect frame-time/memory/GPU regressions build-over-build, wire a perf gate into CI, or understand how the CLI and SDK cooperate on automated profiling runs.
---

# Framedash perf-diff CI

Framedash gates CI on build-over-build performance regressions. A profiling
build emits `perf_heartbeat` events tagged with a `build_id`; the CLI compares a
candidate build against a known-good baseline and fails the job on a regression.

There are two entry points:

- `framedash run-profile-test` -- end-to-end: launch the build, wait for its
  perf data to ingest, then gate.
- `framedash builds` + `framedash perf-diff` -- a manual two-step when your perf
  data is already uploaded (e.g. a separate CI stage did the profiling run).

## The FRAMEDASH_* session contract

`run-profile-test` exports these to the launched command. An SDK that calls
`BeginAutomatedSessionFromEnvironment()` stamps them onto every event with no
in-game code change:

| Env var | Becomes |
|---------|---------|
| `FRAMEDASH_BUILD_ID` | the candidate `build_id` (first-class column) |
| `FRAMEDASH_GIT_BRANCH` | `ci.branch` attribute |
| `FRAMEDASH_GIT_COMMIT` | `ci.commit` attribute |
| `FRAMEDASH_TEST_SCENARIO` | `ci.scenario` attribute |

`FRAMEDASH_PROJECT_ID` / `FRAMEDASH_BASE_URL` are forwarded too so the build
targets the same project the gate queries.

Important: the build id reaches the game ONLY through the exported
`FRAMEDASH_BUILD_ID`. A build that hardcodes its SDK build id will never match
the awaited build, and the ingest wait will time out. Do not hardcode it.

## Keys: gate key vs ingest key

The gate calls the analytics API and needs an `analytics:read` key. The game
needs a separate `events:write` ingest key. Pass the gate key with
`--api-key` / `--api-key-file`, and keep the game's ingest key in the launched
command's own `FRAMEDASH_API_KEY`, so the gate key is not inherited as the
game's key.

## Recipe 1 -- run-profile-test (end-to-end)

```bash
framedash run-profile-test \
  --command "./Build/Game.exe -nullrhi -ExecCmds='Automation RunTest Perf'" \
  --scenario nightly \
  --baseline "$BASE_SHA" \
  --threshold 5 \
  --fail-on-regression \
  --api-key-file ./gate.key
```

What each part does:

- `--command` launches your game/profiling command through the shell.
- `--build-id <id>` sets the candidate build id (defaults to `--commit`, else
  git HEAD). If a job may re-run the same commit, give each run a unique id such
  as `--build-id "$GIT_SHA-$RUN_ID"` so the comparison is not diluted by a
  previous run's samples.
- `--branch` / `--commit` / `--scenario` populate the CI metadata (default to
  git values; scenario has no default).
- `--command-timeout <s>` bounds the launched process (default 1800s, `0`
  disables). On timeout the whole process tree is killed and the run fails
  WITHOUT gating, so a non-self-quitting build cannot hang CI forever.
- Ingest wait: `--ingest-timeout <s>` (default 180) and `--poll-interval <s>`
  (default 5) control how long the CLI waits for the build's perf-event count to
  grow past its pre-run value. `--skip-wait` skips the wait (e.g. the SDK
  flushed synchronously).
- Gate (runs only when `--baseline` is set): `--baseline <id>` is the known-good
  build; `--metric frame_time|memory|gpu_time` restricts to one metric (default
  all); `--threshold <pct>` tolerates a regression up to that percent (default
  0 = any worsening fails); `--fail-on-regression` exits 1 on a breach (requires
  `--baseline`); `--days 7|14|30|90` sets the comparison window (default 30);
  `--map` / `--platform` restrict the comparison.

## Recipe 2 -- builds + perf-diff (two-step)

When profiling already ran and the data is ingested, list builds then compare:

```bash
# See which build_ids exist for the project
framedash builds --api-key-file ./gate.key

# Compare a candidate against a baseline and gate
framedash perf-diff \
  --baseline "$BASE_SHA" \
  --candidate "$GITHUB_SHA" \
  --threshold 5 \
  --fail-on-regression \
  --api-key-file ./gate.key
```

`perf-diff` requires `--baseline` and `--candidate` (distinct build ids) and
accepts the same `--metric`, `--threshold`, `--days`, `--map`, `--platform`, and
`--fail-on-regression` flags as the gate in Recipe 1.

## GitHub Actions example

```yaml
name: perf-gate
on: [pull_request]

jobs:
  perf:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # need the base SHA for --baseline

      - name: Run profiling build and gate on regression
        env:
          # gate key (analytics:read) -- injected as a repo secret
          FRAMEDASH_GATE_KEY: ${{ secrets.FRAMEDASH_ANALYTICS_KEY }}
          FRAMEDASH_PROJECT_ID: ${{ vars.FRAMEDASH_PROJECT_ID }}
          # ingest key (events:write) for the game process itself
          FRAMEDASH_API_KEY: ${{ secrets.FRAMEDASH_INGEST_KEY }}
        run: |
          # mktemp + trap so the key file is ALWAYS removed, even though
          # run-profile-test --fail-on-regression exits nonzero under `set -e`.
          gate_key="$(mktemp)"
          trap 'rm -f "$gate_key"' EXIT
          printf '%s' "$FRAMEDASH_GATE_KEY" > "$gate_key"
          npx -y @framedash/cli run-profile-test \
            --command "./Build/Game.sh -nullrhi -ExecCmds='Automation RunTest Perf'" \
            --scenario ci \
            --build-id "${GITHUB_SHA}-${GITHUB_RUN_ID}" \
            --baseline "${{ github.event.pull_request.base.sha }}" \
            --threshold 5 \
            --fail-on-regression \
            --api-key-file "$gate_key"
```

Notes:

- The gate key is written to a temp file and passed with `--api-key-file` so the
  raw key is not visible in the process list; the `trap ... EXIT` removes it even
  when the gate fails the job (a plain `rm` after the command is unreachable on a
  nonzero exit under `set -e`).
- Per-SDK ingest key: the `FRAMEDASH_API_KEY` env var above is read as the game's
  ingest key ONLY by the Unity SDK (its `ApiKeyResolver` env fallback). UE5 reads
  the ingest key from its `FramedashSettings` (or the `InitializeTelemetry` call),
  and Godot from the argument to `TelemetrySDK.Initialize(...)` -- neither has an
  env-var resolver. On UE5/Godot, wire the key into the build's own SDK settings
  (you can still source it from `${{ secrets.FRAMEDASH_INGEST_KEY }}` in the step
  and pass it through your project's settings mechanism).
- `--threshold 5` ignores up to 5% noise; drop it (default 0) to fail on any
  worsening.
- Use `framedash builds` / `framedash perf-diff` instead if a separate job
  already produced the perf data.
