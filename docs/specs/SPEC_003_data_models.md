# SPEC_003: Data Models — Единый источник истины

> **Версия:** 1.0
> **Дата:** 2026-03-15
> **Статус:** APPROVED

---

## Правило

Все типы данных в проекте импортируются ТОЛЬКО из `src/types/`. Агент НЕ ИМЕЕТ ПРАВА придумывать типы на ходу.

---

## TypeScript интерфейсы

### Файл: `src/types/index.ts`

```typescript
// === Режимы приложения ===
export type AppMode = 'assistant' | 'lapel' | 'agent';

// === Пользователь ===
export interface User {
  id: string;                    // UUID
  email: string;
  name: string;
  avatar_url: string | null;
  role: 'admin' | 'user';
  created_at: string;            // ISO 8601
  last_login_at: string | null;
}

// === Настройки пользователя ===
export interface UserSettings {
  user_id: string;
  stt_provider: 'deepgram' | 'whisper_local';
  llm_provider: 'groq' | 'openai' | 'deepseek' | 'gemini' | 'ollama';
  tts_provider: 'cartesia' | 'elevenlabs' | 'google' | 'yandex' | 'qwen3_tts' | 'piper';
  voice_id: string | null;       // ID голоса для TTS
  language: 'ru' | 'en';
  theme: 'dark' | 'light';
}

// === Сообщение чата ===
export interface ChatMessage {
  id: string;                    // UUID
  user_id: string;
  role: 'user' | 'assistant' | 'system';
  content: string;
  mode: AppMode;
  created_at: string;            // ISO 8601
  audio_duration_ms: number | null;
  tokens_used: number | null;
}

// === Голосовой пайплайн ===
export interface VoicePipelineResult {
  transcript: string;            // STT результат
  llm_response: string;          // LLM ответ
  audio_url: string | null;      // TTS аудио URL
  latency_ms: {
    stt: number;
    llm: number;
    tts: number;
    total: number;
  };
}

// === Агент ===
export interface Agent {
  id: string;
  name: string;
  description: string;
  system_prompt: string;
  llm_provider: string;
  model: string;
  created_by: string;            // user_id
  is_active: boolean;
  created_at: string;
}

// === Персона (Петлица) ===
export interface LapelPerson {
  id: string;
  name: string;
  voice_print_id: string | null; // ID голосового отпечатка
  avatar_url: string | null;
  notes: string;
  last_seen_at: string | null;
}

// === Health Check ===
export interface HealthStatus {
  status: 'ok' | 'error';
  services: {
    next: boolean;
    sqlite: boolean;
    chromadb: boolean;
    ollama: boolean;
  };
  timestamp: string;
}

// === Конфигурация приложения ===
export interface AppConfig {
  dataDir: string;
  sqliteDir: string;
  logsDir: string;
  voicesDir: string;
  chromadbUrl: string;
  ollamaHost: string;
  pyannoteUrl: string;
  domain: string;
  port: number;
  publicUrl: string;
  useMocks: boolean;
}
```

---

## SQLite DDL схема

### Файл: `src/lib/db/schema.sql`

```sql
-- === Пользователи ===
CREATE TABLE IF NOT EXISTS users (
  id TEXT PRIMARY KEY,
  email TEXT UNIQUE NOT NULL,
  name TEXT NOT NULL,
  avatar_url TEXT,
  role TEXT NOT NULL DEFAULT 'user' CHECK(role IN ('admin', 'user')),
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  last_login_at TEXT
);

-- === Настройки пользователя ===
CREATE TABLE IF NOT EXISTS user_settings (
  user_id TEXT PRIMARY KEY REFERENCES users(id),
  stt_provider TEXT NOT NULL DEFAULT 'deepgram',
  llm_provider TEXT NOT NULL DEFAULT 'groq',
  tts_provider TEXT NOT NULL DEFAULT 'cartesia',
  voice_id TEXT,
  language TEXT NOT NULL DEFAULT 'ru',
  theme TEXT NOT NULL DEFAULT 'dark'
);

-- === Сообщения чата ===
CREATE TABLE IF NOT EXISTS chat_messages (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL REFERENCES users(id),
  role TEXT NOT NULL CHECK(role IN ('user', 'assistant', 'system')),
  content TEXT NOT NULL,
  mode TEXT NOT NULL CHECK(mode IN ('assistant', 'lapel', 'agent')),
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  audio_duration_ms INTEGER,
  tokens_used INTEGER
);

-- === Агенты ===
CREATE TABLE IF NOT EXISTS agents (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  description TEXT NOT NULL DEFAULT '',
  system_prompt TEXT NOT NULL,
  llm_provider TEXT NOT NULL,
  model TEXT NOT NULL,
  created_by TEXT NOT NULL REFERENCES users(id),
  is_active INTEGER NOT NULL DEFAULT 1,
  created_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- === Персоны (Петлица) ===
CREATE TABLE IF NOT EXISTS lapel_persons (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  voice_print_id TEXT,
  avatar_url TEXT,
  notes TEXT NOT NULL DEFAULT '',
  last_seen_at TEXT
);

-- === Директивы (системные промпты) ===
CREATE TABLE IF NOT EXISTS directives (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL REFERENCES users(id),
  mode TEXT NOT NULL CHECK(mode IN ('assistant', 'lapel', 'agent')),
  content TEXT NOT NULL,
  is_active INTEGER NOT NULL DEFAULT 1,
  created_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- === Логи токенов ===
CREATE TABLE IF NOT EXISTS token_logs (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL REFERENCES users(id),
  provider TEXT NOT NULL,
  model TEXT NOT NULL,
  tokens_in INTEGER NOT NULL,
  tokens_out INTEGER NOT NULL,
  cost_usd REAL,
  created_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- === Индексы ===
CREATE INDEX IF NOT EXISTS idx_chat_messages_user ON chat_messages(user_id);
CREATE INDEX IF NOT EXISTS idx_chat_messages_created ON chat_messages(created_at);
CREATE INDEX IF NOT EXISTS idx_token_logs_user ON token_logs(user_id);
CREATE INDEX IF NOT EXISTS idx_token_logs_created ON token_logs(created_at);
```

---

## Правила использования

1. **Агент импортирует типы из `src/types/index.ts`** — не создаёт свои
2. **DDL применяется через миграцию** — `src/lib/db/schema.sql` выполняется при первом запуске
3. **Новые таблицы/поля** — только через новый SPEC или обновление SPEC_003
4. **UUID генерация** — `crypto.randomUUID()`
5. **Даты** — всегда ISO 8601 строки (SQLite не имеет нативного datetime)
