# TASK_005: STT Pipeline — Deepgram + Whisper Fallback
**Статус:** READY_FOR_ANTIGRAVITY
**Ветка:** feature/task-005-voice-pipeline-stt
**Спецификация:** docs/specs/SPEC_001_voicezettel_2_0_architecture.md — Модуль 3.1, 3.7

## Контекст
STT — первый компонент голосового пайплайна. Deepgram Nova-3 как primary (WebSocket streaming), Whisper как local fallback. Нужен быстрый старт микрофона и auto-fallback при ошибке Deepgram.

## Задача
1. Deepgram Nova-3 WebSocket клиент:
   - Streaming: отправка 16kHz mono PCM chunks
   - Параметры: `model=nova-3`, `language=ru`, `diarize=true`, `interim_results=true`
   - Обработка interim (частичных) и final результатов
   - Reconnect при обрыве соединения
2. Whisper fallback (faster-whisper через API endpoint):
   - `POST /api/stt/whisper` — принимает аудио chunk, возвращает текст
   - Модель: large-v3 или medium (настраиваемо)
3. STT Router:
   - Автоматический fallback: Deepgram → Whisper при ошибке
   - Уведомление при переключении (event emit)
   - Метрика: latency каждого запроса → SQLite activity_logs
4. Быстрый старт микрофона:
   - `getUserMedia()` после первого user interaction
   - Pre-connect к Deepgram WebSocket (warm-up)
   - AudioWorklet для обработки потока (16kHz, mono)
5. VAD (Voice Activity Detection):
   - Простой energy-based VAD на AudioWorklet
   - Не отправлять тишину в Deepgram (экономия)

## Файлы для создания/изменения
- `src/lib/voice-pipeline/stt-router.ts` — маршрутизатор STT
- `src/lib/voice-pipeline/providers/deepgram.ts` — Deepgram клиент
- `src/lib/voice-pipeline/providers/whisper.ts` — Whisper клиент
- `src/lib/voice-pipeline/audio-capture.ts` — getUserMedia + AudioWorklet
- `src/lib/voice-pipeline/vad.ts` — Voice Activity Detection
- `src/app/api/stt/whisper/route.ts` — API endpoint для Whisper
- `src/stores/pipeline-store.ts` — Zustand: текущий STT провайдер, статус

## Acceptance Criteria
- [ ] Deepgram streaming работает: говоришь → текст появляется в реальном времени
- [ ] Русский язык распознаётся корректно
- [ ] Diarization: Deepgram возвращает speaker labels (SPEAKER_0, SPEAKER_1)
- [ ] При отключении Deepgram → автоматический переключение на Whisper
- [ ] Whisper fallback работает: аудио → POST → текст
- [ ] VAD фильтрует тишину (не отправляет пустые chunks)
- [ ] Латентность логируется в SQLite (activity_logs)
- [ ] AudioWorklet обрабатывает поток в 16kHz mono
- [ ] Pre-connect к Deepgram при загрузке (после user interaction)
- [ ] Unit тесты: STT router fallback logic

## Стек
@deepgram/sdk, faster-whisper (Python process), AudioWorklet API, WebSocket