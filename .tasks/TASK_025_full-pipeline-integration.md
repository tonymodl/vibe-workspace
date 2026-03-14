# TASK_025: Полная интеграция голосового пайплайна (end-to-end)
**Статус:** READY_FOR_ANTIGRAVITY
**Ветка:** feature/task-025-full-pipeline-integration
**Спецификация:** docs/specs/SPEC_001_voicezettel_2_0_architecture.md — Модуль 3

## Контекст
После создания отдельных компонентов (STT, LLM, TTS, Memory, Barge-in) — нужно собрать полный end-to-end pipeline. Пользователь говорит → текст → контекст → ответ → озвучка. Полный цикл ≈380-600ms.

## Задача
1. Voice Pipeline Manager:
   - Оркестрирует весь поток: mic → STT → RAG → LLM → TTS → speaker
   - WebSocket соединение клиент ↔ сервер
   - Streaming: STT interim → LLM streaming → TTS streaming
   - Barge-in: интеграция с перебиванием
2. Pipeline Lifecycle:
   - Init: warm-up микрофон, pre-connect WebSockets
   - Active: полный поток в реальном времени
   - Paused: микрофон muted, pipeline на паузе
   - Switching: переключение провайдера (graceful)
3. Latency optimization:
   - STT interim results → начать LLM ДО окончания фразы (predictive)
   - LLM streaming → начать TTS НЕ дожидаясь полного ответа (sentence-by-sentence)
   - TTS streaming → воспроизведение первых chunks ДО полной генерации
4. Integration тесты:
   - Полный цикл: аудио → текст → ответ → аудио
   - Fallback: при ошибке одного звена → fallback → цикл продолжается
   - Barge-in: перебивание в середине ответа → новый цикл

## Файлы для создания/изменения
- `src/lib/voice-pipeline/pipeline-manager.ts` — главный оркестратор
- `src/lib/voice-pipeline/pipeline-websocket.ts` — WebSocket server/client
- `src/lib/voice-pipeline/latency-optimizer.ts` — предиктивный запуск
- `src/app/api/voice/ws/route.ts` — WebSocket endpoint
- `tests/integration/full-pipeline.test.ts` — integration тесты
- `tests/e2e/voice-flow.spec.ts` — Playwright E2E тест

## Acceptance Criteria
- [ ] Полный цикл работает: говоришь → слышишь ответ (end-to-end)
- [ ] Латентность полного цикла: <600ms (от конца фразы до начала ответа)
- [ ] Streaming: ответ начинает звучать до полной генерации LLM
- [ ] Barge-in: перебивание в середине ответа работает за <100ms
- [ ] Fallback: при ошибке Deepgram → Whisper, цикл не прерывается
- [ ] WebSocket стабилен: reconnect при обрыве
- [ ] Переключение режима (assistant/lapel) не ломает pipeline
- [ ] Integration тесты проходят
- [ ] Метрики: latency каждого этапа логируется

## Стек
WebSocket, все компоненты из TASK_005-007, TASK_010-011
