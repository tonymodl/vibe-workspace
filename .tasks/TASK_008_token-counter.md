# TASK_008: Счётчик токенов + API баланс + авто-fallback
**Статус:** READY_FOR_ANTIGRAVITY
**Ветка:** feature/task-008-token-counter
**Спецификация:** docs/specs/SPEC_001_voicezettel_2_0_architecture.md (Модуль 7)
**Зависит от:** TASK_004, TASK_005

## Контекст
Счётчик токенов в speaker1991 работает некорректно. Нам нужно:
- Считать токены для каждой активной модели
- Рубли и доллары по тарифам
- Показывать остаток на каждом API
- Авто-переключение на другой API при исчерпании

## Задача
1. Создать TokenTracker (`src/lib/billing/token-tracker.ts`):
   - Перехватывает каждый вызов STT/LLM/TTS
   - Считает токены/символы/секунды
   - Пересчитывает в доллары/рубли по PRICING таблице
2. Создать PRICING реестр (`src/lib/billing/pricing.ts`):
   - Groq: $0.20/1M input, $0.60/1M output
   - OpenAI GPT-4o: $2.50/$10.00 per 1M
   - Deepgram: $0.0043/min
   - Cartesia: $15/1M chars
   - ElevenLabs: $11/1M chars
   - Google Wavenet: $16/1M chars
   - Yandex: 152₽/1M chars
   - Редактируемые через админку
3. Создать BalanceChecker (`src/lib/billing/balance-checker.ts`):
   - Проверяет остаток на API (где есть endpoint)
   - Для остальных: ручной ввод баланса в админке
4. Создать FallbackChain (`src/lib/billing/fallback-chain.ts`):
   - TTS: Cartesia → ElevenLabs → Wavenet → Yandex → Piper
   - LLM: Groq → OpenAI → Gemini → Ollama
   - STT: Deepgram → Whisper local
   - Уведомление пользователю при переключении
5. UI: виджет на главной странице с текущим расходом (компактный)
6. Админ: раздел "API и баланс" с подробной таблицей

## Acceptance Criteria
- [ ] Каждый вызов API логируется с подсчётом токенов
- [ ] Стоимость считается в $ и ₽ по тарифам
- [ ] Остаток на каждом API виден в админке
- [ ] При исчерпании — автоматический fallback
- [ ] Пользователь видит уведомление о переключении
- [ ] Компактный виджет на главной: "≈$0.03 за сессию"
- [ ] Тарифы редактируются в админке

## Стек
TypeScript, better-sqlite3, Zustand