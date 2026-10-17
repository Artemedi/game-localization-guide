# Tools

[← Contents](../../README.en.md)

Marked: used in the worked example (✔) or listed for reference (·).

## Unreal Engine

| Tool | Purpose | |
|---|---|---|
| [repak](https://github.com/trumank/repak) | list/extract/pack `.pak` (versions V1–V11, mount point) | ✔ |
| [retoc](https://github.com/trumank/retoc) | IoStore `.utoc/.ucas`: list, extract, `to-zen` (including an empty companion container) | ✔ |
| [FModel](https://github.com/4sval/FModel) | browse assets and `.locres` in a GUI (Windows) | · |
| [UAssetGUI](https://github.com/atenfyr/UAssetGUI) | view/edit `.uasset` (StringTable, DataTable) | · |
| [UnrealLocres](https://github.com/akintos/UnrealLocres) | export/import `.locres` to CSV | · |
| Ghidra / IDA | static analysis of the exe: how PAKs are mounted, where `.locmeta` is looked up | ✔ |
| Custom `.locres`/`.locmeta` parser (Python, ~200 lines) | byte-exact round-trip, replacing strings while keeping keys and hashes | ✔ |

## Unity

| Tool | Purpose | |
|---|---|---|
| [AssetRipper](https://github.com/AssetRipper/AssetRipper) | extract assets, TextAssets, fonts | · |
| [UABEA](https://github.com/nesrak1/UABEA) | view/replace assets in `.assets` and bundles | · |
| [Il2CppDumper](https://github.com/Perfare/Il2CppDumper) | class structure of IL2CPP builds | · |
| [BepInEx](https://github.com/BepInEx/BepInEx) + [XUnity.AutoTranslator](https://github.com/bbepis/XUnity.AutoTranslator) | plugin loader + runtime text replacement from a dictionary | · |

## Godot

| Tool | Purpose | |
|---|---|---|
| [gdsdecomp (GDRE Tools)](https://github.com/GDRETools/gdsdecomp) | unpack `.pck`, decompile resources | · |
| [GodotPCKExplorer](https://github.com/DmitriySalnikov/GodotPCKExplorer) | browse/build `.pck` | · |

## General

| Tool | Purpose | |
|---|---|---|
| `ripgrep` / `grep -a` | find a source string in game files | ✔ |
| `inotifywait` / custom watcher | prove the game **opened** the mod file (Linux/Proton) | ✔ |
| Process Monitor (Windows) | same on Windows | · |
| LLM via an OpenAI-compatible API | draft translation of large volumes with a glossary and automatic checks | ✔ |
| `sha256sum` | prove originals are unchanged; reproducible builds | ✔ |

## Linux / Steam Deck (Proton)

- Launch only through Steam: running Proton directly without the `STEAM_COMPAT_*` variables breaks the environment.
- A **staging copy** is handy for experiments: a folder of symlinks to the original files plus the mod files, and a wrapper in Launch Options (`wrapper.sh %command%`) that swaps the exe path. The mod is tested without ever entering the Steam installation.
- The game's `/proc/<pid>/fd` may be unreadable inside pressure-vessel — use inotify on the `Paks` folder.
