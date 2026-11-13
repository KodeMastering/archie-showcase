# 07 — Testing & QA Strategy (Стратегия тестирования и обеспечения качества)

**Проект:** Archie («Архи») — OS-Level AI Symbiote
**Тип документа:** Стратегия тестирования и QA (HOW мы верифицируем контракты)
**Версия:** 0.1.0 (SemVer; golden-датасеты и бейзлайны версионируются отдельно)
**Статус:** Draft → на review инженерной команде и основателю
**Источники истины:** RFC-0001 (`Архитектурный RFC платформы Archie.docx`), Архитектурный документ (`ARCH.txt`), PAD (Archie.docx)
**Связанные документы:** `01-PRD.md` (приёмочные критерии FR/NFR), `03-Threat-Model-Security.md` (модель угроз, PII, песочница), `09-CI-CD-Distribution.md` (пайплайн сборки), `10-Observability-Telemetry-Privacy.md` (телеметрия KPI), `11-External-API-MCP-Contracts.md` (контракты MCP)
**Соответствие стандартам:** ISO/IEC/IEEE 29119-3 (структура тест-плана и трассируемость), ISO/IEC 25010 (модель качества), OWASP WSTG v4.2 / MASVS / Top 10 for LLM Applications (2025), WCAG 2.2 AA (с v1.0)

---

## 1. Purpose & QA Philosophy: «контракты — это тесты»

Archie — система, в которой продукты рискуют измеримо: прерывание речи за 200 мс — это воспринимаемая «живость» ассистента; 150 МБ RSS в фоне — это не конкурирование с браузером студента; ни один необезличенный токен PII в исходящем трафике — это доверие и юридическая безопасность. Такие свойства нельзя «подразумевать» — их нужно непрерывно **проверять автоматизированными, бюджетированными утверждениями**.

Ядро философии настоящего документа:

> **Каждый числовой контракт RFC-0001 получает минимум один автоматизированный тест на самом низком возможном слое (fast, deterministic) плюс один системный witness-тест, подтверждающий контракт end-to-end.**

Канонические числа (из RFC-0001, не подлежат пересмотру здесь):

| Контракт RFC-0001 | Значение | ID для трассировки |
|---|---|---|
| Barge-in end-to-end | < 200 мс (speech-onset → mute/`FORCE_STOP_AUDIO`) | SLA-AUDIO-01 |
| VAD-детекция | 85–100 мс, точность > 95% | SLA-AUDIO-02 |
| OCR окна на CPU | ~200 мс (эталон 185 мс), confidence 0.992 в профиле | SLA-ZSENSE-01 |
| RAM (RSS, всё дерево процессов) | 150 МБ Idle / 655 МБ Peak / 1450 МБ Offline | SLA-RAM-01/02/03 |
| Индекс Фрустрации | порог 50, S_max = 100, полураспад 20/15/5/1 мин (STUDY/CODING/IDLE/GAMING), cooldown 10 мин (штат) / 20 мин (отмена), reset 15.0 / 10.0 | SLA-FRUST-01 |
| Semantic Router | cosine similarity >= 0.98 → обход облачной LLM | SLA-ROUTER-01 |
| PII Sanitizer | 100% маскирование ключей/карт тегами `[REDACTED_*]` до любого egress | SLA-PII-01 |
| Staging FS | физический запрет системного вызова Delete; 100% переноса в `Archie_Staging` | SLA-FS-01 |
| Soft Widget | окно отмены 3–5 с | SLA-WIDGET-01 |
| Graceful Degradation | переключение на локальную 1.5B при `API_TIMEOUT` без потери голоса | SLA-GD-01 |

Второй принцип: **traceability blocking, coverage advisory**. Процент покрытия — вспомогательная метрика; блокирующим гейтом является матрица трассируемости (раздел 16): CI падает, если хоть один ID контракта RFC-0001 не закрыт зелёным тестом. Это защита от «coverage theater» — погони за 80% тривиальными тестами при нетестируемых реальных рисках.

Третий принцип: **измерять то, что воспринимает пользователь**. Латентность barge-in, измеренная «от лога до лога» в приложении, врёт: она не включает буферы драйвера и аудиоклок устройства. Поэтому все латентностные тесты Archie привязаны к аппаратному тактированию аудиоустройства и hardware-in-the-loop ригу (раздел 6).

---

## 2. Scope, Roles & Phase Alignment

### 2.1. Scope

**In scope:** полная тестовая пирамида для всех модулей монорепо (`event bus`, `frustration scorer`, `audio loop / barge-in`, `Z-Sense/OCR`, трёхуровневая память, `semantic router`, `MCP hub`, `staging FS`, `soft widget`, `PII sanitizer`, `IPC bridge` Rust↔Python↔TS); CI-гейты и политики merge; матрица тестовых окружений; golden-датасеты (OCR, PII adversarial corpus); RAM/performance regression harness (150/655/1450 МБ); security-тесты SOP-песочницы; приёмочные чеклисты по фазам.

**Out of scope:** нагрузочное тестирование multi-tenant Enterprise-масштаба (v2.0 — только бюджет заложен); формальные аудиты SOC2/GDPR (отдельный compliance-документ); QA-процессы вендоров облачных LLM; аппаратная сертификация.

**Покрытие по фазам:** полная глубина для Фазы 1 (MVP Alpha) и Фазы 2 (B2C Beta); пункты v1.0 специфицированы, но помечены `deferred`; v2.0 Enterprise — плейсхолдеры.

### 2.2. RACI (сокращённо)

| Активность | Dev | QA | Founder | Внешний pentest |
|---|---|---|---|---|
| Unit/property-тесты, PR-гейты | R/A | C | I | — |
| Loopback-риг, RAM-регрессия, golden-датасеты | C | R/A | I | — |
| PII adversarial corpus, red-team challenge | C | R/A | C | C |
| Security-аудит песочницы (перед Beta) | C | C | A | R |
| Sign-off фазовых гейтов | C | R | A | I |

