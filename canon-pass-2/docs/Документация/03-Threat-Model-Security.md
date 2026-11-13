# 03 — Модель угроз и спецификация безопасности (Threat Model & Security Specification)

**Проект:** Archie («Архи») — автономный проактивный ИИ-напарник уровня ОС (OS-Level AI Symbiote)
**Статус:** PROPOSED / READY FOR IMPLEMENTATION · **Версия:** 1.0.0 · **Связанные документы:** RFC-0001 (`../../ARCH`-уровень), PAD, `02-UX-Voice-UX.md`, `07-Testing-QA-Strategy.md`, `09-CI-CD-Distribution.md`, `10-Observability-Telemetry-Privacy.md`, `11-External-API-MCP-Contracts.md`

---

## 1. Введение и область действия

Настоящий документ определяет архитектуру безопасности платформы Archie **до** дистрибуции MVP Alpha внешним пользователям. Его задачи:

1. Дать STRIDE-обоснование каждой границы доверия архитектуры (Event Bus, Staging FS, MCP hub, cloud boundary, канал обновлений).
2. Превратить архитектурные намерения (PII Sanitizer, SOP sandbox, Permission Tiers, IPC 0600) в **проверяемые инженерные контракты** с критериями приёмки (format: `SEC-NNN`).
3. Разблокировать compliance-критичные решения: consent-UX для always-listening, GDPR/152-FZ, дизайн `action_audit_log`.

Без этой спецификации QA-команда не может написать adversarial-сьюты для SOP sandbox и PII Sanitizer, CI/CD-спека не может выбрать требования подписи/обновления, а сбор SOC 2-евиденций для Phase 4 (B2B) не начнётся вовремя. Документ препятствует классической ошибке «навешивания безопасности после фиксации четырёхфазной дорожной карты».

**В области действия:** целевая архитектура безопасности и её фазовые гейты для всех локальных подсистем клиента (Z-Sense audio/video-петля, OCR-конвейер, Event Bus, frustration scorer, память Tier 1–3, Semantic Router, SOP sandbox, MCP hub, staging FS, Tauri IPC), облачная граница API, канал автообновлений, хранение ключей, retention-политики и целостность аудита — от MVP Alpha до v2.0 Enterprise.

> **Current Phase 1 MVP vs Target/Roadmap.** Archie — гибридный edge+cloud ассистент, не fully-local. В текущем MVP есть локальные VAD/STT/Piper TTS, память, Z-Sense и базовый PII-санитайзер; `aijuju` cloud path configurable/opt-in, а `local`/`mock` дают офлайн-фолбэк. Сырые кадры экрана и raw audio предназначены оставаться локально; через cloud boundary допускается только санитизированный текстовый контекст при включённых и проверенных контролях. Текущий sanitizer — базовый scaffold, не полный regex+NER-конвейер; без egress-тестов это не безусловная гарантия. Telemetry, crash upload, analytics и support bundle в Phase 1 не реализованы и остаются planned/conditional opt-in. Остальные `SEC-NNN`, полноценный egress choke point и privacy controls ниже — target/roadmap requirements, если явно не помечены как MVP.

**Вне области действия:** безопасность backend/инфраструктуры за пределами API-границы (отдельная Backend Security-спека); внутренняя безопасность LLM-провайдера (покрывается контрактно, DPA); платёжные риски; general OS hardening хост-машины; физические атаки; enterprise SSO (Phase 4 addendum).

**Инвариант производительности (целевой).** Для целевой архитектуры RFC-0001 задаёт Barge-in < 200 мс, OCR ~200 мс (185 мс эталон) и RAM 150/655/1450 МБ; target-бюджет sanitizer — ≤ 20 мс p99 на cloud-boundary сообщениях, AST validation — ≤ 50 мс p99. Эти числа не являются доказанными SLA Phase 1 MVP: фактический RSS рабочего воркера с Qwen3-1.7B — около 2.4 ГБ, а full-duplex AEC3 отсутствует. Sanitizer не стоит на audio path; в MVP он является базовым scaffold на пути экранного контекста, а полноценный egress-контроль — target/roadmap.

---

## 2. Активы и цели безопасности

| ID | Актив | Место | C | I | A | Ущерб при компрометации |
|----|-------|-------|---|---|---|--------------------------|
| A1 | Raw audio buffer (кольцевой, RAM-only) | audio_loop | H | M | M | Утечка разговоров пользователя и третьих лиц в помещении |
| A2 | OCR-текст экрана (visible_text_snippet, JSON-граф Z-Sense) | zsense → Event Bus | H | M | M | Утечка всего видимого на экране: код, переписка, пароли |
| A3 | Память Tier 1–3 (SQLite / FTS5-trigram, Profile Rebuild) | storage/ | H | H | M | Полный поведенческий профиль пользователя, дневники и ночные сводки |
| A4 | SOP Library (compiled SOPs, blob Python scripts) | SQLite + VectorDB | M | **H** | H | Tampered SOP → arbitrary code execution |
| A5 | MCP tool credentials (Obsidian/Notion/Git/Jira/Slack tokens) | keyring/secret service | **H** | M | L | Access to external services on behalf of the user |
| A6 | Cloud LLM API keys (DeepSeek/OpenAI/Gemini) | OS keychain | **H** | M | L | Direct financial and quota abuse, key theft |
| A7 | action_audit_log (hash-chained) | storage/ | M | **H** | M | Потеря доказуемости действий; подмена истории Tier 2 |
| A8 | Frustration/emotion profile (frustration_telemetry) | SQLite, local-only | H | M | H | Выводы о состоянии здоровья/эмоциях (GDPR Art. 9-adjacent) |
| A9 | System Prompt persona «Старший брата» | persona_prompts.py | L | M | L | Кража IP персоны, вектор для инъекций |
| A10 | Привилегии исполнителя действий (archie-stagingfs, MCP hub) | Rust daemon | **H** | H | H | Compromised = произвольные действия с ФС и сервисами |

**Цели безопасности:** конфиденциальность A1–A3, A6, A8 (локальность по умолчанию); целостность A4, A7, A10 (детерминированность SOP, неизменяемость аудита); управляемость A10 (минимальные привилегии, deny-by-default). Метрика поверхности атаки: **никакого слушающего TCP-порта в дефолтной конфигурации; весь локальный IPC — same-user only; облачный egress ограничен доменами LLM-провайдеров и endpoint обновлений.**

---

## 3. Границы доверия и DFD

