# Godot [overview]

[← Engines](README.md)

> General information; not verified on a real game within this guide.

## Where to look for text

- Game resources live in a `*.pck` archive (next to the `.exe` or embedded in it).
- Translations are `*.translation` resources (compiled from CSV) or `*.po`/`*.mo` (Godot 3.x/4.x), registered in `project.godot` → `[internationalization] locale/translations=…`.
- Keys are `tr("KEY")` strings in scripts and UI node text with auto-translate enabled.

## Ways to add a language

1. An extra `.pck` loaded on top (if the game supports mods/`--main-pack` or `ProjectSettings.load_resource_pack`).
2. Rebuilding the `.pck` with a new `ru.translation` and an updated `project.godot` — modifies the original.

## Fonts

Fonts are `.ttf/.otf` or `FontFile` resources; if Cyrillic is missing, add a fallback font in the theme (`Theme` → default font fallbacks).

Tools: [tools.md](../tools.md#godot)
