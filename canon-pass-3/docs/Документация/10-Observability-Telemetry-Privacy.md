# 10. Observability, Telemetry & Privacy — Спецификация наблюдаемости, телеметрии и приватности

**Проект:** Archie (OS-Level AI Symbiote) · **Документ:** `10-Observability-Telemetry-Privacy.md`
**Статус:** PROPOSED · **Версия:** 1.0 · **Нормативный источник чисел:** RFC-0001 (Barge-in < 200 мс; VAD 85–100 мс; OCR ~200 мс / 185 мс эталон; RAM 150/655/1450 МБ; порог фрустрации 50; SOP-триггер ≥ 0.98; окно отмены 3–5 с).

**Назначение документа.** Archie — постоянно слушающий и читающий экран ассистент, поэтому доверие к нему держится на двух симметричных способностях: система должна *видеть себя* (здоровье воркеров, латентности, память) и *показывать пользователю, что она видит и что покидает устройство*. Данный документ описывает **целевую/дорожную** observability-архитектуру: local-first structured event logging, `WorkerHealthMetrics`, пользовательскую диагностику, debug replay, opt-in crash reporting, privacy-preserving product analytics, performance tracing и diagnostics bundle.

> **Текущий Phase 1 MVP против Target/Roadmap.** В Phase 1 реализованы локальные логи и SQLite-таблица `frustration_telemetry`; полноценная telemetry-подсистема этого документа (structured sinks, egress log, crash upload, analytics, support bundle/replay dashboard) **не реализована**. Crash upload, analytics и support bundle — только планируемые/условные opt-in. Поэтому формулировки ниже о `egress`, `WorkerHealthMetrics`, PII-гейтах и retention являются требованиями target/roadmap, а не доказанными гарантиями текущего MVP.

---

## 1. Цели и принципы (Local-First Observability)

### 1.1 Принцип «наблюдать ассистента, а не пользователя»

Ниже описан целевой контракт приватности, а не автоматически выполненная гарантия Phase 1: сырые кадры экрана и аудио предназначены оставаться на устройстве, а через облачный boundary допускается только санитизированный текстовый контекст при включённых и проверенных sanitizer/egress-контролях. Текущий MVP-санитайзер — базовый scaffold; его нельзя считать полной regex+NER-защитой без egress-тестов.

Классическая телеметрия отвечает на вопрос «что делает пользователь». Observability Archie отвечает на вопрос «что делает сам ассистент»: жив ли аудио-цикл, уложился ли Barge-in в 200 мс, сколько RAM удерживает RapidOCR после истечения TTL, почему Semantic Router выбрал этот SOP. Всё, что относится к первому вопросу (контент, нажатия клавиш, аудио, текст экрана), **конструкционно исключено** из любых выгрузок (см. раздел 12).

### 1.2 Target: Local-first наблюдаемость (MVP Path vs Roadmap)

> **Целевое консенсусное правило:** сырое аудио/screen frames должны оставаться local-only (санитайзер в path-ограничении). Текущий MVP реализует базовые кольцевые буферы и log-logging; полный целевой PII-санитайзер (target RFC-0001) ещё не в MVP.

- **Current MVP Phase 1:** logs и `frustration_telemetry` локально; нет реальных egress'ов, вся observability локально; `ArchiveCore-эвенты` записываются в SQLite.
- **Target Roadmap (Phase 2–4):** Разделение на consents, PII-санитайзер, валидация schema, egress-log перед сетью, fail-closed при отказе.

> **Target Architecture (Future)**  
> - **Границы и контроль:** консенсусные тумблеры `listening`, `screen context`, `cloud processing`, `telemetry` (§2.3.2, doc 03).
> - **Egress:** единая точка: `egress.rs`/`egress.py` → PII-санитайзер → allowlist validation → запись в `egress_log` **до** сетевого вызова → fail-closed при любой непройденной проверке.
> - **Current MVP does not have this yet.**

### 1.3 Что система обязана уметь ответить

