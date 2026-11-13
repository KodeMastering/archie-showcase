# 09 — CI/CD, Distribution & Release Engineering

**Проект:** Archie (OS-Level AI Symbiote) — гибридный Edge+Cloud проактивный десктоп-ассистент.
**Статус:** PROPOSED / READY FOR IMPLEMENTATION. **Целевой релиз:** поддержка всех фаз 1–4 (MVP Alpha → v2.0 Enterprise).
**Связанные документы:** RFC-0001 (IPC, RAM-бюджет, SLA), 03-Threat-Model-Security.md (канал обновлений как attack surface), 07-Testing-QA-Strategy.md (SLA-гейты), 10-Observability-Telemetry-Privacy.md (crash-free метрики), 08-Project-Plan-Team.md (закупка сертификатов, второй releaser).

---

## 1. Executive Summary

Документ определяет, как коммит в монорепозитории `archie-os-symbiote/` превращается в подписанный, автоматически обновляемый инсталлятор для Windows / macOS / Linux и как параллельно распространяются артефакты моделей (ONNX/GGUF). Пайплайн разделён на **две независимые рельсы**:

```
commit → CI (lint/test/contract) → build matrix (3 OS) → sign+notarize →
  → size-budget gate (≤8 MB) → channels (dev → staging → prod) → Tauri v2 updater → user

models/ → models.json registry (sha256, ram_class) → CDN (R2) →
  → first-run consent download → verify → staging FS zone → activation
```

Ключевые инварианты:

- **Инсталлятор ≤ 8 МБ** (цель 7.5 МБ) без весов моделей, без WebView2 bootstrapper (Windows) и без системного WebKitGTK (Linux). Хард-гейт в CI: сборка **падает** при превышении.
- **Модели — не часть приложения.** RapidOCR, Silero VAD, Piper, Sherpa, BGE-эмбеддер и опциональный Qwen2.5-1.5B Q4_K_M GGUF (~1080 МБ, offline-only) скачиваются при первом запуске по явному согласию, с sha256-адресацией и выбором по RAM-классу из RFC-0001 (150/655/1450 МБ).
- **Zero unsigned artifacts**: 100% инсталляторов подписаны (Developer ID + notarization, Authenticode EV / Azure Trusted Signing, GPG для Linux).
- **Reproducibility**: пины toolchain (`rust-toolchain.toml`, `uv.lock`, pinned Tauri CLI) + встроенный `archie --build-info` с git SHA и хэшем models-манифеста.

Фазовая зрелость: Alpha — ручной promotion, без staging; Beta — nightly + staged rollout 5%→25%→100%; v1.0 — автопромоция по crash-free ≥ 99.5%, delta-updates; v2.0 — MSI/org-policy, on-prem model mirror, air-gapped режим, SBOM (CycloneDX).

## 2. Goals / Non-Goals

**Goals:** воспроизводимые сборки из чистого CI-раннера; one-command release (`just release-check`); подписанные автообновляемые инсталляторы; независимое версионирование app и моделей; гейт размера с первого коммита Alpha; rollback за ≤ 15 мин.

**Non-Goals:** собственная CDN-логика (только конфигурация R2/S3); поддержка 32-bit и экзотических дистрибутивов в фазах 1–2 (только Ubuntu LTS + Fedora x64; Linux aarch64 — open question); threat model канала обновлений (док. 03); тест-кейсы (док. 07); marketing релизов (док. 06).

## 3. Repository & Toolchain Layout

Пининг — фундамент воспроизводимости. Всё, что участвует в сборке, зафиксировано в репо:

| Артефакт | Файл | Политика |
|---|---|---|
| Rust toolchain + targets | `rust-toolchain.toml` | Коммитится; обновление — отдельный PR |
| Rust deps | `Cargo.lock` (workspace) | **Всегда коммитится**; обновление через `cargo update` PR |
| Python 3.11+ env | `pyproject.toml` + `uv.lock` | Коммитится; `uv sync --frozen` в CI — гарантия идентичности sidecar-окружения |
| Frontend | `package.json` + `pnpm-lock.yaml`; Tauri CLI пин в devDependencies | Коммитится |
| Сборочные команды | `Justfile` — единая точка входа (`just ci`, `just release-check`, `just build`) | Локальная разработка и CI выполняют **буквально одни и те же команды**; `TAURI_ENV` обнаруживается там, где необходимо |

