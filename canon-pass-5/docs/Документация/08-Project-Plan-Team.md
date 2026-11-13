# 08 — План проекта и структура команды

**Проект:** Archie («Архи») — автономный проактивный ИИ-напарник уровня ОС (OS-Level AI Symbiote)
**Документ:** Project Plan & Team Structure
**Версия:** 0.1.0 (MVP Alpha baseline) · **Статус:** согласовывается с основателем
**Связанные документы:** `01-PRD.md` (scope), `07-Testing-QA-Strategy.md` (тестовые гейты), `09-CI-CD-Distribution.md` (сборка/подписи/обновления), `10-Observability-Telemetry-Privacy.md` (метрики), `05-Business-Model-Financial-Plan.md` (бюджет/headcount)

---

## 1. Резюме

Стратегия поставки: **модель фазовых гейтов поверх Scrum**. Четыре фазы дорожной карты (MVP Alpha → B2C Beta → v1.0 Production → v2.0 Enterprise) оформлены как гейты с измеримыми критериями выхода, взятыми из RFC-0001 и матрицы приёмки архитектурного документа. Внутри фаз команда работает двухнедельными спринтами; ML-исследования выносятся в отдельную «спайк-дорожку» с таймбоксами и критериями остановки (kill criteria), чтобы не загрязнять данные по velocity.

Команда: **малая команда из 5 человек** (основатель-архитектор, 2 Rust-инженера, 1 Python ML-инженер, 1 фронтенд-инженер TS, 1 product/QA), растущая по фазовым триггерам до ~8–10 FTE к v1.0 и ~12 к v2.0. Один флагманский критерий на фазу:

- **Phase 1 (MVP Alpha):** Barge-in — полная остановка TTS по `FORCE_STOP_AUDIO` **< 200 мс** на живом железе.
- **Phase 2 (B2C Beta):** RAM-бюджет **655 МБ Peak** зелёный в CI + рабочий цикл проактивности (порог фрустрации **50**, окно отмены **3–5 с**).
- **Phase 3 (v1.0):** Semantic Router обходит облачную LLM при косинусном совпадении **≥ 0.98**, SOP исполняется в песочнице, автообновление подтверждено staged rollout.
- **Phase 4 (v2.0):** Enterprise-пилот в закрытом контуре (On-Premise), PII-санитайзер **100% маскирование** `[REDACTED_*]`.

Топ-3 риска поставки: регрессии задержки аудио-тракта (Barge-in), качество OCR на русскоязычном тексте UI, и расползание скоупа (scope creep) при демонстрациях инвесторам.

---

## 2. Модель поставки и фазовых гейтов

**Почему гибрид, а не чистый Scrum.** Продукт — realtime-система (голосовой цикл < 200 мс) с ML-компонентами (OCR, VAD, frustration-модель, эмбеддинги). Качество модели и латентность не раскладываются в story points: спринт может быть «зелёным» по бёрндам и провальным по SLA. Поэтому:

1. **Фазовые гейты (PRINCE2-style stage boundaries):** критерии выхода фазы фиксируются в разделе 8 **до старта фазы** и не пере договариваются на выходе. Красный критерий блокирует выход; обход возможен только письменным ваивером за подписью основателя (счёт ваиверов — отчётная метрика, цель 0–1 на фазу).
2. **2-недельные спринты** для потока разработки (детали — раздел 7).
3. **Спайк-дорожка (Shape Up-таймбоксы)** для ML-задач: фиксированный аппетит (≤ 1 спринта), kill criteria, письменный итог (ADR или декомпозиция задач). Velocity спринтов не учитывает спайки.
4. **Демо на живом железе каждый спринт-ревью:** живой Barge-in и OCR на реальном экране докладчика — это регрессионный тест, который команда реально почувствует. Записанное видео не считается демо.

**Правило изменения скоупа** (наследуется из PRD): новая функциональность вытесняет равную по приоритету (MoSCoW per phase + «will NOT build» списки); Definition of Ready (раздел 7.4) не даёт неоформленным требованиям попасть в спринт.

**Календарь фаз (оценки, не обязательства; перевалидируются по velocity спринтов 1–3):**

| Фаза | Длительность | Команда (FTE-экв.) | Объём (чел-недель) |
|---|---|---|---|
| Sprint 0 (bootstrap) | 2 недели | 5 | 8 |
| Phase 1 — MVP Alpha | ~4–5 мес (8–10 спринтов) | 5 | ~80 |
| Phase 2 — B2C Beta | ~4 мес (8 спринтов) | 6–7 | ~95 |
| Phase 3 — v1.0 Production | ~5–6 мес (10–12 спринтов) | 8–10 | ~120 |
| Phase 4 — v2.0 Enterprise | ~4–6 мес (8–12 спринтов) | ~12 | ~110 |

---

## 3. Декомпозиция работ (WBS) по фазам

