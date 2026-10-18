# Example: Wyldheart Demo (Unreal Engine 5.7) — Russian localization from scratch

[← Contents](../../../README.en.md) · Mechanics: [Unreal Engine](../engines/unreal-engine.md)

The Steam demo of Wyldheart ships with en, de, fr, ja, ko, zh-Hans. No Russian. Below is how it was added as a **separate mod archive**, without modifying a single game file.

> **Ready-made translation:** [release wyldheart-ru-v0.1.0](https://github.com/Artemedi/game-localization-guide/releases/tag/wyldheart-ru-v0.1.0) — archive with an installer for Windows and Linux/Steam Deck. Unofficial, mostly machine-translated with manual fixes; targets the Steam demo buildid 25823320.

## 1. Reconnaissance

| What | Value | How established |
|---|---|---|
| Engine | UE **5.7.4**, branch `++kauri+stage-CL-39317`, Shipping, AngelScript fork; project `Kauri` | `<EngineVersion>` in CrashContext.runtime-xml, `Kauri.exe` strings, `Kauri/Script/*.Cache` |
| Build | Windows (`Kauri/Binaries/Win64/Kauri.exe`), on Linux via Proton; Steam appid 4475270, depot 4475271, buildid 25823320 | file SHA-1s match `depotcache/4475271_*.manifest` |
| Containers | mixed layout: `Kauri-Windows.pak` + `Kauri-Windows.utoc/.ucas` (IoStore) + `global.utoc/.ucas` | `repak info`, `retoc info` |
| Language option | yes: en, de, fr, ja, ko, zh-Hans (`DefaultGame.ini` → `CulturesToStage`); the choice is saved in `GameUserSettings.ini` → `[Internationalization] Culture=` | ini from the PAK, profile in `compatdata` |
| Logs | none in Shipping: no `LogPak`/`LogTextLocalizationManager` categories in the exe, `-abslog` created no file | exe strings, a run |

## 2. Data and tools

### 2.1 Tools

| Tool | Version | Used for |
|---|---|---|
| repak (`repak_cli`) | 0.2.3 | `info`/`list`/`unpack` of the PAK; `pack` of the mod PAK |
| retoc (`retoc_cli`) | 0.1.5 | `info`/`list` of IoStore, `to-legacy` (Zen → `.uasset/.uexp`), `to-zen` for the empty IoStore companion |
| Python | 3.14 | own parsers, no third-party packages |
| REA + Ghidra | — | static analysis of `Kauri.exe`: how `*.pak` files are found, `_P` priority, the `.utoc` requirement |
| inotify, `sha256sum` | — | file opens in `Paks`, checking the original |
| Claude Opus 5.5 | via the Wello API | machine translation of the `Game` target (`tools/l10n_mt.py`) |
| own scripts | `wyldheart-ru/tools/` repository | `locres.py` (read/write locres v0–3, byte-exact round-trip of all 24 files), `locres_set.py`, `locmeta.py`, `stringtable.py`, `font_coverage.py`, `l10n.py` (extract/validate/build), `make_dist.sh`, `watch_open.py`, `observe_run.sh`, `steam-staging-wrapper.sh` |

### 2.2 Unpacking

`PAKS` = `Wyldheart Demo/Kauri/Content/Paks`. Everything is extracted into the translation repository's `extracted/` (not in Git) and never edited.

```bash
repak info  PAKS/Kauri-Windows.pak         # V11 (Fnv64BugFix), mount ../../../, Oodle, unencrypted, 4764 files
repak list  PAKS/Kauri-Windows.pak
repak unpack -q -o extracted/stage0-pak -i Kauri/Content/Localization/Game ... -i Kauri/Config PAKS/Kauri-Windows.pak
retoc info  PAKS/Kauri-Windows.utoc        # 36,274 chunks / 25,276 packages, Oodle; global.utoc — 1 chunk
retoc list --path PAKS/Kauri-Windows.utoc
retoc to-legacy --no-shaders -f /Localization/StringTables/ PAKS/ extracted/stage0-iostore   # needs the whole Paks dir (global.utoc)
python3 -I tools/locres.py dump extracted/stage0-pak/Kauri/Content/Localization/Game/en/Game.locres > game-en.json
python3 -I tools/stringtable.py extracted/stage0-iostore/Kauri/Content/Localization/StringTables/ST_*.uexp
python3 -I tools/font_coverage.py extracted/fonts/Kauri/Content/Localization/Fonts/*.ufont
```

Pitfalls: `repak unpack` without `-o` extracts **next to the PAK** — always pass `-o`. `retoc to-legacy` needs the whole `Paks` directory, not a single `.utoc`. `retoc asset-registry` cannot read AssetRegistry v22 (UE 5.7) — table references were found through package name maps.

### 2.3 Where the text lives

```
Kauri-Windows.pak  (virtual paths)
Kauri/Content/Localization/
├── Game/      Game.locmeta          NativeCulture=en, CompiledCultures=[de,en,fr,ja,ko,zh-Hans]
│              <culture>/Game.locres 5795 entries
├── Settings/  Settings.locmeta, <culture>/Settings.locres   323
├── Lyra/      Lyra.locmeta,     <culture>/Lyra.locres       75
├── Script/    Script.locmeta,   <culture>/Script.locres     63
└── Fonts/     Beaufort-*, Hideout-*, Noto-Sans/*  (.ufont)
Kauri/Config/DefaultGame.ini         CulturesToStage, LocalizationPaths
```

| Text | Container | Path / object | Format and field | Example |
|---|---|---|---|---|
| Blueprint and data texts (bulk) | `Kauri-Windows.pak` | `Localization/Game/<culture>/Game.locres` | locres v3, namespace `""`, key — a 32-hex-digit GUID; 5305 entries | `""` / `00822BC349AED70BF60A72A4DAF2F57C`: "Join! JOIN!" |
| StringTable strings (translations) | `Kauri-Windows.pak` | same `Game.locres` | namespace = table name (`ST_Common_Menu`, `ST_Dialogue`, `ST_MonsterNames`… — 10 tables, 488 keys) | `ST_Common_Menu` / `Menu_Quit`: "Quit", `source_hash 0x1D9A681F`; fr — "Quitter" |
| Settings menu | `Kauri-Windows.pak` | `Localization/Settings/<culture>/Settings.locres` | namespace `Wayfinder` and others | `Wayfinder` / `AntiAliasingMedium`: "Medium" |
| Common buttons and loading tips | `Kauri-Windows.pak` | `Localization/Lyra/<culture>/Lyra.locres` | namespaces `""`, `ST_MissingLocalization`, `Messaging`, `CommonGame` | `""` / `596BD2C9…0188`: "Controller" |
| Chat and system messages | `Kauri-Windows.pak` | `Localization/Script/<culture>/Script.locres` | namespaces `ItemOperation`, `Error`, `Chat`, `Interaction`… | `EngagementDefinition` / `InputPrompt`: "Listen" |
| A target's culture list | `Kauri-Windows.pak` | `Localization/<Target>/<Target>.locmeta` | `CompiledCultures` | the original `Settings` lists only `en` |
| StringTable source (English only) | `Kauri-Windows.utoc/.ucas` | `/Game/Localization/StringTables/ST_*` | `UStringTable`: Namespace, Entries, MetaData | `ST_Common_Menu`: 115 keys |
| A widget's reference to a string | `Kauri-Windows.utoc/.ucas` | `WBP_FrontEndMenu` (Zen package) | FText: `Flags=0`, `HistoryType=0x0B` (StringTableEntry), `TableId=FName#6` (`/Game/Localization/StringTables/ST_Common_Menu.ST_Common_Menu`), `Key="Menu_Quit"`; offset 45431, name batch @60 | `ST_Common_Menu` — referenced by 54 packages, `ST_Dialogue` — by 93 |
| Fonts | PAK: `.ufont`; IoStore: `*_Font.uasset` (composite) | `Localization/Fonts/` | `.ufont` = `i32 size \| sfnt \| 4 bytes`; composite: DefaultTypeface + FallbackTypeface Noto Sans SC/JP/KR | Cyrillic: Beaufort, Hideout — 0/66; Noto Sans, Noto Sans CJK — 66/66 |

Bottom line: the translatable text lives **entirely in the PAK** (`.locres`); IoStore holds only the tables' English sources and widget references. The chain `WBP_FrontEndMenu` → `ST_Common_Menu/Menu_Quit` → `Game.locres[ST_Common_Menu][Menu_Quit]` was confirmed statically and by run R2.

`.locres` v3 layout (`Optimized_CityHash64_UTF16`), little-endian:

```
magic GUID (16) | u8 version=3 | i64 string array offset | u32 entry count
u32 namespace count
  per namespace: u32 hash | FString name | u32 key count
    per key: u32 hash | FString key | u32 source_hash | i32 string index
string array: i32 count | per string: FString text | i32 refcount
FString: i32 length; >0 — ASCII with NUL, <0 — UTF-16LE with NUL (length in chars)
```

- `source_hash` = `FCrc::StrCrc32` of the English source (CRC32 over UTF-32LE) — matched 5795/5795.
- Strings are shared: "Quitter" has refcount 3 (also `ST_Common_Gameplay/Gameplay_LeaveDungeon`). Never edit a string in place: repoint the entry to a new string and decrement the old one's refcount.
- `.locmeta`: `magic GUID (16) | u8 version | FString NativeCulture | FString NativeLocRes | TArray<FString> CompiledCultures | u32 bIsUGC`.
- StringTable in `.uexp` (after `retoc to-legacy`): property prefix, `FString Namespace`, `TMap<FString,FString> Entries`, `TMap<FString,TMap<FName,FString>> MetaData`, then the package footer `C1 83 2A 9E`. The parser accepts an offset only if the structure ends exactly at the footer.

### 2.4 Editing and building

Canonical format — `translation/ru/<Target>.json` (`Game`, `Settings`, `Lyra`, `Script`):

```json
{"format": "wyldheart-l10n/1", "target": "Game", "culture": "ru", "source_culture": "en",
 "entries": [{"id": "Game/ST_Common_Menu/Menu_Quit", "namespace": "ST_Common_Menu", "key": "Menu_Quit",
              "source": "Quit", "source_hash": "0x1D9A681F", "translation": "Выход"}]}
```

- Entries are sorted by `(namespace, key)`; an empty `translation` builds as the English source; `note: "mt:…"` marks machine translation, `note: "manual: …"` a fix from a screenshot.
- `python3 -I tools/l10n.py extract` — from `en/<Target>.locres`; `validate` — `source_hash`, `{…}` placeholders, markup tags, dialogue directives and choice brackets, line-break count, edge whitespace, `stale` entries.
- `python3 -I tools/l10n.py build --name z_Wyldheart-RU_P`: per target `en/<T>.locres` → `ru/<T>.locres` through `locres_set` (namespaces, keys and hashes preserved; the build fails if the canonical `source_hash` differs from the game's), `ru` is added to `<T>.locmeta`; then
  `repak pack --version V11 -m ../../../ tree/ z_Wyldheart-RU_P.pak`,
  `retoc to-zen --version UE5_7 <empty dir> z_Wyldheart-RU_P.utoc` (only `.utoc/.ucas` are kept),
  check: `repak unpack` and `diff -r` against the build tree. repak writes entries in nondeterministic order, so a build's identity is the content `manifest.sha256`, not the PAK hash.
- `tools/make_dist.sh VERSION` — archive with Windows (`.bat` + `.ps1`) and Linux (`.sh`) installers.

### 2.5 Capture and verification (Linux/Steam, Proton)

- Staging: `cp -as "<Steam>/Wyldheart Demo" staging/` — symlinks to every file; mod files only in `staging/…/Paks`.
- Steam launch option: `…/tools/steam-staging-wrapper.sh %command% -culture=ru` — swaps the exe path to staging, writes `logs/launch-*.log` (argv, `found_original_exe`).
- `/proc/<pid>/fd` is unavailable (pressure-vessel user namespace → Permission denied) and Shipping writes no logs. So `tools/observe_run.sh RN` → inotify on the original and staging `Paks`: how often `z_…_P.pak/.utoc/.ucas` were opened, whether the original directory was listed.
- Signs: the mod PAK opened once and closed → rejected (R1); `.pak` + `.utoc` + `.ucas` opened and read → mounted (R2). Plus a screenshot and `GameUserSettings.ini` in `compatdata/4475270/…`.
- After every run — SHA-256 of the original containers and exe against the baseline (`research/stage0/baseline-hashes.txt`).
- Trap: Steam may reset the launch options — the game then silently starts from the original (R8). No fresh `logs/launch-*` → the run is invalid.

## 3. Experiments

| Runs | What was tested | Result |
|---|---|---|
| R-1–R0 | control: original and staging, `-culture=fr` | "Quitter"; `*.pak` lookup happens in staging |
| R1–R2 | `_P` mod PAK with one string in `fr/Game.locres`: without `.utoc`, then with an empty IoStore companion | **M1** → **M4** «РУССКИЙ ТЕСТ» |
| R3–R5 | new culture `ru` (`locmeta` + `ru/*.locres`), `-culture=ru`, a second target `Settings` | **M4**; each target is needed separately |
| R6–R7 | build from the canonical JSON, small targets complete | **M4** |
| R8 | — | invalid: Steam reset the launch options |
| R9–R11 | full translation, length fixes, release name `z_Wyldheart-RU_P` from the installer | **M4** |

The full log is in the translation repository's README.

Conclusions that cost the most time:
- **"IoStore present, so the PAK is unused" — wrong.** Localization lives in the PAK; IoStore only requires an empty `.utoc/.ucas` companion.
- **Each target separately.** Without `ru` in `Settings` the settings menu falls back to `en` even when `Game` is translated.
- **locres strings are shared** — an in-place edit changes other keys too.
- **Fonts needed no changes:** Beaufort/Hideout lack Cyrillic, but the composite fonts take it from the Noto Sans CJK fallback; only the typeface differs.

## 4. Final mod

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

## 5. Translation

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

## 6. Known limitations

- `Game`: 17 of 5795 strings untranslated; machine strings (`mt:`) not fully proofread.
- VSync shows an English "OFF" — source not found (probably the `Engine` target, not verified).
- Inconsistent address: `Game` uses informal «ты», `Lyra` formal «вы».
- Decorative headings are drawn by the Noto Sans CJK fallback — the typeface differs from Latin; «ё» not checked on screen.
- Long-string clipping checked in the menu, Options and Status; inventory, skills, trade and dialogues — not.
- The Windows installer (`.bat`, PowerShell 5.1, registry lookup) was never run on Windows; Steam Deck not tested.