Контракты `contracts/` (`events_envelope.schema.json`, `zsense_context.schema.json`, `blueprint_sop.schema.json`, `ipc_bridge.proto`) валидируются в CI на каждой сборке (см. §4) — это механический гейт semver-политики из §10. Реестр моделей — `models-registry/models.json` (JSON Schema draft-07), версионируется git'ом и продвигается вместе с релизом.

## 4. CI Pipeline Design (GitHub Actions)

Провайдер: **GitHub Actions** (лучшая поддержка Tauri: `tauri-action` от команды Tauri делает build+sign+release+updater JSON одним job'ом; runner'ы macOS/Linux/Windows из коробки). Локальный dry-run — `act`.

### 4.1 Лейны и бюджеты времени

| Лейн | Триггер | Состав | Бюджет |
|---|---|---|---|
| **fast-lane (PR)** | `pull_request` | fmt/clippy/ruff/eslint+tsc, unit-тесты (nextest, pytest, vitest), contract-валидация. **Без packaging.** | **< 15 мин** |
| **build (main/release)** | push в `main`, `release/x.y` | fast-lane + полная build-matrix 3 OS + size-budget + sign | ≤ 25 мин warm (sccache/uv/actions-cache), ≤ 45 мин cold |
| **nightly** | cron, main @ HEAD SHA | build + подпись + SLA-smoke на reference hardware | **< 2 ч** |
| **weekly e2e** | cron, воскресенье | полный e2e-пак док. 07 на всех 3 OS + barge-in/OCR-замеры на reference runner | — |

PR-фидбек (Rust unit + clippy + audit) — ≤ 10 мин. Экономия: concurrency-группа с `cancel-in-progress: true` на superseded-коммитах; полный matrix только на `main`/`release/*`.

### 4.2 Jobs по стекам (одинаковые команды локально через Justfile)

| Job | Команда | Гейт |
|---|---|---|
| `rust-quality` | `cargo fmt --check`; `cargo clippy --workspace --all-targets -- -D warnings`; `cargo deny check` + `cargo audit` (supply chain + license) | fail on warning |
| `rust-test` | `cargo nextest run` (+ `cargo llvm-cov` покрытие) | fail on fail |
| `python-quality` | `uv sync --frozen`; `uv run ruff check`; `uv run mypy` | fail on warning |
| `python-test` | `uv run pytest -m "not hardware"` | fail on fail |
| `ts-quality` | `pnpm lint`; `pnpm tsc --noEmit` | fail on error |
| `ts-test` | `pnpm vitest run` | fail on fail |
| `contracts` | Валидация `ipc_bridge.proto` (protoc lint), JSON Schema'ы против fixture-примеров из RFC-0001; codegen TS/Rust/Python и diff сгенерированного | fail при рассинхронизации |
| `tauri-build` | `pnpm tauri build` в matrix (§5) | артефакт |
| `size-budget` | скрипт `du`/`stat` по артефакту, сравнение с таблицей §8 | **fail > 8 MB** |
| `ram-budget` (nightly) | проверка `sum(model RAM per ram_class) ≤ бюджет − app RSS` по матрице §9.4 | fail при нарушении |
| `sla-smoke` (nightly/weekly) | barge-in < 200 мс; OCR ~200 мс (185 мс эталон); RSS в 150/655/1450 МБ на reference hardware | promotability-гейт |

Кеши: `actions/cache` для `~/.cargo` + `sccache`, `uv` cache dir, `pnpm` store; macOS runner — dominant cost, поэтому кеши обязательны.

### 4.3 Reference runner для аудио/OCR

