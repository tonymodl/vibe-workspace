# TASK_024: Счётчик токенов + Auto-Fallback система
**Статус:** READY_FOR_ANTIGRAVITY
**Ветка:** feature/task-024-token-tracking
**Спецификация:** docs/specs/SPEC_001_voicezettel_2_0_architecture.md — Модуль 16

## Контекст
Каждый запрос к API тратит токены и деньги. Нужен точный учёт per-model, per-user в USD и RUB. Auto-fallback при ошибке провайдера с уведомлением. Тарифы обновляются через админку.

## Задача
1. Token Counter middleware:
   - Перехватывает каждый запрос к STT/LLM/TTS
   - Считает токены (input/output для LLM, символы для TTS, секунды для STT)
   - Вычисляет стоимость по тарифам из provider_rates
   - Записывает в SQLite token_usage
2. Provider Rates CRUD:
   - Начальные тарифы всех провайдеров (из SPEC_001)
   - API для обновления тарифов (admin)
   - Курс USD/RUB (обновляемый)
3. Auto-Fallback Engine:
   - Обёртка для каждого pipeline (STT, LLM, TTS)
   - При ошибке → следующий провайдер по приоритету
   - Toast notification пользователю
   - Telegram notification admin
   - Автоматический retry упавшего через 5 мин
   - Логирование fallback event
4. Dashboard виджеты:
   - Расходы сегодня/неделя/месяц (USD + RUB)
   - Breakdown по провайдерам (pie chart)
   - Trend по дням (line chart)

## Файлы для создания/изменения
- `src/lib/voice-pipeline/token-counter.ts` — middleware подсчёта
- `src/lib/voice-pipeline/fallback-engine.ts` — auto-fallback логика
- `src/lib/voice-pipeline/cost-calculator.ts` — расчёт стоимости
- `src/app/api/admin/rates/route.ts` — API: CRUD тарифов
- `src/app/api/admin/costs/route.ts` — API: расходы и статистика
- `src/components/admin/CostDashboard.tsx` — виджеты расходов

## Acceptance Criteria
- [ ] Каждый запрос к LLM/TTS/STT записывается с tokens и cost_usd
- [ ] cost_rub рассчитывается по курсу
- [ ] Auto-fallback: при ошибке Groq → автоматически OpenAI
- [ ] Toast notification при fallback (видно пользователю)
- [ ] Telegram notification admin при fallback
- [ ] Retry через 5 мин для упавшего провайдера
- [ ] Dashboard: расходы по провайдерам (pie chart + line chart)
- [ ] Тарифы обновляемые через админку
- [ ] Unit тесты: cost calculation, fallback logic, retry timing

## Стек
SQLite, React (charts: Recharts), Telegram Bot API
