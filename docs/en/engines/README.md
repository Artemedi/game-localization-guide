# Engines: how to identify them and where to look for text

[← Contents](../../../README.en.md)

## Identify the engine from the game files

| Sign in the game folder | Engine | Document |
|---|---|---|
| `<Project>/Content/Paks/*.pak`, `*.utoc`, `*.ucas`; `Engine/`; `<Project>.exe` + `<Project>/Binaries/Win64/<Project>-Win64-Shipping.exe` | Unreal Engine 4/5 | [unreal-engine.md](unreal-engine.md) **[verified]** |
| `<Game>_Data/`, `UnityPlayer.dll`, `globalgamemanagers`, `*.assets`, `StreamingAssets/` | Unity | [unity.md](unity.md) **[partly verified]** |
| `*.pck` next to the `.exe` (or embedded in it), string `Godot Engine` in the binary | Godot | [godot.md](godot.md) [overview] |

## Decision tree

```mermaid
flowchart TD
    A[Game folder] --> B{Content/Paks present?}
    B -- yes --> UE[Unreal Engine]
    B -- no --> C{*_Data and UnityPlayer.dll?}
    C -- yes --> U[Unity]
    C -- no --> D{.pck / Godot string?}
    D -- yes --> G[Godot]
    D -- no --> O[Other engine: search by UI strings]

    UE --> UE1{.utoc/.ucas present?}
    UE1 -- yes --> UE2[UE5 / UE4.26+ with IoStore: mod = .pak + empty .utoc/.ucas]
    UE1 -- no --> UE3[Classic PAK: mod = one .pak]
```

## Universal approach for an unknown engine

1. Pick a unique English UI string (a button label, for example).
2. Search the game files for it (`grep -rla "Start New Campaign"`); if files are compressed, search the extracted contents.
3. Check whether files for other languages (`fr`, `de`, `ja`) sit nearby — their structure is the template for the new language.
