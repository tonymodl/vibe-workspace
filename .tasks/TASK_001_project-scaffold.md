# TASK_001: Инициализация проекта VoiceZettel 2.0
**Статус:** READY_FOR_ANTIGRAVITY
**Ветка:** feature/task-001-project-scaffold
**Спецификация:** docs/specs/SPEC_001_voicezettel_2_0_architecture.md

## Контекст
Создаём базовый каркас проекта VoiceZettel 2.0 — Next.js 15 приложение с TypeScript.
За основу берём UI и структуру из https://github.com/speaker1991/voicezettel
Но перестраиваем архитектуру под модульность согласно SPEC_001.

## Задача
1. Создать Next.js 15 проект с App Router в папке `src/`
2. Настроить TypeScript, Tailwind CSS, ESLint, Prettier
3. Установить зависимости: next, react, react-dom, zustand, three, @react-three/fiber, @react-three/drei, lucide-react
4. Установить shadcn/ui, настроить компоненты
5. Создать структуру папок:
   - `src/app/` — маршруты
   - `src/components/` — UI компоненты
   - `src/lib/` — утилиты и провайдеры
   - `src/stores/` — Zustand stores
   - `src/hooks/` — хуки
   - `src/types/` — TypeScript типы
   - `src/providers/` — STT/LLM/TTS провайдеры
6. Создать .env.example со всеми переменными из SPEC_001
7. Создать .gitignore (исключить .env, node_modules, .next, data/)
8. Настроить `scripts` в package.json: dev, build, start, lint, test
9. Добавить docker-compose.yml для ChromaDB

## Файлы для создания/изменения
- `src/` — вся структура проекта
- `package.json`
- `tsconfig.json`
- `tailwind.config.ts`
- `next.config.ts`
- `.env.example`
- `.gitignore`
- `docker-compose.yml`
- `components.json` (shadcn)

## Acceptance Criteria
- [ ] `npm run dev` запускается без ошибок на порту 3000
- [ ] Все папки из п.5 существуют с файлами index.ts
- [ ] .env.example содержит все переменные из SPEC_001 (15+ переменных)
- [ ] Tailwind работает (видна стилизованная страница)
- [ ] shadcn/ui установлен и Button компонент рендерится
- [ ] TypeScript strict mode включён
- [ ] docker-compose.yml поднимает ChromaDB на порту 8000
- [ ] ESLint + Prettier настроены и `npm run lint` проходит

## Стек
Next.js 15, TypeScript, Tailwind CSS, shadcn/ui, Zustand, Three.js, Docker