---
type: llm
weight: 1
---

Judge the SQL the response provides and the columns it names.

PASS requires all of the following:

1. The response gives a read-only SELECT query that reads FROM events, and the query itself begins with SELECT. A leading WITH ... AS (...) CTE is a FAIL: the validator behind the query tool and framedash query requires the first token to be SELECT and otherwise answers "Only SELECT queries are allowed", so a cohort subquery has to sit in FROM.
2. That query actually computes day-1 retention: it derives each player's first-seen day as the cohort day (for example min(toDate(timestamp)) per player_id) and counts the players from that cohort who were active the next day (for example countIf(has(active_days, first_day + 1)) or an equivalent day-1 return check), reported as a count or a rate. The first-seen day is derived over the full retained history, i.e. the per-player aggregation that computes the first-seen day carries no timestamp window bound, and any cohort window is applied to the derived first-seen day (for example WHERE first_day >= today() - 30 outside the per-player GROUP BY, or a HAVING on that aggregation). That same per-player aggregation excludes blank player_id (player_id != '' or an equivalent).
3. That query uses the real Framedash column names player_id and timestamp.
4. The response also names further columns of the Framedash events schema, drawn from: event_name, session_id, map_id, build_id, source, platform, engine_version, fps, frame_time_ms, memory_used_bytes, gpu_time_ms, game_thread_ms, render_thread_ms, position_x, position_y, position_z, camera_yaw, camera_pitch, attributes, metrics.

FAIL if any of the four is missing, or if one of these is true:

- The first-seen day is computed from a subquery whose WHERE clause bounds timestamp (for example timestamp >= today() - 30 inside the per-player GROUP BY), which dates players first seen before the window by their first in-window event.
- The response presents some other column as a real Framedash column, for example user_id, event_type, created_at, or install_date.
- The SQL is something other than a SELECT, for example INSERT, UPDATE, DELETE, or CREATE.
- The SQL itself selects or filters on project_id or tenant_id. Warning the reader that those two must never be referenced, because scoping is injected automatically, is correct guidance and is not a FAIL.
- The response gives no SQL at all.
