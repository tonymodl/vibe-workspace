# TASK_003: Авторизация Google OAuth + Whitelist
**Статус:** READY_FOR_ANTIGRAVITY
**Ветка:** feature/task-003-auth-system
**Спецификация:** docs/specs/SPEC_001_voicezettel_2_0_architecture.md — Модуль 1

## Контекст
Авторизация через Google OAuth с белым списком email. НЕ используем Google Cloud Console Audience для ограничения — вместо этого whitelist в SQLite. Роли: admin (только evsinanton@gmail.com) и user.

## Задача
1. Настроить NextAuth.js v5 с Google провайдером
2. Middleware для проверки авторизации на всех /api/* и /admin/* маршрутах
3. Проверка email ∈ whitelist при каждом входе
4. Страница "Access Denied" для неавторизованных email
5. Страница входа с кнопкой "Войти через Google"
6. API endpoints для управления whitelist (только admin):
   - `POST /api/admin/whitelist` — добавить email
   - `DELETE /api/admin/whitelist/:email` — удалить
   - `GET /api/admin/whitelist` — список
7. Сессия содержит: email, name, role, avatar_url

## Файлы для создания/изменения
- `src/lib/auth.ts` — NextAuth конфигурация
- `src/middleware.ts` — проверка авторизации
- `src/app/auth/signin/page.tsx` — страница входа
- `src/app/auth/access-denied/page.tsx` — отказ в доступе
- `src/app/api/admin/whitelist/route.ts` — CRUD whitelist
- `src/types/next-auth.d.ts` — расширение типов сессии

## Acceptance Criteria
- [ ] Google OAuth работает: клик → Google → redirect → сессия
- [ ] Email не из whitelist → страница Access Denied
- [ ] evsinanton@gmail.com всегда admin, role сохраняется в сессии
- [ ] Middleware блокирует /api/* и /admin/* для неавторизованных
- [ ] API whitelist работает (добавить/удалить/список), доступен только admin
- [ ] Сессия содержит email, name, role, avatar_url
- [ ] При logout — полная очистка сессии
- [ ] Unit тесты: whitelist check, role assignment

## Стек
NextAuth.js v5, Google OAuth, better-sqlite3