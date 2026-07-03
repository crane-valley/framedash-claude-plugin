---
name: framedash-query-cookbook
description: Reference for the Framedash events schema and a cookbook of read-only SQL for game telemetry (DAU, retention, funnels, perf percentiles per build, automated-vs-organic splits). Use when a user wants to query Framedash data with the MCP `query` tool or `framedash query`, asks what columns exist, or needs a starting SQL query for an analytics question.
---

# Framedash query cookbook

Framedash exposes a read-only SQL endpoint (the MCP `query` tool and
`framedash query`) over your project's ClickHouse event data. Queries are
validated, rewritten to enforce tenant/project isolation, and run against a
read-only connection.

## Response shape

Results come back as JSON:

```json
{ "rows": [ { "col": "value" }, ... ], "rowCount": 123 }
```

`rowCount` is the number of returned rows, bounded by `limit`. The two access
paths have different caps:

- **MCP `query` tool**: default 100, **max 1000** rows per call.
- **REST `/v1/query` endpoint (and `framedash query` CLI)**: default 1000,
  **max 10000** rows per call.

If you need more than 1000 rows, use the REST endpoint or the CLI rather than
the MCP tool; for larger extracts, page with `LIMIT ... OFFSET ...` or narrow the
`timestamp` range.

## Query rules (enforced by the validator)

- `SELECT` only. No `INSERT/UPDATE/DELETE/DROP/ALTER/CREATE/TRUNCATE`, no
  `SYSTEM/OPTIMIZE/KILL`, no `UNION/EXCEPT/INTERSECT`.
- Only two tables may appear in `FROM`/`JOIN`: **`events`** (raw events) and
  **`daily_sessions_project_mv`** (pre-aggregated daily sessions). Write plain
  `FROM events`; isolation filters are injected for you.
- **Do not reference `tenant_id` or `project_id`** in your SQL -- they are
  auto-scoped, and referencing them directly is rejected.
- String literals use single quotes only. Double quotes and backticks are
  rejected (they are identifier syntax that would bypass validation).
- No `SETTINGS`, `FORMAT`, or `GLOBAL` clauses. `IN (SELECT ...)` and
  `IN ('a','b')` are fine; `IN some_table` is not.
- If you omit a top-level `LIMIT`, one is appended automatically.

## Events schema (`events` table)

Columns you can select and filter on:

| Column | Type | Notes |
|--------|------|-------|
| `event_name` | String | e.g. `player.death`, `level.start`, `perf_heartbeat` |
| `timestamp` | DateTime64(6) | Event time (UTC) |
| `session_id` | String | Play-session id |
| `player_id` | String | Stable player/device id |
| `position_x`, `position_y`, `position_z` | Float32 | World coordinates |
| `map_id` | String | Map/level identifier |
| `fps` | Float32 | Frames per second (perf events) |
| `frame_time_ms` | Float32 | Frame time in ms (perf events) |
| `memory_used_bytes` | Int64 | Memory in bytes (perf events) |
| `gpu_time_ms` | Float32 | GPU time in ms (perf events) |
| `game_thread_ms` | Float32 | Game-thread time in ms |
| `render_thread_ms` | Float32 | Render-thread time in ms |
| `camera_yaw`, `camera_pitch` | Nullable(Float32) | Camera orientation |
| `source` | String | `automated` for CI/auto-profiling sessions; player sessions carry the organic source tag |
| `build_id` | String | First-class build identifier (perf-diff candidate) |
| `platform` | String | e.g. `windows`, `android` |
| `engine_version` | String | Engine/build version string |
| `attributes` | Map(String, String) | Custom string attributes |
| `metrics` | Map(String, Float64) | Custom numeric metrics |

Map access uses bracket syntax: `attributes['ci.branch']`,
`metrics['score']`.

### Key conventions

- The automatic performance event is **`perf_heartbeat`** (filter perf queries
  with `WHERE event_name = 'perf_heartbeat'`).
- Automated sessions (CI auto-profiling) have **`source = 'automated'`**.
- CI metadata rides in `attributes`: **`attributes['ci.branch']`**,
  **`attributes['ci.commit']`**, **`attributes['ci.scenario']`**, alongside the
  first-class `build_id` column.

