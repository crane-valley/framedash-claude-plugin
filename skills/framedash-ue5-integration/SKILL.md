---
name: framedash-ue5-integration
description: Integrate the Framedash Unreal Engine 5 SDK (C++ plugin) into a UE5 project and verify telemetry reaches ingest from a headless -game run in CI. Use when a user wants to install the UE5 plugin, configure FramedashSettings, initialize the subsystem, send events, run a null-RHI headless verification, read the authoritative Saved/Logs log, handle the map-path and Git-Bash pitfalls, or debug why UE5 events are not arriving.
---

# Framedash UE5 integration

Install the Framedash UE5 SDK (a C++ source plugin), enable it, and prove events
reach ingest from a headless run. The SDK auto-collects FPS, frame time, GPU
time, and memory once the subsystem initializes. This skill focuses on the
install steps that break silently and the headless-CI recipe.

Full guide: https://docs.framedash.dev/en/sdk/unreal/
Troubleshooting: https://docs.framedash.dev/en/sdk/troubleshooting/

## Requirements

- Unreal Engine 5.3 or newer (build-verified through 5.7; the plugin sets no
  `EngineVersion`, so newer engines compile too).
- A C++ project. Blueprint-only projects cannot compile the source plugin.
- An ingest API key with the `events:write` scope.

## Install (three steps, all required)

1. Put the plugin at `Plugins/Framedash` so `Plugins/Framedash/Framedash.uplugin`
   exists. The release zip ALREADY contains a top-level `Framedash/` folder, so
   extract it into `Plugins/` (NOT into `Plugins/Framedash`, which would nest it
   as `Plugins/Framedash/Framedash/Framedash.uplugin` and UE would never discover
   the plugin). If you cloned the repo instead, copy the repo contents into
   `Plugins/Framedash`.
2. Enable the plugin in the `.uproject`:

```json
{ "Plugins": [ { "Name": "Framedash", "Enabled": true } ] }
```

3. Add the module dependency in your `Build.cs`:

```csharp
PrivateDependencyModuleNames.Add("Framedash");
```

Skipping step 3 is the most common failure: `#include "FramedashSubsystem.h"`
fails to compile. Enabling the plugin in the `.uproject` alone is not enough.
Rebuild after all three steps.

## Configure and initialize

Auto-initialize from `Config/DefaultGame.ini`:

```ini
[/Script/Framedash.FramedashSettings]
ApiKey=your-api-key
bAutoInitialize=True
BuildId=1.0.0
SamplingRate=1.0
```

The compiled default `EndpointUrl` already points at
`https://ingest.framedash.dev/v1/events`. If you must override it, wrap the value
in double quotes: Unreal's INI parser treats `//` as a comment and would read a
bare `https://...` as `https:`, silently dropping every batch.

Or initialize from C++:

```cpp
// GetGameInstance() can be null during early init and in some editor modes;
// null-check it before dereferencing.
if (UGameInstance* GI = GetGameInstance())
{
    if (auto* S = GI->GetSubsystem<UFramedashSubsystem>())
    {
        FString Key = FPlatformMisc::GetEnvironmentVariable(TEXT("FRAMEDASH_API_KEY"));
        S->InitializeTelemetry(Key); // single method; EndpointUrl/BuildId/PlayerId are defaulted params
    }
}
```

## API-key safety

- Do not commit a live key in `DefaultGame.ini`. Exclude the file from version
  control or source the key from `FRAMEDASH_API_KEY` in the C++ path.
- Never echo the key to a log. If one was committed, scrub history and rotate it.

## Send events

```cpp
// GetGameInstance() can be null during early init / editor modes; null-check it.
if (UGameInstance* GI = GetGameInstance())
{
    if (auto* F = GI->GetSubsystem<UFramedashSubsystem>())
    {
        // If the project sets a global SamplingRate < 1.0, the marker could be
        // sampled out. Force this event's rate to 1.0 so verification is reliable.
        F->SetEventSamplingRate(TEXT("run_1720000000_ab12"), 1.0f);
        F->Track(TEXT("run_1720000000_ab12"), TEXT("Map01"), FVector(1000, 2000, 50));
    }
}
```

Use `TrackWithData(...)` to attach a `TMap` of attributes and metrics.

## Headless / CI

First build the editor target (no editor UI needed):

```
"<EngineRoot>\Engine\Build\BatchFiles\Build.bat" <Project>Editor Win64 Development -Project="<full path>\<Project>.uproject"
```

Then launch the game headless with null RHI. Replace `<MapPath>` with a real full
map path (see the table); a `<MapPath>` placeholder avoids a `/Game//Game/`
duplication bug:

```
UnrealEditor-Cmd.exe <Project>.uproject <MapPath> -game -nullrhi -nosound -unattended -nosplash -stdout
```

Template map path differs by engine generation:

| Engine | Third Person template map path |
|--------|--------------------------------|
| 5.3 - 5.5 | `/Game/ThirdPerson/Maps/ThirdPersonMap` |
| 5.6+ | `/Game/ThirdPerson/Lvl_ThirdPerson` |

### `-game` never self-exits

The `-game` loop runs forever and does not quit on its own. CI MUST bound the run
with an external timeout or process kill. Success is the HTTP 2xx log line
appearing, NOT the process exiting.

### The log looks frozen but is not

A piped or redirected `-stdout` is block-buffered, so a successful run can look
hung (for example stuck at `Waiting on static mesh...`) while the HTTP 2xx
(typically 202) already happened. The authoritative record is
`Saved/Logs/<Project>.log`. Check for SUCCESS with a 2xx-tolerant match on the
result line ONLY (`SendBatch:` logs the attempt BEFORE the HTTP result, so
matching it would pass on a mere attempt that later 401'd or failed transport):

```
# success gate: matches only a delivered 2xx result
grep -E "Batch sent successfully \(HTTP 2[0-9]{2}\)" Saved/Logs/<Project>.log

# debugging only (attempts, not success): the send attempt with endpoint + count
grep -E "SendBatch:" Saved/Logs/<Project>.log
```

Add `-FORCELOGFLUSH` if you need real-time stdout. Add `-LogCmds="LogFramedash Verbose"`
for verbose transport logging.

### Git-Bash / MSYS mangles `/Game/...`

MSYS path conversion rewrites a leading `/Game/...` argument into a bogus Windows
path, surfacing as a confusing `Failed to load package` error. Prefix the command
with `MSYS_NO_PATHCONV=1`, or run it from `cmd.exe`.

## Offline queue

The offline queue is written on a graceful shutdown OR a transient send failure
ONLY, and drained on the next initialization once the world ticks. A hard kill
(a CI timeout kill) does NOT write it, so buffered events are lost. In CI, wait
for the HTTP 2xx line before killing the process rather than relying on the queue.

## Verify ingest

Pre-existing data makes "data exists" meaningless. Emit a UNIQUE per-run marker
event named `run_<unix_ts>_<random>` and confirm a row with that exact name
landed today. `framedash query` needs a project id (`--project-id <id>` or
`FRAMEDASH_PROJECT_ID`) and a key with the `data:admin` scope (an ingest
`events:write` key cannot read):

```bash
framedash query --api-key-file full.key --project-id <project-uuid> \
  "SELECT count() FROM events \
   WHERE event_name = 'run_1720000000_ab12' AND timestamp >= today()"
```

A count of 1 or more confirms the run reached ingest. If it is 0, grep the log
for the HTTP status first; a 401/403 means the key is missing or lacks
`events:write`, and a silent drop usually traces back to the unquoted-INI or the
wrong map path above.
