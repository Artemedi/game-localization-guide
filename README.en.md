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
| 5. Worked example: Wyldheart (Unreal Engine 5.7) | [docs/en/examples/wyldheart-ue5.md](docs/en/examples/wyldheart-ue5.md) |

## Repository layout

```
README.md / README.en.md        ← this page
docs/
  ru/ | en/
    overview.md                 ← general localization mechanics
    engines/
      README.md                 ← how to identify the engine
      unreal-engine.md          ← UE4/UE5: .locres/.locmeta, PAK + IoStore  [verified]
      unity.md                  ← Unity: where to look for text              [overview]
      godot.md                  ← Godot: .translation/.po inside .pck        [overview]
    tools.md                    ← tools per engine
    pipeline.md                 ← canonical format, checks, build, in-game test
    examples/
      wyldheart-ue5.md          ← full example with paths inside the archives
```

Labels: **[verified]** — confirmed in game on a real project; **[overview]** — commonly known information, not verified within this guide.

## Ground rules

1. **Never touch the originals.** The translation is a separate "mod" file that the game loads on top of its own. Delete it and the game is original again.
2. **Prove first, translate later.** One test string on screen is worth more than a thousand translated strings the game never reads.
3. **One variable per experiment.** If something fails, it must be clear what.
4. **Automatically check what breaks the game:** placeholders `{0}`, tags `<b>…</>`, script markup, line breaks.
5. **The final check is on screen only.** A string missing from a log does not prove the game did not load it.

## License

Guide texts — [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Game and engine names belong to their owners; the guide contains none of their assets.
