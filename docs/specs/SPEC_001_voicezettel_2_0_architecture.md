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

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         NEXT.JS 15 (App Router)                         │
│                                                                          │
│  ┌─────────────┐ ┌──────────────┐ ┌───────────┐ ┌────────────────────┐  │
│  │  Frontend   │ │  API Routes  │ │   Admin   │ │  Agent Dashboard   │  │
│  │  React 19   │ │  /api/*      │ │   Panel   │ │  (оркестратор UI)  │  │
│  │  Three.js   │ │  WebSocket   │ │           │ │                    │  │
│  │  Zustand    │ │              │ │           │ │                    │  │
│  └──────┬──────┘ └──────┬───────┘ └─────┬─────┘ └────────┬───────────┘  │
│         │               │               │                │              │
│  ┌──────┴───────────────┴───────────────┴────────────────┴────────────┐  │
│  │                    Voice Pipeline Manager                          │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────────────┐ │  │
│  │  │STT Router│  │LLM Router│  │TTS Router│  │  Voice Cloner      │ │  │
│  │  │Deepgram  │  │Groq      │  │Cartesia  │  │  ElevenLabs/XTTS   │ │  │
│  │  │+Whisper  │  │OpenAI    │  │ElevenLabs│  │  /RVC/OpenVoice    │ │  │
│  │  │          │  │DeepSeek  │  │Google    │  │                    │ │  │
│  │  │          │  │Gemini    │  │Yandex    │  │                    │ │  │
│  │  │          │  │Ollama    │  │Qwen3-TTS │  │                    │ │  │
│  │  │          │  │          │  │Piper     │  │                    │ │  │
│  │  └──────────┘  └──────────┘  └──────────┘  └────────────────────┘ │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ┌──────────────────────┐  ┌──────────────────────────────────────────┐  │
│  │  Memory & Context    │  │  Lapel Engine (20+ спикеров)            │  │
│  │  ┌────────────────┐  │  │  ┌────────────┐  ┌──────────────────┐  │  │
│  │  │ ChromaDB       │  │  │  │ pyannote   │  │ Person Resolver  │  │  │
│  │  │ per-user       │  │  │  │ diarize    │  │ (voice prints +  │  │  │
│  │  │ 6 коллекций    │  │  │  │ +Deepgram  │  │  context match)  │  │  │
│  │  ├────────────────┤  │  │  ├────────────┤  ├──────────────────┤  │  │
│  │  │ Adaptive       │  │  │  │ 3 States   │  │ Mafia Game       │  │  │
│  │  │ System Prompt  │  │  │  │ Controller │  │ Engine           │  │  │
│  │  ├────────────────┤  │  │  ├────────────┤  ├──────────────────┤  │  │
│  │  │ Obsidian       │  │  │  │ Ear Hints  │  │ Daily Digest     │  │  │
│  │  │ REST API       │  │  │  │ (whisper)  │  │ Generator        │  │  │
│  │  └────────────────┘  │  │  └────────────┘  └──────────────────┘  │  │
│  └──────────────────────┘  └──────────────────────────────────────────┘  │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────────┐│
│  │                    Agent System                                      ││
│  │  ┌──────────────┐  ┌────────────────┐  ┌──────────────────────────┐ ││
│  │  │ Agent CRUD   │  │  Orchestrator  │  │  Local Model Queue       │ ││
│  │  │ (3-step flow)│  │  (маршрутизация│  │  (VRAM management,       │ ││
│  │  │              │  │   задач)       │  │   load/unload models)    │ ││
│  │  ├──────────────┤  ├────────────────┤  ├──────────────────────────┤ ││
│  │  │ RAG Engine   │  │  Agent Comms   │  │  External Agents         │ ││
│  │  │ (замена      │  │  (конференции, │  │  (Perplexity, Cursor,    │ ││
│  │  │  NotebookLM) │  │   взаимный     │  │   Antigravity, OpenClaw) │ ││
│  │  │              │  │   контроль)    │  │                          │ ││
│  │  └──────────────┘  └────────────────┘  └──────────────────────────┘ ││
│  └──────────────────────────────────────────────────────────────────────┘│
│                                                                          │
│  ┌─────────────────┐  ┌────────────────┐  ┌────────────────────────────┐│
│  │  Shelestun      │  │  Smart Home    │  │  Perfocard (Gamification) ││
│  │  (фоновый       │  │  Yandex IoT    │  │  Google Sheets API        ││
│  │   аналитик)     │  │  + Home Asst.  │  │  + авто-трекинг           ││
│  │  Ollama node    │  │  + MQTT/Zigbee │  │                            ││
│  └─────────────────┘  └────────────────┘  └────────────────────────────┘│
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────────┐│
│  │  Infrastructure: Telegram Bot Admin · Token Tracker · GA4 · Metrika ││
│  │  Nginx Reverse Proxy · Let's Encrypt · Fail2ban · Backup            ││
│  └──────────────────────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────────────────────┘
         │              │              │              │
    ┌────▼────┐   ┌─────▼─────┐  ┌────▼────┐   ┌────▼──────┐
    │ChromaDB │   │ Obsidian  │  │ SQLite  │   │ Ollama    │
    │ Server  │   │ Local     │  │ (meta)  │   │ (models)  │
    └─────────┘   └───────────┘  └─────────┘   └───────────┘
```

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

```typescript
// Файл: src/lib/auth.ts
interface AuthConfig {
  provider: 'google'; // только Google OAuth
  whitelist: string[]; // белый список email (из SQLite)
  roles: 'admin' | 'user';
  adminEmail: 'evsinanton@gmail.com'; // единственный admin
}
```

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

### 1.3 Таблицы SQLite

```sql
-- Пользователи
CREATE TABLE users (
  id TEXT PRIMARY KEY DEFAULT (lower(hex(randomblob(16)))),
  email TEXT UNIQUE NOT NULL,
  name TEXT NOT NULL,
  role TEXT NOT NULL DEFAULT 'user' CHECK(role IN ('admin', 'user')),
  avatar_url TEXT,
  is_active INTEGER NOT NULL DEFAULT 1,
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  last_login TEXT,
  settings_json TEXT DEFAULT '{}'
);

-- Белый список (отдельная таблица для гибкости)
CREATE TABLE email_whitelist (
  email TEXT PRIMARY KEY,
  added_by TEXT NOT NULL,
  added_at TEXT NOT NULL DEFAULT (datetime('now')),
  note TEXT
);
```

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

### 2.2 Particle Visual System (Агент)

Технологический стек:
- **Three.js Points** с `AdditiveBlending` для рендеринга
- **GLSL ShaderMaterial** — кастомные vertex/fragment шейдеры
- **GPUComputationRenderer** (Three.js) — физика частиц на GPU при >3000 частиц
- **Web Audio API AnalyserNode** — FFT данные для аудио-реактивности

```typescript
// Файл: src/components/particle-system/ParticleAvatar.ts
interface ParticleAvatarConfig {
  particleCount: number; // 3000 по умолчанию
  morphTargets: {
    sphere: Float32Array; // позиции сферы
    face: Float32Array;   // позиции лица (из 3D скана / готовой модели)
    cloud: Float32Array;  // рандомное облако
    custom: Float32Array; // кастомная форма
  };
  visemes: Record<string, Float32Array>; // Aa, Ee, Ih, Oh, Ou → смещения частиц
  mood: 'neutral' | 'happy' | 'sad' | 'angry' | 'thinking'; // setMood()
}
```

**Lip-sync через visemes:**
1. TTS провайдер возвращает timestamps visemes (ElevenLabs / Cartesia поддерживают)
2. Если TTS не даёт visemes → LLM-предикция по тексту (simple phoneme mapper)
3. Viseme set (TalkingHead OVR): `aa, E, I, O, U, PP, SS, TH, CH, FF, kk, nn, RR, DD, sil`
4. Каждый viseme = вектор смещений для частиц в области рта
5. Плавная интерполяция между visemes (lerp, ~60fps)

**GLSL uniform'ы для шейдера:**
```glsl
uniform float uViseme_Aa;
uniform float uViseme_Ee;
uniform float uViseme_Ih;
// ... все 15 visemes
uniform float uMorphProgress; // 0=облако, 1=цель
uniform float uAudioLevel;    // из AnalyserNode
uniform float uAudioBass;     // низкие частоты
uniform float uAudioTreble;   // высокие частоты
uniform vec3 uMoodColor;      // цвет настроения
uniform float uBreathPhase;   // idle breathing
uniform float uBlinkPhase;    // idle blinking
```

**Idle-анимации:**
- Breathing: sin(time * 0.5) — медленное расширение/сжатие
- Blink: случайный интервал 3-7сек, длительность 150ms
- Subtle sway: Perlin noise на позиции

**Audio-reactive:** Web Audio API → `AnalyserNode.getByteFrequencyData()` → uniform'ы в шейдер:
- Общий уровень → масштаб сферы
- Басы → "пульсация" (scale)
- Высокие → "искры" (скорость частиц)
- Голос → морфинг рта (visemes)

### 2.3 Чат-интерфейс

- Взять из speaker1991/voicezettel
- **Цветные облачка**: каждый спикер (в петлице) получает уникальный цвет
- **Обновление имени задним числом**: когда ИИ узнаёт имя говорящего, все предыдущие сообщения "Speaker N" обновляются на реальное имя
- Markdown рендеринг в сообщениях
- Streaming ответов LLM (token by token)
- Индикатор "печатает..." с анимацией частиц

### 2.4 Панель настроек

Порядок блоков (сверху вниз):
1. **Виджеты** — включение/выключение компонентов
2. **Голосовой пайплайн** — STT / LLM / TTS провайдеры
3. **Системный промпт** — ручная настройка + history директив
4. **Сущности** — типы заметок, toggle вкл/выкл

**Голосовой пайплайн настройки:**
- Пресеты: ⚡ Молния / ⚖️ Оптимальный / 🎭 Эмоциональный / 💻 Локальный
- Мозги (LLM): выпадающий список с иконками провайдеров, свободный выбор приоритета
- Озвучка (TTS): выпадающий список + кнопка превью голоса
- Голос: Мужской / Женский / Клонированный + кнопка "Попросить голосом" (запись сэмпла)
- **Имя ассистента**: редактируемое в любой момент — голосом, текстом или в настройках. Имя = триггер для петлицы ("Вика, подскажи...")
- **Голос по умолчанию**: женский

**ПРИНЦИП: пользователь свободно выбирает приоритеты моделей в ЛЮБОЙ конфигурации. Система не навязывает ограничения.**

---

## Модуль 3: Голосовой пайплайн

### 3.1 STT (Speech-to-Text)

**Primary**: Deepgram Nova-3 (WebSocket streaming)
- Русский язык
- Diarization (speaker labels)
- ~150ms latency
- Endpoint: `wss://api.deepgram.com/v1/listen?model=nova-3&language=ru&diarize=true`

**Fallback**: Whisper (faster-whisper, локально на GPU)
- Модель: large-v3 (best accuracy) или medium (faster)
- faster-whisper с CTranslate2 → ~1.5сек на 10сек аудио на RTX 4090

**Цепочка fallback**: Deepgram → Whisper (local)

### 3.2 LLM Router

```typescript
// Файл: src/lib/voice-pipeline/llm-router.ts
interface LLMProvider {
  id: string;
  name: string;
  icon: string;
  endpoint: string;
  model: string;
  priority: number; // пользователь устанавливает
  maxTokens: number;
  costPerMToken: { input: number; output: number }; // USD
  capabilities: ('fast' | 'smart' | 'code' | 'vision' | 'local')[];
}

// Доступные провайдеры
const PROVIDERS: LLMProvider[] = [
  { id: 'groq-maverick', name: 'Groq + Llama 4 Maverick', capabilities: ['fast'] },
  { id: 'openai-gpt4o', name: 'GPT-4o', capabilities: ['smart', 'vision'] },
  { id: 'gemini-25pro', name: 'Gemini 2.5 Pro', capabilities: ['smart'] },
  { id: 'deepseek-v3', name: 'DeepSeek V3', capabilities: ['smart', 'code'] },
  { id: 'ollama-qwen3', name: 'Ollama: Qwen3-32B', capabilities: ['smart', 'local'] },
  { id: 'ollama-coder', name: 'Ollama: Qwen2.5-Coder-32B', capabilities: ['code', 'local'] },
];
```

**Принцип**: пользователь свободно выставляет приоритеты. Новые модели легко добавляются (конфиг), старые — удаляются. Fallback автоматический по приоритету.

### 3.3 TTS Router

| Провайдер | TTFB | Русский | Эмоции | Тип |
|-----------|------|---------|--------|-----|
| Cartesia Sonic 3 | 40ms | ✅ | ✅ (speed, emotion controls) | Cloud |
| ElevenLabs Flash v2.5 | 75ms | ✅ | ✅ ([sighs], [whispers]) | Cloud |
| Google Wavenet | ~300ms | ✅ | ❌ | Cloud |
| Yandex SpeechKit | ~200ms | ✅ (alena) | ✅ (роли: good, evil) | Cloud |
| Qwen3-TTS | ~100ms | ✅ | ✅ | Local (localhost:8880) |
| Piper TTS | ~50ms | ✅ | ❌ | Local (fallback) |

**Цепочка fallback**: Cartesia → ElevenLabs → Google → Yandex → Qwen3-TTS → Piper

### 3.4 Voice Cloning

Три уровня клонирования:

| Метод | Где работает | Сэмпл | Латентность | Качество | Когда использовать |
|-------|-------------|-------|-------------|----------|-------------------|
| ElevenLabs IVC | Cloud | 10 сек | 75ms | Высокое | Основной метод, мультиязычный |
| XTTS v2 (Coqui) | RTX 4090 | 6 сек | <200ms | Хорошее | Локальный, русский поддерживается |
| RVC v2 | RTX 4090 | 5 мин обучения | <100ms | Отличное | Voice conversion, real-time |
| OpenVoice v2 | RTX 4090 | 30 сек | GPU-dependent | Хорошее | Zero-shot, MIT license |

**Привязка к агенту**: клонированный голос привязывается на Шаге 3 создания агента.

**Для знаменитостей**: RVC v2 + локальный инференс (серая зона юридически → только для личного использования).

### 3.5 Barge-in (перебивание)

```
Пользователь начал говорить
    │
    ▼
VAD детектирует голос → INTERRUPT SIGNAL
    │
    ├─→ 1. Flush audio playback buffer (мгновенно)
    ├─→ 2. Abort текущий LLM stream (AbortController)
    ├─→ 3. Close TTS WebSocket connection
    └─→ 4. Start new STT stream
    
    Время: < 100ms от детекции до нового потока
```

### 3.6 Переключение на лету

- Без перезагрузки страницы
- Lazy init провайдеров (не создавать соединения пока не нужны)
- При переключении: graceful shutdown текущего → init нового → ~200ms переход
- Переключение доступно голосом ("переключи на DeepSeek") И через UI
- State хранится в Zustand, persist в localStorage

### 3.7 Быстрый старт микрофона

```typescript
// При загрузке приложения (до клика пользователя использовать нельзя — браузеры блокируют)
// → после первого клика / тапа:
async function warmUpAudio() {
  // 1. getUserMedia() — получаем доступ к микрофону
  const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
  
  // 2. Warm-up Deepgram WebSocket (pre-connect)
  const dgSocket = new WebSocket('wss://api.deepgram.com/v1/listen?...');
  
  // 3. Pre-connect к API endpoints (DNS prefetch + TCP handshake)
  // <link rel="preconnect" href="https://api.groq.com" />
  // <link rel="preconnect" href="https://api.cartesia.ai" />
}
```

---

## Модуль 4: Память и контекст

### 4.1 Адаптивный системный промпт

**КРИТИЧНО**: поправки пользователя сохраняются НАВСЕГДА.

```sql
CREATE TABLE user_directives (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id TEXT NOT NULL REFERENCES users(id),
  directive TEXT NOT NULL,        -- "Не называй меня на вы"
  source TEXT NOT NULL,           -- 'voice' | 'text' | 'settings'
  is_active INTEGER DEFAULT 1,
  created_at TEXT DEFAULT (datetime('now')),
  deactivated_at TEXT,
  deactivated_by TEXT             -- 'user_command' | 'opposite_directive'
);
```

**Механизм:**
1. Каждая фраза пользователя проходит через LLM classifier
2. Classifier определяет: это поправка/директива? (да/нет)
3. Если да → сохранить в `user_directives`
4. Если противоположная команда → деактивировать старую директиву
5. При сборке системного промпта: базовый промпт + все активные директивы

### 4.2 ChromaDB (per-user коллекции)

Для каждого пользователя — 6 коллекций:

| Коллекция | Хранит | Embedding model |
|-----------|--------|-----------------|
| `{user_id}_conversations` | История разговоров (пары вопрос-ответ) | text-embedding-3-small |
| `{user_id}_notes` | Заметки из Obsidian | text-embedding-3-small |
| `{user_id}_facts` | Факты о пользователе | text-embedding-3-small |
| `{user_id}_entities` | Все сущности (персоны, проекты, локации) | text-embedding-3-small |
| `{user_id}_voice_prints` | Голосовые отпечатки персон (pyannote embeddings) | pyannote (192-dim) |
| `{user_id}_strategic` | Стратегическая информация (от Шелестуна) | text-embedding-3-small |

### 4.3 Контекстный RAG

Каждая фраза пользователя:
1. Транскрипция (STT) → текст
2. Embed текст → вектор
3. Поиск top-10 в ChromaDB (conversations + facts + entities + strategic)
4. Сборка промпта: system prompt + директивы + RAG контекст + текущий диалог
5. LLM генерация ответа
6. Сохранение пары (вопрос, ответ) в `conversations`

### 4.4 Shared Memory

- Все LLM провайдеры используют ОДНУ и ту же ChromaDB
- Переключение мозга НЕ теряет контекст
- History сессии хранится в Zustand (short-term) + ChromaDB (long-term)

---

## Модуль 5: Режим петлицы (Lapel Mode) — 20+ персон

### 5.1 Архитектура diarization

**Гибридный подход**: Deepgram (real-time) + pyannote (идентификация).

```
Микрофон (16kHz mono)
    │
    ├──→ Deepgram WebSocket (streaming)
    │    └─→ Текст + SPEAKER_0, SPEAKER_1, ... (anonymous labels)
    │    └─→ Таймстампы + длительность
    │
    └──→ Audio Buffer (5-sec chunks)
         │
         └──→ pyannote segmentation + embedding extraction
              └─→ 192-dim embedding per speaker per chunk
              └─→ Поиск в ChromaDB voice_prints (cosine similarity)
              └─→ Если > threshold (0.7) → "Это Миша!"
              └─→ Если < threshold → "Speaker N" (новый голос)
```

**pyannote на RTX 4090:**
- 14 секунд обработки на 1 час аудио (250x real-time для batch)
- Streaming: <100ms на 500ms chunk для embedding extraction
- Модель: `pyannote/speaker-diarization-3.1` + `pyannote/segmentation-3.0`
- Kmax=3 одновременных спикеров на 5-сек окно (для мафии где говорят по очереди — достаточно)
- Глобальный agglomerative clustering отслеживает 20+ идентичностей

**Параметры pipeline:**
```python
pipeline = Pipeline.from_pretrained("pyannote/speaker-diarization-3.1")
pipeline = pipeline.to(torch.device("cuda"))
diarization = pipeline(audio, min_speakers=2, max_speakers=25)
```

### 5.2 Идентификация персон

3 уровня идентификации:
1. **Голосовой отпечаток** — pyannote embedding → ChromaDB voice_prints → cosine > 0.7
2. **Контекстный** — "Давай, Миша, твой ход" → LLM извлекает имя из контекста
3. **Ручной** — пользователь говорит "это Миша" → сохранить embedding + имя

**Обновление задним числом**: когда имя стало известно → обновить ВСЕ предыдущие реплики Speaker N → Миша (в чате и в ChromaDB).

### 5.3 Три состояния петлицы

| Состояние | Описание | Сфера | Активация |
|-----------|----------|-------|-----------|
| **Запись** | Пассивная транскрибация, молча слушает | Мягкая пульсация, приглушённые тона | По умолчанию |
| **Справочник** | Отвечает на прямое обращение по имени | Яркая пульсация при ответе | "Вика, подскажи..." |
| **Участник** | Активный участник, мозговой штурм, заполняет паузы | Активная анимация, цветные всплески | Включается вручную или по команде |

### 5.4 Подсказки на ухо (Ear Hints)

- Скрытый наушник: AirPods, Galaxy Buds, микронаушник, проводной — любой
- ИИ подсказывает НАТИВНО, не сбивая пользователя
- **Аудио-роутинг**: TTS → наушник (канал A), STT ← микрофон (канал B)
- Разделение каналов через Web Audio API routing / system audio routing

**Адаптация под контекст:**

| Контекст | Поведение |
|----------|-----------|
| Мафия | Вероятности "кто мафия", подсказки по голосованию |
| Переговоры | Факты о собеседнике, аргументы, подсказки |
| Тет-а-тет | Дополнительная информация, имена, даты |
| Экзамен | Ответы, формулы, подсказки |
| Повседневность | Напоминания, рекомендации |

### 5.5 Петлица 24/7

- Рассчитана на круглосуточную работу
- **Авто-суммаризация дня**: ежедневный дайджест (с кем говорил, ключевые темы, решения)
- Дайджест учитывает пожелания пользователя (что важно, что пропустить)
- Хранение: ChromaDB conversations + SQLite daily_digests

```sql
CREATE TABLE daily_digests (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id TEXT NOT NULL REFERENCES users(id),
  date TEXT NOT NULL,
  summary_text TEXT NOT NULL,
  speakers JSON NOT NULL,        -- [{name, duration_min, topics}]
  key_decisions JSON,
  action_items JSON,
  mood_analysis TEXT,
  created_at TEXT DEFAULT (datetime('now'))
);
```

### 5.6 Анимации сферы в петлице

| Событие | Анимация |
|---------|----------|
| Запись (idle) | Мягкая пульсация, приглушённые тона |
| Голос владельца | Синий trail, усиленная пульсация |
| Голос другого | Уникальный цвет (генерируется из hash имени/ID) |
| Несколько говорящих | Разноцветные потоки переплетаются |
| Переход состояний | Плавный морфинг 0.5с |
| ИИ отвечает | Яркий всплеск цвета ассистента |

### 5.7 Игровой режим "Мафия"

**ЧАСТЫЙ сценарий** — требует отдельного UI и логики.

```typescript
// Файл: src/modules/mafia/MafiaEngine.ts
interface MafiaGame {
  id: string;
  players: MafiaPlayer[];
  rounds: MafiaRound[];
  currentPhase: 'day' | 'night' | 'vote' | 'discussion';
  timeline: MafiaEvent[]; // хронология в реальном времени
}

interface MafiaPlayer {
  id: string;
  name: string;
  voicePrintId: string; // ссылка на ChromaDB voice_prints
  role?: 'mafia' | 'citizen' | 'don' | 'sheriff' | 'doctor'; // скрыта до конца
  isAlive: boolean;
  suspicionScore: number; // 0-100, вероятность что мафия
  votes: { round: number; votedFor: string }[];
  psychoProfile?: string; // склейка с Telegram переписками
}

interface MafiaRound {
  number: number;
  phase: 'day' | 'night';
  speeches: { playerId: string; text: string; timestamp: number }[];
  votes: { voterId: string; targetId: string }[];
  eliminated?: string; // playerId
}
```

**Функциональность:**
1. Отдельный UI: список игроков, роли (скрытые), статус живой/мёртвый
2. Трекинг: кто голосовал за кого, кто кого подозревает
3. Вероятности "кто мафия" — на основе логики, хронологии, слов, поведения
4. Психопрофили игроков — склейка с данными из Telegram переписок (Шелестун)
5. Хронология всей игры в реальном времени
6. Подсказки на ухо владельцу

---

## Модуль 6: Система агентов

### 6.1 Три режима приложения

Переключение свайпом по шару: **Ассистент** ↔ **Петлица** ↔ **Агент**

В режиме "Агент":
- Список созданных агентов (карточки с аватарами)
- Кнопка "Создать нового агента"
- Активный агент — голосовой диалог, остальные в фоне

### 6.2 Создание агента (3 шага)

**ШАГ 1 — Экспертиза:**
```typescript
interface AgentExpertise {
  prompt: string;             // "создай агента по инвестициям"
  sources: ExpertiseSource[]; // собранные источники
  ragCollection: string;      // ChromaDB collection для агента
}

interface ExpertiseSource {
  type: 'youtube' | 'paper' | 'forum' | 'telegram' | 'website' | 'book';
  url: string;
  title: string;
  summary: string;
  embeddingIds: string[];  // IDs в ChromaDB
}
```

**Процесс:**
1. Пользователь даёт короткий промпт (текст/голос): "создай агента-инвестора"
2. ИИ решает какая экспертиза нужна
3. Автоматический сбор СОТЕН источников: YouTube, научные статьи, форумы, Telegram
4. Источники → ingestion в ChromaDB (отдельная коллекция для агента)
5. **Замена NotebookLM**: собственный RAG на LangChain + ChromaDB + Gemini Flash
   - Document ingestion (PDF, URL, video transcripts)
   - Source-grounded Q&A
   - Summary generation
6. Мягкий вопрос пользователю: "есть ли пожелания по экспертизе?"
7. Всё общение с агентом — в долговременной памяти

**ШАГ 2 — Характер:**
```typescript
interface AgentCharacter {
  name: string;               // задаётся пользователем или генерируется
  traits: CharacterTraits;
  voiceStyle: 'formal' | 'casual' | 'friendly' | 'authoritative';
  directives: string[];       // вечные поправки пользователя
}

interface CharacterTraits {
  humor: number;      // 0-100 (бегунок/слайдер)
  empathy: number;    // 0-100
  directness: number; // 0-100
  creativity: number; // 0-100
  formality: number;  // 0-100
  patience: number;   // 0-100
}
```

- VoiceZettel знает пользователя → автоматически подбирает оптимальные черты
- Бегунки/слайдеры для ручной корректировки
- Голосовые замечания сохраняются навсегда (директивы)
- Имя меняется в любой момент

**ШАГ 3 — Внешность:**
- Шар из облака точек генерирует образ агента
- Варианты: женщина, мужчина, персонаж, существо, голливудская звезда
- С ГОЛОСОМ И ВНЕШНОСТЬЮ знаменитости (клонированный голос + particle avatar)
- Стиль ВСЕГДА из частиц

### 6.3 Инструменты агента

```typescript
interface AgentToolkit {
  // Встроенные
  memory: ChromaDBAccess;      // доступ к своей коллекции
  obsidian: ObsidianAPI;       // чтение/запись заметок
  webSearch: SearchAPI;        // поиск в интернете
  
  // Внешние
  openClaw: OpenClawAPI;       // computer-use agent
  telegramBot: TelegramBotAPI; // отправка уведомлений
  smartHome: SmartHomeAPI;     // управление умным домом (отдельный агент)
  
  // Специализированные (добавляются по необходимости)
  calendar: CalendarAPI;
  email: EmailAPI;
  github: GitHubAPI;
}
```

### 6.4 Оркестратор агентов

**Конференции агентов:**
```typescript
interface AgentConference {
  id: string;
  topic: string;
  participants: AgentParticipant[];
  moderator: 'user' | 'auto'; // пользователь или автоматически
  mode: 'debate' | 'brainstorm' | 'review' | 'custom';
  rounds: ConferenceRound[];
  output: ConferenceOutput; // итоговое решение/суммари
}
```

**Взаимный контроль:** агенты могут проверять работу друг друга (code review, fact-check).

**Очередь задач + VRAM менеджмент:**
```typescript
interface ModelQueue {
  queue: ModelTask[];
  currentModel: string | null; // что сейчас загружено в VRAM
  vramBudget: number;          // 24GB
  
  async loadModel(modelName: string): Promise<void> {
    if (this.currentModel === modelName) return; // уже загружена
    if (this.currentModel) await this.unloadModel(this.currentModel); // освободить VRAM
    await ollamaClient.pull(modelName); // загрузить нужную
    this.currentModel = modelName;
  }
}
```

### 6.5 Внешние агенты

| Агент | Интеграция | Когда использовать |
|-------|-----------|-------------------|
| Perplexity | REST API | Актуальный поиск с источниками |
| Cursor | MCP | Code tasks |
| Antigravity | API | Специализированные задачи |
| OpenClaw | WebSocket | Computer-use (автоматизация рабочего стола) |

---

## Модуль 7: Шелестун

**Концепция**: Шелестун — это отдельный Ollama-агент, работающий в фоне 24/7, только для admin (Антона). Он наблюдает за информационным полем и "шелестит" интересными находками.

### 7.1 Источники данных

```typescript
interface ShelestunSources {
  telegram: {
    channels: string[];      // подписки пользователя
    chats: string[];         // переписки (с согласия)
    groups: string[];        // групповые чаты
    parser: 'telegram-exporter' | 'pyrogram'; // инструмент парсинга
  };
  social: {
    twitter: string[];       // аккаунты за которыми следим
    reddit: string[];        // subreddits
    hn: boolean;             // Hacker News top
  };
  clock: {
    timezone: string;        // Asia/Barnaul
    workHours: [9, 22];      // активные часы для уведомлений
  };
  geo: {
    homeLocation: [number, number]; // координаты дома
    trackDevice: 'phone' | 'watch' | 'both';
  };
  health: SamsungHealthData; // через companion app
  calendar: GoogleCalendarAPI;
}
```

### 7.2 Социальные графы

```typescript
interface SocialGraph {
  nodes: Person[];
  edges: Relationship[];
  
  // Из Telegram переписок
  buildFromTelegram(chats: TelegramChat[]): void;
  
  // Обновление при новых взаимодействиях
  updateRelationship(person1: string, person2: string, interaction: Interaction): void;
}

interface Person {
  id: string;
  name: string;
  aliases: string[];           // "Миша", "Михаил", "Misha"
  voicePrintId?: string;       // если записан
  telegramId?: string;
  psychoProfile?: string;      // аналитика характера
  lastSeen?: string;
  topics: string[];            // о чём обычно говорит
  sentiment: number;           // -1 to 1
}
```

### 7.3 Уведомления и триггеры

```typescript
interface ShelestunAlert {
  type: 'news' | 'person' | 'opportunity' | 'reminder' | 'health' | 'location';
  priority: 'whisper' | 'normal' | 'urgent'; // "шелест" vs нормальное vs срочное
  delivery: 'voice' | 'telegram_bot' | 'notification';
  timing: 'immediate' | 'next_conversation' | 'scheduled';
  content: string;
}
```

**Примеры триггеров:**
- "Миша вернулся из отпуска" (анализ Telegram)
- "Выход нового исследования по теме инвестиций"
- "Напоминание: завтра встреча с Ромой"
- "Стресс-индекс высокий сегодня" (Galaxy Watch)
- "Ты рядом с супермаркетом — список покупок:"

### 7.4 Ollama для Шелестуна

```yaml
# docker-compose.yml (Shelestun service)
shelestun:
  image: ollama/ollama
  volumes:
    - ./shelestun:/app
  environment:
    OLLAMA_MODEL: qwen3:32b
    TASK_SCHEDULE: "*/15 * * * *"  # каждые 15 минут
  deploy:
    resources:
      reservations:
        devices:
          - driver: nvidia
            device_ids: ['0']
            capabilities: [gpu]
