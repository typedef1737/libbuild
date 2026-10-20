# Masuka — план работ (чек-лист)

Источник требований: ТЗ Masuka (`masuka.txt`) и [аудит реализации](audit-and-plan.md).
Формат файлов `vince.masuka` / `*.masuka` — канонический, как в проекте.
Отмечать пункты: `- [x]` — готово, `- [ ]` — не готово.

**Текущая итерация:** Этапы 0–4 закрыты (M1-1…M1-10, M2-1…M2-8, M3-1…M3-6, M4-1…M4-6),
дальше Этап 5 — runner (`-r`/`-e`/`-a`).
Этап 0 сокращён до MVP: M0-1/M0-3/M0-4 закрыты, M0-2 снят, M0-5/M0-6 — в Backlog.

Легенда приоритетов: **P0** — блокирует сборку, **P1** — требование ТЗ,
**P2** — удобство/расширение, **P3** — миграция/чистка.

---

## Milestones

- [x] **M-A (Этапы 0–1):** сгенерированные скрипты валидны, проект собирается их запуском.
- [x] **M-B (Этапы 2–4):** `masuka -rb -b debug -p linux` собирает всё; `compile_commands.json` от Masuka; `-c` работает.
- [ ] **M-C (Этап 5):** `masuka -r -a '...'` и `-e` работают как в ТЗ.
- [ ] **M-D (Этапы 6–7):** Windows + все sample'ы на Masuka, Premake удалён.

---

## Этап 0 — инфраструктура `Sc`

- [x] **M0-1 (P1)** `sc_system_cpu_count()` — количество CPU для проверки `-j`.
  - [x] Linux: `sysconf(_SC_NPROCESSORS_ONLN)`, Windows: `GetSystemInfo`.
  - [x] Всегда `>= 1`.
- [x] **M0-3 (P1)** `sc_directory_delete_recursive()` — удаление дерева для `-c`.
- [x] **M0-4 (P1)** `sc_directory_set_current()` — `chdir` для `-d`/дефолтного каталога.

> **M0-2 снят:** процессы (компиляция, линковка, runner) запускаются из
> сгенерированных `.sh`/`.bat` через `system()`/`cmd.exe`. Отдельный
> `sc_process_run` не делаем — он дублировал бы шелл и требовал своего
> экранирования argv.

