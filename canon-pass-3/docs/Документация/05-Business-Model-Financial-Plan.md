# 05. Бизнес-модель и финансовый план

**Проект:** Archie («Архи») — OS-Level AI Symbiote (гибридный edge+cloud проактивный ИИ-напарник)
**Статус документа:** v1.0 (draft для внутренней экспертизы и seed-раунда)
**Связанные документы:** [01-PRD](01-PRD.md) (feature fence → requirements), [04-Market-Competitor-Analysis](04-Market-Competitor-Analysis.md) (ценовые якоря), [06-Go-to-Market-Strategy](06-Go-to-Market-Strategy.md) (CAC по каналам), RFC-0001 (RAM-бюджеты, семантический роутер, порог фрустрации)
**Конвенция:** все численные оценки, не подтверждённые измерениями Alpha, помечены как **[assumption]** или **[uncertain]** и ссылаются на реестр допущений (раздел 12). Числа из RFC-0001 — канон.

---

## 1. Executive Summary

**Тезис модели:** Archie — edge-first продукт. Дефолтный пользовательский опыт (голос, персона, базовая память, проактивность, SOP-рутина) исполняется **локально** на CPU в рамках RAM-бюджета RFC-0001 (150/655/1450 МБ); cloud LLM API — это **uplift**, а не baseline. Это разворачивает экономику: у pure-cloud конкурентов (ChatGPT, Copilot) COGS растёт линейно с каждым бесплатным пользователем, у Archie — только с добровольным «апгрейдом качества». Local-first — это одновременно privacy-story, compliance-saver (152-FZ/GDPR) и **margin moat**, главный инвестиционный аргумент.

**Ключевые числа (базовый сценарий, все [assumption]):**

| Метрика | Значение |
|---|---|
| Цена Plus (RU / EN) | 299 ₽/мес / $9.99/мес |
| Цена Pro (RU / EN) | 599 ₽/мес / $19.99/мес |
| Blended cloud COGS на активного free-пользователя | ≤ $0.30/мес (жёсткий cap роутером) |
| Cloud COGS на платящего | ≤ 20% цены подписки (steady state) |
| Gross margin платящих тиров | ≥ 70% (после routing-оптимизации), ≥ 60% в Beta |
| Conversion free→paid | 3–5% blended |
| B2C churn | ≤ 10%/мес (target), стресс-тест 20% |
| LTV/CAC | ≥ 3 (B2C к v1.0), ≥ 5 (B2B) |
| Seed | $300–600K, runway 18–24 мес |
| Break-even | ~3.3–4K платящих B2C при burn $25K/мес — достижимо в Y3 base / Y2 bull |

**Структура доходов (диверсификация):** B2C-подписки (Free/Plus/Pro + annual), B2B-лицензии (per-seat SaaS, on-premise v2.0, Burnout Tracker add-on), long-term — marketplace MCP-плагинов. BYOK (bring-your-own-key) — zero-COGS escape hatch для тяжёлых пользователей.

---

## 2. Value Proposition Canvas

### 2.1. Сегменты и ценность (персоны из PAD/PRD, не переизобретаются)

**Сегмент 1 — студенты / подготовка к ЕГЭ (B2C, RU-приоритет)**
- *Боли:* застревание на задачах в одиночестве, нет «старшего товарища» рядом, дешёвое/слабое железо (ноутбуки 8 ГБ RAM без GPU), прокрастинация.
- *Gain-creators:* проактивная помощь при фрустрации (порог 50, decay-модель — «Archie замечает раньше, чем ты сдашься»), автоматический дневник в Obsidian/Notion, Context Workspace «Хочу учиться».
- *Anti-pains:* работает на CPU (RAM-бюджет 150 МБ idle), не шлёт сырые скриншоты в облако (JSON-граф экономит 99% трафика → копеечные API-запросы → дешёвый free-tier).

**Сегмент 2 — junior-разработчики (B2C, RU+EN)**
- *Боли:* StackOverflow-каруселя, страх git push, рутина (коммит → лог → отчёт).
- *Gain-creators:* SOP-Blueprints («закончил кодить» → commit + лог, косинус-совпадение ≥ 0.98 → детерминированный скрипт без галлюцинаций), MCP-хаб (Jira/Slack/Obsidian), Staging-зоны (риск потери данных = 0).
- *Anti-pains:* MCP-интеграции и неограниченные SOP — за деньги, но локальный опыт реально хорош в Free (конкуренция с OSS/Obsidian+local-LLM стеком).

