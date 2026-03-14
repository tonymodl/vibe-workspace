# SPEC_001: VoiceZettel 2.0 — Полная архитектурная спецификация

> **Версия:** 2.0-RC1 (полный пересмотр)
> **Дата:** 2026-03-14
> **Автор:** Perplexity Computer (PM) для Антона
> **Статус:** APPROVED — готова к разбивке на TASK файлы

---

## Обзор

VoiceZettel 2.0 — гибридный AI-ассистент с голосовым управлением, объединяющий:
- **speaker1991/voicezettel** — UI (сфера частиц Three.js, чат, настройки, админка), Google OAuth, multi-user
- **tonymodl/Voicezettel** — ChromaDB RAG, Obsidian-интеграция, Self-Heal, расширяемые сущности

Ключевые возможности 2.0:
1. **Три режима**: Ассистент / Петлица (20+ персон) / Агент — переключение свайпом шара
2. **Голосовой пайплайн**: ≈380–600 мс полный цикл, fallback-цепочки, переключение на лету
3. **Система агентов**: создание по 3 шагам, оркестрация локальных ИИ, конференции
4. **Шелестун**: автономный фоновый аналитик (социальные графы, Telegram, часы, геолокация)
5. **Умный дом**: Yandex Alice + Home Assistant + Zigbee
6. **Голосовое клонирование**: любой голос включая знаменитостей
7. **Перфокарта**: геймификация жизни с авто-трекингом
8. **Визуал**: particle morphing, lip-sync через visemes, audio-reactive анимации

---

## Аппаратная база (сервер = домашний ПК Антона)

| Компонент | Значение |
|-----------|----------|
| CPU | Intel i9-14900KF (24 ядра / 32 потока) |
| GPU | NVIDIA RTX 4090 24GB VRAM |
| RAM | 96 GB DDR5 |
| SSD | MSI M480 4TB (основной) + 500GB (система) |
| ОС | Windows/WSL2 или Linux |
| Локальные модели | Qwen3-32B, Qwen2.5-Coder-32B, Qwen3-TTS (localhost:8880) |
| Сеть | Белый IP (заказан у провайдера), домен voicezettel.online |

**ОГРАНИЧЕНИЕ VRAM**: 24GB не вместит 2×32B модели одновременно → нужна очередь задач с выгрузкой/загрузкой моделей.

---

## Архитектура верхнего уровня

[SEE FULL CONTENT BELOW — PLACEHOLDER FOR ARCHITECTURE DIAGRAM]

Полная архитектурная диаграмма включает: Next.js 15 App Router, Frontend (React 19, Three.js, Zustand), API Routes, Admin Panel, Agent Dashboard, Voice Pipeline Manager (STT/LLM/TTS Router + Voice Cloner), Memory & Context (ChromaDB, Adaptive System Prompt, Obsidian), Lapel Engine (20+ speakers), Agent System, Shelestun, Smart Home, Perfocard, Infrastructure.

---

## Стек технологий

