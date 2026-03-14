# TASK_022: Шелестун — автономный фоновый аналитик
**Статус:** READY_FOR_ANTIGRAVITY
**Ветка:** feature/task-022-shelestun
**Спецификация:** docs/specs/SPEC_001_voicezettel_2_0_architecture.md — Модуль 7

## Контекст
Шелестун — автономный ИИ-аналитик (только для admin). Анализирует Telegram переписки, голосовые записи, социальные сети. Строит карту социальных связей, склеивает персон из разных источников. Работает в фоне нон-стоп.

## Задача
1. Telegram Exporter Ingestion:
   - Парсинг JSON экспорта из Telegram Exporter (morf3uzzz)
   - Формат: messages → {from, text, date, media}
   - Embed → ChromaDB
   - Извлечение персон, тем, дат
2. Social Graph Builder:
   - Карта связей: персона → персона (сила связи, частота)
   - Склейка персон: Telegram ID + голосовой отпечаток + VK + Instagram
   - Визуализация графа (D3.js / Cytoscape.js)
3. Автономный процесс:
   - Фоновый worker (Node.js или Python)
   - Планировщик: какие данные анализировать, приоритеты
   - Итеративный анализ: перечитывает данные по нескольку раз
   - Ollama (Qwen3-32B) для генерации инсайтов
4. Дайджесты:
   - Еженедельные/ежемесячные отчёты
   - Аномалии: необычное поведение, новые связи
   - Сохранение: Obsidian/Стратегическое/ + ChromaDB strategic
5. API для запуска по запросу:
   - `/api/admin/shelestun/run` — запустить анализ
   - `/api/admin/shelestun/status` — статус
   - `/api/admin/shelestun/insights` — последние инсайты

## Файлы для создания/изменения
- `src/modules/shelestun/shelestun-engine.ts` — основной движок
- `src/modules/shelestun/telegram-ingestion.ts` — парсинг Telegram экспорта
- `src/modules/shelestun/social-graph.ts` — граф социальных связей
- `src/modules/shelestun/person-merger.ts` — склейка персон из разных источников
- `src/modules/shelestun/insight-generator.ts` — генерация инсайтов (Ollama)
- `src/modules/shelestun/scheduler.ts` — планировщик фоновых задач
- `src/app/api/admin/shelestun/route.ts` — API: управление Шелестуном
- `src/app/admin/shelestun/page.tsx` — UI: dashboard Шелестуна

## Acceptance Criteria
- [ ] Парсинг Telegram JSON экспорта: сообщения → ChromaDB
- [ ] Граф социальных связей строится из переписок
- [ ] Склейка персон: Telegram + голосовой отпечаток → одна персона
- [ ] Фоновый worker работает автономно
- [ ] Инсайты генерируются через Ollama (Qwen3-32B)
- [ ] Дайджесты сохраняются в Obsidian/Стратегическое/
- [ ] API: запуск, статус, инсайты
- [ ] Только admin имеет доступ
- [ ] Unit тесты: telegram parsing, person merging, graph building

## Стек
Ollama, ChromaDB, Obsidian REST API, Node.js worker, D3.js/Cytoscape.js
