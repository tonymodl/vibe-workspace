# SPEC_001: VoiceZettel 2.0 — Полная архитектурная спецификация

## Обзор

VoiceZettel 2.0 — гибридный голосовой ассистент на базе Next.js 15, объединяющий лучшее из двух проектов:
- **speaker1991/voicezettel** — UI (сфера частиц, чат, настройки, админка), Google OAuth, multi-user
- **tonymodl/Voicezettel** — ChromaDB RAG, Obsidian-интеграция, Self-Heal, расширяемые сущности

Ключевое отличие 2.0: сверхбыстрый голосовой пайплайн (≈380–600 мс), адаптивная память, режим петлицы с распознаванием персон, переключение мозгов/голосов на лету, автономная работа агентов.

Ссылка на исходные проекты:
- https://github.com/speaker1991/voicezettel
- https://github.com/tonymodl/Voicezettel

---

## Архитектура

### Общая схема

```
┌─────────────────────────────────────────────────────┐
│                    NEXT.JS 15 APP                    │
│  ┌───────────────┐  ┌──────────────┐  ┌───────────┐ │
│  │   Frontend    │  │   API Routes │  │  Admin    │ │
│  │  React 19     │  │  /api/*      │  │  Panel    │ │
│  │  Zustand      │  │              │  │           │ │
│  │  Three.js     │  │              │  │           │ │
│  └───────┬───────┘  └──────┬───────┘  └─────┬─────┘ │
│          │                 │                │       │
│  ┌───────┴─────────────────┴────────────────┴─────┐ │
│  │              Voice Pipeline Manager             │ │
│  │  ┌─────────┐ ┌──────────┐ ┌─────────────────┐  │ │
│  │  │   STT   │ │   LLM    │ │      TTS        │  │ │
│  │  │ Router  │ │  Router  │ │     Router      │  │ │
│  │  └────┬────┘ └────┬─────┘ └───────┬─────────┘  │ │
│  │       │           │               │             │ │
│  │  ┌────▼────┐ ┌────▼─────┐  ┌──────▼──────────┐ │ │
│  │  │Deepgram │ │Groq      │  │Cartesia Sonic 3 │ │ │
│  │  │Nova-3   │ │+Llama 4  │  │ElevenLabs Flash │ │ │
│  │  │Whisper  │ │OpenAI    │  │Google Wavenet   │ │ │
│  │  │(fallbk) │ │Gemini    │  │Yandex SpeechKit │ │ │
│  │  │         │ │Ollama    │  │(Алиса)          │ │ │
│  │  └─────────┘ └──────────┘  └─────────────────┘ │ │
│  └─────────────────────────────────────────────────┘ │
│                                                     │
│  ┌──────────────────────────────────────────────────┐│
│  │              Memory & Context Layer              ││
│  │  ┌────────────┐ ┌───────────┐ ┌───────────────┐ ││
│  │  │ ChromaDB   │ │ Adaptive  │ │  Obsidian     │ ││
│  │  │ per-user   │ │ System    │ │  REST API     │ ││
│  │  │ collections│ │ Prompt    │ │  + Vault      │ ││
│  │  └────────────┘ └───────────┘ └───────────────┘ ││
│  └──────────────────────────────────────────────────┘│
│                                                     │
│  ┌──────────────────────────────────────────────────┐│
│  │              Lapel Mode Engine                   ││
│  │  ┌────────────┐ ┌───────────┐ ┌───────────────┐ ││
│  │  │ Speaker    │ │ Person    │ │  3 States     │ ││
│  │  │ Diarize    │ │ Voice DB  │ │  Controller   │ ││
│  │  │ (Deepgram) │ │ (ChromaDB)│ │               │ ││
│  │  └────────────┘ └───────────┘ └───────────────┘ ││
│  └──────────────────────────────────────────────────┘│
│                                                     │
│  ┌──────────────────────────────────────────────────┐│
│  │              Analytics & Monitoring              ││
│  │  Google Analytics · Yandex Metrika · Token Cost  ││
│  │  User Logs · API Balance Tracker                 ││
│  └──────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────┘
         │                              │
    ┌────▼────┐                   ┌─────▼─────┐
    │ChromaDB │                   │  Obsidian  │
    │ Server  │                   │  Local     │
    └─────────┘                   └────────────┘
```

### Стек технологий

| Компонент | Технология | Почему |
|-----------|-----------|--------|
| Фреймворк | Next.js 15 (App Router) | SSR, API routes, TypeScript |
| UI | React 19 + Tailwind CSS + shadcn/ui + Three.js | Сфера частиц + компоненты |
| State | Zustand | Лёгкий, реактивный |
| Auth | NextAuth.js + Google OAuth | Multi-user, роли |
| DB метаданных | SQLite (better-sqlite3) | Пользователи, настройки, логи — без внешней БД |
| Векторная БД | ChromaDB (per-user коллекции) | RAG, память, голосовые отпечатки |
| STT | Deepgram Nova-3 (primary) | 150 мс, русский, diarization |
| LLM (fast) | Groq + Llama 4 Maverick | TTFT ~500 мс, 498 tok/s |
| LLM (smart) | OpenAI GPT-4o / Gemini 2.5 Pro | Сложные задачи |
| LLM (local) | Ollama (Qwen3-14B / Llama 3.1) | Оффлайн fallback |
| TTS (fast) | Cartesia Sonic 3 | TTFB 40 мс, русский, эмоции |
| TTS (expressive) | ElevenLabs Flash v2.5 | 75 мс, [sighs], [whispers] |
| TTS (standard) | Google Wavenet | Стабильный, дешёвый |
| TTS (Алиса) | Yandex SpeechKit | Голос Алисы |
| Obsidian | REST API (Obsidian Local REST API plugin) | Чтение/запись заметок |
| Аналитика | Google Analytics + Яндекс Метрика | Веб-аналитика |