Отложено в [Backlog](#backlog-после-mvp): M0-5 (нормализация путей), M0-6 (тесты `Sc`).

---

## Этап 1 — модель и язык (`MskFile` → `MskProject`)

- [x] **M1-1 (P1)** Расширить `MskCliConfig` под все флаги ТЗ (сделано на этапе CLI).
- [x] **M1-2 (P0)** `{project}` — переменная проекта, а не глобальная.
  - [x] Убрать добавление `project` в общий `MskVince.aVariables` из `msk_language_file_new`.
  - [x] Подставлять имя проекта при препроцессинге конкретного файла (set/restore).
  - [x] Критерий: у двух проектов разные `{project}` в `objdir` одновременно.
- [x] **M1-3 (P0)** Починить подстановку переменных в массивах.
  - [x] Писать в `mskFile.aXxx[i].pName`, а не в локальную копию `MskString`.
  - [x] Не терять строки, не течь, не возвращать NULL для пустых полей.
  - [x] Критерий: в `build.sh` нет `{vulkan-path}` и других `{...}`.
- [x] **M1-4 (P0)** `{build}`/`{platform}` из выбранных значений.
  - [x] Частично: `msk_language_vince_set_selection()` подставляет имена из `vince.masuka`.
  - [x] Убрать зависимость от системных `Debug`/`Linux` в `msk_language_vince_new`
    (дефолт — `aBuilds[0]`/`aPlatforms[0]`, система — только фолбэк).
- [x] **M1-5 (P1)** Платформенные блоки `linux:` / `windows:`.
  - [x] Частично: `pName` выставляется при парсинге (release больше не падает).
  - [x] Мерж `links`/`include`/`preprocessors` выбранной платформы в проект
    (`msk_file_apply_platform()`).
  - [x] `preprocessors:` на уровне проекта.
- [x] **M1-6 (P0)** `msk_project_from_file()` — единственный конвертер AST → `MskProject`.
  - [x] Разворот source-глобов относительно каталога проекта (`msk_project_sources_resolve`).
  - [x] `provide-include` относительно каталога проекта (`MskProject.aProvided`).
  - [x] Добавление `-D` build/platform preprocessors каждому проекту.
  - [x] Проверка обязательных полей (`type`, `bin`, `objdir`, `source`) — в M1-7.
  - [x] Генератор и `-c` переведены на `MskProject` (`msk_generator_*`, `msk_compiler_clean`).
- [x] **M1-7 (P1)** Валидация `MskFile`/`MskVince` с внятными ошибками (файл, поле).
  - [x] `msk_language_file_validate()`: путь файла + имя поля в сообщении.
  - [x] `msk_language_vince_validate()`: builds/platforms/дубликаты имён.
  - [x] Код возврата `2` при битом `.masuka`/`vince.masuka`.
- [x] **M1-8 (P1)** Граф зависимостей (`msk_projects_resolve`).
  - [x] Резолв по имени, проверка существования и дубликатов, самозависимость.
  - [x] Топологическая сортировка (deps-first), детект циклов с цепочкой в ошибке.
  - [x] Транзитивные `provide-include` в includes зависимых проектов (с дедупликацией).
- [x] **M1-9 (P2)** Удалён мёртвый дубль парсера (`msk_parse_root`, `msk_parse_msk_file*`,
  `msk_build_*`, `msk_project_compile`, batch-обвязка): −1300 строк, одна модель данных.
- [x] **M1-10 (P1)** `*.masuka` в корне workspace находятся наравне с подкаталогами.
  - [x] Добавляются файлы корня; файлы без `project`-блока пропускаются с WARN.
  - [x] Критерий: workspace с проектом рядом с `vince.masuka` собирается (проверено).

---

## Этап 2 — CLI и проверки по ТЗ

- [x] **M2-1 (P1)** Все флаги: `-b -c -gb -rb -gccj -d -p -j -nc -r -e -a -a{N}` (+ `-t`).
- [x] **M2-2 (P1)** Семантика: `-r` — запуск, `-e` — entry, `-c` — clean, `-rb` — clean+generate+build, `-gb` — только генерация.
- [x] **M2-3 (P1)** Парсер значений: значения не трактуются как флаги, поддержка `-a{N}`.
- [x] **M2-4 (P1)** Проверки:
  - [x] `-d` — существующий каталог.
  - [x] `-p` — есть в `vince.masuka` (platforms).
  - [x] `-b` — есть в `vince.masuka` (builds).
  - [x] `-j` — число, `0 < j <= 2 * cpu_count`.
  - [x] `-e` — проект существует (`pProjectName`).
  - [x] `-a` — длина `< 1024` байт.
  - [x] `-a{N}` — число `> 0`.
  - [x] Неизвестный флаг / флаг без значения — ошибка.
- [x] **M2-5 (P1)** Коды возврата: `0` — успех, `1` — сборка/запуск, `2` — конфигурация.
- [x] **M2-6 (P1)** Опциональность: дефолты из `sc_system_info_get()` + `argv[0]`
  (`build`/`platform`/`jobs=1`/каталог exe с подъёмом до `vince.masuka`).
- [x] **M2-7 (P2)** `-h` / `--help` со списком флагов; usage при ошибке валидации.
  - [x] Полный список флагов, дефолты (`-j` от CPU), коды возврата (`msk_cli_print_help`).
  - [x] `-h`/`--help` — ранний выход до инициализации лога и чтения workspace, код `0`.
  - [x] Подсказка `Run 'masuka -h' for usage.` при ошибках валидации CLI.
- [x] **M2-8 (P2)** Юнит-тесты Masuka — прямо в `Masuka.c` (`msk_tests_run`, `--self-test`).
  - [x] CLI: парсинг всех флагов, `-a{N}`, неизвестный флаг, значение-флаг, `-h`.
  - [x] Валидация: `-a`/`-a{N}`, `-j`, отсутствующие значения.
  - [x] Язык: `vince.masuka`, парсер `*.masuka` из памяти, блоки платформ, имена с цифрами.
  - [x] Препроцессинг: `{var}`, пустой/незакрытый `{`, строка без переменных.
  - [x] Граф: топосорт, транзитивные includes, неизвестная зависимость, цикл.
  - [x] Запуск: `masuka --self-test` → код `0`/`1`, без создания файлов вне `.tmp-masuka-tests`.

---

## Этап 3 — генераторы: скрипты и `compile_commands.json`

- [x] **M3-1 (P0)** `msk_generator_build_create_single` на `MskProject`: флаги и объекты по списку.
  - [x] Флаги: `-std=c99 -m64`, `-g -O0` (debug) / `-O2` (`optimize: 1`), `-D` проекта+build+platform,
    `-I` свои + транзитивные `provide-include` зависимостей, `-fPIC` для `DynamicLib`.
  - [x] Объекты по явному списку; путь объекта зеркалирует источник (`objdir/Source/Sc/Sc.o`), коллизий имён нет.
  - [x] Линковка: `-L` + `-l` по собственным и транзитивным зависимостям, `links` — флаги (`-pthread`) или имена (`vulkan` → `-lvulkan`).
- [x] **M3-2 (P0)** `build_{Project}.sh` в каталоге проекта.
  - [x] `set -e`, `cd` в абсолютный корень workspace (скрипт не зависит от cwd), `mkdir -p` каталогов объектов и `bin`.
  - [x] `rm -f` перед `ar`, `ar rcs` по явному списку объектов; `-shared` + `.so`, rpath `$ORIGIN` при `.so`-зависимостях.
- [x] **M3-3 (P0)** Главный `build.sh` в корне workspace.
  - [x] Топологический порядок проектов, вызовы `bash <проект>/build_<проект>.sh`, `echo`-прогресс.
  - [x] `-j` fan-out внутри проекта (`pids=()` + `cmd &` + `wait "$pid"`), `-j 1` — последовательно.
- [x] **M3-4 (P1)** Режимы: без `-gb` — генерация в память (запуск — M4-1); `-gb` — только запись скриптов;
  `-rb` — clean + запись + сборка.
- [x] **M3-5 (P1)** `compile_commands.json` (`-gccj`).
  - [x] Формат `[{directory, file, output, arguments[]}]`, абсолютные пути, тот же набор флагов; пишется в корень workspace.
  - [x] Проверено: 44 TU на текущем репозитории, `arguments` из файла компилируются clang'ом, `directory` — каталог проекта.
- [x] **M3-6 (P2)** Экранирование путей с пробелами (`'a b'` для sh, `"a b"` для cmd) и `.bat`-синтаксис:
  `@echo off`, `cd /d`, `if not exist ... mkdir`, `del /q`, `call`, `if errorlevel 1 exit /b 1`, `llvm-ar`, `.exe/.lib/.dll`,
  `\` в путях. `.bat` генерируется только на Windows-хосте (M6), текст проверяется тестами.

---

## Этап 4 — компиляция, линковка, clean

- [x] **M4-1 (P0)** Реальный `msk_compiler_compile` / `_from_scripts` — запуск сгенерированных `.sh`/`.bat` через `system()`.
  - [x] `-gb`: запуск `<root>/build.sh` (`.bat` — через `cmd /c`), вывод компилятора идёт в терминал.
  - [x] Без `-gb`: скрипты пишутся в `bin/.masuka-tmp`, выполняются и удаляются (даже при ошибке); `-c` чистит каталог.
  - [x] Остановка на первой ошибке: `system()` != 0 → `Command failed (status N)`, `Build failed!`, код `1`.
- [x] **M4-2 (P1)** `-j`: параллельная компиляция объектов и проектов по графу.
  - [x] `msk_projects_levels` — уровни зависимостей; проекты одного уровня независимы.
  - [x] Главный скрипт запускает волну параллельно (`pids=()`/`&`/`wait`), `-j` делится между проектами волны.
  - [x] Внутри проекта — fan-out по TU. Проверено: `-j 4` — 1.9 c, `-j 1` — 3.5 c на текущем репозитории.
- [x] **M4-3 (P1)** Линковка: объекты списком, `-L`, `-l`, `links` как сырые флаги, порядок библиотек
  (команды из M3-2, теперь реально исполняются).
- [x] **M4-4 (P1)** `StaticLib` (`ar` по списку, `rm -f` до сборки) и `DynamicLib` (`-shared -fPIC`, `.so`/`.dll`).
- [x] **M4-5 (P2)** Инкрементальность: пропуск TU по mtime + флаги.
  - [x] В скрипте: `if [ ! -f obj ] || [ src -nt obj ] || [ "$MSK_FLAGS" != "$(cat "$MSK_STAMP")" ]; then ... fi`.
  - [x] Штамп флагов `objdir/.masuka_flags`: смена флагов пересобирает все TU проекта.
  - [x] Заголовки не отслеживаются (нет depfile), `.bat` всегда пересобирает — известные ограничения (M6).
- [x] **M4-6 (P1)** `-c`: чистка `bin/<build>-<platform>`, objdir, сгенерированных скриптов и `bin/.masuka-tmp`.

---

## Этап 5 — runner и аргументы запуска

- [ ] **M5-1 (P1)** `msk_runner_run_program`.
  - [ ] Entry: `-e` или `entry` из `vince.masuka`.
  - [ ] Путь: `bin/{build}-{platform}/{project}[.exe]`.
  - [ ] Запуск через `system()` с цитированным argv из `-a` (без `sc_process_run`).
  - [ ] cwd = рабочая директория, код возврата процесса — наружу.
- [ ] **M5-2 (P1)** Парсер `-a`: `arg0="0" arg1='1' arg2=2`.
  - [ ] Пробелы, двойные/одинарные кавычки, экранирование.
  - [ ] Лимит `< 1024` для `-a`; явная длина для `-a{N}`.
  - [ ] Юнит-тест на разбор argv.
- [ ] **M5-3 (P2)** `msk_runner_run_tests` для `-t` (вне ТЗ).
- [ ] **M5-4 (P2)** Passthrough stdout/stderr дочернего процесса.

---

## Этап 6 — Windows

- [ ] **M6-1 (P1)** Генерация `.bat`: `@echo off`, `mkdir`, `del /q`, `call`, `%ERRORLEVEL%`.
- [ ] **M6-2 (P1)** Тулчейн: `clang`, `llvm-ar`, `.exe/.lib/.dll`, `-shared`, разделители путей.
- [ ] **M6-3 (P1)** `.bat` исполняется через `system()`/`cmd.exe`; `sc_system_cpu_count`.
- [ ] **M6-4 (P2)** `-p windows` — только target (кросс-сборка вне ТЗ), документировать.

---

## Этап 7 — миграция, тесты, документация

- [ ] **M7-1 (P2)** `*.masuka` для всех Samples (`ScSim`, `ScVkSample`, `ScUiSample`,
  `SetupSample`, `ScAutoTestSample`, `ScFormating`, `ScParsersSample`, `ScVkMemory`,
  `ScWindowSample`, `Ted`, `MrqlPhyz`) по данным `*-premake5.lua`.
- [ ] **M7-2 (P2)** Починить битые пути: `Samples/Masuka/masuka.masuka`
  (`Projects/Masuka/Source/` → `Samples/Masuka/Source`).
- [ ] **M7-3 (P3)** Удалить Premake (`Makefile`, `*-premake5.lua`, `Scripts/Linux/*`),
  обновить `build.sh`, `.gitignore`, `.clangd`, `AGENTS.md`, `README.md`.
- [ ] **M7-4 (P2)** Тесты Masuka (встроенные, `masuka --self-test`).
  - [x] vince-парсер, парсер проектов, препроцессор, блоки платформ, глобы, CLI, топосорт, циклы.
  - [x] генераторы: пути объектов, `.sh`/`.bat`-скрипты, порядок проектов, `-j` fan-out, `compile_commands.json`, экранирование.
  - [ ] `-a`-парсер argv (после M5-2).
  - [ ] smoke-сборка: `-rb` + запуск результата (после M4).
- [ ] **M7-5 (P2)** `Documentation/Masuka/README.md`: язык, поля, правила путей, CLI, коды возврата.

---

## Backlog (после MVP)

Сознательно отложено: на MVP и DoD по ТЗ не влияет, вернуться после M-D.

- [ ] **M0-5 (P2)** `sc_path_get_absolute()` / нормализация путей.
  - [ ] Убрать `./../..` в логах и путях по умолчанию (сейчас работает, но некрасиво).
- [ ] **M0-6 (P2)** Тесты новых API `Sc` в `ScAutoTestSample` (вместе с тестами Masuka, M7-4).

---

## Definition of Done по ТЗ

- [x] `masuka` собирает текущий build/platform без флагов.
- [x] `masuka -b <build> -p <platform>` подставляет выбранные значения во все `{build}`/`{platform}`.
- [x] `masuka -c` удаляет артефакты текущей конфигурации (скрипты, objdir, `bin/<build>-<platform>`).
- [x] `masuka -rb` = clean + generate + build.
- [x] `masuka -gb` генерирует `build.sh` (и `.bat` на Windows), который собирает проект.
- [x] `masuka -gccj` генерирует рабочий `compile_commands.json`.
- [ ] `masuka -r` запускает entry-проект, `-e "test"` — указанный.
- [ ] `masuka -a 'arg0="0" arg1='"'"'1'"'"' arg2=2'` передаёт argv без искажений.
- [ ] `-a{N}` работает и валидирует длину.
- [x] `-j` реально включает параллельную компиляцию.
- [ ] Все проверки из ТЗ дают понятную ошибку и код `2`.
- [ ] Значения не переданы → берутся из `sc_system_info_get()` и `argv[0]`.

---

## Открытые решения

1. База путей: `source`/`provide-include` от каталога проекта; `include`/`bin`/`objdir`/`links` от корня workspace.
   Готово: в скриптах пути относительно корня workspace (скрипт сам делает `cd` в абсолютный корень), в json — абсолютные.
2. Объектные файлы: **решено (M3-1)** — зеркалируется путь источника (`objdir/Source/Sc/Sc.o`).
3. `links` — сырые флаги/системные библиотеки; `dependencies` — только проекты workspace.
4. `-t` — оставить как расширение вне ТЗ или убрать.
5. Premake — держать до M-B, удалять на M-D.
6. `vince.masuka_example` — удалить или переписать под канон.
7. Распил `Masuka.c` (3500 строк) на `MasukaCli/MasukaLanguage/MasukaGenerator/MasukaCompiler/MasukaRunner` — до или после M-B.
8. Процессы не запускаются напрямую: компиляция/линковка идут через сгенерированные `.sh`/`.bat` (`system()`), runner — через `system()` по `bin/{build}-{platform}/...` с цитированием argv. `sc_process_run` не реализуем.
