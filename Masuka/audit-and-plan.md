# Masuka: схема существующего кода, сверка с ТЗ и план работ

Документ составлен по состоянию рабочего дерева на момент аудита.
Проверялись: `vince.masuka`, `MiracleEngine/Source/Tools/Masuka.{c,h}`,
`MiracleEngine/Source/Tools/*.masuka_example`, все `*.masuka` проекта,
`Samples/Masuka/Source/MasukaTest.c`, сгенерированный `build.sh`,
`Makefile`/`*-premake5.lua`/`Scripts/Linux/*` (текущая рабочая сборка),
`Documentation/Promts/Masuka-refactoring-and-features.txt`.

---

## 1. Инвентаризация файлов билд-системы

| Файл | Строк | Роль | Состояние |
|---|---:|---|---|
| `MiracleEngine/Source/Tools/Masuka.h` | 349 | Типы, API, декларации | Часть типов не используется / устарела |
| `MiracleEngine/Source/Tools/Masuka.c` | 2746 | Вся реализация (CLI, парсер языка, проекты, генераторы, компилятор, runner) | Одна реализация: `MskFile → MskProject → build.sh`; компилятор/runner — заглушки (M4/M5) |
| `vince.masuka` | 35 | Корневой файл workspace: `let`, `entry`, `builds`, `platforms` | Соответствует канону проекта |
| `MiracleEngine/miracle.masuka` | 37 | Проект движка (`StaticLib`) | Канон |
| `Samples/Masuka/masuka.masuka` | 31 | Проект `masuka` (`Executable`) | Есть битые пути (`Projects/Masuka/Source/`) |
| `Dependencies/{stb,MiniAudio,qoi}/*.masuka` | 9 | Проекты зависимостей (`StaticLib`) | Канон |
| `MiracleEngine/Source/Tools/vince.masuka_example` | 57 | Пример «старого» формата с `project`-блоками внутри workspace | **Конфликтует** с каноническим форматом из проекта |
| `MiracleEngine/Source/Tools/format.masuka_example` | 7 | Осколок формата | Мусор |
| `Samples/Masuka/Source/MasukaTest.c` | ~160 | `main` → `msk_cli_start` | Активен, весь старый код закомментирован |
| `build.sh` (корень) | 11 | Артефакт, сгенерированный текущей Masuka | **Невалиден** (см. §4, дефект D4/D5) |
| `Makefile`, `*-premake5.lua`, `Scripts/Linux/*` | — | Единственная реально работающая сборка (Premake5 + bear) | Миграция не завершена |
| `compile_commands.json` | 90 записей | Сгенерирован Premake (`bear`), **не** Masuka | Masuka его не умеет |

Дополнительно: `bin/Debug-linux-x86_64/...` — артефакты Premake; Masuka генерирует пути
`bin/Debug-Linux/...` (другая схема имён, см. §5).

---

## 2. Схема существующего кода

### 2.1. Поток управления (один путь)

```
 argv ───────────► msk_cli_parse_args()                        Masuka.c
                   └─► MskCliConfig
                              │
 vince.masuka ────► msk_language_vince_new() → msk_language_vince_validate()
                   └─► MskVince {pName, pEntryName, aVariables,
                                 aBuilds[], aPlatforms[]}
                              │
 скан каталогов ──► msk_get_all_files_from_directory("*.masuka")
                   (подкаталоги + корень workspace, M1-10; файлы без
                    `project` пропускаются с WARN)
                   └─► msk_language_file_new() → msk_language_file_validate()
                       └─► MskFile[]  {type, bin, objdir, source[],
                                        dependencies[], links[], include[],
                                        provide-include[], linux:/windows:}
                              │
                   msk_preprocessor_file() + msk_file_apply_platform()
                   └─► {var}, {build}/{platform}, блок платформы
                              │
                   msk_project_from_file()                     M1-6
                       ├─ глобы source → список файлов от каталога проекта
                       ├─ provide-include → MskProject.aProvided
                       └─ -D build/platform на каждый проект
                       └─► MskProject[]
                              │
                   msk_projects_resolve()                      M1-8
                       ├─ проверка имён, дубликатов и зависимостей
                       ├─ топосорт (deps-first) + детект циклов
                       └─ транзитивные provide-include → aIncludes
                              │
        ┌─────────────────────┴──────────────────────────────────────────┐
        │ msk_cli_start                                                  │
        │  msk_compiler_clean()                ─► -c                     │
        │  msk_generator_json_create()         ─► compile_commands.json   │
        │  msk_generator_build_create/_write() ─► build.sh + build_*.sh  │
        │  msk_compiler_compile/_from_scripts()─► system(bash build.sh)  │
        │  msk_runner_run_program()/_tests()   [ЗАГЛУШКИ] (M5-1/M5-3)   │
        └────────────────────────────────────────────────────────────────┘
```

**Ключевой вывод:** дублирующий парсер и «второй этаж» из REFACTORING ZONE удалены
(M1-9). Осталась одна модель данных: `MskFile` (AST) → `MskProject` (resolved) →
генератор. Граф зависимостей строится на `MskProject` (M1-8), поэтому порядок в
`build.sh` детерминированный, а не «как обошли каталоги». Генератор (M3-1…M3-6)
строит из этих проектов валидные скрипты и `compile_commands.json`, а компилятор
(M4-1…M4-6) запускает их через `system()`; runner — M5.

### 2.2. Модель данных (2 уровня + конвертер)

| Уровень | Тип | Назначение | Заполняется | Используется |
|---|---|---|---|---|
| DSL-AST | `MskFile` (`Masuka.h`) | сырой результат парсинга `*.masuka` | `msk_language_file_new` | конвертер (M1-6) |
| DSL-AST | `MskVince` | workspace: vars/builds/platforms/entry | `msk_language_vince_new` | активный путь |
| Resolved | `MskProject` | проект с развёрнутыми путями/глобами/зависимостями/платформой | `msk_project_from_file` (M1-6) | генератор, `-c` |
| Граф | `MskProject[]` + `msk_projects_resolve` | порядок сборки, транзитивные includes | M1-8 | генератор скриптов |
| CLI | `MskCliConfig` | разобранные флаги | `msk_cli_parse_args` | активный путь |
| Не используются | `MskWorkspace`, `MskBatch`, `MskBatchProject`, `MskCompileConfig`, `MskCompileResult` | остатки удалённого «второго этажа» | — | — |
| Компилятор | `MskCompilerConfig`, `MskCompilerScriptsConfig` | проекты/`-j`/build/platform/корень для запуска скриптов | `msk_cli_start` | `msk_compiler_compile` / `_from_scripts` (M4) |
| Пустые | `MskRunnerConfig`, `MskTestRunnerConfig` | — | — | передаются как `{}` в заглушки (M5) |

### 2.3. Карта функций по секциям `Masuka.c`

| Секция | Функции | Статус |
|---|---|---|
| CLI | `msk_cli_start`, `msk_cli_parse_args`, `msk_cli_apply_system_defaults`, `msk_cli_validate_args` | активны, закрыты по ТЗ: все флаги, проверки, дефолты, коды возврата |
| Language | `msk_language_vince_new/_is_valid/_validate/_set_selection`, `msk_language_file_new/_from_memory/_is_valid/_validate`, `msk_file_platform_parse`, `msk_parse_next_*` | активны; валидация с файлом/полем (M1-7); парсер доступен и из памяти (тесты); `msk_parse_next_single_path` возвращает `NULL`; `msk_language_path_char` — мусор |
| Preprocessor | `msk_preprocessor_file`, `msk_preprocess_string_field/_array/_string`, `msk_file_apply_platform` | активны: подстановка в поля и массивы, `{project}` на проект, мерж блока платформы |
| Project | `msk_project_from_file`, `msk_projects_from_files`, `msk_projects_resolve`, `msk_project_sources_resolve` | активны: глобы, `provide-include`, `-D`, граф и топосорт (M1-6/M1-8) |
| Generator | `msk_generator_build_create/_single/_write`, `msk_generator_json_create`, `msk_project_object_path`, `msk_script_quote_word`, `msk_generator_project_script_path/_main_script_path` | активны (M3): главный скрипт + `build_{Project}.sh|.bat`, `compile_commands.json`, экранирование путей |
| Compiler | `msk_compiler_compile`, `_from_scripts`, `msk_compiler_run_script`, `msk_compiler_clean` | активны (M4): `system()` по сгенерированным скриптам, `bin/.masuka-tmp` для сборки из памяти, `-c` |
| Runner | `msk_runner_run_program`, `msk_runner_run_tests` | заглушки (M5-1/M5-3) |
| Utils | `msk_parse_next_*`, `msk_preprocess_string`, `msk_file_regex_match`, `msk_get_all_files_from_directory` | рабочие |
| Refactoring zone | — | удалён (M1-9), `Masuka.c` −1300 строк |