| Компонент | Технология | Почему |
|-----------|-----------|--------|
| Фреймворк | Next.js 15 (App Router) | SSR, API routes, TypeScript |
| UI | React 19 + Tailwind CSS + shadcn/ui | Компоненты + стили |
| 3D | Three.js + GLSL шейдеры + GPUComputationRenderer | Сфера частиц, lip-sync, morphing |
| State | Zustand | Лёгкий, реактивный |
| Auth | NextAuth.js + Google OAuth | Multi-user, роли, whitelist |
| DB метаданных | SQLite (better-sqlite3) | Пользователи, настройки, логи, директивы |
| Векторная БД | ChromaDB (per-user коллекции) | RAG, память, голосовые отпечатки |
| STT (cloud) | Deepgram Nova-3 (WebSocket streaming) | 150 мс, русский, diarization |
| STT (local) | Whisper (whisper.cpp / faster-whisper) | Fallback |
| Diarization | pyannote/speaker-diarization-3.1 (GPU) | 20+ спикеров, voice embeddings |
| LLM (fast) | Groq + Llama 4 Maverick | TTFT ~500 мс, 498 tok/s |
| LLM (smart) | OpenAI GPT-4o / Gemini 2.5 Pro / DeepSeek | Сложные задачи |
| LLM (local) | Ollama (Qwen3-32B, Qwen2.5-Coder-32B) | Оффлайн, агенты |
| TTS (fast) | Cartesia Sonic 3 | TTFB 40 мс, русский, эмоции |
| TTS (expressive) | ElevenLabs Flash v2.5 | 75 мс, теги эмоций |
| TTS (standard) | Google Wavenet | Стабильный, дешёвый |
| TTS (Алиса) | Yandex SpeechKit | Голос Алисы (alena) |
| TTS (local) | Qwen3-TTS (localhost:8880) + Piper | Локальный fallback |
| Voice Clone (cloud) | ElevenLabs Instant Voice Cloning | 10 сек сэмпла, мультиязычный |
| Voice Clone (local) | XTTS v2 (Coqui) | 6 сек сэмпла, русский, <200ms на RTX 4090 |
| Voice Clone (VC) | RVC v2 | Конвертация голоса, real-time, 210x RT на 4090 |
| Lip-sync | TalkingHead (Three.js) + Oculus OVR visemes | 15 visemes → blend shapes |
| Obsidian | REST API (Obsidian Local REST API plugin) | Чтение/запись заметок |
| Smart Home | Yandex IoT API + Home Assistant + MQTT | Zigbee, камеры, датчики, AC |
| Health | Samsung Health Data SDK (Android companion app) | Galaxy Watch 8 все метрики |
| Telegram | Telegraf.js (бот) + Telegram Exporter (парсинг) | Админка + импорт переписок |
| Аналитика | Google Analytics 4 + Яндекс Метрика | Веб-аналитика |
| Бэкап | Backblaze B2 / Yandex Object Storage | Облачный бэкап |

---

## Модуль 1: Авторизация и управление пользователями

### 1.1 Google OAuth

- NextAuth.js с провайдером Google
- **НЕ через Google Cloud Console Audience** — белый список email хранится в SQLite
- Таблица `users`: id, email, name, role, avatar_url, created_at, last_login
- Middleware проверяет `session.user.email` ∈ whitelist
- Новый пользователь без whitelist → страница "Access Denied"
- Admin добавляет email через админку ИЛИ Telegram бота

### 1.2 Роли и изоляция данных

| Роль | Возможности |
|------|------------|
| admin (только Антон) | Полный доступ: все настройки, Шелестун, управление пользователями, Telegram бот |
| user | Свой ассистент, своя память, свой ChromaDB namespace, нет доступа к чужим данным |

- Каждый пользователь — изолированный ChromaDB namespace
- Шелестун — ТОЛЬКО для admin
- Агенты — у каждого пользователя свои
- Множество пользователей, но ТОЛЬКО доверенные близкие люди

---

## Модуль 2: UI и фронтенд

### 2.1 Сфера частиц (Three.js)

**Источник**: speaker1991/voicezettel → Three.js particle sphere.

**Оптимизация загрузки** (текущая проблема — медленная):
1. Lazy WebGL init — показывать CSS-placeholder (SVG gradient), грузить Three.js после первого взаимодействия
2. Web Worker для расчёта позиций частиц
3. OffscreenCanvas для рендеринга в Worker (если поддерживается)
4. Progressive particles: 500 → 1000 → 3000 (за ~300ms ступеньками)
5. Кеширование скомпилированных шейдеров (ShaderMaterial cache key)
6. `requestIdleCallback` для инициализации неприоритетных эффектов

**Три визуальных режима с уникальными анимациями:**

| Режим | Анимация | Описание |
|-------|----------|----------|
| Ассистент | Стандартная сфера | Пульсация при разговоре, idle breathing, аудио-реактивность |
| Петлица | "Ушная раковина" | Цветные потоки для разных спикеров, переплетение при одновременной речи |
| Агент | Морфинг облако→фигура | Из облака точек формируется образ агента (лицо, фигура, существо) |

**Переключение режимов**: горизонтальный свайп по шару (touch/mouse drag) с плавным морфингом 0.5с.

---

## Модуль 3: Голосовой пайплайн

### 3.1 STT (Speech-to-Text)

**Primary**: Deepgram Nova-3 (WebSocket streaming) — ~150ms, русский, diarization

