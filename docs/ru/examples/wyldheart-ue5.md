# Пример: Wyldheart Demo (Unreal Engine 5.7) — русская локализация с нуля

[← К оглавлению](../../../README.md) · Механика: [Unreal Engine](../engines/unreal-engine.md)

Демо-версия Wyldheart в Steam поставляется с языками en, de, fr, ja, ko, zh-Hans. Русского нет. Ниже — как он был добавлен **отдельным mod-архивом**, без изменения ни одного файла игры.

> **Готовый перевод:** [релиз wyldheart-ru-v0.1.0](https://github.com/Artemedi/game-localization-guide/releases/tag/wyldheart-ru-v0.1.0) — архив с установщиком для Windows и Linux/Steam Deck. Неофициальный, в основном машинный перевод с правками; рассчитан на сборку демо Steam buildid 25823320.

## 1. Разведка

| Что | Значение | Как установлено |
|---|---|---|
| Движок | UE **5.7.4**, ветка `++kauri+stage-CL-39317`, Shipping, форк с AngelScript; проект `Kauri` | `<EngineVersion>` в CrashContext.runtime-xml, строки `Kauri.exe`, `Kauri/Script/*.Cache` |
| Сборка | Windows (`Kauri/Binaries/Win64/Kauri.exe`), на Linux — через Proton; Steam appid 4475270, depot 4475271, buildid 25823320 | SHA-1 файлов совпали с `depotcache/4475271_*.manifest` |
| Контейнеры | смешанная схема: `Kauri-Windows.pak` + `Kauri-Windows.utoc/.ucas` (IoStore) + `global.utoc/.ucas` | `repak info`, `retoc info` |
| Выбор языка | есть: en, de, fr, ja, ko, zh-Hans (`DefaultGame.ini` → `CulturesToStage`); выбор сохраняется в `GameUserSettings.ini` → `[Internationalization] Culture=` | ini из PAK, профиль в `compatdata` |
| Логи | в Shipping нет: категорий `LogPak`/`LogTextLocalizationManager` в exe нет, `-abslog` файл не создал | строки exe, запуск |

## 2. Данные и инструменты

### 2.1 Инструменты

| Инструмент | Версия | Для чего |
|---|---|---|
| repak (`repak_cli`) | 0.2.3 | `info`/`list`/`unpack` PAK; `pack` мод-PAK |
| retoc (`retoc_cli`) | 0.1.5 | `info`/`list` IoStore, `to-legacy` (Zen → `.uasset/.uexp`), `to-zen` пустой IoStore-компаньон |
| Python | 3.14 | свои парсеры, без сторонних пакетов |
| REA + Ghidra | — | статика `Kauri.exe`: как ищутся `*.pak`, приоритет `_P`, требование `.utoc` |
| inotify, `sha256sum` | — | открытие файлов в `Paks`, сверка оригинала |
| Claude Opus 5.5 | через Wello API | машинный перевод таргета `Game` (`tools/l10n_mt.py`) |
| свои скрипты | репозиторий `wyldheart-ru/tools/` | `locres.py` (чтение/запись locres v0–3, побайтовый round-trip всех 24 файлов), `locres_set.py`, `locmeta.py`, `stringtable.py`, `font_coverage.py`, `l10n.py` (extract/validate/build), `make_dist.sh`, `watch_open.py`, `observe_run.sh`, `steam-staging-wrapper.sh` |

### 2.2 Распаковка

`PAKS` = `Wyldheart Demo/Kauri/Content/Paks`. Всё извлекается в `extracted/` репозитория перевода (не в Git) и не редактируется.

```bash
repak info  PAKS/Kauri-Windows.pak         # V11 (Fnv64BugFix), mount ../../../, Oodle, без шифрования, 4764 файла
repak list  PAKS/Kauri-Windows.pak
repak unpack -q -o extracted/stage0-pak -i Kauri/Content/Localization/Game ... -i Kauri/Config PAKS/Kauri-Windows.pak
retoc info  PAKS/Kauri-Windows.utoc        # 36 274 чанка / 25 276 пакетов, Oodle; global.utoc — 1 чанк
retoc list --path PAKS/Kauri-Windows.utoc
retoc to-legacy --no-shaders -f /Localization/StringTables/ PAKS/ extracted/stage0-iostore   # нужен весь каталог Paks (global.utoc)
python3 -I tools/locres.py dump extracted/stage0-pak/Kauri/Content/Localization/Game/en/Game.locres > game-en.json
python3 -I tools/stringtable.py extracted/stage0-iostore/Kauri/Content/Localization/StringTables/ST_*.uexp
python3 -I tools/font_coverage.py extracted/fonts/Kauri/Content/Localization/Fonts/*.ufont
```

Ловушки: `repak unpack` без `-o` распаковывает **рядом с PAK** — всегда указывать `-o`. `retoc to-legacy` нужен весь каталог `Paks`, не один `.utoc`. `retoc asset-registry` не читает AssetRegistry v22 (UE 5.7) — ссылки на таблицы искались по name map пакетов.

### 2.3 Где лежит текст

```
Kauri-Windows.pak  (виртуальные пути)
Kauri/Content/Localization/
├── Game/      Game.locmeta          NativeCulture=en, CompiledCultures=[de,en,fr,ja,ko,zh-Hans]
│              <culture>/Game.locres 5795 записей
├── Settings/  Settings.locmeta, <culture>/Settings.locres   323
├── Lyra/      Lyra.locmeta,     <culture>/Lyra.locres       75
├── Script/    Script.locmeta,   <culture>/Script.locres     63
└── Fonts/     Beaufort-*, Hideout-*, Noto-Sans/*  (.ufont)
Kauri/Config/DefaultGame.ini         CulturesToStage, LocalizationPaths
```

| Текст | Контейнер | Путь / объект | Формат и поле | Пример |
|---|---|---|---|---|
| Тексты Blueprint и данных (основной объём) | `Kauri-Windows.pak` | `Localization/Game/<culture>/Game.locres` | locres v3, namespace `""`, ключ — GUID из 32 hex-символов; 5305 записей | `""` / `00822BC349AED70BF60A72A4DAF2F57C`: «Join! JOIN!» |
| Строки StringTable (переводы) | `Kauri-Windows.pak` | там же, `Game.locres` | namespace = имя таблицы (`ST_Common_Menu`, `ST_Dialogue`, `ST_MonsterNames`… — 10 таблиц, 488 ключей) | `ST_Common_Menu` / `Menu_Quit`: «Quit», `source_hash 0x1D9A681F`; fr — «Quitter» |
| Меню настроек | `Kauri-Windows.pak` | `Localization/Settings/<culture>/Settings.locres` | namespace `Wayfinder` и др. | `Wayfinder` / `AntiAliasingMedium`: «Medium» |
| Общие кнопки и подсказки загрузки | `Kauri-Windows.pak` | `Localization/Lyra/<culture>/Lyra.locres` | namespace `""`, `ST_MissingLocalization`, `Messaging`, `CommonGame` | `""` / `596BD2C9…0188`: «Controller» |
| Чат и системные сообщения | `Kauri-Windows.pak` | `Localization/Script/<culture>/Script.locres` | namespace `ItemOperation`, `Error`, `Chat`, `Interaction`… | `EngagementDefinition` / `InputPrompt`: «Listen» |
| Список культур таргета | `Kauri-Windows.pak` | `Localization/<Target>/<Target>.locmeta` | `CompiledCultures` | у `Settings` оригинал содержит только `en` |
| Исходник StringTable (только английский) | `Kauri-Windows.utoc/.ucas` | `/Game/Localization/StringTables/ST_*` | `UStringTable`: Namespace, Entries, MetaData | `ST_Common_Menu`: 115 ключей |
| Ссылка виджета на строку | `Kauri-Windows.utoc/.ucas` | `WBP_FrontEndMenu` (Zen-пакет) | FText: `Flags=0`, `HistoryType=0x0B` (StringTableEntry), `TableId=FName#6` (`/Game/Localization/StringTables/ST_Common_Menu.ST_Common_Menu`), `Key="Menu_Quit"`; смещение 45431, name batch @60 | `ST_Common_Menu` — в 54 пакетах, `ST_Dialogue` — в 93 |
| Шрифты | PAK: `.ufont`; IoStore: `*_Font.uasset` (composite) | `Localization/Fonts/` | `.ufont` = `i32 size \| sfnt \| 4 байта`; composite: DefaultTypeface + FallbackTypeface Noto Sans SC/JP/KR | кириллица: Beaufort, Hideout — 0/66; Noto Sans, Noto Sans CJK — 66/66 |

Итог: переводимый текст **целиком в PAK** (`.locres`); IoStore хранит только английские исходники таблиц и ссылки виджетов. Цепочка `WBP_FrontEndMenu` → `ST_Common_Menu/Menu_Quit` → `Game.locres[ST_Common_Menu][Menu_Quit]` подтверждена статически и прогоном R2.

Раскладка `.locres` v3 (`Optimized_CityHash64_UTF16`), little-endian:

```
magic GUID (16) | u8 version=3 | i64 смещение массива строк | u32 число записей
u32 число namespace
  на namespace: u32 хеш | FString имя | u32 число ключей
    на ключ: u32 хеш | FString ключ | u32 source_hash | i32 индекс строки
массив строк: i32 число | на строку: FString текст | i32 refcount
FString: i32 длина; >0 — ASCII с NUL, <0 — UTF-16LE с NUL (длина в символах)
```

- `source_hash` = `FCrc::StrCrc32` английского исходника (CRC32 от UTF-32LE) — совпал для 5795/5795.
- Строки разделяются: «Quitter» — refcount 3 (ещё `ST_Common_Gameplay/Gameplay_LeaveDungeon`). Менять строку на месте нельзя: запись перенаправляется на новую строку, refcount старой уменьшается.
- `.locmeta`: `magic GUID (16) | u8 version | FString NativeCulture | FString NativeLocRes | TArray<FString> CompiledCultures | u32 bIsUGC`.
- StringTable в `.uexp` (после `retoc to-legacy`): префикс свойств, `FString Namespace`, `TMap<FString,FString> Entries`, `TMap<FString,TMap<FName,FString>> MetaData`, затем footer пакета `C1 83 2A 9E`. Парсер принимает смещение, только если структура дочитывается ровно до footer.

### 2.4 Редактирование и сборка

Канонический формат — `translation/ru/<Target>.json` (`Game`, `Settings`, `Lyra`, `Script`):

```json
{"format": "wyldheart-l10n/1", "target": "Game", "culture": "ru", "source_culture": "en",
 "entries": [{"id": "Game/ST_Common_Menu/Menu_Quit", "namespace": "ST_Common_Menu", "key": "Menu_Quit",
              "source": "Quit", "source_hash": "0x1D9A681F", "translation": "Выход"}]}
```

- Записи отсортированы по `(namespace, key)`; пустой `translation` — собирается английским исходником; `note: "mt:…"` — машинный перевод, `note: "manual: …"` — правка по скриншоту.
- `python3 -I tools/l10n.py extract` — из `en/<Target>.locres`; `validate` — `source_hash`, плейсхолдеры `{…}`, теги разметки, директивы и скобки выбора в диалогах, число переносов, пробелы по краям, устаревшие (`stale`) записи.
- `python3 -I tools/l10n.py build --name z_Wyldheart-RU_P`: для каждого таргета `en/<T>.locres` → `ru/<T>.locres` через `locres_set` (namespace, ключи, хеши сохраняются; сборка падает, если `source_hash` канона не совпал с игрой), `ru` добавляется в `<T>.locmeta`; затем
  `repak pack --version V11 -m ../../../ tree/ z_Wyldheart-RU_P.pak`,
  `retoc to-zen --version UE5_7 <пустой каталог> z_Wyldheart-RU_P.utoc` (берутся только `.utoc/.ucas`),
  проверка: `repak unpack` и `diff -r` с деревом сборки. repak пишет записи в недетерминированном порядке, поэтому идентичность сборки — `manifest.sha256` содержимого, а не хеш PAK.
- `tools/make_dist.sh VERSION` — архив с установщиками Windows (`.bat` + `.ps1`) и Linux (`.sh`).

### 2.5 Захват и проверка (Linux/Steam, Proton)

- Staging: `cp -as "<Steam>/Wyldheart Demo" staging/` — симлинки на все файлы; файлы мода только в `staging/…/Paks`.
- Параметр запуска Steam: `…/tools/steam-staging-wrapper.sh %command% -culture=ru` — подменяет путь exe на staging, пишет `logs/launch-*.log` (argv, `found_original_exe`).
- `/proc/<pid>/fd` недоступен (user namespace pressure-vessel → Permission denied), логов Shipping нет. Поэтому `tools/observe_run.sh RN` → inotify на оригинальный и staging `Paks`: сколько раз открыты `z_…_P.pak/.utoc/.ucas`, перечислялся ли оригинальный каталог.
- Признаки: мод-PAK открыт 1 раз и закрыт → отвергнут (R1); `.pak` + `.utoc` + `.ucas` открыты и читаются → смонтирован (R2). Плюс скриншот и `GameUserSettings.ini` в `compatdata/4475270/…`.
- После каждого запуска — SHA-256 оригинальных контейнеров и exe против baseline (`research/stage0/baseline-hashes.txt`).
- Ловушка: Steam может сбросить параметры запуска — игра тихо стартует из оригинала (R8). Нет свежего `logs/launch-*` → прогон недействителен.

## 3. Эксперименты

| Запуски | Что проверялось | Результат |
|---|---|---|
| R-1–R0 | контроль: оригинал и staging, `-culture=fr` | «Quitter»; поиск `*.pak` идёт в staging |
| R1–R2 | мод-PAK `_P` с одной строкой в `fr/Game.locres`: без `.utoc`, затем с пустым IoStore-компаньоном | **M1** → **M4** «РУССКИЙ ТЕСТ» |
| R3–R5 | новая культура `ru` (`locmeta` + `ru/*.locres`), `-culture=ru`, второй таргет `Settings` | **M4**; каждый таргет нужен отдельно |
| R6–R7 | сборка из канонического JSON, малые таргеты целиком | **M4** |
| R8 | — | недействителен: Steam сбросил параметры запуска |
| R9–R11 | полный перевод, правки длин, релизное имя `z_Wyldheart-RU_P` из установщика | **M4** |

Полный журнал — README репозитория перевода.

Выводы, которые стоили больше всего времени:
- **«Есть IoStore — PAK не используется» — неверно.** Локализация лежит в PAK; IoStore требует только пустой компаньон `.utoc/.ucas`.
- **Каждый таргет — отдельно.** Без `ru` в `Settings` меню настроек откатывается на `en`, даже если `Game` переведён.
- **Строки в locres разделяются** — правка «на месте» меняет и чужие ключи.
- **Шрифты трогать не пришлось:** в Beaufort/Hideout кириллицы нет, но composite-шрифты берут её из fallback Noto Sans CJK; отличается только начертание.

## 4. Итоговый мод

```
Kauri/Content/Paks/
├── z_Wyldheart-RU_P.pak     ← mount ../../../, PAK V11, без сжатия
├── z_Wyldheart-RU_P.utoc    ← пустой IoStore-компаньон (обязателен)
└── z_Wyldheart-RU_P.ucas

Содержимое .pak:
Kauri/Content/Localization/Game/Game.locmeta            (+ ru)
Kauri/Content/Localization/Game/ru/Game.locres
Kauri/Content/Localization/Settings/Settings.locmeta    (+ ru)
Kauri/Content/Localization/Settings/ru/Settings.locres
Kauri/Content/Localization/Lyra/Lyra.locmeta            (+ ru)
Kauri/Content/Localization/Lyra/ru/Lyra.locres
Kauri/Content/Localization/Script/Script.locmeta        (+ ru)
Kauri/Content/Localization/Script/ru/Script.locres
```

Каждый `ru/*.locres` = копия `en/*.locres` (те же namespace, ключи и source_hash), в которой заменены только тексты переведённых записей. Непереведённое остаётся английским.

Выбор языка: Настройки → Игровой процесс → Язык → «русский», либо параметр запуска `-culture=ru`. Игра сохраняет выбор в
`%LOCALAPPDATA%/Kauri/Saved/Config/Windows/GameUserSettings.ini` → `[Internationalization] Culture=ru`
(на Linux/Proton — тот же путь внутри `compatdata/<appid>/pfx/…`).

## 5. Перевод

| Таргет | Строк | Как |
|---|---|---|
| Settings | 300 / 323 | вручную; без перевода — DLSS/FSR/XeSS, соотношения сторон, `{0}%` |
| Lyra | 65 / 75 | вручную; без перевода — названия платформ и контроллеров, Discord |
| Script | 62 / 63 | вручную; безличные формы в чате («Выброшено: {DisplayName}») |
| Game | 5778 / 5795 | LLM (Claude Opus 5.5), 3825 уникальных строк, ~420 тыс. токенов, ~15 мин; затем правки по скриншотам |

Правки после проверки в игре (примеры):
- колонка «Состояние» вмещает ~23 символа → «Восст. выносливости», «Сопр. ошеломлению», «Макс. призывов»;
- вкладка инвентаря вмещает ~8 букв → `Pack` = «Вещи»;
- многозначность без контекста: слот `Chest` = «Торс» (а не «Сундук»), `Feet` = «Ступни»;
- значение селектора «Полноэкранное окно» не помещается → «Окно без рамки».

## 6. Известные ограничения

- `Game`: 17 из 5795 строк не переведены; машинные строки (`mt:`) не вычитаны целиком.
- VSync показывает английское «OFF» — источник не найден (предположительно таргет `Engine`, не проверено).
- Обращение не единое: `Game` на «ты», `Lyra` на «вы».
- Декоративные заголовки рисуются fallback-шрифтом Noto Sans CJK — начертание отличается от латиницы; «ё» на экране не проверялась.
- Обрезка длинных строк проверена в меню, Options и «Состоянии»; инвентарь, навыки, торговля, диалоги — нет.
- Установщик Windows (`.bat`, PowerShell 5.1, поиск через реестр) на Windows не запускался; Steam Deck не проверялся.