## Cookbook

### 1. DAU today

```sql
SELECT uniqExact(player_id) AS dau
FROM events
WHERE timestamp >= today()
```

### 2. DAU per day, last 14 days

```sql
SELECT toDate(timestamp) AS day, uniqExact(player_id) AS dau
FROM events
WHERE timestamp >= today() - 14
GROUP BY day
ORDER BY day
```

### 3. Event volume by name (last 7 days)

```sql
SELECT event_name, count() AS events, uniqExact(session_id) AS sessions
FROM events
WHERE timestamp >= today() - 7
GROUP BY event_name
ORDER BY events DESC
```

### 4. Sessions and events per day

```sql
SELECT toDate(timestamp) AS day,
       uniqExact(session_id) AS sessions,
       count() AS events
FROM events
WHERE timestamp >= today() - 30
GROUP BY day
ORDER BY day
```

### 5. Funnel-style step conversion

Count distinct players who reached each named step (order the steps as you like
in the client):

```sql
SELECT event_name, uniqExact(player_id) AS players
FROM events
WHERE event_name IN ('tutorial.start', 'level.start', 'level.complete', 'purchase')
  AND timestamp >= today() - 7
GROUP BY event_name
ORDER BY players DESC
```

### 6. Day-1 retention (cohort by first-seen day)

Compute each player's first-seen day and the set of days they were active in one
subquery, then cohort by first day and check for activity the next day. This
uses only aggregate functions (no window functions), so it runs as-is:

```sql
SELECT first_day,
       count() AS cohort,
       countIf(has(active_days, first_day + 1)) AS retained_d1,
       round(retained_d1 / cohort, 3) AS d1_rate
FROM (
  SELECT player_id,
         min(toDate(timestamp)) AS first_day,
         groupUniqArray(toDate(timestamp)) AS active_days
  FROM events
  WHERE timestamp >= today() - 30
  GROUP BY player_id
)
GROUP BY first_day
ORDER BY first_day
```

The inner query yields one row per player, so `count()` in the outer query is
the cohort size. Extend to day-N retention by checking
`has(active_days, first_day + N)`.

### 7. Perf percentiles per build (`perf_heartbeat`)

```sql
SELECT build_id,
       count() AS samples,
       round(quantile(0.50)(fps), 1) AS fps_p50,
       round(quantile(0.95)(frame_time_ms), 2) AS frametime_p95_ms,
       round(quantile(0.99)(frame_time_ms), 2) AS frametime_p99_ms,
       round(quantile(0.95)(gpu_time_ms), 2) AS gpu_p95_ms
FROM events
WHERE event_name = 'perf_heartbeat'
  AND timestamp >= today() - 14
GROUP BY build_id
ORDER BY samples DESC
```

### 8. Perf by CI branch (build-over-build comparison)

```sql
SELECT attributes['ci.branch'] AS branch,
       build_id,
       round(quantile(0.95)(frame_time_ms), 2) AS frametime_p95_ms
FROM events
WHERE event_name = 'perf_heartbeat'
  AND source = 'automated'
  AND timestamp >= today() - 14
GROUP BY branch, build_id
ORDER BY branch, frametime_p95_ms DESC
```

### 9. Automated vs organic split

```sql
SELECT source,
       uniqExact(session_id) AS sessions,
       count() AS events
FROM events
WHERE timestamp >= today() - 7
GROUP BY source
ORDER BY events DESC
```

### 10. Top maps by activity

```sql
SELECT map_id,
       uniqExact(player_id) AS players,
       count() AS events
FROM events
WHERE timestamp >= today() - 7
  AND map_id != ''
GROUP BY map_id
ORDER BY events DESC
LIMIT 20
```

## Tips

- Prefer `uniqExact(...)` for exact distinct counts on player/session ids; use
  `uniq(...)` for a faster approximate count on very large ranges.
- Always bound `timestamp` so queries prune partitions (the table is partitioned
  by day) and stay fast.
- For perf-oriented questions, filter `event_name = 'perf_heartbeat'` first; not
  every event carries `fps`/`frame_time_ms`.
