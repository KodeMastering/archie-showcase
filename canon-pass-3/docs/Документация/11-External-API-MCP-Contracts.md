# 11. External API & MCP Contracts Specification

**Проект:** Archie (OS-Level AI Symbiote)
**Статус:** PROPOSED / READY FOR IMPLEMENTATION
**Версия документа:** 0.9.0 (до фазы v1.0 — контракт 0.x, breaking changes допустимы)
**Целевые релизы:** MVP Alpha → B2C Beta → v1.0 → v2.0 Enterprise
**Связанные документы:** RFC-0001 (IPC, Event Bus, RAM-бюджет), 01-PRD (FR-MCP), 03-Threat-Model, 07-Testing-QA, 09-CI-CD, 10-Observability

---

## 1. Назначение и обзор

Настоящий документ определяет **целевую/дорожную внешнюю поверхность расширения** платформы Archie: контракт, который реализуют сторонние разработчики, чтобы подключить MCP-серверы (Model Context Protocol) к хабу Archie, и гарантии, которые Archie возвращает разработчикам и пользователям. Документ закрывает четыре задачи:

> **Current Phase 1 MVP vs Target/Roadmap.** MCP-hub, SOP, Permission Tiers и внешние серверы не реализованы в текущем Phase 1 MVP. Этот документ — контракт целевой архитектуры (MVP Alpha → B2C Beta → v1.0 → v2.0), а не утверждение текущего runtime. Cloud path Archie (aijuju) конфигурируемый/opt-in; локальные VAD/STT/Piper TTS, память и `local`/`mock` fallback работают без cloud. Raw screen/audio должны оставаться local-only; в будущий внешний контур допускается только санитизированный текст после проверенных controls. Текущий sanitizer — базовый scaffold, не доказанная полная regex+NER-гарантия; telemetry и egress upload в Phase 1 не реализованы.

1. Реализацию MCP-хаба и брокера tool-вызовов без редизайна протокола под каждую интеграцию.
2. Разработку четырёх эталонных серверов (`obsidian_vault_server`, `notion_sync_server`, `git_automation_server`, `enterprise_jira_slack`) с явными схемами, уровнями разрешений и правилами жизненного цикла.
3. Проверку безопасности: декларация permission tier — это **контроль безопасности**, вход для модели угроз (doc 03).
4. Интеграционную историю v2.0 Enterprise (Jira/Slack, каталог коннекторов, политики администратора).

### 1.1. Две семейства контрактов

| Семейство | Формат | Расположение | Потребители |
|---|---|---|---|
| **Внешний MCP-контракт** | JSON-RPC 2.0 / MCP, JSON Schema draft 2020-12 | `contracts/schema/*.schema.json`, `contracts/examples/manifests/*.json` | Сторонние разработчики, MCP-хаб |
| **Внутренний IPC-контракт** | Protobuf v3 (gRPC поверх UDS / Named Pipes) | `contracts/ipc_bridge.proto` и `contracts/proto/` | Rust-ядро ↔ Python-воркер (RFC-0001) |

Оба семейства разделяют **единую политику эволюции**: semver 2.0.0, additive-only в пределах major, machine-гейты в CI (раздел 11). Внутренние proto-контракты определены в RFC-0001 (`EventEnvelope`, `SystemActionRequest/Response`, `PermissionTier`, `WorkerHealthMetrics`); настоящий документ управляет их **политикой изменений**, а не содержанием.

### 1.2. Принципы

- **Не изобретать протокол.** Проводной контракт — профиль открытой спецификации MCP (JSON-RPC 2.0, stdio + Streamable HTTP). Эталонные серверы Archie работают и с другими MCP-клиентами.
- **Контракты живут в коде.** JSON Schema и proto — в `contracts/`; типы генерируются в CI; «чего нет в сгенерированном коде — не существует».
- **Декларируй возможности — проверяй поведение.** Аннотации MCP (`readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint`) — это заявления, а не факты; хаб кросс-проверяет их с фактическим поведением (staging, сетевой egress).
- **Fail closed.** Невалидный манифест не загружается никогда; неоднозначные инструменты получают самый строгий tier. 

> **Current MVP:** в Phase 1 MVP этот constraint — не part of runtime; target-level security contract.
- **Rate limiting в хабе.** Бюджеты enforcing до диспетчеризации (защита хоста), обработка 429/Retry-After наружу (защита удалённой стороны), один медленный сервер не блокирует остальных.

---

## 2. Архитектурное размещение MCP-хаба

MCP-хаб (`services/archie-engine/src/archie_engine/mcp_hub/mcp_client_manager.py`) располагается между Семантическим роутером и внешними серверами. Ключевое правило SOP-песочницы: **песочница никогда не порождает shell-команды напрямую — она эмитирует tool-интенты**, которые исполняются только хабом. Это правило из ARCH-рисков (раздел 6, изоляция SOP): даже скомпрометированный промпт не может обойти карантинную директорию `Archie_Staging` или выполнить несанкционированное удаление.

