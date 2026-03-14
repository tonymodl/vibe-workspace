# TASK_004: Voice Pipeline — архитектура роутера STT/LLM/TTS
**Статус:** READY_FOR_ANTIGRAVITY
**Ветка:** feature/task-004-voice-pipeline-router
**Спецификация:** docs/specs/SPEC_001_voicezettel_2_0_architecture.md (Модуль 2)
**Зависит от:** TASK_001

## Контекст
Все STT/LLM/TTS провайдеры — подключаемые модули с единым интерфейсом.
Переключение на лету без перезагрузки. Lazy-инициализация.
Критичная проблема speaker1991: настройки не применялись на ходу.

## Задача
1. Создать TypeScript интерфейсы провайдеров:
   - `STTProvider`: startStream, stopStream, supportedLanguages, supportsVAD, supportsDiarization
   - `LLMProvider`: chat (AsyncGenerator), maxContext, isLocal, costPerInputToken, costPerOutputToken
   - `TTSProvider`: synthesize (returns AudioStream), voices[], supportsEmotions, costPerChar
2. Создать VoicePipelineManager (`src/lib/voice-pipeline/manager.ts`):
   - Управляет lifecycle всех провайдеров
   - switchSTT/switchLLM/switchTTS — graceful shutdown + init нового
   - applyPreset — переключает все три за раз
3. Создать 4 пресета:
   - "lightning": Deepgram + Groq/Llama4 + Cartesia
   - "optimal": Deepgram + GPT-4o + ElevenLabs
   - "emotional": Deepgram + GPT-4o + Cartesia emotion API
   - "local": Whisper + Ollama + Piper
4. Реализовать barge-in (перебивание):
   - VAD detection → flush audio buffer → abort LLM → close TTS
5. Создать Zustand store: `src/stores/voicePipelineStore.ts`
6. Создать реестр провайдеров: `src/providers/registry.ts`

## Файлы для создания/изменения
- `src/types/voice-pipeline.ts` — все интерфейсы
- `src/lib/voice-pipeline/manager.ts` — VoicePipelineManager
- `src/lib/voice-pipeline/presets.ts` — 4 пресета
- `src/lib/voice-pipeline/barge-in.ts` — механизм перебивания
- `src/providers/registry.ts` — реестр всех провайдеров
- `src/providers/stt/deepgram.ts` — заглушка Deepgram
- `src/providers/llm/groq.ts` — заглушка Groq
- `src/providers/tts/cartesia.ts` — заглушка Cartesia
- `src/stores/voicePipelineStore.ts`

## Acceptance Criteria
- [ ] Все интерфейсы STTProvider/LLMProvider/TTSProvider определены в types
- [ ] VoicePipelineManager создаётся с конфигом пресета
- [ ] switchSTT/switchLLM/switchTTS возвращают Promise<void> и работают без перезагрузки
- [ ] applyPreset переключает все 3 провайдера за раз
- [ ] Barge-in: метод interrupt() очищает все буферы
- [ ] Zustand store содержит: activeSTT, activeLLM, activeTTS, activeVoice, activePreset
- [ ] Реестр провайдеров: getProvider(type, name) возвращает экземпляр
- [ ] Заглушки провайдеров реализуют интерфейсы (mock data)
- [ ] Тесты: unit-тест на переключение провайдеров

## Стек
TypeScript, Zustand, WebSocket (для будущего Deepgram/Cartesia)