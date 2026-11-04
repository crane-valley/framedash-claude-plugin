---
type: llm
weight: 1
---

PASS requires all of the following:

1. The response names the Unity package id com.framedash.sdk, or a git dependency URL ending in framedash-unity-sdk.git.
2. The response refers to the TelemetrySDK API by name, for example TelemetrySDK.Initialize, TelemetrySDK.Instance.Track, or TelemetrySDK.Instance.Flush.
3. The response explains how the batchmode run actually delivers a batch, through at least one of these:

- A plain Unity -batchmode -executeMethod run stays in Edit mode, so coroutines and UnityWebRequest never advance and a PlayMode test is needed to drive the player loop.
- The process must call Flush() and stay alive until a Framedash HTTP 2xx log line appears (typically HTTP 202), because process exit is not a success signal.

4. The response confirms the events themselves arrived, by querying the events table for a unique per-run marker event name, or by stating that an HTTP 2xx line proves a batch was delivered and not which events it carried.

Points 3 and 4 are separate requirements: delivery of a batch and confirmation of the marker event are different claims, and the prompt asks for both.

FAIL if any of the four is missing, if the response gives only generic Unity analytics advice, if it invents a package name or an API that is not listed above, or if it never mentions TelemetrySDK.