Топология привилегий (архитектурный инвариант): **Rust-ядро владеет всеми привилегированными решениями** (сетевой egress, ключи, executor, Staging FS); Python-воркеры (audio, OCR, router) — untrusted-by-default, общаются только через Event Bus, без сети и ФС вне staging-зон; оба Tauri-окна (widget-overlay, main_dashboard) не держат секретов и говорят с ядром через тот же schema-validated IPC.

```
  ┌────────────────────────────────────────────────────────────────────────┐
  │                          ОС ХОСТА (пользователь kamyka)                │
  │                                                                        │
  │  [User] ──(голос/клик)──► ┌─────────── Tauri v2 ────────────┐          │
  │    ▲                      │  soft_widget / approval_modal   │          │
  │    │ (T4: egress в облако │  main_dashboard (скрытый)       │          │
  │    │  только через        └───────────────┬──────────────────┘          │
  │    │  PII Sanitizer)                      │ T5: Tauri invoke / events  │
  │  ┌─┴──────────────────────────┐  ┌────────▼───────────────┐            │
  │  │  Cloud APIs (DeepSeek,     │  │  RUST DAEMON (Host)    │            │
  │  │  OpenAI, Gemini; HTTPS/TLS)│◄─┤  archie-shell          │            │
  │  └────────────────────────────┘  │  archie-event-bus ─────┼──┐         │
  │                                  │  archie-os-hooks       │  │ T3:     │
  │  ┌───────────────────────┐       │  archie-stagingfs ─────┼──┼─MCP hub │
  │  │ MCP Servers (Obsidian,│       │  secrets / audit       │  │         │
  │  │ Notion, Git, Jira/    │◄──────┴────────────────────────┘  │         │
  │  │ Slack; FastMCP)       │        T2: UDS 0600 / Named Pipe  │         │
  │  └───────────────────────┘        ACL + gRPC (tonic)         ▼         │
  │                                  ┌──────────────────────────────┐      │
  │  [OS Kernel: hooks, экран]──T1──►│ PYTHON ENGINE (archie-engine)│      │
  │  (Shared Memory archie_shm_      │ audio_loop / zsense / PII    │      │
  │   vision, seqlock)               │ memory / router / SOP sandbox│      │
  │                                  └──────────────────────────────┘      │
  └────────────────────────────────────────────────────────────────────────┘
        T1: device I/O → Z-Sense    T2: Event Bus / IPC
        T3: sandbox → MCP hub       T4: local → cloud LLM
        T5: UI → Rust core          T6: updater → app binary
```

Границы доверия (каждая раскрыта в STRIDE-таблицах §4):

- **T1** — I/O устройств → Z-Sense: микрофонный PCM, кадры экрана через Shared Memory (`archie_shm_vision`, 16 МБ, seqlock), OS-hooks.
- **T2** — Rust ↔ Python по Event Bus: UDS `/tmp/archie-ipc-{uid}.sock` (0600) / Named Pipe `\\.\pipe\archie-ipc-{session_id}` (ACL, `reject_remote_clients(true)`), gRPC-контракт `contracts/ipc_bridge.proto`.
- **T3** — SOP sandbox → MCP hub / staging FS: единственная поверхность вызова инструментов.
- **T4** — local → cloud LLM: HTTPS egress только из cloud_gateway, вход строго через PII Sanitizer.
- **T5** — Tauri-окна → Rust core: invoke/listen, схема валидируется ядром.
- **T6** — канал автообновлений → бинарник приложения (минисign/ed25519).

---

## 4. STRIDE-модель угроз

Риск = вероятность (1–5) × влияние (1–5). Бэнды: 1–6 Low, 8–12 Medium, 15–19 High, 20–25 Critical. Правило фазовых гейтов: **Critical и High должны иметь рабочую митигацию до выхода соответствующей фазы** (см. §16 и doc 07). Полная матрица (Appендix-расширение) ведётся в реестре и ревьюится на каждом phase gate.

### 4.1. Граница T1: устройство → Z-Sense

| ID | Компонент | STRIDE | Сценарий атаки | В | У | Риск | Митигация |
|----|-----------|--------|----------------|---|---|------|-----------|
| TM-001 | audio_loop | S | Сторонний процесс ОС имитирует микрофонный вход / внедряется в loopback | 2 | 5 | 10 M | Single-Process Audio Loop (RFC-0001); захват устройства через ОС-права пользователя; аудиоустройство не делится с другими процессами |
| TM-002 | audio_loop | I | Взвешенный вздох/фрустрация-профиль (A8) утекает в телеметрию или облако | 3 | 4 | 12 M | Эмоции — local-only: поля физически отсутствуют в telemetry schema (не фильтруются, а отсутствуют по конструкции); retention = decay-модель (полураспад 20/15/5/1 мин) |
| TM-003 | zsense/OCR | I | OCR-текст с паролями/ключами попадает в event bus без санитизации | 4 | 5 | 20 **Critical** | SEC-001: sanitizer inline на emitter (§5); ZSENSE_GRAPH_READY несёт `pii_sanitization` статус; шина local-only по конструкции RFC-0001 |
| TM-004 | shm_ring (Shared Memory) | T | Подмена кадра в `archie_shm_vision` между захватом и OCR | 2 | 4 | 8 M | seqlock-версия в `ShmFrameDescriptor.seq_lock_version`; регион same-user-only; несовпадение версии → повторное чтение/отказ |
| TM-005 | audio_loop | I | Always-listening пишет речь третьих лиц (семья, одногруппники) без их согласия | 4 | 4 | 16 **High** | Consent-дизайн §12: RAM-only кольцевой буфер ≤ 30 с (для barge-in нужно лишь ~200 мс хвоста), индикатор записи, опция pause-on-second-voice, push-to-talk fallback |

### 4.2. Граница T2: Event Bus / IPC (Rust ↔ Python ↔ Tauri)

