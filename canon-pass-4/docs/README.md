# Archie — OS-Level AI Symbiote (MVP)

**Статус:** Phase 1 MVP (v0.1.0) · Полное видение — [RFC-0001 / docs-комплект](../test/Archie/docs/README.md)

> Это README **реального MVP-репозитория**, а не концепт-документа docs/13. Структура и команды зеркалируют [docs/13-README-Quickstart.md](../test/Archie/docs/13-README-Quickstart.md), но скоуп честно урезан до того, что реально работает в Phase 1. Все канонические числа из RFC-0001 здесь не заявляются как достигнутые — см. таблицу упрощений ниже.

> **Honest product copy (независимо от маркетинга):** Archie — гибридный edge+cloud ассистент: локально работают сенсорика, память, PII-обработка и офлайн-фолбэк; облачный LLM-контур подключается отдельно по opt-in/конфигурации. Сырые кадры экрана и аудио предназначены оставаться на устройстве; через облачный boundary допускается только санированный текстовый контекст — при включённых и проверенных sanitizer/egress-контролях (текущий MVP sanitizer — базовый scaffold, не полный regex+NER RFC; это не безусловно доказанная гарантия без egress-тестов).

---

## Содержание

1. [Что такое Archie](#1-что-такое-archie)
2. [Скоуп MVP: что реально работает](#2-скоуп-mvp-что-реально-работает)
3. [Упрощения MVP vs RFC-0001](#3-упрощения-mvp-vs-rfc-0001)
4. [Структура репозитория](#4-структура-репозитория)
5. [Предварительные требования](#5-предварительные-требования)
6. [Quickstart: запуск MVP](#6-quickstart-запуск-mvp)
7. [Конфигурация](#7-конфигурация)
8. [Рабочий процесс разработки](#8-рабочий-процесс-разработки)
9. [Типичные проблемы (troubleshooting)](#9-типичные-проблемы-troubleshooting)
10. [Дорожная карта](#10-дорожная-карта)
11. [Лицензия](#11-лицензия)

---

## 1. Что такое Archie

**Archie («Архи»)** — автономный проактивный ИИ-напарник, интегрированный на уровень операционной системы. Полное видение (гибридный Edge+Cloud-контур, Z-Sense, полнодуплексный голос с barge-in < 200 мс, трёхуровневая память) — в RFC-0001 и PAD.

**Этот репозиторий — Phase 1 MVP.** В нём уже есть два контура из итоговой архитектуры:

- **Python-engine** (`services-archie-engine/`) — ядро когнитивной логики: индекс фрустрации (порт формулы RFC-0001), машина миссий, LLM-клиенты (`aijuju` qwen3.8-flash + офлайн `local` Qwen GGUF + детерминированный `mock`), персоны «Старшего Брата», gRPC-сервер поверх TCP loopback.
- **Tauri v2 UI** (`apps-desktop-ui/`) — десктопный шелл (Main Dashboard, Soft Widget).

---

## 2. Скоуп MVP: что реально работает

| Компонент | Состояние в MVP |
|---|---|
| Индекс фрустрации | ✅ Работает: `S(t) = min(S_max, S_prev·e^(−λ·dt) + μ·ψ·dS)`, полураспад 20/15/5/1 мин по миссиям, порог 50 — порт канона RFC-0001 (unit-тесты) |
| Машина миссий | ✅ Работает: STUDY_FOCUS / CODING_WORK / IDLE / GAMING_REST, семантический классификатор на локальной Qwen3-0.6B (fallback — эвристика по заголовкам окон) |
| LLM-контур | ✅ Работает: провайдер `aijuju` (OpenAI-совместимый агрегатор, qwen3.8-flash) + провайдер `local` (офлайн-персона на Qwen3 GGUF через `llama_shared`) + провайдер `mock` (детерминированный, для тестов); каскад graceful-degradation `aijuju→local→mock` при GatewayTimeout; `gemini` оставлен как альтернатива (геоблокирован в RU) |
| Персоны | ✅ Промпты `BIG_BROTHER_MENTOR` и заготовки мягких вариантов |
| Память | ✅ Работает: Tier-1 (буфер 500) + Tier-2 эпизоды (SQLite + FTS5-trigram, свёртка Qwen3-1.7B при смене миссии; episodic-маркер открытого эпизода + orphan-recovery после падения воркера; слияние коротких эпизодов) + Tier-3 профиль; **ночной rebuild** профиля (`_nightly_loop`, catch-up при старте, событие `NIGHTLY_PROFILE_REBUILD`); эпизоды и профиль подмешиваются в промпт персоны |
| Голос Арчи | ✅ Работает: TTS Piper, реплика звучит **после окна беззвучной отмены 3–5 с** (default 4 с) — юзер может от mute-нуть вмешательство до того, как услышит голос; конец озвучивания закрывается событием `SYS_TTS_STOP` (SoftWidget выходит из speaking) |
| Микрофон / barge-in | ✅ Работает: VAD (silero на onnxruntime, без torch) + barge-in half-duplex (TTS глушится при речи юзера) + детекция вздоха (+30 к фрустрации, HPF-фикс DC-офсета) |
| STT | ✅ Работает: **sherpa-onnx ИЛИ faster-whisper** (смена через `ARCHIE_STT_BACKEND`): sherpa — streaming zipformer-ru int8 (28 МБ); whisper — whisper-small ct2 int8 (~460 МБ, `models/whisper-ru-small/`), батчевое распознавание с эхо-защитой feed()-проводки. Фразы юзера → Tier-1 |
| IPC engine ↔ UI | ✅ gRPC поверх TCP loopback (port-file `/tmp/archie-ipc-<uid>.port`), контракты в `contracts/` (proto + сгенерированный Python) |
| Z-Sense | ✅ Работает: снимок экрана `mss`/grim + OCR (RapidOCR) + PII-санитайзер; контекст экрана попадает в промпт LLM |
| Миссии/триггеры | ⚠️ Упрощён: поллинг `hyprctl` (активное окно), а не события OS-hooks |
| Аудио-тракт | ⚠️ Каркас: полный дуплекс с AEC3 — нет (half-duplex скипом чанков при играющем TTS) |
| SOP/песочница/Permission Tiers | ❌ Не в MVP |
| MCP-hub | ❌ Не в MVP |

**Текущий Phase 1 MVP против цели/дорожной карты**

> Арчи — гибридный Edge+Cloud ассистент (не полностью локальный): локально работают сенсорика (VAD, Z-Sense), память, базовый PII-санитайзер и офлайн-фолбэк; облачный LLM-контур (aijuju) подключается отдельно по opt-in/конфигурации. Облачный путь — **конфигурируемый и необязательный**, не предполагается как всегда-подключенный. Основной контур — `mock` офлайн без ключей и сети; `local` — офлайн Qwen на GGUF. Ночной rebuild профиля и ASR-треки работают офлайн; облако — opt-in с graceful каскадом к офлайну.

> **Приватность:** Сырые кадры экрана и аудио должны оставаться на устройстве; через облачный boundary допускается только санитизированный текстовый контекст. Текущий базовый PII-санитайзер — **скелет, не полный regex+NER**; это нельзя рекламировать как безусловно доказанную гарантию без egress тестов. Telemetry не реализована в Phase 1 MVP; crash upload/analytics/support bundle — запланированы/условны opt-in.

> **Cloud provider:** aijuju (qwen3.8-flash, OpenAI-совместимый) — дефолт для конфига `ARCHIE_LLM_PROVIDER`, но API-ключ — необязательный (`ARCHIE_AIJUJU_API_KEY`, только через `.env`). Клавиши gemini нет (геоблокирован в RU). Облачный контур — **opt-in/configurable**, не always-on.

> **Контрольные точки упрощения против RFC-0001** (§3):
> 1) Полнодуплексный AEC3 — нет (half-duplex); 2) Shared-memory ring buffer — нет (gRPC TCP); 3) Z-Sense OS-хуки — нет (poll-модель mss+OCR); 4) Миссии OS-hooks — нет (hyprctl poll); 5) Безопасность sandbox и SOP — нет (карантин-каталог есть, блокировка Delete — нет); 6) Semantic Router + SOP — нет; 7) Full PII-санитайзер — нет (базовая заготовка); 8) Permission Tiers — нет; 9) SOP-песочница/инъекции — нет; 10) Облачное окружение — opt-in, не always-on; 11) Телметрия — не реализована в Phase 1.