Аппаратные тесты (реальный микрофонный loopback, реальный CPU-инференс OCR 2560×1440) нельзя гонать в эфемерных контейнерах. Решение: один **self-hosted reference runner** (фиксированный ноутбук 8 ГБ / без dGPU — ровно целевой класс из RFC-0001), подключённый к GitHub Actions через self-hosted runner group. Найтли SLA-smoke (barge-in через TTS loopback, OCR-бенчмарк, 4-часовое RSS-профилирование через `ReportWorkerHealth`) выполняется только на нём. Эталонные числа — из RFC-0001; регрессия > 10% блокирует promotion.

## 5. Cross-Platform Build Matrix

| OS | Runner | Target | Форматы | Особенности |
|---|---|---|---|---|
| Windows | `windows-latest` | `x86_64-pc-windows-msvc` | NSIS (+ MSI c v2.0) | WebView2 bootstrapper (online, ~2 МБ в инсталляторе); single-instance mutex уже в `archie-shell` |
| macOS | `macos-latest` | `aarch64-apple-darwin` + `x86_64-apple-darwin` → **universal binary** | `.app` + DMG | Hardened Runtime entitlements (mic, screen, AX); Apple Vision framework в `macos_vision_prefilter.py` требует linking на macOS SDK |
| Linux | `ubuntu-latest` (контейнеры: ubuntu-lts, fedora) | `x86_64-unknown-linux-gnu` | **AppImage** (+ deb в контейнере Debian, rpm в контейнере Fedora) | WebKitGTK min-version пин; AppImage с zsync |

Кросс-компиляция Python-sidecar — главный источник боли: с macOS-runner собираем на месте для обеих архитектур (universal2 через `lipo` двух PyInstaller-выходов или сборка на native arm64 runner + Rosetta-проверка); на Windows/Linux sidecar собирается нативно. Никогда не кросс-компилируем Python с Linux на macOS (известно-хрупкий путь).

**Дисциплина застройки против дистро-дрейфа:** Linux matrix пинится к двум подтверждённым WebKitGTK-окружениям (Ubuntu LTS, Fedora — включая актуальный Fedora Workstation как reference-дистро пользователя). deb/rpm собираются в соответствующих контейнерах, GPG-подпись metadata apt/yum-репозитория.

## 6. Code Signing & Notarization

### 6.1 Цепочка подписей и кастодия ключей

| Ключ | Назначение | Хранение | Доступ | Ротация |
|---|---|---|---|---|
| Apple Developer ID Application | Подпись `.app` | Apple Keychain на macOS release-машине / CI secret | CI job `sign-macos` (GitHub Environment `signing`, required reviewer) | Годовая, за 30 дней |
| Windows EV cert **или** Azure Trusted Signing | Authenticode PE | HSM-токен EV / Azure KMS | CI job `sign-windows` (тот же Environment) | EV 1–3 года; Trusted Signing — managed |
| **Updater ed25519 (minisign)** | Подпись update-манифестов Tauri v2 | **Offline**: зашифрованный cold storage + бумажная копия; подпись на controlled release-машине | Один-два человека, никогда не в CI как plain secret | Документированный runbook, dual-key overlap window |
| GPG (Linux) | Подпись checksums + repo metadata | 1Password / SOPS+age | Release-инженер | 2 года |

Критическое правило: **потеря updater-ключа = все установленные копии навсегда теряют автообновление** — поэтому offline-бэкап обязателен до первой публичной Beta.

### 6.2 macOS

`notarytool` (не deprecated altool): submit → wait → `stapler staple`. Entitlements-набор под особенности Archie — **в release checklist**, потому что Apple отклоняет notarization при отсутствии usage-description строк:

- **Микрофон** (always-listening audio loop + Silero VAD) — `NSMicrophoneUsageDescription`;
- **Запись экрана** (Z-Sense захват кадров в Shared Memory) — `NSScreenCaptureUsageDescription` (TCC);
- **Accessibility (AX API)** — чтение `accessibility_tree.rs` через Accessibility API; **дизайн промптов TCC**: первый запуск просит разрешения поэтапно — сначала AX (для Z-Sense), затем mic (для аудио-тракта), screen capture последним; каждый промпт предварён объяснением в onboarding wizard (док. 02) с отложенной опцией «Спросить позже» (graceful degradation: без mic работает текстовый режим, без AX — только RapidOCR-путь);
- **Apple Vision framework** (`VNRecognizeTextRequest`) — входит в entitlements сборки universal binary; hardened runtime: sidecar Python должен проходить `otool -hv`-проверку в sign-стадии (отсутствие JIT-нарушений, все dylib подписаны).