```
Semantic Router / SOP Sandbox
        │  tool-intent (JSON, валидирован AST)
        ▼
   MCP Hub (брокер: манифест → tier → rate limit → circuit → dispatch)
        │                                        ▲
        ├─ stdio child process (локальные серверы)│ audit-event → Event Bus
        ├─ Streamable HTTP + OAuth 2.1 (remote)  │ health → WorkerHealthMetrics
        ▼
  Локальная ФС (только через staging) / объявленные hosts / внешние API
```

Все серверы — **on-demand дочерние процессы** (TTL = 30 с, из RAM-матрицы RFC-0001: 22 МБ в пике, 15 МБ офлайн; бюджет хаб+серверы укладывается в строку №12 бюджета 1450 МБ Offline Fallback). Аудит каждого Tier1/Tier2-вызова пишется в `action_audit_log` (SQL-схема ARCH, категория `MCP_TOOL_CALL`) и публикуется в Event Bus — v2.0 Enterprise-консоль потребляет тот же поток без ретрофита.

### 2.1. Профиль протокола MCP

| Возможность | Поддержка | Фаза |
|---|---|---|
| JSON-RPC 2.0 базис | Обязательно | все |
| `initialize` + negotiation `protocolVersion` | Обязательно; хаб эхом возвращает последнюю взаимно поддерживаемую ревизию | все |
| Транспорт **stdio** | Обязательно (локальные/community-серверы) | все |
| Транспорт **Streamable HTTP + OAuth 2.1 (PKCE)** | Обязательно для remote-серверов | v1.0+ |
| Транспорт SSE (legacy) | Только чтение, deprecated | B2C Beta → удаление в v1.0 |
| `tools` (+ `listChanged`, annotations, structured output) | Обязательно | все |
| `resources` | Только `list`/`read`, read-only | MVP |
| `prompts` | Не поддерживается (deferred) | открытый вопрос |
| Sampling / roots (клиентские примитивы) | Не поддерживается | — |

Поддерживаемые ревизии MCP-спецификации: базовая **2025-03-26**; расширения **2025-06-18** (tool annotations, structured output) принимаются с B2C Beta. Ревизия **2024-11-05** принимается в режиме совместимости только на MVP Alpha. Точный пининг ревизий по фазам и ежеквартальный re-baseline — открытый вопрос (раздел 16).

---

## 3. Модель плагинов: манифест MCP-сервера

Каждый сервер описывается **одним манифестом** — JSON-документом, валидируемым схемой `contracts/schema/tool_manifest.schema.json` (draft 2020-12, `additionalProperties: false` везде). Загрузка манифеста при старте хаба — ≤ 500 мс на MVP Alpha-профиле (≤ 10 инструментов, 2 сервера). Подпись считается по **каноническому JSON** (RFC 8785-стиль: сортированные ключи, UTF-8, без пробелов).

### 3.1. Поля манифеста

| Поле | Тип | Обяз. | Описание |
|---|---|---|---|
| `manifest_version` | `"1.0"` | ✔ | Версия формата манифеста. |
| `server.name` | slug | ✔ | Имя сервера, reverse-DNS-подобный slug (`com.vendor.anki_review`). |
| `server.display_name` | string | ✔ | Человекочитаемое имя для `main_dashboard`. |
| `server.version` | semver | ✔ | Версия самого сервера (SemVer 2.0.0). |
| `server.vendor`, `server.license`, `server.homepage` | string | ✔ | Идентификация издателя. |
| `server.archie_contract_version` | semver | ✔ | Версия контракта Archie, под который собран манифест. |
| `server.signature` | object | ✔ (B2C Beta+) | Ed25519-подпись канонического JSON, key id издателя/реестра. |
| `transport.type` | `"stdio" \| "http"` | ✔ | `stdio` — локальный исполняемый файл; `http` — Streamable HTTP endpoint. |
| `transport.command` / `transport.endpoint` | string | ✔ | Команда запуска (stdio) или URL (http, https-only). |
| `network.egress_hosts` | string[] | ✔ | Белый список хостов; пустой массив = серверу **запрещено открывать сокеты** (enforced песочницей ОС). |
| `fs.resource_scopes` | object[] | ✔ | Декларируемые области ФС: пути (glob), режимы (`read`/`write`). Запись вне scopes невозможна (enforced staging-медиацией). |
| `tools[]` | array | ✔ | Инструменты — см. 3.2. |
| `event_subscriptions[]` | array | — | Подписки на типы `EventEnvelope` — см. раздел 8. |
| `data.*` | object | — | `data_categories`, `data_residency` (`RU`/`EU`/`US`), `retention` — для GDPR/152-FZ и consent-UX. |
| `server.rate_class` | enum | — | Квота-класс (см. раздел 9; по умолчанию `standard`). |

