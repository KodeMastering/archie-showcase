# Archie Showcase

Артефакты мега-задачи Archie: лендинг, презентация, видео, диаграмма архитектуры и дизайн приложения.

## Канон продукта

Archie — **гибридная edge+cloud-система**, а не полностью локальный продукт. На edge выполняются VAD/STT/Piper TTS, SQLite/FTS5 memory с nightly rebuild, локальная Qwen3-1.7B, Z-Sense capture/OCR/basic PII sanitizer, loopback IPC и mock/offline fallback. Cloud LLM (aijuju) — configurable/opt-in; каскад: `aijuju → local → mock`.

Raw screen frames, raw audio и local frustration profile остаются локальными. Через cloud boundary проходит только sanitized textual context; текущий sanitizer — **базовый scaffold, не доказанная гарантия**. Telemetry в Phase 1 не реализована. On-prem/VPC — только условная Enterprise v2.0 capability.

Видео заморожено после canon correction по инструкции пользователя; дальнейшие improvement passes не перерендеривали видео.

## Продукты

- **landing** — лендинг уровня Awwwards: орб-симбионт, кинетическая типографика, WebAudio
- **presentation** — питч-дек 13 слайдов: навигация с клавиатуры, живые графики, barge-in демо
- **diagram** — интерактивная SVG-диаграмма архитектуры: 17 узлов, сценарии «голосовой запрос» и «ночной rebuild»
- **app-design** — кликабельный прототип дашборда + Soft Widget орб с 5 состояниями
- **video** — промо-ролик 60 с: HyperFrames-композиция + ffmpeg-саундтрек + edge-tts закадр; frozen after canon correction

## Snapshot archive

Каждый snapshot сохраняется отдельно, чтобы прежние версии оставались воспроизводимыми и сравнимыми.

| Папка | Что хранит |
|---|---|
| `v1/` | Первая production-версия (wave 2), включая v1 HTML-варианты, app design и v1 video assets. |
| `v2/` | Версия после improvement pass 1 (wave 3): полировка, ревью и отдельные v2-варианты; также сохранены финальные варианты wave 3. |
| `final/` | Исторический финал wave 4 до canon archive: landing, presentation, diagram, app-design и video. |
| `canon-canon/` | Canon correction snapshot: hybrid edge+cloud facts, полный комплект пяти продуктов, документация и frozen MP4. |
| `canon-pass-1/` | Improvement pass 1 поверх canon correction: пять продуктов, документация, frozen MP4 и HyperFrames/vendor assets. |
| `canon-pass-2/` | Improvement pass 2: обновлённые продукты и docs, HyperFrames package/vendor; frozen video retained. |
| `canon-pass-3/` | Improvement pass 3: обновлённые продукты, полный `Документация/`, frozen MP4 и HyperFrames/vendor assets. |
| `canon-pass-4/` | Preserved archive copy of the pass-3 product set, created to complete the five-pass archive; video remains frozen. |
| `canon-pass-5/` | Preserved archive copy of the pass-3 product set, created to complete the five-pass archive; video remains frozen. |
| `canon-final/` | Release snapshot assembled from the canonical set with current `.artifacts` landing, presentation, diagram and app-design products; frozen canon video retained. |

`v1/`, `v2/` и `final/` намеренно сохранены. Canon archive folders are additive and do not replace earlier versions.

## Design system

Палитра (dark-first, без оранжевого): bg `#07080F`, ирис `#8B7CFF`, циан `#22D3EE`, бирюза `#2DD4BF`, роза `#FB7185`.
Шрифты: Unbounded (дисплей), Manrope (текст), JetBrains Mono (код/метрики).

## GitHub limits audit

Проверка перед release: файлов свыше GitHub hard limit 100 MB не обнаружено. MP4-файлы в snapshots находятся ниже лимита; Git LFS не требуется. Ничего не было отброшено из-за лимитов.

---
*Собрано агентами Claude Code в ходе мега-задачи Archie.*