Канареечный notarization-job гоняется на **каждой** release-ветке с Alpha — первая попытка подписи не должна случиться в неделю публичной Beta.

### 6.3 Windows

SmartScreen для нового reputation-less издателя = главный дистрибутивный барьер. Решение на выбор (закрыть на старте Alpha, budget в Project Plan):

- **Authenticode EV** (~$300–500/год, орг. верификация 1–3 недели) — мгновенная репутация;
- **Azure Trusted Signing** (pay-per-use, быстрее onboarding, но ограничения идентификации и доступности — verify at decision time).

Signing в CI через `signtool` / Trusted Signing action, timestamp-сервер обязателен (иначе подпись умирает с сертификатом).

### 6.4 Linux

Универсальной подписи нет: AppImage распространяем с sha256-checksums, подписанными GPG; deb/rpm — через GPG-подписанные repo metadata. Endpoint'ы — TLS + hash-pinned в конфиге приложения (защита от MITM-swap AppImage).

## 7. Auto-Update (Tauri v2 Updater)

Plugin updater с **ed25519 (minisign)** подписью. Endpoint — статический JSON на CDN:

```json
{
  "version": "0.4.2",
  "notes": "Смотри CHANGELOG.md",
  "pub_date": "2026-06-01T10:00:00Z",
  "platforms": {
    "darwin-aarch64": { "signature": "...", "url": "https://cdn.../Archie_0.4.2_aarch64.app.tar.gz" },
    "darwin-x86_64":  { "signature": "...", "url": "..." },
    "linux-x86_64":   { "signature": "...", "url": "https://cdn.../Archie_0.4.2_amd64.AppImage" },
    "windows-x86_64": { "signature": "...", "url": "https://cdn.../Archie_0.4.2_x64-setup.exe" }
  }
}
```

Файлы каналов: `archie-stable.json`, `archie-beta.json`, `archie-nightly.json` (+ отдельный `archie-stable-rollout.json` с полем `rollout_percent`).

**Поведение:** check с таймаутом ≤ 5 с; фоновая закачка; **install on next restart по умолчанию** — приложение не должно прерывать сессию (модель фрустрации из RFC-0001: прерванная сессия = фрустрация); prompt через soft widget только если перезапуск нужен mid-session. Обновление никогда не убивает soft widget mid-voice-ответ: шина получает envelope `SHUTDOWN` с reason `UPDATE_INSTALL`, audio-цикл корректно флашит буферы.

**Version floor:** updater отказывается ставить версию, у которой IPC envelope / SOP blueprint schema **ниже** персистентной локальной схемы данных (§10) — защита от «bricked by update». Поле `force_version` + `max_check_interval` в канальном JSON — kill-switch для оффлайн-редких пользователей.

**Staged rollout (stable):** 5% → 25% → 100%, ≥ 24 ч между стадиями, гейт — crash-free-sessions ≥ 99.5% (метрики док. 10, фаза 2+) и 0 P0 в первые 24 ч на 5%. Откат канала — редактирование JSON, эффект ≤ 15 мин.

**Delta updates:** AppImage — zsync; Windows/macOS — bsdiff от Tauri. ROI при 8 МБ payload спорный (open question) — внедрение в v1.0, приоритет для metered/weak-hardware юзеров.

## 8. Installer Size Budget (~8 MB)

Хард-гейт `size-budget` job с первого коммита Alpha (дисциплину невозможно ретрофитить после Beta):