> **Правило честности README:**
> • Все компоненты из §2 со статусом ✅ работают.
> • Все, что не отмечено ✅ в §2, — не работает.
> • Числа из RFC-0001 (barge-in < 200 мс, OCR ~200 мс, RAM-бюджеты) **не заявляются** — см. §3 таблицы упрощений.
> • Облачные облака — **конфигурируемые**, не always-on: база `mock`, `local` — офлайн-персона, аijuju — optional с ключом в `.env`.
> • Полноценная sandbox (SOP, санитайзер, permission tiers, telemetry) — в Phase 2–3.

---

## 3. Упрощения MVP vs RFC-0001

**Этот раздел — главный честный блок README.** Каждая строка — осознанное отклонение MVP от канона RFC-0001, а не скрытый долг:

| # | RFC-0001 (канон) | MVP Phase 1 (реально) | Почему | План |
|---|---|---|---|---|
| 1 | Barge-in < 200 мс через WebRTC AEC3 в одном аудио-процессе | Barge-in есть (VAD + half-duplex: скип чанков при играющем TTS), но без AEC3 — задержка определяется VAD-окном (~500 мс), а не < 200 мс | AEC3/полнодуплекс — отдельная большая подсистема; half-duplex покрывает сценарий «юзер перебил Арчи» | Phase 2 |
| 2 | PCM-фреймы через shared-memory ring buffer (data-plane) | Весь трафик — gRPC-сообщения через TCP loopback | Shared-memory ring buffer не нужен без аудио-потока | Phase 2 (вместе с №1) |
| 3 | Z-Sense: событийные OS-hooks триггеры + Accessibility API | Снимок экрана библиотекой `mss` + OCR по событию (poll-модель) | Accessibility-дерево и event-hooks на 3 ОС — большой кроссплатформенный слой | Phase 2–3 |
| 4 | Миссии по событиям окон/вкладок (OS-hooks) | Поллинг `hyprctl activewindow` с интервалом | Достаточно для Linux-dev-стенда MVP; Wayland-протоколы событий сложнее | Phase 2 |
| 5 | Offline-фолбэк: локальная LLM 1.5B (GGUF/ONNX, RAM 1450 МБ) | Локальная Qwen3-1.7B (GGUF Q8_0, `llama_shared` синглтон) уже работает: семантика миссий, свёртка памяти **и офлайн-персона** (`local`-провайдер, graceful-каскад при таймауте облака) | 1.7B хватает для ответов персоны в офлайне; RAM ~2.1 ГБ на воркер выше канона — осознанное отклонение | — (сделано в Phase 1) |
| 6 | RAM-бюджет 150/655/1450 МБ | Не измеряется как SLA-гейт | Бюджет имеет смысл на финальной топологии | Phase 3 |
| 7 | PII Data Sanitizer до любого облачного вызова | Базовая заготовка (модуль `security/`), не полный regex+NER-конвейер | MVP гоняется на dev-машине с контролируемым контекстом; **облако — opt-in** | Phase 2 (до любого реального использования) |
| 8 | Permission Tiers 1/2/3 + Archie_Staging с блокировкой Delete | Карантин-каталог есть, блокировка syscall Delete — нет | Полноценная staging-FS — большой Rust-слой | Phase 3 |
| 9 | Semantic Router (косинус >= 0.98) + SOP-песочница | Прямой роутинг запроса к провайдеру, без SOP | Router без SOP-библиотеки не имеет смысла | Phase 3 |
| 10 | Память L0/L1/L2 + LanceDB + ночные сводки | Своя SQLite-схема: Tier-2 эпизоды (FTS5-trigram поиск, orphan-recovery, слияние коротких) + Tier-3 профиль; свёртка — локальная Qwen3-1.7B; ночная пересборка профиля — работает (`_nightly_loop` с catch-up) | LanceDB/эмбеддинги не нужны для 5-50 эпизодов одно-юзерного MVP (вердикт разведки); FTS5 покрывает поиск | — (ночные сводки сделаны в Phase 1) |

