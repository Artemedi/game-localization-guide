# Unreal Engine 4 / 5 **[verified]**

[← Engines](README.md) · Example: [Wyldheart, UE 5.7](../examples/wyldheart-ue5.md)

## Where the text lives

```
<Project>/Content/Paks/
  <Project>-Windows.pak            ← PAK: ini, .locres/.locmeta, .ufont fonts, ICU, AssetRegistry
  <Project>-Windows.utoc / .ucas   ← IoStore (UE5, UE4.26+): assets (.uasset) — widgets, string tables
  global.utoc / .ucas              ← engine script objects

Inside the PAK (virtual paths, mount point usually ../../../):
  <Project>/Content/Localization/
    <Target>/                      ← localization "target": Game, plus plugins/modules
      <Target>.locmeta             ← metadata: native culture, CompiledCultures list
      <Target>.manifest / .archive ← (often stripped from shipping builds, not needed by the game)
      en/<Target>.locres           ← text table of the English culture
      fr/<Target>.locres
      …
  Engine/Content/Localization/Engine/<culture>/Engine.locres   ← engine strings
```

There can be several targets: the main `Game` plus separate ones for settings, plugins, online modules. **Each target is translated and built separately**; a target without `ru` falls back to the default language.

## `.locres` format (version 3, UE 4.24+)

- Header: magic GUID, version.
- Table of unique strings (UTF-16), reference counts.
- Namespaces → keys → (source string hash, string index).
- **Namespace** — for StringTable text it equals the table name (`ST_Common_Menu`); for Blueprint/data text it is the empty string and the key is a GUID.
- **source_hash** = CRC32 of the source (English) text computed over UTF-32LE (`FCrc::StrCrc32`). If the hash in your `.locres` differs from the game's source, the translation is ignored → **keep the hashes from the original `en/<Target>.locres`**.

`.locmeta` is a small binary file: version, NativeCulture, NativeLocRes, CompiledCultures. Add the new culture to CompiledCultures so the language shows up in the settings list.

## Layering a translation: a mod PAK

```
<Project>/Content/Paks/
  z_MyGame-RU_P.pak        ← only new/changed files at the same virtual paths
  z_MyGame-RU_P.utoc       ← (only if the game uses IoStore) empty companion
  z_MyGame-RU_P.ucas
```

- The **`_P` suffix** raises mount priority (+100 per `_N_P` level), so mod files override originals at the same path. The `z_` prefix is just sorting.
- The **mount point** must match the original (usually `../../../`), otherwise virtual paths will not line up.
- **IoStore games:** if the original has `.utoc/.ucas`, in the typical configuration a PAK without a paired `.utoc` **is opened but rejected** (`utoc not found`, outcome M1). Fix: an empty IoStore container with the same name (e.g. `retoc to-zen` from an empty directory).
- **PAK version** — same as the original (UE 5.x is usually V11). Compression and encryption are not needed for the mod unless the game requires them.

Minimal mod for a new language:

```
<Project>/Content/Localization/Game/Game.locmeta        ← original + "ru" in CompiledCultures
<Project>/Content/Localization/Game/ru/Game.locres      ← copy of en/Game.locres with translated strings
```

Repeat for every target (`Settings`, `Lyra`, … — depends on the game).

## Selecting the language

- In the game settings (if the game builds the list from CompiledCultures, the new language appears automatically).
- Launch parameter `-culture=ru`.
- User config: `Saved/Config/Windows/GameUserSettings.ini` → `[Internationalization] Culture=ru` (the game usually writes it itself after a menu selection).

## StringTables

String tables (`/Game/Localization/StringTables/ST_*.uasset`) hold **only the source (English) text**; translations still come from `.locres` with namespace = table name. No need to edit `.uasset` files to translate.

## Fonts

A UE font is a composite font: default typeface + fallback + sub-typefaces per range/culture. If the decorative font lacks Cyrillic, the engine takes glyphs from the fallback (often Noto Sans) — no boxes, but a different typeface. Check the cmap coverage of the fonts (`.ufont` = a TTF/OTF inside) **before** attempting to replace fonts.

## Pitfalls

| Symptom | Cause |
|---|---|
| Game does not see the mod at all | wrong name (`_P`), wrong folder, game launched from another copy |
| Mod opened but has no effect (UE5) | missing empty `.utoc/.ucas` |
| Language missing from the list | `ru` not added to the `.locmeta` of that target |
| Part of the UI stays English | other targets not translated (Settings/plugins/Engine) or the string is hard-coded |
| Translating one string "moved" text elsewhere | the same string is shared by several keys — translate by key, not by text |
| After a game update some strings are English again | the source text changed → source_hash changed; rebuild from the new `en/*.locres` |

Tools: [tools.md](../tools.md#unreal-engine)