Каждая строка матрицы модулей (раздел 4) имеет именованного owner — план обязан пережить кадровый рот.

### 2.3. Definition of Done (PR-чеклист)

Фича считается готовой, если: (1) тест написан на правильном слое; (2) контракт трассируем (ID в коде теста); (3) flake-бюджет соблюдён; (4) если задет числовой контракт RFC-0001 — обновлён witness-тест и матрица трассируемости перегенерирована.

---

## 3. Test Pyramid & Layer Definitions

Целевая пропорция: **~70% unit / ~20% integration / ~7% system+e2e / ~3% manual/exploratory**.

| Слой | Доля | Что тестирует | Разрешённые зависимости | Время |
|---|---|---|---|---|
| **Unit** | ~70% | Чистая логика: decay-математика scorer'а, regex-правила sanitizer'а, XY-cut сортировка, роутинг event bus, state-машины. Rust: `cargo-nextest` + `proptest` + `insta`. Python: `pytest` + `Hypothesis`. | Только in-process; никаких сетевых вызовов, реальных аудиоустройств, моделей на диске в general-случае | < 10 мс/тест |
| **Integration** | ~20% | Швы между процессами/модулями: gRPC IPC Rust↔Python поверх UDS/Named Pipes, Shared Memory кадры, шина ↔ подписчики, sandbox-runner ↔ staging FS, MCP hub по stdio. Виртуализированные устройства (null-sink). | Локальные процессы, loopback-устройства, зафиксированные fixtures; никакого интернета | < 30 с/тест |
| **System / E2E** | ~7% | Witness-тесты контрактов: barge-in < 200 мс на риге, OCR на golden-датасете, RAM-профили, e2e-сценарии через `tauri-driver`. | Reference-машина, записанные LLM-фикстуры (VCR) | минуты |
| **Manual / Exploratory** | ~3% | «Живость» персоны, тон голоса, UX-ощущение barge-in, онбординг-потоки. Чеклисты на реальных устройствах. | Всё | сессии |

Правила слоя:

1. **Юнит-тесты не пересекают IPC-мост.** Всё, что дотрагивается до `contracts/ipc_bridge.proto` — integration. Границы слоёв зеркалят границы процессов из архитектуры.
2. **Всё, что касается реального аудио-тайминга или реальных OCR-моделей — только integration/system с виртуализированными устройствами.** Мокать аудио-тайминг в юнитах — проверенный способ обнаружить barge-in-регрессию только от пользователей.
3. **E2E — «few, deep, reliable»:** нумерованные сценарии (E2E-01..05), стабильные, с machine-verifiable assertions; никаких e2e с живой облачной LLM (только записанные фикстуры).
4. **Схемы загружаются буквально из репо:** тесты conformance загружают `contracts/events_envelope.schema.json`, `contracts/ipc_bridge.proto` и SQL DDL из RFC-0001, так что спека и тесты физически не могут разойтись.

---

## 4. Module-by-Module Test Matrix

Формат ID: `<модуль>-<слой>-<NNN>`. Слой: UNIT / INT / SYS.