**Правило честности:** числа RFC-0001 (barge-in < 200 мс, OCR ~200 мс, RAM-бюджеты) в этом репо **не заявляются**. Всё, что не в таблице §2 со статусом ✅, — не работает.

---

## 4. Структура репозитория

```text
archie-os-symbiote/
├── apps-desktop-ui/          # UI: Tauri v2 + React/TS (Main Dashboard, Soft Widget)
│   └── src-tauri/            #   Rust-шелл (окна, трей)
├── services-archie-engine/   # Python 3.12 venv (uv): движок — proactivity, LLM, IPC
│   └── src/archie_engine/
│       ├── proactivity/      #   индекс фрустрации (RFC-0001 §4) + машина миссий
│       ├── llm_clients/      #   cloud_gateway (aijuju/gemini), local (Qwen GGUF), mock, persona_prompts
│       ├── zsense/           #   mss+OCR (упрощённый, см. §3)
│       ├── audio_loop/       #   каркас (TTS Piper)
│       ├── memory/, security/, ipc/
│       └── config.py         #   env-конфигурация (секреты — только env!)
├── contracts/                # archie_v1.proto + gen_py/ (сгенерированный gRPC-код)
├── storage/                  # runtime-данные: archie_core.sqlite3, engine.log (gitignored)
├── run/
│   ├── dev.sh                # dev-стенд: engine в фоне + tauri dev
│   └── env.example           # шаблон env (без реальных ключей)
├── justfile                  # команды разработки (см. §8)
└── README.md                 # этот файл
```

