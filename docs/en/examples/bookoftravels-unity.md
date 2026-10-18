# Example: Book of Travels (Unity 2022.3, Mono) — Russian translation through a runtime plugin

[← Contents](../../../README.en.md) · Mechanics: [Unity](../engines/unity.md)

Book of Travels has no language option: every text is English and lives inside `resources.assets`, and the UI fonts have no Cyrillic. Russian was added with a **BepInEx plugin** that replaces texts and attaches a font while the game runs. No game file is modified.

> **Ready-made translation:** [release bookoftravels-ru-v0.1.0](https://github.com/Artemedi/game-localization-guide/releases/tag/bookoftravels-ru-v0.1.0) — archive with an installer for Linux / Steam Deck (native build). Unofficial, mostly machine-translated and spot-checked; targets Steam buildid 24468107. The Windows build is not supported by this release.

## 1. Reconnaissance

| What | Value | How established |
|---|---|---|
| Engine | Unity **2022.3.62f2**, **Mono** scripting runtime (`BookOfTravels_Data/Managed/*.dll`) | version in `globalgamemanagers`, `Managed/` present |
| Build | native Linux (`BookOfTravels.x86_64`), Steam appid 1152340, buildid 24468107; runs inside the Steam Linux Runtime container (pressure-vessel) | game files, launch argv log |
| Containers | plain Unity serialized files (`resources.assets`, `sharedassets*.assets`, `levelN`), no bundles or Addressables; no type tree | UnityPy |
| Language option | none. The string loader's `chosenLanguage` field is never assigned; no language column other than `English` | decompilation, CSV |

## 2. Data and tools

### 2.1 Tools

| Tool | Version | Used for |
|---|---|---|
| UnityPy | 1.25.4 (`uv venv`) | reading `.assets`: TextAsset and font inventory, raw MonoBehaviours, TTF extraction; the `resources.assets` copy for PoC R1–R2 |
| ilspycmd | `DOTNET_ROLL_FORWARD=Major` | decompiling `Managed/*.dll` (`-p` project, `-il` IL): who reads the text, where it is concatenated |
| `strings` (binutils) | — | exact method names from assembly metadata (ILSpy escapes them) |
| BepInEx | 5.4.23.5 linux x64 (UnityDoorstop, HarmonyX, MonoMod, Cecil) | plugin loader, Harmony hooks |
| Roslyn `csc.dll` | from the .NET SDK | building the plugin without NuGet: `-langversion:7.3 -deterministic`, `-r:` to the game's `Managed/*.dll` and `BepInEx/core/*.dll` |
| Claude Opus 5.5 | via the Wello API | machine translation (`tools/l10n_mt.py`, `tools/npc_titles.py translate`) |
| inotify, `sha256sum` | — | watching file opens, checking the original |
| own scripts | `bookoftravels-ru/tools/` repository | `inventory_textassets.py`, `inventory_fonts.py`, `extract_texts.py`, `extract_ui_text.py`, `extract_baked_poems.py`, `npc_titles.py`, `export_translations.py`, `make_poc.py`, `build_plugin.sh`, `build_dist.sh`, `watch_open.py`, `observe_run.sh` |

### 2.2 Unpacking

Nothing was unpacked to disk: the scripts read `.assets` read-only through UnityPy and write canonical JSON directly. `DATA` is `BookOfTravels_Data` (the staging copy was used).

```bash
.venv/bin/python -I tools/inventory_textassets.py DATA research/stage0/textassets.tsv   # 293 TextAssets: file, path_id, name, size, sha256
.venv/bin/python -I tools/inventory_fonts.py DATA                                        # TMP_FontAsset → research/stage0/fonts.tsv
.venv/bin/python -I tools/extract_texts.py translation/ru DATA/resources.assets DATA/sharedassets33.assets
.venv/bin/python -I tools/extract_ui_text.py DATA                                        # → translation/ru/UI - Prefab Text.json
.venv/bin/python -I tools/extract_baked_poems.py DATA/resources.assets                   # → CharacterVariationsManager - PoemData.json
python3 -I tools/npc_titles.py build                                                     # → translation/ru/npc_titles.json
DOTNET_ROLL_FORWARD=Major ilspycmd -p -r DATA/Managed -o decomp/General DATA/Managed/Years.General.General.Assembly.dll
strings DATA/Managed/Years.General.General.Assembly.dll | grep 'Item\[\]'                # the real method name for Harmony
```

### 2.3 Where the text lives

| Text | Container | Object | Format and field | Example |
|---|---|---|---|---|
| Game tables (bulk) | `resources.assets` — 285 TextAssets; `sharedassets33.assets` — 8 (credits) | TextAsset by name, e.g. `GUI Copy - InfoPrompt`, path_id 4791, 14,040 bytes | `m_Script`: UTF-8 CSV, first line is the header. 56 tables have an `English` column | header `ID,English,Note`; row `INFPR_00221,Are you sure you want to exit the game?,` |
| UI labels | `resources.assets`, `sharedassets*.assets`, `levelN` | MonoBehaviour `TextMeshProUGUI`, `TextMeshPro`, `UnityEngine.UI.Text` — 2,144 components, 641 unique texts | `m_text` (TMP) / `m_Text` (UI.Text), read at its serialized position (see below) | "Settings", "Exit game", "Cost: " |
| NPC poems (shown) | `resources.assets` | MonoBehaviour `CharacterVariationsManager`, 12 objects containing `[/P]` (path_id 98037, 99129, 100400, 98228, …; up to 270 KB) | `List<PoemData{string poem; int poemTag}>`, no type tree; recognised by shape "string + int32 0..3" | `Ah... Yes...[XX]You are so very much more apparent to me now, mortal.…` |
| NPC poems (CSV) | `resources.assets` | TextAsset `ClassPoems - *`, `GossipPoems - *`, `WalkByPoems - *`, `OtherPoems - *`, `CustomPoems - *` (66 tables) | columns `Common/Uncommon/Rare/Unique`, `Poem`; time tag `[E]/[M]/[N]`, markup `[P]`, `[XX]` | the `LoadPoems` loader runs only from the editor — the game never reads these CSVs; they serve as the dictionary for the baked poems |
| NPC titles | `resources.assets` | TextAsset `NpcNames - Characteristics`, `NpcNames - Occupation - <Occupation>` (+ `- Neutral`); MonoBehaviour `npcNameOverrides`; NPC type enums in `Years.General.Base.Assembly.dll` | the server builds `Characteristic + " " + Occupation` or an enum name, stores it in `NPC.syncNameSurface`, in the save as `SNS:<title>` | `Seasoned Gate Warden`, `Peasant`, `Commoner` |
| Strings in code | `Managed/Years.General.General.Assembly.dll` | 7 methods, e.g. `UISettings.SetTooltipSpeed`, `EquipmentView.GetItemTypeText` | IL `ldstr`; for item types `constrained. ItemType; callvirt Object::ToString` (IL_0156) | `"Tool tip speed: " + v + " sec"`; `Reagent`, `Edible` |
| Reagent names | assets | the item asset's name (`Object.name`), enum `ReagentSubType` | code shows the asset name instead of the table name | `Chestnut shells x3` while `REAG_00002` is translated |
| Fonts | `resources.assets` | TMP_FontAsset `Traveler-pt234` (path_id 92861), `HighTower-pt169` (92860), `LiberationSans SDF` (99861); Font `Ubuntu-R` (11533) | TMP — static 1024² atlas, no Cyrillic; `Ubuntu-R` — TTF with full Cyrillic | R2: 125× "character … was not found" |

CSV parsing rules — the game's own, `CSVReader.Read(TextAsset)` in `Years.General.Base.Assembly.dll`:
- lines split on `\r\n|\n\r|\n|\r`, fields on `,(?=(?:[^"]*"[^"]*")*(?![^"]*"))`;
- values are trimmed of `"` and stripped of `\`; a row with an empty first field is skipped; a value accepted by `int/float.TryParse` becomes a number (not translated);
- `[X]` in text is a line break.

Loading: `CharacterVariationsManager.Awake()` → `LoadGUICopy`, `LoadItemNames`, … → `CSVReader.Read` → `List<Dictionary<string, object>>`. So `source` in the translation is the parsed value, not the raw CSV text.

UI labels without a type tree: MonoBehaviour header (`m_GameObject`, `m_Enabled`, `m_Script`, `m_Name` at offset 28), then Graphic (`m_Material` 12, `m_Color` 16, `m_RaycastTarget` 4, `m_RaycastPadding` 16), MaskableGraphic (`m_Maskable` 4, `m_OnCullStateChanged` — call count must be 0), then `TMP_Text.m_text`, or 56 bytes of `m_FontData` followed by `UI.Text.m_Text`. The component class comes from the MonoScripts in `globalgamemanagers.assets`. A component that does not fit the layout is skipped and counted.

### 2.4 Editing and building

Canonical format — `translation/ru/<asset>.json`, one file per table:

```json
{"asset": "GUI Copy - InfoPrompt", "file": "resources.assets", "id_column": "ID",
 "entries": [{"key": "INFPR_00001|English", "id": "INFPR_00001", "column": "English",
              "source": "You cannot discard favourite items",
              "translation": "Вы не можете выбросить любимые предметы",
              "note": "mt:claude-opus-5.5-stable"}]}
```

- The same format for `UI - Prefab Text.json` (column `UIText`), `CharacterVariationsManager - PoemData.json` (`PoemData`) and `Code - UI literals.json` (column `Type.Method`, key `#n|Type.Method`; written by hand from the decompiled code).
- `npc_titles.json`: adjectives with `m/f/n/pl` forms and position `pre/post`, nouns with gender.
- Extraction is repeatable: for an unchanged `(key, source)` the translation and `note` are kept. Machine strings carry `note: mt:…`, manual fixes do not.
- Build: `tools/export_translations.py` → `translations.tsv` (`asset, column, id, source, translation`; `\\ \t \n \r` escaped), `tools/npc_titles.py export` → `npc_titles.tsv`, `tools/build_plugin.sh` → `BookOfTravels.RuFont.dll` + `Ubuntu-R.ttf` (extracted from `resources.assets`), `tools/build_dist.sh VERSION` → archive.
- Assets were never edited. Exception — PoC R1–R2: `tools/make_poc.py` changed one CSV string in a **copy** of `resources.assets` in staging and checked that only that object changed (a UnityPy re-save also changes the size field in the header, offset 28, and 4 bytes of padding).

### 2.5 Capture and verification (Linux/Steam)

- Staging (`tools/build_staging.sh`): symlinks to the game files; real copies only of the executables, `UnityPlayer.so` and `resources.assets` (Unity derives the `*_Data` path from the exe path).
- Steam launch option: `…/tools/steam-staging-wrapper.sh %command%` — rewrites the exe path to staging, writes `logs/launch-*.log`. If staging has a `run_bepinex.sh`, the wrapper runs it — as a player's Steam would.
- `tools/observe_run.sh RN` before launch: inotify (`watch_open.py`) on the original and staging dirs, copies of `Player.log` and `BepInEx/LogOutput.log`, and after exit sha256 of 202 key files of the original.
- Proof of a run: `rewritten_args ≥ 1` in the launch log; Mono path in `Player.log` is staging; 0 "Unicode value … not found"; plugin lines ("patched …", "translated N rows in '…'"); the original `resources.assets` not opened; CLOSE_WRITE 0; baseline ALL-OK; a screenshot or the user's confirmation.

## 3. Experiments

| Runs | What was tested | Result |
|---|---|---|
| R0–R2 | text path: a string in a copy of `resources.assets` — ASCII, then Cyrillic | **M4** marker visible; **M5** fonts lack Cyrillic |
| R3–R5 | loading BepInEx inside the Steam container; fallback font from `Ubuntu-R.ttf` | **M0** with `LD_PRELOAD` outside → loading inside the container → **M4** Cyrillic |
| R6–R10 | translation without touching assets: postfix `CSVReader.Read`, NPC titles, full machine translation | **M4**; poems and prefab labels do not come from the CSV |
| R11–R16 | poems (`NPC.ShowPoem`), prefab labels, font before the first layout | **M3** → **M4** |
| R17–R21 | strings in code, reagent names, item types; the archive with the installer | **M4** |

The full run log with evidence is in the translation repository's README.

Conclusions that cost the most time:
- **Not all text is in the tables.** Prefab labels, baked poems, strings in code and asset names each had to be caught at their own owner.
- **TextMeshPro keeps spaces in place of missing glyphs.** Attach the fallback in the component's `Awake`, before the first layout.
- **Linux + Steam: `LD_PRELOAD` set outside the container is lost.** The stock `./run_bepinex.sh %command%` works.
- **Method names from metadata, not from the decompiler** (`_005B_005D` instead of `[]`).

## 4. Final mod

```
Book of Travels/                       (game folder in Steam)
├── run_bepinex.sh                     ← Steam launch option: ./run_bepinex.sh %command%
├── libdoorstop.so, .doorstop_version  ← UnityDoorstop: loads BepInEx before the game code
└── BepInEx/
    ├── core/                          ← BepInEx 5.4.23.5 (HarmonyX, MonoMod, Cecil)
    └── plugins/RuFont/
        ├── BookOfTravels.RuFont.dll   ← translation plugin
        ├── translations.tsv           ← asset, column, id, source, translation (19,076 rows)
        ├── npc_titles.tsv             ← NPC title parts with gender forms (1,301 rows)
        ├── Ubuntu-R.ttf               ← fallback font (the same file the game ships in its assets)
        └── installed-files.txt        ← list used by uninstall.sh
```

What the plugin does (every replacement matches the exact English source; if a game update changes the text, it stays English):

| Where the text is | Hook |
|---|---|
| CSV tables | postfix `CSVReader.Read(TextAsset)` — replace a field when its current value equals the source |
| Prefab labels | postfix `TextMeshProUGUI.Awake` / `TextMeshPro.Awake` (the fallback font is attached there too) |
| NPC poems | prefix `NPC.ShowPoem(string, …)` |
| NPC titles | prefix on the `NPC.NetworksyncNameSurface` setter: "Seasoned" + "Gate Warden" → «бывалый» + «привратный страж» (masc.) |
| Strings built in code | transpiler: `ldstr` in listed methods; for async ones, in the state machine's `MoveNext` |
| Item types | transpiler: the `enum.ToString()` result in `EquipmentView.GetItemTypeText` goes through the literal table |
| Reagent names | transpiler: `Object.name` / `ReagentSubType.ToString()` → the item's display name |
| Fonts | a fallback from `Ubuntu-R.ttf` for every `TMP_FontAsset`, scale matched by x-height |

## 5. Translation

| Area | Volume | How |
|---|---|---|
| CSV tables | 281 tables, 19,726 entries, 17,883 unique strings, ~1.77M characters | LLM (Claude Opus 5.5), each unique string once in the context of its table; ~12.7M input + 3.7M output tokens |
| Prefab labels | 641 strings, 631 translated | LLM; key names, brands and placeholders kept |
| NPC poems | 1,807 (101 of them baked without a CSV source) | LLM, markup `[P]`, `[XX]` preserved |
| NPC titles | 1,301 parts: adjectives in 4 forms (m/f/n/pl) + nouns with gender | LLM, answering with forms rather than finished titles |
| Strings in code | 18 literals in 6 methods | by hand, respecting concatenation (`" sec"` → `" с"`) |

Glossary, translator instructions (formal "you" to the player, gender-neutral wording), automatic checks of markup and of the `\t\n\r` count. Every machine string is marked `mt:` for review.

## 6. Known limitations

- The weather name in skill-unavailable reasons is an English enum name.
- Nomad names (`NomadNames`) are not translated.
- "Играть без сет…" is cut off in its button; "Сбросить положение элементов интерфейса" does not fit.
- Reagent costs on the skill card and the "not enough reagents" message are translated but not checked on screen.
- `UnityEngine.UI.Text` (old labels, mostly debug) is not hooked.
- The Windows build and Proton were not tested.
