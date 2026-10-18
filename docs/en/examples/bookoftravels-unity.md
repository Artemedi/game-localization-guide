# Example: Book of Travels (Unity 2022.3, Mono) — Russian translation through a runtime plugin

[← Contents](../../../README.en.md) · Mechanics: [Unity](../engines/unity.md)

Book of Travels has no language option: every text is English and lives inside `resources.assets`, and the UI fonts have no Cyrillic. Russian was added with a **BepInEx plugin** that replaces texts and attaches a font while the game runs. No game file is modified.

> **Ready-made translation:** release bookoftravels-ru-v0.1.0 — archive with an installer for Linux / Steam Deck (native build). Unofficial, mostly machine-translated and spot-checked; targets Steam buildid 24468107. The Windows build is not supported by this release.

## 1. Reconnaissance

| What | Value | How established |
|---|---|---|
| Engine | Unity **2022.3.62f2**, **Mono** scripting runtime (`BookOfTravels_Data/Managed/*.dll`) | version in `globalgamemanagers`, `Managed/` present |
| Build | native Linux (`BookOfTravels.x86_64`), Steam appid 1152340, buildid 24468107; runs inside the Steam Linux Runtime container (pressure-vessel) | game files, launch argv log |
| Where text is | 281 CSV tables (`TextAsset`) in `resources.assets`, columns `English`, `English Title`, `English Description`, `English Feedback (Start/End)`… | TextAsset inventory (UnityPy) |
| Who reads the CSV | a single parser `CSVReader.Read(TextAsset)` in `Years.General.Base.Assembly.dll` | decompilation (ILSpy) |
| Language option | none; no other languages | menus and CSV |
| UI labels | the `m_text` field of TextMeshPro components in scenes and prefabs (641 unique strings) — outside the CSV | MonoBehaviour parsing |
| NPC poems | baked into `CharacterVariationsManager` (`PoemData`); the poem CSV loader runs only in the editor | decompilation + R10 |
| NPC titles | assembled in code: "characteristic + occupation" or an enum name (`Peasant`, `SahBasir`) | decompilation of `NPC` |
| Fonts | TMP atlases Traveler-pt234, HighTower-pt169 — no Cyrillic; the game's assets contain `Ubuntu-R.ttf` (with Cyrillic) | `TMP_FontAsset` inventory, R2 |

## 2. Experiments (one variable each)

| Run | What changed | Result |
|---|---|---|
| R0 | staging, unchanged | game runs from staging |
| R1 | copy of `resources.assets`, one CSV string → ASCII marker | **M4**: marker in the exit dialog — text path proven |
| R2 | same marker in Cyrillic | **M5**: letters missing, 125× "character … was not found" — fonts lack Cyrillic |
| R3 | BepInEx: doorstop in `LD_PRELOAD` outside the Steam chain | **M0**: game does not start — pressure-vessel builds its own `LD_PRELOAD` |
| R4 | doorstop enabled inside the container, right before the game exe | BepInEx loaded, 0 plugins |
| R5 | plugin: dynamic fallback font from `Ubuntu-R.ttf` for every `TMP_FontAsset` | **M4**: Cyrillic visible |
| R6 | original `resources.assets`; postfix on `CSVReader.Read(TextAsset)` replaces a field by ID | **M4**: translation without touching assets |
| R7–R8 | `translations.tsv` format; NPC titles: prefix on the `NPC.NetworksyncNameSurface` setter with gender agreement | **M4** |
| R9–R10 | full machine translation of the CSV (18k strings), effect notifications | **M4**; found that poems and prefab labels do not come from the CSV |
| R11–R13 | prefix on `NPC.ShowPoem`: poem dictionary keyed by the source without its time tag; baked poems without CSV | **M3** → **M4** |
| R14 | a separate fallback per game font, `faceInfo.scale` matched by x-height | **M4**: text no longer overlaps |
| R15 | postfix on `TextMeshProUGUI/TextMeshPro.Awake`: prefab labels by exact source | **M3**: login screen blank (2957 "replaced by  ") |
| R16 | fallback attached in the same `Awake`, before the first layout | **M4**: login screen in Russian |
| R17–R18 | `ldstr` transpiler in 6 methods (strings built in code: "Скорость подсказок: 0.5 с") | **M4** |
| R19 | transpiler: reagent name from `CharacterVariationsManager.ItemName` instead of the asset name | **M4**: "Одуванчик x2" |