| Модуль | Закрепляемый контракт | Unit-кейсы | Integration-шов | Owner (роль) | Фаза |
|---|---|---|---|---|---|
| **Event Bus** (`crates/archie-event-bus`) | SLA-шина: микросекундная маршрутизация, ordering, delivery, backpressure, graceful degradation | FB-UNIT-001..030: конформанс envelope к `events_envelope.schema.json` (JSON Schema round-trip + `insta`-снапшоты); ordering per-stream; idempotency publish; приоритеты (0=Real-time audio, 1=UI, 2=telemetry); routing таблица событий (APP_OPENED → mission_state_machine + semantic_router и т.д. по матрице §4.3 ARCH) | FB-INT-001..012: UDS/Named Pipe tonic-сервер ↔ grpcio.aio клиент; slow-subscriber isolation (один медленный подписчик не блокирует Z-Sense/audio > 100 мс); 1000 events/s sustained; kill -9 Python-воркера → replay/recovery; `API_TIMEOUT` → переключение offline_fallback | Rust core dev | Ф1 |
| **Frustration Scorer** (`proactivity/frustration_scorer.py`) | SLA-FRUST-01: модель затухания RFC-0001 §4 | FS-UNIT-001..040 (property-based, см. раздел 5) | FS-INT-001..008: подписка на `ZSENSE_GRAPH_READY`/`FRUSTRATION_SPIKE`; запись в `frustration_telemetry` (SQL CHECK-констрейнты); media-freeze через `active_media_playing` | Python ML dev | Ф2 |
| **Audio Loop / Barge-in** (`audio_loop/*`) | SLA-AUDIO-01/02: остановка TTS < 200 мс; VAD 85–100 мс, >95%; отсутствие самопрерываний при double-talk | AL-UNIT-001..020: VAD-gate на записанных PCM-чанках (30 мс); endpointing-пороги 400/1400 мс; turn-detection классификатор «угу» vs «стоп»; AEC3 far/near выравнивание на фикстурах | AL-INT-001..010 (loopback-риг, раздел 6): end-to-end latency; AEC double-talk; sigh-detector → scorer | Python audio dev | Ф1 |
| **Z-Sense / OCR** (`zsense/*`) | SLA-ZSENSE-01: ~200 мс на CPU; XY-cut порядок; JSON-граф по `zsense_context.schema.json` | ZS-UNIT-001..025: xy_cut_sorter на синтетических наборах боксов (перекрытия, multiline, RTL); json_graph_builder — schema conformance | ZS-SYS-001..006: golden-датасет (раздел 7) — F1 + латентность; Apple Vision префильтр (macOS, deferred) | Python ML dev | Ф1 |
| **PII Sanitizer** (`security/*`) | SLA-PII-01: маскирование до egress | PI-UNIT-001..050: corpus EN+RU (раздел 8); regex_rules + NER; теги `[REDACTED_API_KEY]`, `[REDACTED_FINANCIAL_DATA]` | PI-ADV-001..010: adversarial corpus + canary egress через MCP hub choke point | QA + dev | Ф2 (full), Ф1 (regex-basis) |
| **Memory Tiers** (`memory/*`) | Tier 1 — кольцевой буфер 2–4 ч (TTL, eviction); Tier 2 — ночная сводка .md через MCP; Tier 3 — LanceDB RAG | MT-UNIT-001..025: TTL-истечение Tier1; SQL DDL conformance (`session_memory_tier1` и др.); summarizer-формат; vector search детерминизм | MT-INT-001..010: purge-тест (user-triggered memory purge удаляет все тиры с forensic-проверкой SQLite + staging remnants); FastMCP запись в Obsidian/Notion mock; WAL-режим | Python dev | Ф2 |
| **Semantic Router** (`router_and_sops/*`) | SLA-ROUTER-01: cosine >= 0.98 → `BLUEPRINT_MATCHED_98`, блокировка облачной LLM | SR-UNIT-001..030: параметризованные кейсы 0.97/0.98/0.99 (boundary band); LanceDB search на фикс. эмбеддингах; ast_validator — только whitelisted MCP-вызовы Tier 1–3 | SR-INT-001..008: перехват `APP_CLOSED` до `cloud_gateway`; генерация DRAFT через фикстуру; утверждённый blueprint → sandbox | Python dev | Ф3 |
| **SOP Sandbox** (`sop_sandbox_runner.py`) | SLA-FS-01 + sandbox-инварианты (раздел 9) | SB-UNIT-001..030: ast_validator — реджект опасных системных вызовов, инъекций в аргументы | SB-INT-001..015: fs jail, network deny, fork bomb, tier escalation; cargo-fuzz парсера | QA + Rust dev | Ф2 (staging), Ф3 (sandbox) |
| **MCP Hub** (`mcp_hub/*`) | Транспорты stdio/SSE/Streamable HTTP; on-demand spawn (TTL 30 с); choke point для egress | MH-UNIT-001..015: schema tool-вызовов; таймауты; restart изоляция | MH-INT-001..010: реальные дочерние MCP-серверы на mock-целях; network capture на границе | Python dev | Ф2 |
| **Staging FS** (`crates/archie-stagingfs`) | SLA-FS-01: физический запрет Delete; перенос 100% в `Archie_Staging`; `action_audit_log` | SF-UNIT-001..025: permission_tiers логика; desktop_cleaner правила; workspace_orchestrator (Tier 3) | SF-INT-001..010: реальные FS-операции в jail; симуляция TOCTOU-гонок | Rust dev | Ф2 |
| **Soft Widget / UI** (`apps/desktop-ui`) | SLA-WIDGET-01: окно 3–5 с; resize-on-activation; Win+D immunity; prevent_close+hide | SW-UNIT-001..020: CancelCountdown state machine (0–3 с: отмена активна; >5 с: развязка); компонентные тесты React | SW-INT-001..010: `tauri-driver`/WebDriver — z-order, focus, двухоконная синхронизация, HiDPI scale | Frontend dev | Ф1/Ф2 |
| **IPC Bridge** (`contracts/` + ipc_server.rs) | Proto-контракт, UDS 0600 / Named Pipe ACL, shm-дескрипторы | IB-UNIT-001..020: кодоген TS/Rust/Python сходится; proto round-trip всех `EventEnvelope` payload-вариантов (oneof) | IB-INT-001..010: `StreamEvents` полнодуплексный поток; `TriggerZSenseOcr` по shm_slot; права доступа к сокету | Rust core dev | Ф1 |
| **Cloud Gateway / Fallback** (`llm_clients/*`) | SLA-GD-01: `API_TIMEOUT` → локальная 1.5B без потери голоса | CG-UNIT-001..020: provider-адаптеры на фикстурах; persona_prompts; memory_lifecycle `LazyModelSlot` TTL-логика | CG-INT-001..008: mock 500/timeout → флаг `offline_fallback_active`; mmap 1.5B вытесняет RapidOCR/NER | Python dev | Ф1 |

---

## 5. Event Bus & Frustration Decay: property-based test pack

### 5.1. Event Bus (Rust, proptest + insta)

Инварианты:

- **P-FB-01 (schema conformance):** любой сгенерированный валидный `EventEnvelope` сериализуется/десериализуется без потерь и проходит валидацию по `contracts/events_envelope.schema.json` (файл загружается из репо). `insta`-снапшоты эталонных envelope'ов (включая `ForceStopAudioPayload` с `audio_clock_sample_idx` и `stop_latency_us`).
- **P-FB-02 (ordering):** для одного publisher'а события доставляются подписчику в порядке publish.
- **P-FB-03 (backpressure):** при sustained 1000 events/s (provisional target) — потери событий 0; один slow subscriber с приёмкой < 1 event/s не повышает latency публикации для других > 100 мс (изоляция очередей per-subscriber).
- **P-FB-04 (recovery):** kill -9 подписчика → bus остаётся жив, очередь сбрасывается/реплеится по конфигурации; publisher'ы не блокируются.
- **P-FB-05 (priority):** события priority 0 (`FORCE_STOP_AUDIO`) никогда не ждут за очередью priority 2 (telemetry) при конкуренции.

### 5.2. Frustration Scorer (Python, Hypothesis): инварианты из RFC-0001 §4

Генераторы: случайные последовательности `(t_k, e_k, mission, media_flag)` с монотонным `time.monotonic`-замещением (инъекция `now_ts`, как в эталонной реализации `frustration_scorer.py`).

