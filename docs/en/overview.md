# How game localization works

[← Contents](../../README.en.md)

## How a game shows text

Almost every engine separates **code/UI** from **text**. The UI does not store the string "Quit" but a reference to it — a key:

```
"Quit" button  →  key  ST_Common_Menu / Menu_Quit  →  table of the current language  →  "Quit" / "Quitter" / "Выход"
```

Translation tables are separate files, one per language (culture): `en`, `fr`, `de`, `ru`… At startup the game picks a culture (from settings, the OS, or a launch parameter) and loads the matching table.

Hence the key idea: **to add a language you do not need to change the UI — just add one more table and tell the game that this culture exists.**

## Three questions to answer

| Question | Example answer (Unreal Engine) |
|---|---|
| **Where are the tables?** | `Content/Localization/<Target>/<culture>/<Target>.locres` inside a `.pak` archive |
| **How does the game learn the list of languages?** | `<Target>.locmeta` (CompiledCultures) + presence of `<culture>/` |
| **How to add my file without touching the original?** | a separate higher-priority `.pak` mounted on top |

The answers differ per engine — see [engines/](engines/README.md).

## Ways to layer a translation (best to worst)

1. **A separate mod archive with higher priority than the original** (UE: `*_P.pak`; Godot: an extra `.pck`; Bethesda: a plugin). Originals unchanged; removing the mod reverts everything.
2. **Loose files next to the game**, if the engine reads them (many Unity games: `StreamingAssets`).
3. **Runtime text interception** (Unity: XUnity.AutoTranslator and similar). Works almost everywhere, but it is runtime replacement, not localization.
4. **Editing the original archives** — last resort: breaks on updates, needs backups, conflicts with Steam file verification.

## What almost always needs checking

- **Keys and hashes.** A translation table is matched by key, not by text; some formats also store a hash of the source string and drop the translation if the hash does not match.
- **Placeholders and markup.** `{0}`, `{PlayerName}`, `<highlight>…</>`, `%s`, `\n` must stay in the translation in the same quantity.
- **Fonts.** If a font lacks Cyrillic you get boxes (□□□). Engines often take missing glyphs from a fallback font — the text is readable but the typeface differs.
- **String length.** Russian is on average 15–30 % longer than English: buttons, tabs and columns get clipped.
- **Language selection.** The new culture must appear in the game's settings or be selectable by a launch parameter.

## Experiment outcome scale

Classify each attempt with one code — it shows at once which stage broke:

| Code | What we observe | Where to look |
|---|---|---|
| M0 | The game did not even open the mod file | file name/path/extension, game launched from another copy |
| M1 | File opened but rejected (not mounted) | container format, version, missing companion files |
| M2 | Mounted, but the language did not appear | culture list (metadata), path inside the archive |
| M3 | Language present, text unchanged | keys/namespace/hashes do not match |
| M4 | **Translation visible on screen** | — success |
| M5 | Text visible but boxes instead of letters | fonts (glyph coverage) |

Next: [engines](engines/README.md) · [pipeline](pipeline.md)