> Номера строк — по снимку аудита; искать по именам функций.

### 2.4. Граф проектов (как он есть в `*.masuka`)

```
 stb ─┐
 qoi ─┼──► MiracleEngine (StaticLib) ──► masuka (Executable, entry)
 MiniAudio ─┘
```

Порядок сборки определяется графом зависимостей: `msk_projects_resolve` (M1-8)
проверяет имена и циклы и сортирует проекты deps-first. В репозитории это
`MiniAudio → stb → qoi → MiracleEngine → masuka`; `build.sh` пишется в этом порядке.
`entry: "masuka"` по-прежнему нужен только для `-r`/`-e` (runner — M5), на порядок
сборки не влияет.

---

## 3. Сверка ТЗ с реализацией

Легенда: ✅ реализовано, 🟡 частично/с дефектами, ❌ отсутствует.

### 3.1. Формат и общее поведение

| Требование ТЗ | Статус | Факт |
|---|---|---|
| Свой описательный язык, `*.masuka`, вход `vince.masuka` | ✅ | Парсер есть, формат совпадает с файлами проекта |
| `vince.masuka`: кастомные `let`-переменные | ✅ | `msk_language_vince_new:354` |
| `vince.masuka`: системные переменные (билды, платформы, entry) | 🟡 | `builds`/`platforms` парсятся; `entry` читается в `pEntryName`, но нигде не используется; `{build}/{platform}` подставляются из `sc_build_to_string`/`sc_os_to_string` (`Debug`/`Linux`), а не из выбранных имён (`debug`/`linux`) |
| Сборка из masuka-файлов | ✅ | `masuka` (без флагов) собирает текущий build/platform: генерация в память → `bash build.sh` через `system()` |
| Сборка через генерацию `.sh/.bat` и их запуск | ✅ | `-gb` пишет скрипты и исполняет `build.sh`; `.bat` — на Windows-хосте (M6) |
| Генерация `compile_commands.json` | ✅ | `msk_generator_json_create` (M3-5): `[{directory, file, output, arguments[]}]`, абсолютные пути, 44 TU на текущем репозитории; `arguments` компилируются clang'ом |
| `-e "test"` подменяет entry | ❌ | `-e` не принимает значение, а включает `IsExecute`; поиск проекта по имени не делается |
| `-a "arg0=\"0\" arg1='1' arg2=2"` | ❌ | Флага нет, полей в `MskCliConfig` нет, argv в runner не передаётся |
| `-a{number}` | ❌ | То же |

### 3.2. Консольные команды (актуальное состояние)

CLI реализован: `msk_cli_parse_args` → `msk_cli_apply_system_defaults` →
`msk_cli_validate_args` (ранняя и полная валидация) → исполнение действий.

| Флаг ТЗ | Назначение по ТЗ | Реализация | Статус |
|---|---|---|---|
| `-b` | выбрать build | парсится, валидируется по `vince.masuka`, резолвится в `BuildIndex`, подставляется в `{build}` | ✅ |
| `-c` | clean | `msk_compiler_clean`: удаляет скрипты, objdir и bin текущего build/platform; работает самостоятельно | ✅ |
| `-gb` | сгенерировать билд-файлы | только генерация, без компиляции; пишет `build.sh` в корень и `build_{Project}.sh` в каталоги проектов | ✅ |
| `-rb` | rebuild = clean + generate + build | полный цикл: clean → запись скриптов → `bash build.sh` | ✅ |
| `-gccj` | compile_commands.json | `msk_generator_json_create`: абсолютные пути, тот же набор флагов, пишет в корень workspace | ✅ |
| `-d` | working directory | проверяется и применяется через `chdir`; относительные пути проекта считаются от него | ✅ |
| `-p` | платформа запуска | парсится, валидируется по `vince.masuka`, резолвится в `PlatformIndex`, подставляется в `{platform}` | ✅ |
| `-j` | кол-во потоков | валидируется `0 < j <= 2 * CPU`; делится между проектами волны и TU внутри проекта; `-j 4` — 1.9 c против 3.5 c у `-j 1` | ✅ |
| `-nc` | без компиляции | отключает компиляцию у любого набора флагов | ✅ |
| `-r` | запуск проекта | включает `IsExecute`; runner — заглушка (M5-1) | 🟡 |
| `-e` | указать точку входа | принимает имя, валидирует по `pProjectName`, включает `IsExecute`; runner — заглушка | 🟡 |
| `-a` | аргументы запуска (<1024) | парсится, хранится с длиной; передача в процесс — M5-2 | ✅ (парсинг) |
| `-a{number}` | аргументы с длиной | `-aN "строка"`: N проверяется (`0 < N`, `N <= 1023`), хранится в `ProgramArgumentsLength` | ✅ (парсинг) |
| `-t` (вне ТЗ) | тесты проектов | флаг есть, runner — заглушка (M5-3) | 🟡 |
| `-h`, `--help` | справка | `msk_cli_print_help`: все флаги, дефолты, коды возврата; ранний выход с кодом `0` | ✅ |
| `--self-test` (вне ТЗ) | самотесты | `msk_tests_run` (M2-8), см. §3.8 | ✅ |

Неизвестный флаг теперь является ошибкой (`pUnknownArgument`), флаг без значения — тоже
(`pMissingValueFlag`); при ошибках CLI печатается подсказка «Run 'masuka -h' for usage.».

### 3.3. Проверки входных параметров

Все проверки ТЗ реализованы в `msk_cli_validate_args` (+ helper-функции):

| Проверка ТЗ | Статус |
|---|---|
| `-d` — строка существует и это каталог | ✅ `sc_path(...) == ScPathItemType_Directory` |
| `-p` — имя есть в `vince.masuka` (platforms) | ✅ `sc_list_index_of` + `sc_string_compare` |
| `-j` — число, `0 < j <= 2 * cpu_count` | ✅ `sc_system_cpu_count()` (новое API `Sc`) |
| `-e` — проект существует, сравнение через `sc_string_compare` по `pName` | ✅ проверка по `MskFile.pProjectName` |
| `-a` — длина строки < 1024 байт | ✅ `MSK_CLI_PROGRAM_ARGUMENTS_MAX_LENGTH` |
| `-a{number}` — число, `0 < number` | ✅ + верхняя граница 1023 |
| `-b` — имя есть в `vince.masuka` | ✅ (сверх ТЗ) |
| неизвестный флаг → ошибка | ✅ (сверх ТЗ) |

Коды возврата: `0` — успех, `1` — ошибка сборки/запуска, `2` — ошибка конфигурации/валидации.

### 3.4. Опциональность (значения по умолчанию)

`msk_cli_apply_system_defaults` вызывается сразу после парсинга и заполняет
только те значения, которые не были переданы явно:

| Параметр | Значение по умолчанию |
|---|---|
| `-b` | `sc_system_info_get().Build` → `debug` / `release` / `distribute` |
| `-p` | `sc_system_info_get().Os` → `linux` / `windows` |
| `-j` | `1` |
| `-d` | каталог текущего exe (`argv[0]`), подъём вверх до ближайшего `vince.masuka`; если не найден — текущий cwd |

Если значение передано явно и невалидно (например `-p foo` или `-b relase`), это
**ошибка** (требование ТЗ), а не тихий откат на системное значение. Откат применяется
только к отсутствующим значениям, а для неявных build/platform — ещё и когда
системное имя не найдено в `vince.masuka` (тогда берётся первый элемент с WARN).

### 3.5. Итог сверки

- Формат `vince.masuka`/`*.masuka` в репозитории **канонический** и парсится.
- **CLI-часть ТЗ закрыта полностью** (флаги, семантика, проверки, коды возврата,
  значения по умолчанию), кроме собственно исполнения: компилятор, runner и
  `compile_commands.json` пока заглушки.
- `-d` теперь действительно меняет рабочий каталог, поэтому `masuka` можно запускать
  из любого каталога (в т.ч. из `bin/...`).
- `{build}`/`{platform}` подставляются из выбранных имён `vince.masuka`;
  `-b release` → `bin/release-linux`.