**Fallback**: Whisper (faster-whisper, локально на GPU) — large-v3 или medium

**Цепочка fallback**: Deepgram → Whisper (local)

### 3.2 LLM Router

Доступные провайдеры:
- Groq + Llama 4 Maverick (fast)
- GPT-4o (smart, vision)
- Gemini 2.5 Pro (smart)
- DeepSeek V3 (smart, code)
- Ollama: Qwen3-32B (smart, local)
- Ollama: Qwen2.5-Coder-32B (code, local)

**Принцип**: пользователь свободно выставляет приоритеты. Fallback автоматический по приоритету.

### 3.3 TTS Router

| Провайдер | TTFB | Русский | Эмоции | Тип |
|-----------|------|---------|--------|-----|
| Cartesia Sonic 3 | 40ms | ✅ | ✅ | Cloud |
| ElevenLabs Flash v2.5 | 75ms | ✅ | ✅ | Cloud |
| Google Wavenet | ~300ms | ✅ | ❌ | Cloud |
| Yandex SpeechKit | ~200ms | ✅ (alena) | ✅ | Cloud |
| Qwen3-TTS | ~100ms | ✅ | ✅ | Local |
| Piper TTS | ~50ms | ✅ | ❌ | Local |

**Цепочка fallback**: Cartesia → ElevenLabs → Google → Yandex → Qwen3-TTS → Piper

### 3.4 Voice Cloning

| Метод | Где | Сэмпл | Латентность | Качество |
|-------|-----|-------|-------------|----------|
| ElevenLabs IVC | Cloud | 10 сек | 75ms | Высокое |
| XTTS v2 (Coqui) | RTX 4090 | 6 сек | <200ms | Хорошее |
| RVC v2 | RTX 4090 | 5 мин обучения | <100ms | Отличное |
| OpenVoice v2 | RTX 4090 | 30 сек | GPU-dependent | Хорошее |

### 3.5 Barge-in (перебивание)

VAD детектирует голос → INTERRUPT SIGNAL:
1. Flush audio playback buffer (мгновенно)
2. Abort текущий LLM stream (AbortController)
3. Close TTS WebSocket connection
4. Start new STT stream

Время: < 100ms от детекции до нового потока

---

## Модуль 4: Память и контекст

### 4.1 Адаптивный системный промпт

**КРИТИЧНО**: поправки пользователя сохраняются НАВСЕГДА.

Механизм:
1. Каждая фраза проходит через LLM classifier
2. Classifier определяет: это поправка/директива?
3. Если да → сохранить в `user_directives`
4. Если противоположная команда → деактивировать старую директиву
5. При сборке системного промпта: базовый промпт + все активные директивы

### 4.2 ChromaDB (per-user коллекции)

Для каждого пользователя — 6 коллекций:

| Коллекция | Хранит | Embedding model |
|-----------|--------|-----------------|
| `{user_id}_conversations` | История разговоров | text-embedding-3-small |
| `{user_id}_notes` | Заметки из Obsidian | text-embedding-3-small |
| `{user_id}_facts` | Факты о пользователе | text-embedding-3-small |
| `{user_id}_entities` | Все сущности | text-embedding-3-small |
| `{user_id}_voice_prints` | Голосовые отпечатки | pyannote (192-dim) |
| `{user_id}_strategic` | Стратегическая информация | text-embedding-3-small |

### 4.3 Контекстный RAG

Каждая фраза: STT → embed → ChromaDB top-10 → сборка промпта → LLM → сохранение пары

### 4.4 Shared Memory

- Все LLM провайдеры используют ОДНУ ChromaDB
- Переключение мозга НЕ теряет контекст

---

## Модуль 5: Режим петлицы (Lapel Mode) — 20+ персон

### 5.1 Архитектура diarization

**Гибридный подход**: Deepgram (real-time) + pyannote (идентификация).

Поток: Микрофон → Deepgram (текст + SPEAKER_N) + Audio Buffer → pyannote segmentation + embedding → ChromaDB voice_prints → идентификация персоны

**pyannote на RTX 4090:**
- 14 секунд обработки на 1 час аудио (250x real-time для batch)
- Streaming: <100ms на 500ms chunk
- Модель: `pyannote/speaker-diarization-3.1` + `pyannote/segmentation-3.0`
- Kmax=3 одновременных спикеров на 5-сек окно
- Глобальный agglomerative clustering — 20+ идентичностей

