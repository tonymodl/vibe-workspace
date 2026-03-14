# TASK_007: TTS Router — Multi-provider + Barge-in
**Статус:** READY_FOR_ANTIGRAVITY
**Ветка:** feature/task-007-voice-pipeline-tts
**Спецификация:** docs/specs/SPEC_001_voicezettel_2_0_architecture.md — Модуль 3.3, 3.5

## Контекст
TTS озвучивает ответы ИИ. 6 провайдеров с auto-fallback. Критично: механизм перебивания (barge-in) — пользователь начинает говорить → мгновенная остановка озвучки.

## Задача
1. TTS Router с провайдерами:
   - Cartesia Sonic 3 (TTFB 40ms, WebSocket streaming)
   - ElevenLabs Flash v2.5 (75ms, WebSocket)
   - Google Wavenet (REST)
   - Yandex SpeechKit (REST, голос alena)
   - Qwen3-TTS local (localhost:8880, REST)
   - Piper TTS local (REST/CLI)
2. Единый интерфейс:
   - `synthesize(text, voice, options)` → AudioStream
   - Streaming: первые chunks аудио приходят до окончания генерации
3. Barge-in (перебивание):
   - VAD детектирует голос пользователя → INTERRUPT сигнал
   - Flush audio playback buffer (мгновенно)
   - Abort текущий TTS stream (AbortController / WebSocket close)
   - Abort текущий LLM stream
   - Время от детекции до остановки: <100ms
4. Audio playback:
   - Web Audio API AudioBufferSourceNode
   - Queue chunks для плавного воспроизведения
   - Volume control
5. Голос по умолчанию: женский
6. Переключение провайдера на лету (без перезагрузки)

## Файлы для создания/изменения
- `src/lib/voice-pipeline/tts-router.ts` — маршрутизатор TTS
- `src/lib/voice-pipeline/providers/cartesia.ts` — Cartesia клиент
- `src/lib/voice-pipeline/providers/elevenlabs-tts.ts` — ElevenLabs клиент
- `src/lib/voice-pipeline/providers/google-tts.ts` — Google Wavenet
- `src/lib/voice-pipeline/providers/yandex-tts.ts` — Yandex SpeechKit
- `src/lib/voice-pipeline/providers/qwen-tts.ts` — Qwen3-TTS local
- `src/lib/voice-pipeline/providers/piper-tts.ts` — Piper local
- `src/lib/voice-pipeline/barge-in.ts` — механизм перебивания
- `src/lib/voice-pipeline/audio-player.ts` — воспроизведение аудио

## Acceptance Criteria
- [ ] Все 6 TTS провайдеров генерируют речь из текста
- [ ] Streaming: первый chunk аудио воспроизводится ДО окончания генерации
- [ ] Barge-in: при начале речи пользователя аудио останавливается за <100ms
- [ ] Auto-fallback: Cartesia → ElevenLabs → Google → Yandex → Qwen3-TTS → Piper
- [ ] Русский язык звучит естественно на всех провайдерах
- [ ] Голос по умолчанию — женский
- [ ] Переключение провайдера без перезагрузки страницы
- [ ] Token/cost tracking для каждого TTS запроса
- [ ] Unit тесты: barge-in timing, fallback logic

## Стек
Web Audio API, WebSocket (Cartesia, ElevenLabs), REST APIs