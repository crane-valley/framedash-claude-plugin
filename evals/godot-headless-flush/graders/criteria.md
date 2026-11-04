---
type: llm
weight: 1
---

PASS requires both of the following:

1. The cause, at least one of: Flush() only starts an async send and does not block until delivery; or Godot has no offline queue, so anything still buffered when the process exits is lost.
2. The fix that keeps the batch alive: keep the SceneTree alive after Flush() until the verbose HTTP 2xx line (typically 202) appears, and only then call Quit(). Quitting as soon as that line is observed, or waiting on a timer past the retry budget (around 40 seconds) as the upper bound, both satisfy this point; quitting on a short fixed timer does not.

Mentioning the first-flush race in addon versions before 0.1.3 (a synchronous Flush() in the same _Ready() that first touches TelemetrySDK.Instance fails with ERR_UNCONFIGURED; avoided by awaiting one ProcessFrame signal or requiring 0.1.3 or newer) strengthens a PASS but does not satisfy point 2 on its own: it fixes dispatch, not the process exiting before delivery.

The headless C# build path (godot --headless --build-solutions --quit then dotnet build, and never dotnet new sln) strengthens a PASS but does not satisfy point 2: it gets the addon to compile and says nothing about the batch surviving process exit.

FAIL if the response states only facts with no fix that keeps the batch alive, gives only a fix with no cause, is generic Godot or generic networking advice, or invents Framedash API names.
