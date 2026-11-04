---
type: llm
weight: 1
---

PASS requires all of the following:

1. The response answers the Unity question directly.
2. It explains that Awake runs first, when the script instance is loaded, and that Start runs later, just before the first frame update.
3. It gives at least one practical consequence, for example using Awake for an object's own initialization and Start for wiring up references to other objects, or noting that Awake still runs on a disabled component while Start does not.

FAIL if the response brings up Framedash, game telemetry, analytics, or an SDK, or if it does not answer the question.
