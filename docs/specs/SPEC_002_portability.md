# SPEC_002: Портативность и переносимость сервера

> **Версия:** 1.0
> **Дата:** 2026-03-15
> **Автор:** Perplexity Computer (PM)
> **Статус:** APPROVED

---

## Обзор

VoiceZettel 2.0 должен в любой момент переноситься на другой компьютер, другой диск или другую ОС за минимальное время. Никаких хардкодов путей, IP-адресов, имён хостов. Всё через конфигурацию.

**Принцип:** `git clone` + `cp .env.example .env` + `npm install` + `docker compose up` + `npm run dev` = рабочий проект.

---

## Правила портативности (ОБЯЗАТЕЛЬНЫ для всех TASK)

### 1. Абсолютные пути ЗАПРЕЩЕНЫ

```typescript
// ❌ ЗАПРЕЩЕНО
const DB_PATH = '/home/anton/voicezettel/data/sqlite.db';
const OBSIDIAN_PATH = '/mnt/d/Obsidian/';

// ✅ ПРАВИЛЬНО
const DB_PATH = path.resolve(process.env.DATA_DIR || './data', 'sqlite.db');
const OBSIDIAN_PATH = process.env.OBSIDIAN_VAULT_PATH || './obsidian-vault';
```

### 2. Все пути через переменные окружения

```bash
# .env.example — ВСЕ пути конфигурируемые
# === Пути хранения данных ===
DATA_DIR=./data                          # Корневая директория данных
SQLITE_DIR=${DATA_DIR}/sqlite            # SQLite базы данных
CHROMADB_DIR=${DATA_DIR}/chromadb        # ChromaDB persistence
BACKUP_DIR=${DATA_DIR}/backups           # Локальные бэкапы
VOICE_SAMPLES_DIR=${DATA_DIR}/voices     # Голосовые сэмплы для клонирования
LOGS_DIR=${DATA_DIR}/logs                # Логи приложения

# === Внешние сервисы (пути) ===
OBSIDIAN_VAULT_PATH=                     # Путь к Obsidian vault (опционально)
OLLAMA_HOST=http://localhost:11434       # Ollama API endpoint

# === Домен и сеть ===
DOMAIN=localhost                         # Домен (voicezettel.online для прода)
PORT=3000                                # Порт приложения
PUBLIC_URL=http://localhost:3000         # Полный публичный URL
```

### 3. Docker Compose для всех сервисов

```yaml
# docker-compose.yml
version: '3.8'

services:
  # === ChromaDB (векторная БД) ===
  chromadb:
    image: chromadb/chroma:latest
    ports:
      - "${CHROMADB_PORT:-8000}:8000"
    volumes:
      - ${CHROMADB_DIR:-./data/chromadb}:/chroma/chroma
    environment:
      - IS_PERSISTENT=TRUE
      - ANONYMIZED_TELEMETRY=FALSE
    restart: unless-stopped

  # === Ollama (локальные LLM) ===
  ollama:
    image: ollama/ollama:latest
    ports:
      - "${OLLAMA_PORT:-11434}:11434"
    volumes:
      - ${OLLAMA_MODELS_DIR:-./data/ollama}:/root/.ollama
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    restart: unless-stopped

  # === pyannote (speaker diarization) ===
  pyannote:
    build:
      context: ./docker/pyannote
      dockerfile: Dockerfile
    ports:
      - "${PYANNOTE_PORT:-8001}:8001"
    volumes:
      - ${PYANNOTE_MODELS_DIR:-./data/pyannote-models}:/models
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    restart: unless-stopped

# === Volumes (именованные для переносимости) ===
volumes:
  chromadb_data:
  ollama_models:
  pyannote_models:
```

### 4. Структура данных (DATA_DIR)

```
data/                           # DATA_DIR — ВСЯ персистентная информация
├── sqlite/                     # Все SQLite базы
│   ├── main.db                 # users, settings, logs, directives
│   └── main.db-wal             # Write-Ahead Log
├── chromadb/                   # ChromaDB persistence (через Docker volume)
├── ollama/                     # Ollama модели (через Docker volume)
├── pyannote-models/            # pyannote кеш моделей
├── voices/                     # Голосовые сэмплы (voice cloning)
│   ├── profiles/               # Сохранённые голосовые профили
│   └── samples/                # Исходные аудиосэмплы
├── telegram-exports/           # Импортированные Telegram чаты (JSON)
├── backups/                    # Локальные бэкапы
│   ├── daily/
│   ├── weekly/
│   └── monthly/
└── logs/                       # Логи приложения
    ├── app.log
    ├── voice-pipeline.log
    └── agents.log
```

**ВАЖНО:** директория `data/` ЦЕЛИКОМ в `.gitignore`. При переносе — копировать всю папку.

### 5. Конфигурация без хардкодов