### 5.2 Идентификация персон

3 уровня идентификации:
1. **Голосовой отпечаток** — pyannote embedding → ChromaDB voice_prints → cosine > 0.7
2. **Контекстный** — "Давай, Миша, твой ход" → LLM извлекает имя
3. **Ручной** — пользователь говорит "это Миша" → сохранить embedding + имя

**Обновление задним числом**: когда имя стало известно → обновить ВСЕ предыдущие реплики Speaker N

### 5.3 Три состояния петлицы

| Состояние | Описание | Активация |
|-----------|----------|-----------|
| **Запись** | Пассивная транскрибация | По умолчанию |
| **Справочник** | Отвечает на прямое обращение | "Вика, подскажи..." |
| **Участник** | Активный участник | Вручную или по команде |

### 5.4 Подсказки на ухо (Ear Hints)

- Скрытый наушник: AirPods, Galaxy Buds, микронаушник, проводной
- **Аудио-роутинг**: TTS → наушник (канал A), STT ← микрофон (канал B)

| Контекст | Поведение |
|----------|-----------|
| Мафия | Вероятности "кто мафия", подсказки по голосованию |
| Переговоры | Факты о собеседнике, аргументы |
| Тет-а-тет | Дополнительная информация, имена, даты |
| Экзамен | Ответы, формулы, подсказки |

### 5.5 Петлица 24/7

- Рассчитана на круглосуточную работу
- **Авто-суммаризация дня**: ежедневный дайджест (с кем говорил, ключевые темы, решения)

### 5.6 Игровой режим "Мафия"

Функциональность:
1. Отдельный UI: список игроков, роли (скрытые), статус живой/мёртвый
2. Трекинг: кто голосовал за кого, кто кого подозревает
3. Вероятности "кто мафия" — на основе логики, хронологии, слов, поведения
4. Психопрофили игроков — склейка с данными из Telegram переписок
5. Хронология всей игры в реальном времени
6. Подсказки на ухо владельцу

---

## Модуль 6: Система агентов

### 6.1 Три режима приложения

Переключение свайпом по шару: **Ассистент** ↔ **Петлица** ↔ **Агент**

### 6.2 Создание агента (3 шага)

**ШАГ 1 — Экспертиза:**
1. Пользователь даёт короткий промпт: "создай агента-инвестора"
2. ИИ решает какая экспертиза нужна
3. Автоматический сбор СОТЕН источников: YouTube, научные статьи, форумы, Telegram
4. Источники → ingestion в ChromaDB (отдельная коллекция для агента)
5. **Замена NotebookLM**: собственный RAG на LangChain + ChromaDB + Gemini Flash

**ШАГ 2 — Характер:**
- VoiceZettel знает пользователя → автоматически подбирает оптимальные черты
- Бегунки/слайдеры: humor, empathy, directness, creativity, formality, patience (0-100)
- Голосовые замечания сохраняются навсегда (директивы)

**ШАГ 3 — Внешность:**
- Шар из облака точек генерирует образ агента
- Варианты: женщина, мужчина, персонаж, существо, голливудская звезда
- С ГОЛОСОМ И ВНЕШНОСТЬЮ знаменитости (клонированный голос + particle avatar)
- Стиль ВСЕГДА из частиц

### 6.3 Инструменты агента

- memory: ChromaDB (своя коллекция)
- obsidian: ObsidianAPI (чтение/запись заметок)
- webSearch: SearchAPI
- openClaw: computer-use agent
- telegramBot: отправка уведомлений
- smartHome: управление умным домом

### 6.4 Оркестратор агентов

FIFO очередь задач с VRAM-менеджером:
- Задачи с флагом needs_gpu → проверяют доступную VRAM
- Если недостаточно → выгрузить наименее используемую модель
- Scheduler: каждые 100ms проверяет очередь

### 6.5 Конференции агентов

- Несколько агентов обсуждают задачу
- Каждый агент → отдельный LLM instance с контекстом
- Итоговый ответ = синтез мнений
- Голосовой вывод: каждый агент говорит своим голосом (voice clone)

