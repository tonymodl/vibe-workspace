# TASK_016: Система агентов — создание (3 шага) + базовый оркестратор
**Статус:** READY_FOR_ANTIGRAVITY
**Ветка:** feature/task-016-agent-system
**Спецификация:** docs/specs/SPEC_001_voicezettel_2_0_architecture.md — Модуль 6.1, 6.2, 6.3, 6.4

## Контекст
Пользователь создаёт ИИ-агентов по 3 шагам: Экспертиза → Характер → Внешность. Каждый агент имеет свою RAG-базу знаний, характер, голос. Базовый оркестратор управляет моделями на GPU (VRAM 24GB).

## Задача
1. Agent CRUD:
   - Создание агента (3 шага)
   - Список агентов пользователя
   - Удаление/архивация агента
   - SQLite таблица agents
2. Шаг 1 — Экспертиза:
   - Промпт от пользователя (текст/голос)
   - Автоматический сбор источников (web search → URLs)
   - Ingestion: URL → текст → chunking → embed → ChromaDB (отдельная коллекция)
   - RAG Engine для Q&A по источникам (замена NotebookLM)
3. Шаг 2 — Характер:
   - Слайдеры: humor, empathy, directness, creativity, formality, patience (0-100)
   - Автоподбор на основе профиля пользователя
   - Имя агента (задаётся пользователем)
4. Шаг 3 — Внешность:
   - Выбор типа: женщина/мужчина/персонаж/существо
   - Привязка клонированного голоса
   - Placeholder для particle avatar (будет в TASK_018)
5. Базовый Model Orchestrator:
   - Маршрутизация задач: code → Qwen2.5-Coder, research → Qwen3
   - VRAM Manager: отслеживание загруженных моделей
   - Очередь задач с приоритетами
   - Выгрузка неиспользуемых моделей при нехватке VRAM

## Файлы для создания/изменения
- `src/modules/agents/agent-manager.ts` — CRUD агентов
- `src/modules/agents/expertise-builder.ts` — Шаг 1: сбор экспертизы
- `src/modules/agents/character-builder.ts` — Шаг 2: характер
- `src/modules/agents/appearance-builder.ts` — Шаг 3: внешность
- `src/modules/agents/rag-engine.ts` — RAG для агентов (замена NotebookLM)
- `src/modules/orchestrator/model-orchestrator.ts` — маршрутизация задач
- `src/modules/orchestrator/vram-manager.ts` — управление VRAM
- `src/modules/orchestrator/task-queue.ts` — очередь задач
- `src/app/agents/page.tsx` — UI: список агентов
- `src/app/agents/create/page.tsx` — UI: wizard создания (3 шага)
- `src/app/api/agents/route.ts` — API: CRUD агентов

## Acceptance Criteria
- [ ] Wizard создания агента: 3 шага, навигация вперёд/назад
- [ ] Шаг 1: промпт → автосбор источников → ingestion в ChromaDB
- [ ] Шаг 2: слайдеры характера работают, автоподбор предлагает значения
- [ ] Шаг 3: выбор типа внешности, привязка голоса
- [ ] Список агентов: карточки с именем, описанием, статусом
- [ ] RAG Engine: вопрос к агенту → ответ на основе его источников
- [ ] VRAM Manager: отслеживает загруженные модели, выгружает при нехватке
- [ ] Очередь задач: задачи выполняются в порядке приоритета
- [ ] Unit тесты: agent CRUD, VRAM management, task routing

## Стек
LangChain, ChromaDB, Ollama REST API, React 19, shadcn/ui