| Компонент | Бюджет |
|---|---|
| Tauri shell + window glue | ~1.5 МБ |
| Rust core (event bus, os-hooks, stagingfs) | ~2.5–3 МБ |
| Frontend assets (двухоконный UI, dual-window) | ~0.5–1 МБ |
| Python sidecar (замороженные deps, strict diet) | ~1.5–2 МБ |
| Локали, иконки, misc | ~0.2 МБ |
| Headroom | ~1 МБ |
| **TOTAL** | **≤ 8 МБ** (target 7.5) |

**Не включено:** ONNX/GGUF-модели (§9), WebView2 bootstrapper, системный WebKitGTK.

Главный риск — Python sidecar: PyInstaller onefile несёт 3–5 МБ интерпретатора+stdlib **до** первой зависимости. Митигации (решение — open question §15): zipapp с вычищенными locale/test-файлами, `cargo opt-level=z` + `strip` + zstd-упаковка, hot-paths в maturin-расширениях вместо тяжёлых Python-deps. Замер — CI-скрипт (`du -sb` артефакта + проверка strip), еженедельный size-trend в release-метриках.

## 9. Model Artifact Distribution

### 9.1 Принцип

Модели — «расширения» (паттерн VS Code / RustDesk / Ollama): sha256-адресованные, независимо обновляемые, сменяемые per hardware class. CDN — **Cloudflare R2** (zero egress — критично для опционального GGUF ~1 ГБ) + fallback mirror.

### 9.2 Реестр `models.json`

```json
{
  "manifest_version": "1.2.0",
  "models": [
    {
      "id": "rapidocr-ppocrv5",
      "files": [
        { "name": "det_v5.onnx", "sha256": "…", "bytes": 4700000 },
        { "name": "rec_v5.onnx", "sha256": "…", "bytes": 10000000 }
      ],
      "format": "onnx", "quant": "int8",
      "ram_class": ["low", "mid"], "min_app": "0.3.0",
      "license": "Apache-2.0 (PaddleOCR)", "source_url": "…"
    },
    { "id": "silero-vad", "format": "onnx", "quant": "fp16",
      "files": [{ "name": "silero_vad.onnx", "sha256": "…", "bytes": 1800000 }],
      "ram_class": ["low","mid","high"], "min_app": "0.3.0", "license": "MIT" },
    { "id": "piper-tts-ru-medium", "format": "onnx", "ram_class": ["mid","high"], "min_app": "0.3.0", "license": "…" },
    { "id": "sherpa-kroko-int8", "format": "onnx", "quant": "int8", "ram_class": ["mid","high"], "min_app": "0.3.0", "license": "…" },
    { "id": "bge-micro-embedder", "format": "onnx", "quant": "int8", "ram_class": ["low","mid","high"], "min_app": "0.3.0" },
    { "id": "ner-pii-int8", "format": "onnx", "quant": "int8", "ram_class": ["low","mid","high"], "min_app": "0.3.0" },
    {
      "id": "qwen2.5-1.5b-instruct",
      "format": "gguf", "quant": "Q4_K_M",
      "files": [{ "name": "qwen2.5_1.5b_q4_k_m.gguf", "sha256": "…", "bytes": 1080000000 }],
      "ram_class": ["high"], "min_app": "0.3.0",
      "optional": true, "offline_only": true, "consent_required": true,
      "license": "Apache-2.0 (Qwen)", "source_url": "…"
    }
  ]
}
```

Каждая запись: `sha256`, `bytes`, `ram_class`, `min_app`, `license`+`attribution`. **Ни одна модель не входит в реестр без license-ревью** (allow-list; GGUF с ограничительными лицензиями — только bring-your-own импорт, с v1.0: локальная папка + верификация по локальному manifest).

### 9.3 First-run state machine (Rust downloader, reqwest, Range-resume)

```
IDLE → CONSENT(показ размера, RAM-класс, кнопка) → CHECKING(fetch manifest+signature)
  → DOWNLOADING(Range resume, прогресс на soft widget) → VERIFYING(sha256)
  → STAGING(запись в staging FS zone) → ACTIVATED(atomic rename)
  ошибки: RETRY(backoff) / LOW_DISK(abord с явным сообщением) / ABORT
```

