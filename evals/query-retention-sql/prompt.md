---
name: query-retention-sql
tags: [trigger]
runs: 2
max_turns: 12
allowed_tools: [Read, Glob, Grep, Skill]
expected_outcome: The query cookbook skill fires and the answer is SELECT-only SQL over the events table using real schema columns.
---

I want to pull day-1 retention out of my Framedash data with raw SQL. What does that query look like, and which columns can I actually select from?
