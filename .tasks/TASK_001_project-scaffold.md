# TASK_001: Скаффолд проекта Next.js 15 + портативная архитектура
**Статус:** READY_FOR_ANTIGRAVITY
**Ветка:** feature/task-001-project-scaffold
**Спецификации:**
- `docs/specs/SPEC_001_voicezettel_2_0_architecture.md` — главная архитектура
- `docs/specs/SPEC_002_portability.md` — портативность и переносимость

## Контекст
Создание базового скаффолда VoiceZettel 2.0. Это фундамент для ВСЕХ 24 последующих задач. Проект ОБЯЗАН быть портативным — переносимым на другой компьютер/диск командой `git clone` + `cp .env` + `npm install` + `docker compose up`.

**КРИТИЧНО**: Прочитай SPEC_002 перед началом работы. Портативность — не опция, а фундаментальное требование.

## Задача

### Шаг 1: Инициализация Next.js 15
1. `npx create-next-app@latest . --typescript --tailwind --eslint --app --src-dir --import-alias "@/*"`
2. Обновить `tsconfig.json`: strict mode, paths aliases
3. Обновить `next.config.ts`: output standalone (для Docker деплоя)

### Шаг 2: Структура директорий
```
src/
├── app/                        # Next.js App Router
│   ├── layout.tsx              # Корневой layout
│   ├── page.tsx                # Главная (placeholder)
│   ├── api/                    # API routes
│   │   └── health/route.ts     # GET /api/health — статус сервисов
│   └── admin/                  # Админ pages
│       └── page.tsx            # Placeholder
├── components/                 # React компоненты
│   ├── ui/                     # shadcn/ui
│   ├── chat/                   # Чат (TASK_008)
│   ├── particle-system/        # Three.js (TASK_004)
│   └── settings/               # Настройки (TASK_009)
├── lib/                        # Утилиты
│   ├── config.ts               # ⭐ ЕДИНСТВЕННЫЙ источник конфигурации
│   ├── db.ts                   # SQLite подключение (TASK_002)
│   ├── auth.ts                 # NextAuth (TASK_003)
│   ├── logger.ts               # Логгер (pino)
│   └── voice-pipeline/         # Голосовой пайплайн (TASK_005-007)
├── modules/                    # Бизнес-модули
│   ├── agents/                 # TASK_016
│   ├── lapel/                  # TASK_012-013
│   ├── mafia/                  # TASK_020
│   ├── shelestun/              # TASK_022
│   ├── smart-home/             # TASK_023
│   └── perfocard/              # Google Sheets
├── stores/                     # Zustand stores
│   └── app-store.ts            # Главный store (mode, settings)
└── types/                      # TypeScript типы
    ├── index.ts                # Общие типы
    └── voice-pipeline.ts       # Типы пайплайна
```

### Шаг 3: config.ts (САМЫЙ ВАЖНЫЙ ФАЙЛ)
Создать `src/lib/config.ts` по спецификации SPEC_002. Все пути через `process.env`. Валидация при старте. Ни один другой файл в проекте НЕ ДОЛЖЕН обращаться к `process.env` напрямую — только через config.

### Шаг 4: Docker Compose
```yaml
# docker-compose.yml — ChromaDB обязателен, Ollama опционален
services:
  chromadb:
    image: chromadb/chroma:latest
    ports: ["${CHROMADB_PORT:-8000}:8000"]
    volumes: ["${CHROMADB_DIR:-./data/chromadb}:/chroma/chroma"]
    environment:
      IS_PERSISTENT: "TRUE"
      ANONYMIZED_TELEMETRY: "FALSE"
    restart: unless-stopped
```

