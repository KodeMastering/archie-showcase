# 13. README и Quickstart проекта Archie

**Статус:** черновик v1.0.0 · **Аудитория:** новые инженеры (день 1–2), open-source-контрибьюторы, инженеры B2B-клиентов (Phase 4) · **Связанные документы:** [12-ADR-Glossary.md](12-ADR-Glossary.md), RFC-0001, Архитектурная спецификация, [docs/README.md](README.md)

> Документ №13 комплекта. Отражается в корневом `README.md` репозитория `archie-os-symbiote/`. Правила согласованности комплекта — см. [docs/README.md](README.md); все числа ниже скопированы из RFC-0001 без изменений.

---

## Содержание

1. [Что такое Archie (hero)](#1-что-такое-archie-hero)
2. [Канонические числа RFC-0001](#2-канонические-числа-rfc-0001)
3. [Обзор архитектуры (диаграмма)](#3-обзор-архитектуры-диаграмма)
4. [Структура монорепозитория](#4-структура-монорепозитория)
5. [Предварительные требования](#5-предварительные-требования)
6. [Quickstart: от клона до работающего Archie](#6-quickstart-от-клона-до-работающего-archie)
7. [Загрузка моделей](#7-загрузка-моделей)
8. [Конфигурация](#8-конфигурация)
9. [Рабочий процесс разработки](#9-рабочий-процесс-разработки)
10. [Контрибьюция](#10-контрибьюция)
11. [Сообщество и поддержка](#11-сообщество-и-поддержка)
12. [FAQ](#12-faq)
13. [Дорожная карта](#13-дорожная-карта)
14. [Лицензия](#14-лицензия)

---

## 1. Что такое Archie (hero)

**Archie («Архи»)** — автономный проактивный ИИ-напарник, интегрированный на уровень операционной системы (OS-Level AI Symbiote). В отличие от реактивных ассистентов класса «вопрос — ответ» (Siri, базовый Copilot, Алиса), Archie сам замечает, что вы застряли, устали или зациклились на рутине, и предлагает помощь **до** того, как вы об этом попросите. Сердце системы — событийное экранное «зрение» Z-Sense и единый внутрипроцессный аудио-тракт: Archie не пишет и не стримит видео с экрана, а по триггерам ОС собирает компактный текстовый слепок контекста, локально анонимизирует его (PII Masking) и превращает в лёгкий JSON-граф. Гибридная Edge+Cloud топология делит вычисления на два контура: сверхлёгкий Rust-демон с шиной событий на вашем компьютере и тяжёлую когнитивную логику в облаке (DeepSeek / OpenAI / Google Gemini) с изящной деградацией на локальную CPU-модель 1.5B при потере сети — поэтому Archie работает на обычных студенческих ноутбуках без дискретной видеокарты, сохраняя голос и характер даже офлайн.

Ключевые дифференциаторы:

- **Проактивность с эмпатией.** Индекс фрустрации (эвристики: стагнация вкладки, циклы переписывания кода, повторные ошибки консоли, детекция вздохов Silero VAD) накапливает баллы, и при пороге 50 запускается «Внутренний Монолог» — Archie сначала показывает пульсирующий Soft Widget с окном отмены 3–5 с и лишь потом мягко заговаривает в роли «Старшего брата / Ментора».
- **Живой голос с мгновенным перебиванием.** Полнодуплексный аудио-цикл в одном процессе (WebRTC AEC3 + Silero VAD + Sherpa-ONNX ASR + Piper TTS) обеспечивает **Barge-in < 200 мс** — Archie замолкает быстрее, чем вы заканчиваете фразу «стоп, я понял».
- **Событийное зрение вместо записи экрана.** Z-Sense с локальным OCR (RapidOCR, **~200 мс на CPU**, эталон 185 мс) и Accessibility API экономит **99% сетевого трафика** и никогда не покидает машину в виде изображений.
- **Бережливость к ресурсам.** RAM-бюджет: **150 МБ (Idle) / 655 МБ (Peak) / 1450 МБ (Offline)**; инсталлятор на Tauri v2 — **~8 МБ** без весов моделей.
- **Детерминированная безопасность.** Permission Tiers 1/2/3, карантинная зона `Archie_Staging` (системный вызов Delete физически заблокирован), SOP-песочница с AST-валидацией, PII-санитайзер до любого облачного вызова, Semantic Router с совпадением триггера **>= 0.98** — утверждённые рецепты исполняются жёсткими Python-скриптами с нулевым риском галлюцинаций LLM.

Полное видение — в [Archie.docx (PAD)](../Archie.txt); продуктовые требования — в [01-PRD.md](01-PRD.md).

---

## 2. Канонические числа RFC-0001

Этот блок — точная копия бюджета из RFC-0001. **Никто не имеет права менять числа в PR без параллельного RFC-изменения** (см. [политику регрессий SLA](#политика-регрессии-sla-rfc-0001)).

```text
(источник: RFC-0001)
Barge-in (полная остановка TTS по FORCE_STOP_AUDIO)  < 200 мс
VAD: детекция речи 85–100 мс, точность > 95%         (модель 1–2 МБ, < 1 мс/чанк CPU)
OCR Z-Sense: ~200 мс на CPU                          (эталон 185 мс, PP-OCRv5 ONNX)
RAM: 150 МБ Idle / 655 МБ Peak / 1450 МБ Offline     (Tier-контур деградации)
Индекс фрустрации: порог 50                          (S_max = 100, полураспад 20/15/5/1 мин
                                                      по миссиям STUDY_FOCUS/CODING_WORK/IDLE/GAMING_REST)
Совпадение SOP-триггера Semantic Router:             >= 0.98 (косинусная близость)
Окно отмены Soft Widget:                             3–5 с
Инсталлятор:                                         ~8 МБ (без весов моделей)
Сетевой трафик (отказ от видео/скриншотов):          экономия 99%
```

---

## 3. Обзор архитектуры (диаграмма)

```text
                     ┌──────────────────────────────┐
                     │           ПОЛЬЗОВАТЕЛЬ        │
                     │  голос / клавиатура / мышь    │
                     └───────┬──────────────▲───────┘
                             │              │ Soft Widget (окно 3–5 с),
                             ▼              │ карточки Tier 2 Approval, Main Dashboard
   ┌─────────────────────────────────────────────────────────────────────┐
   │                    OS HOOKS (событийные триггеры ОС)                │
   │  смена активного окна/вкладки · выделение текста · буфер обмена ·   │
   │  ошибки консоли · циклы переписывания кода · миссии (GAMING_REST /   │
   │  STUDY_FOCUS / CODING_WORK / IDLE)                                  │
   └──────────────────────────────┬──────────────────────────────────────┘
                                  ▼
   ┌─────────────────────────────────────────────────────────────────────┐
   │        RUST DAEMON — archie-shell (Tauri v2, ~8 МБ инсталлятор)     │
   │  ┌───────────────────┐ ┌──────────────────┐ ┌────────────────────┐  │
   │  │  ARCHIE-EVENT-BUS │ │     SHELL        │ │  ARCHIE-STAGINGFS  │  │
   │  │  (tokio, брокер,  │ │  (окна, трей,    │ │  (карантин         │  │
   │  │  маршрутизация,   │ │  single-instance,│ │  Archie_Staging,   │  │
   │  │  graceful         │ │  hit-test, HiDPI)│ │  Permission Tiers, │  │
   │  │  degradation)     │ │                  │ │  Desktop Cleaner)  │  │
   │  └─────────┬─────────┘ └──────────────────┘ └────────────────────┘  │
   └────────────┼────────────────────────────────────────────────────────┘
                │  gRPC / UDS 0600 (Linux) + named pipe ACL (Windows)
                │  control-plane: event envelope (contracts/);  data-plane:
                │  PCM-фреймы через shared-memory ring buffer
                ▼
   ┌─────────────────────────────────────────────────────────────────────┐
   │        PYTHON ENGINE — services/archie-engine (Python 3.11+, uv)    │
   │  ┌────────────┐ ┌───────────────┐ ┌──────────────┐ ┌─────────────┐  │
   │  │  Z-SENSE   │ │  AUDIO LOOP   │ │   MEMORY     │ │   ROUTER    │  │
   │  │  (RapidOCR │ │  (AEC3, VAD,  │ │  (L0/L1/L2:  │ │  (Semantic  │  │
   │  │  + Access. │ │  Sherpa ASR,  │ │  RAM-сессия, │ │  Router,    │  │
   │  │  API, XY-  │ │  Piper TTS,   │ │  ночные      │ │  SOP-       │  │
   │  │  cut, PII  │ │  sigh detect, │ │  сводки,     │ │  Blueprints,│  │
   │  │  sanitizer)│ │  barge-in)    │ │  LanceDB     │ │  sandbox)   │  │
   │  └────────────┘ └───────────────┘ └──────────────┘ └─────────────┘  │
   │  ┌───────────────────────────────────────────────────────────────┐  │
   │  │  LLM CLIENTS: cloud_gateway (DeepSeek / OpenAI / Gemini) ·    │  │
   │  │  local_1_5b_fallback (ONNX/GGUF на CPU) · persona_prompts     │  │
   │  └───────────────────────────────────────────────────────────────┘  │
   └────────────┬─────────────────────────────────────────┬──────────────┘
                │ Cloud APIs (санитизированные            │ MCP hub (FastMCP:
                │ JSON-сниппеты, PII уже замаскирован)    │ stdio / SSE / Streamable HTTP)
                ▼                                         ▼
   ┌──────────────────────────┐          ┌──────────────────────────────────┐
   │  DeepSeek / OpenAI /     │          │  MCP-серверы: Obsidian · Notion · │
   │  Google Gemini           │          │  Git · Jira · Slack               │
   └──────────────────────────┘          └──────────────────────────────────┘
```

Двухконтурная логика: локальный Edge-контур (Rust + Python на машине пользователя) собирает контекст, считает индекс фрустрации, ищет в векторной памяти и исполняет SOP-скрипты; облачный когнитивный контур берёт тяжёлую генерацию кода и менторские диалоги. При `API_TIMEOUT` шина событий включает офлайн-фолбэк на локальную модель 1.5B (RAM-потолок 1450 МБ) — офлайн это спроектированное UX-состояние, а не ошибка.

---

## 4. Структура монорепозитория

Топология `archie-os-symbiote/` (строго соответствует архитектурной спецификации — расхождение README с реальным репо считается doc-багом):

```text
archie-os-symbiote/
├── apps/                    # Пользовательский интерфейс (TypeScript / Tauri v2 WebView)
│   └── desktop-ui/          #   Soft Widget, approval-карточки Tier 2, Main Dashboard, онбординг
├── crates/                  # Системное ядро на Rust
│   ├── archie-shell/        #   Жизненный цикл окон/трея, single-instance, HiDPI, Win32 hit-test
│   ├── archie-event-bus/    #   Модульная шина событий (tokio): маршрутизация, degradation
│   ├── archie-os-hooks/     #   Событийные триггеры ОС: окна, accessibility-дерево, clipboard
│   └── archie-stagingfs/    #   Карантинная FS: Archie_Staging, Permission Tiers, workspace
├── services/
│   └── archie-engine/       # Python ML-движок: Z-Sense, audio loop, memory, router, LLM, MCP
├── contracts/               # events_envelope.schema.json, ipc_bridge.proto, blueprint_sop.schema.json
├── models/                  # ONNX/GGUF-артефакты + MANIFEST (лицензии весов!)
├── storage/                 # archie_core.sqlite3, vector_store.lancedb/, Archie_Staging/
├── docs/                    # Этот комплект документации (01–13 + ADR-лог)
├── justfile                 # Команды разработки: just setup / just dev / just contracts / just test
├── Cargo.toml               # Workspace-манифест Rust
├── pyproject.toml           # Манифест Python (uv), uv.lock закоммичен
└── package.json             # pnpm-workspace для apps/desktop-ui
```

«Где менять X» — сводная таблица маршрутизации изменений:

| Что меняем | Куда | Какие тесты | ADR/RFC? |
|---|---|---|---|
| Добавить облачного провайдера | `services/archie-engine/.../llm_clients/` + `contracts/` | provider conformance suite | ADR-0006 |
| Добавить MCP-инструмент | `services/archie-engine/.../mcp_hub/servers/` | MCP contract tests | нет (см. doc 11) |
| Поведение миссий | `proactivity/mission_state_machine.py` | unit + golden-миссии | нет |
| Полураспад фрустрации | `proactivity/frustration_scorer.py` | regression на 20/15/5/1 мин | **да — RFC-0001** |
| Новый SOP-blueprint | `router_and_sops/blueprint_generator.py` | sandbox AST suite | ADR-0005 |
| Новое состояние виджета | `apps/desktop-ui/src/windows/soft_widget/` | `pnpm test` + visual | нет |
| Новый тип события шины | `contracts/events_envelope.schema.json` + паблишер/подписчики | codegen + schema-compat | **да — RFC-0001** |
| Горячий аудио-путь | `audio_loop/` (один процесс!) | barge-in latency test (< 200 мс) | **ADR-0002** |

---

## 5. Предварительные требования

### 5.1. Инструменты (все ОС)

| Инструмент | Версия | Установка | Назначение |
|---|---|---|---|
| Rust (rustup, stable) | >= 1.75 (MSRV — см. open question в doc 08) | `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs \| sh` | crates/*, Tauri-ядро |
| rustfmt + clippy | компоненты toolchain | `rustup component add rustfmt clippy` | форматирование/линт |
| uv | latest | `curl -LsSf https://astral.sh/uv/install.sh \| sh` | Python 3.11+, зависимости engine |
| Node.js + pnpm | Node 20 LTS, pnpm 9 | `corepack enable && corepack prepare pnpm@latest --activate` | apps/desktop-ui |
| Tauri CLI v2 | v2.x | `cargo install tauri-cli --version '^2'` | сборка/запуск десктоп-шелла |
| just | latest | `cargo install just` | все команды quickstart |
| cargo-nextest | latest | `cargo install cargo-nextest` | прогон Rust-тестов (CI-раннер) |

Железо: **минимум 8 ГБ RAM, рекомендуется 16 ГБ**; дискретная видеокарта **не требуется** (весь ML-инференс — CPU, ONNX Runtime / OpenVINO). Для сборки достаточно ~10 ГБ на диске (плюс ~1.4 ГБ под модели, см. §7).

### 5.2. Системные зависимости Tauri v2 по платформам

**Linux (Fedora/Ubuntu):**

```bash
# Fedora (RHEL-семейство)
sudo dnf install webkit2gtk4.1-devel openssl-devel curl-devel \
     libappindicator-gtk3-devel librsvg2-devel alsa-lib-devel \
     pulseaudio-libs-devel libudev-devel

# Ubuntu/Debian
sudo apt install libwebkit2gtk-4.1-dev build-essential curl wget file \
     libxdo-dev libssl-dev libayatana-appindicator3-dev librsvg2-dev \
     libasound2-dev libpulse-dev libudev-dev
```

**Windows:** WebView2 (устанавливается автоматически на Win 10/11; для offline-сборок — `Microsoft Edge WebView2 Runtime` из winget/MSI), Visual Studio Build Tools 2022 с workload «Desktop development with C++», Python из winget или официальный установщик.

**macOS:** Xcode Command Line Tools: `xcode-select --install`.

### 5.3. Разрешения ОС (обязательные, выдаются на первом запуске)

| ОС | Разрешение | Зачем |
|---|---|---|
| macOS | **Microphone** (TCC) | аудио-тракт, Silero VAD, вздохи |
| macOS | **Screen Recording** (TCC) | Accessibility-дерево + fallback-OCR (текстовые снапшоты, не видео) |
| macOS | **Accessibility** (TCC) | глобальные хоткеи, чтение UI-дерева |
| Windows | Микрофон (Settings → Privacy) | аудио-тракт |
| Linux (Wayland) | Portals: `xdg-desktop-portal` (screen-share/mic) | захват при событийных триггерах |

> **Важно:** без микрофона голосовой контур не стартует, но текстовый режим (Z-Sense + виджет) работает — это осознанная деградация, а не сбой.

---

## 6. Quickstart: от клона до работающего Archie

Цель: **новый инженер — от `git clone` до живого Archie в dev-режиме за 30 минут** на подготовленной машине (без загрузки локальной LLM; с ней — плюс 10–30 минут).

### 6.1. Клонирование и подготовка окружения

```bash
# 1. Клонировать репозиторий
git clone <REPO_URL> archie-os-symbiote
cd archie-os-symbiote

# 2. One-shot настройка: Rust-компоненты, uv sync, pnpm install,
#    проверка системных зависимостей Tauri (без установки пакетов — только аудит)
just setup

# 3. Сгенерировать межъязыковые контракты (proto + JSON Schema -> Rust/Python/TS)
just contracts
```

Что делает `just setup` под капотом (эквивалент вручную):

```bash
rustup component add rustfmt clippy
uv sync                        # создаёт .venv для services/archie-engine (uv.lock закоммичен)
pnpm install --frozen-lockfile # apps/desktop-ui
cargo build --workspace
```

### 6.2. Запуск в dev-режиме

```bash
# Запускает всё: Rust-демон + шину событий, archie-engine (uv), UI (Tauri dev-server)
just dev
```

Под капотом: `cargo tauri dev` поднимает shell и UI; archie-shell спавнит `services/archie-engine` через uv-окружение; UDS-сокет (Linux, права 0600) или named pipe (Windows, ACL) связывает контуры.

**Критерий успеха «первых 5 минут»:** в трее появился значок Archie, Soft Widget реагирует на пробное событие (смена активного окна), в логах шины видны `APP_OPENED`-события, а при разговоре в микрофон VAD-гейт логирует фреймы.

### 6.3. Первый запуск: онбординг Path A / Path B

При первом старте `OnboardingWizard.tsx` (в `apps/desktop-ui/src/windows/main_dashboard/`) предлагает **Opt-in Setup** — пользователь сам выбирает уровень вовлечённости:

- **Путь А — Интерактивное интервью (INTERACTIVE_INTERVIEW).** ~15-минутное голосовое интервью: какие экзамены сдаёт пользователь (EGE), на чём пишет код, что раздражает. Archie сразу строит профиль и калибрует проактивность.
- **Путь Б — Тихое наблюдение (SILENT_OBSERVATION).** Без интервью: Archie несколько дней наблюдает за окнами/вкладками с минимальной активностью и само собирает профиль; первые вмешательства деликатно отложены.

Режим хранится в `user_profile.onboarding_mode` (`archie_core.sqlite3`). Для dev-теста можно пропустить мастера: `uv run python -m archie_engine.devtools --skip-onboarding`.

### 6.4. Типичные проблемы

| Симптом | Причина | Решение |
|---|---|---|
| «Mic permission denied» на macOS | TCC-разрешение не выдано | System Settings → Privacy & Security → Microphone; перезапустить `just dev` |
| UDS-сокет не создаётся (Linux) | правах/пути `/run/user/$UID/archie.sock` | проверить `XDG_RUNTIME_DIR` и что каталог не занят другим инстансом; права должны быть 0600 |
| Engine не стартует | uv-окружение не синхронизировано | `uv sync --frozen` и проверить, что `just dev` запускает engine из `.venv` |
| OCR медленный/пустой | веса RapidOCR не скачаны | `just models-fetch` (см. §7) |
| Виджет «проваливается» под Win+D | regression Show Desktop immunity | `cargo test -p archie-shell show_desktop` — баг, если падает |
| Двойной инстанс на Windows | упал именованный мьютекс | проверить orphaned-процесс `archie-shell.exe` в Task Manager |

---

## 7. Загрузка моделей

Все веса — локальные, CPU-friendly (ONNX / GGUF), лицензии артефактов — в `models/MANIFEST` (веса имеют собственные лицензии, независимые от лицензии кода!).

```bash
# Основной набор (~300 МБ): Silero VAD (1–2 МБ), RapidOCR PP-OCRv5 (детект+распознавание),
# Piper TTS (голосовые модели), NER-модель PII-санитайзера, embedding-модель памяти
just models-fetch

# Опционально (~1080 МБ): офлайн-фолбэк LLM 1.5B (квантованный GGUF/ONNX, CPU-инференс)
just models-fetch --with-fallback-llm

# Разработка только с облаком (без локальной LLM) — флаг по умолчанию в CI:
just dev --cloud-only
```

Модели кладутся в `models/` согласно манифесту (аналог подхода Ollama/LM Studio: версия, sha256, лицензия, размер). Загрузка идёт с зеркала CI-канала (см. doc 09); офлайн-машина может получить `models/` вручную — manifest-driven, без сети не распаковывается.

---

## 8. Конфигурация

### 8.1. Ключи API — только в OS-ключнице

**Ключи никогда не коммитятся и не кладутся в конфиг-файлы репозитория.** Archie хранит их в системной ключнице: Keychain (macOS), Credential Manager (Windows), libsecret/gnome-keyring (Linux). UI-мастер онбординга (`ShortcutsConfig` / настройки Main Dashboard) записывает ключи туда.

```bash
# CLI-путь для dev/CI (пример; сами ключи берутся из вашего хранилища секретов)
just config set-provider deepseek --api-key        # интерактивно, в ключницу
just config set-provider openai    --api-key
just config set-provider gemini    --api-key

# Проверка: ключи читаются из ключницы, но не печатаются
just config check
```

Альтернатива для headless-машин: переменные окружения `ARCHIE_PROVIDER_KEY_<NAME>`, но это dev-only путь; продакшн-путь — ключница (модель угроз — doc 03).

### 8.2. Выбор провайдера и режимы

```toml
# ~/.config/archie/archie.toml (Linux) / %APPDATA%/archie (Win) / ~/Library/Application Support (macOS)
[llm]
primary_provider   = "deepseek"     # deepseek | openai | gemini | local-1.5b
fallback_provider  = "local-1.5b"   # нижняя ступень лестницы деградации

[proactivity]
enabled            = true
mission_states     = true           # GAMING_REST гасит проактивность до 0%
frustration_threshold = 50          # CANON RFC-0001 — не менять без RFC

[privacy]
local_only_mode    = false          # true = ни один байт не покидает машину
telemetry          = false          # opt-in, по умолчанию ВЫКЛ
```

Смена облачного провайдера — **конфигурация, а не код** (ADR-0006): провайдер-трейт в `contracts/`, conformance-тесты гарантируют одинаковые схемы входа/выхода. `local_only_mode = true` переводит систему в полностью локальный контур (лестница деградации замыкается на local 1.5B).

### 8.3. Тюнинг миссий и персоны

- **Миссии** (STUDY_FOCUS / CODING_WORK / IDLE / GAMING_REST) переключаются автоматически по окнам (Minecraft/Steam → GAMING_REST), но домены приложений настраиваются в Main Dashboard → Mission Rules.
- **Полураспад фрустрации** 20/15/5/1 мин (по миссиям) — канон RFC-0001; тюнинг в UI разрешён только в пределах, не ломающих S_max = 100 / порог 50.
- **Пресеты персоны** (таблица `user_profile.persona_preset`): `BIG_BROTHER_MENTOR` (по умолчанию — «Старший брат»), плюс мягкие варианты: `CALM_TUTOR`, `STRICT_COACH`, `SILENT_GUARDIAN`. Промпты — `services/archie-engine/.../llm_clients/persona_prompts.py`.

---

## 9. Рабочий процесс разработки

### 9.1. Тесты

```bash
# Rust (unit + integration crates/*)
cargo nextest run --workspace
cargo clippy --workspace --all-targets -- -D warnings

# Python (engine)
uv run pytest services/archie-engine

# TypeScript (UI-виджеты)
pnpm test

# Всё сразу + линтеры (то же, что CI-гейт)
just test
```

Ключевые SLA-тесты (идентификаторы — в [07-Testing-QA-Strategy.md](07-Testing-QA-Strategy.md)): barge-in latency (< 200 мс, audio loopback), OCR golden set (~200 мс / 185 мс эталон), RAM-регрессия 150/655/1450 МБ, sandbox AST-rejection suite, provider conformance.

### 9.2. Codegen контрактов

Контракты — единственный источник истины для событийного конверта между Rust/Python/TS. После любого изменения в `contracts/`:

```bash
just contracts   # proto + JSON Schema -> Rust (tonic-build), Python (mypy stubs), TS (types)
git diff --exit-code contracts/generated/  # сгенерированный код закоммичен и обновлён
```

Schema-совместимость — часть PR-гейтлиста (§10.3): ломающие изменения конверта требуют RFC.

### 9.3. Документация

Вся проектная документация — этот каталог `docs/` (индекс и правила — [docs/README.md](README.md)); ADR-лог и глоссарий — [12-ADR-Glossary.md](12-ADR-Glossary.md). Изменение архитектурного решения без ADR не мержится.

---

## 10. Контрибьюция

### 10.1. Ветки и коммиты

Ветки: `feat/<краткое-имя>`, `fix/<...>`, `adr/<номер>-<...>`, `docs/<...>` от `main`.

Коммиты — **Conventional Commits 1.0.0**: `<type>(<scope>): <описание>`; типы `feat/fix/docs/adr/refactor/perf/test/chore`; scope = `apps/desktop-ui`, `crates/archie-shell`, `crates/archie-event-bus`, `crates/archie-os-hooks`, `crates/archie-stagingfs`, `services/archie-engine`, `contracts`, `models`, `docs`. Примеры:

```text
feat(crates/archie-event-bus): add API_TIMEOUT debouncing per RFC-0001 E-4
fix(services/archie-engine): freeze tab-stall timer during active video playback
adr(docs): accept ADR-0007 event-bus library selection
```

Ломающие изменения — маркер `!` и футер `BREAKING CHANGE:` (связка с semver-политикой CI/CD — doc 09). Языки: **документы — RU, идентификаторы/термины — EN, коммиты и код — EN** (политика комплекта, см. [docs/README.md](README.md)).

### 10.2. Стиль кода

Стиль — это конфиги, а не проза; CI гоняет то же, что локально (`just lint`):

| Слой | Инструмент | Правила |
|---|---|---|
| Rust | rustfmt (defaults, `rustfmt.toml`), clippy | `-D warnings`; `unsafe` только в аудированных IPC/FFI-модулях с ADR-обоснованием или SAFETY-комментарием |
| Python | ruff (lint+format, `ruff.toml`) | type hints обязательны в `services/archie-engine/`; `py.typed`; зависимости только из `uv.lock` |
| TS | prettier + strict TS | `apps/desktop-ui`; доступность виджета — WCAG 2.2 AA (см. doc 02) |
| Docs | markdownlint-cli2, link checker | fenced-блоки с тегом языка; RU-проза / EN-термины |

### 10.3. Чек-лист PR

- [ ] Тесты зелёные локально и в CI: `cargo nextest`, `uv run pytest`, `pnpm test`
- [ ] `cargo clippy -D warnings` / `ruff` / `prettier` чисты
- [ ] Schema-совместимость: `contracts/` изменён осознанно; ломающие изменения конверта сопровождаются RFC-предложением; codegen перегенерирован (`just contracts`)
- [ ] RFC-0001 числа не изменены (или предложено отдельное RFC-изменение — канон см. §2)
- [ ] Глоссарий обновлён, если введён новый термин ([12-ADR-Glossary.md](12-ADR-Glossary.md))
- [ ] ADR связан, если изменение архитектурное (`docs/adr/`); принятое ADR не редактируется — только supersede
- [ ] Quickstart не сломан (PR, меняющий тулчейн, обязан тронуть §6)
- [ ] Секретов/ключей в диффе нет; аудио-фикстуры с реальными голосами не коммитятся (только синтетика)

### 10.4. Политика регрессии SLA (RFC-0001)

Бюджеты из §2 — это **контракт приёмки, а не ориентир**. Правила:

1. Каждый PR, затрагивающий горячий аудио-путь (`audio_loop/`), шину, OCR-конвейер или память, прогоняет SLA-тесты в CI — падение любого бюджета = красный билд, исключений нет.
2. Регрессия бюджета (например, barge-in 210 мс или Peak RAM 700 МБ) трактуется как **падение теста**, а не как «приемлемое отклонение»: фикс в том же PR, либо откат.
3. Изменить само число можно только через RFC-процесс (новый RFC или ADR-supersession с перекрёстными ссылками); правка цифры в PR мимо RFC отклоняется ревью автоматически.
4. RAM-регрессия проверяется на эталонной CI-машиине по трём состояниям (Idle 150 / Peak 655 / Offline 1450 МБ) — methodology в doc 07.

### 10.5. Пайплайн ревью и ADR-процесс

- Ревью-SLA: первый ответ — 1 рабочий день (Phase 1, малая команда).
- CODEOWNERS маршрутизирует: `crates/archie-shell` → владелец аудио/шелла; `services/archie-engine` → ML-owner; `contracts/` → RFC-owner (имена — doc 08).
- Архитектурное предложение: issue-шаблон `adr_proposal` → черновик `docs/adr/NNNN-*.md` (MADR-lite: Context / Decision / Consequences / Alternatives) → PR → ревью. Принятые ADR **неизменяемы**; смена решения = новый ADR со ссылкой `supersedes`. Для Phase 4 (SOC 2) git-история ADR — аудиторский артефакт.

---

## 11. Сообщество и поддержка

- Баги и фичи — issue-шаблоны: `bug_report.md`, `feature_request.md`, `adr_proposal.md`.
- Вопросы по документации — сначала [docs/README.md](README.md) и глоссарий.

### Security

**Не открывайте security-issues публично.** Уязвимости (особенно: обход SOP-песочницы, инъекция в blueprint-компилятор, обход Archie_Staging/Permission Tiers, утечка PII мимо санитайзера) отправляйте приватно на **security@archie.local** (placeholder — адрес публикуется вместе с open-source-решением, см. doc 09). SLO ответа: 48 часов на триаж, 90 дней на патч-координацию перед раскрытием. Публичный `SECURITY.md` появится в Phase 2 (OSS-readiness), модель угроз — [03-Threat-Model-Security.md](03-Threat-Model-Security.md).

---

## 12. FAQ

**Пишет ли Archie мой экран?**
Нет. Записи видео/скриншотов не существует в принципе. Z-Sense — событийный парсинг: по триггерам ОС (смена окна, выделение текста, ошибка консоли) читается текст через Accessibility API или локальный OCR (~200 мс на CPU) и превращается в текстовый JSON-сниппет. Экономия 99% трафика, полная приватность визуальных данных.

**Слушает ли микрофон постоянно?**
Аудио-контур активен только с явного разрешения микрофона (opt-in на первом запуске) и всегда локален: Silero VAD (1–2 МБ, < 1 мс/чанк) гейтит поток на вашей машине — в облако уходит только текст реплики, и то после PII-санитайзера. В TTS-эхоподавлении AEC3 вычитает голос самого Archie. Микрофонный тракт можно полностью отключить — останется текстовый режим.

**Что покидает мой компьютер?**
Только санитизированные JSON-сниппеты контекста: PII Data Sanitizer (regex + локальная NER) замещает пароли, ключи API, номера карт и проприетарный код тегами `[REDACTED_API_KEY]`, `[REDACTED_FINANCIAL_DATA]` **до** любого облачного вызова. Голос Archie, все модели, векторная база (LanceDB), SQLite и Archie_Staging — локальны. `local_only_mode = true` гарантирует нулевой исходящий трафик.

**Что будет без интернета?**
Спроектированная деградация, а не «кирпич»: `API_TIMEOUT` переключает роутинг на локальную модель 1.5B на CPU (RAM-потолок 1450 МБ). Голос, характер, локальные действия (сортировка файлов, музыка, запуск программ) сохраняются; отключается помощь с кодом и сложной математикой. Archie честно предупреждает: «Интернет отвалился, я пока без тяжелых мозгов».

**Может ли Archie удалить мои файлы?**
Нет — системный вызов Delete физически заблокирован в archie-stagingfs. «Очистка рабочего стола» переносит файлы в карантин `~/Archie_Staging`; критичные действия (git push, письма) требуют карточки Tier 2 Critical Approval; SOP-скрипты проходят AST-валидацию (никаких import/exec/сети вне MCP tool-call DSL).

**Archie помнит всё вечно?**
Память трёхуровневая и прозрачная: L1 — RAM-буфер последних 2–4 часов; L2 — ночные Markdown-сводки в ваш Obsidian/Notion (через MCP, вы видите каждый файл); L3 — векторный граф фактов/предпочтений в локальной LanceDB. Всё хранится у вас; Main Dashboard → Memory Explorer показывает и удаляет любые записи.

**Нужно ли покупать ключи всех провайдеров?**
Нет, BYOK: достаточно одного ключа (DeepSeek/OpenAI/Gemini — на выбор, смена = конфиг). Без ключей вообще работает локальный режим с моделью 1.5B (ограниченный когнитивный контур).

**Какие ОС поддерживаются?**
Windows 10/11, macOS 12+, Linux (X11 и Wayland через xdg-desktop-portal). Требования: 8 ГБ RAM минимум, GPU не нужен.

---

## 13. Дорожная карта

| Фаза | Релиз | Ключевой скоуп | Критерий приёмки (выборка RFC-0001) |
|---|---|---|---|
| 1 | MVP Alpha | Гибридное ядро + Event Bus + Graceful Degradation; Z-Sense (RapidOCR); аудио-цикл с Barge-in; Tauri-шелл (Soft Widget, трей) | Barge-in < 200 мс; VAD 85–100 мс (> 95%); OCR ~200 мс; деградация на 1.5B при API_TIMEOUT без потери голоса |
| 2 | B2C Beta | Mission-State Machine + Индекс фрустрации + Soft Widget; трёхуровневая память + ночной MCP-дневник (Obsidian/Notion); Permission Tiers + Archie_Staging; онбординг Path A/B | 0% проактивности в GAMING_REST; срабатывание при >= 50 баллах; окно отмены 3–5 с; 100% ночная .md-сводка через MCP |
| 3 | v1.0 Production | Interactive Blueprints (Smart Blueprints / SOP) + Semantic Router + Python-песочница | Обход облачной LLM при косинусном совпадении >= 0.98; AST-rejection suite зелёный; предсказуемое исполнение без галлюцинаций |
| 4 | v2.0 Enterprise | B2B: PII-санитайзер Enterprise-класса, MCP-коннекторы Jira/Slack, Junior Mentor, Burnout Tracker, On-Premise | 100% маскирование `[REDACTED_*]`; работа внутри закрытого периметра; SOC 2-евиденции из ADR-лога |

Детали фаз — в [01-PRD.md](01-PRD.md) и [08-Project-Plan-Team.md](08-Project-Plan-Team.md) (даты — там же).

### Документация

| # | Документ | О чём |
|---|---|---|
| 01 | [01-PRD.md](01-PRD.md) | Продуктовые требования, персоны, user stories, NFR |
| 02 | [02-UX-Voice-UX.md](02-UX-Voice-UX.md) | UX/UI и голосовой UX, персона, состояния, доступность |
| 03 | [03-Threat-Model-Security.md](03-Threat-Model-Security.md) | Модель угроз STRIDE, SOP-песочница, PII, хранение ключей |
| 04 | [04-Market-Competitor-Analysis.md](04-Market-Competitor-Analysis.md) | Рынок и конкуренты, позиционирование |
| 05 | [05-Business-Model-Financial-Plan.md](05-Business-Model-Financial-Plan.md) | Бизнес-модель и финансовый план |
| 06 | [06-Go-to-Market-Strategy.md](06-Go-to-Market-Strategy.md) | GTM-стратегия, метрики, pivot-триггеры |
| 07 | [07-Testing-QA-Strategy.md](07-Testing-QA-Strategy.md) | Пирамида тестирования, SLA-тесты, CI-гейты |
| 08 | [08-Project-Plan-Team.md](08-Project-Plan-Team.md) | План, WBS, команда, RACI, риски |
| 09 | [09-CI-CD-Distribution.md](09-CI-CD-Distribution.md) | CI/CD, кроссплатформенные сборки, раздача моделей |
| 10 | [10-Observability-Telemetry-Privacy.md](10-Observability-Telemetry-Privacy.md) | Телеметрия, краш-репорты, privacy |
| 11 | [11-External-API-MCP-Contracts.md](11-External-API-MCP-Contracts.md) | Внешние API, MCP-контракты, версионирование |
| 12 | [12-ADR-Glossary.md](12-ADR-Glossary.md) | ADR-лог решений и глоссарий проекта |
| 13 | этот документ | README, quickstart, контрибьюция |

Исходники уровня концепции — в корне `../` (PAD, архитектурная спецификация, RFC-0001); индекс — [docs/README.md](README.md).

---

## 14. Лицензия

**Лицензия: TBD (placeholder).** Выбор (проприетарная / open-source + DCO или CLA, возможная схема «код — закрыт, docs/ — CC BY») заблокирован на GTM-решении об open-sourcing — см. открытые вопросы [06-Go-to-Market-Strategy.md](06-Go-to-Market-Strategy.md) и риски doc 09. До решения: © Archie Team, все права защищены, распространение кода и весов моделей запрещено.

Веса моделей в `models/` распространяются по **собственным лицензиям артефактов** (ONNX/GGUF model cards в `models/MANIFEST`), независимым от будущей лицензии кода. Товарный знак «Archie» — статус регистрации RU/EU/US подтверждается перед публичным README (открытый вопрос, doc 06/08).

---

*Числа и термины этого документа синхронизированы с RFC-0001 и глоссарием (doc 12) согласно правилам [docs/README.md](README.md). Найденное расхождение — doc-баг: заводите issue `docs/13`.*
