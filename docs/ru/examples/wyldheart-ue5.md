# Пример: Wyldheart Demo (Unreal Engine 5.7) — русская локализация с нуля

[← К оглавлению](../../../README.md) · Механика: [Unreal Engine](../engines/unreal-engine.md)

Демо-версия Wyldheart в Steam поставляется с языками en, de, fr, ja, ko, zh-Hans. Русского нет. Ниже — как он был добавлен **отдельным mod-архивом**, без изменения ни одного файла игры.

## 1. Разведка

| Что | Значение | Как установлено |
|---|---|---|
| Движок | UE **5.7.4**, shipping, кодовое имя проекта `Kauri` | строка `EngineVersion` в CrashContext, строки в exe |
| Контейнеры | смешанная схема: `Kauri-Windows.pak` (V11, Oodle, без шифрования, mount `../../../`) + `Kauri-Windows.utoc/.ucas` (IoStore) + `global.utoc/.ucas` | `repak info`, `retoc info` |
| Где текст | только в PAK: `.locres/.locmeta`, ini, шрифты `.ufont` | `repak list` |
| Где интерфейс и StringTables | в IoStore (`.uasset`); StringTables хранят только английский | `retoc list`, разбор `.uexp` |
| Таргеты | `Game` (5795 записей), `Settings` (323, меню настроек), `Lyra` (75, общие кнопки/подсказки), `Script` (63, чат и системные сообщения) | содержимое `Content/Localization/` |

Дерево внутри архива (виртуальные пути):

```
Kauri/Content/Localization/
├── Game/
│   ├── Game.locmeta              NativeCulture=en, CompiledCultures=[de,en,fr,ja,ko,zh-Hans]
│   ├── en/Game.locres            5795 записей: 10 StringTable-namespace + 5305 Blueprint-текстов (namespace "")
│   ├── fr/Game.locres
│   └── …
├── Settings/  Settings.locmeta, <culture>/Settings.locres
├── Lyra/      Lyra.locmeta,     <culture>/Lyra.locres
└── Script/    Script.locmeta,   <culture>/Script.locres
Kauri/Content/Localization/Fonts/  Beaufort-*, Hideout-*, Noto-Sans/*   (.ufont)
```

## 2. Эксперименты (по одной переменной)

| Запуск | Что изменено | Результат |
|---|---|---|
| R0 | ничего (контроль, язык fr) | меню на французском, «Quitter» |
| R1 | + `z_…_P.pak` с `fr/Game.locres`, где одна строка = «РУССКИЙ ТЕСТ» | **M1**: PAK открыт, но не смонтирован — у IoStore-игр нужен парный `.utoc` |
| R2 | + пустые `z_…_P.utoc/.ucas` (`retoc to-zen` из пустого каталога) | **M4**: «РУССКИЙ ТЕСТ» вместо «Quitter» |
| R3 | новая культура: `ru/Game.locres` + `ru` в `Game.locmeta` | **M4**: в настройках появился «русский», выбор → строка видна |
| R4 | запуск с `-culture=ru` | **M4** сразу; но меню настроек английское → это другой таргет (`Settings`) |
| R5 | + `ru` для таргета `Settings` | **M4**: настройки по-русски |
| R6 | сборка из канонического JSON (4 таргета, тестовые строки) | **M4**: конвейер подтверждён |
| R7 | Settings, Lyra, Script переведены полностью | **M4** |
| R9 | Game переведён (LLM, 5778/5795) | **M4**: весь интерфейс, дневник, чат, инвентарь; найдены обрезки длинных подписей |

Выводы, которые стоили больше всего времени:
- **«Есть IoStore — значит PAK не используется» — неверно.** Вся локализация лежит именно в PAK; IoStore лишь требует пустого компаньона.
- **Каждый таргет — отдельно.** Без `ru` в `Settings` меню настроек остаётся английским, даже если `Game` переведён.
- **Шрифты не пришлось трогать.** В декоративных шрифтах (Beaufort, Hideout) кириллицы 0 из 66 букв, но composite-шрифты имеют fallback на Noto Sans SC/JP/KR, где кириллица есть (66/66). Квадратов нет, отличается только начертание.

## 3. Итоговый мод

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

## 4. Перевод

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

## 5. Проверка без риска для установки (Linux/Steam)

- Staging-каталог: симлинки на все оригинальные файлы + файлы мода в `Paks/`.
- В Launch Options Steam — обёртка `wrapper.sh %command% -culture=ru`, которая подменяет путь к exe на staging-копию и пишет лог запуска.
- Наблюдение — inotify на каталоги `Paks` (оригинальный и staging): видно, открыт ли файл мода.
- После каждого запуска — сверка SHA-256 всех оригинальных контейнеров с манифестом Steam.
- Ловушка: Steam может сбросить Launch Options — тогда игра тихо стартует из оригинала (в эксперименте это был R8). Проверяйте наличие свежего лога обёртки.
