---
type: llm
weight: 1
---

PASS requires all of the following:

1. The response names an install or configuration specific: "Framedash" added to PrivateDependencyModuleNames in the module Build.cs, the plugin placed at Plugins/Framedash so that Plugins/Framedash/Framedash.uplugin exists, or FramedashSettings configuration such as the [/Script/Framedash.FramedashSettings] section of Config/DefaultGame.ini or InitializeTelemetry on UFramedashSubsystem.
2. The response runs the headless check as -game with -nullrhi, and says that run never exits on its own, so CI must bound it with a timeout or a kill.
3. The response treats Saved/Logs/<Project>.log as the authoritative record, and a delivered result line such as "Batch sent successfully (HTTP 2xx)" as the success signal, rather than the process exiting or a SendBatch attempt line.

FAIL if the response is generic Unreal or generic telemetry advice, or if it invents Framedash class, setting, or log-line names.