| Вопрос | Источник сигнала |
|---|---|
| «Barge-in сейчас здоров?» | p95 `barge_in_latency_us` из аудио-цикла (те же таймстемпы, что sequence-спека RFC-0001) |
| «Почему этот SOP сработал / не сработал?» | replay записанной последовательности событий + бит-в-бит воспроизводимые косинусные значения |
| «Что было в краше #4123?» | локальный crash-артефакт, санитизированный перед любым upload |
| «Что Archie отправил обо мне на этой неделе?» | egress log на Privacy-панели дашборда |

### 1.4 Non-goals

Не входят в скоуп: полный threat model телеметричного канала (владелец — doc 03); реализация PII Sanitizer (владелец — RFC-0001 / архитектура; здесь — требования и тесты на телеметричном пути); целевые значения KPI (владелец — PRD/GTM; здесь — *как* они измеряются); A/B-testing инфраструктура; session replay видео; backend-архитектура аналитики (только ingestion-контракт).

---

## 2. Архитектура наблюдаемости (обзор)

Сигналы накладываются на существующую архитектуру без параллельных пайплайнов:

```
archie-event-bus (Rust, брокер) ──► log sink: JSONL ring + SQLite (event_log)
   ▲  EventEnvelope + observability-расширение          │
   │  (severity, correlation_id, span_id, ...)          ├─► WorkerHealthMetrics (1 Hz)
Rust-воркеры ── ReportWorkerHealth ──┘                  ├─► tracing spans (bounded 5 МБ)
Python-воркеры (archie-engine) ── StreamEvents ─────────┤
Tauri webview (ошибки) ────────────────────────────────┘
   все egress ──► egress.rs / egress.py ──► PII Sanitizer ──► allowlist schema ──► сеть (opt-in)
   всё локальное ──► main_dashboard: вкладка Diagnostics
```

- **Лог-запись = запись события.** RFC-0001 уже даёт типизированную шину с `EventEnvelope`; лог — это строгое расширение конверта, а не отдельный фреймворк. Правило «одного субстрата»: lint-гейт (`clippy disallowed-methods: println!, dbg!` + Python log-filter) запрещает неструктурированные логи в воркерах.
- `WorkerHealthMetrics` живёт в supervisor/health-модуле, который уже владеет жизненным циклом воркеров (heartbeat, рестарты) — расширяется семплированием RSS 1 Гц (Linux: `/proc/<pid>/statm`) и гистограммами латентностей.
- Дашборд — вкладка в существующем предсозданном скрытом окне `main_dashboard` (arch. doc: prevent_close() + hide(), 16 МБ в скрытом состоянии), а **не** третий webview (RAM-бюджет).
- PII Sanitizer (regex_rules + ner_onnx_masker, RFC-0001) — единственный санитайзер для всех путей: облачного LLM и телеметрии. Одна гарантия, два потребителя.

---

## 3. Structured Event Logging

### 3.1 Расширение EventEnvelope

Каждое событие, опубликованное в `archie-event-bus`, автоматически доступно как лог-запись. Расширение (не форк!) конверта, в том же контракте `contracts/events_envelope.schema.json`:

```jsonc
{
  "event_id": "evt_9f8a7d6c-2026-0515",        // RFC-0001, неизменно
  "timestamp_unix_us": 1747317000000000,        // RFC-0001, неизменно
  "event_type": "ZSENSE_GRAPH_READY",           // RFC-0001 enum
  "publisher_module": "zsense.json_graph_builder",
  "mission_state": "STUDY_FOCUS",
  "priority": 2,
  // ---- observability-расширение (все поля optional) ----
  "severity": "info",                            // trace|debug|info|warn|error (RFC 5424)
  "correlation_id": "turn_7c31...",              // id разговора/оборота (Barge-in связка)
  "span_id": "a41f", "parent_span_id": "77ba",   // tracing-контекст, см. §5
  "stage": "ocr",                                // vad|asr_partial|barge_in_decision|ocr|
                                                 // sanitizer|router|sop_match|memory|tts|mcp
  "duration_ms": 185,
  "outcome": "OK",                               // OK|REJECTED|TIMEOUT|FALLBACK|SCRUBBED
  "redaction_status": "sanitized",               // raw_local|sanitized|counts_only
  "payload_ref": "ring://evt_9f8a.../zsense"     // косвенная ссылка, НИКОГДА не inline
}
```

