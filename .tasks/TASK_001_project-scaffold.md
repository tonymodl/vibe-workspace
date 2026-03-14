# TASK_001: Скаффолд проекта
**Статус:** READY_FOR_ANTIGRAVITY
**Ветка:** feature/task-001-project-scaffold

## [CONTEXT_FILES]
- `.agent/AGENT.md`
- `docs/specs/SPEC_003_data_models.md` (секция TypeScript интерфейсы)

НЕ ЧИТАЙ SPEC_001. НЕ ЧИТАЙ другие TASK файлы.

## [STEPS]

### Шаг 1: Создать ветку
```bash
git checkout -b feature/task-001-project-scaffold
```

### Шаг 2: Инициализировать Next.js
Создай проект во временной папке и скопируй (в текущей папке уже есть файлы):
```bash
npx create-next-app@latest temp-init --typescript --tailwind --eslint --app --src-dir --import-alias "@/*" --yes
cp -r temp-init/* .
cp temp-init/.eslintrc.json . 2>/dev/null; cp temp-init/.gitignore . 2>/dev/null
rm -rf temp-init
```

### Шаг 3: Установить зависимости
```bash
npm install zustand pino pino-pretty
```
Коммит: `feat(task-001): init next.js + зависимости`

### Шаг 4: Создать `src/types/index.ts`
Скопируй ВСЕ типы из SPEC_003 (секция TypeScript интерфейсы). Ничего не меняй, не добавляй.
Коммит: `feat(task-001): типы данных из SPEC_003`

### Шаг 5: Создать `src/lib/config.ts`
```typescript
import path from 'path';

const PROJECT_ROOT = process.cwd();

export const config = {
  dataDir: path.resolve(PROJECT_ROOT, process.env.DATA_DIR || './data'),
  sqliteDir: path.resolve(PROJECT_ROOT, process.env.SQLITE_DIR || './data/sqlite'),
  logsDir: path.resolve(PROJECT_ROOT, process.env.LOGS_DIR || './data/logs'),
  voicesDir: path.resolve(PROJECT_ROOT, process.env.VOICE_SAMPLES_DIR || './data/voices'),
  chromadbUrl: process.env.CHROMADB_URL || 'http://localhost:8000',
  ollamaHost: process.env.OLLAMA_HOST || 'http://localhost:11434',
  pyannoteUrl: process.env.PYANNOTE_URL || 'http://localhost:8001',
  domain: process.env.DOMAIN || 'localhost',
  port: parseInt(process.env.PORT || '3000', 10),
  publicUrl: process.env.PUBLIC_URL || 'http://localhost:3000',
  useMocks: process.env.USE_MOCKS === 'true',
  adminEmail: process.env.ADMIN_EMAIL || '',
} as const;
```
Коммит: `feat(task-001): config.ts`

### Шаг 6: Создать `src/lib/logger.ts`
```typescript
import pino from 'pino';

// Логгер приложения
export const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  transport: process.env.NODE_ENV !== 'production'
    ? { target: 'pino-pretty', options: { colorize: true } }
    : undefined,
});
```
Коммит: `feat(task-001): logger`

### Шаг 7: Создать `src/stores/app-store.ts`
```typescript
'use client';
import { create } from 'zustand';
import type { AppMode } from '@/types';

interface AppState {
  // Текущий режим приложения
  mode: AppMode;
  setMode: (mode: AppMode) => void;
  // Статус записи голоса
  isRecording: boolean;
  setIsRecording: (val: boolean) => void;
}

export const useAppStore = create<AppState>((set) => ({
  mode: 'assistant',
  setMode: (mode) => set({ mode }),
  isRecording: false,
  setIsRecording: (isRecording) => set({ isRecording }),
}));
```
Коммит: `feat(task-001): zustand store`

### Шаг 8: Создать `src/app/api/health/route.ts`
```typescript
import { NextResponse } from 'next/server';
import type { HealthStatus } from '@/types';

// GET /api/health — статус сервисов
export async function GET() {
  const status: HealthStatus = {
    status: 'ok',
    services: {
      next: true,
      sqlite: false,  // Будет в TASK_002
      chromadb: false, // Будет в TASK_010
      ollama: false,   // Будет позже
    },
    timestamp: new Date().toISOString(),
  };
  return NextResponse.json(status);
}
```
Коммит: `feat(task-001): health endpoint`

### Шаг 9: Создать `src/app/admin/page.tsx`
```typescript
export default function AdminPage() {
  return (
    <div className="flex min-h-screen items-center justify-center">
      <h1 className="text-2xl font-bold">Админ-панель VoiceZettel 2.0</h1>
      <p className="text-gray-500 mt-2">Будет реализована в TASK_014</p>
    </div>
  );
}
```
Коммит: `feat(task-001): admin placeholder`