**Сегмент 3 — геймеры (B2C, вторичный)**
- *Боли:* переключение контекста, рутинная настройка сессий.
- *Gain-creators:* миссия GAMING/REST — проактивность 0%, «не лезет, когда отдыхаете» (это тоже ценность); сортировка файлов, музыка, запуск программ — Tier-1 автономные действия локально и бесплатно.

**Сегмент 4 — B2B (Phase 4, v2.0)**
- *Боли (компания):* джуниоры отвлекают сеньоров; выгорание непрозрачно; данные не могут покидать контур.
- *Gain-creators:* Junior Mentor (onboarding-автоматизатор), Burnout Tracker (анонимная тепловая карта HR — на базе уже существующего Индекса Фрустрации), On-Premise с local-only mode (PII-sanitizer + локальный inference — данные не покидают площадку), Permission Tiers как governance-story для IT.

### 2.2. Продукт ↔ ценность ↔ платежи

| Что пользователь *чувствует* (free forever) | Что *стоит денег* (дорого для нас) |
|---|---|
| Голос, Barge-in <200 мс, персона «старший брат» | Frontier-модель качество ответов |
| Проактивность, Soft Widget, базовая память L0–L2 | Объём cloud-ответов в день |
| Локальные действия, Staging, Permission Tiers | Неограниченные SOP-Blueprints, MCP-хаб, multi-device sync |
| Фрустрация-детекция и check-in | Приоритетный cloud-модель, ночные консолидации длинных профилей |

Принцип: **fence on value density, not core value** — базовое ощущение «живого напарника» никогда за paywall (это retention-движок и word-of-mouth); платят за качество frontier-моделей, объём и интеграции.

---

## 3. Контекст рынка и willingness-to-pay

Ценовые якоря детально разобраны в [04-Market-Competitor-Analysis]; здесь — только платежные ориентиры:

| Якорь | Цена | Роль для Archie |
|---|---|---|
| ChatGPT Plus | $20/мес | Верхний якорь; free-tier ChatGPT жёстко rate-limited — канонический пример COGS-фенсинга |
| GitHub Copilot | $10–19/мес | Якорь dev-сегмента; student-free — прямая конкуренция за нашего студента |
| Cursor / Replit Pro | $20/мес | Якорь «agentic AI делает работу за тебя» — поддерживает Pro $15–25 |
| Raycast AI | free + BYOK / paid cloud | Ближайший структурный аналог (app-first hybrid) |
| Yandex Plus / GigaChat | RU-бандлы; API ~1–8 ₽/1K токенов [uncertain] | RU-якорь; GigaChat — вариант RU-cloud для 152-FZ-локализации |
| Character.AI / Moxie-класс | $10–15/мес, churn 8–20%/мес [uncertain] | Evidence эмоциональной монетизации и её волатильности |

**RU vs EN:** паритетной цены не делаем — RU-лист в ₽ (299–599 ₽ ≈ $3–6.5 [assumption]), EN в USD. RU-студент платит меньше, но CAC тоже ниже (комьюнити-каналы, сарафан ЕГЭ-сезона). Двухпрайсовая модель — стандарт для RU-founded продуктов, но требует двух юрлиц и двух платёжных рельсов (см. раздел 11, санкционный риск).

---

## 4. B2C pricing architecture: freemium с metered cloud LLM

### 4.1. Экономика токенов — почему freemium структурно жизнеспособен

Ключ из RFC-0001/PAD: экран превращается в **JSON-граф** (лёгкий текстовый слепок: `active_window`, `visible_text_snippet`, `error_state`), а не в скриншоты. Контекстный сниппет одного запроса — **300–800 токенов in**, ответ — **200–500 токенов out**. Внутренний монолог (запрос при фрустрации) — короткий: «Юзер застрял на IndentationError 15 минут, вздыхает. Что предложить?» — **100–200 токенов in, 150–300 out** [all assumptions, инструментация Phase 1 уточнит].

