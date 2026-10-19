# Пример: Book of Travels (Unity 2022.3, Mono) — русский перевод runtime-плагином

[← К оглавлению](../../../README.md) · Механика: [Unity](../engines/unity.md)

В Book of Travels нет выбора языка: все тексты английские и лежат внутри `resources.assets`, а шрифты интерфейса без кириллицы. Русский добавлен **плагином BepInEx**, который подменяет тексты и подключает шрифт во время работы игры. Ни один файл игры не изменяется.

> **Готовый перевод:** [релиз bookoftravels-ru-v0.2.0](https://github.com/Artemedi/game-localization-guide/releases/tag/bookoftravels-ru-v0.2.0) — архив с установщиками для Windows (`install.bat`) и Linux / Steam Deck (`install.sh`: нативная версия или Windows-версия через Proton). Неофициальный, в основном машинный перевод с выборочной вычиткой; рассчитан на Steam buildid 24468107. В игре проверены нативная Linux-версия и Windows-версия через Proton; Windows без Proton и Steam Deck — нет.

## 1. Разведка

| Что | Значение | Как установлено |
|---|---|---|
| Движок | Unity **2022.3.62f2**, скриптовый рантайм **Mono** (`BookOfTravels_Data/Managed/*.dll`) | версия в `globalgamemanagers`, наличие `Managed/` |
| Сборка | нативная Linux (`BookOfTravels.x86_64`), Steam appid 1152340, buildid 24468107; запускается в контейнере Steam Linux Runtime (pressure-vessel). Windows-версия (`BookOfTravels.exe`, тот же buildid): тексты в `resources.assets` и точки перехвата в `Managed/*.dll` совпадают с Linux-версией (сравнение файлов и IL) | файлы игры, лог параметров запуска |
| Контейнеры | обычные сериализованные файлы Unity (`resources.assets`, `sharedassets*.assets`, `levelN`), без бандлов и Addressables; без typetree | UnityPy |
| Выбор языка | нет. Поле `chosenLanguage` загрузчика строк нигде не присваивается; кроме `English` других языковых колонок нет | декомпиляция, CSV |

## 2. Данные и инструменты

### 2.1 Инструменты

| Инструмент | Версия | Для чего |
|---|---|---|
| UnityPy | 1.25.4 (`uv venv`) | чтение `.assets`: инвентарь TextAsset и шрифтов, сырые MonoBehaviour, извлечение TTF; копия `resources.assets` для PoC R1–R2 |
| ilspycmd | `DOTNET_ROLL_FORWARD=Major` | декомпиляция `Managed/*.dll` (`-p` проект, `-il` IL): кто читает текст, где он склеивается |
| `strings` (binutils) | — | точные имена методов из метаданных сборки (ILSpy их экранирует) |
| BepInEx | 5.4.23.5 linux x64 (UnityDoorstop, HarmonyX, MonoMod, Cecil) | загрузчик плагина, перехваты Harmony |
| Roslyn `csc.dll` | из .NET SDK | сборка плагина без NuGet: `-langversion:7.3 -deterministic`, `-r:` на `Managed/*.dll` игры и `BepInEx/core/*.dll` |
| Claude Opus 5.5 | через Wello API | машинный перевод (`tools/l10n_mt.py`, `tools/npc_titles.py translate`) |
| inotify, `sha256sum` | — | наблюдение за открытием файлов, сверка оригинала |
| свои скрипты | репозиторий `bookoftravels-ru/tools/` | `inventory_textassets.py`, `inventory_fonts.py`, `extract_texts.py`, `extract_ui_text.py`, `extract_baked_poems.py`, `npc_titles.py`, `export_translations.py`, `make_poc.py`, `build_plugin.sh`, `build_dist.sh`, `watch_open.py`, `observe_run.sh` |

### 2.2 Распаковка

На диск ничего не распаковывалось: скрипты читают `.assets` через UnityPy только на чтение и сразу пишут канонический JSON. `DATA` — `BookOfTravels_Data` (бралась staging-копия).

```bash
.venv/bin/python -I tools/inventory_textassets.py DATA research/stage0/textassets.tsv   # 293 TextAsset: файл, path_id, имя, размер, sha256
.venv/bin/python -I tools/inventory_fonts.py DATA                                        # TMP_FontAsset → research/stage0/fonts.tsv
.venv/bin/python -I tools/extract_texts.py translation/ru DATA/resources.assets DATA/sharedassets33.assets
.venv/bin/python -I tools/extract_ui_text.py DATA                                        # → translation/ru/UI - Prefab Text.json
.venv/bin/python -I tools/extract_baked_poems.py DATA/resources.assets                   # → CharacterVariationsManager - PoemData.json
python3 -I tools/npc_titles.py build                                                     # → translation/ru/npc_titles.json
DOTNET_ROLL_FORWARD=Major ilspycmd -p -r DATA/Managed -o decomp/General DATA/Managed/Years.General.General.Assembly.dll
strings DATA/Managed/Years.General.General.Assembly.dll | grep 'Item\[\]'                # настоящее имя метода для Harmony
```

### 2.3 Где лежит текст

| Текст | Контейнер | Объект | Формат и поле | Пример |
|---|---|---|---|---|
| Таблицы игры (основной объём) | `resources.assets` — 285 TextAsset; `sharedassets33.assets` — 8 (титры) | TextAsset по имени, напр. `GUI Copy - InfoPrompt`, path_id 4791, 14 040 байт | `m_Script`: CSV в UTF-8, первая строка — заголовок. 56 таблиц с колонкой `English` | заголовок `ID,English,Note`; строка `INFPR_00221,Are you sure you want to exit the game?,` |
| Подписи интерфейса | `resources.assets`, `sharedassets*.assets`, `levelN` | MonoBehaviour `TextMeshProUGUI`, `TextMeshPro`, `UnityEngine.UI.Text` — 2144 компонента, 641 уникальный текст | `m_text` (TMP) / `m_Text` (UI.Text), читается по позиции сериализации (см. ниже) | «Settings», «Exit game», «Cost: » |
| Стихи NPC (показываемые) | `resources.assets` | MonoBehaviour `CharacterVariationsManager`, 12 объектов с `[/P]` (path_id 98037, 99129, 100400, 98228, …; до 270 КБ) | `List<PoemData{string poem; int poemTag}>`, без typetree; распознаётся по форме «строка + int32 0..3» | `Ah... Yes...[XX]You are so very much more apparent to me now, mortal.…` |
| Стихи NPC (CSV) | `resources.assets` | TextAsset `ClassPoems - *`, `GossipPoems - *`, `WalkByPoems - *`, `OtherPoems - *`, `CustomPoems - *` (66 таблиц) | колонки `Common/Uncommon/Rare/Unique`, `Poem`; метка времени `[E]/[M]/[N]`, разметка `[P]`, `[XX]` | загрузчик `LoadPoems` вызывается только из редактора — в игре эти CSV не читаются, служат словарём для вшитых стихов |
| Звания NPC | `resources.assets` | TextAsset `NpcNames - Characteristics`, `NpcNames - Occupation - <Профессия>` (+ `- Neutral`); MonoBehaviour `npcNameOverrides`; enum типов NPC в `Years.General.Base.Assembly.dll` | сервер собирает `Characteristic + " " + Occupation` или имя enum, хранит в `NPC.syncNameSurface`, в сохранении — `SNS:<title>` | `Seasoned Gate Warden`, `Peasant`, `Commoner` |
| Строки из кода | `Managed/Years.General.General.Assembly.dll` | 7 методов, напр. `UISettings.SetTooltipSpeed`, `EquipmentView.GetItemTypeText` | IL `ldstr`; для типов предметов — `constrained. ItemType; callvirt Object::ToString` (IL_0156) | `"Tool tip speed: " + v + " sec"`; `Reagent`, `Edible` |
| Имена реагентов | assets | имя ассета предмета (`Object.name`), enum `ReagentSubType` | код выводит имя ассета вместо названия из таблицы | `Chestnut shells x3` при переведённом `REAG_00002` |
| Шрифты | `resources.assets` | TMP_FontAsset `Traveler-pt234` (path_id 92861), `HighTower-pt169` (92860), `LiberationSans SDF` (99861); Font `Ubuntu-R` (11533) | TMP — статический атлас 1024², кириллицы нет; `Ubuntu-R` — TTF с полной кириллицей | R2: 125× «character … was not found» |

Правила разбора CSV — как у игры, `CSVReader.Read(TextAsset)` в `Years.General.Base.Assembly.dll`:
- строки делятся по `\r\n|\n\r|\n|\r`, поля — по `,(?=(?:[^"]*"[^"]*")*(?![^"]*"))`;
- у значения обрезаются `"`, удаляются `\`; строка с пустым первым полем пропускается; значение, которое принимает `int/float.TryParse`, становится числом (не переводится);
- `[X]` в тексте — перенос строки.

Загрузка: `CharacterVariationsManager.Awake()` → `LoadGUICopy`, `LoadItemNames`, … → `CSVReader.Read` → `List<Dictionary<string, object>>`. Поэтому `source` в переводе — значение после разбора, а не сырая строка CSV.

Подписи интерфейса без typetree: заголовок MonoBehaviour (`m_GameObject`, `m_Enabled`, `m_Script`, `m_Name` со смещения 28), затем Graphic (`m_Material` 12, `m_Color` 16, `m_RaycastTarget` 4, `m_RaycastPadding` 16), MaskableGraphic (`m_Maskable` 4, `m_OnCullStateChanged` — число вызовов обязано быть 0), затем `TMP_Text.m_text` или 56 байт `m_FontData` и `UI.Text.m_Text`. Класс компонента — по MonoScript из `globalgamemanagers.assets`. Компонент, не подходящий под раскладку, пропускается и считается.

### 2.4 Редактирование и сборка

Канонический формат — `translation/ru/<asset>.json`, один файл на таблицу:

```json
{"asset": "GUI Copy - InfoPrompt", "file": "resources.assets", "id_column": "ID",
 "entries": [{"key": "INFPR_00001|English", "id": "INFPR_00001", "column": "English",
              "source": "You cannot discard favourite items",
              "translation": "Вы не можете выбросить любимые предметы",
              "note": "mt:claude-opus-5.5-stable"}]}
```

- Тот же формат у `UI - Prefab Text.json` (колонка `UIText`), `CharacterVariationsManager - PoemData.json` (`PoemData`) и `Code - UI literals.json` (колонка `Type.Method`, ключ `#n|Type.Method`; составлен вручную по декомпиляции).
- `npc_titles.json`: прилагательные — формы `m/f/n/pl` и позиция `pre/post`, существительные — с родом.
- Извлечение повторяемо: при неизменном `(key, source)` перевод и `note` сохраняются. Машинные строки — `note: mt:…`, ручные правки — без `mt:`.
- Сборка: `tools/export_translations.py` → `translations.tsv` (`asset, column, id, source, translation`; `\\ \t \n \r` экранируются), `tools/npc_titles.py export` → `npc_titles.tsv`, `tools/build_plugin.sh` → `BookOfTravels.RuFont.dll` + `Ubuntu-R.ttf` (извлекается из `resources.assets`), `tools/build_dist.sh VERSION` → архив.
- Ассеты не редактировались. Исключение — PoC R1–R2: `tools/make_poc.py` менял одну строку CSV в **копии** `resources.assets` в staging и проверял, что изменился только этот объект (пересохранение UnityPy меняет ещё поле размера в заголовке, смещение 28, и 4 байта выравнивания).

### 2.5 Захват и проверка (Linux/Steam)

- Staging (`tools/build_staging.sh`): симлинки на файлы игры; настоящие копии — исполняемые файлы, `UnityPlayer.so` и `resources.assets` (Unity выводит путь `*_Data` из пути exe).
- Параметр запуска Steam: `…/tools/steam-staging-wrapper.sh %command%` — переписывает путь exe на staging, пишет `logs/launch-*.log`. Если в staging есть `run_bepinex.sh`, обёртка запускает его — как у игрока.
- `tools/observe_run.sh RN` перед запуском: inotify (`watch_open.py`) на каталоги оригинала и staging, копия `Player.log` и `BepInEx/LogOutput.log`, после выхода — sha256 202 ключевых файлов оригинала.
- Windows-версия через Proton (R22–R23): staging копирует `BookOfTravels.exe`, `UnityPlayer.dll`, `UnityCrashHandler*.exe` и лаунчер; параметр запуска `WINEDLLOVERRIDES="winhttp=n,b" …/steam-staging-wrapper.sh %command%`; `Player.log` — в префиксе Wine `compatdata/1152340/pfx/drive_c/users/steamuser/AppData/LocalLow/MightAndDelight/`; baseline — 369 ключевых файлов.
- Доказательство прогона: в launch-логе `rewritten_args ≥ 1`; путь Mono в `Player.log` — staging; 0 «Unicode value … not found»; строки плагина («patched …», «translated N rows in '…'»); оригинальный `resources.assets` не открывался; CLOSE_WRITE 0; baseline ALL-OK; скриншот или подтверждение пользователя.

## 3. Эксперименты

| Запуски | Что проверялось | Результат |
|---|---|---|
| R0–R2 | путь текста: строка в копии `resources.assets` — ASCII, затем кириллица | **M4** маркер виден; **M5** кириллицы нет в шрифтах |
| R3–R5 | загрузка BepInEx в контейнере Steam; fallback-шрифт из `Ubuntu-R.ttf` | **M0** с `LD_PRELOAD` снаружи → загрузка внутри контейнера → **M4** кириллица |
| R6–R10 | перевод без изменения ассетов: postfix `CSVReader.Read`, звания NPC, полный машинный перевод | **M4**; стихи и подписи префабов не из CSV |
| R11–R16 | стихи (`NPC.ShowPoem`), подписи префабов, шрифт до первой разметки | **M3** → **M4** |
| R17–R21 | строки из кода, имена реагентов, типы предметов; архив с установщиком | **M4** |
| R22–R23 | тот же архив в Windows-версии через Proton (`winhttp.dll`) | **M0** без `WINEDLLOVERRIDES` → **M4** |

Полный журнал прогонов с доказательствами — README репозитория перевода.

Выводы, которые стоили больше всего времени:
- **Не весь текст — в таблицах.** Подписи префабов, вшитые стихи, строки из кода и имена ассетов пришлось ловить отдельно, каждое у своего владельца.
- **TextMeshPro запоминает пробелы вместо отсутствующих глифов.** Fallback нужно подключать в `Awake` компонента, до первой разметки.
- **Linux + Steam: `LD_PRELOAD` снаружи контейнера теряется.** Работает штатный `./run_bepinex.sh %command%`.
- **Proton: без `WINEDLLOVERRIDES="winhttp=n,b"` Wine берёт свой `winhttp.dll`**, и BepInEx не загружается без единой ошибки в логах (R22).
- **Имена методов — из метаданных, не из декомпилятора** (`_005B_005D` вместо `[]`).

## 4. Итоговый мод

```
Book of Travels/                       (каталог игры в Steam)
├── run_bepinex.sh                     ← Linux: параметр запуска Steam ./run_bepinex.sh %command%
├── libdoorstop.so, .doorstop_version  ← UnityDoorstop (Linux): загружает BepInEx до кода игры
├── winhttp.dll, doorstop_config.ini   ← UnityDoorstop (Windows); под Proton: WINEDLLOVERRIDES="winhttp=n,b" %command%
└── BepInEx/
    ├── core/                          ← BepInEx 5.4.23.5 (HarmonyX, MonoMod, Cecil)
    └── plugins/RuFont/
        ├── BookOfTravels.RuFont.dll   ← плагин перевода
        ├── translations.tsv           ← asset, column, id, source, translation (19 076 строк)
        ├── npc_titles.tsv             ← части званий NPC с формами рода (1301 строка)
        ├── Ubuntu-R.ttf               ← шрифт для fallback (тот же файл, что в ассетах игры)
        └── installed-files.txt        ← список, по которому работает uninstall.sh
```

Что делает плагин (каждая подмена — по точному английскому исходнику; изменился текст после обновления → остаётся английский):

| Где текст | Перехват |
|---|---|
| CSV-таблицы | postfix `CSVReader.Read(TextAsset)` — замена поля, если текущее значение = source |
| Подписи префабов | postfix `TextMeshProUGUI.Awake` / `TextMeshPro.Awake` (там же подключается fallback-шрифт) |
| Стихи NPC | prefix `NPC.ShowPoem(string, …)` |
| Звания NPC | prefix сеттера `NPC.NetworksyncNameSurface`: «Seasoned» + «Gate Warden» → «бывалый» + «привратный страж» (м. р.) |
| Строки из кода | транспайлер: `ldstr` в перечисленных методах; для async — в `MoveNext` машины состояний |
| Типы предметов | транспайлер: результат `enum.ToString()` в `EquipmentView.GetItemTypeText` — через таблицу литералов |
| Имена реагентов | транспайлер: `Object.name` / `ReagentSubType.ToString()` → отображаемое имя предмета |
| Шрифты | fallback из `Ubuntu-R.ttf` на каждый `TMP_FontAsset`, масштаб выровнен по x-height |

## 5. Перевод

| Область | Объём | Как |
|---|---|---|
| CSV-таблицы | 281 таблица, 19 726 записей, 17 883 уникальные строки, ~1,77 млн символов | LLM (Claude Opus 5.5), каждая уникальная строка один раз в контексте своей таблицы; ~12,7 млн входных + 3,7 млн выходных токенов |
| Подписи префабов | 641 строка, 631 переведена | LLM; клавиши, бренды и заглушки оставлены |
| Стихи NPC | 1807 (из них 101 вшитый без CSV) | LLM с сохранением разметки `[P]`, `[XX]` |
| Звания NPC | 1301 часть: прилагательные в 4 формах (м/ж/ср/мн) + существительные с родом | LLM, ответ — формы, а не готовые звания |
| Строки из кода | 18 литералов в 6 методах | вручную, с учётом склейки (`" sec"` → `" с"`) |

Глоссарий, инструкции переводчику («вы» к игроку, нейтральный род), автопроверка разметки и числа `\t\n\r`. Все машинные строки помечены `mt:` для вычитки.

## 6. Известные ограничения

- Название погоды в причинах недоступности навыка — имя enum на английском.
- Имена кочевников (`NomadNames`) не переведены.
- «Играть без сет…» обрезается в кнопке, «Сбросить положение элементов интерфейса» не помещается.
- Стоимость реагентов на карточке навыка и сообщение «не хватает реагентов» переведены, но на экране не проверены.
- `UnityEngine.UI.Text` (старые подписи, в основном отладочные) не перехватывается.
- Windows без Proton и Steam Deck в игре не проверялись; `install.ps1`/`uninstall.ps1` проверены только на тестовых каталогах (PowerShell 7 на Linux).