> Расхождение с docs/13 §4 (`apps/`, `crates/`, `services/`): MVP использует плоские имена `apps-desktop-ui/`, `services-archie-engine/`, `contracts/` — Rust-крейты (event-bus, os-hooks, stagingfs) ещё не выделены; их функционал частично живёт в `src-tauri/` Tauri-шелла. Это осознанное отклонение от целевой топологии.

---

## 5. Предварительные требования

| Инструмент | Версия | Назначение |
|---|---|---|
| Rust (rustup, stable) | >= 1.90 (`src-tauri` MSRV) | Tauri-шелл |
| uv | latest | Python 3.12 + зависимости engine |
| Node.js + pnpm | Node 20+, pnpm 9+ | apps-desktop-ui |
| just | >= 1.5 | команды quickstart (опционально — есть `run/dev.sh`) |
| bash | 5.x | run/dev.sh |

Системные зависимости Tauri v2 на Fedora:

```bash
sudo dnf install webkit2gtk4.1-devel openssl-devel curl-devel \
     libappindicator-gtk3-devel librsvg2-devel alsa-lib-devel \
     pulseaudio-libs-devel libudev-devel
```

> Из активной Hyprland-сессии **не перезапускать GDM/демоны** (см. уроки системы); Tauri dev в этом не нуждается.

---

## 6. Quickstart: запуск MVP

### 6.1. Подготовка окружения

```bash
git clone <REPO_URL> archie-os-symbiote
cd archie-os-symbiote

# Проверка инструментов + uv sync (engine venv) + pnpm install (UI)
just setup
```

### 6.2. Запуск dev-стенда (рекомендуемый путь)

```bash
# engine в фоне (mock-провайдер, офлайн) + Tauri UI в foreground;
# при выходе из UI engine прибивается автоматически (trap)
just dev          # эквивалент: bash run/dev.sh
```

Что делает `run/dev.sh`: экспортирует env из `run/env` (если есть), стартует engine с логом в `storage/engine.log`, ждёт появления UDS-сокета (до 30 с), затем `pnpm exec tauri dev` в foreground.

### 6.3. Запуск по частям (два терминала)