| ID | Инвариант | Формулировка свойства |
|---|---|---|
| P-FS-01 | **Диапазон** | `0.0 <= S(t) <= 100.0` после любой последовательности операций (насыщение `min(S_max, ...)`, clamp в 0 при < 0.05) |
| P-FS-02 | **Монотонное затухание между событиями** | без новых эвристик `S` не возрастает; `S(t2) <= S(t1)` при `t2 > t1` |
| P-FS-03 | **Границы затухания** | для каждой миссии: `S(t+T_1/2) ≈ S(t)/2` (допуск 1e-3); точные значения λ: STUDY_FOCUS 20 мин (λ = 0.000578 с⁻¹), CODING_WORK 15 мин (0.000770), IDLE 5 мин (0.002310), GAMING_REST 1 мин (0.011552). Тест сверяет `HALF_LIFE_SECONDS` из кода с таблицей RFC — дрифт констант = провал |
| P-FS-04 | **Переполнение/NaN** | никаких inf/NaN при любых dt (включая dt = 0, огромные dt, отрицательные — clamp `max(0.0, ...)`) |
| P-FS-05 | **Zero-gain режимы** | в `GAMING_REST` и `IDLE` `record_heuristic` не меняет score (μ = 0) — «0% срабатываний в GAMING_REST» |
| P-FS-06 | **Media-фильтр ψ** | при `active_media_playing=True`: `BROWSER_TAB_STALL_15M` → ΔS = 0; `AUDIO_HEAVY_SIGH_DETECTED` → ΔS = 15 (0.5 × 30); консольные/IDE-эвристики → полный вес |
| P-FS-07 | **Порог 50, boundary band** | срабатывание `should_trigger` ⇔ `S >= 50 ∧ t >= cooldown_until`. Явные кейсы: S = 49.999 → нет; S = 50.0 → да; S = 49.9 + ΔS=10 (STALL) → 59.9 → да (в активной миссии); накопление 10+15+20+30 = 75 → clamp к 75 ≤ 100; порог из 3 событий: 20 (CONSOLE) + 30 (SIGH) = 50 → ровно на границе → trigger |
| P-FS-08 | **Cooldown 10/20 мин** | после штатного вмешательства: S = 15.0, cooldown = now + 600 с; после отмены виджета: S = 10.0, cooldown = now + 1200 с; событие в окне cooldown не триггерит, score продолжает накапливаться/затухать; после истечения — следующий `S >= 50` триггерит |
| P-FS-09 | **Golden traces (worked examples RFC-0001)** | (а) 30-минутный сценарий CODING_WORK из §4.2: кластер из 2–3 ошибок за 10–15 мин детектируется, разрозненные события за многочасовую сессию — нет; (б) сценарий видеолекции: STALL+вздох при media → не достигается порог; (в) смена миссии CODING→GAMING: остаточный счёт распадается с T_1/2 = 1 мин (через 5 мин < 0.05 → 0.0) |
| P-FS-10 | **SQL-телеметрия** | каждая эвристика и spike пишутся в `frustration_telemetry` с валидным `heuristic_code` (CHECK-констрейнт), `score_delta`/`accumulated_score` консистентны с моделью |

Пакет выполняется на PR (юнит-слой, секунды); критерий приёмки — **100% pass инвариантов**, без исключений.

---

## 6. Audio Loopback Rig: Barge-in < 200 мс

### 6.1. Зачем hardware-in-the-loop

Латентность, измеренная от лога до лога внутри приложения, не включает буферы драйвера, AEC-обработку и вывод — то есть не совпадает с тем, что слышит пользователь. Прецеденты индустрии (Amazon AVS SDK integration testing, Mycroft/OpenVoiceOS CI с replayed-аудио) подтверждают: loopback-риг с виртуальным аудиокабелем — норма для voice-ассистентов.

### 6.2. Конструкция рига

- **Linux (reference, Фаза 1, blocking):** PipeWire `null-sink` + `pw-loopback`; целевой вход/выход рига — виртуальная пара capture/playback. Единственный CI-stage, которому разрешено трогать аудио-железо, — self-hosted runner, он же reference-машина.
- **Windows:** VB-CABLE; **macOS:** BlackHole (позже фаз, advisory до B2C Beta).
- **Тестовый бинарник (Rust harness):** генерирует (1) TTS-вывод Archie — реальный поток `piper_tts_stream` в playback-устройство рига; (2) синтетическую «речь пользователя» (записанный PCM-файл с фразой-прерыванием) — инъекция в capture-устройство **с покадровыми таймстампами от аудио-клока устройства** (sample index), а не от логов приложения.
- **Измеряемая величина:** `t(mute) - t(speech_onset)`, где `t(speech_onset)` — сэмпл-индекс первого речевого фрейма в инжектируемом потоке по device clock, `t(mute)` — момент, когда амплитуда выходного потока TTS упала ниже порога (детекция silence на loopback capture), плюс эмит `FORCE_STOP_AUDIO` в шину.
- **N >= 30 повторений**, гейт на **p95 < 200 мс**; p99 — трекается как KPI. Одиночные прогоны не гейтят (защита от flakiness).

### 6.3. Дополнительные кейсы рига

- **AL-INT-003 (double-talk):** TTS играет, инъекция речи пользователя → самопрерывание НЕ происходит при отсутствии внешней речи (AEC3 вычитает собственный голос; ложный `FORCE_STOP_AUDIO` = провал).
- **AL-INT-004 (VAD SLA):** на корпусе записанных 30-мс чанков — детекция в 85–100 мс, точность > 95% (стратифицированный корпус: чистая речь / шум / шёпот / музыка).
- **AL-INT-005 (endpointing):** внутрифразовая пауза ~400 мс и конец реплики ~1400 мс — пороги выдерживаются на потоковой транскрипции.
- **Флашинг-протокол:** одиночный авто-retry с другим seed; повторный провал = карантин с issue, **lane отключать нельзя**.

Open question (блокирует финализацию): операционное определение 200 мс — «speech-onset → mute» (принято как рабочее) или «→ полный context switch + ASR partial». До решения RFC-0001 гейтится рабочее определение; witness-тест замеряет оба числа.

---

## 7. OCR Golden-Image Dataset

### 7.1. Состав и квоты diversity