Правила: низкий RAM-класс (low) получает минимальный набор ~200–400 МБ (на 100 Mbps ≤ 60 с, resumable); GGUF 1080 МБ — **отдельное явное согласие** (offline-only, не обязателен для работы; на metered-соединении не предлагается автоматически). Ошибка загрузки **никогда не блокирует** старт app shell — модели lazy per Z-Sense/audio need (RFC-0001). Локальный store — content-addressed (filename = sha256), разные версии приложения делят одни веса; rollback модели = pin предыдущего хэша. CI-job перед promotion канала: fetch всех файлов реестра с CDN + ре-хэш (registry consistency gate).

### 9.4 RAM-класс матрица (консистентность с RFC-0001 по построению)

| ram_class | Суммарный RAM-бюджет (RFC-0001) | Набор моделей |
|---|---|---|
| low | Idle 150 МБ / Peak ограничен | Silero VAD + RapidOCR int8 + NER int8 + BGE micro; STT/TTS минимальные квантования; **без** GGUF |
| mid | Peak 655 МБ | + Piper medium + Sherpa int8 полный |
| high | Offline 1450 МБ | + Qwen 1.5B Q4_K_M (mmap, вытесняет RapidOCR/NER) |

CI `ram-budget`-job валидирует `sum(model RAM) ≤ budget − app RSS` — спецификация остаётся консистентной с RAM SLA механически, а не «по надежде».

## 10. Semver Policy

Пять независимых осей версий; правила bump'а:

| Ось | Major (breaking) | Minor | Patch |
|---|---|---|---|
| **App** (`x.y.z`) | — | новая фича | фикс |
| **IPC event envelope** (`contracts/events_envelope.schema.json`, `ipc_bridge.proto`) | удаление/сужение поля, enum-значение, изменение семантики priority | новое payload-поле, новый EventType | — |
| **SOP blueprint schema** | удаление поля рецепта, изменение semantics execution_steps | новое опциональное поле, новый action_category | — |
| **Memory SQL schema** (миграции `archie_core.sqlite3`) | необратимая миграция данных | аддитивная таблица/колонка + **обязательный** down-migration + replay-тест в PR | фикс миграции |
| **Models manifest** | смена структуры models.json | новая модель/квантование | фикс sha256/метадаты |

**Правило проекта:** изменение публичного контракта в `contracts/` → **major**; аддитивное событие/протокол → **minor**. Совместимость задаётся data-driven матрицей в репо (`compat.json`: app version → min envelope / min SOP / min manifest); updater читает её, а не хардкодит.

Миграции SQLite: forward-only в runtime; каждый PR с миграцией несёт down-migration (для dev/rollback tooling) и fixture-replay; updater отказывается от апгрейда при version floor-нарушении; staging-snapshot БД перед миграцией на первом запуске после update (staging-зоны RFC-0001 = атомарная точка откката).

## 11. Release Branching & Cadence

**Trunk-based**: `main` — единственная ветка разработки; короткоживущие `release/x.y` — только стабилизация (cherry-pick, forbidden features). Nightly — из `main` автотегом SHA (tag-less, SHA-именованные артефакты); beta — RC-теги с `release/*`; stable — теги `vX.Y.Z`; hotfix — cherry-pick в последнюю release-ветку → патч-тег.

Календарь: Alpha — еженедельные теги; Beta — bi-weekly RC; v1.0+ — monthly minor, критический патч ≤ 72 ч; kill-switch канала ≤ 4 ч. Changelog — **Conventional Commits → release-please**: автогенерация CHANGELOG.md, semver-bump'ов и `notes`-поля updater JSON (малые аттрибутируемые дельты — предусловие staged rollout).

## 12. Rollback Strategy (три слоя)

| Симптом | Детекция | Действие | SLA |
|---|---|---|---|
| Crash-loop после апдейта | crash reporter (док. 10) | `force_version` в канальном JSON на предыдущую подписанную версию | ≤ 4 h stable |
| Сломанная модель | health-метрики, registry-жалобы | pin предыдущего sha256 в models.json (content-addressed store делает это тривиальным) | ≤ 4 h |
| Bad SOP/SQL миграция | отказ старта, аудит | migration down + restore staging-snapshot БД; updater floor не даёт зайти в dead-end | ≤ 24 h |
| Updater «bricked» | — | `disable_updates` flag канала; ручная переустановка поверх | runbook |