Формат строки: **ID | модуль монорепо | контракт RFC-0001 | владелец | чел-нед. | зависимости**. Оценки — person-weeks (календарных, включая тесты и документацию, см. DoD в разделе 7.5). Роли: TL (Tech Lead / Rust #1), R2 (Rust #2), ML (Python ML #1), FE (Frontend TS), PQ (Product/QA), F (Founder-архитектор).

### 3.1 Sprint 0 — Bootstrap (WB-0, 8 чел-нед., 1 спринт)

Скам AFFOLD: всё, что вешается на стабильные внутренние контракты.

| ID | Модуль | Контракт | Владелец | н/нед | Зависимости |
|---|---|---|---|---|---|
| WB-0.1 | Каркас монорепо: `crates/` (cargo workspace), `services/archie-engine/` (uv), `apps/desktop-ui/` (Tauri v2) | — | TL | 2 | — |
| WB-0.2 | CI-скелет: cargo test/clippy, pytest, статический линтер схем (передано в `09-CI-CD-Distribution.md`) | — | TL+ML | 2 | WB-0.1 |
| WB-0.3 | `contracts/events_envelope.schema.json` v0 + `ipc_bridge.proto` (UDS 0600) — заморозка первых контрактов | event envelope, IPC proto | TL | 2 | WB-0.1 |
| WB-0.4 | Измерительный стенд: audio loopback rig (заглушка) + golden-image датасет OCR + acceptance-сет из 10 SOP | SLA-тесты RFC-0001 | ML | 1 | — |
| WB-0.5 | SQL-схема `archie_core.sqlite3` v0 (миграции) | SQL DDL | R2 | 1 | WB-0.3 |

### 3.2 Phase 1 — MVP Alpha (~80 чел-нед., 8–10 спринтов)

Состав фазы по архитектурной дорожной карте: event bus, Tauri shell, cloud gateway + graceful degradation, Z-Sense OCR pipeline, полнодуплексный аудио-цикл, промпты персоны.

| ID | Workstream | Модуль монорепо | Контракт RFC-0001 | Владелец | н/нед | Зависимости |
|---|---|---|---|---|---|---|
| WB-1 | Модульная Шина Событий | `crates/archie-event-bus/` (`broker.rs`, `router.rs`, `graceful_degradation.rs`) | event envelope; события `APP_OPENED`, `API_TIMEOUT`, `FORCE_STOP_AUDIO`, `ZSENSE_GRAPH_READY`, `FRUSTRATION_SPIKE`, `BLUEPRINT_MATCHED_98` | TL | 10 | WB-0.3 |
| WB-2 | Tauri-оболочка: двухоконная схема, трей, single-instance, HiDPI, Win+D immunity, глобальные хоткеи (через штатный плагин Tauri, **не** rdev — см. риск R5) | `crates/archie-shell/` (`window_manager.rs`, `tray_controller.rs`, `single_instance.rs`, `hidpi_scaler.rs`, `win32_hittest.rs`) | инсталлятор ~8 МБ | FE (Rust-часть — TL) | 12 | WB-0.1 |
| WB-3 | OS-хуки: активное окно, дерево доступности, буфер обмена, input heuristics | `crates/archie-os-hooks/` | event envelope | R2 | 8 | WB-1 |
| WB-4 | Cloud gateway + graceful degradation: адаптеры DeepSeek/OpenAI/Gemini, офлайн-фолбэк на локальную SLM 1.5B (llama.cpp / ONNX CPU) | `services/.../llm_clients/` (`cloud_gateway.py`, `local_1_5b_fallback.py`) | реакция на `API_TIMEOUT` (без потери голоса и локальных функций) | ML | 8 | WB-1 |
| WB-5 | Z-Sense OCR pipeline: триггерный захват, RapidOCR (PP-OCRv5, ONNX Runtime), XY-cut сортировка, JSON-граф, префильтр Apple Vision на macOS | `services/.../zsense/` (`ocr_pipeline.py`, `rapidocr_runner.py`, `xy_cut_sorter.py`, `json_graph_builder.py`) | OCR ~200 мс (эталон 185 мс) на CPU, пакет ~80 МБ, экономия трафика 99% | ML | 14 | WB-0.4, WB-3 |
| WB-6 | Полнодуплексный аудио-цикл: единый intra-process loop, WebRTC AEC3 (livekit), Silero VAD, sigh detector, endpointing, Sherpa-ONNX STT, Piper TTS, трёхуровневый каскад прерывания | `services/.../audio_loop/` (`full_duplex_engine.py`, `webrtc_aec_processor.py`, `silero_vad_gate.py`, `sherpa_stt_stream.py`, `piper_tts_stream.py`) | Barge-in < 200 мс; VAD 85–100 мс, точность > 95%; единый аудио-клок (один процесс!) | ML (TL — ревью latency-пути) | 16 | WB-1 |
| WB-7 | Промпты персоны «Старший брат / Ментор» + стриминговая генерация с буферизацией до знака препинания | `services/.../llm_clients/persona_prompts.py` | — | F+PQ | 3 | WB-4 |
| WB-8 | Интеграция, сквозные SLA-тесты (loopback rig, golden set), документация, догфудинг-цикл | монорепо в целом | все SLA фазы | PQ | 9 | WB-1..7 |

**Итого Phase 1: ~80 чел-недель.** Ключевой принцип секвенирования: два самых рискованных SLA (WB-6 аудио и WB-5 OCR) стартуют в **спринте 1**, а не в конце фазы.

### 3.3 Phase 2 — B2C Beta (~95 чел-нед., 8 спринтов)

Состав: mission state machine, frustration scorer, soft widget, онбординг (Путь А/Б), трёхуровневая память, MCP Obsidian/Notion, permission tiers + staging FS.

| ID | Workstream | Модуль | Контракт RFC-0001 | Владелец | н/нед | Зависимости |
|---|---|---|---|---|---|---|
| WB-9 | Машина состояний миссии (GAMING_REST / STUDY_FOCUS / CODING_WORK / IDLE) | `services/.../proactivity/mission_state_machine.py` | SQL DDL `user_profile.current_mission_state` | ML | 6 | WB-3 |
| WB-10 | Frustration scorer: эвристики (+10/+15/+20/+30), полураспад, медиа-фриз (видеолекция не считается стагнацией) | `proactivity/frustration_scorer.py` | порог 50, S_max=100, полураспад 20/15/5/1 мин по миссиям | ML | 8 | WB-9, WB-6 (sigh detector) |
| WB-11 | Soft Widget: пульсирующий индикатор, окно отмены, resize-on-activation для карточек | `apps/desktop-ui/src/windows/soft_widget/` | окно отмены 3–5 с; 0% срабатываний в GAMING_REST | FE | 8 | WB-2 |
| WB-12 | Онбординг: OnboardingWizard (Путь А — интервью, Путь Б — тихое наблюдение) | `main_dashboard/OnboardingWizard.tsx` | `onboarding_mode IN (INTERACTIVE_INTERVIEW, SILENT_OBSERVATION)` | FE+F | 10 | WB-2 |
| WB-13 | Трёхуровневая память: Tier 1 (RAM 2–4 ч), Tier 2 (ночные сводки), Tier 3 (векторный граф, LanceDB) | `services/.../memory/` (`tier1_ram_session.py`, `tier2_nightly_summarizer.py`, `tier3_vector_graph.py`, `sqlite_repository.py`) | сессия 2–4 ч; ночная .md-сводка; RAM-бюджет | ML | 14 | WB-5, WB-0.5 |
| WB-14 | MCP-хаб + серверы Obsidian/Notion (FastMCP, stdio/SSE/Streamable HTTP) | `services/.../mcp_hub/` (`obsidian_vault_server.py`, `notion_sync_server.py`) | 100% ночная генерация .md в Obsidian/Notion | R2 | 10 | WB-13 |
| WB-15 | Permission Tiers (1/2/3) + staging FS: физический запрет Delete, перенос в `Archie_Staging`, карточки Tier 2 Approval | `crates/archie-staging-fs/` + `approval_modal/Tier2ApprovalCard.tsx` | запрет системного вызова Delete; 100% файлов → Staging | TL (R2 deputy) | 12 | WB-2, WB-14 |
| WB-16 | QA: acceptance-тесты по матрице фазы, телеметрия (opt-in), бета-инфраструктура, документация | — | RAM 655 МБ Peak (CI-гейт) | PQ | 17 | все |

**Итого Phase 2: ~95 чел-недель.**

### 3.4 Phase 3 — v1.0 Production (~120 чел-нед., 10–12 спринтов)

Состав: semantic router, blueprint generator, SOP sandbox, blueprint manager UI + релизная готовность (передано в `09-CI-CD-Distribution.md`).

| ID | Workstream | Модуль | Контракт RFC-0001 | Владелец | н/нед | Зависимости |
|---|---|---|---|---|---|---|
| WB-17 | Semantic Router: перехват событий до LLM, векторный поиск триггера | `router_and_sops/semantic_router.py` | обход LLM при косинусном совпадении ≥ 0.98 | ML | 10 | WB-13 |
| WB-18 | Blueprint Generator: инференс повторяющихся паттернов → черновик правила → [Утвердить]/[Изменить] | `router_and_sops/blueprint_generator.py` + `approval_modal/BlueprintDiffViewer.tsx` | `smart_blueprints_sop` DDL; similarity_threshold 0.98 | ML+FE | 12 | WB-17, WB-15 |
| WB-19 | SOP Sandbox: компиляция шагов черновика только в вызовы типизированных MCP-инструментов и Tier 1–3 API; AST-валидация (никаких системных вызовов из рецепта) | `router_and_sops/sop_sandbox_runner.py` | изоляция исполнения, карантин `Archie_Staging` | TL (ML deputy) | 14 | WB-18, WB-14 |
| WB-20 | Blueprint Manager UI: редактор рецептов, история, конфликт-детектор шорткатов | `main_dashboard/BlueprintManager.tsx`, `ShortcutsConfig.tsx` | — | FE | 10 | WB-18 |
| WB-21 | Релизная готовность: подписи/нотариализация, автообновление, дистрибуция моделей (ONNX/GGUF), staged rollout | см. `09-CI-CD-Distribution.md` | RAM 1450 МБ Offline; инсталлятор ~8 МБ | R2 (+release eng с найма) | 14 | WB-0.2 |
| WB-22 | Качество: WCAG 2.2 AA, мультимонитор, HiDPI-матрица, нагрузочное тестирование, документация | — | все SLA v1.0 | PQ+все | 16 | все |

**Итого Phase 3: ~120 чел-недель.**

### 3.5 Phase 4 — v2.0 Enterprise (~110 чел-нед., 8–12 спринтов)

| ID | Workstream | Модуль | Контракт | Владелец | н/нед | Зависимости |
|---|---|---|---|---|---|---|
| WB-23 | PII NER hardening: adversarial-сеты, квантованная ONNX NER, теги `[REDACTED_*]` | `security/` (`pii_sanitizer.py`, `ner_onnx_masker.py`, `regex_rules.py`) | 100% маскирование ключей/карт/проприетарного кода | ML (+security eng) | 12 | WB-4 |
| WB-24 | Корпоративные коннекторы Jira/Slack + режим Junior Mentor | `mcp_hub/servers/enterprise_jira_slack.py` | MCP-транспорты | R2 (solutions eng) | 12 | WB-14, WB-19 |
| WB-25 | Burnout Tracker: агрегация метрик фрустрации в анонимные тепловые карты для HR | расширение `proactivity/` + `action_audit_log` | анонимизация | ML | 8 | WB-10 |
| WB-26 | On-Premise: замена cloud gateway на коннектор к локальным серверам компании, SSO/admin console, audit log integrity | `llm_clients/` + админ-консоль | работа в закрытом контуре без внешнего интернета | TL | 12 | WB-1, WB-23 |
| WB-27 | SOC 2 readiness, политики доступа, пилотный онбординг enterprise-клиента, документация | процесс | — | F+PQ | 12 | все |

**Итого Phase 4: ~110 чел-недель.**

### 3.6 ASCII-Gantt (спринты по 2 недели; M — месяц)

```
            M1  M2  M3  M4 | M5  M6  M7  M8 | M9  M10 M11 M12 M13 M14 | M15 M16 M17 M18
Sprint 0    ██                  |
Ph1 WB-1    ████████            |
Ph1 WB-2    ██████████          |
Ph1 WB-3     ██████             |
Ph1 WB-4       ████████         |
Ph1 WB-5       ██████████████   |      ← OCR стартует в спринте 1–2
Ph1 WB-6     ██████████████████ |      ← аудио-цикл стартует в спринте 1
Ph1 WB-7             ███        |
Ph1 WB-8               ████████ |
GATE-1 ◄────────── ALPHA EXIT ──┤
Ph2 WB-9..12                     ████████
Ph2 WB-13..15                     ████████████
Ph2 WB-16                           ████████
GATE-2                            ◄── BETA EXIT
Ph3 WB-17..20                                ████████████
Ph3 WB-21..22                                    ████████████
GATE-3                                          ◄── v1.0 EXIT
Ph4 WB-23..27                                                  ████████████
GATE-4                                                        ◄── v2.0 EXIT
```

Критический путь: **WB-0 → WB-1 (event bus) → WB-6 (audio loop) → WB-10 (frustration) → WB-17 (router) → WB-19 (sandbox) → WB-26 (on-prem)**. Всё, что слева от гейта, должно быть закрыто: красный критерий блокирует переход фазы.

---

## 4. Состав команды и карта владения

**Топология (Team Topologies):** одна stream-aligned-команда с «платформенными» компонентами внутри (event bus, MCP-хуб, память). Разделение на отдельные ML/infra/frontend-команды на этом размере запрещено — стоимость хендофов превышает выгоду специализации до ~15 человек.

| Роль | Фаза 1 | Фаза 2 | Фаза 3 | Фаза 4 | Зона ответственности |
|---|---|---|---|---|---|
| Founder / CEO (архитектор) | ✔ | ✔ | ✔ | ✔ | Продукт, SOP-курация, GTM, найм, финальная архитектурная экспертиза |
| Tech Lead / Senior Rust #1 | ✔ | ✔ | ✔ | ✔ | `archie-event-bus`, IPC, `archie-shell` (Rust-часть), `archie-staging-fs`, ревью, архитектура |
| Rust Engineer #2 | ✔ | ✔ | ✔ | ✔ | `archie-os-hooks`, MCP-хаб, телеметрия, SQL-ядро; deputy по event bus |
| Python ML Engineer #1 | ✔ | ✔ | ✔ | ✔ | Z-Sense OCR, аудио-цикл, frustration scorer, память, эмбеддинги роутера, PII |
| Frontend Engineer (TS) | ✔ | ✔ | ✔ | ✔ | Tauri v2 dual-window, soft widget, approval cards, dashboard, онбординг |
| Product / QA | ✔ (part) | ✔ | ✔ | ✔ | Acceptance-тесты по матрице PRD, golden-сеты, телеметрия KPI, бета-поддержка |
| QA / Designer (контракт) | — | ✔ (part) | — | — | Контрактный дизайнер виджета; QA-автоматизация |
| Release/DevOps Engineer | — | — | ✔ | ✔ | Подписи, нотариализация, CDN моделей, канал автообновления |
| ML Engineer #2 | — | — | ✔ | ✔ | Качество SOP-матчинга, квантизация ONNX/GGUF |
| Frontend #2 | — | — | ✔ | ✔ | Кроссплатформенность, WCAG 2.2 AA |
| Security/Compliance Engineer | — | — | — | ✔ | SOC 2, audit log, threat modeling |
| Solutions Engineer (enterprise) | — | — | — | ✔ | Пилоты, Jira/Slack-интеграции |

**Владение бюджетами SLA (роли принуждения, не «команда»):**

| Бюджет | Владелец | Deputy |
|---|---|---|
| Barge-in < 200 мс (latency-путь) | Tech Lead | ротация: зам проводит месячный прогон на живом железе |
| RAM 150 / 655 / 1450 МБ (WorkerHealthMetrics в CI) | Rust #2 (с Beta) | Tech Lead |
| Качество SOP-матчинга (косинус ≥ 0.98, acceptance-сет) | ML #1 | ML #2 с v1.0 |
| PII egress — ноль сырых кадров/аудио за пределы устройства | ML #1 → security eng в v2.0 | PQ |
| Калибровка frustration-модели (порог 50, полураспад 20/15/5/1 мин) | ML #1 | Product/QA |

Правило bus-factor: у каждого критического модуля — именованные primary **и** deputy к концу Beta; к v1.0 каждый модуль из топ-10 реестра рисков имеет bus factor ≥ 2.

---

## 5. План найма

Триггер — **начало фазы, которой нужен человек** (lead time ~2–3 мес), не появление боли. Обратное правило: не наращивать headcount до прохождения гейта предыдущей фазы.

| Роль | Триггер найма | Ключевые требования скоркард-карты | Формат |
|---|---|---|---|
| Rust #2 | Старт Phase 1 (найма к концу Alpha-velocity-данных, спринт 3+) | Realtime-системы, tokio, IPC/UDS, event-driven; Tauri — плюс | сотрудник |
| QA-контрактор (part-time) | Старт Phase 2 | Acceptance-автоматизация, pytest-bdd, golden-датасеты | контракт |
| Дизайнер (контракт) | Старт Phase 2 | Motion/interaction для оверлеев, понимание ограничений dual-window поверх fullscreen-игр | контракт |
| Release/DevOps | Старт Phase 3 | Код-подписи, нотариализация macOS, CDN, каналы обновлений | сотрудник |
| ML #2 | Старт Phase 3 (перенос в Phase 2, если локальные модели на слабом железе окажутся сложнее) | Model eval, ONNX/GGUF-квантизация, качество эмбеддингов | сотрудник |
| Frontend #2 | Старт Phase 3 | Кроссплатформенный Tauri, доступность (WCAG) | сотрудник |
| Security/Compliance | Старт Phase 4 (по факту enterprise-LOI) | SOC 2, audit logs, threat modeling | сотрудник |
| Solutions Engineer | Старт Phase 4 (по LOI пилота) | Enterprise-пилоты, Jira/Slack MCP, customer-facing | сотрудник |

Ограничения: не более одного «рискованного» (недоказанного в домене) найма на фазу; каждая позиция закрывается только после подписания IP assignment (до первого коммита). Микс сотрудник/контрактор и вилки компенсаций — делегированы в `05-Business-Model-Financial-Plan.md`.

**Онбординг (2 недели до первого смерженного PR):** дни 1–2 — окружение + монорепо-тур + чтение PAD/PRD/RFC-0001 + установка и **догфудинг** самого Archie; неделя 1 — 2–3 задачи `good-first-issue` с FR-ID, пейринг с владельцем модуля; неделя 2 — маленькая фича end-to-end через story DoD. Бадди онбординга — deputy модуля (намеренно не primary). Метрика: первый merged PR ≤ 10 рабочих дней; самостоятельное владение модулем ≤ 6 недель.

---

## 6. Матрица RACI

Роли: **F**ounder, **TL** (Tech Lead), **R**ust-eng, **ML**, **FE**, **D**esigner, **QA**, **RE** (Release Eng). Ровно один A в строке.

| Артефакт / решение | F | TL | Rust | ML | FE | D | QA | RE |
|---|---|---|---|---|---|---|---|---|
| RFC/ADR (технические решения) | I | **A** | R | R | R | C | C | C |
| Изменение скоупа PRD | **A** | R | C | C | C | C | C | I |
| Изменение чисел SLA (только через RFC-0001) | **A** | **A*** | R | R | C | I | C | I |
| Выход из фазы (gate exit) | **A** | R | C | C | C | I | C | C |
| Release go/no-go | C | **A** | C | C | C | I | C | R |
| Курирование библиотеки SOP | **A** | I | I | C | I | C | I | — |
| Security-инцидент | C | **A** | R | R | I | I | C | R |
| Найм (инженерные роли) | **A** | R | C | C | C | I | I | I |
| Определения KPI/телеметрии | **A** | C | C | R | I | I | C | C |
| Эскалация бета-поддержки | **A**→TL c Beta | R | C | C | C | — | R | — |
| Курирование golden-сетов / acceptance-критериев | I | C | C | C | I | — | **A** | C |

\* SLA-изменения — совместно F+TL, обязаны пройти цикл RFC-0001 → этот план → PRD (doc-sync CI-проверка держит таблицы согласованными).

---

## 7. Спринт-ритм, ритуалы и Definition of Done

### 7.1 Каденция (2-недельный спринт, фиксированный шаблон Mon–Fri)

| Ритуал | Когда | Длит. | Вход | Выход |
|---|---|---|---|---|
| Sprint Planning | Пн, нед. 1 | 45 мин | прогрумленный бэклог, DoR | спринт-гол, назначение владельцев |
| Async-стендап | ежедневно, до 11:00 в чате | — | — | запись blocked/on/need, поискаемая в канале |
| Grooming + spike-check | Пт, нед. 1 | 30 мин | входящие FR, спайки | готовые к DoR истории |
| Sprint Review (демо на живом железе) | Пт, нед. 2 | 30 мин | работающий продукт | записи в `docs/demos/` (с согласием участников) |
| Retro | Пт, нед. 2 | 30 мин | метрики спринта | действия в трекер |
| Documentation hour | Пт, нед. 2 | 60 мин | RFC-стабы, этот план | обновлённые docs/ |
| Founder–TL sync | еженедельно | 30 мин | скоуп, найм, риски | решения |
| ADR review | раз в 2 нед при открытых RFC | 30 мин | RFC-треды | принятые/отклонённые ADR |
| Phase-gate review | конец фазы | полдня | матрица приёмки + evidence | подписанный gate-запись в `docs/gates/<phase>/` |

Бюджет синхронных встреч: **≤ 6 ч/нед на инженера**. Всё остальное — async-first (письменные RFC с дедлайнами комментариев), чтобы беречь блоки deep-work для аудио-цикла и ML-оценок. С Beta вводится ротация on-call/interrupt (один инженер в неделю, ≤ 20% ёмкости) и отдельный канал инцидентов.

### 7.2 Definition of Done — Story

- [ ] Код отревьюен владельцем **и** deputy модуля.
- [ ] Unit-тесты для математики (decay фрустрации, косинус-матчинг, XY-cut) и интеграционные для bus/IPC.
- [ ] FR/NFR-ID из PRD указан в PR и названиях тестов (grep-проверяемо).
- [ ] Замерен RAM-delta: нет регрессии сверх фазового бюджета (WorkerHealthMetrics).
- [ ] Latency-пути замерены на rig, где применимо.
- [ ] Документация обновлена (RFC/ADR, если затронут контракт).
- [ ] **Ноль PII** в новых логах/телеметрии — пункт чеклиста ревьюера.
- [ ] Фича-флаг, если экспериментально.

### 7.3 DoD — Sprint / Phase

**Sprint:** демо на реальном железе; CI зелёный, включая RAM/latency-гейты; ничего в статусе «done but untested»; телеметрия нового user-facing поведения подключена к метрик-registry; actions ретро зафиксированы.
**Phase:** матрица приёмки (раздел 8) зелёная со ссылками на evidence в `docs/gates/<phase>/`; PRD/QA/Release-доки синхронизированы; риски перескорированы; retro фазы; этот документ версионируется (0.x → по фазам, 1.0 на запуске v1.0).

**Анти-чеклист (что НЕ «готово»):** «код написан, тесты потом»; демо с записанного видео; SLA «почти зелёный»; контракт изменён устно в чате; PII «временно логируется».

### 7.4 Definition of Ready

История не входит в спринт без: владельца, FR-ID, критерия приёмки, оценки задач ≤ 1 дня гранулярности. Это самый дешёвый контроль против scope creep.

### 7.5 Ёмкость и оценки

- Velocity бейслайнится на спринтах 1–3 и перевалидируется на каждой фазе (новые наймы: 50% первый спринт, 75% второй).
- Буфер прерываний 20% (бета-поддержка, ревью, инциденты).
- ML-задачи качества формулируются как измеримые планки (WER, OCR-accuracy, косинус-precision), а не «улучшить модель»; спайки ≤ 1 спринта с kill criteria.
- Целевые health-метрики: variance velocity < 20% после спринта 3; turnaround ревью < 24 ч (медиана); CI wall-clock p95 < 20 мин; escaped defects падают от фазы к фазе; наём → первый PR ≤ 10 дней.

---

## 8. Матрица приёмки по фазам (гейты дорожной карты)

Все числа — канон RFC-0001; этот документ только **планирует их проверку**. Evidence хранится в `docs/gates/<phase>/` (коммит-хеши + версии доков на момент «готово»).

### GATE-1 — MVP Alpha

| Критерий | Метрика (RFC-0001) | Метод | Evidence | Владелец |
|---|---|---|---|---|
| Barge-in | остановка TTS по `FORCE_STOP_AUDIO` **< 200 мс** | audio loopback rig в CI + месячный прогон на живых микрофонах | логи rig | TL |
| VAD | детекция речи **85–100 мс**, точность **> 95%** | rig | отчёт | ML |
| Z-Sense OCR | инференс **~200 мс** (эталон 185 мс), жёсткий fail 300 мс; пакет ~80 МБ; трафик −99% | golden-image бенчмарк CI | отчёт CI | ML |
| Graceful Degradation | `API_TIMEOUT` → локальная 1.5B без потери голоса и локальных действий | fault-injection тест | лог теста | ML |
| RAM | **≤ 150 МБ** Idle+sensing | WorkerHealthMetrics в CI | отчёт CI | R2 |
| Персона | промпт «Старший брат» живой в диалоге | догфудинг | демо | F |
| Догфудинг | ≥ 3 участников daily-active 2 недели подряд | телеметрия (opt-in) | отчёт | PQ |

### GATE-2 — B2C Beta

RAM **655 МБ Peak** зелёный в CI 4 недели подряд; порог фрустрации **50** (S_max=100, полураспад 20/15/5/1 мин) откалиброван, 0% срабатываний в GAMING_REST; окно отмены Soft Widget **3–5 с**; ночная .md-сводка в Obsidian/Notion через MCP — 100% успешных ночей; Tier 1/2/3 + физический запрет Delete (100% → `Archie_Staging`); оба пути онбординга (А/Б) инструментированы; 100–500 внешних бета-пользователей.

### GATE-3 — v1.0 Production

Semantic Router обходит LLM при косинусе **≥ 0.98** на acceptance-сете; SOP исполняется только через песочницу (AST-валидация, MCP-инструменты, Tier 1–3) — ноль эскейпов в adversarial-сьюте; RAM **1450 МБ** Offline; инсталлятор **~8 МБ** (без весов моделей); подписанные/нотаризованные инсталляторы; автообновление подтверждено staged rollout; WCAG 2.2 AA; Tier-2 staging commit/rollback показан в production-использовании.

### GATE-4 — v2.0 Enterprise

PII-санитайзер: **100% маскирование** ключей/карт/проприетарного кода тегами `[REDACTED_*]` на adversarial-сете; работа в закрытом контуре **без внешнего интернета** (on-prem/VPC); коннекторы Jira/Slack в пилоте; Burnout Tracker с анонимизацией; audit log integrity; SOC 2 Type I запущен.

---

## 9. Реестр рисков поставки (топ-10)

Скоринг: вероятность × влияние (1–5). Перескорируется на каждом гейте и поквартально; полный реестр — `docs/risks.md`.

| ID | Риск | P | I | Митигация | Триггер эскалации | Владелец |
|---|---|---|---|---|---|---|
| R1 | **Регрессии латентности аудио-цикла** (Barge-in вылезает за 200 мс после рефакторингов) | 4 | 5 | loopback rig в CI с Phase 1; monthly live-прогон deputy; критический путь аудио не трогается между измерениями | p95 > 180 мс в CI два прогона подряд | TL |
| R2 | **Качество OCR на русскоязычном тексте** (PP-OCRv5 на RU UI/геймминг-оверлеях ниже 99.2%) | 3 | 4 | golden-сет RU-скриншотов с Phase 1; миграция на PP-OCRv6 за флагом; спайк «RU fine-tune» с kill criteria | accuracy < 95% на RU-подмножестве | ML |
| R3 | **Всплески стоимости cloud API** ломают юнит-экономику | 3 | 4 | лимиты/бюджеты на аккаунтах; кэширование и Semantic Router (≥ 0.98 — ноль вызовов LLM); margin-гейт в `05-Business-...md`; фолбэк на локальную 1.5B | cost/MAU выше плана 2 месяца подряд | F |
| R4 | **Задержки нотариализации macOS** (Apple notarization, export compliance) срывают релиз v1.0 | 3 | 3 | ранний dry-run пайплайна подписи в Beta (см. `09-CI-CD-Distribution.md`); буфер 2 недели в календаре Phase 3; запасной канал ручной раздачи | первый notarization fail | RE/TL |
| R5 | **Крахи rdev на macOS** (конфликт CGEventTap с event-loop Tauri) | 4 | 4 | rdev **запрещён**: глобальные хоткеи — только штатный Tauri-плагин; компактный Soft Widget + resize-on-activation вместо полноэкранного полотна; ручное тестирование macOS-ветки с Alpha | любой crash на macOS при перехвате ввода | TL |
| R6 | **Эскейп SOP-песочницы** (рецепт обходит карантин, unsafe-вызовы) | 2 | 5 | AST-валидация компилятора рецептов; только типизированные MCP-инструменты и Tier 1–3 API; adversarial-сьют в CI; запрет прямого exec из `sop_sandbox_runner.py` | любой fail adversarial-сьюта | TL |
| R7 | **Scope creep** от демо-запросов инвесторов/юзеров | 4 | 3 | правило вытеснения равного FR (PRD MoSCoW); DoR блокирует неоформленное; демо-фичи только за флагом с issue на удаление | > 2 внеплановых FR в спринт | F |
| R8 | **ML-инженер — единственная точка отказа** (аудио + OCR + роутер в одной голове) | 4 | 5 | deputy-пейринг с Phase 1; trigger найма ML #2 в Phase 3 (перенос в Phase 2 при сложностях локальных моделей); модель-registry и ADR-дисциплина | ML #1 в отпуске > 2 нед с блокерами | F/TL |
| R9 | **Перегрузка основателя «клей-ворком»** (продукт+найм+код+GTM+поддержка) | 4 | 3 | явный аудит capacity основателя в каждом sprint review; делегирование SOP-курации и бета-поддержки с Phase 2; жёсткий cap на Accountable-строки RACI | capacity > 110% два спринта | F |
| R10 | **Эрозия фазовых гейтов** под дедлайнами («дочиним OCR в Beta») | 4 | 3 | критерии фиксируются до старта фазы; evidence в `docs/gates/`; красный критерий = письменный ваивер за подписью основателя (счёт ваиверов — метрика) | первый ваивер в фазе | F |

Вне топ-10, но отслеживаются: bus-factor Tech Lead (R11), silo контракторов-QA/дизайнера (R12), churn схемы event envelope (митигируется заморозкой v0 в Sprint 0 + версионирование), выгорание 4–5-членной команды (sustainable pace, interrupt cap 20%), хайринг дольше плана (контрактный мост), кросс-бордер телеметрия RU-команда/EU-бета (решение по residency **до** включения бета-телеметрии).

---

## 10. Инструменты

| Назначение | Инструмент | Примечания |
|---|---|---|
| Репозиторий и трекер | GitHub (issues + Projects); метки = FR-префиксы (FR-ZS, FR-AL, …), milestones = фазы дорожной карты | матрица трассировки FR → issue → тест → evidence генерируется скриптом из экспорта трекера, не руками |
| CI | GitHub Actions (переход на self-hosted runner, когда audio rig и загрузка моделей вылезут за ~20 мин wall-clock) — спецификация гейтов в `09-CI-CD-Distribution.md` | RAM-гейт читает WorkerHealthMetrics; PR-шаблон вшивает story DoD-чеклист |
| Документация и решения | markdown в `docs/` монорепо, PR-ревью; ADR-лог общий с `12-ADR-Glossary.md`; markdownlint + проверка уникальности FR-ID в CI | план версионируется по фазам |
| Дизайн | Figma с dev-mode; дизайнер пейрится в PR фронтенда | ограничения dual-window поверх fullscreen-игр — обязательный пункт брифа |
| Коммуникации | один чат-инструмент (Telegram/Slack — решение команды), письменные стендапы в канале, RFC-треды с дедлайнами, канал инцидентов с Beta | async-first правила — `docs/communication.md` |
| Риски/гейты | `docs/risks.md` + `docs/gates/<phase>/` как versioned markdown | без тяжёлого GRC-тулинга на этом размере |
| Метрики | единый metric registry (YAML в репо) — общий с `10-Observability-Telemetry-Privacy.md` и PRD: team-KPI и product-KPI не могут разъехаться | velocity, review turnaround, interrupt %, bus-factor status |

---

## 11. Открытые вопросы

1. Микс сотрудник/контрактор и юрисдикция команды (RU-юрлицо vs международные контракторы) — делегировано в `05-Business-...md`; влияет на IP assignment и стоимость.
2. Длина спринта 2 vs 1 неделя на Alpha — решение на retro спринта 3.
3. QA: part-time-контрактор до v1.0 или полный найм раньше — по данным escaped defects от Alpha-догфудинга.
4. Тайминг найма ML #2 (Phase 3 vs Phase 2) — по бенчмаркам локальных моделей на слабом железе с Alpha.
5. Семантика порога фрустрации 50 (гейт vs триггер) и дефолты cooldown — финализируется sign-off RFC-0001 до закрытия критериев GATE-1.
6. Политика ваиверов гейтов: сообщать счёт инвесторам или внутренняя — решение основателя.
7. Кросс-бордер телеметрия (RU-команда, EU-бета) — решение о residency до включения телеметрии Phase 2.

**Принцип этого документа:** он — операционная система команды, а не витрина для инвесторов. Если реальный процесс разошёлся с планом — правится план (через PR и ADR), а не молча процесс.