Манифест **не содержит поля для сырых медиа-типов контекста** (raw audio / screen frames): контракт normative-запрещает серверам запрашивать сырой контекст — на поверхность MCP выходят только post-router интенты и post-sanitizer текст (см. 8.2).

### 3.2. Схема инструмента (`tools[]`)

| Поле | Описание |
|---|---|
| `name` | Уникальное в пределах сервера имя (slug, `vault.read_note`). |
| `description` | Описание для LLM-роутера и approval-карточек. |
| `version` | Semver **самого инструмента** (независимо от версии сервера — см. раздел 7). |
| `inputSchema` | JSON Schema draft 2020-12, strict: `additionalProperties: false`, все `$ref` резолвлены. Обязательно ≥ 5 примеров payload (valid + invalid) в `examples/` манифеста — иначе регистрация отклоняется (CONTRACT-VALIDATE-1). |
| `output.content_types` | Типы возвращаемого контента (`text/markdown`, `application/json`). |
| `annotations` | MCP-хинты: `readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint`. |
| `archie.permission_tier` | `0 \| 1 \| 2` — минимальный tier исполнения (см. раздел 4). |
| `archie.staging_required` | `true` для любого инструмента, меняющего ФС. |
| `archie.resource_scope` | Per-tool скоуп (подмножество серверных scopes; default — минимум). |
| `archie.timeout_class` | `fast` (2 с) / `normal` (10 с) / `long` (60 с). |
| `archie.idempotency_key` | Опционально: поддержка key для безопасного ретрая. |
| `archie.deprecated` / `archie.replaced_by` | Маркеры деприкации (раздел 7). |

---

## 4. Permission Tier: декларирование и enforcement

### 4.1. Соответствие внешних и внутренних tier

Внутренняя модель RFC-0001 (`PermissionTier` enum: `TIER_1_AUTONOMOUS`, `TIER_2_CRITICAL_APPROVAL`, `TIER_3_WORKSPACE_ORCHESTRATION`) — канон для Action Engine. Внешний MCP-контракт добавляет **Tier 0** для read-only-инструментов и закрывает Tier 3 для третьих лиц:

| Внешний tier | Внутренний эквивалент | Семантика | UX-подтверждение |
|---|---|---|---|
| **Tier 0** (read-only) | — (не требует approval) | Только чтение, секретов не потребляет, сателлитная диспетчеризация без карточек | Молчаливое исполнение |
| **Tier 1** | `TIER_1_AUTONOMOUS` | Обратимые записи внутри staging-зон либо неразрушающая удалённая синхронизация | One-click approval card (soft widget) |
| **Tier 2** | `TIER_2_CRITICAL_APPROVAL` | Разрушающие/необратимые операции (git push, удаление страниц, публичные сообщения) | **Всегда** модальная карточка с обратным отсчётом 3–5 с (`Tier2ApprovalCard.tsx`) и staging-rollback, где применимо |
| ~~Tier 3~~ | `TIER_3_WORKSPACE_ORCHESTRATION` | **Недоступен третьим лицам** — оркестрация окружений только внутреннему `workspace_orchestrator` | — |

### 4.2. Контракт enforcement (CONTRACT-TIER-1)

Инструмент декларирует **минимальный tier** в `archie.permission_tier`. Хаб выполняет 3 проверки при регистрации и 1 при рантайме:

1. **Tier ↔ аннотации:** `readOnlyHint: true` ⇒ tier обязан быть 0; `destructiveHint: true` ⇒ tier обязан быть 2. Несовпадение = отказ регистрации.
2. **Tier ↔ resource_scope:** Tier 0 не может декларировать write-режим scopes; Tier 2 с write-scope обязан иметь `staging_required` либо пометку `irreversible: true` с явным текстом для карточки.
3. **Tier ↔ network:** декларация egress согласована с `openWorldHint`.
4. **Рантайм:** наблюдаемые записи сверяются со staging-ограничением. **Любая запись вне staging-медиации → вызов abort, инцидент безопасности, downgrade trust-оценки сервера.** Первая недекларация — логируемое событие + понижение trust score; вторая — сервер отключается.

### 4.3. Ограничения песочницы

- **ФС:** никаких операций вне декларированных `resource_scopes`; все записи Tier 1/2 маршрутизируются через staging-коммиты (`archie-stagingfs`) — физический запрет системного вызова Delete наследуется от ARCH.
- **Сеть:** сервер с пустым `egress_hosts` не может открыть сокет (enforced ОС-механизмами: seccomp/landlock + bubblewrap на Linux, App Sandbox на macOS, restricted tokens на Windows — Linux-first в MVP Alpha). Обращение к недекларированному хосту = `ARCHIE_SANDBOX_VIOLATION` + отключение.
- **Голосовое согласие:** если голосовой consent отозван, хаб не исполняет Tier 1/2 вызовы, производные от voice-init интентов (правило из regulatory-блока; точный gating — открытый вопрос).
- **Неоднозначные инструменты:** fail-closed — максимальный tier (2).