| ID | Компонент | STRIDE | Сценарий | В | У | Риск | Митигация |
|----|-----------|--------|----------|---|---|------|-----------|
| TM-010 | ipc_server | S | Чужой процесс под той же ОС подключается к шине и инжектит `EventEnvelope` (spoofed publisher) | 3 | 5 | 15 High | SEC-010: UDS 0600 + SO_PEERCRED uid-проверка каждого коннекта (Linux); Named Pipe ACL current-user-only + `first_pipe_instance(true)` + `reject_remote_clients(true)` + `GetNamedPipeClientProcessId` (Windows); envelope `publisher_module` сверяется с фактическим peer'ом |
| TM-011 | event-bus | T | Malformed/concrafted envelope ломает десериализацию (gRPC/protobuf фаззинг) | 3 | 3 | 9 M | Валидация каждого envelope против `contracts/` JSON Schema / proto-границ до диспетчеризации; malformed → drop connection + audit event, никогда dispatch |
| TM-012 | event-bus | E | Публикация FORCE_STOP_AUDIO/`WIDGET_CANCELLED` от имени audio_loop для DoS голосового тракта | 2 | 3 | 6 L | Peer-credential привязка publisher; high-priority (priority=0) события принимаются только от аутентифицированных источников аудио-модуля |
| TM-013 | worker services | S | Фейковый «второй Archie» (запуск дублирующего процесса) перехватывает pipe/UDS | 2 | 4 | 8 M | Single-instance lock (именованный мьютекс в archie-shell); `first_pipe_instance(true)` отклоняет второй сервер |
| TM-014 | archie_shm_vision | I/E | Несанкционированный читатель разделяемой памяти с кадрами экрана | 2 | 5 | 10 M | ОС-хэндлы same-user-only; имя региона сессионно-уникально; перезапись слота по TTL |

### 4.3. Граница T3: SOP sandbox → MCP hub / Staging FS

| ID | Компонент | STRIDE | Сценарий | В | У | Риск | Митигация |
|----|-----------|--------|----------|---|---|------|-----------|
| TM-020 | sop_sandbox_runner | E | Escape из песочницы: dunder-доступ, `exec`/`eval`, bytecode через `compile()`, exception traceback-утечки | 4 | 5 | 20 **Critical** | §6: многослойная защита — AST whitelist (слой 1) + интерпретатор без достижимых I/O-примитивов (слой 2, запрет import/open/subprocess/socket by construction) + resource limits (слой 3) + tier check в MCP hub на каждый вызов (слой 4). Консенсус RestrictedPython-сообщества: AST-фильтр ≠ песочница — поэтому escape-корпус тестирует композицию, не парсер |
| TM-021 | blueprint_generator | T | Prompt injection в генерацию blueprint: облако (или скомпрометированный провайдер) возвращает SOP-текст с внедрённым небезопасным кодом | 4 | 5 | 20 **Critical** | §6.5: LLM-текст — только черновик; компилятор шагов транслирует их исключительно в типизированные MCP-вызовы и Tier 1–3 методы Action Engine; AST validation до исполнения; DRAFT-статус до явного [Утвердить рецепт] пользователем |
| TM-022 | smart_blueprints_sop | T | Локальный тампинг SOP-библиотеки (A4): изменённый `compiled_python_script` исполняется при ≥ 0.98 match | 2 | 5 | 10 M | Перевалидация AST + capability + tier **при каждой загрузке** из SQLite (не только при компиляции); подпись локального SOP-кеша; статус APPROVED_ACTIVE требует подтверждённого `approved_at` |
| TM-023 | semantic_router | E | «Tier squatting»: входной вектор-триггер сконструирован так, чтобы cosine ≥ 0.98 совпал с чужим Tier 2-рецептом и авто-исполнил его | 2 | 4 | 8 M | Tier рецепта не зависит от качества match: даже при BLUEPRINT_MATCHED_98 шаг Tier 2 проходит approval gate (флоу 3, шаг 6 RFC-0001); граница полосы 0.97/0.98/0.99 тестируется (doc 07, §9) |
| TM-024 | archie-stagingfs | E | Обход карантина: SOP добирается до системного вызова Delete | 3 | 5 | 15 High | §7.4: бан Delete реализован в исполнителе (archie-stagingfs), не в скрипте; любые операции удаления = перенос в `~/Archie_Staging` (SEC-030) |
| TM-025 | MCP hub | E | Скомпрометированный/вредоносный MCP-сервер (Obsidian/Notion/Jira-коннектор) запрашивает действия выше своего tier | 3 | 5 | 15 High | §11 MCP-контракты: каждый сторонний tool декларирует tier в манифесте; write-capability → минимум Tier 1 + пер-инструментальный allowlist; executor проверяет tier независимо от декларации |
| TM-026 | sop_sandbox | D | DoS: SOP-скрипт зацикливается / исчерпывает MCP-вызовы, вешает воркер | 3 | 3 | 9 M | Resource limits: wall-clock timeout 10 с, ≤ 5 MCP tool calls на прогон, лимит выходных токенов; abort без частичных side effects (атомарность шагов) |

### 4.4. Граница T4: local → Cloud LLM

| ID | Компонент | STRIDE | Сценарий | В | У | Риск | Митигация |
|----|-----------|--------|----------|---|---|------|-----------|
| TM-030 | cloud_gateway | I | Unsanitized PII уходит к провайдеру (санитайзер обойдён другим кодовым путём) | 4 | 5 | 20 **Critical** | §5.2: архитектурный choke point — тип `SanitizedText` с единственным конструктором в санитайзере; только cloud_gateway открывает сетевые соединения к LLM API; телеметрия идёт тем же путём |
| TM-031 | PII Sanitizer | I | Bypass санитайзера: гомоглифы, zero-width символы, смешанный RU/EN, разбиение токенов | 5 | 4 | 20 **Critical** | §5.3: нормализация Unicode (NFKC + strip zero-width/format chars + гомоглиф-маппинг) **до** regex/NER; Luhn/SNILS/INN-checksum валидаторы; adversarial corpus в CI |
| TM-032 | cloud_gateway | I | MitM / перехват TLS-трафика | 2 | 5 | 10 M | TLS 1.3, certificate pinning для доменов провайдеров; egress только к whitelisted доменам |
| TM-033 | cloud LLM | T | Ответ провайдера содержит инъекцию в действие (вредоносный tool output → SOP-текст → исполнение) | 4 | 5 | 20 **Critical** | §8: LLM-вывод никогда не диспетчеризуется напрямую — только propose actions через executor tier check; инъекция наследует все гарантии песочницы |
| TM-034 | cloud LLM | I | Retention данных провайдером (логирование промптов, обучение) | 3 | 4 | 12 M | §13: контрактные zero-data-retention условия; преференс провайдеров с enforceable ZDR (open question OQ-05) |
| TM-035 | inner_monologue | I | Эмоциональный профиль уходит в запрос «Внутреннего Монолога» в облако | 3 | 3 | 9 M | Inner monologue prompt содержит агрегированный счёт/эвристику, не raw-аудио и не сырые выводы emotion-модели |

### 4.5. Границы T5/T6: UI и канал обновлений