### 6.6 Внешние агенты

| Агент | Задача |
|-------|--------|
| Perplexity AI | Веб-поиск + синтез |
| Cursor | Написание кода |
| Antigravity (Computer) | Сложные задачи |
| OpenClaw | Computer-use |

---

## Модуль 7: Шелестун (фоновый аналитик)

### 7.1 Обзор

Автономный фоновый агент — ТОЛЬКО для admin.

**Источники данных:**
| Источник | Данные |
|----------|--------|
| Telegram переписки | Социальные графы, намерения, ключевые темы |
| Galaxy Watch 8 | ЧСС, HRV, сон, стресс, активность |
| Часы системы | Что делал когда |
| Геолокация | Где был, паттерны перемещений |
| Голосовые разговоры (петлица) | С кем говорил, о чём |
| Obsidian заметки | Проекты, задачи, мысли |
| Obsidian граф | Связи между концепциями |
| Перфокарта | Привычки, достижения, паттерны |

### 7.2 Функции

1. **Социальный граф**: кто кому кем приходится, сила связи, паттерны общения
2. **Анализ намерений**: "Миша хочет занять денег" (из переписки)
3. **Предиктивный инсайт**: "Послезавтра встреча с Лёшей — он обычно опаздывает"
4. **Контекст для петлицы**: кто этот человек, что о нём известно
5. **Психопрофили**: склейка данных из Telegram, голоса, поведения
6. **Стратегические рекомендации**: еженедельный отчёт
7. **Мониторинг здоровья**: аномалии ЧСС/сна → предупреждение

### 7.3 Архитектура

```
Ollama node (Qwen3-32B, фоновый процесс)
    │
    ├── Scheduled jobs (cron):
    │   ├── Каждые 30 мин: анализ новых Telegram сообщений
    │   ├── Каждый час: обновление социального графа
    │   ├── Ежедневно 3:00: генерация инсайтов + синхронизация с ChromaDB
    │   └── Еженедельно: стратегический отчёт
    │
    ├── Event-driven (мгновенно):
    │   ├── Начало разговора петлицы → контекст из социального графа
    │   └── Запрос пользователя → real-time анализ
    │
    └── Хранилище: ChromaDB `{admin_id}_strategic` + SQLite `shelestun_insights`
```

---

## Модуль 8: Умный дом

### 8.1 Стек

| Уровень | Технология |
|---------|------------|
| Верхний | Yandex IoT API (Алиса, устройства Яндекс) |
| Средний | Home Assistant (local, REST API) |
| Нижний | MQTT Broker (Mosquitto) + Zigbee2MQTT |
| Устройства | Zigbee датчики, умные розетки, освещение, AC |

### 8.2 Голосовое управление

```
Пользователь: "выключи свет в спальне"
    │
    ├── Intent recognition (LLM): entity="спальня", action="выключить", type="свет"
    ├── Device lookup: ChromaDB devices → {id: "bedroom_light", protocol: "zigbee"}
    └── Execute:
        ├── Если Яндекс-устройство → Yandex IoT API
        ├── Если Zigbee → Home Assistant REST API → Zigbee2MQTT → устройство
        └── Confirm: "Свет в спальне выключен"
```

### 8.3 Контексты умного дома

- Сценарии (routine): "Я иду спать" → выключить всё, закрыть жалюзи, 18°C
- Геолокация: "Я дома" / "Я вышел" → автоматические сценарии
- Интеграция со здоровьем: плохой сон → снизить яркость утром
- Умный будильник: постепенное включение света за 30 мин до подъёма

---

## Модуль 9: Telegram бот (админка)

### 9.1 Функции

| Команда | Действие |
|---------|----------|
| /whitelist add email@test.com | Добавить пользователя |
| /whitelist remove email@test.com | Удалить пользователя |
| /stats | Статистика использования |
| /logs | Последние логи |
| /models | Активные LLM модели |
| /budget | Трекинг токенов и стоимость |
| /restart | Рестарт сервисов |

### 9.2 Уведомления (push от сервера)

- Ошибки и исключения (> severity threshold)
- Превышение бюджета токенов
- Новый пользователь запросил доступ
- Шелестун нашёл важный инсайт
- Аномалии здоровья (из Galaxy Watch)

