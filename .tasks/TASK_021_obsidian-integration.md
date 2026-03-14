# TASK_021: Obsidian интеграция + расширяемые сущности
**Статус:** READY_FOR_ANTIGRAVITY
**Ветка:** feature/task-021-obsidian-integration
**Спецификация:** docs/specs/SPEC_001_voicezettel_2_0_architecture.md — Модуль 13

## Контекст
Obsidian — vault для структурированных заметок. VoiceZettel автоматически создаёт заметки из голосовых разговоров. Расширяемые сущности: базовые + кастомные типы с toggle вкл/выкл.

## Задача
1. Obsidian REST API клиент:
   - Подключение к Obsidian Local REST API plugin
   - CRUD: создание/чтение/обновление/удаление заметок
   - Поиск по vault
   - Чтение структуры директорий
2. Структура vault (создать при первом подключении):
   - Zettelkasten/ (Заметки, Идеи, Факты, Персоны, Задачи)
   - Архив/ (Телеграм переписки, Старые заметки)
   - Стратегическое/ (Инсайты ИИ, Факты обо мне, Аналитика)
   - _templates/
3. Расширяемые сущности:
   - EntityType интерфейс: поля, шаблоны, правила извлечения
   - Базовые: Заметки, Идеи, Факты, Персоны, Задачи
   - Кастомные: пользователь создаёт через админку
   - Toggle вкл/выкл для каждого типа
4. Автоматическое извлечение:
   - LLM анализирует текст разговора
   - Извлекает сущности по правилам EntityType
   - Создаёт заметки в Obsidian по шаблонам
5. Синхронизация с ChromaDB:
   - Новые заметки → embed → ChromaDB notes collection

## Файлы для создания/изменения
- `src/lib/obsidian/client.ts` — REST API клиент
- `src/lib/obsidian/vault-setup.ts` — создание структуры vault
- `src/modules/entities/entity-manager.ts` — управление типами сущностей
- `src/modules/entities/entity-extractor.ts` — извлечение из текста (LLM)
- `src/modules/entities/templates/` — markdown шаблоны сущностей
- `src/app/api/entities/route.ts` — API: CRUD типов сущностей
- `src/components/settings/EntityManager.tsx` — UI: управление сущностями

## Acceptance Criteria
- [ ] Подключение к Obsidian REST API работает
- [ ] Структура vault создаётся при первом подключении
- [ ] CRUD заметок: создание, чтение, обновление, удаление
- [ ] 5 базовых типов сущностей работают с шаблонами
- [ ] Кастомные типы: создание через UI с полями и правилами
- [ ] Toggle вкл/выкл для каждого типа
- [ ] LLM извлекает сущности из текста разговора
- [ ] Новые заметки синхронизируются в ChromaDB
- [ ] Unit тесты: entity extraction, CRUD, vault setup

## Стек
Obsidian REST API, LLM Router, ChromaDB, React 19
