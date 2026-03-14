# TASK_014: Админ-панель — Dashboard, пользователи, логи
**Статус:** READY_FOR_ANTIGRAVITY
**Ветка:** feature/task-014-admin-panel
**Спецификация:** docs/specs/SPEC_001_voicezettel_2_0_architecture.md — Модуль 12

## Контекст
Админ-панель — web UI для управления проектом. Доступна только admin (evsinanton@gmail.com). Dashboard с метриками, управление пользователями, просмотр логов, мониторинг расходов.

## Задача
1. Dashboard:
   - Статус сервисов (зелёный/красный): STT, LLM, TTS, ChromaDB, Ollama
   - GPU usage (VRAM, GPU%, temperature)
   - Активные пользователи сейчас
   - Расходы сегодня (USD + RUB)
   - Количество запросов за сегодня
2. Управление пользователями:
   - Таблица: email, name, role, last_login, is_active
   - Добавить email в whitelist
   - Заблокировать/разблокировать пользователя
   - Изменить роль (admin/user)
3. Логи:
   - Таблица с фильтрацией: event_type, user, время
   - Пагинация, поиск
   - Детальный просмотр event_data (JSON)
4. API & Баланс:
   - Per-model cost tracking (таблица + график)
   - Расходы по дням/неделям/месяцам
   - Топ провайдеров по расходам
5. Настройки проекта:
   - API ключи (маскированные)
   - Конфигурация провайдеров

## Файлы для создания/изменения
- `src/app/admin/layout.tsx` — layout админки
- `src/app/admin/page.tsx` — Dashboard
- `src/app/admin/users/page.tsx` — управление пользователями
- `src/app/admin/logs/page.tsx` — просмотр логов
- `src/app/admin/costs/page.tsx` — расходы и баланс
- `src/app/admin/settings/page.tsx` — настройки проекта
- `src/app/api/admin/stats/route.ts` — API: статистика dashboard
- `src/app/api/admin/logs/route.ts` — API: логи с фильтрацией
- `src/components/admin/ServiceStatus.tsx` — виджет статуса сервиса
- `src/components/admin/CostChart.tsx` — график расходов

## Acceptance Criteria
- [ ] Dashboard показывает реальные статусы сервисов (ping проверка)
- [ ] Таблица пользователей с CRUD операциями
- [ ] Whitelist: добавление/удаление email работает
- [ ] Логи: фильтрация по event_type, user, дате
- [ ] Расходы: таблица per-model + график по дням
- [ ] Только admin имеет доступ (middleware проверка role)
- [ ] Mobile-friendly layout
- [ ] Все данные из SQLite (реальные, не mock)

## Стек
Next.js App Router, shadcn/ui (Table, Card, Chart), Tailwind CSS, Recharts
