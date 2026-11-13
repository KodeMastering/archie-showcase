# STATE.md — паспорт проекта для следующей Claude-сессии
Обновлено: 2026-11-07. Источник истины по деталям — этот файл; README.md — для людей.

## Состояние сейчас
- **Проект живой и стабильный**: e2e-пайплайн работает (окна → миссии → фрустрация → Z-Sense OCR → aijuju LLM → голос после окна отмены → TTS → SYS_TTS_STOP).
- **HEAD `f632f4d`** (голос после окна отмены + 8 фиксов adversarial-ревью), перед ним `0a257bd` — большой Phase-1 цикл (память/ночной rebuild/VAD-onnx/local-LLM/whisper). Рабочее дерево чистое.
- **Тесты: 251 passed** (`services-archie-engine`, venv `bin/python`, 15 с). Разбивка: proactivity 76 (+18 фиксы ревью), audio 33, memory 32, zsense 26, semantic 25, nightly 25, llm 18, stt 16.
- **Сервисы**: `systemctl --user`: `archie-app` (active, enabled; последний рестарт 19:25 07.11 — но по заданию оркестратора считать 11:28 07.11) + `archie-vite` (active).
- **RAM воркера**: ~2.4 ГБ RSS (pid 4677; Qwen3-1.7B-Q8_0 один инстанс через `llama_shared.shared_llama`).

## Архитектура в двух словах
- Монорепо: `apps-desktop-ui/` (Tauri v2, Rust `src-tauri/` = шина+хуки+окна; React 19 + TS фронт) + `services-archie-engine/` (Python 3.12 venv `bin/python`, `src/archie_engine/` = движок) + `contracts/` (proto + gen_py).
- IPC: gRPC поверх TCP loopback (port-file `/tmp/archie-ipc-<uid>.port`). UDS не работает — grpcio/tonic несовместимы (зафиксировано в README).
- Миссии: поллинг `hyprctl activewindow -j` каждые 2с → SemanticClassifier (Qwen3-1.7B few-shot, `llama_shared` синглтон) → fallback-эвристика (conf < 0.55 / `ARCHIE_SEMANTIC_ENABLED=0`).
- LLM: провайдер **aijuju** (OpenAI-совместимый, `https://api.aijuju.rest/v1`, qwen3.8-flash) — дефолт; каскад graceful degradation **aijuju → local → mock** до конца сессии при GatewayTimeout. `local` = офлайн-персона на Qwen3-1.7B GGUF. Gemini geo-block RU — не использовать.
- Память: Tier-1 буфер → Tier-2 эпизоды (SQLite + FTS5-trigram, свёртка Qwen few-shot с naive fallback; episodic-маркер + orphan-recovery) → Tier-3 `user_profile` (ночной rebuild 04:00 + catch-up); `build_memory_block()` подмешивает профиль + 3 эпизода в промпт персоны.
- Аудио: arecord 16k/mono → SileroGate VAD (**onnxruntime, torch выкинут**) с энергия-предфильтром → BargeInLoop (half-duplex) → STT (**sherpa-onnx ИЛИ faster-whisper**, `ARCHIE_STT_BACKEND`) → TTS piper (ru_RU-ruslan-medium); голос Арчи звучит после окна беззвучной отмены 4 с, конец речи = событие SYS_TTS_STOP.

**Текущий Phase 1 MVP против целевой архитектуры**

