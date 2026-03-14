# TASK_006: LLM Router — Multi-provider с свободным приоритетом
**Статус:** READY_FOR_ANTIGRAVITY
**Ветка:** feature/task-006-voice-pipeline-llm
**Спецификация:** docs/specs/SPEC_001_voicezettel_2_0_architecture.md — Модуль 3.2

## Контекст
LLM Router управляет несколькими провайдерами. Пользователь СВОБОДНО выставляет приоритеты в любой конфигурации. Auto-fallback при ошибке. Новые модели добавляются через конфиг, старые — удаляются.

## Задача
1. LLM Router с провайдерами:
   - Groq (Llama 4 Maverick) — fast
   - OpenAI (GPT-4o) — smart
   - Gemini 2.5 Pro — smart
   - DeepSeek V3 — smart/code
   - Ollama (Qwen3-32B, Qwen2.5-Coder-32B) — local
2. Единый интерфейс для всех провайдеров:
   - `stream(messages, options)` → AsyncGenerator<string>
   - `complete(messages, options)` → string
   - Каждый провайдер реализует этот интерфейс
3. Приоритеты:
   - Пользователь задаёт порядок в настройках (массив providerIds)
   - При ошибке текущего → следующий по приоритету
   - Автоматический retry упавшего через 5 мин
4. Token counting:
   - Считать input/output токены для каждого запроса
   - Записывать в SQLite token_usage (cost_usd, cost_rub)
5. Streaming:
   - Все провайдеры поддерживают streaming ответов
   - Отправка токенов клиенту через WebSocket / SSE

## Файлы для создания/изменения
- `src/lib/voice-pipeline/llm-router.ts` — маршрутизатор LLM
- `src/lib/voice-pipeline/providers/groq.ts` — Groq клиент
- `src/lib/voice-pipeline/providers/openai.ts` — OpenAI клиент
- `src/lib/voice-pipeline/providers/gemini.ts` — Gemini клиент
- `src/lib/voice-pipeline/providers/deepseek.ts` — DeepSeek клиент
- `src/lib/voice-pipeline/providers/ollama.ts` — Ollama клиент
- `src/lib/voice-pipeline/token-counter.ts` — подсчёт токенов и стоимости
- `src/types/llm.ts` — TypeScript интерфейсы
- `src/app/api/llm/route.ts` — API endpoint (WebSocket/SSE streaming)

## Acceptance Criteria
- [ ] Все 5 провайдеров работают: отправка промпта → получение ответа
- [ ] Streaming работает для всех провайдеров (token by token)
- [ ] При ошибке Groq → автоматически OpenAI (по приоритету)
- [ ] Пользователь может менять приоритеты (Zustand + API)
- [ ] Token counting: каждый запрос записывается в SQLite с cost_usd
- [ ] Ollama работает локально (localhost:11434)
- [ ] Интерфейс провайдера универсален — новый провайдер = 1 файл
- [ ] Unit тесты: fallback logic, token counting, provider interface

## Стек
openai SDK, @google/generative-ai, Ollama REST API, WebSocket/SSE