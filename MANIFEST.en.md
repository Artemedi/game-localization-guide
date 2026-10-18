# Repository manifest

[Русский](MANIFEST.md) · **English**

What is kept in this repository and its releases, where, and in what form. A new game, page or release follows these rules.

## 1. What is here and what is not

| Here | Not here |
|---|---|
| The guide: mechanics, engines, tools, pipeline | The game's working translation repository (`<game>-ru`): translation sources, scripts, run logs |
| One page per analysed game | Game assets, original containers, extracted files |
| Ready-made translation archives — **only in [Releases](https://github.com/Artemedi/game-localization-guide/releases)** | Archives, DLLs or other binaries in Git |
| | API keys, logs containing keys, the user's personal paths |

## 2. Layout

```
README.md / README.en.md          entry point: steps, list of games, rules
MANIFEST.md / MANIFEST.en.md      this document
LICENSE                           CC BY 4.0 (guide texts only)
docs/<lang>/                      ru and en, identical file sets
  overview.md                     general mechanics, M0–M5 scale
  pipeline.md                     canonical format, checks, LLM, in-game verification
  tools.md                        tools per engine: ✔ used / · reference
  engines/README.md               how to identify the engine
  engines/<engine>.md             engine mechanics; label [verified: <game>] or [overview]
  examples/<game>-<engine>.md     game page (section 3)
```

- Every document exists in both languages under the same name: `docs/ru/X.md` ↔ `docs/en/X.md`. Changing one means changing both in the same commit.
- `<game>` is a short lowercase slug without spaces (`wyldheart`, `bookoftravels`); `<engine>` is `ue5`, `unity`, `godot`…
- Labels: **[verified]** — the mechanism was confirmed in game within this guide (name the game); **[overview]** — common knowledge, not verified.

## 3. Game page: `docs/<lang>/examples/<game>-<engine>.md`

The page is a technical report: tables, paths, commands, offsets. Prose only where the reason is unclear without it; no more than a couple of sentences in a row.

Required sections, in this order:

0. **Header**: links to the contents and to the engine page; one sentence on which languages are missing and how Russian was added.
   A "**Ready-made translation:**" block — link to the release, installer platforms, translation quality, game version (Steam buildid).
1. **Reconnaissance** — table "What | Value | How established": engine and version, runtime, containers, whether there is a language option.
2. **Data and tools** — the main section of the page: where the data came from and how. Subsections:
   1. **Tools** — table "Tool | Version | Used for". Only what was actually used; own scripts named by their file in the game's repository.
   2. **Unpacking** — commands (paths as `<game>/…`), where things were extracted, what was read from them; tool pitfalls (e.g. extracting next to the original).
   3. **Where the text lives** — a table row per text source: "Text | Container | Object / inner path | Format and field | Example". Down to the identifier: archive path, `path_id`, asset name, namespace/key, column, offset — whatever applies. The example is a real key and source string. Then the format layout or parsing rules, if they had to be reconstructed (fields, hashes, encoding).
   4. **Editing and building** — the canonical translation format (schema of one entry), what builds it and into what; what was edited by hand and what was generated. Extracted originals are never edited.
   5. **Capture and verification** — how runs were observed: staging, wrapper, inotify, game/loader logs, screenshots, hash checks; which signs counted as proof.
3. **Experiments** — brief: table "Runs | What was tested | Result", groups of runs on one row, M0–M5 codes, at most ~8 rows. The full log lives in the game's repository. Then 3–5 conclusions that cost the most time.
4. **Final mod** — tree of the files added to the game directory and what each does.
5. **Translation** — volume, method (manual / LLM, model), what stayed English and why, fixes from screenshots.
6. **Known limitations** — what does not work or was not verified.

Facts only when confirmed by a run or by files; conclusions from code and hypotheses are labelled as such. A line missing from a log is not proof. API keys, personal paths and logs containing them never go on the page.
When a game confirms an engine mechanism, update `engines/<engine>.md` (label `[verified: <game>]`) and `tools.md` (✔), and add the game to the tables in `README.md`/`README.en.md`.

## 4. Translation releases

| Field | Format | Example |
|---|---|---|
| Tag | `<game>-ru-v<semver>` | `bookoftravels-ru-v0.1.0` |
| Title | `<Game> — русский перевод <version>` | `Book of Travels — русский перевод 0.1.0` |
| Type | pre-release while the translation is unreviewed machine output | |
| Asset | one archive `<Game>-RU-<version>.zip` | `BookOfTravels-RU-0.1.0.zip` |

The version is the translation's (semver): patch — text fixes, minor — new text areas or mechanisms, major — an incompatible change of the install method. The game version (buildid) goes into the archive README and the release notes.

### Archive contents

```
<Game>-RU-<version>/
  README.txt            RU, then EN: contents, install, enable, uninstall, notes
  install.sh            Linux / Steam Deck  (install.bat + install.ps1 if Windows is supported)
  uninstall.sh          (uninstall.bat + uninstall.ps1)
  files/                exactly what is copied into the game directory, at the same relative paths
  checksums.sha256      sha256 of the files in files/; the installer checks them before copying
  THIRD_PARTY.txt       for bundled third-party components (mod loader, font): author, version, license, source
  licenses/             full texts of those licenses
```

Installer requirements:
- it only **adds** files; it never overwrites or deletes the game's original files;
- it finds the game through Steam (`libraryfolders.vdf`) or takes the path as an argument, and checks it is the right game;
- it records the added files; `uninstall` removes only those, and only directories left empty;
- re-running it is safe (it updates its own files);
- it downloads nothing and installs no other software;
- whatever a script cannot do safely (for example Steam launch options) is printed as an instruction.

The archive is built by a script in the game repository (`tools/build_dist.sh` or similar), reproducibly from committed sources.

### Release notes (bilingual)

1. One line: unofficial localization of `<Game>` (Steam, buildid N).
2. **Установка / Install** per platform; manual option.
3. **Включение / Enable** — how to select the language or enable the mod.
4. What changes in the game directory and how to remove it.
5. **Состояние / Status**: what is translated (numbers), translation quality, known issues, **what it was verified on and what not**.
6. "Не связано с разработчиками игры. / Not affiliated with the game's developers."
7. SHA-256 of the archive.

### Pre-publication checklist

- [ ] archive built by the script from a committed state; `checksums.sha256` matches;
- [ ] installer and uninstaller tested on a test directory (not the Steam install);
- [ ] an in-game run with the archive installed: translation visible, originals unchanged (hash check);
- [ ] third-party licenses included;
- [ ] game page (ru/en) and `README` updated, release link points to the tag;
- [ ] release notes state honestly what was not verified.