**Cost per call по классам моделей [uncertain — цены 2026, волатильны]:**

| Класс | $/1M in | $/1M out | Типовой запрос (500 in / 300 out) |
|---|---|---|---|
| Cloud small (GPT-4o-mini / GigaChat-класс) | ~$0.15 | ~$0.60 | **$0.000255** |
| Cloud frontier (GPT-4o / топ-класс) | ~$2.5 | ~$10 | **$0.00425** |
| Локальная ONNX/GGUF 1.5–3B | $0 | $0 | $0 (CPU, в RAM-бюджете) |

Ночные консолидации памяти (L0→L1→L2 суммаризация): batch, ~2K in / 500 out на small-модели ≈ $0.0006/ночь ≈ $0.018/мес [assumption]. Генерация SOP-Blueprint — разовое frontier-событие (~3K in / 800 out ≈ $0.0155), **амортизируется**: после утверждения рецепта роутер с косинус-совпадением ≥ 0.98 исполняет его локальным скриптом — ноль cloud-затрат на каждый последующий запуск. Это уникальный для Archie паттерн: *использование растёт, COGS — нет.*

**Персоны: blended cloud cost на активного пользователя в месяц [assumptions]:**

| Персона | Активность | Cloud-вызовы/день | Профиль (small/frontier) | COGS/мес |
|---|---|---|---|---|
| Студент light | 30 мин/д | ~5 | 5×small | ~$0.04 |
| Студент EGE-heavy | 2 ч/д | ~25 | 20 small + 5 frontier | ~$0.75 |
| Junior dev | 4 ч/д | ~40 (20 SOP-хиты — local, $0) | 15 small + 5 frontier | ~$3.10 |
| Геймер | 1 ч/д | ~3 | 3×small | ~$0.02 |
| **Free-tier blended (cap 20 cloud-ответов/день)** | — | ≤ 20 small | роутер принудительно small | **≤ $0.30** |

Даже при ×10 buffer на злоупотребления free-пользователь остаётся в пределах $0.30/мес — за счёт (а) JSON-графа вместо картинок, (б) коротких inner-monologue запросов, (в) жёсткого cap в семантическом роутере, (г) SOP-амортизации. Cap = **N = 20 cloud-quality ответов/день [uncertain]** — инженерится в сам роутер (RFC-0001), с graceful degradation на локальную модель, а не «policy-страницей».

### 4.2. Тиры

| | **Free** | **Plus** | **Pro** |
|---|---|---|---|
| Цена RU / EN | 0 ₽ / $0 | 299 ₽ / $9.99/мес | 599 ₽ / $19.99/мес |
| Annual (2 мес free) | — | 2 990 ₽ / $99.9 | 5 990 ₽ / $199.9 |
| Локальный движок (голос, персона, память, Tier-1/2 действия, Staging) | ✅ без лимитов | ✅ | ✅ |
| Cloud-quality ответы/день | 20 (small-модель) | 200 (small + frontier mix) | без жёсткого cap, fair-use + авто-throttle при COGS > 30% цены тира |
| Frontier-модель ответы | — | 20/день | приоритетно |
| Активные SOP-Blueprints | 2 | 15 | неограниченно |
| MCP-хаб (Obsidian/Notion/Jira/Slack/git) | Obsidian только | ✅ | ✅ + кастомные MCP-серверы |
| Multi-device memory sync | — | 2 устройства | 5 устройств |
| Ночная консолидация L1/L2 | базовая | ✅ | расширенная (длинные профили) |
| BYOK (свой API-ключ OpenAI/GigaChat/DeepSeek) | — | — | ✅ (COGS → $0) |
| Поддержка | community | email | priority |

Годовые планы — с первого дня публичной Beta: улучшают cash flow и churn-математику. Региональное ценообразование — тоже с первого дня. Lifetime-лицензия local-only (~$40–80) — **open question**, в модель не включена.

### 4.3. Margin waterfall на тир [assumptions]

| Строка | Plus (EN, $9.99) | Pro (EN, $19.99) |
|---|---|---|
| Цена | $9.99 | $19.99 |
| Cloud COGS (post-routing; heavy-user buffer 2×) | $1.20 | $3.50 |
| Платёжка (direct + Paddle MoR ~5% + процессинг) | $0.85 | $1.70 |
| App-store fee | $0 (direct-download-first: Tauri updater + свой checkout) | $0 |
| Support/инфра alloc | $0.40 | $0.60 |
| **Gross margin** | **75%** | **71%** |