```bash
# Терминал 1 — engine с офлайн mock-провайдером (без ключей и сети)
just engine-dev

# Терминал 1 (вариант) — engine с офлайн-персоной local (Qwen GGUF, без сети)
cd services-archie-engine
PYTHONPATH="../contracts/gen_py:src" ARCHIE_LLM_PROVIDER=local \
  bin/python -m archie_engine --provider local

# Терминал 1 (вариант) — engine с облачным aijuju (ключ из run/env или env)
cd services-archie-engine
set -a; . ../run/env; set +a
PYTHONPATH="../contracts/gen_py:src" ARCHIE_LLM_PROVIDER=aijuju \
  bin/python -m archie_engine --provider aijuju

# Терминал 1 (legacy) — engine с Gemini (геоблокирован в RU, не рекомендуется)
just engine-gemini        # требует run/env (см. §6.4)

# Терминал 2 — UI
just ui-dev
```

**Критерий успеха «первых 5 минут» (MVP, честный):** port-файл `/tmp/archie-ipc-<uid>.port` появился, окно Archie (Main Dashboard) открылось, `storage/engine.log` пишет строки старта engine. Голосовой контур и проактивные вмешательства — **не** входят в критерий успеха MVP (см. §3).

### 6.4. Ключ API aijuju

MVP работает офлайн (`mock` без ключей и сети; `local` — Qwen GGUF из `models/`, без ключей). Облачный контур — opt-in:

```bash
cp run/env.example run/env   # run/env в .gitignore
$EDITOR run/env              # подставьте свой ключ ARCHIE_AIJUJU_API_KEY
# затем запуск с ARCHIE_LLM_PROVIDER=aijuju (см. §6.3)
```

`gemini` геоблокирован в RU и **не рекомендуется**: эндпоинт generateContent периодически недоступен, движок честно деградирует в local/mock. Ключ `ARCHIE_GEMINI_API_KEY` нужен только для legacy-рецепта `just engine-gemini`.

---

## 7. Конфигурация

**Секреты — только через env, никогда в файлах репозитория** (см. `.gitignore`: `.env`, `run/env`).

| Переменная | По умолчанию | Значение |
|---|---|---|
| `ARCHIE_LLM_PROVIDER` | `aijuju` | `gemini` \| `aijuju` \| `mock` \| `local` (каскад деградации при таймауте: `aijuju`/`gemini` → `local` → `mock`) |
| `ARCHIE_AIJUJU_API_KEY` | — (пусто) | ключ облачного провайдера aijuju (только env, не в файлах) |
| `ARCHIE_GEMINI_API_KEY` | — (пусто) | ключ Gemini; только для legacy `engine-gemini` (геоблок RU) |
| `ARCHIE_STT_BACKEND` | `off` | `off` \| `sherpa` \| `whisper` — бэкенд распознавания речи |
| `ARCHIE_NIGHTLY_REBUILD_ENABLED` | `1` | ночная пересборка профиля памяти (вкл/выкл) |
| `ARCHIE_NIGHTLY_HOUR` | `4` | час ночного rebuild (локальное время; catch-up при старте, если профиль старше 20 ч) |
| `ARCHIE_MIC_ENABLED` | `0` | микрофонный контур (VAD + barge-in + вздох); включать после e2e-проверки |
| `ARCHIE_SEMANTIC_MODEL` | `Qwen3-1.7B-Q8_0.gguf` | GGUF-модель семантики/памяти/офлайн-персоны из `models/` |
| `ARCHIE_PIPER_VOICE` | `ru_RU-ruslan-medium` | голос TTS (вес в `models/`) |
| `ARCHIE_UDS_PATH` | `/tmp/archie-ipc-<uid>.sock` | legacy UDS-путь (IPC фактически по TCP, см. §2) |

Шаблон — [`run/env.example`](run/env.example). Канонические параметры фрустрации (порог 50, S_max 100, полураспад 20/15/5/1 мин) зашиты в `config.py` как константы RFC-0001 — **не менять без RFC**.

---

## 8. Рабочий процесс разработки

Все команды — через `just` (justfile в корне):