### 9.3 Импорт Telegram переписок

- Telegram Data Export (JSON) → парсер → ChromaDB
- Telegram Exporter скрипт для автоматизации
- Цель: обогатить Шелестун данными о социальных связях

---

## Модуль 10: Трекинг токенов и бюджет

### 10.1 Таблица SQLite

```sql
CREATE TABLE token_usage (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id TEXT NOT NULL REFERENCES users(id),
  provider TEXT NOT NULL,
  model TEXT NOT NULL,
  prompt_tokens INTEGER NOT NULL,
  completion_tokens INTEGER NOT NULL,
  cost_usd REAL NOT NULL,
  session_id TEXT,
  created_at TEXT DEFAULT (datetime('now'))
);
```

### 10.2 Дашборд

- Расход по провайдерам/моделям
- Графики за день/неделю/месяц
- Бюджет лимит: предупреждение в Telegram при превышении 80%
- Топ-10 самых дорогих запросов
- Сравнение стоимости при разных конфигурациях

---

## Модуль 11: Инфраструктура

### 11.1 Сервисы

| Сервис | Порт | Технология |
|--------|------|------------|
| Next.js App | 3000 | Node.js |
| ChromaDB | 8000 | Python |
| Ollama | 11434 | Go |
| Qwen3-TTS | 8880 | Python |
| pyannote service | 8765 | Python/WebSocket |
| Home Assistant | 8123 | Python |
| MQTT Broker | 1883 | Mosquitto |

### 11.2 Nginx + HTTPS

- Nginx reverse proxy: voicezettel.online → localhost:3000
- Let's Encrypt: автоматический SSL
- Fail2ban: защита от брут-форса
- Rate limiting: 100 req/min per IP

### 11.3 Backup

- SQLite: ежедневный snapshot → Backblaze B2
- ChromaDB: еженедельный export → Yandex Object Storage
- Obsidian: git sync (уже настроен)
- Конфиги: git repo (private)

### 11.4 Docker Compose

Все сервисы в docker-compose.yml:
- next-app, chromadb, ollama, qwen3-tts, pyannote-service, homeassistant, mosquitto
- Volumes для персистентных данных
- Health checks для каждого сервиса
- Restart policy: always

---

## Модуль 12: Перфокарта (геймификация)

### 12.1 Концепция

Перфокарта — система трекинга привычек и геймификации жизни.

**Принцип**: жизнь = RPG. Каждое действие = очки. Цель = прокачать персонажа.

### 12.2 Авто-трекинг

| Источник | Что трекается |
|----------|---------------|
| Galaxy Watch 8 | Шаги, ЧСС, сон, стресс, тренировки |
| Петлица (голос) | Разговоры, встречи, выступления |
| Шелестун | Социальные взаимодействия, продуктивность |
| Telegram бот | Ручные отметки |
| Obsidian | Заметки, задачи, проекты |

### 12.3 Google Sheets интеграция

- Основное хранилище: Google Sheets (наглядно, гибко)
- Google Sheets API v4: чтение/запись из Next.js
- Структура: лист "Привычки", лист "Очки", лист "Достижения", лист "График"
- Auto-update: раз в час или по событию

### 12.4 Уведомления

- Telegram: ежедневный итог (очки за день, streak, достижения)
- Push-notification (если Progressive Web App включён)
- Голосовой отчёт через ассистента по запросу

---

## Модуль 13: Obsidian интеграция

### 13.1 Obsidian Local REST API

**Plugin**: `coddingtonbear/obsidian-local-rest-api`

Эндпоинты:
- `GET /vault/{filename}` — читать заметку
- `PUT /vault/{filename}` — записать/обновить заметку
- `POST /vault/{filename}` — создать заметку
- `GET /search/simple/?q={query}` — поиск
- `POST /commands/execute` — выполнить команду

### 13.2 Сценарии использования

1. **Ассистент читает заметки**: "что я писал про проект X?" → поиск + RAG
2. **Ассистент создаёт заметки**: "запомни: встреча с Колей в пятницу" → новая заметка
3. **Шелестун обновляет граф**: добавляет инсайты, связи, анализ
4. **Агент-исследователь**: собирает и структурирует информацию в Obsidian
5. **Синхронизация с ChromaDB**: при изменении заметок → re-embed → обновить векторы

