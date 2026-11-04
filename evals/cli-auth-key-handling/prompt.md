---
name: cli-auth-key-handling
tags: [trigger]
runs: 2
max_turns: 12
allowed_tools: [Read, Glob, Grep, Skill]
expected_outcome: The CLI skill fires and the answer uses a concrete framedash subcommand with a key-file or stdin key path rather than a raw --api-key argument.
---

I have a Framedash API key sitting on our build machine. From the terminal, what is the safest way to check that the key actually works without leaking it into shell history, and how do I get the dashboard numbers back as JSON?