> **Арчи — гибридный Edge+Cloud ассистент (не полностью локальный):**
>   - Локально работают сенсорика (VAD, Z-Sense), память, базовый PII-санитайзер (скелет) и офлайн-фолбэк (`local`-провайдер, Qwen3-1.7B GGUF).
>   - Облачный LLM-контур (`aijuju`) — **конфигурируемый и необязательный** (opt-in, ключ `ARCHIE_AIJUJU_API_KEY` в `run/env`, дефолт конфига — `aijuju` но без ключа → воркер деградирует в `local` и затем в `mock`).
>   - Базовая конфигурация — **офлайн**: `mock` без сети/ключей → `local` (офлайн-персона Qwen) → `mock` при таймауте.
>   - Облачный контур — **optional**: gemini геоблокирован в RU → не используется, aijuju доступен только при наличии ключа (`ARCHIE_AIJUJU_API_KEY` в `run/env`).
>   - Ночной rebuild профиля работает локально; память, миссии и frustration-профиль не отправляются в cloud path.
>   - Сырые кадры экрана и raw audio предназначены оставаться локально; через cloud boundary допускается только санитизированный текстовый контекст при проверенных sanitizer/egress-контролях. Текущий sanitizer — базовый scaffold, не полный regex+NER.
>   - Telemetry, crash upload, analytics и support bundle в Phase 1 не реализованы; они остаются planned/conditional opt-in.

> **Current Phase 1 MVP против архитектуры RFC-0001:**
>   • Full-duplex с AEC3 — нет (half-duplex).
>   • Shared-memory ring buffer — нет (gRPC TCP).
>   • Z-Sense OS-hooks — нет (poll-модель `mss` + OCR).
>   • Миссии OS-hooks — нет (поллинг `hyprctl`).
>   • PII-санитайзер — базовый скелет (модуль `security/`, не финальный Regex+NER).
>   • Permission Tiers + sandbox — нет (карантин-каталог есть, блокировка Delete — нет).
>   • Semantic Router + SOP — нет.
>   • Full-data sanitation на аудио-путях — нет (санитайзер — только на облачном LLM и telemetry; аудио — local-only с VAD).
>   • Always-listening с облаком — нет (конфиг опционален, включая облачный провайдер aijuju).
>   • Full telemetry — нет (только local logs + opt-in-но crash/аналитика — планируется в Phase 2).
>   • On-Prem/VPC — не реализовано, только условные Enterprise v2.0.

> **Summary:** Phase 1 MVP — гибридный Edge+Cloud ассистент с местом для облачного контура (`aijuju` опционален), локальным fallback (`local`, `mock`), базовым санитайзером и полным офлайн-режимом в стартовом конфиге. Не fully-local, не always-cloud. Инваринты (RFC: Barge-in < 200 мс, OCR ~200 мс, RAM 150/655/1450 МБ) **не заявляются** из-за упрощений (см. README §3).

## Как проверять
```bash
systemctl --user status archie-app                       # состояние воркера
journalctl --user -u archie-app -f                       # живой лог
cd services-archie-engine && bin/python -m pytest tests/ -q        # 251 passed
bin/python -m archie_engine.semantic.smoke               # живой смоук семантики
#  (нужен ARCHIE_SEMANTIC_MODEL=Qwen3-1.7B-Q8_0.gguf; см. юнит)
cd apps-desktop-ui && pnpm exec tsc --noEmit && pnpm exec vite build # UI-гейт
cargo check                                              # при правках src-tauri (не трогали)
```
Ключи лежат в `~/.config/systemd/user/archie-app.service` (`ARCHIE_AIJUJU_API_KEY`) и `run/env` — оба gitignored, **не вставлять в STATE.md** (файл в git).