RU-тиры: 299 ₽ ≈ $3.2 при COGS ~$1.0 → margin ~60% [uncertain, зависит от курса]; частично компенсируется дешёвым GigaChat-роутингом внутри RU. Продажа через stores (15–30%) отложена — распределение = финансовое решение, дефолт direct.

**SLA-консистентность с RFC-0001 (стоимостная политика не ломает перфоманс):** интерактивные ходы (barge-in <200 мс) — всегда local; cloud — только неинтерактивная генерация; OCR ~200 мс — local; консолидация памяти — batch off-peak (сглаживает API-spend и ценовые пики).

---

## 5. B2B licensing (Phase 4, v2.0)

Архитектурно B2B — продукт-делита поверх существующего: on-premise = local-only mode уже поддержан дизайном (PII-sanitizer + локальный inference), новые модули — SSO, audit log, admin console.

| Deployment | Basis | Inference | Data boundary | Цена [uncertain] |
|---|---|---|---|---|
| SaaS per-seat | $/seat/мес, мин. 10 seats | vendor-hosted или BYOK | DPA, санитизация на edge | $12–18/seat/мес |
| VPC | годовая лицензия + 20–22% support | BYOK / VLM в VPC клиента | данные в контуре клиента | $25–40/seat/мес |
| On-premise (air-gapped) | site license annual + 20% support | полностью local (бандл vLLM + RU-open-weights — open question) | ничего не покидает площадку | $30–60/seat или $25–50K/год site |
| **Burnout Tracker add-on** | поверх любого deployment | local (Индекс Фрустрации уже считается на edge) | только анонимизированная агрегированная heatmap для HR | **$2–5/seat/мес** |

Burnout Tracker — маржинальный аддон с почти нулевым COGS (индекс считается локально по жёсткой эвристике RFC-0001; порог 50, decay 20/15/5/1 мин) — но чувствителен по compliance: EU AI Act может классифицировать emotion inference на рабочих местах как ограниченную категорию → в EN-сегменте продаём как voluntary self-improvement wellness с opt-in, не surveillance. Позиционирование — критично (см. [04-Market]).

**Контрактная механика:** 90-дневный платный pilot с прописанными success criteria → annual. Cофинансирование enterprise-фич зарезервировано в seed (раздел 9) — известная ловушка «B2B pivot funded too late».

---

## 6. Unit Economics

Inputs по CAC — из [06-GTM]; здесь потребляются как данность. LTV = ARPU × contribution margin / churn.

| Метрика | Free→Plus cohort | Plus→Pro upgrade | B2B seat |
|---|---|---|---|
| CAC blended [assumption, GTM doc] | $8–12 (community-first) | ~$0 (in-app) | $400–800 (outbound+events) |
| Activation (Path A/B onboarding) | 40% [uncertain] | — | — |
| Conversion | 3–5% | 15% от Plus-базы | 25% от пилотов |
| ARPU/мес (blended RU+EN) | $5.5 [uncertain] | $15 | $15 |
| Contribution margin | 65–75% | 70% | 85% |
| Churn/мес (год) | 10% (stress 20%) | 8% | годовой logo churn ≤15% |
| **LTV** | **$36–41** (base) / $18 (stress) | **$131** | **$5 100** |
| **LTV/CAC** | **3.4–5** (base) / 1.7 (stress) | >10 | 6–12 |
| **Payback CAC** | **2–3 мес** (base) | мгновенно | **7–9 мес** |

Чувствительность: LTV рушится не от CAC, а от churn — при 20%/мес base-модель LTV/CAC падает ниже 2. Поэтому leading indicator LTV — **proactivity acceptance rate** (доля принятых вмешательств Soft Widget; окно отмены 3–5 с — встроенный измеритель) + frustration-check-in acceptance: эмоциональная непрерывность (память L0–L2, персона) — retention-moat, атакующий churn напрямую.

---

## 7. 3-летний финансовый projection-скелет