---

## 5. Референсные реализации

Четыре встроенных сервера — одновременно **conformance-набор** контракта: они покрывают Tier 0/1/2, staging, remote OAuth и rate limits до появления третьих лиц.

### 5.1. `obsidian_vault_server` (local, stdio, MVP Alpha)

**Назначение:** ночная выгрузка Markdown-сводок Tier 2 памяти в пользовательский vault. Конфигурация vault-пути задаётся в `main_dashboard` (user-selected directory) — путь не хардкодится и не хранится в манифесте, а подставляется хабом как runtime-параметр скоупа. Сетевой egress — пустой (никаких сокетов).

| Инструмент | Tier | Аннотации | Примечание |
|---|---|---|---|
| `vault.search` | 0 | readOnly | Полнотекстовый поиск по заметкам. |
| `vault.read_note` | 0 | readOnly | Чтение заметки. |
| `vault.list_tags` | 0 | readOnly, idempotent | Список тегов. |
| `vault.append_note` | 1 | idempotent: false | **Atomic write**: запись через staging-транзакцию — temp-файл + `os.replace` (атомарная замена) + staging-копия для rollback; сбой на любом шаге оставляет оригинал нетронутым. `staging_required: true`. |

Контракт с таблицей `nightly_summaries_tier2`: сервер получает `summary_id`, дату и три Markdown-блока (`learned_concepts_md`, `code_written_summary_md`, `resolved_blockers_md`); после записи возвращает `resource_uri` для поля `mcp_resource_uri` и статус `SYNCED`/`FAILED`. KPI фазы 2: 100% успешная ночная генерация `.md`-отчёта.

### 5.2. `notion_sync_server` (remote HTTP + OAuth 2.1, B2C Beta)

Альтернативная цель ночной выгрузки (`mcp_target_app = 'NOTION'`). Инструменты: `notion.search` (Tier 0), `notion.create_page` / `notion.update_page` (Tier 1), `notion.delete_page` (Tier 2, `destructiveHint: true`, irreversible). Секреты — только OS keyring; токены никогда не попадают в vault, конфиги, логи (enforced PII-санитайзер-тестами). Обрабатывает пагинацию search и 429 Notion (~3 req/s) через общую backpressure-механику раздела 9. `data_residency` обязателен в манифесте (152-FZ/GDPR — сервер процессор пользовательского контента, data minimization через `resource_scope`).

### 5.3. `git_automation_server` (local, stdio, MVP Alpha)

Коммит-стейджинг для SOP-рецепта «IDE закрыта после 4 ч кодинга → commit + запись в Obsidian».

| Инструмент | Tier | Аннотации |
|---|---|---|
| `git.status` / `git.diff` / `git.log` | 0 | readOnly, idempotent; timeout `fast` — p95 ≤ 2 с (CONTRACT-LATENCY-1, чтобы проактивные check-in укладывались в UX-бюджет RFC-0001) |
| `git.commit` | 1 | staging: изменения стейджатся в commit-транзакцию; `staging_required: true` |
| `git.push` | **2** | `destructiveHint: true`; **никогда не исполняется без явного Tier 2 approval** — модальная карточка «Я подготовил коммит. Пушим?»; любой путь исполнения push через SOP обязан содержать Tier-2 шаг, AST-валидатор песочницы отклоняет рецепт с push без approval-узла |

Скоуп — только user-approved репозитории (per-tool scope, default minimum). Сервер local-first (stdio), что отражает осознанное расхождение с GitMCP-паттерном hosted-автоматизации.

### 5.4. `enterprise_jira_slack` (remote, v2.0)

Admin-provisioned OAuth (Atlassian Remote MCP / Slack MCP паттерны). Инструменты: `jira.search` (0), `jira.create_ticket` / `jira.transition` (1), `slack.search` (0), `slack.post_message` — **Tier 1 для приватных каналов, Tier 2 для публичных** (зеркалит гранулярность Slack-scopes). Поверх — org policy overlay из админ-консоли (может ужесточать, но не ослаблять tiers). Каждый Tier 1/2 вызов эмитирует audit-event (actor, tool, args-hash, scope, decision, rollback) — поток NIST AU / SOC 2 Type II evidence. Art. 28 processor-terms обязательны для вендорской поверхности.

---

## 6. Версионирование схем инструментов

