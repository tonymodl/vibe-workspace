# TASK_015: Инфраструктура — Nginx, SSL, Fail2ban, Backup
**Статус:** READY_FOR_ANTIGRAVITY
**Ветка:** feature/task-015-infrastructure
**Спецификация:** docs/specs/SPEC_001_voicezettel_2_0_architecture.md — Модуль 14

## Контекст
Проект хостится на домашнем ПК с белым IP. Нужна надёжная инфраструктура: Nginx reverse proxy, SSL через Let's Encrypt, защита Fail2ban, автоматические бэкапы. НЕ Cloudflare.

## Задача
1. Nginx reverse proxy:
   - voicezettel.online → localhost:3000 (Next.js)
   - WebSocket proxy (для Deepgram, TTS streaming)
   - Static files cache headers
   - Rate limiting (100 req/min per IP)
   - Gzip compression
2. SSL/TLS:
   - Let's Encrypt через certbot + nginx plugin
   - Auto-renewal через cron (каждые 60 дней)
   - HSTS headers
3. Fail2ban:
   - SSH brute-force protection
   - API rate abuse (nginx access log)
   - Auth failure blocking (3 failed attempts → 1 hour ban)
4. Backup:
   - Скрипт: SQLite + ChromaDB + Obsidian + .env + config
   - Local: /backup/voicezettel/ (на том же SSD)
   - Cloud: Backblaze B2 (или Yandex Object Storage)
   - Remote: SSH к компьютеру жены (другой дом)
   - Cron: каждую ночь 3:00
   - Retention: 7 daily, 4 weekly, 3 monthly
5. Документация по настройке белого IP

## Файлы для создания/изменения
- `infra/nginx/voicezettel.conf` — Nginx конфигурация
- `infra/fail2ban/jail.local` — Fail2ban правила
- `infra/fail2ban/filter.d/voicezettel-auth.conf` — фильтр auth failures
- `infra/backup/backup.sh` — скрипт бэкапа
- `infra/backup/restore.sh` — скрипт восстановления
- `infra/certbot/renew-hook.sh` — хук обновления сертификата
- `infra/setup.sh` — скрипт установки всей инфраструктуры
- `docs/infrastructure-setup.md` — документация

## Acceptance Criteria
- [ ] Nginx проксирует HTTP → Next.js, WebSocket работает
- [ ] SSL: https://voicezettel.online работает (A+ на ssllabs.com)
- [ ] Fail2ban: 3 неудачных входа → бан IP на 1 час
- [ ] Rate limit: >100 req/min → 429 Too Many Requests
- [ ] Backup: SQLite + ChromaDB + Obsidian бэкапятся ежедневно
- [ ] Backup в Backblaze B2 работает (или Yandex OS)
- [ ] Restore скрипт восстанавливает данные из бэкапа
- [ ] setup.sh устанавливает всё за один запуск
- [ ] Документация покрывает все шаги настройки

## Стек
Nginx, certbot, fail2ban, rclone (для cloud backup), bash