```

**Архитектура**: отдельный Node.js сервис (`shelestun-service`) → общается с основным приложением через WebSocket/Redis.

---

## Модуль 8: Obsidian-интеграция и сущности

### 8.1 Obsidian REST API

```typescript
// Файл: src/lib/obsidian-client.ts
interface ObsidianClient {
  baseUrl: string; // http://localhost:27123
  apiKey: string;
  
  // Основные операции
  readNote(path: string): Promise<string>;
  writeNote(path: string, content: string): Promise<void>;
  appendToNote(path: string, content: string): Promise<void>;
  searchNotes(query: string): Promise<SearchResult[]>;
  listNotes(folder: string): Promise<string[]>;
  
  // Теги и метаданные
  getTaggedNotes(tag: string): Promise<string[]>;
  updateFrontmatter(path: string, data: object): Promise<void>;
}
```

### 8.2 Система сущностей

| Тип сущности | Описание | Хранение |
|-------------|----------|----------|
| Персона | Контакт, знакомый, публичная фигура | Obsidian + ChromaDB |
| Проект | Работа, личный проект, хобби | Obsidian + SQLite |
| Локация | Место, адрес, точка интереса | Obsidian + SQLite |
| Идея | Случайная мысль, инсайт | Obsidian |
| Задача | To-do, ремайндер, дедлайн | Obsidian + SQLite tasks |
| Событие | Встреча, праздник, дедлайн | Obsidian + Google Calendar |
| Материал | Книга, фильм, курс, ссылка | Obsidian |
| Рефлексия | Дневниковые записи, эмоции | Obsidian (приватное) |

**Toggle-управление**: каждый тип сущности включается/выключается в настройках. Выключенный тип не распознаётся и не сохраняется.

### 8.3 Self-Heal механизм

**Источник**: tonymodl/Voicezettel → Self-Heal логика.

```typescript
// Файл: src/lib/self-heal.ts
interface SelfHealCheck {
  component: string;
  status: 'ok' | 'degraded' | 'failed';
  lastCheck: string;
  autoFix?: () => Promise<void>;
}