- **Сервер и каждый инструмент версифицируются по SemVer 2.0.0 независимо.** `archie_contract_version` манифеста семантически «pinned» против поддерживаемого хабом диапазона.
- **MINOR (additive-only):** новые опциональные поля, новые инструменты, новые не-breaking значения enum. Правило: минор не ломает старый хаб — **unknown-field tolerance обязателен с обеих сторон** (CONTRACT-COMPAT-1). CI прогоняет матрицу old-hub + new-manifest и new-hub + old-manifest.
- **PATCH:** изменения description/examples, без semantics.
- **MAJOR:** удаление/переименование/семантическое изменение любого поля; **эскалация tier существующего инструмента**; **расширение resource_scope**; сужение enum. Всё это требует новой `archie_contract_version`, migration-дока в PR и повторного approval скоупов пользователем (карточка показывает точный scope-delta).
- **Деприкация инструмента:** помечается `deprecated: true` + `replaced_by`; хаб логирует warning-событие при вызовах; удаление — только в следующем major либо после 2 минорных циклов. **Окно совместимости major-версий: двойная поддержка N и N-1 ≥ 6 месяцев (B2C) / ≥ 12 месяцев (Enterprise).** Bump `protocolVersion` MCP трактуется как внешняя зависимость с тем же окном. Каждое удаление отражается в changelog и матрице совместимости (раздел 12). Паттерны: Stripe versioning + GitHub Sunset (RFC 8594).

---

## 7. Rate limits и квоты

Приоритетный инвариант: **MCP-трафик строго ниже аудио-петли** на Event Bus — SLA barge-in < 200 мс (RFC-0001) не может быть кончендед tool-вызовами. Полный retry-бюджет одного вызова ≤ 30 с.

| Скоуп | Лимит | Поведение при превышении |
|---|---|---|
| Per-server concurrency | 1 поток (настраивается до 4) | Вызов в очередь |
| Hub queue depth (outbound) | 16, приоритетные лейны по mission state (GAMING_REST подавляет некритические вызовы) | Структурированная retryable-ошибка роутеру за ≤ 10 мс (CONTRACT-RL-1); шина не блокируется |
| Per-server token bucket (hub, **до** диспетчеризации) | 30 вызовов/мин, burst 10 | `ARCHIE_RATE_LIMITED` (retryable) |
| Timeout-классы | `fast` 2 с / `normal` 10 с / `long` 60 с (например git push) | `ARCHIE_SERVER_TIMEOUT`, circuit-breaker сервера |
| Remote HTTP (исходящие) | 429 + `Retry-After` (RFC 7231); иначе экспоненциальный backoff c jitter, база 1 с, **максимум 3 ретрая**; уважать `x-ratelimit-*` провайдера | Retry-бюджет ≤ 30 с суммарно |
| Per-server circuit | независимые фьючерсы/таски | Один медленный сервер не блокирует Tier 0 вызовы другим (запрет head-of-line blocking) |

Квоты конфигурируются из тех же budget-таблиц, что RAM/latency-бюджеты RFC-0001; классы `rate_class` в манифесте: `light` / `standard` / `heavy` (Enterprise).

---

## 8. Подписка на события (Event Subscription Contract)

### 8.1. Разрешённые типы `EventEnvelope`

Третьи лица подписываются через `event_subscriptions[]` манифеста; хаб фильтрует по allowlist:

| Тип | Разрешён третьим лицам | Поставка |
|---|---|---|
| `APP_OPENED` / `APP_CLOSED` | ✔ | Только метаданные (`process_name`, `window_title`); `project_path` — по явному scope-гранту |
| `MISSION_STATE_CHANGED` | ✔ | Полный payload |
| `FRUSTRATION_SPIKE` | ✔ | **Только score и heuristic-код**; `inner_monologue_prompt` отрезается |
| `ZSENSE_GRAPH_READY` | ✔ (условно) | **Только после PII-прохода** — см. 8.2 |
| `FORCE_STOP_AUDIO`, `API_TIMEOUT`, `BLUEPRINT_MATCHED_98`, `SOFT_WIDGET_USER_DECISION` | ✘ | Внутренние, подписка отклоняется на регистрации |

### 8.2. Граница доверия ZSENSE

**Контент ZSENSE никогда не покидает локальный trust boundary в сторону третьих лиц без PII-прохода.** Нормативно: подписчики получают не сам `ZSenseGraphResponse`, а производную `ZSENSE_GRAPH_SANITIZED`, формируемую только после каскада `pii_sanitizer` (regex + NER, теги `[REDACTED_*]`), и только поля, перечисленные в scope-гранте подписки (`active_window`, `error_state`, `mission_state`); `visible_text_snippet` и `lines[]` — только по явному granular-гранту пользователя и всегда санитизированными. Хаб физически не пересылает несанитизированные кадры/тексты — поле для raw-типов в схеме манифеста отсутствует. Голосовые реплики (`VOICE_*`) — вне MCP-поверхности полностью.

### 8.3. Формат подписки

```json
{
  "event_type": "MISSION_STATE_CHANGED",
  "filter": { "mission_state": ["STUDY_FOCUS", "CODING_WORK"] },
  "max_rate": "1/10s",
  "dedup_window_ms": 500
}
```

