# SPEC_002: Развёртывание проекта через Antigravity

> **Версия:** 2.0
> **Дата:** 2026-03-15
> **Автор:** Perplexity Computer (PM)
> **Статус:** APPROVED

---

## Обзор

VoiceZettel 2.0 должен разворачиваться на новом компьютере **полностью автономно агентом Antigravity**. Пользователь не запускает никаких команд вручную — Antigravity сам клонирует репо, устанавливает зависимости, поднимает сервисы, настраивает окружение и запускает проект.

**Принцип:** Пользователь открывает Antigravity на новом ПК, даёт команду «разверни VoiceZettel» → агент делает ВСЁ сам.

---

## Сценарий развёртывания

### Что делает Antigravity автономно:

1. **Клонирует репозиторий**
   ```bash
   git clone https://github.com/tonymodl/vibe-workspace.git
   cd vibe-workspace
   ```

2. **Проверяет системные требования**
   - Node.js 20+ установлен? Если нет — устанавливает через nvm
   - Docker установлен и запущен? Если нет — устанавливает
   - nvidia-smi доступен? Проверяет VRAM (нужно 24GB для полных моделей)
   - Достаточно места на диске (минимум 50GB для моделей)

3. **Создаёт и заполняет .env**
   - Копирует `.env.example` → `.env`
   - Спрашивает у пользователя API ключи (Deepgram, OpenAI, Groq, ElevenLabs, Cartesia, Google OAuth, Yandex, DeepSeek)
   - Устанавливает ADMIN_EMAIL, DOMAIN, PUBLIC_URL
   - Генерирует NEXTAUTH_SECRET автоматически

4. **Устанавливает зависимости**
   ```bash
   npm install
   ```

5. **Создаёт структуру данных**
   ```bash
   mkdir -p data/{sqlite,chromadb,ollama,pyannote-models,voices/{profiles,samples},telegram-exports,backups/{daily,weekly,monthly},logs}
   ```

6. **Поднимает Docker сервисы**
   ```bash
   docker compose up -d
   ```
   Сервисы: ChromaDB, Ollama (если GPU), pyannote (если GPU)

7. **Скачивает модели Ollama**
   ```bash
   ollama pull qwen3:32b
   ollama pull qwen2.5-coder:32b
   ```

8. **Применяет миграции БД**
   ```bash
   npm run db:migrate
   ```

9. **Проверяет работоспособность**
   ```bash
   npm run build        # Сборка проходит
   npm run dev          # Dev сервер стартует
   curl localhost:3000/api/health  # Health check OK
   ```

10. **Настраивает сеть (если нужен внешний доступ)**
    - nginx reverse proxy
    - Let's Encrypt SSL для voicezettel.online
    - Проверка что домен резолвится

### Миграция с существующего сервера

Если проект уже работал на другом ПК:

1. Antigravity на старом ПК: `tar -czf voicezettel-data.tar.gz data/` + `.env`
2. Переносит архив на новый ПК (пользователь указывает путь или Antigravity скачивает)
3. Antigravity на новом ПК: распаковывает `data/`, копирует `.env`
4. Далее стандартный процесс развёртывания (шаги 1-9)

---

## Скрипт bootstrap.sh (для Antigravity)

Antigravity может использовать этот скрипт как основу, но НЕ обязан — он может выполнять команды напрямую.

```bash
#!/bin/bash
# scripts/bootstrap.sh — автоматическое развёртывание
# Antigravity запускает этот скрипт или выполняет шаги вручную

set -e

echo "=== VoiceZettel 2.0 Bootstrap ==="

# 1. Проверить зависимости
command -v node >/dev/null 2>&1 || { echo "❌ Node.js не найден"; exit 1; }
command -v docker >/dev/null 2>&1 || { echo "❌ Docker не найден"; exit 1; }
command -v nvidia-smi >/dev/null 2>&1 && echo "✅ GPU: $(nvidia-smi --query-gpu=name --format=csv,noheader)" || echo "⚠️ GPU не обнаружен"

# 2. .env
if [ ! -f .env ]; then
  cp .env.example .env
  echo "⚠️ Создан .env — НУЖНО ЗАПОЛНИТЬ API КЛЮЧИ"
fi

# 3. Структура данных
mkdir -p data/{sqlite,chromadb,ollama,pyannote-models,voices/{profiles,samples},telegram-exports,backups/{daily,weekly,monthly},logs}
echo "✅ Директории данных созданы"

# 4. Зависимости
npm install
echo "✅ npm зависимости установлены"

# 5. Docker сервисы
docker compose up -d
echo "✅ Docker сервисы запущены"

# 6. Миграции
npm run db:migrate 2>/dev/null || echo "⚠️ Миграции пока не настроены (TASK_002)"

# 7. Сборка
npm run build
echo "✅ Сборка успешна"

echo ""
echo "=== Готово! ==="
echo "Запуск: npm run dev"
echo "Адрес: http://localhost:3000"
```

---

## Требования к коду для поддержки развёртывания

1. **`.env.example`** — всегда актуальный, содержит ВСЕ переменные с комментариями
2. **`docker-compose.yml`** — все внешние сервисы (ChromaDB, Ollama, pyannote)
3. **`scripts/bootstrap.sh`** — скрипт развёртывания
4. **`package.json` scripts** — db:migrate, dev, build, lint
5. **`/api/health`** — endpoint для проверки что всё работает

---

## Что НЕ входит в эту спецификацию

- Правила портативности кода (хардкоды путей и т.д.) — это часть общих code standards в AGENT.md
- CI/CD — проект деплоится на домашний ПК, не в облако
- Мониторинг — будет в TASK_017 (Telegram бот)
