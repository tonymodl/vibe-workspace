# TASK_010: Админ-панель — Файлы + Логи + Аналитика
**Статус:** READY_FOR_ANTIGRAVITY
**Ветка:** feature/task-010-admin-files-logs-analytics
**Спецификация:** docs/specs/SPEC_001_voicezettel_2_0_architecture.md (Модуль 6)
**Зависит от:** TASK_005

## Контекст
Дополнительные разделы админ-панели: файловый менеджер, логи, аналитика.
Файловый менеджер нужен для Comet Agent и ручного редактирования.

## Задача
1. Файловый менеджер (`/admin/files`):
   - Дерево файлов слева
   - Monaco Editor справа (или CodeMirror)
   - Кнопка "Сохранить" (Ctrl+S)
   - API: GET/PUT /api/admin/files?path=...
2. Логи (`/admin/logs`):
   - Таблица из activity_logs
   - Фильтры: по пользователю, по типу, по дате
   - Пагинация
   - Экспорт в CSV
3. Аналитика (`/admin/analytics`):
   - Встроить Google Analytics 4 скрипт в layout
   - Встроить Яндекс Метрику скрипт в layout
   - Страница: iframe с GA + iframe с ЯМ (или ссылки)
   - Внутренняя аналитика: график запросов по дням, стоимость по API

## Acceptance Criteria
- [ ] Дерево файлов показывает структуру проекта
- [ ] Клик на файл открывает его в редакторе
- [ ] Ctrl+S сохраняет файл
- [ ] Логи отображаются с фильтрами
- [ ] GA и ЯМ скрипты подключены в layout
- [ ] График запросов по дням рендерится

## Стек
@monaco-editor/react, better-sqlite3, recharts, shadcn/ui