---
name: unity-sdk-install
tags: [trigger]
runs: 2
max_turns: 12
allowed_tools: [Read, Glob, Grep, Skill]
expected_outcome: The Unity integration skill fires and the answer names com.framedash.sdk, TelemetrySDK, and the headless PlayMode / HTTP 2xx caveat.
---

We just signed up for Framedash and I need telemetry out of our Unity game, including from the batchmode job on our build machine. How do I install the SDK, start it up, and confirm the events actually reached ingest?
