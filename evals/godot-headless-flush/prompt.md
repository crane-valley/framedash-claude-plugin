---
name: godot-headless-flush
tags: [trigger]
runs: 2
max_turns: 12
allowed_tools: [Read, Glob, Grep, Skill]
expected_outcome: The Godot integration skill fires and the answer explains that Flush is fire-and-forget with no offline queue, and the flush-then-wait pattern or the first-flush race.
---

Our Godot 4 C# game reports Framedash events fine when I run it from the editor, but the headless CI run finishes and nothing ever arrives. What is going wrong, and how do I make the CI run reliable?