Команда: **4 чел (Phase 1–2) → 6 (v1.0) → 8–10 (v2.0)**. RU-команда, burn $15–40K/мес [assumption]. Квартальная гранулярность; полная таблица — артефакт `docs/business/financial-model.xlsx` с числами, ссылающимися на реестр допущений по ID (A1, A2…).

| Строка (тыс. $) | Y1 (Phase 1–2: Alpha→Beta) | Y2 (Phase 3: v1.0) | Y3 (Phase 4: v2.0) |
|---|---|---|---|
| MAU (конец года) | 10K | 60K | 150K |
| Платящие B2C | ~50 (Beta Plus) | 2 400 | 7 500 |
| ARPU/год | ~$50 | $66 | $72 |
| **B2C revenue** | 2.5 | 158 | 540 |
| B2B seats/licenses | 0 | 0–1 pilot | 500 seats + 1 site license |
| **B2B revenue** | 0 | 5 | 90 |
| **Revenue итого** | **2.5** | **163** | **630** |
| COGS (LLM API + платежи + CDN моделей + support) | 1.5 | 33 | 95 |
| **Gross profit / margin** | 1.0 / 40% | 130 / 80% | 535 / 85% |
| Payroll+taxes (команда 4→6→9) | 260 | 400 | 620 |
| Cloud fixed + tooling + CDN | 25 | 45 | 90 |
| Legal/compliance (152-FZ, GDPR, MoR, SOC 2 с Y3) | 15 | 25 | 70 |
| Marketing | 10 | 60 | 120 |
| **EBITDA** | **−310** | **−400** | **−390** |
| Накопленный burn | −310 | −710 | −1 100 |

**Сценарии (конец Y3):**

| | Bear | Base | Bull |
|---|---|---|---|
| MAU | 40K | 150K | 400K |
| Conversion | 2% | 3.5% | 6% |
| Churn/мес | 18% | 10% | 7% |
| Revenue Y3 | $150K | $630K | $2.2M |
| B2B | 0 (pivot не взлетел) | 1 pilot + 500 seats | 3 site licenses |
| Runway (при seed $500K + rev) | кончился в Y2 → нужен bridge/A-раунд earlier | кончился Y3Q1 → A-round на данных | self-sustainable к Y3Q4 |

Bear-сценарий — не катастрофа, а триггерная механика: pivot-триггеры (раздел 11) переводят engineering на B2B-фичи заранее.

---

## 8. Cost structure

| Категория | Y1 | Y3 | Примечание |
|---|---|---|---|
| Payroll + налоги (60–75% opex) | $260K | $620K | RU-зарплаты: senior Rust/ML $3–6K/мес, mid $2–3.5K [uncertain] |
| LLM API (variable) | ~$1K | ~$60K | роутер = margin-политика; метрика COGS/rev ≤25% (Beta) → ≤15% (v1.0) |
| Infra: CDN моделей (1–4 GB артефактов ONNX/GGUF [uncertain]), auth, update, telemetry | $10K | $40K | Tauri updater self-hosted ≈ $0; usage-based cloud, без reserved capacity до v1.0 |
| Tooling | ~$0 | ~$5K | OSS-дисциплина: PostHog self-hosted, GitHub Actions, LibreOffice/Sheets |
| Legal/compliance | $15K | $70K | 152-FZ-локализация (минимизирована local-first), GDPR DPIA/consent, MoR-настройка, SOC 2 Type I+II ($20–60K, Y3) [uncertain], два юрлица |
| Payments fees | % от rev | % | Paddle ~5%+ (EN, MoR снимает VAT-головную боль), CloudPayments/YooKassa ~3–5% [uncertain] (RU recurring) |

Дисциплина pre-revenue: fixed costs — в людях; всё остальное usage-based. Per-cohort margin telemetry с Beta-дня-1: cloud COGS на пользователя **измеряется** (каждый cloud-вызов логируется на event bus: tokens in/out/model/user-id, агрегация nightly), не моделируется; алерт при COGS > 30% rev когорты → роутер авто-поджимает cap.

---

## 9. Funding needs и use of funds

**Seed: $300–600K [uncertain], 18–24 мес runway, milestone-gated.** Спрос сформулирован на майлстоунах, не времени: «seed funds Alpha → 10K MAU Beta → 1K paying + 1 B2B pilot».

