# TASK_006: Memory Layer — адаптивный системный промпт + ChromaDB RAG
**Статус:** READY_FOR_ANTIGRAVITY
**Ветка:** feature/task-006-memory-layer
**Спецификация:** docs/specs/SPEC_001_voicezettel_2_0_architecture.md (Модуль 3)
**Зависит от:** TASK_001, TASK_002, TASK_004

## Контекст
Самая важная фича: ассистент ВСЕГДА помнит контекст разговора и поправки пользователя навсегда.
Когда пользователь говорит "не повторяй мой ответ" — это добавляется в системный промпт НАВСЕГДА.
Общая память для всех мозгов (LLM) — при переключении LLM память сохраняется.

## Задача
1. Установить: chromadb (JS client), openai (for embeddings)
2. Создать Context Manager (`src/lib/memory/context-manager.ts`):
   - Short-term: последние 50 сообщений в RAM
   - Long-term: ChromaDB per-user коллекции
   - Adaptive System Prompt: директивы из SQLite
3. Создать ChromaDB клиент (`src/lib/memory/chroma-client.ts`):
   - Инициализация per-user коллекций: conversations, notes, facts, entities, voice_prints, strategic
   - addDocument, search, delete
4. Создать Prompt Builder (`src/lib/memory/prompt-builder.ts`):
   - Собирает: base prompt + user directives + RAG context + short-term + текущая фраза
5. Создать Directive Detector (`src/lib/memory/directive-detector.ts`):
   - LLM classifier: определяет является ли фраза пользователя поправкой/директивой
   - Если да → INSERT INTO user_directives
   - Противоположная команда → деактивировать старую + добавить новую
6. API route: GET/POST /api/memory/directives — для просмотра/ручного добавления

## Файлы для создания/изменения
- `src/lib/memory/context-manager.ts`
- `src/lib/memory/chroma-client.ts`
- `src/lib/memory/prompt-builder.ts`
- `src/lib/memory/directive-detector.ts`
- `src/app/api/memory/directives/route.ts`
- `src/types/memory.ts`

## Acceptance Criteria
- [ ] ChromaDB клиент подключается к localhost:8000
- [ ] Per-user коллекции создаются автоматически при первом запросе
- [ ] Prompt Builder собирает полный контекст из 3 источников
- [ ] Директивы сохраняются в SQLite и попадают в системный промпт
- [ ] Directive Detector распознаёт поправки ("не повторяй", "говори короче", "обращайся на ты")
- [ ] При переключении LLM — память не теряется
- [ ] Тест: добавление директивы и проверка её в собранном промпте

## Стек
ChromaDB JS client, OpenAI embeddings (text-embedding-3-small), better-sqlite3, TypeScript