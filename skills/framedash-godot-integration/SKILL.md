---
name: framedash-godot-integration
description: Integrate the Framedash Godot SDK (C# addon, Godot 4.3+ .NET) into a project and verify telemetry reaches ingest from a headless run in CI. Use when a user wants to install the Godot addon, initialize TelemetrySDK, send events, build the C# solution headlessly (including the never-opened-project .csproj fallback), use the correct flush-then-wait pattern, avoid the first-flush race, or debug why Godot events are not arriving.
---

# Framedash Godot integration

Install the Framedash Godot SDK (a C# addon), initialize `TelemetrySDK`, and prove
events reach ingest from a headless run. The SDK auto-collects FPS, frame time,
memory, and GPU/render/process time. This skill covers the C# build path that
fails silently on fresh projects and the flush-then-wait pattern CI needs.

Full guide: https://docs.framedash.dev/en/sdk/godot/
Troubleshooting: https://docs.framedash.dev/en/sdk/troubleshooting/

## Requirements

- Godot 4.3+ .NET (C#) build (verified on 4.6.3). The standard GDScript-only build
  cannot compile the C# addon.
- .NET 8 SDK or newer (the addon builds into a `net8.0` game assembly).
- An ingest API key with the `events:write` scope.
- Web (HTML5) export is out of scope (Godot 4 .NET Web export is unsupported).

## Install

Clone the addon and copy it into your project:

1. Copy `Runtime/`, `FramedashEditorPlugin.cs`, and `plugin.cfg` into
   `res://addons/framedash/`. Do NOT copy `Tests/` (its NUnit deps break the game
   build, since a Godot .NET project compiles every `.cs` under the project).
2. Enable "Framedash Telemetry SDK" under Project Settings > Plugins.
3. Run a C# build once so the autoload type is available.

Require addon version 0.1.3 or newer (see the first-flush race below).

## Initialize

Enabling the plugin registers a `Framedash` autoload singleton, but it is not
required: `TelemetrySDK.Initialize(...)` creates the singleton itself if absent,
which is what headless/CI projects rely on.

```csharp
using Framedash;
using Godot;

public partial class GameBootstrap : Node
{
    public override void _Ready()
    {
        TelemetrySDK.Initialize(apiKey: "...", buildId: "1.0.0");
    }
}
```

`endpointUrl` is optional and defaults to
`https://ingest.framedash.dev/v1/events`.

## API-key safety

- Never commit a live key in a tracked `.cs` file or scene, and never print it.
  Pass the key from an environment variable or a key file at startup.
- If a key was committed, scrub history and rotate it in the dashboard.

## Send events

```csharp
TelemetrySDK.Instance.Track(
    eventName: "run_1720000000_ab12",
    mapId: "map_01",
    position: playerNode.GlobalPosition);
```

`session_start` fires at init and `perf_heartbeat` every 10 seconds; both batch
with your manual events.

## Flush is fire-and-forget

`Flush()` only STARTS an async send; it does not block until delivery. A short or
headless run can exit before delivery completes, and Godot has NO offline queue,
so anything still buffered at exit is lost. The reliable pattern: initialize, turn
on verbose logging, track, let one frame pass, flush, then keep the tree alive on
a timer until the HTTP 2xx (typically 202) verbose line appears, and only then
quit.

```csharp
public override async void _Ready()
{
    TelemetrySDK.Initialize(apiKey: "...", buildId: "1.0.0");
    TelemetrySDK.Instance.VerboseLogging = true;
    // If the project's global SamplingRate is < 1.0, the marker could be sampled
    // out while the automatic session_start still yields a 2xx line. Force this
    // event's rate to 1.0 so the marker is always kept.
    TelemetrySDK.Instance.SetEventSamplingRate("run_1720000000_ab12", 1f);
    TelemetrySDK.Instance.Track(
        eventName: "run_1720000000_ab12", mapId: "ci", position: Vector3.Zero);
    await ToSignal(GetTree(), SceneTree.SignalName.ProcessFrame);
    TelemetrySDK.Instance.Flush();
    // A 10s timer can Quit() mid-retry: the transport allows 3 attempts at a 10s
    // per-attempt timeout with 1/2s backoff, so a wedged endpoint takes ~33s to
    // exhaust. There is NO offline queue, so quitting early drops the batch. Use
    // a timer beyond the retry budget (~40s), or quit as soon as the 2xx line is
    // observed.
    await ToSignal(GetTree().CreateTimer(40.0), SceneTreeTimer.SignalName.Timeout);
    GetTree().Quit();
}
```

Success is the HTTP 2xx (typically 202) verbose line, not the process exiting;
any HTTP 2xx is treated as success and the actual status code is logged. As soon
as you observe that line you can quit; the 40s timer is only the upper bound for a
retrying endpoint.

### First-flush race (versions before 0.1.3)

When the autoload is auto-created (plugin not enabled) and `Flush()` is called
synchronously in the same `_Ready()` that first touches `TelemetrySDK.Instance`,
the SDK can fail with `ERR_UNCONFIGURED` and silently drop the batch. Fix it by
requiring addon 0.1.3+, OR defer the first flush by one frame with the
`await ToSignal(GetTree(), SceneTree.SignalName.ProcessFrame)` shown above.

## Headless / CI build

The editor Build generates the C# `.csproj` / `.sln`. Headless, generate them via
Godot's own project-generation path, then build:

```bash
godot --headless --build-solutions --quit
dotnet build
```

Do NOT use `dotnet new sln` to create the solution: it produces an empty solution
that does not reference `Godot.NET.Sdk`, so the addon never compiles. Newer .NET
SDKs also emit the `.slnx` format, which Godot's tooling may not expect. Let Godot
generate the solution.

### Never-opened-project fallback

On a project never opened in the editor, `godot --headless --build-solutions
--quit` can exit 0 with no error yet generate NO `.csproj` / `.sln` (seen on 4.6.3
mono). If the files are missing after the command, hand-author `<project>.csproj`
and run `dotnet build`. The compiled assembly lands in
`.godot/mono/temp/bin/Debug/`.

```xml
<Project Sdk="Godot.NET.Sdk/4.6.3">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <EnableDynamicLoading>true</EnableDynamicLoading>
  </PropertyGroup>
</Project>
```

Set the `Godot.NET.Sdk/<version>` to match your engine version.

## Verify ingest

Pre-existing data makes "data exists" meaningless. Use a UNIQUE per-run marker
event named `run_<unix_ts>_<random>` and confirm a row with that exact name landed
today. `framedash query` needs a project id (`--project-id <id>` or
`FRAMEDASH_PROJECT_ID`) and a key with the `data:admin` scope (an ingest
`events:write` key cannot read):

```bash
framedash query --api-key-file full.key --project-id <project-uuid> \
  "SELECT count() FROM events \
   WHERE event_name = 'run_1720000000_ab12' AND timestamp >= today()"
```

A count of 1 or more confirms the run reached ingest. If it is 0, read the verbose
log: `HTTP 0` is a transport-level failure (DNS, TLS, IPv6, proxy), not an API
rejection, and Godot's async DNS resolver can fail where `curl` succeeds. See the
troubleshooting page for the transport checks.