async function runHealthChecks(): Promise<SelfHealReport> {
  const checks = await Promise.all([
    checkChromaDB(),
    checkObsidianAPI(),
    checkOllamaModels(),
    checkSTTProviders(),
    checkLLMProviders(),
    checkTTSProviders(),
  ]);
  
  // Автоматическое исправление если возможно
  for (const check of checks.filter(c => c.status !== 'ok')) {
    if (check.autoFix) await check.autoFix();
  }
  
  return { checks, timestamp: new Date().toISOString() };
}
```

Health check dashboard доступен в admin панели.

---

## Модуль 9: Умный дом

### 9.1 Яндекс IoT

```typescript
// Файл: src/lib/smart-home/yandex-iot.ts
interface YandexIoTClient {
  // Алиса
  sendAliceCommand(command: string): Promise<void>;
  getAliceStatus(): Promise<AliceStatus>;
  
  // Устройства (не через Алису, напрямую)
  listDevices(): Promise<YandexDevice[]>;
  controlDevice(deviceId: string, action: DeviceAction): Promise<void>;
  getDeviceState(deviceId: string): Promise<DeviceState>;
}
```

### 9.2 Home Assistant

```typescript
// Файл: src/lib/smart-home/home-assistant.ts
interface HomeAssistantClient {
  wsUrl: string; // ws://homeassistant.local:8123/api/websocket
  token: string;
  