| Рецепт | Что делает |
|---|---|
| `just setup` | проверяет наличие cargo/node/pnpm/uv/python3, `uv sync --extra dev`, `pnpm install` |
| `just engine-dev` | engine с провайдером `mock` (PYTHONPATH на `contracts/gen_py` + `src`) |
| `just engine-gemini` | engine с провайдером `gemini` (legacy; геоблок RU — см. §6.4) |
| `just ui-dev` | `cd apps-desktop-ui && pnpm exec tauri dev` |
| `just ui-check` | `tsc` + `vite build` (type-check + сборка фронтенда) |
| `just py-test` | `pytest` из venv engine |
| `just all-check` | `py-test` + `ui-check` + `cargo check` (src-tauri) |
| `just dev` | dev-стенд целиком (`bash run/dev.sh`) |
| `just clean` | удаляет `dist/`, `target/`, `__pycache__` (не трогает `storage/`) |

Для `aijuju`/`local` готовых рецептов нет (см. §6.3 — запуск руками с `ARCHIE_LLM_PROVIDER`); `just engine-gemini` сохранён только как legacy.

```bash
just all-check   # локальный эквивалент CI-гейта MVP
```

---

## 9. Типичные проблемы (troubleshooting)

| Симптом | Причина | Решение |
|---|---|---|
| `ARCHIE_STT_BACKEND=whisper`, но STT не работает | нет модели `models/whisper-ru-small/` (model.bin/config.json/tokenizer.json) | backend вернёт `None` и воркер продолжит без STT (warning в логе); скачать ct2 int8-модель в `models/whisper-ru-small/` |
| `ARCHIE_LLM_PROVIDER=local`, ответы — заглушки | нет GGUF-файла в `models/` или не установлен llama-cpp-python | local-провайдер бросает GatewayTimeout → воркер деградирует в `mock` (событие API_TIMEOUT); проверить `models/Qwen3-1.7B-Q8_0.gguf` и `uv sync --extra semantic` |
| `run/dev.sh`: «engine умер до поднятия сокета» | engine упал на старте | смотреть `storage/engine.log` (скрипт печатает последние 30 строк) |
| `run/dev.sh`: «сокет не появился за 30 с» | долгий импорт ML-зависимостей или engine висит | `tail storage/engine.log`; увеличить таймаут в скрипте; проверить, что не занят старый сокет `rm /tmp/archie-ipc-<uid>.sock` |
| `just engine-gemini` выходит сразу | `ARCHIE_GEMINI_API_KEY` не задан / `your_key_here` | создать `run/env` из `env.example` и вписать ключ (но gemini геоблокирован в RU — лучше aijuju, §6.4) |
| `bin/python: No module named archie_engine.__main__` | entrypoint engine ещё не реализован в текущей ветке | обновиться до ветки с `src/archie_engine/__main__.py`; engine-dev/ui рецепты уже настроены под него |
| `pnpm exec tauri dev` падает на линковке | системные зависимости Tauri не установлены | `sudo dnf install ...` из §5; **не** перезапускать GDM из сессии |
| `cd services-archie-engine && uv sync` ничего не ставит | uv ищет `[project]`-манифест — он там и лежит | проверьте, что запускаете `uv sync --extra dev` внутри `services-archie-engine/` |
| Ошибка «No module named archie_ipc» | не задан PYTHONPATH | это делают рецепты just/dev.sh: `PYTHONPATH=contracts/gen_py:services-archie-engine/src` |
| Сокет занят от прошлой сессии | orphaned engine | `rm /tmp/archie-ipc-<uid>.sock`; `pgrep -f archie_engine` и прибить |

---

## 10. Дорожная карта

| Фаза | Скоуп | Что закрывает из таблицы §3 |
|---|---|---|
| 1 — **этот MVP** | Ядро engine + фрустрация/миссии + aijuju/local/mock + Tauri UI + gRPC TCP IPC | — |
| 1.5 — **сделано в Phase-1 цикле 0a257bd+f632f4d** | ночной rebuild профиля (sleep-time compute), ротация/слияние эпизодов, локальная офлайн-персона (Qwen GGUF), faster-whisper STT (460 МБ int8), VAD на onnxruntime | — |
| 2 | Аудио-дуплекс (AEC3 — полноценный полнодуплекс), OS-hooks события, Z-Sense событийный, AEC-гейты | №1, №2, №3, №4, №7 |
| 3 | Permission Tiers + staging-FS, Semantic Router + SOP, RAM-SLA-гейты | №6, №8, №9 |
| 4 | Enterprise: PII-enterprise, MCP-коннекторы, SOC 2 | — |

