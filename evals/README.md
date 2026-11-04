# Framedash plugin eval suite

Behavioral tests for this plugin, run by `claude plugin eval` (Claude Code
2.1.269 or newer). Each case is a prompt a user might actually type plus a set
of graders; the runner scores the reply with the plugin loaded and again with no
plugin loaded, and reports the difference.

## Run it

From the plugin root (`integrations/claude-plugin`):

```
claude plugin eval . --no-publish
```

Faster while iterating on one case:

```
claude plugin eval . --runs 1 --ablation none --case unity-sdk-install --no-publish
```

`--ablation none` drops the no-plugin arm, so the run costs half as much but
tells you nothing about what the plugin contributed. Use the full form before
drawing conclusions.

Useful additions: `--tag trigger` (or `negative`, `mcp`) to filter,
`-j 2` to run two agent runs at once, `--max-cost-usd 15` as a budget ceiling,
`--threshold 0.8` to set the pass bar, and `--json <path>.json` for a machine
readable result.

## Windows

Native Windows has no sandbox backend, so any case that grants `Bash`, `Write`,
or `Edit` is refused and scores 0. Every case here sticks to the read-only set
(`Read`, `Glob`, `Grep`, `Skill`), and the one case that needs live data gets it
from MCP mocks, so the suite runs on Windows as-is. Add shell-granting cases only
under WSL2.

## Cases

- `unity-sdk-install`, `ue5-headless-verify`, `godot-headless-flush`,
  `perf-ci-regression-gate`, `query-retention-sql`, `cli-auth-key-handling`
  (tag `trigger`): one realistic prompt per skill. Each asserts the intended
  skill fired, that the closest confusable siblings did not (the two other
  engine skills for the engine cases; one analytics sibling for the analytics
  cases, since `framedash-perf-ci` legitimately cross-references
  `framedash-cli`), and that the answer carries substance only that skill
  teaches.
- `negative-generic-unity` (tag `negative`): a plain Unity question with no
  Framedash context. Asserts that no `framedash-*` skill fires.
- `mcp-retention-flow` (tag `mcp`): an analytics question that needs real data.
  Answered from the mocks in `mcp-retention-flow/mocks/framedash/`, so no
  `FRAMEDASH_API_KEY` and no running server are involved.

## Mocks

`mcp-retention-flow/mocks/framedash/` stands in for the `framedash` MCP server
declared in `.mcp.json`. Only tools with a mock file are visible to the agent,
and the real server never starts. `_tools.json` carries the real tool
descriptions and input schemas so the agent picks tools the way it would against
the live server; keep it in step with the server when its tool surface changes.
`get_retention` and `query` guard their input with `expect:`, so a call that
drops the project id the prompt supplied, asks `get_retention` for a window
other than the prompt's week, or sends `query` SQL that never reads `events`
aborts the case with score 0 instead of being answered from the fixture. The
`query` guard checks no date bound, so a wider SQL window is answered from the
fixture and left to the response grader. The `sql` guard stops there because
`expect:` regexes take a restricted dialect -- literals, `.`, escapes,
character classes and quantifiers on single atoms, with optional `^`/`$`, but
no groups, alternation or lookaround -- so "reads `events`" is expressible and
"is retention-shaped" is not; a pattern using alternation fails the whole case
at mock-load time. `get_retention`'s body mirrors the live tool, which returns
the `RetentionCohort[]` array of the REST retention endpoint (`cohort_date`,
`cohort_size`, `d1`/`d7`/`d30` counts and their `_pct` fields, where the
`_pct` values are 0-100 to two decimals).

The mocks are deliberately scoped to that one case rather than kept in a
suite-wide `evals/mocks/`. A suite-wide mock is offered to every case, and a
`fixed` mock answers with its body whatever input it is handed: the perf-CI case
called `query`, got the retention body back, correctly judged it inconsistent
with the SQL it had sent, and spent the rest of its turn budget re-checking the
tool instead of answering. Add new mocks under the case that needs them.

`query-cookbook-skill-indicator` in the MCP case is an indicator, not a scored
grader: it reports whether the `framedash-query-cookbook` skill was invoked,
and nothing about which tool answered. In practice Claude answers through
`get_retention` without consulting the cookbook, so the indicator stays red.
Both routes are correct, so read it as "was the cookbook consulted", not as a
defect.

## Scope of graders

Graders require only what the plugin's `SKILL.md` files explicitly teach, at the
level the case targets. A case measures whether the skill steers Claude to the
guidance it ships, not whether that guidance is complete.

Fixed mocks validate the shape of the arguments a tool was called with, not the
semantic correctness of the SQL inside them: a `fixed` mock answers with its body
whatever input passes its `expect:` guard, so the guard is a shape check and the
rubric judges the answer.

Production-semantics gaps noticed while reviewing graders are recorded as skill
follow-ups in the repo's `PLANS.md`, not encoded into a grader ahead of the skill.
A grader that demands more than the skill teaches fails on the plugin's own
correct output and measures nothing.

## Results

Each run writes `results/<timestamp>/aggregate-result.json` and a self-contained
`report.html`. `results/` is git-ignored; do not commit it.