  // Entity control
  callService(domain: string, service: string, entityId: string, data?: object): Promise<void>;
  getState(entityId: string): Promise<HAState>;
  subscribeStateChanges(callback: (state: HAState) => void): void;
  
  // Примеры
  turnOnLight(entityId: string): Promise<void>;
  setTemperature(entityId: string, temp: number): Promise<void>;
  lockDoor(entityId: string): Promise<void>;
}
```

### 9.3 MQTT / Zigbee

```typescript
// Файл: src/lib/smart-home/mqtt-client.ts
interface MQTTSmartHome {
  broker: string; // mqtt://localhost:1883
  topics: {
    sensors: 'zigbee2mqtt/+/+'; // температура, влажность, движение
    commands: 'zigbee2mqtt/+/set'; // управление
    status: 'zigbee2mqtt/bridge/state';
  };
}
```

### 9.4 Голосовые команды умного дома

| Команда | Действие |
|---------|----------|
| "Выключи свет" | HA → turn_off all lights |
| "Включи кондей на 22" | Yandex IoT → set AC temp |
| "Покажи камеру в прихожей" | RTSP stream → UI overlay |
| "Кто звонил в дверь?" | Дверной звонок event log |
| "Есть ли кто дома?" | Motion sensors status |

### 9.5 Galaxy Watch 8 (Samsung Health)

```typescript
// Android companion app → HTTP API → VoiceZettel backend
interface GalaxyWatchData {
  heartRate: number;
  hrv: number;          // Heart Rate Variability
  stressIndex: number;  // 1-100
  spo2: number;         // кислород в крови
  steps: number;
  caloriesBurned: number;
  sleep: SleepData;
  workoutData: WorkoutData;
  bodyTemperature: number;
  bloodPressure?: BloodPressureData;
}
```

**Использование**: Шелестун анализирует данные часов → предупреждает о высоком стрессе, аномальном пульсе, рекомендует паузы.

---

## Модуль 10: Телеграм-бот (adminка)

```typescript
// Файл: src/bot/telegram-bot.ts
const bot = new Telegraf(process.env.TELEGRAM_BOT_TOKEN!);