---

## 11. Лицензия

**TBD (placeholder), как и в docs/13 §14.** До решения: © Archie Team, все права защищены. Секреты и ключи в репозиторий не попадают (`.gitignore`: `.env`, `run/env`, `storage/`, `models/`).

---

*Честность скоупа — главный принцип этого README: всё, что заявлено работающим, перечислено в §2; всё остальное — в таблице упрощений §3. Расхождение README с реальным кодом — doc-баг.*

## Источники (факты vs road-map)

Этот README описывает **Phase 1 MVP**. Факты проверены по коду и конфигурации на момент последнего обновления; road-map приведена из `Документация/` (RFC-0001, 03-Threat-Model, 10-Observability, 11-External-API-MCP-Contracts):

- **Гибрид edge+cloud (не fully-local):** `services-archie-engine/src/archie_engine/config.py` — `ARCHIE_LLM_PROVIDER` дефолт `aijuju`, `ARCHIE_AIJUJU_API_KEY` из env (опционален); каскад `aijuju→local→mock` в `ipc/worker.py` (§8.8).
- **Локальные/офлайн:** `local`-провайдер (Qwen3-1.7B GGUF, `llama_shared`), `mock`; VAD (silero onnxruntime), STT (sherpa/faster-whisper), TTS Piper; память SQLite+FTS5 + `_nightly_loop`.
- **Сырой контекст:** Z-Sense `mss`+OCR; `security/pii_sanitizer.py` — базовый scaffold, не полный regex+NER.
- **Telemetry в Phase 1 — не реализована:** в коде только локальные логи и SQLite-таблица `frustration_telemetry` (см. `memory/sqlite_repository.py`); egress/crash upload/analytics/support bundle — planned/conditional opt-in (road-map, `10-Observability-Telemetry-Privacy.md`).
- **On-Prem/VPC:** не реализовано (только conditional Enterprise v2.0).
- **Канонические числа RFC-0001 (Barge-in < 200 мс, OCR ~200 мс, RAM 150/655/1450 МБ) не заявляются как достигнутые:** фактический RSS рабочего воркера ~2.4 ГБ, AEC3 полнодуплекса нет.

## Запуск (проверено 2026-11-02)

Стек под systemd (user):
```bash
systemctl --user start archie-vite   # фронтенд (Vite, порт 5173)
systemctl --user start archie-app    # приложение + движок
```
Окно «Archie» открывается на активном workspace. Провайдер LLM по умолчанию —
aijuju (qwen3.8-flash; ключ `ARCHIE_AIJUJU_API_KEY` в юните). При таймауте
облака движок деградирует в local (офлайн Qwen), затем в mock — честно
предупреждает голосом (событие API_TIMEOUT). Gemini геоблокирован в RU
(наблюдалось 02.11.2026) и не рекомендуется.

## Статус MVP (финал)
- Конвейер: хуки Hyprland → шина gRPC (TCP loopback) → движок → LLM → TTS ✅
- Память: эпизоды + профиль + ночной rebuild (событие NIGHTLY_PROFILE_REBUILD) ✅
- Голос: реплика после окна беззвучной отмены 4 с; SYS_TTS_STOP по концу речи ✅
- STT: sherpa ИЛИ faster-whisper (смена `ARCHIE_STT_BACKEND`) ✅
- 251/251 python-тестов (services-archie-engine, venv `bin/python`), cargo build, tsc, vite build — все зелёные ✅
- Отклонения от RFC-0001: TCP вместо UDS (grpcio/tonic несовместимы по UDS),
  TTS через плеер-субпроцесс (AEC3-аудио в Phase 2), OS-хуки поллингом hyprctl.