| ID | Компонент | STRIDE | Сценарий | В | У | Риск | Митигация |
|----|-----------|--------|----------|---|---|------|-----------|
| TM-040 | soft_widget / approval card | S | UI-спуфинг: карточка показывает «безобидное» описание, фронтит Tier 2-действие | 3 | 4 | 12 M | Approval card рендерит параметры из **канонического action-объекта исполнителя** (Rust), а не из фразировки LLM; `SystemActionRequest.approval_prompt_text` — только вербализация, canonical — `action_category` + `source_paths` |
| TM-041 | widget-overlay | E | TOCTOU между approval и исполнением: параметры действия меняются после клика «Пушим?» | 3 | 5 | 15 High | Approval token: mint по клику, **ре-валидация параметров в executor в момент исполнения**; токен одноразовый; 3–5 с countdown отменяет токен без частичных эффектов |
| TM-042 | updater | T | Подписанное малеварем обновление / понижение версии (downgrade) | 2 | 5 | 10 M | §11: minisign/ed25519, публичный ключ вшит в бинарник, отказ downgrade, kill-switch; updater — отдельная STRIDE-граница с рейтингом Critical для ключа подписи (TM-043) |
| TM-043 | signing key | E/R | Компрометация ключа подписи обновлений | 2 | 5 | 10 M | Оффлайн-хранение ключа (YubiKey/зашифрованный том + печатная копия), ротация, runbook §14.3 |

### 4.6. Сводная матрица рисков (флагманские attack trees)

```
Sandbox Escape (TM-020/021)                       Consented-Mic Exfiltration (TM-001/005/030)
├─ Инъекция в blueprint-генерацию ──► AST reject  ├─ Bypass санитайзера ──► NFKC+checksum+corpus
├─ Dunder/traceback escape ─────► нет I/O-примит. ├─ Сторонний слушатель буфера ─► same-user, RAM 30с
├─ Динамический import/exec ────► запрет by constr.├─ Утечка в телеметрию ────► поле отсутствует в схеме
└─ TOCTOU tier ─────────────────► executor re-check└─ Облачный ретеншн ───────► ZDR-контракт
```

---

## 5. PII Sanitizer: спецификация и анализ угроз bypass

**Модули:** `services/archie-engine/src/archie_engine/security/` (`pii_sanitizer.py`, `regex_rules.py`, `ner_onnx_masker.py`); Rust regex-биндинги + квантованная ONNX NER (`models/security/ner_pii_int8.onnx`). Замена на теги `[REDACTED_API_KEY]`, `[REDACTED_FINANCIAL_DATA]`, `[REDACTED_PII_*]`.

### 5.1. Область детекции (RU/EN dual-market corpus)

- **Форматы с checksum-валидаторами (бар: 0% пропусков):** PAN банковских карт (Luhn), SNILS (контрольная сумма), INN, паспорт РФ (серия/номер), IBAN. Checksum-валидатор, не просто regex — пропущенный «почти-валидный» токен не является валидным секретом.
- **Regex-классы:** email, RU-телефоны (+7/8, форматы с пробелами/скобками), адреса (RU/EN паттерны), заголовки private key (`-----BEGIN`), AWS Access Key IDs/Secrets, GitHub/Slack токены, `password =`/`api_token =`-паттерны (как в эталоне `# api_token = [REDACTED_API_KEY]`).
- **NER-классы (бар: < 0.5% пропуска на корпусе к Beta):** имена (кириллица + латиница), организации, гео-локусы; MI-адреса.

### 5.2. Ordering guarantee (архитектурный инвариант — SEC-001)

Ни одно сообщение Event Bus и ни один LLM-запрос не может достичь cloud boundary **кроме как через санитайзер**. Реализация: cloud-boundary emitter (`cloud_gateway`) принимает на вход только тип `SanitizedText`, конструируемый единственным конструктором санитайзера — принудительно type system'ой и код-ревью-правилом. Тот же sanitized path обязателен для телеметрии. Шина local-only по RFC-0001 — несанитизированные потоки физически не доходят до сети. Проверяется инвариант-тестом: поиск всех сетевых вызовов вне egress-модуля (Semgrep taint-правило) + property-тест «никакой путь не строит LLM-запрос из raw-типа».

### 5.3. Adversarial-анализ bypass-сценариев

| Вектор | Сценарий | Контрмера |
|--------|----------|-----------|
| Гомоглифы | `Аpple_Secret` (кириллическая А), латиница/кириллица-mix в токене | Гомоглиф-нормализация (confusables-маппинг) до детекции; после замаскирования — восстановление не требуется |
| Zero-width / format chars | `sk‑live​ABC...` | Strip Unicode-категорий Cf/Cs/Zs перед regex; NFKC-нормализация |
| Смешанный RU/EN | Номер телефона в русской раскладке, email с транслитом | Явные паттерны для обоих алфавитов + нормализация транслита в high-risk классах |
| Разбиение токена | `4111-1111-1111-` / `1111` на разных OCR-строках | Склейка OCR-lines в порядок чтения (XY-cut) до санитизации; regex по window'у всего слепка, не по строке |
| Canary-токены | Злоумышленник не участвует — canary'ы ставим **мы** в тестах: помеченные «PII»-токены в golden-скриншотах | CI-гейт: полный network capture в тестовом прогоне + поиск canary в egress; любая утечка = S1, release-blocking (doc 07 §8) |
| Проприетарный код | Код компании пользователя уходит в промпт | Классификация контекста: тег кода `project_name`-scoped; enterprise-режим (v2.0) — запрет cloud-пути для кодовых сниппетов вне on-prem |

**Residual-risk statement:** NER вероятностный; PII во внетекстовых каналах (числа без формата, имена в никнеймах) может проходить. Бюджет утечки: high-sensitivity классы (PAN/SNILS/passport/API keys) — **0%**, прочие — < 0.5% на курируемом RU/EN-корпусе к Beta. Остаточный риск декларируется в DPIA и покрывается ZDR-контрактами провайдера.

**SEC-001** (choke point), **SEC-002** (0% high-sensitivity), **SEC-003** (< 0.5% NER), **SEC-004** (sanitizer ≤ 20 мс p99, не на аудио-пути). Верификация: adversarial corpus + canary + network capture, гейт CI (см. §16, doc 07 §8).

---

## 6. SOP Sandbox: многослойная спецификация

**Модули:** `router_and_sops/sop_sandbox_runner.py`, `ast_validator.py`; MCP hub — единственная поверхность вызова инструментов; staging FS — единственная файловая поверхность.

### 6.1. Порядок пайплайна исполнения