- Осталось по плану: Этап 1 (пути/глобы/платформа/граф) → Этап 3–5
  (генераторы/компиляция/runner).

Новые API, добавленные в `Sc` на этом этапе:
`sc_system_cpu_count()`, `sc_directory_delete_recursive()`, `sc_directory_set_current()`.

### 3.6. Модель и препроцессинг (M1-2…M1-5)

Что изменилось в языке/препроцессоре и чем это проверено:

| Возможность | Реализация | Проверка |
|---|---|---|
| `{project}` — на проект | ставится в `msk_preprocessor_file`, снимается после | `build.sh`: objdir `.../alpha/...` и `.../beta/...` в одном прогоне |
| Подстановка в массивах | `msk_preprocess_string_field/_array` пишут в `aXxx[i].pName` | `-I/home/.../vulkan`, ни одного `{...}` в `build.sh` |
| `{build}`/`{platform}` | имена из `vince.masuka`, перекрываются `-b/-p` | `-b release -p windows` → `bin/release-windows/...` |
| Платформенные блоки | `msk_file_apply_platform()` после препроцессинга | `linux:` даёт `-DSC_LINUX_ONLY`/`-I...inc-linux`, `windows:` — только `-DSC_WINDOWS_ONLY` |
| `preprocessors:` проекта | новый разбор в `msk_language_file_new` | `-DSC_COMMON` в команде компиляции |

Прочее по препроцессору:

- Если в строке нет `{`, возвращается тот же указатель — без аллокации; поэтому
  `msk_preprocess_string_field` освобождает старое значение только при замене.
- Незакрытый `{` больше не съедает остаток строки; пустой результат — пустая строка,
  а не `NULL`.
- Диагностика не использует `%.*s` (`sc_format` не умеет точность — это вызывало
  SIGSEGV), имя переменной копируется в буфер; `%lld` не используется.
