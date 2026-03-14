# TASK_017: Telegram бот администрирования
**Статус:** READY_FOR_ANTIGRAVITY
**Ветка:** feature/task-017-telegram-bot
**Спецификация:** docs/specs/SPEC_001_voicezettel_2_0_architecture.md — Модуль 11

## Контекст
Telegram бот — удалённое управление сервером. Кнопки проверки статуса, перезагрузки, управления пользователями. Автоматические уведомления при ошибках. Только для admin.

## Задача
1. Бот на Telegraf.js:
   - Webhook или polling (настраиваемо)
   - Авторизация: только Telegram ID admin'а
2. Команды:
   - `/status` — статус всех сервисов + CPU/RAM/GPU
   - `/health` — здоровье пайплайна
   - `/restart <service>` — перезагрузка сервиса
   - `/reboot` — перезагрузка сервера
   - `/users` — список пользователей
   - `/whitelist add/remove <email>` — управление whitelist
   - `/logs <service> <lines>` — последние N строк логов
   - `/errors` — ошибки за последний час
   - `/costs` — расходы (сегодня/неделя/месяц)
   - `/backup now` — бэкап немедленно
3. Inline keyboard:
   - Быстрые кнопки: Status | Restart | Costs
   - Перезагрузка: Next.js | ChromaDB | Ollama | ALL
4. Автоматические уведомления:
   - Сервис упал → сообщение
   - CPU >90% / RAM >85% / VRAM >95% → alert
   - Fallback сработал → информация
   - Превышение бюджета → warning
   - Бэкап: успех/ошибка

## Файлы для создания/изменения
- `src/modules/telegram-bot/bot.ts` — инициализация бота
- `src/modules/telegram-bot/commands/status.ts` — /status
- `src/modules/telegram-bot/commands/restart.ts` — /restart
- `src/modules/telegram-bot/commands/users.ts` — /users, /whitelist
- `src/modules/telegram-bot/commands/logs.ts` — /logs, /errors
- `src/modules/telegram-bot/commands/costs.ts` — /costs
- `src/modules/telegram-bot/commands/backup.ts` — /backup
- `src/modules/telegram-bot/notifications.ts` — автоуведомления
- `src/modules/telegram-bot/system-monitor.ts` — мониторинг CPU/RAM/GPU

## Acceptance Criteria
- [ ] Бот отвечает на /status с реальными данными (CPU, RAM, GPU, uptime)
- [ ] /restart перезагружает указанный сервис
- [ ] /whitelist add/remove работает
- [ ] Inline keyboard с кнопками быстрого доступа
- [ ] Автоуведомления: при падении сервиса → сообщение в Telegram
- [ ] Только admin Telegram ID может использовать бота
- [ ] /costs показывает реальные расходы из SQLite
- [ ] /logs показывает реальные логи из файлов
- [ ] Бот стартует вместе с приложением

## Стек
Telegraf.js, systeminformation (для CPU/RAM), nvidia-smi (для GPU)