### 13.3 Двустороняя синхронизация

```
Obsidian изменил заметку
    │
    └── Webhook / polling (каждые 30 мин) → Next.js
            │
            └── Re-embed → ChromaDB `{user_id}_notes` обновить
```

---

## Модуль 14: Аватар агента (Particle Avatar)

### 14.1 Концепция

- Аватар агента = 3D объект из частиц (dots, particles)
- Источник: Three.js Points с GLSL шейдерами
- Морфинг между формами: сфера → лицо → персонаж → существо
- Lip-sync через visemes (15 форм рта)
- Настроение = цвет + анимация частиц

### 14.2 TalkingHead интеграция

**TalkingHead** (Open-source Three.js библиотека):
- Загружает GLTF/GLB аватар
- Генерирует viseme animations по аудио
- Blend shapes для мимики
- Eye blinking, head movement

**Адаптация под particles:**
- Вместо mesh → particles на позициях вершин
- Blend shapes = морфинг позиций частиц
- Сохраняем particles aesthetic, добавляем анимацию

### 14.3 Morph Targets

Минимальный набор morph targets для агента:
- sphere: базовая сфера (нейтральное состояние)
- face_neutral: нейтральное лицо
- face_speaking: открытый рот (говорит)
- face_smile: улыбка
- face_thinking: задумчивость
- cloud: облако частиц (переходное состояние)
- custom: кастомная форма (задаётся при создании)

---

## Модуль 15: Клонирование голоса

(Описание выше в Модуле 3.4)

Дополнительно:
- **Привязка к агенту**: клонированный голос = идентичность агента
- **Мультиязычность**: ElevenLabs IVC поддерживает 32 языка включая русский
- **Качество vs Скорость**: ElevenLabs (облако, высокое качество) vs XTTS/RVC (локально, быстро)
- **Real-time voice conversion**: RVC v2 на RTX 4090 — 210x real-time = практически без задержки

---

## Модуль 16: Интеграция Samsung Health

### 16.1 Galaxy Watch 8 данные

| Метрика | Частота | Использование |
|---------|---------|---------------|
| ЧСС | Каждые 10 мин | Мониторинг стресса |
| HRV | Ночью | Качество восстановления |
| SPO2 | По запросу | Здоровье |
| Стресс | Каждые 30 мин | Рекомендации |
| Сон | Ежедневно | Оптимизация режима |
| Шаги | Ежедневно | Активность |
| Тренировки | По событию | Трекинг |

### 16.2 Архитектура

- Samsung Health Data SDK (Android)
- Android companion app (или Tasker + HTTP)
- → REST API endpoint на сервере
- → SQLite health_metrics таблица
- → ChromaDB для трендов и аномалий

### 16.3 Использование данных

- Шелестун: аномалии ЧСС/сна → предупреждения
- Перфокарта: активность = очки
- Умный дом: плохой сон → режим "мягкого утра"
- Ассистент: "судя по данным часов, ты не выспался — хочешь сократить задачи на сегодня?"

---

## Модуль 17: Административная панель

### 17.1 Роутинг

`/admin` — только для role=admin

### 17.2 Разделы

| Раздел | Функция |
|--------|----------|
| Пользователи | CRUD, whitelist, роли, статистика |
| Модели | Активные провайдеры, API ключи, приоритеты |
| Токены | Расход, бюджет, топ запросы |
| Шелестун | Инсайты, граф, настройки |
| Логи | Real-time логи, ошибки, фильтры |
| Системные настройки | Все глобальные конфиги |
| Telegram бот | Статус, тест, лог команд |

---

## Модуль 18: Полная интеграция пайплайна

### 18.1 End-to-end поток (Ассистент режим)

```
[0ms]     Пользователь начал говорить
[10ms]    VAD детектирует голос
[20ms]    Deepgram WebSocket: начало стриминга
[170ms]   Deepgram: первые слова (partial results)
[300ms]   Deepgram: финальный transcript
[320ms]   ChromaDB RAG: top-10 контекст
[340ms]   Сборка промпта (system + directives + RAG + history)
[360ms]   Groq API call (streaming)
[500ms]   Первый токен от Groq (TTFT)
[520ms]   Cartesia TTS: первый аудио чанк (TTFB 40ms от начала TTS)
[560ms]   Пользователь слышит первый звук
[600ms]   Lip-sync: viseme анимация начинается

Итого: ~560-600ms end-to-end
```