| Транш | Триггер | Сумма | Use of funds |
|---|---|---|---|
| T1 ($250K) | закрытие раунда (Alpha build) | 60% headcount, 15% infra/CDN, 15% compliance, 10% contingency | Phase 1: инструментация per-persona usage (calls/day by task type, tokens/call, local-vs-cloud ratio — эти данные де-рискуют весь план) |
| T2 ($150K) | Alpha метрики: proactivity acceptance > 25%, retention D14 > 20% [uncertain] | Phase 2: B2C Beta + caps live + Plus tier | |
| T3 ($100–200K) | Beta: 10K MAU, conversion ≥ 2% | v1.0 Pro + annual + **резерв на Phase 4 enterprise-build (SSO/audit/on-prem installer)** — pre-committed | |

Dilution guidance — вне scope ([uncertain], founder+legal). Точка спора при raise: до Beta (безопаснее runway, хуже терм) vs после (лучше термы с retention-proof, execution risk) — открытый вопрос для founder risk-appetite. Alternativa: pre-seed $100–150K на Alpha + beta-LOIs, seed post-Beta.

---

## 10. Break-even

При fixed base $25K/мес (команда 6, v1.0) и gross margin 75%:
- Требуемый MRR = $25K / 0.75 ≈ **$33K/мес**.
- При blended ARPU $10: **~3.3K платящих** → при conversion 3.5% — **~95K MAU**; при 5% — ~66K MAU [assumptions].
- Базовый план достигает этого в Y3; bull — в Y3Q3; с burn 8–10 чел ($40K/мес) break-even сдвигается на ~5.3K платящих → A-round обязателен. B2B ускоряет: один site-license $35K ≈ год fixed-cost покрытия одной строкой.

**Честная оговорка:** на seed-этапе цель — не operating break-even, а дешёвые доказательства юнит-экономики (LTV/CAC ≥ 3 на когортах, COGS/rev в норме) для A-раунда.

---

## 11. Риски, чувствительности, хеджи

| Риск | Вероятность/impact | Хедж |
|---|---|---|
| **LLM price volatility** (цены в модели — срез 2026) | Средняя / высокий | Сенситивность: −50% → gross margin +4–6 п.п.; **+100% → margin падает до ~55%** на Pro — план выживает за счёт: (а) multi-vendor роутинг (OpenAI/DeepSeek/GigaChat swappable per region), (б) BYOK, (в) **local-only survival mode**: v1.0 обязан функционировать degraded-but-usable на локальных моделях — RU-санкционный хедж + инвест-аргумент |
| **Heavy users (top 1–5% = 10–50× mean)** | Высокая / средний | Caps в роутере + staging-FS-квоты + авто-throttle при COGS>30% + BYOK self-selection (тяжёлые приносят ключи сами) + fair-use в ToS |
| **Churn 2×** (consumer AI 8–20%/мес) | Средняя / высокий | Stress-test встроен; frustration-check-in механика как retention-инструмент; annual-планы; leading-indicator мониторинг |
| **RU платёжные рельсы / санкции** (Stripe недоступен RU-энтити [uncertain]) | Средняя / высокий | Две сущности (RU для ₽/CloudPayments, foreign для $/Paddle) с дня-1; GigaChat как RU-cloud; local-only survival mode |
| **Store fees 15–30%** | Решение, не риск | Direct-download-first |
| **B2C CAC рост / conversion < 2%** | Средняя / высокий | **Pre-committed pivot-триггеры:** CAC > $15 или conversion < 2% через 6 мес после Beta → engineering shift на B2B (SSO, audit, on-prem); резерв T3 earmarked |
| «Free while in Beta» burn-trap | Высокая / средний | Caps shipped **вместе** с публичной Beta, не после (HN/YouTube-спайк убьёт неограниченный demo-tier) |

**Ассумпшн-дисциплина:** модель строится до цен (не наоборот — ловушка «скопировали $20, power-user стоил $40»); каждый номер в реестре допущений имеет source/benchmark, confidence H/M/L, sensitivity-impact, owner, review date; quarterly refresh; PR-ами обновляем assumptions по ID.

---

## 12. Revenue diversification (долгосрочная картина)