`datasets/ocr-golden/` (SemVer, immutable версии, деградация только через новую major-версию):

- **Синтетические скриншоты** (генератор в репо, детерминированный seed): >= 4 разрешения (1366×768, 1920×1080, 2560×1440, 3840×2160), >= 2 класса DPI (96/192, scale_factor 1.0/2.0), light/dark темы, >= 3 фонтовых стека (Segoe UI / SF Pro / Noto Sans + CJK-стек), Latin / CJK / mixed `en_ru`, subpixel/ClearType-рендеринг, анимированные/движущиеся области (кадры с motion blur).
- **Field-capture set (50–100 изображений):** скриншоты реальных экранов **согласившихся внутренних пользователей**, refresh ежеквартально; содержимое — только consent-safe (без личных переписок, платёжных данных и т.п. — синтетические токены подставляются скриптом аннотации).
- **Ground truth:** JSON на изображение — `{boxes: [{x,y,w,h}], transcript, char_boxes}`; эталонный пример поля — `visible_text_snippet`/`lines` из RFC-0001 §4.1 (включая `error_state`, `low_confidence` флаги).

### 7.2. Гейты

- **Точность:** word-level F1 >= 0.95 (Latin), >= 0.90 (CJK/mixed) — [uncertain], пересматривается по продуктовому фидбеку; char-level CER — advisory.
- **Латентность:** per-image `elapsed_ms <= ~200 мс` (бюджет SLA-ZSENSE-01) на reference-машине; эталон 185 мс на кадре 2560×1440 из RFC-0001 — golden-пример.
- **Детерминизм XY-cut:** вывод сортировки сравнивается с golden-порядком (insta-снапшоты); `scale_factor`/`origin` учитываются.
- **Прогон:** nightly на pinned-машине; датасет в git-lfs/DVC; карточка датасета (model-card формат: версия, квоты, лицензии, известные пробелы, скрипт регенерации).

---

## 8. PII Sanitizer: adversarial corpus & zero-leak egress

### 8.1. Корпус (`tests/pii_adversarial/`, EN + RU)

- **Позитивные классы:** e-mails; телефоны (+7 / 8-формат / RU-раскладка; US-форматы), IBAN, PAN карт (16 цифр, LUHN-валидные), СНИЛС/паспорт-серии (RU), SSN, ключи API (`api_token = ...`, `sk-...`, bearer-токены), имена в контексте.
- **Adversarial-классы:** гомоглифы (Cyrillic `а`↔Latin `a` в `mail@example.com`); zero-width chars (ZWSP/ZWNJ внутри токена); split tokens через границы OCR-строк; PII в заголовках окон (`window_title`); base64/hex-блобы с закодированным PAN; вложенные кавычки; `при­мер@mail.ru` с soft-hyphen; PII, размазанная по `lines[]` так, что regex-каскад видит её только после XY-cut-склейки.
- **Отрицательные классы (false-positive контроль):** технические строки, похожие на PII (SHA-хеши, инвентарные номера, `2026-05-15`-датировки) — sanitizer НЕ должен выжечь весь контекст (иначе облако слепнет).

### 8.2. Zero-leak egress assertion (stop-ship)

Ключевая architectural-возможность: **MCP hub — единственная choke point исходящего трафика** (архитектура: все облачные/MCP-вызовы идут через него). Это делает zero-leak assertion практически проверяемым:

1. В каждый корпус-кейс вшивается уникальный **canary token** (raw строка PII).
2. Интеграционный тест поднимает **network-capture mock** на границе MCP hub / cloud gateway и перехватывает все исходящие фреймы.
3. **Утверждение: canary-строка не встречается ни в одном outbound payload** (байтовое сравнение + нормализованные варианты — Unicode NFKC, zero-width strip).
4. Любая утечка = **severity S1, stop-ship**: merge/release блокируется немедленно, независимо от фазы.