Нарушение `max_rate` в подписке → хаб агрегирует/дропает, не баня сервер. События доставляются серверу как MCP-уведомления (server-initiated по Streamable HTTP) либо через отдельный stdio-notification-канал.

---

## 9. Стандарт ошибок (Error Envelope)

Все ошибки — JSON-RPC error object с обязательным структурированным `data` (`contracts/schema/error_data.schema.json`): `error_code` (string enum), `retryable` (bool), `requires_user_action` (bool), `safe_message` (≤ 200 симв., для approval-карточек), `details` (санитизированный объект). **100% текстов ошибок проходят PII-санитайзер перед логированием в Event Bus** — один санитайзер, те же 4 chokepoint-а, что для экранных слепков (CONTRACT-ERR-1).

| Код | error_code | retryable | Семантика / ретраи |
|---|---|---|---|
| −32700 | `PARSE_ERROR` | ✔ | Битый JSON; ретрай не имеет смысла на том же payload |
| −32600 | `INVALID_REQUEST` | ✘ | Нарушение JSON-RPC |
| −32601 | `METHOD_NOT_FOUND` | ✘ | Метод вне профиля Archie |
| −32602 | `INVALID_PARAMS` | ✘ | Провал inputSchema — permanent, ретрай только с другими аргументами |
| −32603 | `INTERNAL_ERROR` | ✔ | Внутренняя ошибка сервера; backoff |
| −32001 | `ARCHIE_RATE_LIMITED` | ✔ | Bucket исчерпан; уважать Retry-After/backoff |
| −32002 | `ARCHIE_SERVER_TIMEOUT` | ✔ | Превышен timeout-класс; ≤ 3 ретрая, затем circuit |
| −32003 | `ARCHIE_SCOPE_VIOLATION` | ✘ | Обращение вне resource_scope; security-событие, trust downgrade |
| −32004 | `ARCHIE_SANDBOX_VIOLATION` | ✘ | Сетевой/ФС egress вне декларации; сервер отключается |
| −32005 | `ARCHIE_TIER_DECLARATION_INVALID` | ✘ | Провал кросс-проверки tier ↔ аннотации |
| −32006 | `ARCHIE_TOOL_DEPRECATED` | ✔ | Warning-семантика; вызов исполняется в окне деприкации |
| −32007 | `ARCHIE_SIGNATURE_INVALID` | ✘ | Подпись манифеста не сходится — отказ загрузки (fail closed) |

Классификация ретраев: сетевые → retryable с backoff; schema (−32602) → permanent; permission-denied → `requires_user_action: true`, хаб показывает approval-карточку вместо слепого ретрая.

---

## 10. Эволюция схем и codegen (contracts/ + CI)

### 10.1. Раскладка и поток генерации

```
contracts/
├── schema/   tool_manifest.schema.json, error_data.schema.json,
│             event_subscription.schema.json, audit_event.schema.json
├── proto/    ipc_bridge.proto (RFC-0001) + будущие внешние proto
├── examples/manifests/{obsidian_vault, notion_sync, git_automation}.json
└── docs/     changelog контракта, матрица совместимости (генерируется)
```

Codegen-цели (CI, никаких runtime-генов): **TypeScript** — типы для `apps/desktop-ui` (json-schema-to-typescript / quicktype); **Rust** — schemars/serde-типы для хаба + prost/tonic для IPC; **Python** — datamodel-code-generator (mypy-checked) для `services/archie-engine` + grpcio-стабы. Сгенерированные артефакты коммитятся в-tree и ревьюятся в PR. Таблицы spec-документа (списки инструментов, матрица совместимости) генерируются из `contracts/examples` скриптом — док и схемы не расходятся молча.

### 10.2. CI-гейты (CONTRACT-CI-1)

1. **schema-validate:** каждый манифест в `examples/` валиден против `tool_manifest.schema.json` (Python `jsonschema`); runtime-валидация в Rust — schemars.
2. **buf breaking/lint:** proto-контракты проверяются против snapshot main-ветки (buf.build).
3. **golden snapshot:** сгенерированные Rust/Python типы компилируются и round-trip-ят эталонные манифесты; pytest-conformance регистрирует референсные серверы против сборки хаба и прогоняет golden tool-вызовы с assert-ами на tier/timeout/errors.
4. **version-bump check:** любой не-additive diff схемы красит CI; override — только PR с major-bump + migration-дока.

### 10.3. Чеклист breaking change (для reviewer)

- [ ] Удалено/переименовано поле или инструмент?
- [ ] Сужен enum / тип стал строже?
- [ ] Эскалация `permission_tier` инструмента?
- [ ] Расширен `resource_scope` / `egress_hosts`?
- [ ] Изменена семантика существующего поля (не только текст)?
- [ ] Если да хоть в одном пункте → MAJOR: новая `archie_contract_version`, ADR-запись (doc 12), migration-дока, план dual-support ≥ 6/12 мес, re-approval скоупов пользователем, обновление changelog и матрицы.