```typescript
// src/lib/config.ts — ЕДИНСТВЕННОЕ место для получения конфигурации
import path from 'path';

// Все пути резолвятся относительно PROJECT_ROOT
const PROJECT_ROOT = process.env.PROJECT_ROOT || process.cwd();

export const config = {
  // Пути данных
  dataDir: path.resolve(PROJECT_ROOT, process.env.DATA_DIR || './data'),
  sqliteDir: path.resolve(PROJECT_ROOT, process.env.SQLITE_DIR || './data/sqlite'),
  logsDir: path.resolve(PROJECT_ROOT, process.env.LOGS_DIR || './data/logs'),
  voicesDir: path.resolve(PROJECT_ROOT, process.env.VOICE_SAMPLES_DIR || './data/voices'),
  
  // Внешние сервисы
  chromadbUrl: process.env.CHROMADB_URL || 'http://localhost:8000',
  ollamaHost: process.env.OLLAMA_HOST || 'http://localhost:11434',
  pyannoteUrl: process.env.PYANNOTE_URL || 'http://localhost:8001',
  
  // Домен и сеть
  domain: process.env.DOMAIN || 'localhost',
  port: parseInt(process.env.PORT || '3000', 10),
  publicUrl: process.env.PUBLIC_URL || 'http://localhost:3000',
  
  // Obsidian (опционально)
  obsidianVaultPath: process.env.OBSIDIAN_VAULT_PATH || null,
  obsidianApiUrl: process.env.OBSIDIAN_API_URL || 'http://localhost:27123',
} as const;

// Валидация при старте
export function validateConfig(): string[] {
  const errors: string[] = [];
  const { dataDir, sqliteDir } = config;
  // Проверить что директории существуют или могут быть созданы
  // Проверить подключение к ChromaDB, Ollama
  return errors;
}
```

### 6. Миграция базы данных

```typescript
// src/lib/db/migrations.ts
// Все миграции версионированные и идемпотентные
interface Migration {
  version: number;
  name: string;
  up: (db: Database) => void;
  down: (db: Database) => void;
}

// При первом запуске на новом компьютере:
// 1. Создаётся data/sqlite/main.db
// 2. Применяются ВСЕ миграции последовательно
// 3. Создаётся admin пользователь (из .env ADMIN_EMAIL)
```

### 7. Скрипт первого запуска

```bash
#!/bin/bash
# scripts/setup.sh — запускается ОДИН раз на новом компьютере

set -e

echo "=== VoiceZettel 2.0 Setup ==="

# 1. Проверить зависимости
command -v node >/dev/null 2>&1 || { echo "Нужен Node.js 20+"; exit 1; }
command -v docker >/dev/null 2>&1 || { echo "Нужен Docker"; exit 1; }
command -v nvidia-smi >/dev/null 2>&1 && echo "GPU: $(nvidia-smi --query-gpu=name --format=csv,noheader)" || echo "⚠️  GPU не обнаружен (локальные модели будут медленными)"

# 2. Создать .env из шаблона
if [ ! -f .env ]; then
  cp .env.example .env
  echo "✅ Создан .env из шаблона. ЗАПОЛНИ API КЛЮЧИ!"
fi

# 3. Создать директории данных
mkdir -p data/{sqlite,chromadb,ollama,pyannote-models,voices/{profiles,samples},telegram-exports,backups/{daily,weekly,monthly},logs}
echo "✅ Структура data/ создана"

# 4. Установить зависимости
npm install
echo "✅ npm зависимости установлены"

# 5. Запустить Docker сервисы
docker compose up -d chromadb
echo "✅ ChromaDB запущен"

# 6. Применить миграции БД
npm run db:migrate
echo "✅ База данных инициализирована"

# 7. Инициализировать shadcn/ui
npx shadcn@latest init -y 2>/dev/null || true
echo "✅ shadcn/ui инициализирован"

echo ""
echo "=== Готово! ==="
echo "1. Заполни API ключи в .env"
echo "2. Запусти: npm run dev"
echo "3. Открой: http://localhost:3000"
```

### 8. Скрипт миграции сервера

```bash
#!/bin/bash
# scripts/migrate-server.sh — перенос на другой компьютер

set -e

echo "=== Миграция VoiceZettel 2.0 ==="
echo "На СТАРОМ сервере:"
echo "  1. tar -czf voicezettel-data.tar.gz data/"
echo "  2. Скопировать voicezettel-data.tar.gz + .env на новый сервер"
echo ""
echo "На НОВОМ сервере:"
echo "  1. git clone https://github.com/tonymodl/vibe-workspace.git"
echo "  2. cd vibe-workspace"
echo "  3. tar -xzf voicezettel-data.tar.gz"
echo "  4. cp путь/к/.env .env"
echo "  5. Обнови DOMAIN и PUBLIC_URL в .env если домен изменился"
echo "  6. bash scripts/setup.sh"
echo "  7. npm run dev"
echo ""
echo "Время миграции: ~10 минут (без учёта копирования data/)"
```

---

## .gitignore (обязательные записи)

```gitignore
# === Данные (НИКОГДА не в git) ===
data/
*.db
*.db-wal
*.db-shm

# === Секреты ===
.env
.env.local
.env.production

# === Node ===
node_modules/
.next/
dist/

# === OS ===
.DS_Store
Thumbs.db

# === IDE ===
.vscode/
.idea/
```

---

## Правила для Antigravity (добавить в AGENT.md)

1. **НИКОГДА** не хардкодить абсолютные пути — только `process.env.X` или `config.x`
2. **ВСЕ** пути к данным через `src/lib/config.ts`
3. **ВСЕ** сервисы (ChromaDB, Ollama, pyannote) через URL из `.env`
4. **data/** директория = единственное место персистентных данных
5. **docker-compose.yml** для всех внешних сервисов
6. При создании нового файла, который использует путь — импортировать из `config.ts`
7. Тесты должны работать с `DATA_DIR=./test-data` (изолированные данные)
8. `scripts/setup.sh` должен обновляться при добавлении новых зависимостей

---

## Чек-лист при проверке PR

- [ ] Нет абсолютных путей (grep для `/home/`, `/mnt/`, `/Users/`, `C:\\`)
- [ ] Новые переменные добавлены в `.env.example`
- [ ] Новые Docker сервисы добавлены в `docker-compose.yml`
- [ ] Новые data-директории создаются в `scripts/setup.sh`
- [ ] `src/lib/config.ts` обновлён если нужны новые пути