// Команды
bot.command('status', async (ctx) => { /* статус системы */ });
bot.command('whitelist_add', async (ctx) => { /* добавить email */ });
bot.command('whitelist_remove', async (ctx) => { /* удалить email */ });
bot.command('users', async (ctx) => { /* список пользователей */ });
bot.command('stats', async (ctx) => { /* статистика использования */ });
bot.command('tokens', async (ctx) => { /* потраченные токены */ });
bot.command('shelestun', async (ctx) => { /* статус Шелестуна */ });

// Уведомления
interface TelegramNotifications {
  newUserRequest: (email: string) => void; // кто-то просит доступ
  systemAlert: (message: string) => void;   // критическая ошибка
  shelestunAlert: (alert: ShelestunAlert) => void; // Шелестун нашёл что-то интересное
  dailyDigest: (digest: DailyDigest) => void; // дайджест дня
}
```

**Polling vs Webhook**: webhook через ngrok/белый IP для production, polling для разработки.

---

## Модуль 11: Перфокарта (Gamification)

### 11.1 Концепция

Перфокарта = геймификация ЖИЗНИ. Автоматический трекинг активностей → очки → достижения → прогресс.

### 11.2 Google Sheets интеграция

```typescript
// Файл: src/lib/perfocard/google-sheets.ts
interface PerfocardSheets {
  spreadsheetId: string;
  sheets: {
    activities: 'Activities';  // лог активностей
    habits: 'Habits';         // привычки и стрики
    achievements: 'Achievements'; // разблокированные достижения
    dashboard: 'Dashboard';   // сводная статистика
  };
}
```

### 11.3 Авто-трекинг через голос и сенсоры

```typescript
interface ActivityDetector {
  // Из разговоров (NLU)
  extractActivities(transcript: string): Activity[];
  // Примеры: "поиграл в теннис" → {type: 'sport', name: 'tennis', duration: ?}
  