### Шаг 10: Обновить `src/app/page.tsx`
Замени содержимое дефолтной страницы Next.js на:
```typescript
export default function HomePage() {
  return (
    <div className="flex min-h-screen items-center justify-center bg-black text-white">
      <div className="text-center">
        <h1 className="text-4xl font-bold">VoiceZettel 2.0</h1>
        <p className="text-gray-400 mt-4">Голосовой AI-ассистент</p>
      </div>
    </div>
  );
}
```
Коммит: `feat(task-001): главная страница`

### Шаг 11: Создать `.env.example`
```env
# === Режим ===
NODE_ENV=development
USE_MOCKS=true

# === Пути данных ===
DATA_DIR=./data
SQLITE_DIR=./data/sqlite
LOGS_DIR=./data/logs
VOICE_SAMPLES_DIR=./data/voices

# === Сервисы (URL) ===
CHROMADB_URL=http://localhost:8000
OLLAMA_HOST=http://localhost:11434
PYANNOTE_URL=http://localhost:8001

# === Домен и сеть ===
DOMAIN=localhost
PORT=3000
PUBLIC_URL=http://localhost:3000

# === Auth ===
NEXTAUTH_SECRET=сгенерируй-случайную-строку
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
ADMIN_EMAIL=evsinanton@gmail.com

# === STT провайдеры ===
DEEPGRAM_API_KEY=

# === LLM провайдеры ===
GROQ_API_KEY=
OPENAI_API_KEY=
DEEPSEEK_API_KEY=
GOOGLE_AI_API_KEY=

# === TTS провайдеры ===
ELEVENLABS_API_KEY=
CARTESIA_API_KEY=
YANDEX_API_KEY=

# === Telegram ===
TELEGRAM_BOT_TOKEN=
TELEGRAM_ADMIN_CHAT_ID=

# === Логирование ===
LOG_LEVEL=info
```
Коммит: `feat(task-001): .env.example`

### Шаг 12: Создать `docker-compose.yml`
```yaml
services:
  chromadb:
    image: chromadb/chroma:latest
    ports:
      - "${CHROMADB_PORT:-8000}:8000"
    volumes:
      - ./data/chromadb:/chroma/chroma
    environment:
      IS_PERSISTENT: "TRUE"
      ANONYMIZED_TELEMETRY: "FALSE"
    restart: unless-stopped
```
Коммит: `feat(task-001): docker-compose`

### Шаг 13: Создать скрипты
`scripts/bootstrap.sh`:
```bash
#!/bin/bash
set -e
echo "=== VoiceZettel 2.0 Bootstrap ==="
command -v node >/dev/null 2>&1 || { echo "Нужен Node.js 20+"; exit 1; }
if [ ! -f .env ]; then cp .env.example .env; echo "Создан .env — заполни ключи!"; fi
mkdir -p data/{sqlite,chromadb,voices/{profiles,samples},logs}
npm install
echo "Готово! Запуск: npm run dev"
```

`scripts/dev.sh`:
```bash
#!/bin/bash
docker compose up -d 2>/dev/null || true
npm run dev
```
Коммит: `feat(task-001): скрипты bootstrap и dev`

### Шаг 14: Обновить `.gitignore`
Добавь в .gitignore:
```
data/
*.db
*.db-wal
*.db-shm
.env
.env.local
.env.production
```
Коммит: `feat(task-001): обновлён .gitignore`

### Шаг 15: Обновить `tsconfig.json`
Убедись что `strict: true` включён. Если нет — включи.

### Шаг 16: Создать структуру пустых директорий
```bash
mkdir -p src/components/{ui,chat,particle-system,settings}
mkdir -p src/modules/{agents,lapel,mafia,shelestun,smart-home,perfocard}
mkdir -p src/lib/voice-pipeline
mkdir -p src/lib/db
touch src/types/voice-pipeline.ts
```
Коммит: `feat(task-001): структура директорий`

## [VERIFICATION]
Выполни эти команды. ВСЕ должны пройти с exit code 0:
```bash
npm run build
npm run lint
npx tsc --noEmit
curl -s http://localhost:3000/api/health 2>/dev/null || echo "OK: dev сервер не запущен, это нормально"
```
Если build или lint падают — чини и повторяй.

После успешной верификации:
1. Обнови статус этого файла на `DONE_BY_ANTIGRAVITY`
2. `git push origin feature/task-001-project-scaffold`
3. Создай Pull Request в main

## Стек
Next.js 15, React 19, TypeScript 5.6+, Tailwind CSS, Zustand 5, pino
