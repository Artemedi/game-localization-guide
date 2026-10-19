# Game Localization Guide

[Русский](README.md) · **English**

A practical guide to adding a new localization (for example, Russian) to games that do not ship one — **without modifying the game's original files**.

There are no ready-made translations or game assets here: only the mechanics, file layouts, tools and a workflow proven on a real game.

## Who it is for

- Fan translators who want to understand *where* a game keeps its text and *how* to slip their own in.
- Translators who want a reproducible process: extract → translate → validate → build → verify in game.

## Where to start

| Step | Document |
|---|---|
| 1. General mechanics: how games find text and why a translation can be layered on top | [docs/en/overview.md](docs/en/overview.md) |
| 2. Identify the engine and find the localization | [docs/en/engines/](docs/en/engines/README.md) |
| 3. Tools | [docs/en/tools.md](docs/en/tools.md) |
| 4. Translation and verification pipeline | [docs/en/pipeline.md](docs/en/pipeline.md) |
| 5. Worked examples: Wyldheart (Unreal Engine 5.7), Book of Travels (Unity 2022.3) | [wyldheart-ue5.md](docs/en/examples/wyldheart-ue5.md), [bookoftravels-unity.md](docs/en/examples/bookoftravels-unity.md) |
| How the repository and releases are organised | [MANIFEST.en.md](MANIFEST.en.md) |

## Ready-made translations

| Game | Engine | Method | Platforms | Page | Release |
|---|---|---|---|---|---|
| Wyldheart Demo | UE 5.7 | `.pak` mod archive + empty `.utoc/.ucas` | Windows, Linux/Steam Deck (Proton) | [example](docs/en/examples/wyldheart-ue5.md) | [0.1.0](https://github.com/Artemedi/game-localization-guide/releases/tag/wyldheart-ru-v0.1.0) |
| Book of Travels | Unity 2022.3 (Mono) | BepInEx plugin | Windows, Linux/Steam Deck (native and Proton) | [example](docs/en/examples/bookoftravels-unity.md) | [0.2.0](https://github.com/Artemedi/game-localization-guide/releases/tag/bookoftravels-ru-v0.2.0) |

## Repository layout

```
README.md / README.en.md        ← this page
MANIFEST.md / MANIFEST.en.md    ← what goes where and in what form: game pages, releases, archive layout
docs/
  ru/ | en/
    overview.md                 ← general localization mechanics
    engines/
      README.md                 ← how to identify the engine
      unreal-engine.md          ← UE4/UE5: .locres/.locmeta, PAK + IoStore  [verified]
      unity.md                  ← Unity: text, BepInEx runtime plugin, TMP fonts  [partly verified]
      godot.md                  ← Godot: .translation/.po inside .pck        [overview]
    tools.md                    ← tools per engine
    pipeline.md                 ← canonical format, checks, build, in-game test
    examples/
      wyldheart-ue5.md          ← full example with paths inside the archives
      bookoftravels-unity.md    ← Unity without a language option: runtime plugin
```

Labels: **[verified]** — confirmed in game on a real project; **[overview]** — commonly known information, not verified within this guide.

## Ground rules

1. **Never touch the originals.** The translation is a separate "mod" file that the game loads on top of its own. Delete it and the game is original again.
2. **Prove first, translate later.** One test string on screen is worth more than a thousand translated strings the game never reads.
3. **One variable per experiment.** If something fails, it must be clear what.
4. **Automatically check what breaks the game:** placeholders `{0}`, tags `<b>…</>`, script markup, line breaks.
5. **The final check is on screen only.** A string missing from a log does not prove the game did not load it.

## License

Guide texts — [CC BY 4.0](LICENSE). Game and engine names belong to their owners; the guide contains none of their assets. Archives in [Releases](https://github.com/Artemedi/game-localization-guide/releases) are unofficial fan translations built from the game's text; they are not affiliated with the developers and are not covered by CC BY 4.0.
