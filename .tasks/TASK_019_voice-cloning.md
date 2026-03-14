# TASK_019: Voice Cloning — ElevenLabs + XTTS + RVC
**Статус:** READY_FOR_ANTIGRAVITY
**Ветка:** feature/task-019-voice-cloning
**Спецификация:** docs/specs/SPEC_001_voicezettel_2_0_architecture.md — Модуль 3.4, Блок N

## Контекст
Клонирование ЛЮБОГО голоса — включая знаменитостей. ElevenLabs для cloud, XTTS v2 для локального, RVC v2 для voice conversion. Клонированный голос привязывается к агенту.

## Задача
1. ElevenLabs Instant Voice Cloning:
   - API: загрузка аудио сэмпла (10+ сек)
   - Создание клонированного голоса
   - Использование в TTS pipeline
   - Мультиязычный (русский поддерживается)
2. XTTS v2 (Coqui) — локальный:
   - Python sidecar process с FastAPI
   - `POST /xtts/clone` — загрузить сэмпл (6+ сек)
   - `POST /xtts/synthesize` — генерация речи клонированным голосом
   - GPU: RTX 4090, <200ms латентность
   - Русский язык
3. RVC v2 — voice conversion:
   - Python sidecar process
   - Обучение модели голоса (5 мин аудио)
   - Real-time конвертация голоса (любой → целевой)
   - 210x real-time на RTX 4090
4. UI для клонирования:
   - Запись сэмпла голоса (в настройках агента, Шаг 3)
   - Загрузка аудиофайла
   - Превью клонированного голоса
   - Привязка к агенту / использование как основной голос
5. Управление клонированными голосами:
   - Список голосов (SQLite)
   - Удаление
   - Привязка к агентам

## Файлы для создания/изменения
- `src/lib/voice-pipeline/providers/elevenlabs-clone.ts` — ElevenLabs clone API
- `services/xtts-api/server.py` — XTTS v2 FastAPI sidecar
- `services/rvc-api/server.py` — RVC v2 FastAPI sidecar
- `src/modules/voice-cloning/clone-manager.ts` — управление голосами
- `src/components/settings/VoiceClonePanel.tsx` — UI клонирования
- `src/app/api/voice-clone/route.ts` — API: загрузка сэмпла, создание клона

## Acceptance Criteria
- [ ] ElevenLabs: загрузка 10сек сэмпла → клонированный голос говорит по-русски
- [ ] XTTS v2: загрузка 6сек сэмпла → генерация речи <200ms на GPU
- [ ] RVC v2: обучение на 5 мин аудио → real-time voice conversion
- [ ] UI: запись/загрузка сэмпла, превью клонированного голоса
- [ ] Привязка голоса к агенту (Шаг 3 создания)
- [ ] Список клонированных голосов с управлением
- [ ] Русский язык поддерживается всеми методами
- [ ] Unit тесты: clone manager CRUD

## Стек
ElevenLabs API, XTTS v2 (Python), RVC v2 (Python), FastAPI
