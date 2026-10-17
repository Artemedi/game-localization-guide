# Unity [обзор]

[← Движки](README.md)

> Общие сведения; в рамках этого руководства на реальной игре не проверялись.

## Где искать текст

| Вариант | Где лежит | Как распознать |
|---|---|---|
| Пакет **Unity Localization** | String Table Collections в `*.assets` или Addressables-бандлах (`StreamingAssets/aa/…/*.bundle`) | `Unity.Localization.dll` в `<Game>_Data/Managed/` |
| **I2 Localization** | ассет `I2Languages` (LanguageSourceAsset) с колонками языков | `I2Localization` в списке сборок/классов |
| Свой формат | `TextAsset` (CSV/JSON/XML) в ассетах или файлы в `StreamingAssets/` | поиск исходной строки по файлам |
| Текст в коде | строковые литералы в `Assembly-CSharp.dll` (Mono) или `GameAssembly.dll` + `global-metadata.dat` (IL2CPP) | — |

## Способы добавить язык

1. **Новый файл в `StreamingAssets/`**, если игра читает переводы оттуда (самый чистый вариант).
2. **Замена ассета в бандле/`.assets`** (UABEA и т. п.) — меняет оригинальный файл, нужны резервные копии.
3. **Runtime-перехват** — XUnity.AutoTranslator (через BepInEx/MelonLoader) со словарём `ru` в текстовых файлах: не трогает ассеты, работает почти в любой Unity-игре.

## Шрифты

TextMeshPro использует атласы шрифтов (`TMP_FontAsset`) с фиксированным набором символов — кириллицы в атласе может не быть. Обычное решение — добавить fallback-атлас с кириллицей или подменить шрифт runtime-плагином.

Инструменты: [tools.md](../tools.md#unity)