### 18.2 Параллельные потоки

```
STT stream → partial results
    ├──→ LLM stream (при достаточном тексте)
    │    └──→ TTS stream (при первом токене)
    │         └──→ Audio playback + Lip-sync
    └──→ RAG поиск (параллельно с LLM)
```

### 18.3 Fallback стратегия

```
Deepgram fails?
    └──→ Whisper local (500ms дополнительно)

Groq fails?
    └──→ Next priority LLM (по настройкам)

Cartesia fails?
    └──→ ElevenLabs → Google → Qwen3-TTS → Piper

Сервер недоступен?
    └──→ PWA offline mode (базовые функции)
```

### 18.4 Мониторинг производительности

- Каждый запрос логируется в SQLite: STT latency, LLM TTFT, TTS TTFB, total
- P50/P95/P99 метрики в admin dashboard
- Alerts при деградации (Telegram бот)

---

## Задачи разработки (TASK файлы)

| Task | Название | Зависимости | Приоритет |
|------|----------|-------------|-----------|
| TASK_001 | Project Scaffold | — | P0 |
| TASK_002 | SQLite Schema | TASK_001 | P0 |
| TASK_003 | Auth System | TASK_002 | P0 |
| TASK_004 | Particle Sphere | TASK_001 | P0 |
| TASK_005 | Voice Pipeline: STT | TASK_003 | P0 |
| TASK_006 | Voice Pipeline: LLM | TASK_005 | P0 |
| TASK_007 | Voice Pipeline: TTS | TASK_006 | P0 |
| TASK_008 | Chat Interface | TASK_007 | P0 |
| TASK_009 | Settings Panel | TASK_008 | P1 |
| TASK_010 | ChromaDB Memory | TASK_006 | P0 |
| TASK_011 | Adaptive Prompt | TASK_010 | P1 |
| TASK_012 | Lapel Diarization | TASK_005 | P1 |
| TASK_013 | Lapel Ear Hints | TASK_012 | P2 |
| TASK_014 | Admin Panel | TASK_003 | P1 |
| TASK_015 | Infrastructure | TASK_001 | P1 |
| TASK_016 | Agent System | TASK_010 | P2 |
| TASK_017 | Telegram Bot | TASK_003 | P1 |
| TASK_018 | Particle Avatar | TASK_004 | P2 |
| TASK_019 | Voice Cloning | TASK_007 | P2 |
| TASK_020 | Mafia Engine | TASK_012 | P2 |
| TASK_021 | Obsidian Integration | TASK_010 | P2 |
| TASK_022 | Shelestun | TASK_016 | P3 |
| TASK_023 | Smart Home | TASK_003 | P2 |
| TASK_024 | Token Tracking | TASK_006 | P1 |
| TASK_025 | Full Pipeline Integration | TASK_007 | P0 |

---

## Глоссарий

| Термин | Определение |
|--------|-------------|
| Barge-in | Прерывание ответа ИИ пользователем |
| ChromaDB | Векторная база данных для RAG и памяти |
| Diarization | Разделение аудио по спикерам |
| Embedding | Числовое представление текста/голоса (вектор) |
| Fallback | Резервный провайдер при ошибке основного |
| HRV | Heart Rate Variability — вариабельность ритма сердца |
| Lapel mode | Режим петлицы — транскрибация многоголосья |
| LLM | Large Language Model |
| Ollama | Локальный запуск LLM моделей |
| pyannote | Python библиотека для diarization |
| RAG | Retrieval-Augmented Generation |
| RVC | Retrieval-based Voice Conversion |
| STT | Speech-to-Text |
| TalkingHead | Open-source Three.js библиотека для lip-sync аватаров |
| Viseme | Визуальная фонема — форма рта для звука |
| VRAM | Video RAM — память GPU |
| XTTS | Cross-lingual Text-to-Speech (Coqui) |
| Перфокарта | Система трекинга привычек / геймификации жизни |