  // Из Galaxy Watch
  detectWorkout(watchData: GalaxyWatchData): Workout | null;
  
  // Из геолокации
  detectPlaceVisit(location: GeoLocation): PlaceVisit | null;
  
  // Из календаря
  importCalendarEvents(events: CalendarEvent[]): Activity[];
}
```

### 11.4 Структура активности

```typescript
interface Activity {
  id: string;
  type: ActivityType; // sport | work | learning | social | health | creative | rest
  name: string;
  duration?: number;   // минуты
  intensity?: number;  // 1-10
  points: number;      // автоматически расчитываются
  timestamp: string;
  source: 'voice' | 'watch' | 'manual' | 'calendar' | 'geo';
  verified: boolean;   // подтверждено пользователем
}

type ActivityType = 
  | 'sport'      // спорт, тренировки
  | 'work'       // работа, проекты
  | 'learning'   // обучение, чтение
  | 'social'     // общение, встречи
  | 'health'     // медицина, самочувствие
  | 'creative'   // творчество, хобби
  | 'rest';      // отдых, сон
```

### 11.5 Достижения

```typescript
interface Achievement {
  id: string;
  name: string;
  description: string;
  icon: string; // emoji или SVG
  condition: AchievementCondition;
  rarity: 'common' | 'rare' | 'epic' | 'legendary';
  unlockedAt?: string;
}
```

---

## Модуль 12: Трекинг токенов

```typescript
// Файл: src/lib/token-tracker.ts
interface TokenUsage {
  provider: string;    // 'groq', 'openai', 'anthropic', etc.
  model: string;
  inputTokens: number;
  outputTokens: number;
  cost: number;        // USD
  timestamp: string;
  userId: string;
  sessionId: string;
  context: string;     // 'assistant' | 'lapel' | 'agent' | 'shelestun'
}

