# Инструменты

[← К оглавлению](../../README.md)

Отмечено, что применялось в разобранных примерах (✔) и что приведено для справки (·).

## Unreal Engine

| Инструмент | Для чего | |
|---|---|---|
| [repak](https://github.com/trumank/repak) | список/распаковка/упаковка `.pak` (версии V1–V11, mount point) | ✔ |
| [retoc](https://github.com/trumank/retoc) | IoStore `.utoc/.ucas`: список, извлечение, `to-zen` (в т. ч. пустой контейнер-компаньон) | ✔ |
| [FModel](https://github.com/4sval/FModel) | просмотр ассетов и `.locres` в GUI (Windows) | · |
| [UAssetGUI](https://github.com/atenfyr/UAssetGUI) | просмотр/редактирование `.uasset` (StringTable, DataTable) | · |
| [UnrealLocres](https://github.com/akintos/UnrealLocres) | экспорт/импорт `.locres` в CSV | · |
| Ghidra / IDA | статический анализ exe: как монтируются PAK, где ищется `.locmeta` | ✔ |
| Собственный парсер `.locres`/`.locmeta` (Python, ~200 строк) | побайтовый round-trip, замена строк с сохранением ключей и хешей | ✔ |

## Unity

| Инструмент | Для чего | |
|---|---|---|
| [AssetRipper](https://github.com/AssetRipper/AssetRipper) | извлечение ассетов, TextAsset, шрифтов | · |
| [UABEA](https://github.com/nesrak1/UABEA) | просмотр/замена ассетов в `.assets` и бандлах | · |
| [Il2CppDumper](https://github.com/Perfare/Il2CppDumper) | структура классов IL2CPP-сборок | · |
| [BepInEx](https://github.com/BepInEx/BepInEx) 5 + [HarmonyX](https://github.com/BepInEx/HarmonyX) | загрузчик плагинов (Mono) и перехват методов игры: postfix/prefix/транспайлер | ✔ |
| [XUnity.AutoTranslator](https://github.com/bbepis/XUnity.AutoTranslator) | готовая runtime-подмена текста по словарю (плагин BepInEx) | · |
| [UnityPy](https://github.com/K0lb3/UnityPy) | чтение `.assets` из Python: TextAsset, шрифты, сырые MonoBehaviour (поиск подписей без typetree) | ✔ |
| [ILSpy](https://github.com/icsharpcode/ILSpy) (`ilspycmd`) | декомпиляция и IL сборок `Managed/*.dll`: кто читает текст и откуда | ✔ |
| Roslyn `csc.dll` из .NET SDK | сборка плагина против `Managed/*.dll` игры без NuGet | ✔ |

## Godot

| Инструмент | Для чего | |
|---|---|---|
| [gdsdecomp (GDRE Tools)](https://github.com/GDRETools/gdsdecomp) | распаковка `.pck`, декомпиляция ресурсов | · |
| [GodotPCKExplorer](https://github.com/DmitriySalnikov/GodotPCKExplorer) | просмотр/сборка `.pck` | · |

## Общие

| Инструмент | Для чего | |
|---|---|---|
| `ripgrep` / `grep -a` | поиск исходной строки по файлам игры | ✔ |
| `inotifywait` / свой watcher | доказать, что игра **открыла** файл мода (Linux/Proton) | ✔ |
| Process Monitor (Windows) | то же для Windows | · |
| LLM через OpenAI-совместимый API | черновой перевод больших объёмов с глоссарием и автопроверкой | ✔ |
| `sha256sum` | контроль, что оригиналы не изменились; воспроизводимость сборки | ✔ |

## Linux / Steam Deck (Proton)

- Запуск только через Steam: прямой запуск Proton без переменных `STEAM_COMPAT_*` ломает окружение.
- Для экспериментов удобна **staging-копия**: каталог с симлинками на оригинальные файлы + файлы мода, и обёртка в Launch Options (`wrapper.sh %command%`), подменяющая путь к exe. Так мод тестируется, не попадая в установку Steam.
- `/proc/<pid>/fd` игры внутри pressure-vessel может быть недоступен — используйте inotify на каталоге `Paks`.