Conclusions that cost the most time:
- **Not all of a game's text is in its tables.** The CSVs covered most of it, but prefab labels, poems (CSV loader is editor-only), strings concatenated in code and asset names (`Chestnut shells x3`) each had to be caught at their own owner.
- **TextMeshPro keeps spaces in place of missing glyphs.** A fallback attached after the first layout no longer helps: the login screen is built before the plugin's scene loads. Attach it in the component's `Awake`.
- **Linux + Steam: `LD_PRELOAD` set outside the container is lost.** The stock `run_bepinex.sh %command%` re-executes itself inside the container right before the game exe — that is the way.
- **Take method names from metadata, not from the decompiler.** ILSpy escapes `[]` as `_005B_005D`; Harmony will not find such a method.
- **Text missing on screen is a fact too.** The NPC trade summary is built on the server and the client never shows it; there is nothing to translate.

## 3. Final mod

```
Book of Travels/                       (game folder in Steam)
├── run_bepinex.sh                     ← Steam launch option: ./run_bepinex.sh %command%
├── libdoorstop.so, .doorstop_version  ← UnityDoorstop: loads BepInEx before the game code
└── BepInEx/
    ├── core/                          ← BepInEx 5.4.23.5 (HarmonyX, MonoMod, Cecil)
    └── plugins/RuFont/
        ├── BookOfTravels.RuFont.dll   ← translation plugin
        ├── translations.tsv           ← asset, column, id, source, translation (19,074 rows)
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
| Reagent names | transpiler: `Object.name` / `ReagentSubType.ToString()` → the item's display name |
| Fonts | a fallback from `Ubuntu-R.ttf` for every `TMP_FontAsset`, scale matched by x-height |

## 4. Translation

| Area | Volume | How |
|---|---|---|
| CSV tables | 281 tables, 19,726 entries, 17,883 unique strings, ~1.77M characters | LLM (Claude Opus 5.5), each unique string once in the context of its table; ~12.7M input + 3.7M output tokens |
| Prefab labels | 641 strings, 631 translated | LLM; key names, brands and placeholders kept |
| NPC poems | 1,807 (101 of them baked without a CSV source) | LLM, markup `[P]`, `[XX]` preserved |
| NPC titles | 1,301 parts: adjectives in 4 forms (m/f/n/pl) + nouns with gender | LLM, answering with forms rather than finished titles |
| Strings in code | 18 literals in 6 methods | by hand, respecting concatenation (`" sec"` → `" с"`) |

Glossary, translator instructions (formal "you" to the player, gender-neutral wording), automatic checks of markup and of the `\t\n\r` count. Every machine string is marked `mt:` for review.

## 5. Risk-free verification (Linux/Steam)

- Staging: symlinks to the game files; real copies only of the executables, `UnityPlayer.so` and `resources.assets` (Unity derives the `*_Data` path from the exe path).
- Steam launch option — a wrapper that rewrites the exe path to staging and logs argv. To verify the archive, the wrapper runs the `./run_bepinex.sh` installed into staging exactly as Steam does for a player.
- Observation: inotify on the original and staging dirs (was the original `resources.assets` opened, any writes), the BepInEx log, `Player.log` (count of "Unicode value … not found").
- After every run — sha256 of 202 key files of the original.

## 6. Known limitations

- The weather name in skill-unavailable reasons is an English enum name.
- Nomad names (`NomadNames`) are not translated.
- "Играть без сет…" is cut off in its button; "Сбросить положение элементов интерфейса" does not fit.
- Reagent costs on the skill card and the "not enough reagents" message are translated but not checked on screen.
- `UnityEngine.UI.Text` (old labels, mostly debug) is not hooked.
- The Windows build and Proton were not tested.
