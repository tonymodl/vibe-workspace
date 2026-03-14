# TASK_001: Скаффолд проекта Next.js 15
**Статус:** READY_FOR_ANTIGRAVITY
**Ветка:** feature/task-001-project-scaffold
**Спецификация:** docs/specs/SPEC_001_voicezettel_2_0_architecture.md

## Контекст
Создание базового скаффолда проекта VoiceZettel 2.0. Это фундамент для всех последующих задач. Проект строится на Next.js 15 (App Router), TypeScript, Tailwind CSS, shadcn/ui.

## Задача
1. Инициализировать Next.js 15 с App Router и TypeScript
2. Настроить Tailwind CSS 4 + shadcn/ui
3. Создать структуру директорий:
   ```
   src/
   ├── app/            # Next.js App Router pages
   │   ├── layout.tsx
   │   ├── page.tsx
   │   ├── api/        # API routes
   │   └── admin/      # Админ-панель pages
   ├── components/     # React компоненты
   │   ├── ui/         # shadcn/ui компоненты
   │   ├── chat/       # Чат-интерфейс
   │   ├── particle-system/ # Three.js сфера частиц
   │   └── settings/   # Панель настроек
   ├── lib/            # Утилиты и конфигурация
   │   ├── db.ts       # SQLite подключение
   │   ├── auth.ts     # NextAuth конфигурация
   │   └── voice-pipeline/ # Голосовой пайплайн
   ├── modules/        # Бизнес-логика модулей
   │   ├── agents/
   │   ├── lapel/
   │   ├── shelestun/
   │   ├── smart-home/
   │   └── perfocard/
   ├── stores/         # Zustand stores
   └── types/          # TypeScript типы
   ```
4. Установить базовые зависимости (см. SPEC_001 раздел "Зависимости")
5. Настроить ESLint + Prettier
6. Создать .env.example с перечислением ВСЕХ нужных переменных
7. Создать docker-compose.yml для ChromaDB + Ollama (опционально)

## Файлы для создания/изменения
- `package.json` — зависимости
- `tsconfig.json` — TypeScript конфигурация
- `tailwind.config.ts` — Tailwind настройки
- `next.config.ts` — Next.js конфигурация
- `src/app/layout.tsx` — корневой layout
- `src/app/page.tsx` — главная страница (placeholder)
- `.env.example` — шаблон переменных окружения
- `.eslintrc.json` — ESLint правила
- `docker-compose.yml` — ChromaDB + Ollama

## Acceptance Criteria
- [ ] `npm run dev` запускается без ошибок
- [ ] TypeScript strict mode включён
- [ ] Tailwind CSS работает (видна стилизованная страница)
- [ ] shadcn/ui установлен (Button, Input компоненты доступны)
- [ ] Структура директорий создана с placeholder файлами
- [ ] .env.example содержит ВСЕ переменные из SPEC_001
- [ ] ESLint + Prettier настроены, `npm run lint` проходит
- [ ] Zustand store создан (пустой, но инициализирован)

## Стек
Next.js 15, React 19, TypeScript 5.6+, Tailwind CSS 4, shadcn/ui, Zustand 5, ESLint 9