---

## 11. Безопасность и доверие (требования к авторам плагинов)

- **Подпись манифеста:** Ed25519 (`ed25519-dalek` / `cryptography`) поверх канонического JSON. С **B2C Beta** unsigned-серверы запускаются только под dev-флагом с предупреждением; с v1.0 подпись обязательна для community-серверов.
- **Дистрибуция:** публикация в MCP Registry (namespace + content-addressed версии + publisher identity) и/или курируемый каталог Archie; для stdio-установок из git-releases — hash pinning (sha256 бинарника сверяется при каждом запуске).
- **Долгосрочно (v2.0+): code signing пакетов MCP-серверов** с верифицированной publisher-идентичностью и revocation-workflow; модель — VS Code marketplace / npm trust lessons.
- **Ревью для first-party marketplace:** Security-минимум листинга v1.0: подписанный манифест, SECURITY.md с coordinated disclosure, passing conformance-набор (подпись vs подпись+песочница+тесты — открытый вопрос сертификации), проверка inputSchema на инъекции (OWASP LLM01/06/08 делегированы из Threat-Model).
- **Секреты:** токены Notion/Jira/Slack — только OS keyring (Secret Service / Credential Manager); появление секрета в логах/телеметрии/vault = провал CI-теста санитайзера.
- **OAuth 2.1 + PKCE** для remote-серверов, хаб — public client.

---

## 12. Жизненный цикл сервера (user-visible в main_dashboard)

Все состояния видны в `main_dashboard` (раздел Connectors), каждое Tier 1/2 действие — в audit-логе:

| Состояние | Действие | Контракт |
|---|---|---|
| **Install** | Установка из каталога/реестра или URL | Проверка подписи, hash pinning, показ **scope-delta-карточки** (какие пути/хосты/события запрашиваются), consent на data-категории/резидентность; при cross-border (RU) — явное согласие в flow установки |
| **Enable** | Активация | Регистрация манифеста, все 4 проверки tier, spawn on-demand (TTL 30 c) |
| **Disable** | Деактивация | Процессы гасятся, подписки отписываются; конфиг и секреты сохраняются |
| **Update** | Обновление | Diff манифестов: additive — тихо (changelog-уведомление); tier-эскалация / расширение скоупа — **re-approval карточкой**; major-переход — миграционная заметка |
| **Uninstall** | Удаление | Удаление бинарника, конфигов, **секрета из keyring**, hash-pin; staging-файлы и audit-лог остаются (prerogative пользователя) |

Trust-состояние сервера (nominal / downgraded / quarantined) отображается рядом; два misdeclaration tier → автоматический quarantine + карточка инцидента.

---

## 13. Полный пример: гипотетический `anki_review_server`

Сторонний сервер напоминаний о повторении карточек Anki (Tier 1, локальный, с подпиской на mission state).

**Манифест (`contracts/examples/manifests/anki_review.json`):**

```json
{
  "manifest_version": "1.0",
  "server": {
    "name": "net.example.anki_review",
    "display_name": "Anki Review Scheduler",
    "version": "1.2.0",
    "vendor": "Example Labs",
    "license": "MIT",
    "homepage": "https://example.dev/anki-review",
    "archie_contract_version": "0.9.0",
    "signature": {
      "algorithm": "ed25519",
      "key_id": "example-labs-2026",
      "signature_b64": "kQ7... (по каноническому JSON)"
    }
  },
  "transport": { "type": "stdio", "command": "anki-review-mcp" },
  "network": { "egress_hosts": [] },
  "fs": {
    "resource_scopes": [
      { "path_glob": "$USER_HOME/Documents/Anki/*.txt", "mode": "read" }
    ]
  },
  "data": {
    "data_categories": ["study_notes_metadata"],
    "data_residency": "RU",
    "retention": "session_only"
  },
  "tools": [
    {
      "name": "anki.due_cards",
      "description": "Возвращает количество карточек, подлежащих повторению сегодня.",
      "version": "1.0.0",
      "inputSchema": {
        "type": "object",
        "properties": {
          "deck": { "type": "string", "pattern": "^[a-zA-Z0-9_ -]{1,64}$" },
          "limit": { "type": "integer", "minimum": 1, "maximum": 100 }
        },
        "required": ["deck"],
        "additionalProperties": false
      },
      "output": { "content_types": ["application/json"] },
      "annotations": { "readOnlyHint": true, "idempotentHint": true },
      "examples": [
        { "valid": true, "payload": { "deck": "EGE_Math", "limit": 20 } },
        { "valid": true, "payload": { "deck": "English B2" } },
        { "valid": false, "payload": { "deck": "x", "extra": 1 },
          "reason": "additionalProperties false" },
        { "valid": false, "payload": { "deck": 123 }, "reason": "type" },
        { "valid": false, "payload": { "deck": "a".repeat(65) }, "reason": "pattern" }
      ],
      "archie": {
        "permission_tier": 0,
        "timeout_class": "fast",
        "resource_scope": "$USER_HOME/Documents/Anki/*.txt:read"
      }
    },
    {
      "name": "anki.schedule_review",
      "description": "Планирует сессию повторения и пишет журнал сессии в файл колоды.",
      "version": "1.1.0",
      "inputSchema": {
        "type": "object",
        "properties": {
          "deck": { "type": "string", "pattern": "^[a-zA-Z0-9_ -]{1,64}$" },
          "session_minutes": { "type": "integer", "minimum": 5, "maximum": 60 }
        },
        "required": ["deck", "session_minutes"],
        "additionalProperties": false
      },
      "output": { "content_types": ["text/markdown"] },
      "annotations": { "readOnlyHint": false, "idempotentHint": false },
      "archie": {
        "permission_tier": 1,
        "staging_required": true,
        "timeout_class": "normal",
        "resource_scope": "$USER_HOME/Documents/Anki/*.txt:write",
        "idempotency_key": "deck"
      }
    }
  ],
  "event_subscriptions": [
    {
      "event_type": "MISSION_STATE_CHANGED",
      "filter": { "mission_state": ["STUDY_FOCUS"] },
      "max_rate": "1/10s",
      "dedup_window_ms": 500
    }
  ]
}
```