// SQLite таблица
// SELECT provider, model, SUM(cost) FROM token_usage GROUP BY 1, 2
// Отображается в: настройки (текущий пользователь) + admin панель (все)
```

**Дашборд** показывает:
- Расход по провайдерам (график)
- Топ-5 дорогих сессий
- Прогноз месячных расходов
- Сравнение с предыдущими периодами

---

## Модуль 13: Аналитика (GA4 + Яндекс Метрика)

```typescript
// Файл: src/lib/analytics.ts
interface AnalyticsEvents {
  // Голосовые события
  voice_session_start: { mode: 'assistant' | 'lapel' | 'agent'; provider: string };
  voice_session_end: { duration: number; turns: number };
  barge_in: { provider: string };
  
  // Провайдеры
  provider_switch: { from: string; to: string; reason: 'user' | 'fallback' };
  provider_error: { provider: string; error: string };
  
  // Агенты
  agent_created: { step: 1 | 2 | 3 };
  agent_conversation: { agentId: string; turns: number };
  agent_conference: { participants: number; mode: string };
  
  // Умный дом
  smart_home_command: { device: string; action: string };
}
```

---

## Модуль 14: Инфраструктура и деплой

### 14.1 Nginx конфигурация

```nginx
# /etc/nginx/sites-available/voicezettel
server {
    listen 80;
    listen [::]:80;
    server_name voicezettel.online www.voicezettel.online;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name voicezettel.online www.voicezettel.online;
    
    ssl_certificate /etc/letsencrypt/live/voicezettel.online/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/voicezettel.online/privkey.pem;
    
    # Next.js приложение
    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
    
    # WebSocket для голоса
    location /ws {
        proxy_pass http://localhost:3001;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_read_timeout 3600;
    }
    
    # ChromaDB (внутренний, не публичный)
    location /chromadb {
        internal;
        proxy_pass http://localhost:8000;
    }
}
```

### 14.2 PM2 конфигурация

```javascript
// ecosystem.config.js
module.exports = {
  apps: [
    {
      name: 'voicezettel',
      script: 'npm',
      args: 'start',
      env: { NODE_ENV: 'production', PORT: 3000 },
      restart_delay: 5000,
      max_restarts: 10
    },
    {
      name: 'shelestun',
      script: 'node',
      args: 'services/shelestun/index.js',
      cron_restart: '0 4 * * *' // рестарт в 4 утра
    },
    {
      name: 'chromadb',
      script: 'chroma',
      args: 'run --path ./data/chroma --port 8000'
    }
  ]
};
```

### 14.3 Бэкап стратегия

```
Ежедневно в 03:00:
├── SQLite dump → ./backups/sqlite/YYYY-MM-DD.db
├── ChromaDB snapshot → ./backups/chroma/YYYY-MM-DD/
├── Obsidian vault → ./backups/obsidian/YYYY-MM-DD.zip
└── .env файл → encrypted → Backblaze B2

Еженедельно:
└── Полный архив → Yandex Object Storage

Полное время восстановления: < 30 мин
```

### 14.4 Безопасность

```
Fail2ban:
- SSH: 3 попытки → бан 1 час
- HTTPS: 100 req/min → бан 10 мин

ufw:
- 22/tcp (SSH) — только whitelist IP
- 80/tcp (HTTP → redirect)
- 443/tcp (HTTPS)
- Всё остальное DENY

Secretes:
- .env не в git
- API ключи через systemd environment
- Ротация ключей каждые 90 дней (напоминание через Telegram бот)
```

---

## Модуль 15: Frontend роутинг и структура проекта

```
src/
├── app/                        # Next.js App Router
│   ├── page.tsx                # Главная (сфера + чат)
│   ├── admin/                  # Админ панель
│   │   ├── page.tsx            # Дашборд
│   │   ├── users/page.tsx      # Управление пользователями
│   │   ├── tokens/page.tsx     # Трекинг токенов
│   │   └── health/page.tsx     # Self-heal статус
│   ├── agents/                 # Агенты
│   │   ├── page.tsx            # Список агентов
│   │   ├── create/page.tsx     # Создание (3 шага)
│   │   └── [id]/page.tsx       # Агент детали
│   ├── mafia/page.tsx          # Игра мафия
│   └── api/                   # API роуты
│       ├── auth/[...nextauth]/ # NextAuth.js
│       ├── voice/              # Голосовой пайплайн
│       │   ├── stt/route.ts
│       │   ├── llm/route.ts
│       │   └── tts/route.ts
│       ├── memory/             # ChromaDB операции
│       ├── obsidian/           # Obsidian API proxy
│       ├── agents/             # CRUD агентов
│       ├── smart-home/         # Умный дом
│       └── admin/              # Административные операции
├── components/
│   ├── particle-system/        # Three.js сфера, avatar, lip-sync
│   ├── chat/                   # Интерфейс чата
│   ├── settings/               # Панель настроек
│   ├── lapel/                  # Режим петлицы
│   ├── agents/                 # UI агентов
│   └── ui/                    # shadcn/ui компоненты
├── lib/
│   ├── auth.ts                # NextAuth конфиг
│   ├── db.ts                  # SQLite (better-sqlite3)
│   ├── chroma.ts              # ChromaDB клиент
│   ├── voice-pipeline/        # STT/LLM/TTS роутеры
│   ├── obsidian-client.ts     # Obsidian REST API
│   ├── smart-home/            # IoT интеграции
│   ├── shelestun/             # Шелестун клиент
│   ├── perfocard/             # Перфокарта
│   ├── token-tracker.ts       # Трекинг токенов
│   ├── self-heal.ts           # Self-heal
│   └── analytics.ts           # GA4 + Яндекс Метрика
├── modules/
│   └── mafia/                 # Игровая логика мафии
├── store/
│   ├── voice.ts               # Zustand voice state
│   ├── agents.ts              # Zustand agents state
│   └── settings.ts            # Zustand settings state
└── types/
    └── index.ts               # Глобальные типы
