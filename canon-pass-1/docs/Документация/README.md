# Archie — Полный комплект документации проекта

**Archie («Архи»)** — гибридный edge+cloud проактивный ИИ-напарник уровня ОС (OS-Level AI Symbiote).

> **Current Phase 1 MVP vs Target/Roadmap.** Целевые документы ниже описывают дорожную архитектуру и не равны текущему runtime. В Phase 1 локально работают VAD/STT/Piper TTS, Z-Sense, SQLite/FTS5-память, nightly rebuild, local Qwen и mock/offline fallback. Cloud LLM aijuju конфигурируем и opt-in; при timeout действует cascade aijuju → local → mock. Raw screen/audio предназначены оставаться local-only; через cloud boundary допускается только санитизированный текст при проверенных controls. Текущий sanitizer — базовый scaffold, не полный regex+NER. Telemetry, crash upload, analytics и support bundle в Phase 1 не реализованы; On-Prem/VPC — только conditional Enterprise v2.0.

Полное target-видение: полнодуплексный голос с Barge-in (<200 мс), событийный Z-Sense OCR (~200 мс на CPU), трёхуровневая память, Semantic Router SOP-рецептов, Permission Tiers и карантинная зона Archie_Staging. Эти свойства привязаны к соответствующим фазам и не заявляются как достигнутые MVP.

## Структура комплекта

### Исходные документы (в `../` — уровни проработки от концепта к RFC)
| # | Документ | Уровень |
|---|----------|---------|
| 1 | `Archie.docx` — Product & Architecture Document (PAD) | Бизнес-концепт |
| 2 | `Архитектурный документ проекта Archie.docx` — системная архитектура | Инженерная спецификация |
| 3 | `Архитектурный RFC платформы Archie.docx` — RFC-0001 | Производственная спецификация (IPC, sequence, RAM, decay) |

### Документы этого комплекта (`docs/`)
| # | Файл | Назначение |
|---|------|-----------|
| 4 | `01-PRD.md` | Продуктовые требования: персоны, user stories, NFR, приёмка по фазам |
| 5 | `02-UX-Voice-UX.md` | UX/UI и голосовой UX: виджет, онбординг, персона, состояния |
| 6 | `03-Threat-Model-Security.md` | Модель угроз (STRIDE), безопасность SOP-песочницы, PII, ключи |
| 7 | `04-Market-Competitor-Analysis.md` | Анализ рынка и конкурентов, позиционирование |
| 8 | `05-Business-Model-Financial-Plan.md` | Бизнес-модель, юнит-экономика, финансовый план |
| 9 | `06-Go-to-Market-Strategy.md` | GTM: каналы, метрики активации/удержания, pivot-триггеры |
| 10 | `07-Testing-QA-Strategy.md` | Стратегия тестирования: пирамида, SLA-тесты, CI-гейты |
| 11 | `08-Project-Plan-Team.md` | План проекта: WBS, команда, RACI, реестр рисков |
| 12 | `09-CI-CD-Distribution.md` | CI/CD, кроссплатформенные сборки, подписи, автообновление |
| 13 | `10-Observability-Telemetry-Privacy.md` | Наблюдаемость: телеметрия, краш-репорты, privacy |
| 14 | `11-External-API-MCP-Contracts.md` | Внешние API и MCP-контракты: плагины, версионирование |
| 15 | `12-ADR-Glossary.md` | ADR-лог решений, глоссарий терминов проекта |
| 16 | `13-README-Quickstart.md` | Корневой README: быстрый старт, структура репо, контрибьюция |

## Правила согласованности (обязательны для всех документов)

1. **Числа из RFC-0001 — канон**: Barge-in < 200 мс; VAD детекция 85–100 мс
   (точность > 95%); OCR ~200 мс (185 мс эталон); RAM: 150 МБ (Idle) / 655 МБ
   (Peak) / 1450 МБ (Offline); порог фрустрации 50 (S_max = 100, полураспад
   20/15/5/1 мин по миссиям); совпадение SOP-триггера >= 0.98; окно отмены
   Soft Widget 3–5 с; инсталлятор ~8 МБ (без весов моделей).
2. **Структура монорепо** — как в архитектурном документе: `apps/desktop-ui/`,
   `crates/{archie-shell, archie-event-bus, archie-os-hooks, archie-stagingfs}/`,
   `services/archie-engine/`, `contracts/`, `models/`, `storage/`.
3. **Фазы**: 1 — MVP Alpha (ядро, Z-Sense, голос); 2 — B2C Beta (проактивность,
   память, безопасность); 3 — v1.0 Production (Semantic Router, Blueprints);
   4 — v2.0 Enterprise (B2B, On-Premise, Burnout Tracker).
4. **Двуязычие**: документы пишутся на русском языке, технические термины и
   идентификаторы — на английском (как в исходных .docx).