---

## Модуль 1: Авторизация и управление пользователями

### 1.1 Архитектура аутентификации

```
Google OAuth 2.0 → NextAuth.js → SQLite (users table)
                                      │
                              ┌───────┴────────┐
                              │ ADMIN          │
                              │ - manage users │
                              │ - add emails   │
                              │ - assign roles │
                              └────────────────┘
```

**Принцип**: НЕ через Google Cloud Console Audience. Вместо этого — белый список email в админке.

### 1.2 Таблица пользователей (SQLite)

```sql
CREATE TABLE users (
  id TEXT PRIMARY KEY DEFAULT (lower(hex(randomblob(16)))),
  email TEXT UNIQUE NOT NULL,
  name TEXT,
  avatar_url TEXT,
  role TEXT DEFAULT 'user' CHECK (role IN ('admin', 'user')),
  is_allowed INTEGER DEFAULT 0,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  last_login DATETIME,
  settings_json TEXT DEFAULT '{}'
);

INSERT INTO users (email, role, is_allowed) VALUES
  ('evsinanton@gmail.com', 'admin', 1);
```

### 1.3 Флоу авторизации

1. Пользователь нажимает "Войти через Google"
2. NextAuth проверяет email в таблице `users`
3. Если `is_allowed = 1` → впускаем, обновляем `last_login`
4. Если email не в таблице или `is_allowed = 0` → "Доступ запрещён"
5. Админ добавляет email заранее → при входе пользователь сразу попадает

### 1.4 Админ-панель: управление доступом

- Таблица пользователей: email, имя, роль, статус, последний вход
- Кнопка "Добавить пользователя" → ввод email → `is_allowed = 1`
- Переключатель роли: user / admin
- Кнопка "Заблокировать" → `is_allowed = 0`

---

## Модуль 2: Голосовой пайплайн (Voice Pipeline)

См. полную спецификацию в локальном файле: SPEC_001_voicezettel_2_0_architecture.md
Основные решения:

- Все STT/LLM/TTS — подключаемые модули с единым интерфейсом
- Переключение на лету без перезагрузки (lazy-инициализация провайдеров)
- 4 пресета: Молния (380-600мс), Оптимальный, Эмоциональный, Локальный
- Механизм перебивания (barge-in) с полной очисткой буферов
- UI: блок голосового пайплайна поднят над промптами

## Модуль 3: Память и контекст

- Адаптивный системный промпт: поправки пользователя сохраняются навсегда в SQLite
- ChromaDB per-user коллекции: conversations, notes, facts, entities, voice_prints, strategic
- RAG при каждом запросе: embed → search top-10 → context builder
- Short-term (RAM, 50 сообщений) + Long-term (ChromaDB)

## Модуль 4: Obsidian

- Структура: Zettelkasten/ + Архив/ + Стратегическое/
- Расширяемые сущности через конфиг EntityType
- "Шелестун" — фоновая Ollama модель для анализа архивов → инсайты

## Модуль 5: Режим петлицы

- Deepgram Nova-3 diarization + голосовые отпечатки в ChromaDB
- 3 состояния: Запись / Справочник / Участник
- Уникальные анимации сферы для каждого режима и говорящего
- Идентификация персон по контексту + голосу

## Модуль 6: Админ-панель

- 7 разделов: Пользователи, Файлы, Логи, Аналитика, API баланс, Сущности, Настройки
- Файловый менеджер с Monaco Editor + Git панель
- Механизм отчётности Comet Agent через GitHub Issues

## Модуль 7: Счётчик токенов

- Подсчёт в рублях и долларах по тарифам каждого API
- Авто-переключение при исчерпании баланса (fallback chain)
- Мониторинг остатков на каждом API

## Модуль 8: Ускорение загрузки

- CSS-placeholder пока Three.js грузится
- Pre-warm микрофон и Deepgram WebSocket
- Web Worker для частиц

## Модуль 9: Голоса

- Мужской / Женский / Клонированный
- Переключение голосом через LLM classifier
- Yandex Алиса (voice: alena)

## Модуль 10: CI/CD цикл

- Улучшенный цикл: SPEC → TASK → Antigravity кодит → тесты → PR → ревью → merge
- Автотесты (Jest + Playwright)
- Squash merge для чистых откатов
- Comet Agent отчитывается через GitHub Issues

---

## Зависимости (.env)

```env
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
NEXTAUTH_SECRET=
DEEPGRAM_API_KEY=
GROQ_API_KEY=
OPENAI_API_KEY=
GOOGLE_AI_API_KEY=
CARTESIA_API_KEY=
ELEVENLABS_API_KEY=
YANDEX_SPEECHKIT_API_KEY=
CHROMA_URL=http://localhost:8000
OBSIDIAN_REST_API_URL=http://localhost:27124
OBSIDIAN_REST_API_KEY=
NEXT_PUBLIC_GA_ID=
NEXT_PUBLIC_YM_ID=
OLLAMA_URL=http://localhost:11434
```

## Риски

1. Deepgram diarization: качество падает при >4 говорящих
2. ChromaDB: >100K документов — нужна архивация
3. Ollama: Qwen3-14B требует 16GB RAM
4. Yandex SpeechKit: нет WebSocket streaming
5. API ключи: НИКОГДА не коммитить