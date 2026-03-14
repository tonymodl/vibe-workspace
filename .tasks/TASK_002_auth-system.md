# TASK_002: Авторизация Google OAuth + управление пользователями
**Статус:** READY_FOR_ANTIGRAVITY
**Ветка:** feature/task-002-auth-system
**Спецификация:** docs/specs/SPEC_001_voicezettel_2_0_architecture.md (Модуль 1)
**Зависит от:** TASK_001

## Контекст
Нужна авторизация через Google OAuth с белым списком email в SQLite.
Админ управляет доступом из админ-панели, НЕ через Google Cloud Console.
Сейчас в проекте speaker1991 пользователи добавляются через Google Cloud Console Audience — нам это не нужно.

## Задача
1. Установить: next-auth, better-sqlite3, @types/better-sqlite3
2. Создать SQLite схему (файл `src/lib/db.ts`):
   - Таблица `users`: id, email, name, avatar_url, role (admin/user), is_allowed, created_at, last_login, settings_json
   - Таблица `user_directives`: id, user_id, directive, source, created_at, is_active
   - Таблица `activity_logs`: id, user_id, event_type, event_data, tokens_used, cost_usd, provider, created_at
3. Миграция: предзаполнить evsinanton@gmail.com как admin
4. Настроить NextAuth с Google Provider (`src/app/api/auth/[...nextauth]/route.ts`)
5. Callback signIn: проверять email в таблице users → is_allowed === 1
6. Создать middleware для защиты маршрутов
7. Создать страницу логина `/login` с кнопкой "Войти через Google"
8. Создать страницу "Доступ запрещён" если email не в белом списке

## Файлы для создания/изменения
- `src/lib/db.ts` — инициализация SQLite + схема
- `src/app/api/auth/[...nextauth]/route.ts` — NextAuth config
- `src/middleware.ts` — защита маршрутов
- `src/app/login/page.tsx` — страница входа
- `src/app/denied/page.tsx` — страница отказа
- `src/stores/authStore.ts` — Zustand store для auth

## Acceptance Criteria
- [ ] Страница /login показывает кнопку "Войти через Google"
- [ ] После успешного входа — редирект на главную
- [ ] Email не в таблице users → страница "Доступ запрещён"
- [ ] SQLite файл создаётся автоматически при первом запуске в `data/voicezettel.db`
- [ ] evsinanton@gmail.com предзаполнен как admin, is_allowed=1
- [ ] Все маршруты кроме /login и /denied защищены middleware
- [ ] Session содержит: email, name, role, userId
- [ ] last_login обновляется при каждом входе

## Стек
NextAuth.js, Google OAuth 2.0, better-sqlite3, Zustand