## Сделано (цикл 0a257bd + f632f4d, сверен с git log)
- **Контракт NIGHTLY_PROFILE_REBUILD**: EventType=16, ProfileRebuildPayload (oneof=20) сквозь proto → gen_py → Rust events.rs → TS bridge → EventsTab (фиолетовый бейдж).
- **Память/жизненный цикл (P2)**: episode_marker в SQLite + orphan-recovery открытого эпизода после падения воркера; слияние коротких эпизодов; journal-восстановление через get_tier1_between.
- **_nightly_loop**: in-process ночной rebuild профиля (04:00) с catch-up (профиль пуст/старше 20 ч), эмитом NIGHTLY_PROFILE_REBUILD, отключение по env (`ARCHIE_NIGHTLY_REBUILD_ENABLED/_HOUR`).
- **SIGTERM-graceful**: корректный выход из run(), обработка сигнала только в MainThread.
- **VAD на onnxruntime**: silero sequence-модель, ленивая сессия, фолбэк importlib; torch выкинут из зависимостей (~2 ГБ экономии).
- **local_provider**: офлайн-персона на Qwen GGUF через locked_completion; ветка "local" в cloud_gateway; GatewayTimeout при недоступной модели → mock.
- **locked_completion**: инференс-лок shared-инстанса llama_shared — фикс гонки «консолидация в executor vs классификация в event-loop» (data race внутри llama.cpp).
- **WhisperBackend (P3)**: faster-whisper ct2 int8 (~460 МБ, `models/whisper-ru-small/`), батчевое распознавание + проводка on_speech_chunk из BargeInLoop в feed() с эхо-защитой.
- **Голос после окна отмены 4 с**: реплика Арчи звучит только после беззвучного окна (accept/timeout/нет UI); `_voice_reply_cancelled` — фикс CRITICAL-гонки «отмена раньше ответа LLM» (отмена глушит TTS, SYS_TTS_SPEAK не публикуется).
- **SYS_TTS_STOP после конца речи**: воркер трекает utterance-пачку (_tts_reply_ids) и эмитит SYS_TTS_STOP когда все реплики отзвучали — SoftWidget не виснет в 'speaking'.
- **Срез TriggerZSenseOcr из Python**: Rust-stub остаётся для совместимости proto, реальный OCR в Python-фолбэке.
- **8 фиксов adversarial-ревью** (worker.py): cancel-гонка; UnboundLocalError в _on_mic_sigh при None-скорере; gateway-таймаут реплики уважают окно отмены; SYS_TTS_STOP после речи; сбой SQLite не глушит sigh-триггер; перезапись _pending_voice_reply отменяет старый фолбэк-таймер; local→local каскад → сразу mock; честный triggering_heuristic для sigh-монолога.
- Дефолты конфига: aijuju/qwen3.8-flash, Qwen3-1.7B, провайдер local в choices.

## Не доделано
- **UX-верификация на живом Tauri**: SoftWidget 'speaking'→idle-orb по SYS_TTS_STOP — Rust-путь готов (lib.rs) и TS-обработка добавлена, но живой проверки не было.
- **Живой смоук Qwen-персоны**: в тестах мок, реального прогона local-провайдера не делали.
- **Whisper latency** на фразах >10 с: батчевое распознавание, задержка может быть заметной — не меряли.
- **finish_utterance по концу речевого рана**: сейчас 2с-детект тишины — при длинной речи режет посреди фразы.
- **_tts_reply_ids очистка** в force_stop-путях: при force_stop набор может копиться.
- **ARCHIE_STT_BACKEND=whisper** не включён в дефолт-юните (ручной смоук ок).

## Идеи дальше
- Полноценный голосовой диалог (barge-in уже есть; полный дуплекс AEC3 — Phase 2).
- Реакция на фрустрацию как отдельный поведенческий слой.
- Z-Sense: событийные OS-hooks вместо поллинга (Phase 2).

## Подводные камни
- **GateGuard-хук (ECC plugin)**: перед первым Edit в новой сессии требует предъявить факты — сначала Read/grep, потом правки.
- **429-квота API при длинных workflow**: resumeFromRunId-кэш спасает; каскад aijuju→local→mock — запасная аэродромка.
- **Whisper-галлюцинации на неречевом аудио** (синус → «Субтитры…»): VAD отсекает неречевые сегменты.
- **.venv на Python 3.14 сломан**: uv add создал; рабочий venv — `bin/` на 3.12.
- **uv add кладёт в основную секцию pyproject**: переносить в optional-dependencies вручную.
- **DC-офсет микрофона**: не убирать HPF в детектере вздохов — внутренний микрофон на 100% gain даёт «тихий» RMS 0.26 и ложные вздохи.
- **models/**: `Qwen3-0.6B-Q8_0.gguf` оставлен на диске для отката.
- **Не перезапускать GDM/демоны из активной Hyprland-сессии** (системный урок, чёрный экран).
- Ключи только в юните и `run/env`, оба gitignored; в git их не тащить.