- Прогоны под ASan (`detect_leaks=0`) на репозитории и на тестовом workspace с
  блоками платформ: без отчётов (см. §3.7 — там же исправлены найденные ASan'ом баги).

### 3.7. Проекты, граф и валидация (M1-6…M1-10)

| Возможность | Реализация | Проверка |
|---|---|---|
| Конвертер AST → проект | `msk_project_from_file` — единственный путь к `MskProject` | генератор и `-c` используют `MskProject` |
| Глобы source | `msk_project_sources_resolve` расширяет `Source/**.c` от каталога проекта | `clang -c MiracleEngine/Source/Sc/Sc.c ...` вместо `Source/**.c` |
| `provide-include` | `MskProject.aProvided` от каталога проекта | `-IDependencies/stb/src` у MiracleEngine |
| `-D` build/platform | добавляются в `msk_project_from_file` | `-DSC_BUILD_DEBUG -DSC_PLATFORM_LINUX` в каждой команде |
| Валидация файла | `msk_language_file_validate` + `pFilePath`/`pTypeName` | `broken.masuka: missing or empty 'bin' field`, код `2` |
| Валидация vince | `msk_language_vince_validate` | пустые/дублирующиеся builds/platforms → код `2` |
| Резолв зависимостей | `msk_projects_resolve` | `Project 'u' depends on unknown project 'nope'`, код `2` |
| Циклы | DFS-цвета + стек | `Dependency cycle detected: b -> a -> b`, код `2` |
| Топосорт | post-order DFS, deps-first | порядок в логе и `build.sh`: deps раньше зависимых |
| Транзитивные includes | замыкание `aProvided` по графу | masuka получает `MiracleEngine/Source` и `Dependencies/{stb,qoi,MiniAudio}/src` |
| `*.masuka` в корне | скан файлов корня + фильтр `pProjectName == NULL` | workspace с `rootproj.masuka` рядом с `vince.masuka` собирается |

Побочно исправлено при проверке (найдено ASan и тестовыми workspace'ами):

- **D27.** Парсер `*.masuka` уходил за конец файла: если case-блок съедал `\0`
  (например, `linux` внутри строки), общий `++pRead` шагал за терминатор —
  heap-buffer-overflow. Теперь перед инкрементом стоит `if (c == '\0') break;`.
- **D28.** `linux`/`windows` распознавались как платформенные блоки в любом месте,
  включая `name: "linux"` внутри массива `platforms`. Теперь нужен синтаксис
  блока (`linux:`/`windows:`).
- **D29.** Имя проекта обрезалось по первому не-буквенному символу (`p1` → `p`),
  из-за чего два проекта становились дубликатами. Теперь буквы, цифры и `_`.

### 3.8. Справка и самотесты (M2-7, M2-8)

- `masuka -h` / `--help` — полный список флагов с дефолтами и кодами возврата,
  выполняется до инициализации лога и чтения workspace (код `0`).
- `masuka --self-test` — встроенные тесты `msk_tests_run` прямо в `Masuka.c`
  (можно звать и из `ScAutoTestSample`). Возвращает `0`, если всё прошло, иначе `1`;
  ожидаемые ошибки негативных тестов глушатся отдельным «тихим» бэкендом лога.
  Временные файлы создаются только для проверки глобов (`.tmp-masuka-tests`),
  каталог удаляется до и после прогона.

| Тест | Что проверяет |
|---|---|
| `msk_tests_cli_parse_args` | все флаги, `-nc`, `-gb`, `-d`, `-e`, `-a`, `IsCompile` |
| `msk_tests_cli_parse_args_special` | `-h`, `--help`, `--self-test`, неизвестный флаг, значение-флаг |
| `msk_tests_cli_validate_arguments` | `-a{N}` (N > длины, `0`, без значения) |
| `msk_tests_cli_validate_jobs` | `-j` `0`, `999999`, нечисло, `1` |
| `msk_tests_language_vince` | workspace/entry/builds/platforms, дефолты `{build}`/`{platform}`, ошибки |
| `msk_tests_language_file` | поля проекта, блок `linux:`, `p1` (цифры в имени), ошибки валидации |
| `msk_tests_preprocess_string` | `{var}`, строка без переменных, незакрытый `{` |
| `msk_tests_language_platform_block` | `linux:`/`windows:` vs `name: "linux"`, символы имени |
| `msk_tests_project_from_file` | глобы от каталога проекта, `provide-include`, `-D` project/build/platform |
| `msk_tests_projects_resolve` | топосорт, транзитивные includes, неизвестная зависимость, цикл |
| `msk_tests_generator_object_path` | зеркалирование пути объекта, одинаковые имена в разных каталогах, файлы вне проекта |
| `msk_tests_generator_script_shell` | порядок проектов, `ar rcs` по списку, флаги `-std=c99 -m64 -g -O0/-O2`, `-D`/`-I`, `-shared/-fPIC/.so`, транзитивные `-l`, `links`, rpath, `-j` fan-out |
| `msk_tests_generator_script_batch` | `.bat`: `@echo off`, `cd /d`, `call`, `if errorlevel 1 exit /b 1`, `mkdir`/`del /q`, `llvm-ar`, `.lib/.exe` |
| `msk_tests_generator_json` | `compile_commands.json`: 4 записи, ключи `directory/file/output/arguments`, абсолютные пути, флаги |
| `msk_tests_generator_quoting` | экранирование слов (`'a b'`, `'it'\''s'`, `"a b"`), имена `build.sh`/`build.bat`, host-kind |
| `msk_tests_generator_parallel_waves` | параллельная волна независимых проектов, дележ `-j`, `-j 1` — последовательно |
| `msk_tests_compiler_compile` | smoke-сборка: скрипты в память → `clang` → `ar` → линковка → запуск программы; временные скрипты удалены; повторная сборка (инкрементальная) и провал при битом исходнике |

Итог прогона: **181 проверка в 17 функциях**, отчёт по каждой функции и код возврата.

### 3.9. Генераторы скриптов и `compile_commands.json` (M3-1…M3-6)

Один проход по разрешённым проектам (порядок — из `msk_projects_resolve`) даёт три
артефакта; каждый генерируется и в память, и на диск.

| Артефакт | Где | Что внутри |
|---|---|---|
| `build.sh` / `build.bat` | корень workspace | `set -e` (`@echo off` для batch), `cd` в абсолютный корень, вызовы подскриптов в топологическом порядке |
| `build_{Project}.sh` / `.bat` | каталог проекта | `mkdir -p` каталогов объектов и `bin`, компиляция TU (с `-j` fan-out), `rm -f` + `ar rcs` или линковка |
| `compile_commands.json` | корень workspace (`-gccj`) | `[{directory, file, output, arguments[]}]`, абсолютные пути, те же флаги |

Решения генератора:

- **Флаги компиляции:** `clang -std=c99 -m64`, затем `-g -O0` (build с `optimize: 0`)
  или `-O2`; `-D` из `aPreprocessors` (проект + build + платформа), `-I` из
  `aIncludes` (свои + транзитивные `provide-include`), `-fPIC` для `DynamicLib`.
- **Объекты:** путь зеркалирует дерево исходников (`objdir/Source/Sc/Sc.o` для
  `MiracleEngine/Source/Sc/Sc.c`), поэтому одноимённые файлы не перезаписывают друг друга.
- **Линковка:** объекты явным списком, `-L` для своего и зависимых `bin`, `-l<имена>`
  в порядке «зависимость → её зависимости», затем `links` (запись `-pthread`
  вставляется как флаг, `vulkan` — как `-lvulkan`), rpath `$ORIGIN` при `.so`-зависимостях.
- **`-j`:** внутри проекта компиляция идёт пачками: `pids=()`, `cmd &` + `pids+=("$!")`,
  затем `wait "$pid"` (ошибка любой работы останавливает скрипт из-за `set -e`).
- **Экранирование:** для sh — одинарные кавычки с `'\''`, для cmd — двойные;
  batch-пути получают `\`, для сборки используется `llvm-ar`, `.exe/.lib/.dll`.
- **Пути:** скрипты относительны корня workspace, но начинаются с `cd` в абсолютный
  корень, поэтому их можно запускать из любого каталога; в json пути абсолютные.

Проверено на реальном репозитории: `masuka -nc -gb` пишет 6 скриптов, `bash build.sh`
собирает все 5 проектов (`MiniAudio → stb → qoi → MiracleEngine → masuka`), полученный
`bin/debug-linux/masuka` проходит `--self-test` (181 проверка); `-gccj` даёт 44 TU,
`arguments` которых компилируются clang'ом. На тестовом workspace (проект в каталоге с
пробелом, `-j 2`, StaticLib + DynamicLib + Executable) скрипты собираются с нуля и
результат запускается; `-c` убирает скрипты, objdir и `bin`.

### 3.10. Сборка, инкрементальность и `-j` (M4-1…M4-6)

Сборка — это запуск сгенерированных скриптов; отдельного процессного API нет (§7).

| Режим | Что происходит |
|---|---|
| `masuka` (без `-gb`) | скрипты пишутся в `bin/.masuka-tmp`, исполняются `system("bash ...")` и удаляются (даже при ошибке); `-c` тоже чистит каталог |
| `masuka -gb` | скрипты остаются в каталогах проектов и в корне; сборка (если не `-nc`) запускает `<root>/build.sh` |
| `masuka -rb` | clean → запись скриптов → `bash build.sh` |
| `masuka -c` | удаляет скрипты, `bin/.masuka-tmp`, objdir и `bin/<build>-<platform>` |

- **Ошибки:** `system()` != 0 → `Command failed (status N)` + `Build failed!`, код
  возврата `1`; скрипты идут с `set -e`, поэтому сборка останавливается на первой
  ошибке (в параллельной волне уже запущенные работы дожидаются `wait`).
- **`-j`:** `msk_projects_levels` считает уровни зависимостей; главный скрипт запускает
  волну в параллель (`pids=()`/`cmd &`/`wait "$pid"`), а `-j` делится между проектами
  волны (2 проекта и 4 jobs → по 2). Внутри проекта — fan-out по TU.
  Замер на репозитории: `-j 4` — 1.9 c, `-j 1` — 3.5 c.
- **Инкрементальность (P2):** в скрипте проекта
  `if [ ! -f obj ] || [ src -nt obj ] || [ "$MSK_FLAGS" != "$(cat "$MSK_STAMP")" ]; then … fi`;
  штамп флагов — `objdir/.masuka_flags`, пишется в конце успешной сборки. Смена флагов
  пересобирает все TU проекта; повторный запуск — no-op (0.12 c против 1.9 c).
  Ограничения: заголовки не отслеживаются (нет depfile), `.bat` всегда пересобирает (M6).
- **Проверка ошибок:** битый исходник в тестовом workspace → `clang` пишет ошибку,
  `masuka` возвращает `1` и не оставляет временных скриптов.

---

## 4. Найденные дефекты (по приоритету)

> **Закрыто на этапе CLI:** D12 (`pName` платформенного блока — теперь выставляется
> при парсинге, заодно перестал падать release-билд),
> D15 (`-c`/clean реализован), D20 (принудительные флаги убраны, семантика `-r`/`-e`/`-c`/`-rb`
> приведена к ТЗ), D21 (парсер флагов переписан: точное сравнение, значения не
> трактуются как флаги, `-a{N}`).
>
> **Закрыто на этапе «модель и язык» (M1-2…M1-5, см. [plan.md](plan.md)):** D1
> (подстановка пишется в элементы списков, старое значение освобождается),
> D2 (`{project}` — переменная конкретного проекта: ставится и снимается вокруг его
> препроцессинга), D3 (значения по умолчанию — имена из `vince.masuka`), D11
> (препроцессинг получает выбранные `-b/-p`), а также мерж платформенного блока
> `linux:`/`windows:` в проект и `preprocessors:` на уровне проекта.
>
> **Закрыто на этапе «проекты и граф» (M1-6…M1-10, см. §3.7 и [plan.md](plan.md)):**
> D4 (глобы разворачиваются), D7 (топосорт), D10 (одна модель данных), D26 (скан
> корня); D5 закрыт для `source`, D6 — частично (генератор на `MskProject`).
> Найдены и исправлены D27 (выход за конец файла в парсере), D28 (`linux`/`windows`
> внутри строк распознавались как блоки), D29 (имя проекта обрезалось по цифре).
> Остальные пункты актуальны.

### Блокирующие (из-за них `build.sh` непригоден)

**D1. Подстановка переменных в массивах не работает.**
`msk_preprocessor_file` (`:997–1031`, `:1034–1074`) делает
`MskString s = mskFile.aSources[i]; s.pName = msk_preprocess_string(...)` — правит
**локальную копию структуры**, элемент списка не меняется.
Подтверждено на живом прогоне: в сгенерированном `build.sh` остались
`-I{vulkan-path}/include/vulkan` и `Source/**.c`.
**Закрыто (M1-3):** `msk_preprocess_string_field`/`_array` пишут в `aXxx[i].pName`,
освобождают старое значение и не аллоцируют строку, если `{` нет.

**D2. `{project}` — одна глобальная переменная, а не переменная проекта.**
`msk_language_file_new` (`:935–947`) добавляет `project` в общий список `MskVince`.
Так как препроцессинг идёт после разбора всех файлов, значение — имя **последнего**
разобранного проекта. Подтверждено: в `build.sh` objdir для `qoi`, `stb`, `MiniAudio`
указывает на `.../MiracleEngine/...`.
**Закрыто (M1-2):** `project` больше не попадает в workspace-переменные; он
добавляется в начало `msk_preprocessor_file` и снимается после (`sc_list_remove_index`),
поэтому у каждого проекта свой `{project}`.

**D3. `{build}`/`{platform}` берутся из системы, а не из `vince.masuka`.**
`msk_language_vince_new` (`:292–305`) кладёт `sc_build_to_string()` → `"Debug"` и
`sc_os_to_string()` → `"Linux"`. В итоге `bin/{build}-{platform}` → `bin/Debug-Linux`,
а выбранные `-b debug -p linux` ни на что не влияют. В проекте принято
`bin/debug-linux` (по именам из `vince.masuka`), Premake использует третий вариант
(`bin/Debug-linux-x86_64`).
**Закрыто (M1-4):** переменные создаются пустыми, значения проставляются после
разбора — `aBuilds[0].pName`/`aPlatforms[0].pName`, затем их перекрывает
`msk_language_vince_set_selection()` выбранными `-b/-p`. Системные имена остались
только фолбэком для пустого `vince.masuka`.

**D4. Source-глобы не разворачиваются.** `msk_preprocess_sources` (`:2369`) есть, но
не вызывается; в скрипт попадает `clang -c Source/**.c -o bin/.../**.o`
(`build.sh:2`) — shell такое не раскрывает.
**Закрыто (M1-6):** `msk_project_sources_resolve` разворачивает `Source/**.c` от
каталога проекта; в `build.sh` — явный список файлов (`MiracleEngine/Source/Sc/Sc.c`).

**D5. Пути не согласованы с рабочим каталогом.** Команды пишутся относительно каталога
проекта (`Source/MasukaTest.c`), а выполняются из корня workspace; `cwd`/`-d` не
учитываются. Для `MiracleEngine` нужно `MiracleEngine/Source/...`.
**Закрыто для `source` (M1-6):** пути комбинируются с каталогом проекта
(`MiracleEngine/Source/...`). `include`/`bin`/`objdir` остаются относительно корня
workspace — это зафиксированное соглашение (§5), а не дефект.

**D6. Генератор игнорирует зависимости и модель `MskBatchProject`.**
`msk_generator_build_create_single` (`:1157`) берёт только `mskFile.aIncludes` —
`provide-include` зависимостей (`MiracleEngine/Source`) не добавляются; `dependencies`
подставляются в линковку как `-l:lib{name}.a` (`:1222–1225`), то есть проектные
зависимости и системные библиотеки смешаны. Поля `links`, `provide-include`,
`aPlatforms` не используются.
**Закрыто частично (M1-6/M1-8):** генератор работает на `MskProject`;
`provide-include` зависимостей (транзитивно) попадают в `aIncludes`; платформенные
блоки и `preprocessors` уже применены. Остаётся M3/M4: `links` в линковке, явный
список объектов для `ar`, системные библиотеки.

**D7. Порядок сборки недетерминирован.** Проекты выводятся в порядке обхода каталогов,
без топологической сортировки: `masuka` (Executable) может собраться раньше
`MiracleEngine`; граф зависимостей/циклы не проверяются.
**Закрыто (M1-8):** `msk_projects_resolve` проверяет зависимости и циклы и сортирует
проекты deps-first; порядок в `build.sh` — `MiniAudio → stb → qoi → MiracleEngine → masuka`.

### Критичные

**D8. `ar rcs ... %s/*.o`** (`:1202`, `:2681`) — shell-глоб на один уровень, без
`rm -f` перед `ar`, со старыми объектами. Должен быть явный список объектов из батчей.
**Закрыто (M3-2):** `rm -f <target>` и `ar rcs` по явному списку объектов, которые
зеркалят дерево исходников (`objdir/Source/Sc/Sc.o`), поэтому имена не сталкиваются.

**D9. Линковка только под Linux/GNU и только прямые зависимости.**
`-L./%s` (`:1220`, `:2663`, `:2859`) при `bin = bin/debug-linux` даёт `-L./bin/...`;
`-l:lib%s.a` — GNU-синтаксис; нет системных библиотек (`vulkan`, `xcb*`, `Xrandr`, `m`,
`pthread`, `dl`), нет порядка линковки статических библиотек, нет rpath/`.so`/`.dll`,
нет `.exe`.
**Закрыто (M3-2):** переносимый `-L<dir>` + `-l<name>`; транзитивные зависимости в
порядке линковки; `links` — сырые флаги или имена; `-shared`/`-fPIC`/`.so`/`.dll`,
rpath `$ORIGIN`; `.exe`/`.lib` в batch. Остаётся: системные библиотеки приходят из
`links` (`*.masuka` в репозитории пока пустые — миграция данных на M7-1).

**D10. Два параллельных пайплайна, мост отсутствует.** `MskFile` → `MskProject`
конвертера нет; `msk_parse_msk_file_from_memory` (`:1667`) — неполный дубль парсера
(не читает `provide-include`, `preprocessors`, платформенные блоки, не разворачивает
глобы). Активный путь не вызывает `msk_workspace_get_project`, `msk_project_to_batch`,
`msk_batch_compile`, `msk_batch_project_write_to_file`.
**Закрыто (M1-6/M1-9):** мост — `msk_project_from_file`; дубль парсера и «второй
этаж» удалены (−1300 строк), осталась одна модель `MskFile → MskProject`.

**D11. Выбор build/platform не доезжает до проектов.** В активном пути
`MskBuild.pPreprocessor`/`MskPlatform.pPreprocessor` не добавляются к проектам (это
делает только мёртвый `:1704–1705`). `msk_language_vince_is_valid` проверяет лишь
непустоту, но не выбирает build/platform по `-b/-p`.
**Закрыто частично (M1-4):** выбор `-b/-p` доезжает до `{build}`/`{platform}`.
**Закрыто (M1-6):** `-D` build/platform добавляются в `msk_project_from_file`.
Добавление `pPreprocessor` в каждый проект — сделано.

**D12. Платформенные блоки `linux: { ... }` / `windows: { ... }` парсятся и теряются.**
`msk_file_platform_parse` (`:624`) не выставляет `pName`, поэтому
`msk_preprocessor_file` вызывает `sc_string_length(NULL)` — это и есть WARN в логе
прогона. Мержа платформенных `links/preprocessors/include` в проект нет.
**Закрыто (M1-5):** `pName` выставляется при разборе, а `msk_file_apply_platform()`
вливает блок выбранной платформы в `source/dependencies/links/include/
provide-include/preprocessors` после препроцессинга.

**D13. `compile_commands.json` не генерируется.** `msk_generator_json_create`
(`:1121`) — `return 0`, вызывается с `(MskGeneratorJsonConfig) {}` (`:127`), т.е. даже
не получает данные.
**Закрыто (M3-5):** генератор получает `MskProject[]` и абсолютный корень, пишет
`[{directory, file, output, arguments[]}]`; на текущем репозитории — 44 TU.

**D14. Runner отсутствует.** `msk_runner_run_program` (`:1942`) и `_run_tests`
(`:1949`) — заглушки. Запуск — через `system()` по пути `bin/{build}-{platform}/...`:
argv из `-a` цитируется при сборке командной строки (отдельный процессный API не
вводим, см. §7).

**D15. Нет `clean`, нет `rebuild`.** `MskCliConfig.IsRebuild` парсится (`-r`), но не
читается; каталоги/скрипты не удаляются; API удаления каталога в `Sc` нет.

**D16. `-j` не реализован.** Компиляция вызывается в цикле `msk_batch_compile`
(`:2827`) последовательно; количество CPU неизвестно (`ScSystemInfo` не содержит его).
**Закрыто (M4-2):** `sc_system_cpu_count()` для валидации, уровни зависимостей и
параллельные волны в главном скрипте, fan-out по TU внутри проекта.

### Средние

**D17. Падения при неизвестной зависимости.** `project_get_includes` (`:2468`) и
`project_get_libraries` (`:2493`) разыменовывают `pDepProject` без проверки на NULL;
рекурсия по зависимостям без мемоизации и без детекта циклов (`:2782–2811`, `:2490–2501`).

**D18. `provide-include` зависимостей кладётся не туда.** `msk_project_compile`
(`:2803–2809`) добавляет их в `project.aProvided`, а не в `aIncludes`; зависимости
компилируются повторно при каждом упоминании.

**D19. Коллизии объектных файлов.** `objdir/basename.o` (`:2532`, `:1178`, `:1587`).
Сейчас дубликатов имён в 36 `.c` движка нет, но при добавлении — молчаливая
перезапись. Нужно зеркалировать относительный путь.

**D20. Нарушена семантика флагов и принудительные флаги.**
`msk_cli_start` жёстко ставит `IsGenerateCompileCmdsJson = 1` и
`IsGenerateBuildFiles = 1` (`:35–36`), из-за чего ветка компиляции всегда уходит в
`todo` (`:174–177`) — запуск `masuka` вообще ничего не собирает.
Плюс `-r`/`-e` перепутаны местами относительно ТЗ, `-gccj` назван `-gj`,
`-c`/`-rb`/`-a` отсутствуют.

**D21. Парсер флагов хрупкий.** Условие `if (pArg[0] != '-' && pArg[2] != '\0') continue;`
(`:224`) должно быть `||`; значения, начинающиеся с `-`, трактуются как флаги;
`sc_string_compare_ext` сравнивает только префикс (`Sc.c:758`), поэтому
`-b`/`-p`/`-e` не проверяют конец строки и легко получают ложные совпадения.

**D22. Парсер языка без диагностики и с хрупкими местами.**
`builds` добавляется на каждом `}` (`:474–481`); нет номеров строк, сообщений об
ошибках, escape-последовательностей; массивы не поддерживают кавычки внутри строк;
`msk_language_file_is_valid` (`:971`) объявлена и не используется; `{project}` не
проверяется на уникальность.

**D23. Утечки и владение памятью.** Смешение `char*` из `sc_string_new` и сырых
указателей; `msk_batch_project_write_to_file` очищает/освобождает builder в цикле
(`:2620`, `:2698`); `sc_string_destroy(pType)` (`:843`) после чтения первого символа;
`msk_language_file_new` сохраняет указатель на имя проекта в переменную; повторные
`msk_parse_*` не освобождают строки. Под ASan не проверялось.

**D24. `Masuka.h` содержит rot.** `msk_parse_next_single_path` возвращает `NULL`
(`:1317`); `msk_project_type_to_string` (`Masuka.h:273`) не имеет `return` для
неизвестного типа; `MskCompilationCmd*` не используются; пустые конфиги
`MskCompilerConfig`/`MskRunnerConfig`; часть прототипов дублируется в `.c`
(`:1509–1515`).

**D25. Артефакты и окружение.** `build.sh` в корне — невалидный сгенерированный файл;
рабочая сборка держится на Premake + `bear` + `Scripts/Linux/*`, которые ссылаются на
`bin/Debug-linux-x86_64`; `Samples/Masuka/masuka.masuka` содержит несуществующий
include `Projects/Masuka/Source/`.

**D26. `*.masuka` в корне workspace не находятся.** `msk_cli_start` обходит только
*подкаталоги* из `sc_directory_get_items(pRootDirectory, …)` и глобит внутри них
(`:1702–1726`), поэтому проект, лежащий рядом с `vince.masuka`, в сборку не попадает.
Сейчас незаметно: все проекты в подкаталогах. Проверено: workspace с `a.masuka` в
корне даёт пустой `build.sh`. Нужно добавлять файлы корня (и отфильтровывать
`pProjectName == NULL`, чтобы сам `vince.masuka` не становился «проектом»).
**Закрыто (M1-10):** файлы корня добавляются к скану, а всё без `project`-блока
(включая `vince.masuka`) пропускается с WARN. Проверено: `rootproj.masuka` рядом с
`vince.masuka` попадает в `build.sh`.

---

## 5. Канонический формат файлов (как в проекте — его и фиксируем)

### 5.1. `vince.masuka` (единственный в корне workspace)

```masuka
workspace <name>
{
    let <var>: "<value>",          // кастомные переменные, подставляются как {var}
    entry: "<project-name>",       // проект по умолчанию для -r
    builds: [
        { name: "debug", preprocessor: "SC_BUILD_DEBUG", optimize: 0 },
        ...
    ],
    platforms: [
        { name: "linux", preprocessor: "SC_PLATFORM_LINUX" },
        ...
    ]
}
```

### 5.2. `*.masuka` проекта (по одному проекту на файл)

```masuka
project <Name>
{
    type: "Executable" | "StaticLib" | "DynamicLib",   // по первому символу
    bin:      "<dir>",     // относительно корня workspace
    objdir:   "<dir>",     // относительно корня workspace
    source:   [ "<glob или файл>", ... ],   // относительно каталога проекта
    dependencies: [ "<ProjectName>", ... ], // ТОЛЬКО проекты workspace
    links:    [ "<lib/флаг>", ... ],        // сырые флаги линковщика (vulkan, xcb, ...)
    include:  [ "<dir>", ... ],             // относительно корня workspace
    provide-include: [ "<dir>", ... ],      // относительно каталога проекта, экспорт зависимым
    linux:   { links: [], preprocessors: [], include: [] },
    windows: { links: [], preprocessors: [], include: [] }
}
```

**Правила, которые сейчас неявны и требуют фиксации (см. §7, вопросы):**

1. `source`, `provide-include` — от каталога проекта; `include`, `bin`, `objdir` — от
   корня workspace.
2. `dependencies` — имена проектов, а не файлы/библиотеки; `links` — внешние
   библиотеки и флаги (в текущих `*.masuka` пусто, но Premake-конфиги требуют
   `vulkan`, `xcb`, `Xrandr`, `m`, `pthread`, `dl`).
3. Платформенный блок мержится поверх проекта: `links`/`include` дополняются,
   `preprocessors` — объединяются.
4. `{build}`/`{platform}` = **имена выбранных build/platform из `vince.masuka`**
   (`debug`/`linux`), а не системные строки.

### 5.3. Что в каноническом формате отсутствует, но нужно

- `preprocessors: [...]` на уровне проекта (для `SC_MEMORY_DEBUG` и т.п.;
  сейчас только внутри платформенных блоков) — и парсер это уже почти умеет.
- явное поле `entry` для тестовых проектов (если `-t` остаётся).
- версия формата (опционально, `version: 1`), чтобы ловить старые примеры.

`MiracleEngine/Source/Tools/vince.masuka_example` описывает **другой** формат
(`project`-блоки внутри `workspace`, workspace-level `dependencies`, `source:` с
`{project}`). Его нужно либо удалить, либо переписать под канон — иначе он будет
провоцировать реализацию второго формата.

---

## 6. План работ

Порядок соответствует приоритету ТЗ: сначала «сборка вообще работает и соответствует
ТЗ», потом флаги/проверки, потом UX и миграция. Оценки — в человеко-днях, без учёта
ревью.

**Статус на текущей итерации:** Этап 0 закрыт по MVP (CPU count, удаление и смена
каталога); `sc_process_run` снят — процессы запускаются из сгенерированных скриптов,
нормализация путей и тесты `Sc` — в Backlog. Этап 1 закрыт полностью (M1-1…M1-10):
единый конвертер `MskFile → MskProject`, валидация, граф, топосорт, скан корня.
Этап 2 закрыт полностью (M2-1…M2-8): все флаги и проверки, справка `-h/--help`,
встроенные самотесты `--self-test`. Этап 3 закрыт полностью (M3-1…M3-6): валидные
`build.sh` + `build_{Project}.sh/.bat`, `compile_commands.json`, экранирование путей.
Этап 4 закрыт полностью (M4-1…M4-6): `masuka` без флагов собирает репозиторий
(`system("bash build.sh")`), `-j` параллелит проекты по уровням и TU внутри проекта,
`-c`/`-rb` работают, инкрементальность по mtime + штампу флагов.
Дальше — Этап 5 (runner: `-r`, `-e`, `-a`).
Подробности — §3.2–3.10.

> Рабочий чек-лист с галочками (актуальные статусы задач): [plan.md](plan.md).
> Ниже — исходная формулировка этапов с оценками и критериями приёмки.

### Этап 0. Инфраструктура `Sc` (0.5–1 д.)

Основание: без этих примитивов runner/clean/`-j` придётся делать через `system()`.
По решению итерации запуск процессов — **только** из сгенерированных `.sh`/`.bat`
через `system()`/`cmd.exe`, поэтому отдельный `sc_process_run` не делаем.

| ID | Задача | Файлы | Готово, когда |
|---|---|---|---|
| M0-1 | `sc_system_cpu_count()` (Linux `sysconf(_SC_NPROCESSORS_ONLN)`, Windows `GetSystemInfo`) | `Sc/Types.c/.h` | Возвращает ≥1 на обеих платформах; используется в валидации `-j` |
| ~~M0-2~~ | ~~`sc_process_run(ScProcessConfig)` — argv, cwd, env, capture exit code (fork+execvp / CreateProcess); без shell~~ — **снят**: процесс запускается из `.sh`/`.bat` | — | — |
| M0-3 | `sc_directory_delete_recursive(const char*)` | `Sc/Sc.c/.h` | Удаляет дерево `bin/debug-linux` |
| M0-4 | `msk_cli` — `chdir` в `-d`/каталог exe (`sc_directory_set_current`) | `Sc/Sc.c/.h` | Относительные пути считаются от рабочего каталога |
| M0-5 | `sc_path_is_absolute`, `sc_path_normalize` — **Backlog** (после MVP) | `Sc/Sc.c/.h` | Единообразные пути в скриптах/json |

Актуальные статусы и Backlog (M0-5, тесты `Sc`) — в [plan.md](plan.md).

### Этап 1. Модель и язык: `MskFile` → `MskProject` (2–3 д.)

Цель: из `*.masuka` получать полностью разрешённые проекты (пути, глобы, платформа,
build, зависимости), пригодные и для скриптов, и для json, и для компиляции.

| ID | Задача | Файлы | Готово, когда |
|---|---|---|---|
| M1-1 | Расширить `MskCliConfig`: `IsClean`, `IsRebuild`, `IsCompile`, `IsGenerateBuildFiles`, `IsGenerateJson`, `IsExecute`, `IsRunTests`, `pEntryName`, `aArguments`, `ArgumentsCount`, `pBuild`, `pPlatform`, `pDirectory`, `JobsCount` | `Masuka.h:10` | Структура покрывает все флаги ТЗ |
| M1-2 | Per-project переменные: убрать глобальный `project`; подставлять `{project}` при препроцессинге конкретного файла (set/restore или передача дополнительной переменной) | `Masuka.c:935`, `:982` | Два проекта с одинаковым `objdir` получают разные пути |
| M1-3 | Исправить `msk_preprocessor_file`: писать по месту (`mskFile.aXxx[i].pName`), пропускать NULL, не терять строки | `Masuka.c:982–1077` | В скриптах нет `{...}`; `-I{корень}/include` подставлен |
| M1-4 | `{build}`/`{platform}` = имена выбранных build/platform; системные значения — только как fallback | `Masuka.c:288` | `bin/{build}-{platform}` → `bin/debug-linux` при `-b debug -p linux` |
| M1-5 | `msk_file_platform_parse`: выставлять `pName`, хранить в `MskFile.aPlatforms`; мержить нужный блок в проект (links/include/preprocessors) | `Masuka.c:624`, `:790` | Блок `linux:` влияет на сборку; нет WARN `pName is null` |
| M1-6 | `msk_project_from_file(MskFile, MskVince, build, platform)` — единственный конвертер AST → `MskProject`: глобы (`msk_project_sources_resolve`), `-D` build/platform, `provide-include` от каталога проекта | `Masuka.c` (новые функции), `Masuka.h` | `MskProject` заполнен полностью; генератор и `-c` работают на нём |
| M1-7 | Валидация `MskFile`/`MskVince` с внятными ошибками (файл, поле) | `Masuka.c:971`, `Masuka.h` | Битый `.masuka` даёт `ERROR ... file:line` и ненулевой код возврата |
| M1-8 | Граф зависимостей: резолв по имени, проверка существования, топосорт, детект циклов, транзитивные `provide-include` | новые функции в `Masuka.c` | Цикл → понятная ошибка; порядок сборки корректен |
| M1-9 | Удалить мёртвый дубль парсера (`msk_parse_root` и т.д.) после переключения | `Masuka.c` | Минус ~1300 строк дублей, одна модель данных |

**Приёмка этапа:** `masuka -nc -gb -b debug -p linux` даёт `build_*.sh` с корректными
абсолютными/корневыми путями, развёрнутыми глобами, `-DSC_BUILD_DEBUG -DSC_PLATFORM_LINUX`,
`-I` из `include` и `provide-include` зависимостей.

### Этап 2. CLI и проверки по ТЗ (1–2 д.)

| ID | Задача | Файлы | Готово, когда |
|---|---|---|---|
| M2-1 | Полный набор флагов: `-b -c -gb -rb -gccj -d -p -j -nc -r -e -a -a{N}` (+ `-t`, `-h/--help`) | `Masuka.c:201` | Каждый флаг ТЗ распознаётся и меняет поведение |
| M2-2 | Семантика: `-r` — запуск, `-e` — выбор entry, `-rb` — clean+generate+build, `-c` — clean, `-gb` — только генерация | `Masuka.c:21` | Поведение совпадает с ТЗ |
| M2-3 | Парсер значений: `-b/-p/-e/-d/-j/-a` требуют значения; поддержка `-a{N}`; значения, начинающиеся с `-`, не считаются флагами | `Masuka.c:201` | `-a "-x"` работает |
| M2-4 | Проверки: `-d` каталог; `-p`/`-b` есть в vince; `-j` ∈ `(0, 2*cpu]`; `-e` проект по `sc_string_compare(..., pName)`; `-a` < 1024; `-a{N}` число > 0 | `Masuka.c` (новая `msk_cli_validate`) | Каждая ошибка ТЗ воспроизводится тестом и даёт сообщение + код != 0 |
| M2-5 | Единые коды возврата: 0 — ок, 1 — ошибка сборки, 2 — ошибка конфигурации/валидации; `main` возвращает код `msk_cli_start` | `MasukaTest.c` | `echo $?` соответствует типу ошибки |
| M2-6 | Usage-вывод по `-h`/без аргументов | `Masuka.c` | Справка перечисляет флаги ТЗ |

### Этап 3. Генераторы: скрипты и `compile_commands.json` (2 д.)

| ID | Задача | Файлы | Готово, когда |
|---|---|---|---|
| M3-1 | `msk_generator_build_create_single` на `MskProject`: объекты по явному списку, флаги `-std=c99 -m64 -g/-O0/-O2`, каталоги создаются | `Masuka.c` | Сгенерированная команда совпадает по смыслу с Premake |
| M3-2 | `build_{Project}.sh` в каталоге проекта: `set -e`, `mkdir -p`, `rm -f lib`, `ar` по явным объектам, `-L`/`-l`/`links`, rpath | `Masuka.c:2594` | Скрипт собирает проект с нуля в чистом дереве |
| M3-3 | Главный `build.sh` в корне/`-d`: топологический порядок, вызов подскриптов, `-j` fan-out внутри проекта (`cmd & ... wait`) | `Masuka.c:2710` | `stb/qoi/MiniAudio → MiracleEngine → masuka` в правильном порядке |
| M3-4 | Режимы: без `-gb` — генерация в память + выполнение; `-gb` — только запись; `-rb` — clean + запись + сборка | `Masuka.c:21` | Ни один режим не противоречит ТЗ |
| M3-5 | `msk_generator_json_create`: `[{directory, file, output, arguments[]}]`, абсолютные пути, тот же набор флагов; писать `compile_commands.json` в корень; флаг `-gccj` | `Masuka.c:1121` | `clangd`/`bear`-совместимо, все 36+ TU видны |
| M3-6 | Экранирование путей с пробелами; корректный `.bat` (`@echo off`, `mkdir`, `del`, `/`) | `Masuka.c:2599` | Проект в каталоге с пробелом собирается |

**Приёмка:** `make clean`-эквивалент → `masuka -rb -b debug -p linux -r` собирает и
запускает `masuka`; `masuka -gccj` перезаписывает `compile_commands.json`, LSP работает.

### Этап 4. Компиляция, линковка, `clean` (2–3 д.)

| ID | Задача | Файлы | Готово, когда |
|---|---|---|---|
| M4-1 | `msk_compiler_compile` / `msk_compiler_compile_from_scripts` — запуск сгенерированных `.sh`/`.bat` через `system()`, остановка на первой ошибке, лог | `Masuka.c:1923` | Ошибка clang → код 1 и сообщение с файлом |
| M4-2 | `-j`: пул потоков (`sc_thread`/`sc_job`) или fan-out в скриптах; межпроектная параллельность по графу | `Masuka.c`, `Sc` | `-j 8` реально ускоряет сборку; `-j 1` — последовательно |
| M4-3 | Линковка: объекты списком, `-L`, `-l`, `links` как сырые флаги, порядок библиотек, `pthread/dl/m`, rpath | `Masuka.c:2842`, `:2649` | `masuka` линкуется без ручных флагов |
| M4-4 | `StaticLib` (`ar` по объектам, `rm -f`), `DynamicLib` (`-shared -fPIC`, `.so`/`.dll`) | `Masuka.c` | `libMiracleEngine.a` пересобирается без stale-объектов |
| M4-5 | `clean` (`-c`): `bin/<build>-<platform>` + objdir + сгенерированные скрипты; `-rb` вызывает clean | `Masuka.c` | После `-c` дерево чистое |
| M4-6 | (Опционально) инкрементальность: пропуск TU по mtime + хэш флагов | `Masuka.c` | Повторная сборка — no-op за секунды |

### Этап 5. Runner и `-a` (1–2 д.)

| ID | Задача | Файлы | Готово, когда |
|---|---|---|---|
| M5-1 | `msk_runner_run_program`: entry из `vince.entry` или `-e`; путь `bin/{build}-{platform}/{project}[.exe]`; cwd = `-d`/корень; код возврата процесса наружу | `Masuka.c:1262` | `-r` запускает `masuka`/выбранный проект |
| M5-2 | Парсер `-a`: формат `arg0="0" arg1='1' arg2=2`, пробелы, `"..."`, `'...'`, экранирование; лимит <1024 байт; `-a{N}` — явная длина; проверки ТЗ | `Masuka.c` (новый модуль) | Все примеры из ТЗ дают ожидаемый argv (юнит-тест) |
| M5-3 | `msk_runner_run_tests` для `-t` (вне ТЗ): прогон проектов-тестов | `Masuka.c:1268` | `-t` возвращает агрегированный код |
| M5-4 | Passthrough stdout/stderr — автоматически при запуске через `system()` | `Masuka.c` | Логи движка видны как есть |

### Этап 6. Windows (2–3 д.)

| ID | Задача | Готово, когда |
|---|---|---|
| M6-1 | Генерация `.bat`: shebang `@echo off`, `mkdir`, `del /q`, `call`, `%ERRORLEVEL%` | `-gb` на Windows даёт рабочий набор скриптов |
| M6-2 | Тулчейн: `clang`, `llvm-ar`, `.exe/.lib/.dll`, `-shared`, разделители путей | Сборка x64 clang на Windows |
| M6-3 | `.bat` через `system()`/`cmd.exe`, `sc_system_cpu_count` | Runner/`-j` работают |
| M6-4 | `-p windows` — только target (кросс-сборка не входит в ТЗ) | Документировано |

### Этап 7. Миграция с Premake и документация (2–3 д.)

| ID | Задача | Готово, когда |
|---|---|---|
| M7-1 | `*.masuka` для всех Samples (`ScSim`, `ScVkSample`, `ScUiSample`, `SetupSample`, `ScAutoTestSample`, `ScFormating`, `ScParsersSample`, `ScVkMemory`, `ScWindowSample`, `Ted`, `MrqlPhyz`) по данным `*-premake5.lua` | `masuka -rb -r -e <sample>` собирает каждый |
| M7-2 | Исправить битые пути: `Samples/Masuka/masuka.masuka` (`Projects/Masuka/Source/` → `Samples/Masuka/Source`); привести/удалить `Tools/*.masuka_example` | Нет «мусорных» include |
| M7-3 | Удалить Premake (`Makefile`, `*-premake5.lua`, `Scripts/Linux/*`, `Scripts/Templates/*`), обновить `build.sh`, `.gitignore`, `.clangd`, `AGENTS.md`, `README.md` | Единственная сборка — Masuka |
| M7-4 | Тесты в `ScAutoTestSample`: vince-парсер, препроцессор (в т.ч. массивы), глобы, CLI-валидация, топосорт, `-a`-парсер, smoke-сборка | `masuka -t` зелёный |
| M7-5 | Документация: `Documentation/Masuka/README.md` — язык, поля, правила путей, CLI, коды возврата | Новый участник собирает проект по доке |

### Сводка по этапам

| Этап | Содержание | Дней | Закрывает пункты ТЗ |
|---|---|---:|---|
| 0 | `Sc`: CPU, delete-dir, chdir (`sc_process_run` не нужен) | 0.5–1 | основа для `-j/-r/-c` |
| 1 | модель/язык/глобы/платформа/граф | 2–3 | формат, сборка |
| 2 | CLI + все проверки | 1–2 | все флаги и проверки |
| 3 | скрипты + `compile_commands.json` | 2 | `-gb/-gccj`, «через скрипты» |
| 4 | компиляция/линковка/clean/`-j` | 2–3 | сборка, `-c`, `-rb`, `-j` |
| 5 | runner + `-a` | 1–2 | `-r`, `-e`, `-a`, `-a{}` |
| 6 | Windows | 2–3 | `.bat`, платформа |
| 7 | миграция, тесты, документация | 2–3 | завершение миграции |
| **Итого** | | **~13–19** | |

### Milestones

- **M-A (этапы 0–1):** сгенерированные скрипты валидны; проект собирается руками их запуском.
- **M-B (этапы 2–4):** `masuka -rb -b debug -p linux` собирает всё; `compile_commands.json` от Masuka; `-c` работает.
- **M-C (этап 5):** `masuka -r -a '...'` и `-e` работают как в ТЗ.
- **M-D (этапы 6–7):** Windows + все sample'ы на Masuka, Premake удалён.

---

## 7. Решения, которые нужно подтвердить до реализации

**Уже принятые по умолчанию (исправляются, если не согласен):**

- `{build}`/`{platform}` = имена из `vince.masuka` (`debug`/`linux`) → `bin/debug-linux`.
- `-d` = рабочий каталог: `masuka` делает `chdir` в него, все относительные пути
  (`bin`, `objdir`, `source`, `include`) считаются от него.
- `-c` чистит только текущие build/platform (`bin/<b>-<p>`, objdir проектов,
  сгенерированные `build*.sh`) — не весь `bin`.
- `-e` подразумевает запуск (`IsExecute = 1`), как написано в ТЗ: «`-e "test"`
  команда запустит проект тестов».
- Явно переданное невалидное значение (`-p foo`, `-b relase`, `-j abc`) — ошибка
  (требование ТЗ), а не тихий откат на системное значение.
- Запуск процессов — только из сгенерированных `.sh`/`.bat` (`system()`/`cmd.exe`):
  компиляция, линковка и runner не вводят отдельного API, `sc_process_run` (M0-2) снят.

Ниже — решения, которые всё ещё открыты:

1. **База путей** — **решено и реализовано (M1-6/M3):** `source`/`provide-include`
   от каталога проекта; `include`/`bin`/`objdir` от корня workspace; в скриптах пути
   относительны корня, в json — абсолютные.
2. **Объектные файлы** — **решено и реализовано (M3-1):** путь источника
   зеркалируется внутри `objdir` (`objdir/Source/Sc/Sc.o`) — коллизий имён нет.
3. **`links` vs `dependencies`** — **решено и реализовано (M3-2):** `dependencies` —
   только проекты workspace (дают `-l`), `links` — системные библиотеки/сырые флаги
   (`-pthread` как флаг, `vulkan` как `-lvulkan`).
4. **`-t`** (тесты) — оставить как расширение вне ТЗ или убрать?
5. **Premake** — держать до milestone M-B как страховку, удалять на M-D (этап 7).
6. **`vince.masuka_example`** — удалить или переписать под канон (сейчас описывает
   второй, несовместимый формат)?
7. **Распил `Masuka.c`** (3500 строк) на `MasukaCli.c` / `MasukaLanguage.c` /
   `MasukaGenerator.c` / `MasukaCompiler.c` / `MasukaRunner.c` — делать сразу или
   после M-B? По конвенции проекта модуль = `.c/.h` пара, так что распил уместен,
   но он конфликтует с параллельной работой над логикой.
8. **Сборка без `-gb`** — **решено (M4-1):** скрипты пишутся во временный каталог
   `bin/.masuka-tmp` (внутри gitignore-каталога `bin/`), выполняются и удаляются;
   каталог же чистит `-c`.
9. **`cd` в сгенерированных скриптах** — **решено (M3-2):** в начале каждого скрипта
   `cd` в абсолютный корень workspace, чтобы его можно было запустить из любого
   каталога. Артефакт регенерируется `masuka -gb`, поэтому машинно-зависимый путь
   допустим (в репозитории сгенерированные скрипты не считаются переносимыми).

---

## 8. Конвенции, которых нужно держаться

- C99, без C11/C17; один компилятор — Clang.
- `.clang-format` в корне: тип возврата на отдельной строке, `BreakBeforeBraces: Allman`
  для функций, `PointerAlignment: Left`, `ColumnLimit: 0`, отступ 4 пробела.
- Именование: типы `MskPascalCase`/`ScPascalCase`, функции `msk_xxx_yyy`/`sc_xxx_yyy`,
  поля структур `pName`, `aItems`, `Count`, `Index`, `IsFlag`; enum-значения
  `MskProjectType_StaticLib`.
- Ошибки — `ScResult` + `sc_log_erro`/`sc_log_warn`, без `assert` в рантайме сборки.
- Строки — `sc_builder_*` + `sc_string_*`; списки — `sc_list_*`
  (`sc_list_add`, `sc_list_count`, `sc_list_for_each`); файлы — `sc_file_*`;
  каталоги — `sc_directory_*`; процессы — сгенерированные `.sh`/`.bat` и
  `system()` (без отдельного процессного API).
- Каждый модуль — пара `.c/.h` в `MiracleEngine/Source/Tools` или `Sc`, новые
  `*.c` автоматически подхватываются глобами `Source/**.c`.
- Не трогать `Dependencies/`; не менять `compile_commands.json` руками.