### Шаг 5: .env.example (ПОЛНЫЙ)
Создать `.env.example` со ВСЕМИ переменными из SPEC_001 + SPEC_002. Группировать по секциям с комментариями. Включить:
- Пути данных (DATA_DIR, SQLITE_DIR, etc.)
- API ключи (DEEPGRAM, OPENAI, GROQ, ELEVENLABS, CARTESIA, GOOGLE, YANDEX, DEEPSEEK)
- OAuth (GOOGLE_CLIENT_ID/SECRET, NEXTAUTH_SECRET)
- Сервисы (CHROMADB_URL, OLLAMA_HOST, PYANNOTE_URL)
- Домен (DOMAIN, PORT, PUBLIC_URL)
- Telegram (TELEGRAM_BOT_TOKEN, TELEGRAM_ADMIN_CHAT_ID)
- Admin (ADMIN_EMAIL=evsinanton@gmail.com)

### Шаг 6: Скрипты
- `scripts/setup.sh` — первый запуск (см. SPEC_002)
- `scripts/migrate-server.sh` — инструкция переноса
- `scripts/dev.sh` — `docker compose up -d && npm run dev`

### Шаг 7: .gitignore
Строгий `.gitignore` по SPEC_002 — data/, .env, node_modules/, .next/

### Шаг 8: Зависимости (из SPEC_001)
```json
{
  "dependencies": {
    "next": "^15.0.0",
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "zustand": "^5.0.0",
    "pino": "^9.0.0",
    "pino-pretty": "^11.0.0"
  },
  "devDependencies": {
    "typescript": "^5.6.0",
    "@types/node": "^22.0.0",
    "@types/react": "^19.0.0",
    "eslint": "^9.0.0",
    "prettier": "^3.4.0"
  }
}
```

**НЕ устанавливать** three.js, chromadb, better-sqlite3, next-auth и другие тяжёлые зависимости — они будут добавлены в соответствующих TASK.

### Шаг 9: API Health Check
`src/app/api/health/route.ts`:
```typescript
// GET /api/health
// Возвращает статус всех сервисов: next, chromadb, ollama, sqlite
// Для мониторинга и Telegram бота (TASK_017)
```

### Шаг 10: Zustand Store (базовый)
```typescript
// src/stores/app-store.ts
interface AppState {
  mode: 'assistant' | 'lapel' | 'agent';
  isRecording: boolean;
  // Остальные поля добавятся в следующих TASK
}
```

## Файлы для создания
- `package.json` — зависимости
- `tsconfig.json` — strict TypeScript
- `next.config.ts` — output standalone
- `tailwind.config.ts` — Tailwind
- `docker-compose.yml` — ChromaDB (+ Ollama опционально)
- `.env.example` — ВСЕ переменные
- `.gitignore` — строгий, по SPEC_002
- `src/lib/config.ts` — единственный источник конфигурации
- `src/lib/logger.ts` — pino логгер
- `src/app/layout.tsx` — корневой layout
- `src/app/page.tsx` — placeholder
- `src/app/api/health/route.ts` — health check
- `src/stores/app-store.ts` — базовый Zustand store
- `src/types/index.ts` — общие типы
- `scripts/setup.sh` — первый запуск
- `scripts/migrate-server.sh` — перенос сервера
- `scripts/dev.sh` — dev запуск

## Acceptance Criteria
- [ ] `npm run dev` запускается без ошибок на ЧИСТОЙ машине
- [ ] `npm run build` проходит без ошибок
- [ ] `npm run lint` проходит без ошибок
- [ ] TypeScript strict mode включён
- [ ] `src/lib/config.ts` — единственное место чтения `process.env`
- [ ] `.env.example` содержит ВСЕ переменные (30+ штук)
- [ ] `docker-compose.yml` запускает ChromaDB (`docker compose up -d chromadb` работает)
- [ ] `GET /api/health` возвращает JSON со статусом сервисов
- [ ] `data/` директория в `.gitignore`
- [ ] `scripts/setup.sh` создаёт все нужные директории
- [ ] Нет ни одного абсолютного пути в коде (grep -r "/home\|/mnt\|/Users\|C:\\\\" src/ возвращает пусто)
- [ ] Zustand store инициализирован
- [ ] Placeholder страницы отображаются (`/` и `/admin`)

## Стек
Next.js 15, React 19, TypeScript 5.6+, Tailwind CSS 4, shadcn/ui, Zustand 5, pino, Docker Compose, ESLint 9, Prettier