Канальный kill-switch (`force_version` / `paused: true`) действует ≤ 15 мин от правки JSON. Ежеквартальный game-day: фактический откат релиза на beta-канале — непроверенный rollback не является стратегией.

## 13. Release Checklist (шаблон, RACI)

**Code gates (авто, CI-checked):** clippy/audit/deny clean · ruff/mypy clean · lint+tsc+vitest green · contracts валидированы · SLA-smoke: barge-in < 200 мс, OCR ~200 мс, RSS 150/655/1450 МБ per class — регрессия > 10% блокирует · size-budget ≤ 8 МБ · ram-budget матрица green · models.json: все sha256 re-verified против CDN.

**Signing (Releaser):** macOS notarized + stapled, entitlements (mic/screen/AX usage strings) сверены · Windows Authenticode с timestamp · AppImage checksums GPG-подписаны · updater-манифесты подписаны offline-ключом.

**Channels & data (Releaser + Reviewer):** version floor / compat.json обновлён · миграции несут down-scripts · consent-копия (EULA/permissions) version bumped — юридический текст не регрессирует молча · CHANGELOG сгенерирован.

**Comms:** анонс (Telegram/release notes) · status page канала обновлён.

**Post-release smoke (QA):** свежеустановленный signed build на чистой Win/mac/Fedora машине · first-run model download flow · updater с прошлой версии.

## 14. CI Cost, Runners, Secrets

Матрица 3 OS × PR — главная дыра бюджета (macOS ≈ 10× Linux). Контроль: fast-lane без packaging на PR, полный matrix только main/release, cancel-in-progress, self-hosted Linux для nightly, кеши. Оценка Alpha: **≤ $200/мес** (доминируют ~40–60 macOS build-minutes на release cycle × еженедельный cadence), ревью ежеквартально.

Secrets: GitHub Environments (`signing`) с OIDC + required reviewers — второй человек подтверждает release-signing job (вопрос второго releaser'а — в Project Plan); updater-ключ — offline; dev-секреты — 1Password CLI (`op run`) / SOPS+age. Build provenance: `BUILD_INFO` (git SHA, toolchain'ы, channel, hash манифеста моделей) встраивается в бинарник, запрашивается `archie --build-info`; CycloneDX SBOM (`cargo cyclonedx` + `uv export`) на каждый release-build с фазы Beta — дешёвый задел на SOC 2 / EU CRA / enterprise procurement v2.0.

## 15. Open Questions

1. Python sidecar: PyInstaller onefile vs zipapp vs maturin-переписывание hot paths (размер vs RAM vs velocity; бюджет 8 МБ — ограничитель).
2. Windows: EV-сертификат vs Azure Trusted Signing (решение при оформлении юрлица/бюджета; lead time 2 недели в плане).
3. Linux: Ubuntu LTS + Fedora только для Beta? Нужен ли aarch64 Linux до v1.0?
4. Model CDN: R2 vs S3+CloudFront vs спонсорский grant; RU/EU-геодоступность и платёжные ограничения.
5. Delta updates ROI при ≤ 8 МБ payload (metered-сегмент) — отложить в v1.0?
6. Nightly-канал в Alpha (добавляет signing-surface) или ручные download'ы до Beta?
7. GGUF в first-party реестре vs bring-your-own (license + RAM).
8. Update UX: анонс голосом Archie через soft widget (on-brand) vs нативный промпт — co-ownership с Voice UX спекой.

---

*Числа SLA (barge-in < 200 мс, OCR ~200 мс, RAM 150/655/1450 МБ, инсталлятор ~8 МБ) — канон RFC-0001 и синхронизированы с ним. Регуляторные заметки (EU CRA, GDPR, 152-FZ) — инженерные ориентиры, не юридическая консультация: legal review до B2C Beta.*