```

---

## Модуль 16: API маршруты (детально)

### 16.1 Голосовой пайплайн

```
POST /api/voice/stt
  Body: { audio: base64, provider: 'deepgram' | 'whisper' }
  Response: { text: string, speakers?: DiarizationResult[] }

POST /api/voice/llm
  Body: { message: string, userId: string, sessionId: string, agentId?: string }
  Response: SSE stream → { token: string } | { done: true, usage: TokenUsage }

POST /api/voice/tts
  Body: { text: string, provider: string, voiceId?: string }
  Response: audio/mpeg stream

WS /api/voice/stream
  → Real-time голосовой пайплайн (STT → LLM → TTS в одном WebSocket)
```

### 16.2 Агенты

```
GET    /api/agents              → список агентов пользователя
POST   /api/agents              → создать агента
GET    /api/agents/:id          → детали агента
PUT    /api/agents/:id          → обновить агента
DELETE /api/agents/:id          → удалить агента

POST   /api/agents/:id/chat     → сообщение агенту
POST   /api/agents/conference   → запустить конференцию

POST   /api/agents/:id/expertise/ingest → загрузить источник в RAG
GET    /api/agents/:id/expertise/sources → список источников
```

### 16.3 Память

```
POST   /api/memory/search       → семантический поиск
POST   /api/memory/store        → сохранить запись
DELETE /api/memory/:id          → удалить запись
GET    /api/memory/directives   → активные директивы
DELETE /api/memory/directives/:id → деактивировать директиву
```

---

## Модуль 17: Тестирование и CI

### 17.1 Структура тестов

```
tests/
├── unit/
│   ├── voice-pipeline/         # Тесты STT/LLM/TTS роутеров
│   ├── memory/                 # Тесты ChromaDB операций
│   └── agents/                 # Тесты агентной логики
├── integration/
│   ├── api.test.ts             # End-to-end API тесты
│   └── voice-pipeline.test.ts  # Полный пайплайн
└── e2e/
    └── playwright/             # UI тесты (Playwright)
```

### 17.2 CI/CD

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
      - run: npm run type-check
      - run: npm run lint
      - run: npm test
  
  deploy:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to server
        run: |
          ssh $SERVER_USER@$SERVER_IP 'cd /app && git pull && npm ci && npm run build && pm2 restart voicezettel'
```

---

## Модуль 18: Переменные окружения

```bash
# .env.local

# ===== AUTH =====
NEXTAUTH_URL=https://voicezettel.online
NEXTAUTH_SECRET=<random_256bit_hex>
GOOGLE_CLIENT_ID=<from_google_cloud_console>
GOOGLE_CLIENT_SECRET=<from_google_cloud_console>

# ===== STT =====
DEEPGRAM_API_KEY=<key>

# ===== LLM =====
GROQ_API_KEY=<key>
OPENAI_API_KEY=<key>
GEMINI_API_KEY=<key>
DEEPSEEK_API_KEY=<key>
OLLAMA_BASE_URL=http://localhost:11434

# ===== TTS =====
CARTESIA_API_KEY=<key>
ELEVENLABS_API_KEY=<key>
GOOGLE_TTS_API_KEY=<key>
YANDEX_SPEECHKIT_KEY=<key>
QWEN3_TTS_URL=http://localhost:8880

# ===== MEMORY =====
CHROMADB_URL=http://localhost:8000
OPENAI_EMBEDDING_KEY=<key> # для text-embedding-3-small

# ===== INTEGRATIONS =====
OBSIDIAN_API_URL=http://localhost:27123
OBSIDIAN_API_KEY=<key>
TELEGRAM_BOT_TOKEN=<key>
HOME_ASSISTANT_URL=http://homeassistant.local:8123
HOME_ASSISTANT_TOKEN=<long_lived_token>
YANDEX_IOT_TOKEN=<oauth_token>
GOOGLE_SHEETS_CLIENT_ID=<key>
GOOGLE_SHEETS_CLIENT_SECRET=<key>

# ===== ANALYTICS =====
NEXT_PUBLIC_GA4_ID=G-XXXXXXXXXX
NEXT_PUBLIC_YM_ID=XXXXXXXX

# ===== BACKUP =====
BACKBLAZE_KEY_ID=<key>
BACKBLAZE_APPLICATION_KEY=<key>
YANDEX_OBJECT_STORAGE_KEY=<key>

# ===== MISC =====
NODE_ENV=production
SERVER_URL=https://voicezettel.online
ADMIN_EMAIL=evsinanton@gmail.com
```

---

## Приоритет имплементации

| Этап | Что делаем | Зависимости |
|------|-----------|-------------|
| **Phase 1** (фундамент) | Auth + UI skeleton + голосовой пайплайн (базовый) | — |
| **Phase 2** (память) | ChromaDB + адаптивный промпт + директивы | Phase 1 |
| **Phase 3** (петлица) | Diarization + идентификация персон + ear hints | Phase 2 |
| **Phase 4** (агенты) | Создание агентов + RAG + конференции | Phase 2 |
| **Phase 5** (экосистема) | Шелестун + умный дом + перфокарта | Phase 3+4 |
| **Phase 6** (polish) | Мафия + клонирование + визуал | Phase 3 |

---

## Риски и митигация

| Риск | Вероятность | Влияние | Митигация |
|------|------------|---------|----------|
| VRAM переполнение (2×32B) | Высокая | Высокое | ModelQueue с выгрузкой/загрузкой |
| Deepgram недоступен | Средняя | Высокое | Fallback на Whisper (<2сек) |
| Юридические риски клонирования | Высокая | Среднее | Только локальный инференс, личное использование |
| pyannote ложные срабатывания | Средняя | Среднее | Порог 0.7 + ручное подтверждение |
| Obsidian API недоступен | Низкая | Низкое | Graceful degradation, кэш |
| Утечка данных (Telegram) | Низкая | Критическое | Локальное хранение, шифрование, fail2ban |
| Высокий latency LLM | Средняя | Высокое | Groq как primary (498 tok/s), fallback цепочки |

---

## Глоссарий

| Термин | Определение |
|--------|-------------|
| Петлица | Режим записи разговоров с несколькими участниками |
| Шелестун | Фоновый автономный аналитик, только для admin |
| Перфокарта | Система геймификации жизни с авто-трекингом |
| Barge-in | Перебивание ИИ во время его речи |
| VAD | Voice Activity Detection — детектор голосовой активности |
| Diarization | Разделение аудио по спикерам |
| Viseme | Визуальная фонема — форма рта для звука |
| VRAM | Video RAM — память GPU |
| XTTS | Cross-lingual Text-to-Speech (Coqui) |
| Перфокарта | Система трекинга привычек / геймификации жизни |