Red-team-каденция: ежеквартальный внутренний челлендж — любой член команды может добавить в corpus кейс, который должен быть закрыт в течение спринта. Corpus дублируется в Rust (regex hot path sanitizer'а) и Python (NER-путь).

Дополнительно (privacy-инварианты, GDPR Arts. 5/25/32, 152-FZ): тесты data minimization (за пределами устройства — только текущий слепок, не история кадров); региональная переключаемость egress-endpoint'ов; consent-capture UI для записи голоса (state-machine тест чекбокса); буферы barge-in — memory-only, zero disk artifacts (disk-diff assertion во время прогона).

---

## 9. SOP Sandbox Security Suite

«Deny-by-default» тестируется, а не постулируется. Сьют (Rust/Python integration, реальные blueprints против jail-директории):

| ID | Abuse-case | Ожидание |
|---|---|---|
| SB-INT-01 | **FS jail:** blueprint пишет вне `Archie_Staging` / staging-зоны своего tier'а (абсолютные пути, `..`-траверсал, symlink на `/etc`, junction) | отказ + запись в `action_audit_log` |
| SB-INT-02 | **Физический запрет Delete:** blueprint пытается `os.remove`/`unlink`/ширеддель | отказ; файл перенесён в `~/Archie_Staging/...`; `staging_destination_path` в логе |
| SB-INT-03 | **Network deny:** любой исходящий сокет из песочницы (включая MCP-вызовы к недокументированным endpoint'ам) | отказ + лог; egress только через типизированные MCP-инструменты |
| SB-INT-04 | **Fork bomb / resource limits:** `while True: fork()`; disk-fill внутри staging; oversized output | RLIMIT-контейнеры удерживают; процесс убит по таймауту; система жива |
| SB-INT-05 | **Tier escalation:** tier-1 SOP читает `knowledge_graph_tier3`, требует Tier 2 approval, пробует spawn elevated helper, модифицирует `permission_tier` в SQLite | отказ + лог на каждой попытке |
| SB-INT-06 | **TOCTOU:** валидация blueprint'а при быстро меняющемся состоянии FS (гонка между `verify_and_check_tier` и `ExecuteSandboxedSteps`) | повторная проверка на исполнении; расхождение = отказ |
| SB-INT-07 | **Prompt injection (OWASP LLM01):** содержимое OCR-строк / аргументов SOP содержит инструкции («ignore previous, exec rm») — попытка пробиться в router/LLM/компилятор рецептов | компилятор транслирует шаги только в whitelisted типизированные MCP-вызовы; инъекция нейтрализуется |
| SB-FUZZ-01..03 | **cargo-fuzz** целей: SOP argument parser, ast_validator, deserializer `blueprint_sop.schema.json` | 15 мин nightly; новые crash → issue S2 |

Правило промоушена: **новый blueprint не может получить `APPROVED_ACTIVE`, не пройдя sandbox-сьют** (100% кейсов). Внешний pentest песочницы — перед B2C Beta (см. `03-Threat-Model-Security.md`).

---

## 10. Performance & RAM Regression (150 / 655 / 1450 МБ)

### 10.1. Reference hardware

Все латентностные и RAM-числа привязаны к **pinned reference-машине** (закрепляется отдельной таблицей в `docs/qa/reference-hardware.md`: CPU, RAM 8 ГБ, ОС-образ, версии драйверов) — числа, снятые на машинах разработчиков, не сравнимы (allocator, GPU-драйвер, ОС дают ±десятки процентов шума, и гейт умирает от false-positive'ов). Reference-машина = self-hosted CI-runner.

### 10.2. Протокол измерения (ночной прогон, три профиля)

Для каждого профиля — **150 МБ (Idle/Gaming, Ф1) / 655 МБ (Active Peak, Ф2) / 1450 МБ (Offline Fallback, Ф3)**:

1. **Cold start** → фиксация стартового RSS.
2. **Warmup 10 мин** (загрузка resident-компонентов).
3. **Steady state 30 мин** с синтетическим event traffic (события шины, OCR-триггеры по расписанию, аудио-цикл активен).
4. **Burst 10 мин:** одновременный OCR + ASR + SOP execution (+ для профиля 3 — `API_TIMEOUT` → mmap 1.5B с вытеснением RapidOCR/NER).
5. **RSS сэмплируется 1 Гц** (сумма по всему дереву процессов Archie, включая WebView-процессы — в соответствии с покомпонентной матрицей RFC-0001 §5.2, где учтены все 13 подсистем).

Сравнение — **статистическое**: распределение N прогонов против **versioned baseline в VCS**; warning band при **90% бюджета** (135 / 590 / 1305 МБ), hard fail при превышении. Результаты — JSON в репо/artifacts, тренд-чарты в `docs/qa/trends/`.

Специфичные кейс-тесты: TTL-выгрузка моделей (60/120/180 с по классам из §5.2 RFC); `enable_cpu_mem_arena = False` реально возвращает страницы ОС (проверка через `malloc_trim` / smaps); скрытый `main_dashboard` — 16 МБ вместо 65 (DOM unmount); `ReportWorkerHealth` корректно репортит `rss_memory_bytes`/`loaded_models` и реагирует на `MemoryControlDirective`.

Лёгкий PR-проб: старт + 2 мин, только Rust-core (advisory, чтобы ловить грубые регрессии между nightly).

---

## 11. E2E Scenario Library

Все LLM-вызовы — через **записанные VCR-фикстуры** (cassette на сценарий); живые контракты с провайдерами — отдельная ночная lane, fail-open, merge не блокирует.

| ID | Сценарий | Ключевые machine-verifiable критерии |
|---|---|---|
| E2E-01 | Голосовой вопрос с экранным контекстом (VS Code с ошибкой на экране) | ответ учитывает `error_state` из Z-Sense-слепка; `ZSENSE_GRAPH_READY` в шине до LLM-вызова; sanitized payload (canary чист) |
| E2E-02 | Barge-in mid-answer | mute < 200 мс по ригу; `FRUSTRATION_SPIKE`-цепочка; `widget_cancelled_by_user` пишет корректно; после — реплика пользователя уходит в LLM (голос не потерян) |
| E2E-03 | Long-session memory recall через тиры | сессия 2–4 ч → Tier 1 eviction; ночная сводка Tier 2 → .md через MCP-mock; факт → Tier 3 → RAG-поиск находит; recall < бюджетов роутера |
| E2E-04 | SOP file-organize: staging + эскалация | исполнение целиком в staging; Tier 2 prompt возникает ровно на критическом шаге; `action_audit_log` консистентен; Delete не вызван (strace-верификация) |
| E2E-05 | Offline degradation | обрыв сети (mock) → `API_TIMEOUT` → локальная 1.5B; голос работает; RAM в рамках профиля 1450 МБ; восстановление сети → возврат cloud |

Ежемесячно — **soak-тест 8 ч** (симулированная сессия): рост памяти в рамках decay-модели, staging FS не копится бесконечно, дрейф scorer'а отсутствует. E2E-прогон: weekly + release-ветка; **release-blocking сценарии зелёные дважды подряд на двух разных reference-машинах** перед фазовым промоушеном.

---

## 12. CI Pipeline & Merge Gates

| Stage | Состав | Где | Когда | Blocking | Бюджет |
|---|---|---|---|---|---|
| L1 Lint | fmt, clippy (deny warnings), ruff, mypy, tsc | cloud runner | PR | ✅ | 2 мин |
| L2 Unit | `cargo-nextest` (Rust) + `pytest` (Python) + proptest/Hypothesis + `insta` snapshots | cloud runner | PR | ✅ | < 8 мин |
| L3 Fast integration | IPC-швы, sandbox (без аудио), fast RSS-проб | cloud runner | PR | ✅ | < 5 мин |
| L4 Traceability check | генерация матрицы: каждый ID RFC-0001 → >= 1 зелёный тест | job | PR | ✅ | < 1 мин |
| **PR-итог** | | | | | **< 15 мин** |
| N1 Nightly reference | loopback barge-in (p95), OCR golden, PII egress + adversarial, sandbox abuse + cargo-fuzz (15 мин), полный RAM-regression, soak (monthly) | self-hosted reference runner | nightly | ✅ | < 2 ч |
| N2 Live-API contract | контракты DeepSeek/OpenAI/Gemini против живых API | cloud | nightly | ❌ (fail-open) | 15 мин |
| W1 Weekly e2e | E2E-01..05 + smoke WCAG (tauri-driver) | reference + вторая машина | weekly | ✅ (release-ветка) | < 90 мин |
| R1 Release | полный пакет × 2 прогона × 2 reference-машины | release branch | перед промоушеном | ✅ | по чеклисту |

Практики: flaky-тест — авто-retry (1, другой seed), после 2 срывов за 14 дней — **quarantine-лейбл с expiry**, fix-or-delete в спринте (lane отключать нельзя, только карантин с issue); coverage ratchet — Rust core >= 80% line (`cargo-llvm-cov`), Python >= 75% (`pytest-cov`), новое покрытие не может опуститься; телеметрия сама проходит PII-гейт (payload'ы тестируются корпусом раздела 8); secrets-гигиена — gitleaks pre-commit, в CI никогда не используется реальная PII, только синтетика с canary.

---

## 13. Test Environments & Test Data Management

- **Матрица ОС:** Linux (reference, blocking, Ф1+) → Windows (blocking с Ф2 для RAM 655 МБ; loopback через VB-CABLE) → macOS (advisory до v1.0, потом blocking; BlackHole + Apple Vision префильтр-тесты).
- **Виртуальные устройства:** null-sink/VB-Cable/BlackHole — как в разделе 6; никаких тестов на реальных микрофонах/колонках в CI (один manual spot-check на релиз).
- **Синтетические экранные фикстуры:** генератор golden-датасета; контент — consent-safe (синтетические токены вместо реальных данных; field-capture от согласившихся внутренних пользователей с анонимизацией скриптом аннотации).
- **Данные PII:** только синтетический corpus с canary; правило: реальная пользовательская PII не попадает в CI/датасеты никогда.
- **Аудио-фикстуры:** записанные PCM (clean/noise/whisper/music), стратифицированные; для ASR — референсы Common Voice-стиля лицензирования.

---

## 14. Release Acceptance & Phase Exit Criteria

Каждый гейт — чеклист с полями sign-off (имя, дата, ссылка на CI-run как evidence).

**Фаза 1 — MVP Alpha:**
- [ ] Все контракты RFC-0001 (SLA-AUDIO-01/02, SLA-ZSENSE-01, SLA-RAM-01=150 МБ, SLA-GD-01, envelope/IPC conformance) автоматизированы и зелёные **7 ночей подряд**.
- [ ] Barge-in p95 < 200 мс на риге; double-talk самопрерываний 0.
- [ ] RAM-профиль Idle зелёный на pinned-машине.
- [ ] Event bus: backpressure/recovery-тесты зелёные; kill -9 recovery подтверждён.
- [ ] Dogfood-запуск внутри команды (daily usage) без S1.

**Фаза 2 — B2C Beta:**
- [ ] PII adversarial suite (EN+RU) + canary egress: 100%, 0 утечек (S1-инцидентов 0).
- [ ] Sandbox abuse-сьют 100% зелёный; внешний pentest песочницы завершён без критических находок.
- [ ] RAM 655 МБ (Peak) зелёной на Linux (Windows advisory до v1.0).
- [ ] Crash-free sessions > 99% (opt-in telemetry; сама телеметрия прошла PII-гейт).
- [ ] Frustration-модель: property-паки 100%, golden traces (включая видеолекционный false-positive сценарий) зелёные; telemetry порога/cool-down валидирована на dogfood-данных.
- [ ] Онбординг-потоки (Путь А/Б) — manual exploratory pass.
- [ ] E2E-01..05 зелёные дважды на двух машинах.

**Фаза 3 — v1.0:**
- [ ] RAM 1450 МБ (Offline) зелёной на **всех** поддерживаемых ОС; 655 МБ — тоже на всех ОС.
- [ ] Semantic Router: boundary band 0.97/0.98/0.99 зелёная; bypass-облака подтверждён в e2e (E2E-04).
- [ ] WCAG 2.2 AA smoke (soft widget: контраст, keyboard access, screen-reader labels через accessibility-tree из tauri-driver).
- [ ] Long-session soak 8 ч зелёный; memory purge с forensic-проверкой.
- [ ] Crash-free > 99.9%.

**Фаза 4 — v2.0 Enterprise (плейсхолдер):** On-Premise-конфигурация (egress только внутри периметра — тест конфигурацией); PII sanitizer на корпоративных форматах; Burnout Tracker — агрегированная анонимизация (гейт: не-восстанавливаемость персональных метрик); NIST SP 800-53-стиль чеклист (семейства SC/SI/AC); нагрузочное тестирование multi-tenant — отдельный бюджет.

Смок-пак подписанных сборок: installer ~8 МБ (без весов), single-instance mutex, автообновление, первичный онбординг — на каждой release-сборке.

---

## 15. Metrics, Reporting & Continuous Improvement

| KPI | Определение | Цель | Источник | Каденс |
|---|---|---|---|---|
| Contract gate pass rate | доля ночных прогонов, где все blocking-контракты зелёные | 100% (7 подряд перед гейтом) | CI | daily |
| Barge-in p95 / p99 | латентность рига | p95 < 200 мс; p99 трек | N1 | weekly review |
| OCR F1 / latency | golden-датасет | F1 >= 0.95/0.90; <= 200 мс | N1 | weekly |
| RAM trend | RSS против baseline (3 профиля) | внутри ±10% band | N1 | weekly |
| Escaped defects | баги, найденные вне CI | 0 S1; S2 — ретро-анализ | tracker | per-release |
| Flake rate | доля CI-прогонов, затронутых flaky | < 2% | CI | weekly |
| Crash-free sessions | opt-in telemetry | > 99% (Ф2) / 99.9% (Ф3) | telemetry (10- observability) | weekly |
| Traceability coverage | RFC-0001 ID → зелёные тесты | 100% | L4 | every PR |

Ретро-каденция: QA-retro раз в 2 недели; квартальный red-team challenge corpus'а; деградация шумных гейтов — только через явное решение с ADR (см. `12-ADR-Glossary.md`), тихое отключение запрещено.

---

## 16. Traceability Matrix, Risks & Appendix

### 16.1. Traceability (генерируется, не ведётся руками)

Тестовые ID живут **в коде** (`#[case(id = "FS-UNIT-014")]`, pytest-маркеры); CI-джоба L4 на каждый прогон экспортирует матрицу в `docs/qa/traceability.md`. Формат: `RFC-0001 SLA ID → test IDs → CI stage → phase gate`. Отсутствие связи хотя бы у одного ID = красный PR. Матрица — артефакт, никогда не spreadsheet.

| SLA ID | Тесты (мин. набор) | CI stage | Phase gate |
|---|---|---|---|
| SLA-AUDIO-01 | AL-INT-001..003, E2E-02 | N1, W1 | Ф1 |
| SLA-AUDIO-02 | AL-UNIT-00x, AL-INT-004 | PR/N1 | Ф1 |
| SLA-ZSENSE-01 | ZS-SYS-001..006 | N1 | Ф1 |
| SLA-RAM-01/02/03 | RAM-профили 150/655/1450 | N1 | Ф1/Ф2/Ф3 |
| SLA-FRUST-01 | P-FS-01..10, E2E-02 | PR | Ф2 |
| SLA-ROUTER-01 | SR-UNIT-xxx (band 0.97–0.99), E2E-04 | PR/W1 | Ф3 |
| SLA-PII-01 | PI-UNIT/PI-ADV + canary | PR/N1 | Ф2 |
| SLA-FS-01 | SB-INT-01/02, SF-INT, E2E-04 | N1 | Ф2 |
| SLA-WIDGET-01 | SW-UNIT, FS-INT cooldown-кейсы | PR | Ф2 |
| SLA-GD-01 | CG-INT, E2E-05 | N1 | Ф1 |

### 16.2. Bug severity taxonomy

| Sev | Определение | SLA |
|---|---|---|
| **S1** | Утечка PII / побег песочницы / потеря данных пользователя | stop-ship, hotfix немедленно |
| **S2** | Barge-in > 200 мс (регрессия), RAM-пробой бюджета, router-ложный bypass | блокирует release; фикс в спринте |
| **S3** | Ложные срабатывания проактивности (не S1/S2), UI-дефекты | 2 спринта |
| **S4** | Косметика, документация | backlog |

### 16.3. Риски и открытые вопросы (сокращённый реестр)

| Риск | Митигция |
|---|---|
| Латентность меряется лог-в-лог, пользователи чувствуют лаг | риг на device-clock (раздел 6), manual spot-check на релиз |
| Golden-датасет однообразен (одна тема/DPI) → CI зелёный, прод валится | квоты diversity, field-capture refresh (раздел 7) |
| RAM-ночные «регрессии» — шум → гейт игнорируют | pinned hardware + статистика против versioned baseline (раздел 10) |
| Corpus только EN → RU-форматы утекают | EN+RU обязательные классы, квартальный red-team (раздел 8) |
| Flaky-аудио-тесты ретраятся до зелёного → lane пропускает реальную регрессию | quarantine-with-expiry, lane неотключаем |
| E2E зависит от живой LLM (недетерминизм) | VCR-фикстуры везде; живые контракты — fail-open отдельной lane |
| Slow-subscriber не протестирован → Z-Sense stalls | P-FB-03 как контракт |

Открытые вопросы (владелец — founder/QA lead): reference-конфигурация машин (одна или low-end + mid-range); операционное определение 200 мс; финальные OCR F1-пороги; ownership ASR false-accept (< 1/8 ч, provisional — вендор или мы); какие ОС blocking для B2C Beta; трактовка AI-Act transparency-теста (строковая проверка «представился как ИИ», [uncertain]); телеметрия в Ф1 или Ф2 (определяет измеримость crash-free KPI); публичный (OSS-Fuzz) vs приватный fuzzing Rust-core.

### 16.4. Стандарты — как применяются

- **ISO/IEC/IEEE 29119-3** — структура настоящего тест-плана, спецификация кейсов и трассируемости.
- **ISO/IEC 25010** — классификация KPI: performance efficiency (латентности, RAM), reliability (crash-free, recovery), security & **privacy как первоклассная характеристика** (PII-гейты), usability (WCAG).
- **OWASP WSTG v4.2 / MASVS** — abuse-case дизайн для MCP hub и sandbox (deny-by-default verification, API-тестирование IPC-границы).
- **OWASP Top 10 for LLM (2025)** — LLM01 prompt injection (SB-INT-07: OCR-контент как вектор инъекции), LLM02 sensitive disclosure (раздел 8).
- **NIST SP 800-160/800-53 (SC, SI, AC)** — чеклист-источник для v2.0 Enterprise; NIST AI RMF — рамка риска проактивного ассистента.
- **WCAG 2.2 AA / European Accessibility Act** — smoke с v1.0 (soft widget).
- **SemVer 2.0.0** — версии golden-датасетов, fixtures и самого документа; RFC-0001 цитируется по immutable-версии.
- **GDPR Arts. 5/25/32; 152-FZ** — тестируемые дефолты: минимизация, sanitizer-before-egress, retention/TTL, региональные endpoint'ы; IEEE 829 — словарь legacy-документов для enterprise-клиентов.

---

*Документ живёт в `docs/` монорепо рядом с RFC; его матрица трассируемости перегенерируется CI на каждый прогон — документация, тесты и спека остаются в одном контуре. Правки числовых контрактов — только через RFC-0001, не через этот документ.*
