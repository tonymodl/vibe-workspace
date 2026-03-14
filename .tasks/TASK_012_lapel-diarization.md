# TASK_012: Режим петлицы — Diarization 20+ спикеров + идентификация
**Статус:** READY_FOR_ANTIGRAVITY
**Ветка:** feature/task-012-lapel-diarization
**Спецификация:** docs/specs/SPEC_001_voicezettel_2_0_architecture.md — Модуль 5.1, 5.2, 5.3

## Контекст
Петлица — режим записи и анализа разговоров с 20+ участниками. Гибридный подход: Deepgram для real-time STT + pyannote для идентификации по голосовым отпечаткам. Три состояния: Запись, Справочник, Участник.

## Задача
1. Интеграция pyannote-audio (Python sidecar process):
   - HTTP API endpoint для Node.js → Python
   - `POST /pyannote/embed` — извлечь 192-dim embedding из аудио chunk
   - `POST /pyannote/diarize` — полная diarization аудио файла
   - GPU: CUDA (RTX 4090)
   - Модель: pyannote/speaker-diarization-3.1
2. Гибридный pipeline:
   - Deepgram WebSocket → текст + SPEAKER_0, SPEAKER_1 (anonymous)
   - Параллельно: аудио chunks → pyannote embedding extraction
   - ChromaDB voice_prints: cosine similarity > 0.7 → "Это Миша!"
   - Если < 0.7 → "Speaker N" (новый голос, сохранить embedding)
3. Person Resolver:
   - Голосовой отпечаток (embeddings)
   - Контекстный анализ ("Давай, Миша" → LLM извлекает имя)
   - Ручное присвоение ("это Миша" → сохранить)
   - updateSpeakerName → обновить ВСЕ предыдущие записи
4. Три состояния петлицы:
   - Запись: пассивная транскрибация, только запись
   - Справочник: реакция на обращение по имени ассистента
   - Участник: активное участие, мозговой штурм
   - Переключение голосом или кнопкой

## Файлы для создания/изменения
- `src/modules/lapel/lapel-engine.ts` — основной движок петлицы
- `src/modules/lapel/diarization-hybrid.ts` — гибридный Deepgram + pyannote
- `src/modules/lapel/person-resolver.ts` — идентификация персон
- `src/modules/lapel/state-controller.ts` — 3 состояния
- `services/pyannote-api/server.py` — Python sidecar с FastAPI
- `services/pyannote-api/embeddings.py` — извлечение embeddings
- `services/pyannote-api/requirements.txt` — Python зависимости
- `src/stores/lapel-store.ts` — Zustand: спикеры, состояние, сообщения

## Acceptance Criteria
- [ ] pyannote Python sidecar запускается и отвечает по HTTP
- [ ] Deepgram streaming + diarization работают одновременно
- [ ] 20 разных спикеров корректно разделяются (тест с аудио)
- [ ] Voice prints сохраняются в ChromaDB, повторный спикер узнаётся
- [ ] Контекстный анализ: "Давай, Миша" → имя извлекается из текста
- [ ] Обновление имени задним числом: Speaker 3 → Миша во ВСЕХ сообщениях
- [ ] 3 состояния переключаются, в каждом разное поведение
- [ ] Латентность embedding extraction: <100ms на 500ms chunk (GPU)
- [ ] Unit тесты: person resolver, state transitions

## Стек
pyannote-audio 3.1, FastAPI (Python sidecar), Deepgram SDK, ChromaDB