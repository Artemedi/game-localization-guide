# Unity [partly verified: Book of Travels, Mono]

[← Engines](README.md)

> The "Runtime plugin" section and rows marked "verified" were confirmed on [Book of Travels](../examples/bookoftravels-unity.md) (Unity 2022.3, Mono, Linux). The rest is general information [overview].

## Where to look for text

| Variant | Location | How to recognize |
|---|---|---|
| **Unity Localization** package | String Table Collections in `*.assets` or Addressables bundles (`StreamingAssets/aa/…/*.bundle`) | `Unity.Localization.dll` in `<Game>_Data/Managed/` |
| **I2 Localization** | `I2Languages` asset (LanguageSourceAsset) with one column per language | `I2Localization` among assemblies/classes |
| Custom format | `TextAsset` (CSV/JSON/XML) in assets or files in `StreamingAssets/` | search for the source string in files; **verified**: CSV in `resources.assets` |
| Labels in scenes and prefabs | the `m_text` field of TextMeshPro components, `m_Text` of `UI.Text` | **verified**: without a type tree, read at its serialized position |
| Text in code | string literals in `Assembly-CSharp.dll` and other `Managed/` assemblies (Mono) or `GameAssembly.dll` + `global-metadata.dat` (IL2CPP) | **verified** (Mono): `ldstr` in IL |

Even when a game has tables, check the rest: in Book of Travels text lived in five places — CSV, prefabs, data baked into MonoBehaviours, literals in code, and asset names the code shows as display names.

## Ways to add a language

1. **A new file in `StreamingAssets/`**, if the game reads translations from there (cleanest option).
2. **Replacing an asset in a bundle/`.assets`** (UABEA etc.) — modifies the original file, backups required. On a staging copy it is fine for proving the text path (Book of Travels, R1); not for a release.
3. **Runtime plugin** — BepInEx + Harmony: a game-specific plugin or XUnity.AutoTranslator with a dictionary. Assets stay untouched.

## Runtime plugin (BepInEx 5, Mono) [verified]

Text is replaced **at its owner** — where the game obtains it — by exact English source. If a game update changes the source, the string stays English instead of showing a stale translation.

| Text owner | Harmony hook |
|---|---|
| Table parser (`CSVReader.Read(TextAsset)` or similar) | postfix: replace fields of the parsed table |
| TMP components in scenes/prefabs | postfix `TextMeshProUGUI.Awake` / `TextMeshPro.Awake` |
| Strings built in code | transpiler: replace `ldstr` operands in the relevant methods; `async` methods keep their code in the state machine's `MoveNext` (`AsyncStateMachineAttribute.StateMachineType`) |
| Code shows an asset or enum name as a display name | transpiler: `Object.name` / `ToString()` call → a method returning the display name |
| A method that outputs a finished string (poems, titles) | prefix: replace the argument or the setter value |

Pitfalls:
- Take **method names** for `AccessTools.Method` from assembly metadata (`strings`, ildasm), not from ILSpy sources: it escapes `[]` as `_005B_005D`.
- **Building the plugin** without NuGet: Roslyn (`csc.dll` from the .NET SDK) with `-r:` to the game's `Managed/*.dll` and `BepInEx/core/*.dll`, `-langversion:7.3`, `-deterministic`.
- **Linux + Steam:** the game runs inside the Steam Linux Runtime container, and `LD_PRELOAD` set outside is lost. The launch option `./run_bepinex.sh %command%` (shipped with BepInEx) re-executes itself inside the container before the exe; `libdoorstop.so` is loaded by name through `LD_LIBRARY_PATH`, so spaces in the game path do not matter.

## Fonts

TextMeshPro uses font atlases (`TMP_FontAsset`) with a fixed character set — the atlas may lack Cyrillic (symptoms: "character … was not found" in `Player.log`, boxes or gaps).

Verified approach (Book of Travels):
- create a **dynamic** `TMP_FontAsset` from a TTF with Cyrillic (`TMP_FontAsset.CreateFontAsset`) and add it to the game fonts' `fallbackFontAssetTable`;
- **one fallback per game font** with `faceInfo.scale` matched to the primary's x-height (`meanLine / pointSize`) — otherwise Cyrillic is larger than Latin and lines overlap;
- **attach before the first layout** — in the TMP component's `Awake` postfix. If a glyph is missing at the first layout, TMP substitutes ` ` and keeps the space even after the fallback appears.

Look for a Cyrillic TTF in the game's own assets (`Font` objects in `.assets`): Book of Travels itself ships `Ubuntu-R.ttf`.

Tools: [tools.md](../tools.md#unity)
