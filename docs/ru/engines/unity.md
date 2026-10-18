# Unity [проверено частично: Book of Travels, Mono]

[← Движки](README.md)

> Раздел «Runtime-плагин» и строки с пометкой «проверено» подтверждены на [Book of Travels](../examples/bookoftravels-unity.md) (Unity 2022.3, Mono, Linux). Остальное — общие сведения [обзор].

## Где искать текст

| Вариант | Где лежит | Как распознать |
|---|---|---|
| Пакет **Unity Localization** | String Table Collections в `*.assets` или Addressables-бандлах (`StreamingAssets/aa/…/*.bundle`) | `Unity.Localization.dll` в `<Game>_Data/Managed/` |
| **I2 Localization** | ассет `I2Languages` (LanguageSourceAsset) с колонками языков | `I2Localization` в списке сборок/классов |
| Свой формат | `TextAsset` (CSV/JSON/XML) в ассетах или файлы в `StreamingAssets/` | поиск исходной строки по файлам; **проверено**: CSV в `resources.assets` |
| Подписи в сценах и префабах | поле `m_text` компонентов TextMeshPro, `m_Text` у `UI.Text` | **проверено**: без typetree читается по позиции сериализации |
| Текст в коде | строковые литералы в `Assembly-CSharp.dll` и других сборках `Managed/` (Mono) или `GameAssembly.dll` + `global-metadata.dat` (IL2CPP) | **проверено** (Mono): `ldstr` в IL |

Даже если у игры есть таблицы, проверьте остальное: в Book of Travels текст был в пяти местах — CSV, префабы, вшитые в MonoBehaviour данные, литералы в коде и имена ассетов, которые код показывает как названия.

## Способы добавить язык

1. **Новый файл в `StreamingAssets/`**, если игра читает переводы оттуда (самый чистый вариант).
2. **Замена ассета в бандле/`.assets`** (UABEA и т. п.) — меняет оригинальный файл, нужны резервные копии. На копии в staging годится, чтобы доказать путь текста (Book of Travels, R1), для выпуска — нет.
3. **Runtime-плагин** — BepInEx + Harmony: свой плагин под игру или XUnity.AutoTranslator со словарём. Ассеты не меняются.

## Runtime-плагин (BepInEx 5, Mono) [проверено]

Текст подменяется **у его владельца** — там, где игра его получает, — по точному английскому исходнику. Если обновление игры изменило исходник, строка остаётся английской, а не показывает устаревший перевод.

| Владелец текста | Перехват Harmony |
|---|---|
| Парсер таблиц (`CSVReader.Read(TextAsset)` и т. п.) | postfix: заменить поля разобранной таблицы |
| Компоненты TMP в сценах/префабах | postfix `TextMeshProUGUI.Awake` / `TextMeshPro.Awake` |
| Строки, собранные кодом | транспайлер: замена операнда `ldstr` в нужных методах; у `async`-методов код лежит в `MoveNext` машины состояний (`AsyncStateMachineAttribute.StateMachineType`) |
| Код показывает имя ассета или enum как название | транспайлер: вызов `Object.name` / `ToString()` → метод, возвращающий отображаемое имя |
| Метод, который выводит готовую строку (стихи, звания) | prefix: заменить аргумент или значение сеттера |

Подводные камни:
- **Имена методов** для `AccessTools.Method` брать из метаданных сборки (`strings`, ildasm), а не из исходников ILSpy: он экранирует `[]` как `_005B_005D`.
- **Сборка плагина** без NuGet: Roslyn (`csc.dll` из .NET SDK) с `-r:` на `Managed/*.dll` игры и `BepInEx/core/*.dll`, `-langversion:7.3`, `-deterministic`.
- **Linux + Steam:** игра идёт в контейнере Steam Linux Runtime, `LD_PRELOAD`, заданный снаружи, теряется. Параметр запуска `./run_bepinex.sh %command%` (из поставки BepInEx) перезапускает себя внутри контейнера перед exe; `libdoorstop.so` грузится по имени через `LD_LIBRARY_PATH`, поэтому пробелы в пути игры не мешают.

## Шрифты

TextMeshPro использует атласы шрифтов (`TMP_FontAsset`) с фиксированным набором символов — кириллицы в атласе может не быть (признак — «character … was not found» в `Player.log`, квадраты или пустоты).

Проверенное решение (Book of Travels):
- создать **динамический** `TMP_FontAsset` из TTF с кириллицей (`TMP_FontAsset.CreateFontAsset`) и добавить его в `fallbackFontAssetTable` шрифтов игры;
- **отдельный fallback на каждый шрифт** с `faceInfo.scale`, выровненным по x-height (`meanLine / pointSize`) основного шрифта, — иначе кириллица крупнее латиницы и строки налезают;
- **подключать до первой разметки** — в postfix `Awake` TMP-компонента. Если при первой разметке глифа нет, TMP подставляет ` ` и оставляет пробел даже после появления fallback.

TTF с кириллицей стоит поискать в ассетах самой игры (`Font`-объекты в `.assets`): Book of Travels сама поставляет `Ubuntu-R.ttf`.

Инструменты: [tools.md](../tools.md#unity)