```
LLM-текст черновика (untrusted!) → parse (ast.parse)
  → [1] AST-whitelist валидация (любой узел вне списка = hard reject)
  → [2] Capability binding (вызовы только к whitelisted MCP namespace + Tier 1–3 методам Action Engine)
  → [3] Tier projection (max-tier SOP ≤ tier сессии; Tier 2 шаг → approval gate)
  → [4] Staged execution (staging FS, rollback при нарушении)
  → [5] Audit event (action_audit_log)
```

Cached SOP из SQLite/LanceDB проходит тот же пайплайн **при каждой загрузке** (перевалидация — TM-022).

### 6.2. Слой 1 — AST whitelist (структурная валидация, allow-list, не block-list)

**Разрешённые узлы:** `Module`, `FunctionDef`/`AsyncFunctionDef` (без декораторов), `arguments`, `Assign`/`AugAssign`/`AnnAssign`, `Return`, `If`/`Elif`/`Else`, `For`/`While` (с `break`/`continue`), `Compare`, `BoolOp`, `BinOp`/`UnaryOp`, `Call` — только к whitelisted namespace, `Constant` (str/int/float/bool/None), `Name`, `Tuple`/`List`/`Dict` литералы, `f-string` (без format-spec вызовов), `Expr` (только Call).

**Явный deny-list (hard prohibitions):** `Import`/`ImportFrom` (любые), `Exec`/`Eval`, `Lambda` (опционально: allow, если слой 2 надёжен — ADR), `Attribute`-доступ к dunder/`_private`, subscript к `__`-именам, `Global`/`Nonlocal`, comprehensions с `Attribute`, `Try`-блоки, изымающие traceback-объекты (`__traceback__`, `sys.exc_info` недостижимы), `compile()`/`__builtins__`/`getattr`/`setattr`/`globals()`/`locals()`/`vars()`/`dir()`/`type()`/`object.__subclasses__` — весь refleak-поверхностный builtin-набор.

### 6.3. Слой 2 — интерпретатор без достижимых I/O (by construction)

Ключевой принцип (консенсус OSS-практик, см. RestrictedPython-уроки): **безопасность объявляется от capability surface, а не от валидатора**. Среда исполнения конфигурируется так, что `open`, `os`, `sys`, `subprocess`, `socket`, `importlib` недостижимы вовсе: кастомный `__builtins____` словарь, заблокированный import-механизм, отсутствие файловой системы кроме staging-зоны (worker-scoped temp dirs, same-user, no exec bit, auto-clean, rollback), отсутствие сети в воркере песочницы. Выбор CPython-restricted vs embedded interpreter vs WASM/wasmtime — отдельный ADR (документ 12; expressiveness SOP против provable isolation).

### 6.4. Слои 3–4 — resource limits и tier check на MCP hub

- Wall-clock timeout **10 с** (default), ≤ **5 MCP tool calls** на SOP-прогон, лимит output-токенов; при abort — deny-by-default, **без частичных side effects** (шаги атомарны или идемпотентны).
- Каждый tool call из песочницы проходит **tier check в MCP hub** (§7) — инъекция не может миновать его by design.
- Метрика: **100% исполняемых SOP проходят AST-валидацию; 0 escape'ов на 500-скриптовом adversarial corpus** (escape-тесты нацелены на композицию слоёв).

### 6.5. Prompt injection в blueprint-генерацию (TM-021)

LLM-вывод — untrusted черновик: он не исполняется, а рендерится в `BlueprintDiffViewer.tsx` с кнопками [Утвердить]/[Изменить]. Компилятор шагов (`sop_sandbox_runner`) транслирует шаги **исключительно** в типизированные вызовы MCP-инструментов и методов Action Engine Tier 1–3 — текст шага не может стать кодом. Даже скомпрометированный промпт не производит SOP, обходящий `Archie_Staging` или запрет Delete (SEC-030). Утверждённый рецепт = смена статуса DRAFT → APPROVED_ACTIVE с фиксацией `approved_at` и векторизацией триггера.

---

## 7. Permission Tiers: инварианты enforcement

### 7.1. Модель

| Tier | Определение | Примеры (архитектура PAD/ARCH) | UX |
|------|-------------|--------------------------------|-----|
| **Tier 0** | Сенсорика/чтение без побочных эффектов | OCR, VAD, frustration scorer, чтение логов | невидимо для пользователя; consent-тумблеры §12 |
| **Tier 1** (TIER_1_AUTONOMOUS) | Автономные безопасные действия | MEDIA_CONTROL (переключение музыки), поиск в сети, чтение vault Obsidian (read-only) | выполняется без спроса, логируется |
| **Tier 2** (TIER_2_CRITICAL_APPROVAL) | Действия с утверждением | удаление файлов (→ staging), отправка писем, git commit/push, notion/jira/slack write | карточка Critical Approval |
| **Tier 3** (TIER_3_WORKSPACE_ORCHESTRATION) | Управление окружением по явному запросу | «Я хочу учиться»: закрытие вкладок, открытие IDE/PDF, Lo-Fi | инициируется пользователем, оркестрация через workspace_orchestrator |

Default deny: действие без однозначного маппинга класс→tier отклоняется и логируется.

### 7.2. Единственная точка enforcement — SEC-020

Tier проверяется **только в pre-dispatch hook исполнителя** (archie-stagingfs / SystemActionRequest-обработчик Rust-ядра), ниже всех вызывающих (router, SOP, MCP, UI). UI-карточка — лишь UX; баг роутера или MCP-пути не может исполнить Tier 2 без проверки. Верификация: property-тест над enum классов действий — 100% Tier 2-путей проходят approval; Semgrep-правило «ни один вызов stagingfs/MCP-write вне executor».

### 7.3. Approval card контракт и TOCTOU (SEC-021)

Approval card рендерит параметры из канонического action-объекта исполнителя (TM-040). Подтверждение минтит **одноразовый approval token**, привязанный к снапшоту параметров (`action_category`, `source_paths`, target). В момент исполнения executor **ре-валидирует** параметры против токена — смена путей между approval и execution = отказ + audit. Countdown 3–5 с (CancelCountdown.tsx) — security control: отмена инвалидирует токен, действия идемпотентны/атомарны → отмены без частичных эффектов. Открытый вопрос OQ-01: достаточно ли 3–5 с для быстрых, но необратимых действий (send message) — возможно, усиление вторым подтверждением (UX+security, документ 02).

### 7.4. Бан Delete-системного вызова — SEC-030