Проверки при регистрации: `anki.due_cards` — `readOnlyHint: true` и tier 0 ✔, write-scope отсутствует ✔; `anki.schedule_review` — tier 1, write-scope + `staging_required: true` ✔ (запись через staging-коммит, откатываема); egress пуст ⇒ песочница блокирует любые сокеты; подписка `MISSION_STATE_CHANGED` в allowlist ✔. Рантайм: фактическая запись колоды наблюдается через staging-медиацию — при попытке прямой записи мимо staging вызов abort, `ARCHIE_SCOPE_VIOLATION`, trust downgrade.

---

## 14. Матрица совместимости по фазам

| Фаза | Contract major | MCP-ревизии | Транспорты | Бандл | Примечания |
|---|---|---|---|---|---|
| MVP Alpha | 0.x (unstable) | 2024-11-05 (compat), 2025-03-26 | stdio | obsidian_vault, git_automation (dev-flag) | ≤ 10 инструментов, load ≤ 500 мс; unsigned-серверы только dev-flag |
| B2C Beta | 0.x → 1.0-rc | +2025-06-18 (annotations, structured output) | stdio; SSE read-only (deprecated) | + notion_sync (OAuth 2.1) | Подписанные манифесты обязательны для community |
| v1.0 | **1.0** (freeze) | пиннинг min..latest | stdio + **Streamable HTTP** | все 3 публичных; registry-публикация | ≥ 50 concurrent community-серверов, ≤ 15 МБ/stdio-сервер (строка RAM-бюджета); conformance-набор публичен |
| v2.0 | 1.x | re-baseline | + admin-managed remote | + enterprise_jira_slack | Org policy overlay, audit export, SOC 2 evidence, ≥ 12 мес dual-support |

Матрица генерируется CI-задой, которая реально прогоняет old-hub/new-manifest и new-hub/old-manifest — это evidence, а не декларация.

---

## 15. Открытые вопросы

1. Пиннинг MCP-ревизий по фазам и freeze v1.0 (ждать ли стабилизации registry/authorization).
2. Модель дистрибуции v1.0: MCP Registry vs курируемый каталог vs оба; владение signing-ключами и revocation-workflow.
3. Поддержка `prompts`/sampling-примитивов (зависит от эволюции Semantic Router).
4. Граница hosting remote-серверов: прокси через облако Archie (Enterprise audit) vs client-side (local-first, GDPR-простота).
5. Fee-free vs feature-fence для community-коннекторов (кросс-решение с Business Model doc).
6. Механизм 152-FZ egress-enforcement (manifest `data_residency` vs IP-блокировка) и размещение consent-UX.
7. Порог fail-closed к Tier 2 для неоднозначных инструментов (approval fatigue vs безопасность).
8. Сертификационный уровень conformance для community (signing only vs signing+sandbox+tests).

---

*Terminology: MCP, tool manifest, permission tier, staging zone, resource scope, egress host, error envelope, compatibility window — канонические английские термины; числовые SLA (2 с fast-timeout, 30 вызовов/мин, queue depth 16, retry-бюджет ≤ 30 с, RAM-строка хаба) — предложения к ратификации командой, консистентные с RFC-0001; канонические числа RFC (Barge-in < 200 мс, окно отмены 3–5 с, RAM 150/655/1450 МБ) — неизменны.*
