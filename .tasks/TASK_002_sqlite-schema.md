# TASK_002: SQLite схема и подключение
**Статус:** READY_FOR_ANTIGRAVITY
**Ветка:** feature/task-002-sqlite-schema
**Спецификация:** docs/specs/SPEC_001_voicezettel_2_0_architecture.md — Модули 1, 4, 5, 12, 16

## Контекст
SQLite — единственная реляционная БД проекта. Хранит пользователей, директивы, логи, токены, дайджесты. Нужно создать ВСЕ таблицы из спецификации сразу, чтобы другие модули могли использовать.

## Задача
1. Настроить better-sqlite3 с singleton-подключением
2. Создать migration system (простой, на основе SQL файлов)
3. Создать ВСЕ таблицы из SPEC_001:
   - `users` — пользователи
   - `email_whitelist` — белый список
   - `user_directives` — адаптивный системный промпт
   - `activity_logs` — логи активности
   - `token_usage` — использование токенов
   - `provider_rates` — тарифы провайдеров
   - `daily_digests` — дайджесты петлицы
4. Создать TypeScript типы для каждой таблицы
5. Создать DAL (Data Access Layer) с CRUD операциями
6. Заполнить provider_rates начальными тарифами (из SPEC_001)
7. Добавить admin пользователя (evsinanton@gmail.com) в начальную миграцию

## Файлы для создания/изменения
- `src/lib/db.ts` — подключение SQLite, singleton
- `src/lib/migrations/001_initial.sql` — все таблицы
- `src/lib/migrations/002_seed.sql` — начальные данные
- `src/lib/dal/users.ts` — CRUD пользователей
- `src/lib/dal/directives.ts` — CRUD директив
- `src/lib/dal/logs.ts` — запись логов
- `src/lib/dal/tokens.ts` — учёт токенов
- `src/types/db.ts` — TypeScript типы таблиц

## Acceptance Criteria
- [ ] SQLite файл создаётся при первом запуске в `data/voicezettel.db`
- [ ] Все 7 таблиц из SPEC_001 созданы с правильными типами, FK, индексами
- [ ] DAL экспортирует типизированные функции для каждой таблицы
- [ ] Миграции применяются автоматически при запуске
- [ ] evsinanton@gmail.com создан как admin в seed
- [ ] Unit тесты на DAL (Jest): создание/чтение/обновление пользователя
- [ ] TypeScript типы соответствуют SQL схеме 1:1

## Стек
better-sqlite3, TypeScript, Jest