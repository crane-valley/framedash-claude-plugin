---
name: framedash-unity-integration
description: Integrate the Framedash Unity SDK (com.framedash.sdk) into a Unity project and verify telemetry reaches ingest, including headless/batchmode CI. Use when a user wants to install the Unity SDK, initialize TelemetrySDK, send performance or custom events, run a PlayMode smoke test in CI, handle API keys safely, or debug why Unity events are not arriving.
---

# Framedash Unity integration

Install the Framedash Unity SDK, wire up `TelemetrySDK`, and prove events reach
ingest. The SDK auto-collects FPS, frame time, and memory and batches them with
your manual events. This skill covers the install, the verify recipe, and the
CI/headless pitfalls the tutorial glosses over.

Full guide: https://docs.framedash.dev/en/sdk/unity/
Troubleshooting: https://docs.framedash.dev/en/sdk/troubleshooting/

## Requirements

- Unity 2022.3 or newer.
- .NET Standard 2.1 / .NET Framework 4.x.
- An ingest API key with the `events:write` scope (create it in the dashboard).

## Install

Add the package as a git dependency. For CI or scripted installs, edit
`Packages/manifest.json` directly; the package name is `com.framedash.sdk`.

```json
{
  "dependencies": {
    "com.framedash.sdk": "https://github.com/crane-valley/framedash-unity-sdk.git#v0.1.4"
  }
}
```

Pin a version tag (`#v0.1.4`) rather than tracking the default branch, so a CI
build is reproducible.

If a batch-mode run fails right after adding the git package with
`Package Manager Cancelled resolving packages`, the cause is a stale
`Temp/UnityLockfile` left over from project creation: a still-locked project
cannot resolve the new package. Delete `Temp/UnityLockfile` and retry the run.

## Initialize

Call `Initialize` once at startup from a persistent `MonoBehaviour`. The full
declaration (a static method that returns the `TelemetrySDK` instance) is:

```csharp
public static TelemetrySDK Initialize(
    string apiKey = null,
    string endpointUrl = null,
    string buildId = null,
    string playerId = null,
    bool enableOfflineQueue = true)
```

- `endpointUrl` defaults to `https://ingest.framedash.dev/v1/events`; pass it only
  for a local or self-hosted ingest.
- `buildId` is the perf-diff build identifier (commonly `Application.version`).
- `enableOfflineQueue` defaults to `true` (see the offline-queue caveat below).

Key resolution order is: a non-empty `apiKey` argument, then the Inspector field,
then the `FRAMEDASH_API_KEY` environment variable. On desktop and CI you can call
`TelemetrySDK.Initialize()` with no argument and let the env var supply the key.
The env var is NOT available in shipped mobile, console, or WebGL builds; pass
the key from the Inspector or a platform secret store on those targets.

## API-key safety

- Never hardcode a live key in a committed script or scene, and never print it to
  the Console or a log.
- In CI, inject the key as `FRAMEDASH_API_KEY` and call `Initialize()` with no
  argument, so the raw key never appears in a tracked file.
- If a key was committed by mistake, scrub it from history and rotate it in the
  dashboard.

## Auto-collected and manual events

- `session_start` is emitted at initialization.
- `perf_heartbeat` is emitted every 10 seconds with FPS, frame time, and memory.
- Manual events go through `TelemetrySDK.Instance.Track(...)`.

Auto events share the same batch as your manual events, so a verbose
`Flushed N events` line can report more events than you tracked. That is expected,
not duplication.

## Flush policy

Batches flush on whichever comes first:

| Trigger | Default |
|---------|---------|
| Time interval | 30 seconds |
| Event count | 100 events |
| Estimated payload size | 102400 bytes |

A single `Track()` near process end will NOT hit any threshold. Call
`TelemetrySDK.Instance.Flush()` explicitly and keep the process alive until the
HTTP 2xx (typically 202) log line appears. Process exit is not a success signal;
the 2xx line is.

## Headless / CI

A bare `Unity.exe -batchmode -executeMethod ...` runs in Edit mode. Coroutines
and `UnityWebRequest` do not advance there, so events are buffered but never sent.
Drive the player loop with a PlayMode test (Unity Test Framework) instead.

Put the PlayMode test in its own folder with a test assembly definition. The
recommended layout is:

```
Assets/Tests/PlayMode/
  Framedash.Tests.PlayMode.asmdef   # Editor-only; references Framedash.Runtime + the test runner
  TelemetrySmokeTest.cs
```

