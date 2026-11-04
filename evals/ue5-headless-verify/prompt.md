---
name: ue5-headless-verify
tags: [trigger]
runs: 2
max_turns: 12
allowed_tools: [Read, Glob, Grep, Skill]
expected_outcome: The UE5 integration skill fires and the answer covers FramedashSettings or the Build.cs module dependency plus the null-RHI headless run and the Saved/Logs success line.
---

We are on Unreal Engine 5.5 and I dropped the Framedash plugin into our project, but when the build machine runs the game headless nothing shows up in the dashboard. What should I check?
