# TASK_010: ChromaDB — per-user коллекции + RAG pipeline
**Статус:** READY_FOR_ANTIGRAVITY
**Ветка:** feature/task-010-chromadb-memory
**Спецификация:** docs/specs/SPEC_001_voicezettel_2_0_architecture.md — Модули 4.2, 4.3, 4.4

## Контекст
ChromaDB — долговременная память VoiceZettel. Каждый пользователь имеет изолированные коллекции. RAG: каждая фраза → embed → поиск контекста → сборка промпта. Shared memory между всеми LLM провайдерами.

## Задача
1. Запуск ChromaDB сервера (Docker или standalone)
2. Клиент ChromaDB для Node.js:
   - Singleton подключение
   - Автосоздание коллекций при первом обращении
3. 6 коллекций per user (namespace = user_id):
   - `{user_id}_conversations` — история разговоров
   - `{user_id}_notes` — заметки из Obsidian
   - `{user_id}_facts` — факты о пользователе
   - `{user_id}_entities` — персоны, проекты, локации
   - `{user_id}_voice_prints` — голосовые отпечатки (pyannote embeddings, 192-dim)
   - `{user_id}_strategic` — стратегическая информация (от Шелестуна)
4. RAG pipeline:
   - `embed(text)` → вектор (OpenAI text-embedding-3-small или local)
   - `search(query, collections, topK=10)` → relevant documents
   - `buildContext(searchResults, systemPrompt, directives)` → full prompt
   - `save(question, answer, metadata)` → в conversations
5. Embedding fallback: OpenAI → local sentence-transformers

## Файлы для создания/изменения
- `src/lib/chromadb/client.ts` — подключение к ChromaDB
- `src/lib/chromadb/collections.ts` — управление коллекциями
- `src/lib/chromadb/embedding.ts` — embed text (OpenAI / local)
- `src/lib/rag/pipeline.ts` — RAG pipeline (search → build context → save)
- `src/lib/rag/context-builder.ts` — сборка промпта с контекстом
- `src/types/memory.ts` — TypeScript типы для коллекций

## Acceptance Criteria
- [ ] ChromaDB сервер запускается и доступен по localhost:8000
- [ ] Коллекции создаются автоматически при первом обращении пользователя
- [ ] Embedding работает: текст → 1536-dim вектор (OpenAI) или 384-dim (local)
- [ ] RAG search возвращает top-10 релевантных документов
- [ ] Context builder собирает промпт: system + directives + RAG + history
- [ ] save() сохраняет пару (вопрос, ответ) в conversations
- [ ] Изоляция: пользователь A не видит данные пользователя B
- [ ] Voice prints коллекция принимает 192-dim embeddings (для pyannote)
- [ ] Unit тесты: embed, search, save, isolation

## Стек
chromadb npm, OpenAI embeddings, sentence-transformers (fallback)