Физический запрет syscall Delete реализован в исполнителе `archie-stagingfs` (Rust): любая операция «удаления» = перемещение в `~/Archie_Staging` (пользователь чистит сам). Инвариант приёмки: **100% убираемых с рабочего стола файлов переносится в Archie_Staging, вызов Delete недостижим из песочницы и SOP по construction**. Tier-escalation prevention: SOP объявляет `permission_tier` при утверждении; session-tier не может быть превышен (§6.1, шаг 3); elevation-сценарии (confused deputy — TM-040/041, tier squatting — TM-023, TOCTOU — TM-041) покрыты тестами doc 07 §9.

---

## 8. IPC Security (RFC-0001 контракт как enforced security contract)

**SEC-010 — Linux (UDS):** `/tmp/archie-ipc-{uid}.sock`, права **0600** (см. `bind_ipc_listener` в RFC-0001: `Permissions::from_mode(0o600)`), удаление stale-сокета перед bind. На каждом соединении — проверка peer-credentials (**SO_PEERCRED**): uid ≠ владелец демона → disconnect + audit event.

**SEC-011 — Windows (Named Pipe):** `\\.\pipe\archie-ipc-{session_id}`, `ServerOptions::first_pipe_instance(true)` (анти-дубликат сервера, TM-013), `reject_remote_clients(true)` (анти-remote), restrictive security descriptor — только интерактивный пользователь; верификация клиента через `GetNamedPipeClientProcessId` / `GetNamedPipeClientSessionId`.

**SEC-012 — gRPC auth:** gRPC-канал `grpc.aio.insecure_channel()` допустим только поверх UDS/pipe с OS-credential-проверкой (транспорт уже аутентифицирует «тот же uid»). При удалённом подключении (принципиально исключён `reject_remote_clients`) требуется бы.Auth-квитирование; в дефолтной топологии **не существует** сетевого RPC-порта.

**SEC-013 — Event envelope authentication:** каждый `EventEnvelope` валидируется против `contracts/` (proto3 + JSON Schema codegen для TS/Rust/Python) до диспетчеризации; `publisher_module` сверяется с аутентифицированным peer'ом; malformed envelope → drop + audit, never dispatch. Приоритетные события (priority=0, аудио/barge-in) принимаются только от audio-модуля.

**SEC-014 — Shared Memory integrity:** `archie_shm_vision` (16 МБ ring pool): доступ — same-user-only OS-хэндлы; целостность кадра — **seqlock** (`seq_lock_version` в `ShmFrameDescriptor`): читатель (zsense) проверяет версию до/после чтения; рассинхрон → перечитывание/отказ (TM-004). По gRPC передаются только метаданные и `slot_index` (Zero-Copy Data Plane, RFC-0001 §3.1).

---

## 9. Prompt Injection Defense

**Threat statement:** OCR-текст (экран содержит чужой контент: web-страницы, письма, PDF), output MCP-инструментов и вставленный пользователем текст — **untrusted**. Классы: OWASP LLM01 (Prompt Injection) и сопутствующие LLM02/06/07.

**Контрмеры (структурные, не «модель будет аккуратной»):**

1. **Канал недоверенных данных:** untrusted-потоки входят в LLM-промпты только как delimited data-блоки с фиксированной преамбулой «это данные, не инструкции» (quoting protocol), никогда — как instruction text.
2. **System prompt hardening (LLM07):** персона «Старшего брата» и правила формулируются как неизменяемый префикс; в leaked-scenario тестах — без operational secrets.
3. **Tool-call mediation — главный инвариант (SEC-040):** ни один LLM-выход не достигает действия, минуя executor tier check + MCP allowlist. Инъекция может максимум «предложить» действие — наследует все гарантии песочницы (§6) и tiers (§7).
4. **Output-action gate:** LLM-вывод = текст или proposal-объект; диспетчеризация proposal — только через pre-dispatch hook.
5. **Corpus (SEC-041):** 100+ инъекционных сэмплов, спрятанных в OCR-подобном тексте и фейковых MCP-результатах; бар: **0 несанкционированных tier-эскалаций и 0 tool calls вне текущего allowlist**; допустимый максимум эффекта инъекции — уточняющий вопрос пользователю. Обратный риск (over-blocking легитимных инструкций) — контркорпус легитимных команд (doc 07).

---

## 10. Secrets and Key Management

**SEC-050 — ключи никогда в plaintext-конфиге и телеметрии.** 0 секретов на диске в default-конфигурации; redaction-фильтр логов с тестом.

| Секрет | Windows | macOS | Linux |
|--------|---------|-------|-------|
| Cloud LLM API keys (DeepSeek/OpenAI/Gemini) | DPAPI / Credential Manager | Keychain | libsecret / Secret Service (gnome-keyring) |
| MCP tool credentials (Obsidian/Notion/Git/Jira/Slack) | DPAPI | Keychain | libsecret |
| Ключ шифрования memory DB (SQLCipher-класс) и audit log | DPAPI-derived | Keychain-wrapped | Secret Service-wrapped; **fallback-политика**: headless/без secret service — open question OQ-02 (plaintext с предупреждением vs отказ от шифрования vs login-passphrase; ADR в документе 12) |

Ротация: API keys — через UI settings (старый ключ затирается в keychain); audit-log HMAC-ключ — пер-пользовательский, ротация при экспорте/миграции. Модели `models/` и GGUF-артефакты — целостность через подписанный manifest канала обновлений (§11; open question OQ-04: общий manifest vs отдельный model registry).

---

## 11. Secure Update Channel

**SEC-060 — все артефакты обновлений подписаны (ed25519/minisign, Tauri v2 updater):**

1. Подпись каждого артефакта проверяется **до** применения; публичный ключ вшит в подписанный бинарник; сетевой загрузки ключей **нет** (no network-fetched keys).
2. Подписанный update manifest (версии, хэши, минимальная требуемая версия).
3. **Отказ downgrade:** версия ниже текущей установленной — отклоняется (анти-rollback).
4. **Kill-switch:** серверный флаг отключения auto-update на случай компрометации (runbook §14.3).
5. Хранение приватного ключа: оффлайн (YubiKey / зашифрованный том + печатная восстановительная копия); **не** в CI-секретах в одиночку; подписание — manual gate job. Runbook компрометации ключа: kill-switch → отзыв канала → выпуск signed advisory → ротация ключа → принудительное обновление.
6. egress домены обновлений — в whitelist сети (§2, метрика поверхности атаки). Детали пайплайна — документ 09 (CI/CD).

---

## 12. Data Retention, Deletion и Consent-дизайн (always-listening)

**Retention-матрица:**

