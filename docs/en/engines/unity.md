# Unity [overview]

[← Engines](README.md)

> General information; not verified on a real game within this guide.

## Where to look for text

| Variant | Location | How to recognize |
|---|---|---|
| **Unity Localization** package | String Table Collections in `*.assets` or Addressables bundles (`StreamingAssets/aa/…/*.bundle`) | `Unity.Localization.dll` in `<Game>_Data/Managed/` |
| **I2 Localization** | `I2Languages` asset (LanguageSourceAsset) with one column per language | `I2Localization` among assemblies/classes |
| Custom format | `TextAsset` (CSV/JSON/XML) in assets or files in `StreamingAssets/` | search for the source string in files |
| Text in code | string literals in `Assembly-CSharp.dll` (Mono) or `GameAssembly.dll` + `global-metadata.dat` (IL2CPP) | — |

## Ways to add a language

1. **A new file in `StreamingAssets/`**, if the game reads translations from there (cleanest option).
2. **Replacing an asset in a bundle/`.assets`** (UABEA etc.) — modifies the original file, backups required.
3. **Runtime interception** — XUnity.AutoTranslator (via BepInEx/MelonLoader) with an `ru` dictionary in text files: does not touch assets, works in almost any Unity game.

## Fonts

TextMeshPro uses font atlases (`TMP_FontAsset`) with a fixed character set — the atlas may lack Cyrillic. The usual fix is a fallback atlas with Cyrillic or a font swap via a runtime plugin.

Tools: [tools.md](../tools.md#unity)
