# TASK_005: Админ-панель — базовая структура + управление пользователями
**Статус:** READY_FOR_ANTIGRAVITY
**Ветка:** feature/task-005-admin-panel-base
**Спецификация:** docs/specs/SPEC_001_voicezettel_2_0_architecture.md (Модуль 6)
**Зависит от:** TASK_001, TASK_002

## Контекст
Админ-панель — центр управления проектом. Доступна только пользователям с role='admin'.
7 разделов: Пользователи, Файлы, Логи, Аналитика, API баланс, Сущности, Настройки.
В этой задаче реализуем каркас + раздел Пользователи.

## Задача
1. Создать маршрут `/admin` с layout:
   - Боковая навигация с 7 разделами (иконки + названия)
   - Проверка роли: если не admin → редирект на /
2. Раздел "Пользователи" (`/admin/users`):
   - Таблица пользователей: email, имя, аватар, роль, статус (allowed/blocked), последний вход
   - Кнопка "Добавить пользователя": модалка с полем email → INSERT с is_allowed=1
   - Переключатель роли: user ↔ admin (dropdown)
   - Кнопка "Заблокировать/Разблокировать" → toggle is_allowed
   - Поиск и сортировка по колонкам
3. API routes для управления пользователями:
   - GET /api/admin/users — список
   - POST /api/admin/users — добавить
   - PATCH /api/admin/users/[id] — обновить роль/статус
   - DELETE /api/admin/users/[id] — удалить
4. Все API routes проверяют: текущий пользователь — admin

## Файлы для создания/изменения
- `src/app/admin/layout.tsx` — layout админки
- `src/app/admin/page.tsx` — дашборд
- `src/app/admin/users/page.tsx` — управление пользователями
- `src/components/Admin/Sidebar.tsx` — боковая навигация
- `src/components/Admin/UserTable.tsx` — таблица пользователей
- `src/components/Admin/AddUserModal.tsx` — модалка добавления
- `src/app/api/admin/users/route.ts` — GET + POST
- `src/app/api/admin/users/[id]/route.ts` — PATCH + DELETE
- `src/lib/admin-auth.ts` — проверка прав админа

## Acceptance Criteria
- [ ] /admin доступен только для role='admin', иначе редирект
- [ ] Боковая навигация с 7 разделами (5 — заглушки с "Скоро")
- [ ] Таблица пользователей показывает всех из SQLite
- [ ] Кнопка "Добавить" открывает модалку, после ввода email — пользователь появляется в таблице
- [ ] Переключатель роли работает (admin ↔ user)
- [ ] Блокировка/разблокировка работает
- [ ] Нельзя заблокировать самого себя
- [ ] Нельзя снять роль admin если остался единственный admin
- [ ] API routes возвращают 403 для не-админов
- [ ] Тест: API route /api/admin/users возвращает список

## Стек
Next.js App Router, shadcn/ui (Table, Dialog, Button, Badge), better-sqlite3, Tailwind CSS