| Артефакт | Класс данных | Хранение | Шифрование | Retention | Путь удаления |
|----------|--------------|----------|------------|-----------|----------------|
| Raw audio ring buffer | A1 | RAM-only | — | ≤ 30 с (для barge-in < 200 мс нужен лишь хвост ~сотни мс) | автоматический кольцевой перезапуск |
| OCR-текст / Z-Sense граф | A2 | Tier 1 (RAM/SQLite) | SQLCipher-класс | local-only; TTL шины + 2–4 ч буфер сессии | purge по expires_at |
| Tier 1 session memory | A3 | RAM + SQLite WAL | да | 2–4 ч | автоматический expire |
| Tier 2 nightly summaries | A3 | SQLite + Obsidian/Notion (MCP) | локально — да; в Obsidian — под контролем пользователя | бессрочно у пользователя | ручное удаление записи |
| Tier 3 knowledge graph | A3 | LanceDB (mmap) | да | до erasure-запроса | erase node по `vector_embedding_id` |
| frustration_telemetry | A8 | SQLite | да | локально; привязано к decay-модели (полураспад 20/15/5/1 мин), агрегаты — по политике | erasure ≤ 24 ч локально |
| action_audit_log | A7 | append-only файл | да | бессрочно (локально) | экспорт + полный wipe при uninstall |

**SEC-070 — Erasure (GDPR Art. 17 / 152-FZ):** пользовательское «удалить всё» завершается локально ≤ 24 ч (SQLite + LanceDB + staging + audit-wipe с сохранением хэш-цепочки-эпитафа); облачная сторона — по ZDR-условиям провайдера (OQ-05). Right-to-export — экспорт JSON/Markdown из MemoryExplorer.

**Consent-дизайн always-listening (SEC-071):** гранулярные независимые opt-in тумблеры — **listening / screen context / cloud processing / telemetry** (GDPR Art. 6(1)(a)+7(3): отзыв одним действием, легко как выдача). Отключение listening → деградация в push-to-talk с сохранением SOP/экрана (acceptance: выключение слуха **не** ломает прочие функции). Отключение cloud → локальная 1.5B (Graceful Degradation) с ясным уведомлением. Отключение screen context → ручная вставка контекста. Индикатор записи постоянен при активном буфере; опция «пауза при детекции второго голоса» (non-enrolled speaker) — Beta-обсуждение (OQ-03). Путь Б онбординга («тихое наблюдение») — тоже осознанный opt-in с перечислением собираемых классов данных (Art. 13 transparency). Память: L0/RAM — raw допустим; L1 (session SQL) — sanitized-by-default (raw только для явно user-pinned); L2/L3 — sanitized only.

---

## 13. Audit Log Integrity (action_audit_log)

**SEC-080 — события:** 100% Tier 2-грантов (APPROVED/REJECTED), Tier 3-оркестраций, sandbox-отказов (AST reject, capability violation, resource abort), tier-эскалаций/попыток, IPC-spoofing-отключений (peer uid mismatch), sanitizer-bypass-детектов, update-событий. Схема расширяет таблицу `action_audit_log` из ARCH (`action_id`, `permission_tier`, `action_category`, `user_approval_status`, `execution_result`...).

**SEC-081 — append-only + hash chain:** каждая запись содержит `prev_hash` + собственный хэш с **HMAC локальным per-user ключом** (ключ в keychain, §10). Запись — только дописывание; проверка цепочки при каждой загрузке (`archie audit verify`, CLI-подкоманда) и при экспорте; разрыв цепи → пользовательское предупреждение + фиксация факта в новой голове цепочки (эпитаф).

**Производительность (не на hot path):** асинхронное батчевое хэширование — цепь «пломбируется» каждые N событий или T секунд, **не** на событие аудио-пути (иначе угроза Barge-in < 200 мс). PII-free by construction: логируются типы событий, ID и пути (не payload-тексты); при экспорте для enterprise — scrubbing-правило. Budget ≤ 5 МБ RSS (в составе 15 МБ security-бюджета §1).

---

## 14. Incident Response

1. **Severity-классы:** S1 — утечка PII/секретов, sandbox escape, компрометация ключа подписи; S2 — bypass tiers без утечки, DoS ядра; S3 — прочее.
2. **Kill-switch:** (a) отключение auto-update (серверный флаг); (b) глобальный «стоп действий» — executor переходит в Tier 2-only режим (каждое действие требует approval); (c) отключение cloud egress (offline-режим).
3. **Runbook: компрометация update-ключа** (§11.5) и **runbook: обнаруженный sandbox escape CVE** — freeze SOP-исполнения (BLUEPRINT_MATCHED_98 игнорируется, всё через DRAFT+approval), hotfix-канал, advisory.
4. **Уведомление пользователей:** S1 — в приложении и на канале рассылки ≤ 72 ч; раскрытие уязвимостей — координированное, 90-дневное окно (политика bounty — OQ-06).
5. Контакт: security@ (публичный mailbox, задаётся в документе 09/13).

---

## 15. Compliance Mapping (GDPR / 152-FZ)

| Контроль | GDPR | 152-FZ (РФ) | Прочие |
|----------|------|-------------|--------|
| Local-first, RAM-only буферы, минимизация | Art. 5(1)(c), 25 | ст. 6 (соразмерность), 18–21 (обязанности оператора) | NIST AI RMF: Privacy-by-design |
| Consent: 4 гранулярных тумблера, отзыв одним действием | Art. 6(1)(a), 7(3) | ст. 9 (согласие, отзыв) | COPPA-adjacent: возрастной гейт (OQ-07) |
| PII Sanitizer + ZDR-контракты провайдеров | Art. 32, 44–49 (transfer) | локализация первичных БД (RFC: local-first без серверного PII-хранилища) | SCC/DPF при EU→US |
| Erasure ≤ 24 ч + export | Art. 17, 20 | ст. 21 (уничтожение) | SOC 2 Privacy (Phase 4) |
| DPIA: систематический мониторинг экрана+голоса | Art. 35(3)(c) — фактически обязателен до EU-Beta; вход = §2–4 этого документа + inventory | — | CNIL PIA-методология |
| Frustration/emotion confinement (local-only, retention=decay, нет третьих сторон) | Art. 9-adjacent (риск выводов о здоровье) | категории спец. данных — избегаем | |
| Audit log + SOC 2-евиденции | Art. 32 (логирование) | ст. 19 (защита) | SOC 2 Security (Type II — Phase 4), ISO 27001 Annex A vocab |
| Транспарентность ИИ (маркировка в онбординге) | Art. 13/14 | ст. 10–11 (уведомление РКН при старте обработки) | EU AI Act: limited-risk transparency tier (классификация uncertain) |