1. **B2C подписки** (Y1Q4+): Free→Plus→Pro, annual; RU+EN dual pricing. Доля Y3: ~85% rev.
2. **B2B лицензии** (Y3): per-seat SaaS, VPC, on-prem site license + 20% support, Burnout Tracker add-on (почти чистая margin). Доля Y3: ~15%, растёт быстрее всех.
3. **Marketplace MCP-плагинов** (Y3+, exploratory): community MCP-серверы и SOP-template-библиотеки; monetization — rev-share 20–30% на paid-плагинах (паттерн Raycast Store/Obsidian). В модель не включена (0$), но архитектурно (MCP-хаб, contracts/ в монорепо) — задел.
4. **BYOK features-only fee** ($5/мес [open question]) — нулевая COGS-опция.

---

## 13. KPI-дашборд (first-class alongside MRR/ARR)

- Gross margin per tier + per cohort (COGS/rev, алерт > 30%);
- Blended cloud COGS per active user (free vs paid);
- LTV/CAC и payback по когортам/каналам (a16z-notation);
- **Proactivity acceptance rate** — leading indicator LTV (встроен в Soft Widget cancel-window);
- Conversion free→paid, churn/мес, annual-share;
- Router local-vs-cloud ratio (target: > 80% запросов local на v1.0 [assumption]).

---

## 14. Реестр допущений (выжимка; полный — в financial-model.xlsx)

| ID | Допущение | Значение | Источник | Confidence | Sensitivity |
|---|---|---|---|---|---|
| A1 | Free-tier cap, cloud-ответов/день | 20 | estimate (RFC-0001 роутер) | M | высокий |
| A2 | Cloud small: $/1M in/out | $0.15/$0.60 | vendor price lists 2026 [uncertain] | M | высокий |
| A3 | Cloud frontier: $/1M in/out | $2.5/$10 | то же | M | высокий |
| A4 | Tokens/call (JSON-граф сниппет) | 300–800 in | RFC-0001/PAD architecture | H | средний |
| A5 | Conversion free→paid | 3–5% | benchmark (freemium 2–5%, AI-sticky до 8–10%) | M | высокий |
| A6 | B2C churn/мес | 10% target, stress 20% | benchmark (companion AI 8–20%) | L | критический |
| A7 | RU Plus/Pro | 299/599 ₽ | RU-якоря [uncertain] | M | средний |
| A8 | EN Plus/Pro | $9.99/$19.99 | ChatGPT/Copilot/Cursor якоря | H | средний |
| A9 | B2B seat/site | $12–18/$25–50K | enterprise norm [uncertain] | L | средний |
| A10 | Seed | $300–600K | estimate | M | высокий |
| A11 | Burn (команда 4–6) | $15–40K/мес | RU-market salaries [uncertain] | M | высокий |
| A12 | CAC blended | $8–12 | GTM doc (community-first) | M | высокий |
| A13 | Per-persona usage (calls/day) | таблица 4.1 | **измерить в Phase 1 Alpha** — главный де-рискер | L | критический |
| A14 | SOC 2 | $20–60K, 3–6 мес | industry [uncertain] | M | низкий (Y3) |

**Ключевой вывод реестра:** три критических uncertain (A5, A6, A13) закрываются не консенсусом, а **инструментированным Alpha Phase 1** — ровно поэтому seed-транш T1 спланирован вокруг измерений, а не роста. Финансовая модель здесь — каркас, который Alpha наполняет числами.

---

## Приложение: чек-лист согласованности

- [x] Числа RFC-0001 использованы как канон (150/655/1450 МБ, barge-in <200 мс, OCR ~200 мс, порог 50, cosine ≥0.98, окно 3–5 с, ~8 МБ инсталлятор);
- [x] Фазы 1–4 синхронны с README-конвенцией и планом Y1–Y3;
- [x] Feature fence привязан к модулям по именам RFC (semantic router, staging FS, MCP hub, memory L0–L2);
- [x] CAC/conversion потребляются из [06-GTM], ценовые якоря — из [04-Market], без дублирования;
- [x] Все load-bearing оценки помечены [assumption]/[uncertain] и завязаны на реестр A1–A14;
- [ ] Financial model xlsx + pricing page drafts — отдельные артефакты `docs/business/` (todo).
