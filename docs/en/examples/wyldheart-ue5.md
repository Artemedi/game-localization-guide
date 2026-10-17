# Example: Wyldheart Demo (Unreal Engine 5.7) — Russian localization from scratch

[← Contents](../../../README.en.md) · Mechanics: [Unreal Engine](../engines/unreal-engine.md)

The Steam demo of Wyldheart ships with en, de, fr, ja, ko, zh-Hans. No Russian. Below is how it was added as a **separate mod archive**, without modifying a single game file.

## 1. Reconnaissance

| What | Value | How established |
|---|---|---|
| Engine | UE **5.7.4**, shipping, project codename `Kauri` | `EngineVersion` in CrashContext, strings in the exe |
| Containers | mixed: `Kauri-Windows.pak` (V11, Oodle, unencrypted, mount `../../../`) + `Kauri-Windows.utoc/.ucas` (IoStore) + `global.utoc/.ucas` | `repak info`, `retoc info` |
| Where text is | PAK only: `.locres/.locmeta`, ini, `.ufont` fonts | `repak list` |
| UI and StringTables | in IoStore (`.uasset`); StringTables hold English only | `retoc list`, `.uexp` parsing |
| Targets | `Game` (5795 entries), `Settings` (323, settings menu), `Lyra` (75, shared buttons/tips), `Script` (63, chat and system messages) | contents of `Content/Localization/` |

Tree inside the archive (virtual paths):

```
Kauri/Content/Localization/
├── Game/
│   ├── Game.locmeta              NativeCulture=en, CompiledCultures=[de,en,fr,ja,ko,zh-Hans]
│   ├── en/Game.locres            5795 entries: 10 StringTable namespaces + 5305 Blueprint texts (namespace "")
│   ├── fr/Game.locres
│   └── …
├── Settings/  Settings.locmeta, <culture>/Settings.locres
├── Lyra/      Lyra.locmeta,     <culture>/Lyra.locres
└── Script/    Script.locmeta,   <culture>/Script.locres
Kauri/Content/Localization/Fonts/  Beaufort-*, Hideout-*, Noto-Sans/*   (.ufont)
```

## 2. Experiments (one variable each)

| Run | Change | Outcome |
|---|---|---|
| R0 | nothing (control, language fr) | French menu, "Quitter" |
| R1 | + `z_…_P.pak` with `fr/Game.locres` where one string = «РУССКИЙ ТЕСТ» | **M1**: PAK opened but not mounted — IoStore games need a paired `.utoc` |
| R2 | + empty `z_…_P.utoc/.ucas` (`retoc to-zen` from an empty folder) | **M4**: «РУССКИЙ ТЕСТ» instead of "Quitter" |
| R3 | new culture: `ru/Game.locres` + `ru` in `Game.locmeta` | **M4**: «русский» appeared in settings, selecting it shows the string |
| R4 | launch with `-culture=ru` | **M4** immediately; but the settings menu is English → that is another target (`Settings`) |
| R5 | + `ru` for the `Settings` target | **M4**: settings in Russian |
| R6 | build from canonical JSON (4 targets, test strings) | **M4**: pipeline confirmed |
| R7 | Settings, Lyra, Script fully translated | **M4** |
| R9 | Game translated (LLM, 5778/5795) | **M4**: whole UI, journal, chat, inventory; clipping of long labels found |

Lessons that cost the most time:
- **"IoStore present means PAK is unused" — wrong.** All localization lives in the PAK; IoStore only requires an empty companion.
- **Every target separately.** Without `ru` in `Settings` the settings menu stays English even when `Game` is translated.
- **Fonts did not need changes.** Decorative fonts (Beaufort, Hideout) have 0 of 66 Cyrillic letters, but the composite fonts fall back to Noto Sans SC/JP/KR, which has Cyrillic (66/66). No boxes; only the typeface differs.

## 3. Final mod

```
Kauri/Content/Paks/
├── z_Wyldheart-RU_P.pak     ← mount ../../../, PAK V11, uncompressed
├── z_Wyldheart-RU_P.utoc    ← empty IoStore companion (required)
└── z_Wyldheart-RU_P.ucas

Contents of the .pak:
Kauri/Content/Localization/Game/Game.locmeta            (+ ru)
Kauri/Content/Localization/Game/ru/Game.locres
Kauri/Content/Localization/Settings/Settings.locmeta    (+ ru)
Kauri/Content/Localization/Settings/ru/Settings.locres
Kauri/Content/Localization/Lyra/Lyra.locmeta            (+ ru)
Kauri/Content/Localization/Lyra/ru/Lyra.locres
Kauri/Content/Localization/Script/Script.locmeta        (+ ru)
Kauri/Content/Localization/Script/ru/Script.locres
```

Each `ru/*.locres` is a copy of `en/*.locres` (same namespaces, keys and source_hash) with only the translated entries' texts replaced. Untranslated entries stay English.

Language selection: Settings → Gameplay → Language → «русский», or launch parameter `-culture=ru`. The game stores the choice in
`%LOCALAPPDATA%/Kauri/Saved/Config/Windows/GameUserSettings.ini` → `[Internationalization] Culture=ru`
(on Linux/Proton — the same path inside `compatdata/<appid>/pfx/…`).

## 4. Translation

| Target | Strings | How |
|---|---|---|
| Settings | 300 / 323 | manual; left as is — DLSS/FSR/XeSS, aspect ratios, `{0}%` |
| Lyra | 65 / 75 | manual; left as is — platform and controller names, Discord |
| Script | 62 / 63 | manual; impersonal forms in chat |
| Game | 5778 / 5795 | LLM (Claude Opus 5.5), 3825 unique strings, ~420k tokens, ~15 min; then fixes from screenshots |

Fixes after in-game checks (examples):
- the Status column fits ~23 characters → abbreviated stat labels;
- the inventory tab fits ~8 letters → `Pack` = «Вещи»;
- ambiguity without context: slot `Chest` = «Торс» (not «Сундук», a box), `Feet` = «Ступни»;
- the window-mode selector value was too long → «Окно без рамки».

## 5. Risk-free testing (Linux/Steam)

- Staging folder: symlinks to every original file + mod files in `Paks/`.
- Steam Launch Options: a wrapper `wrapper.sh %command% -culture=ru` that swaps the exe path to the staging copy and writes a launch log.
- Observation — inotify on both `Paks` folders (original and staging): shows whether the mod file was opened.
- After every run — SHA-256 check of all original containers against the Steam manifest.
- Trap: Steam may reset Launch Options — the game then silently starts from the original (run R8 in the experiment). Check for a fresh wrapper log.