**152-FZ open item (OQ-08):** relay санитизированных сниппетов к иностранному провайдеру — правовая серая зона (статус оператора, кросс-бордер); решение блокирует RU-Beta go/no-go; cloud-provider abstraction спроектирован так, чтобы рынок мог сменить провайдера/регион без смены архитектуры. Механизм юридических чекпойнтов: пер-рыночные Open Questions с owner'ом и фазой (см. реестр §17) — инженерные решения держатся реверсивными до вердикта юрисконсульта.

---

## 16. Security Testing Plan (хуки в документ 07)

Требования `SEC-NNN` верифицируются adversarial-сьютами, реализованными и документированными в **`07-Testing-QA-Strategy.md`** (§8 PII-сьюта, §9 Sandbox Security Tests, §12 CI-гейты); фикстуры — в общем каталоге `docs/security/corpus/` (co-owned с QA). Идентификаторы тестов шарятся через матрицу трассировки threat→test.

| Сьют (doc 07 §) | Проверяет | Pass bar (CI gate) | Фаза-гейт |
|-----------------|-----------|--------------------|-----------|
| SOP Sandbox Escape Corpus (§9) | TM-020/021/022/023/024, SEC-020/030 | **0 escape'ов на 500-скриптовом adversarial corpus**; 100% эскалационных попыток denied+logged; AST p99 ≤ 50 мс | Alpha |
| PII Bypass Corpus + canary + network capture (§8) | TM-003/030/031, SEC-001..004 | high-sensitivity **0%** утечек (canary в egress — S1, release-blocking); NER < 0.5%; p99 ≤ 20 мс; invariant-тест единственного сетевого egress | Alpha (high-sens), Beta (NER) |
| IPC Spoofing Tests (§4/§9) | TM-010..014, SEC-010..014 | peer-credential rejection от чужого uid (fake UID в userns-контейнере), malformed envelope → drop+audit, seqlock-рассинхрон детектируется | Alpha |
| Permission Tier Property Tests (§9) | TM-040/041, SEC-020/021 | 100% Tier 2 через approval; 0 путей без executor-проверки; TOCTOU-тест (мутация параметров между approval и execution → отказ); отмена countdown — 0 частичных эффектов | Alpha/Beta |
| Prompt Injection Corpus (§9) | TM-021/033, SEC-040/041 | 100+ сэмплов в OCR/MCP-подобном контенте → **0 несанкционированных действий**; максимум — уточняющий вопрос; контркорпус легитимных команд не блокируется | Beta |
| Update Signing E2E (§14, doc 09) | TM-042/043, SEC-060 | подпись/верификация end-to-end, downgrade refused, kill-switch drill | Beta |
| Consent Withdrawal Degradation (§11) | SEC-070/071 | 4 тумблера: каждый off-стейт имеет протестированный degradation path; DPIA подана | Beta |
| Audit Tamper Test (§5) | TM-007, SEC-080/081 | подмена/удаление записи детектируется при загрузке; цепь восстанавливается эпитафом; retention/erasure-тест ≤ 24 ч; runbook drill | v1.0 |
| RAM Regression (§10) | §1 budget | security-слой ≤ 15 МБ steady-state в составе 150/655/1450 МБ гейтов | Alpha+ |
| Enterprise Addendum | SOC 2-экспорт, CMK, on-prem | см. документ 09/11 | v2.0 |

CI-механика (инструменты): Semgrep (taint-правила «нет сетевых вызовов вне egress-модуля», «нет sanitizer bypass»), cargo-audit/cargo-deny, pip-audit, Gitleaks, Hypothesis (property-тесты AST/tier), cargo-fuzz на парсерах envelope/AST, golden-set регрессия корпуса PII по образцу OCR golden-images. Каждая Critical/High STRIDE-строка обязана иметь linked passing test на phase-гейте — реестр генерируется из issue-трекера, не руками.

---

## 17. Open Questions и реестр рисков

| ID | Вопрос | Owner | Целевая фаза |
|----|--------|-------|--------------|
| OQ-01 | Достаточно ли countdown 3–5 с для быстрых необратимых Tier 2 (send message); нужно ли второе подтверждение? | UX + Security | Beta |
| OQ-02 | Linux без Secret Service: plaintext fallback с warning vs отказ от шифрования vs login-passphrase (defaults memory DB + audit) | ADR (док. 12) | Alpha |
| OQ-03 | Second-voice pause: must-have Beta или backlog (стоимость speaker-ID vs legal exposure TM-005) | Product | Beta |
| OQ-04 | Целостность моделей ONNX/GGUF: общий подписанный manifest update-канала или отдельный model registry | CI/CD (док. 09) | Beta |
| OQ-05 | ZDR-условия провайдеров: у кого enforceable zero-data-retention; hard-requirement для default? | Legal + Eng | Beta |
| OQ-06 | Bug bounty для sandbox/sanitizer и disclosure-таймлайн | Security | Beta |
| OQ-07 | Минимальный возраст и parental consent для RU 14–18 / EU 13–16 (EGE-аудитория включает minors) | Legal | RU/EU Beta |
| OQ-08 | 152-FZ: легальность relay санитизированных сниппетов к иностранному провайдеру (статус оператора, кросс-бордер) — блокер RU-Beta | Legal | RU-Beta |
| OQ-09 | ADR: интерпретатор SOP sandbox (CPython-restricted vs embedded vs WASM/wasmtime) | ADR (док. 12) | Alpha |
| OQ-10 | Enterprise-экспорт аудита (signed bundle vs API) и customer-managed keys | Security | v2.0 |

Документ — **живой артефакт**: STRIDE-реестр пересматривается и переподписывается на каждом phase gate (Alpha → v2.0); границы фаз меняются (cloud, сторонние MCP, enterprise) и модель обязана отражать реальный код, а не начальный замысел. Анти-паттерн из research: «митигации, на которые ссылается документ, более не существуют в коде» — превенция через трассировку threat→test→CI.

---

*Числа, SLA и идентификаторы согласованы с RFC-0001: Barge-in < 200 мс; VAD 85–100 мс (>95%); OCR ~200 мс (185 мс эталон); RAM 150/655/1450 МБ; фрустрация порог 50 (S_max = 100, полураспад 20/15/5/1 мин); SOP cosine ≥ 0.98; окно отмены 3–5 с; инсталлятор ~8 МБ; IPC UDS 0600 / Named Pipe ACL; seqlock `ShmFrameDescriptor`; контракт `contracts/ipc_bridge.proto`.*