The test `.asmdef` references `Framedash.Runtime` plus `UnityEngine.TestRunner`
and `UnityEditor.TestRunner`, with `nunit.framework.dll` as a precompiled
reference (set `overrideReferences` true). Gate it to the Editor with
`"includePlatforms": ["Editor"]` -- the documented headless run is an in-Editor
batch-mode PlayMode run, and the Editor gate keeps `UnityEditor.TestRunner`
(an editor-only assembly) from breaking standalone player builds. That scopes
the test to the test framework and lets `-testPlatform PlayMode` discover it.
The test file:

```csharp
using System.Collections;
using System.Text.RegularExpressions;
using Framedash;
using NUnit.Framework;
using UnityEngine;
using UnityEngine.TestTools;

public sealed class TelemetrySmokeTest
{
    // The verbose success line: "[Framedash] Flushed N events (HTTP 202)."
    // Any HTTP 2xx counts as delivered.
    static readonly Regex Delivered =
        new Regex(@"\[Framedash\] Flushed \d+ events? \(HTTP 2\d\d\)");

    [UnityTest]
    public IEnumerator SendsAMarkerEvent()
    {
        var delivered = false;
        void OnLog(string msg, string stack, LogType type)
        {
            if (Delivered.IsMatch(msg)) delivered = true;
        }
        Application.logMessageReceived += OnLog;
        try
        {
            // Initialize FIRST: the first access to Instance auto-inits with
            // defaults, so touching Instance before Initialize would discard the
            // buildId. Then toggle VerboseLogging (it propagates to the live
            // transport). key comes from FRAMEDASH_API_KEY in CI; do not hardcode.
            TelemetrySDK.Initialize(buildId: "ci-smoke");
            TelemetrySDK.Instance.VerboseLogging = true;
            // If the project's global sampling rate is < 1.0, the marker could be
            // sampled out while the automatic session_start still produces a 2xx
            // line -- making the log assertion pass without the marker. Force this
            // event's rate to 1.0 so the marker is always kept.
            TelemetrySDK.Instance.SetEventSamplingRate("run_1720000000_ab12", 1f);
            TelemetrySDK.Instance.Track(
                eventName: "run_1720000000_ab12", mapId: "ci", position: default);
            TelemetrySDK.Instance.Flush();

            // Poll for the 2xx line, do NOT pass on a blind sleep. The deadline
            // sits past the worst retry path (up to 5 attempts at a 10s request
            // timeout with 1/2/4/8s backoff, plus a direct-socket fallback); the
            // loop exits early the instant the line is seen.
            var deadline = Time.realtimeSinceStartup + 90f;
            while (!delivered && Time.realtimeSinceStartup < deadline)
                yield return null;
            Assert.IsTrue(delivered, "No [Framedash] Flushed ... (HTTP 2xx) line seen");
        }
        finally
        {
            Application.logMessageReceived -= OnLog;
        }
    }
}
```

Run it from CI:

```
Unity.exe -batchmode -nographics -projectPath <path> -runTests \
  -testPlatform PlayMode -testResults <path>/results.xml -logFile -
```

Enable `VerboseLogging` after `Initialize`; a successful flush then logs a 2xx
line such as `[Framedash] Flushed N events (HTTP 202)` (any HTTP 2xx is treated as
success and the actual status code is logged). A 2xx line confirms a batch was
delivered but not which events it carried; the definitive check that the marker
itself landed is the query in Verify ingest below.

## Offline queue

With the offline queue enabled (default), events are written to disk on a
graceful shutdown OR a transient send failure, and flushed on the next
initialization once the player loop is running. A hard kill (for example a CI
timeout kill) does NOT write the queue, so any buffered events are lost. In CI,
do not rely on the queue: wait for the HTTP 2xx (typically 202) line before you
stop the process. With `enableOfflineQueue: false`, unflushed events are dropped.

## Verify ingest

Pre-existing data makes "data exists" meaningless. Use a UNIQUE per-run marker
event name, for example `run_<unix_ts>_<random>`, and check that a row with that
exact name landed today. `framedash query` needs a project id
(`--project-id <id>` or `FRAMEDASH_PROJECT_ID`) and a key with the `data:admin`
scope (an ingest `events:write` key cannot read), following the query-cookbook
skill. If you do not know the project id, run `framedash projects list` to print
each accessible project's id:

```bash
framedash query --api-key-file full.key --project-id <project-uuid> \
  "SELECT count() FROM events \
   WHERE event_name = 'run_1720000000_ab12' AND timestamp >= today()"
```

A count of 1 or more confirms the run reached ingest. If it is 0, re-read the
troubleshooting page: check the verbose HTTP status, confirm the PlayMode loop
actually ran, and confirm you queried the same project the key writes to.