**Инвариант payload_ref (OWASP Logging Cheat Sheet):** сырой экранный текст и сырое аудио никогда не попадают в поля, читаемые нелокальным потребителем; payload хранится по ссылке в локальном кольцевом буфере. Экранные строки контролируемы атакующим (log injection) — поэтому поля строго типизированы схемой, а свободный текст допустим только через `payload_ref` с `redaction_status`.

### 3.2 Хранилище: JSONL ring + SQLite

- **Высокочастотный путь** (trace/debug, spans): JSONL-файлы с ротацией в `storage/obs/` — `events-YYYYMMDD.jsonl`, кольцевые буферы в RAM с лимитом **5000 записей / 8 МБ**; при переполнении — drop-oldest + инкремент счётчика `obs_ring_overflow` (никогда не сырые payload'ы).
- **Запрашиваемый путь** (info+, консент, egress, краши): таблица `event_log` в существующей `archie_core.sqlite3` (WAL), с ключом `event_id` из конверта, плюс `egress_log`, `consent_log`. Продолжение SQL-схемы RFC-0001:

```sql
CREATE TABLE event_log (
  event_id TEXT PRIMARY KEY, ts_us INTEGER NOT NULL,
  severity TEXT NOT NULL CHECK (severity IN ('trace','debug','info','warn','error')),
  event_type TEXT NOT NULL, publisher_module TEXT NOT NULL,
  mission_state TEXT, correlation_id TEXT, span_id TEXT,
  stage TEXT, duration_ms REAL, outcome TEXT, redaction_status TEXT NOT NULL,
  payload_ref TEXT, schema_version INTEGER NOT NULL
);
CREATE INDEX idx_evlog_ts ON event_log(ts_us);
CREATE INDEX idx_evlog_corr ON event_log(correlation_id);
CREATE TABLE egress_log (
  egress_id TEXT PRIMARY KEY, ts_us INTEGER NOT NULL,
  channel TEXT NOT NULL CHECK (channel IN ('crash_report','analytics_batch','support_bundle','replay_trace')),
  byte_size INTEGER NOT NULL, consent_basis TEXT NOT NULL,
  destination TEXT NOT NULL, redaction_status TEXT NOT NULL
);
CREATE TABLE consent_log (
  consent_id TEXT PRIMARY KEY, ts_us INTEGER NOT NULL,
  scope TEXT NOT NULL CHECK (scope IN ('crash','analytics','support_bundle','full_fidelity_capture')),
  action TEXT NOT NULL CHECK (action IN ('GIVEN','WITHDRAWN')), surface TEXT NOT NULL
);
```

### 3.3 Ротация и retention

Ротация JSONL: по размеру (лимът 64 МБ/файл, 5 файлов) и по суткам. **Политика удержания (предложено, ратифицируется с legal):** локальные логи — 30 дней по умолчанию (14 дней для payload-кольца), краш-репорты локально — 14 дней при выключенном consent (затем авто-очистка), egress/consent log — 90 дней. Очистка — заданием планировщика, а не политикой на бумаге. Приятный побочный эффект — v2.0 Enterprise: аудited-подмножество `event_log`/`action_audit_log` с hash-chain (`hash(prev)` в записи) даёт tamper-evident аудит-стрим без реархитектуры; поля actor/tenant едут в extension-полях конверта (forward-compat из PRD).

### 3.4 Стоимость записи (OBS-LOG-01)

При выключенном логировании — нулевая дополнительная сериализация (no-op subscriber в Rust `tracing`, no-op exporter в Python). При включённом info: ≤ 50 µs и ≤ 256 байт на запись (подтверждается criterion-микробенчмарком в CI до фазы коммита — числа proposal, hardware-pending).

---

## 4. Метрики здоровья: WorkerHealthMetrics

### 4.1 Реестр метрик (registry-as-code)

`observability/registry.yaml` в корне монорепо — единственный источник имён метрик для: Rust-кодогенерации (enum-имена), дашборда, CI-гейтов, приложения к PRD/GTM (аналитика, §9). Один файл — четыре потребителя; определение KPI в PRD не может молча разойтись с дашбордом.

### 4.2 WorkerHealthMetrics: точка исполнения RAM-бюджета RFC-0001

RFC-0001 уже определяет proto и rpc `ReportWorkerHealth(WorkerHealthMetrics) → MemoryControlDirective`. Архитектура контроля Lazy Loading построена на этом канале; данный документ определяет его наполнение и потребителей:

```proto
message WorkerHealthMetrics {          // RFC-0001, канон — не форкается
  uint32 pid = 1;
  uint64 rss_memory_bytes = 2;
  repeated string loaded_models = 3;   // TTL-состояния моделей (см. §4.4)
  bool audio_loop_healthy = 4;
}
```

Расширение на стороне supervisor (без изменения проволочного формата — производные поля):

| Метрика | Тип | Семплирование | Комментарий |
|---|---|---|---|
| `rss_mb` per worker | gauge | 1 Гц, EWMA-сглажено для дашборда, raw для CI | enforcement 150/655/1450 МБ по фазам |
| `event_loop_lag_ms` | gauge | 1 Гц | tokio metrics / asyncio lag |
| `bus_queue_depth` | gauge | 1 Гц | глубина очередей шины |
| `last_heartbeat_age_ms` | gauge | on-tick | таймаут смерти воркера — 5 с (proposal) |
| `restarts_total` per worker | counter | on-event | классификация «рестарт ≠ краш» — tuning open question |
| `worker_death` | synthetic event | on-timeout | прикрепляет последние 100 записей кольцевого буфера (pre-scrub) |

**Одна метрика — два потребителя:** CI-гейт RAM (см. §11) и дашборд читают одни и те же объекты — «два instrumentation-мира» (CI мерит не то, что показывает дашборд) спроектированы прочь.

### 4.3 Латентности по модулям (percentiles)

Окно агрегации — 5 минут, отчет p50/p95/p99. Точки измерения каноничны (RFC-0001 sequence — «нет вторых часов», чтобы SLA-споры QA ↔ дашборд невозможны):

| Метрика | Точка измерения (по RFC-0001) | SLA / CI-гейт |
|---|---|---|
| `barge_in_latency_us` | speech onset → сброс буфера TTS + state-change (шаги 3–6 флоу 1; поле `stop_latency_us` в `ForceStopAudioPayload`) | p95 < 200 мс; loopback-риг в CI |
| `vad_detection_ms` | → VAD-гейт (шаг 4) | 85–100 мс, точность > 95% |
| `ocr_latency_ms` | capture → result (`elapsed_ms` в `ZSenseGraphResponse`, шаг 3) | ~200 мс (185 мс эталон); CI: p95 ≤ 250 мс, hard fail > 300 мс (PRD split) |
| `ipc_roundtrip_us` | Rust ↔ Python gRPC (UDS / Named Pipe) | µs-класс; тренд-гейт на деградацию |
| `router_decision_ms` | перехват события → cosine-результат (флоу 3, t=+2…+15 мс) | регрессионный бейслайн |
| `sanitizer_ms` | PII-каскад на слепке (шаг 4, ~20 мс) | включается в общий OCR-путь |
| `tts_first_token_ms` | первый LLM-токен → первый аудио-чанк | p95 трекинг |
| `frustration_score` | гистограмма S(t) — **local-only** | decay-модель тюнинг |
| `frustration_gate_decision` | enum {suppressed, allowed} | **egressible** (безопасно для EU AI Act, §12) |

### 4.4 TTL-состояния моделей и audio loop health

`loaded_models` из `WorkerHealthMetrics` плюс supervisor-таблица TTL (RFC-0001: 60 с RapidOCR/NER, 120 с эмбеддер, 180 с STT/TTS, 1.5B — cold on disk) дают states `UNLOADED → LOADING → ACTIVE → TTL_IDLE → EVICTING` на душборд. `audio_loop_healthy = false` (срыв единого аудио-клока, рост `event_loop_lag`) — триггер деградации и пользовательского индикатора (§10.3).

### 4.5 RAM-бюджет самой наблюдаемости

Observability — не исключение, а строка бюджета: **≤ 10 МБ (Alpha) / 20 МБ (Beta) / 30 МБ (v1.0)** из 150/655/1450 МБ соответственно [proposal, микробенчмарк до фазы]. Кольцевые буферы ограничены, реестр метрик фиксированного размера. **Кардиальность метрик enum-only** (OBS-CARD-01): ≤ 50 значений на лейбл; свободный текст, заголовки окон и URL категорически запрещены как лейблы — это и anti-cardinality, и privacy-контроль. Подсистема телеметрии сама сидит строкой в health-таблице («собака на себе»).

### 4.6 Экспорт форматов

OTLP/JSON-совместимый формат (future-proof) и Prometheus text exposition на localhost-only endpoint — dev-удобство (Grafana против живого инстанса), не shipped-конфиг. Модель данных зеркалирует OpenTelemetry (Resource, SeverityNumber, Span, trace context) — без обязательства тащить OTel SDK.

---

## 5. Performance Tracing и бюджет трейсинга

Span-таксономия (фиксированная): `turn.barge_in`, `asr.partial`, `ocr.frame`, `sanitizer.pass`, `router.decide`, `sop.match`, `memory.write`, `tts.speak`, `mcp.call`, `staging.commit`. Контекст (`trace_id`, `span_id`, `parent_span_id`) распространяется через поля конверта — Rust-ядро и Python-воркеры вносят вклад в один трейс без side-каналов.

**Бюджет наблюдателя (OBS-TRACE-BUDGET):**
- ≤ 1% добавленной латентности к Barge-in p95 (< 200 мс — критический путь) и ≤ 50 µs на span [proposal; criterion-микробенчмарк до фазы];
- trace-буферы ≤ 5 МБ; переполнение — drop-oldest + counter, никогда сырые payload;
- режимы: **off** (default в Beta) / **default sampling** / **diagnostic**. Adaptive sampling: записывать «медленный хвост» (SLA-нарушения и ошибки — tail sampling) и семплировать быстрое тело 1:100 [proposal];
- `tracing` crate (Rust) + OTel Python API с локальным exporter — обе стороны в общий sink.

Патфол «инструментация ломает измеряемое» закрывается тем, что микробенчмарк overhead — CI-гейт с MVP Alpha.

---

## 6. Diagnostics Dashboard (вкладка в main_dashboard)

Вкладка **«Диагностика»** в существующем окне `main_dashboard` (не третий webview — RAM). Это продуктовая поверхность, а не инженерный дебаг-инструмент: критерий приемки — нетехничный пользователь отвечает на вопрос «что Archie отправил обо мне за неделю?» менее чем за 60 секунд. Легковесный чартинг (класс uPlot), refresh ≤ 500 мс, рендер поверх существующего IPC-моста `tauriEvents.ts`.

**Панели:**

1. **Health.** Bar-chart RSS по модулям **с линиями бюджета 150/655/1450 МБ прямо на графике**; sparklines p95 `barge_in` / `ocr` против SLA-линий (200 мс / ~200 мс); статусы heartbeat; TTL-состояния моделей (§4.4); рестарты.
2. **Privacy & Egress.** Append-only список всего, что покидало устройство: канал, таймстемп, размер, основание consent'а; текущие состояния переключателей. Тест полноты (OBS-DASH-01): network-stub в интеграционном тесте видит payload ⇔ в egress_log есть строка, 1:1.
3. **Event Log Viewer.** Фильтр по severity, поиск по `correlation_id`; уровень info по умолчанию, debug — за явным подтверждением «показать диагностическую деталь».
4. **Mission Timeline.** Таймлайн состояний миссии (GAMING_REST / STUDY_FOCUS / CODING_WORK / IDLE) по часам; live-просмотр потока событий шины.
5. **Replay.** Кнопка «воспроизвести недавнюю проблему» — запись трейса для саппорта (§7).
6. **Controls.** Мастер-выключатель телеметрии; отдельные переключатели crash / analytics / support bundle; **«purge all local logs»** (≤ 5 с, включая кольцевые буферы, SQLite-синки, crash-dir, analytics queue; сама операция логируется последней записью — документированная допустимая residue, OBS-ERASURE-01); экспорт diagnostics bundle (§8.5).

---

## 7. Debug Replay

**Формат записи:** сериализованная последовательность event-конвертов (те же записи лога) + input fixtures. Fixtures — синтетические заглушки аудио/OCR; записанный реальный контент допустим только в режиме `full_fidelity_capture` (user-initiated, time-boxed, видимо индицируется, авто-удаление) — это и legal, и trust-ограничение.

**Контракт детерминизма (OBS-REPLAY-01):** манифест записи фиксирует версии моделей, seed, платформу; повторный прогон записанной последовательности через semantic_router обязан воспроизвести решения роутера и косинусные значения SOP **бит-в-бит** (симметрия с гарантированным ≥ 0.98 порогом из RFC-0001: «почему выбрался SOP X» становится отвечаемым). Детерминизм поддерживается тестом, а не надеждой: CI-джоба replay-детерминизма на каждый PR, трогающий router/SOP.

**Инструмент:** CLI `observability/replay` потребляет diagnostics-bundle zip (§8.5) — один формат для саппорт-репродукции и CI-фикстур. «Slow-tail» трейсы из продакшена — класс регрессионных тестов: воспроизвести записанный плохой трейс → assert, что фикс изменил исход. Инструмент шарится с QA-доком (07).

---

## 8. Crash Reporting (Opt-In)

### 8.1 Точки захвата

Rust panic hook + backtrace (supervisor); Python `faulthandler` + top-level `sys.excepthook` в раннере каждого воркера; Tauri webview `unhandledrejection` / `window.onerror` → bus-событие; потеря heartbeat > 5 с [proposal] → синтетическое событие `worker_death` с последними 100 записями кольца (pre-scrub, скраб локально до любого возможного upload). Захват в течение 1 с (OBS-CRASH-01); ротация 5 репортов в локальном crash-dir.

### 8.2 Формат и санитизация (OBS-SCRUB-01)

```jsonc
{ "schema_version": 1, "install_id": "rot-...", "app_version": "…", "os": "…", "arch": "…",
  "crash_type": "rust_panic|python_excepthook|webview_rejection|worker_death",
  "stack_frames": ["…"], "scrubbed_message": "…",
  "runtime_context": { "worker": "zsense", "mission_state": "STUDY_FOCUS",
                       "uptime_s": 5412, "rss_mb": 152 },
  "attached_metrics_snapshot": { "…p95 за последние 5 мин…" } }
```

Схема **структурно запрещает** поля контента: нет дампов env, путей вне app-dirs, clipboard, заголовков окон, аудио, скриншотов — проверяется schema-тестом, не только скраббингом. Три слоя санитизации: PII Sanitizer (тот же инстанс, что для облачного пути) → entropy-fallback для токенов, похожих на ключи → **allowlist**: грузиться могут только schema-объявленные поля. Идентификатор — только ротирующийся `install_id` (никаких hardware fingerprints и стабильных cross-device ID).

**Adversarial PII suite (CI-гейт на телеметричном пути, не только LLM-пути):** золотой корпус крашей с RU/EN именами, телефонами, email, картами, адресами, токенами; цель скраббера — ≥ 99.5% recall [proposal] при ≤ 0.1% false-positive на benign-технических строках (пути, UUID, base64) — двойное пороговое условие против «пере-редактирования, убивающего диагностичность».

### 8.3 Consent UX

Опционально при первом краше, plain language: «этот отчёт будет содержать стек, версию и метрики; он НЕ будет содержать ваш экран, аудио и чаты». Показывается **реальный пример payload** (отрендеренный скрабленный репорт), не абстрактное обещание. Отдельные гранулярные тумблеры, снятие в один клик, каждая смена consent'а — запись в `consent_log`. При consent OFF репорты остаются локально 14 дней и авто-очищаются.

### 8.4 Upload и хостинг

Beskpoke allowlist-uploader поверх HTTPS (reqwest) — сознательно без тяжёлых SDK; симво­ликация офф-лайн в support-workflow. **Sentry-модель self-hosted:** эндпоинт краш-коллекции — self-hosted (Sentry-style) инстанс или никуда; для RU-инсталлов — только RF-хостинг либо канал выключен на уровне locale-gating (см. §12, open questions). Server retention 90 дней [proposal].

---

## 8.5. Diagnostics Bundle Export

Кнопка «экспорт диагностики» (support zip): `manifest.json` (версии схем, версий моделей, seed), редактированный event log (severity ≥ info), снапшот health-метрик, системная информация. Формат идентичен CI-фикстурам replay (§7) — один артефакт для саппорта и регрессий.

## 9. Privacy-Preserving Product Analytics (Opt-In)

Только **агрегированные счётчики и категориальные enum'ы**; никакого контента, нажатий клавиш, аудио, free-text полей. Каталог имён фиксирован registry.yaml:

- `onboarding_path_a_completed` / `onboarding_path_b_completed` (выбор Пути А — интервью / Пути Б — тихое наблюдение);
- `intervention_shown` / `intervention_accepted` / `intervention_cancelled` (Soft Widget, окно 3–5 с, `widget_cancelled_by_user` из `frustration_telemetry`);
- `frustration_gate_decision` ∈ {suppressed, allowed} — **не** распределение score;
- `mission_state_minutes` по бакетам часов (STUDY_FOCUS / CODING_WORK / IDLE / GAMING_REST);
- `router_decision` ∈ {local, cloud_small, cloud_frontier} (SOP-матчи vs облачные вызовы);
- `crash_report_uploaded`.

Формат батча: `{schema_version, day, install_id, events: [{name, bucketed_value?, count}]}`; агрегация на клиенте ежедневно, upload ≤ 1 раз/сутки, **fail-closed** (нет сети → payload дропается, не ретраится вечно). Перед выходом: small-cell suppression k ≥ 25 [proposal] (метод — count-only + suppression vs local DP — open question с ADR; рекомендация: start count-only, DP при v1.0 scale). Бэкенд: bespoke batch (рекомендация) > self-hosted PostHog in-region > vendor SaaS (последнее конфликтует с 152-FZ/SOC2 scope). Фаза: v1.0; нужен ли analytics в Beta вообще — open question.

## 10. Log Levels, Alerting

**Severity policy (RFC 5424):** `trace` — span-детали (только diagnostic-режим); `debug` — внутренние решения (VAD-вероятности, cosine кандидаты; за confirm на дашборде); `info` — все бизнес-события шины (default уровень логирования); `warn` — SLA-дрейф, TTL-аномалии, деградация; `error` — падения воркеров, неудачные egress. Production default: `info`.

**Пользовательские алерты (душборд + tray, не инженерные pages):**
- **Degraded mode indicator:** `offline_fallback_active = 1` (API_TIMEOUT → локальная 1.5B) — видимый индикатор «Archie в офлайн-режиме» (симметрия trust: п. arch. док. «прозрачно предупреждая»);
- **Disk space для vector store:** порог свободного места под `storage/vector_store.lancedb/` и WAL SQLite — предупреждение до деградации памяти Tier 3;
- Worker unhealthy (heartbeat / audio_loop_healthy = false), рестарты > N за час.

## 11. Фазирование и критерии приемки

| Фаза | Деливераблы | Acceptance |
|---|---|---|
| **1. MVP Alpha** | extension конверта, JSONL+SQLite sink, `WorkerHealthMetrics` 1 Гц, CI-гейты RAM/латентности, span-таксономия + overhead-микробенчмарк. **Никаких upload-каналов** | CI падает при RSS > +10% от 150/655/1450; loopback p95 Barge-in < 200 мс; OCR p95 ≤ 250 мс (fail > 300); tracing overhead ≤ 1% |
| **2. B2C Beta** | crash pipeline (opt-in), дашборд v1 (Health + Privacy), consent records | скрабленный payload проходит adversarial suite (99.5/0.1); egress-полнота 1:1 в network-stub тесте; не-инженер отвечает «что отправлено» < 60 с |
| **3. v1.0** | analytics (если ADR = yes), replay tooling, support bundle, adaptive sampling | replay бит-в-бит; analytics: schema-тест «нет контентных полей»; purge ≤ 5 с |
| **4. v2.0 Enterprise** | hash-chained audit subset, admin-visible health, enterprise retention config | integrity-checkable audit stream из того же субстрата |

Ишью-лейблы: `obs-eb`, `obs-health`, `obs-crash`, `obs-dash`, `obs-analytics`, `obs-replay`.

## 12. Privacy Impact Assessment: карта данных (сводная таблица)

Правовая база: **GDPR Art. 6(1)(a) consent** (все opt-in каналы), Art. 5 минимизация/ограничение хранения, Art. 17 erasure (install_id — единственный идентификатор → удаление тривиально: локальный purge ≤ 5 с + remote delete ≤ 30 дней), Art. 30 RoPA (egress map — вход), **Art. 35 DPIA** (систематический мониторинг экрана+аудио → DPIA обязателен до EU B2C Beta; эта таблица — технический аннекс DPIA). **152-FZ:** локализация ПД граждан РФ → RU-инсталлы либо полностью локальны, либо endpoint RF-хостинг; перекрестные передачи не допускаются молча — канал отключается.

| Данные | Где хранятся | Retention | Цель | Consent |
|---|---|---|---|---|
| Event log (envelope, без payload inline) | локально: JSONL + `event_log` SQLite | 30 дн (payload-ring 14 дн) | диагностика, replay | не требуется (не покидает устройство) |
| Payload по `payload_ref` (санитизированный экран/аудио-derived) | локальное кольцо 8 МБ | часы (ring) | отладка pipeline | local-only; full-fidelity — явный time-boxed consent |
| `frustration_telemetry`, mission timeline | локально SQLite | 30 дн | тюнинг decay-модели | local-only |
| Health metrics (RSS, latency p50–p99) | локально, real-time | окна 5 мин + лог | RAM/SLA enforcement | local-only |
| Crash report (scrubbed) | локально 14 дн; сервер 90 дн [proposal] | см. слева | стабильность | **opt-in, гранулярный** |
| Analytics batch (counters) | сервер 13 мес [proposal] | годовое сравнение | KPI (PRD/GTM) | **opt-in, k ≥ 25 suppression** |
| Support bundle / replay trace (sanitized) | передается вручную | по тикету | саппорт | **явный экспорт пользователем** |
| Consent records | локально `consent_log` | 90 дн | доказуемость (GDPR/152-FZ) | — |

Миноры (сегмент ЕГЭ): без возраста/вывода о возрасте; consent для несовершеннолетних — parental flow, политика шарится с PRD. EU AI Act: экспорт только gate-outcome, не score (риск emotion-recognition). v2.0 enterprise: employee-visible egress log остаётся видимым даже в managed mode.

## 13. Open Questions

1. Агрегация analytics: count-only + k ≥ 25 vs local DP (ε) — нужен ADR.
2. Нужен ли product analytics в B2C Beta вообще, или локальных логов+крашей достаточно.
3. RU-хостинг телеметрии (провайдер/стоимость) vs locale-gated disable; юридическая трактовка скрабленного metadata под 152-FZ — counsel.
4. Ратификация retention-чисел (14/30/90 дн/13 мес) с legal.
5. Точный observability-allowance 10/20/30 МБ и µs-overhead — микробенчмарк на целевом слабом железе, возможна ренегоциация с владельцем RAM-бюджета RFC-0001.
6. Heartbeat-таймаут 5 с; граница «рестарт vs краш».
7. Full-fidelity audio в replay vs синтетические фикстуры — legal input.
8. Бэкенд analytics (bespoke vs self-hosted PostHog) и SOC2-скоуп для v2.0.
