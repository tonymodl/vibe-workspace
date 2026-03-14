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
- **Цветные облачки**: каждый спикер (в петлице) получает уникальный цвет
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
  codeRunner: CodeExecutor;    // выполнение кода
  fileManager: FileSystemAPI;  // работа с файлами
  scheduler: SchedulerAPI;     // планирование задач
}
```

### 6.4 Оркестратор локальных ИИ

**Супер-умный оркестратор** сам решает какую модель использовать:

```typescript
// Файл: src/modules/orchestrator/ModelOrchestrator.ts
interface ModelOrchestrator {
  // Маршрутизация по типу задачи
  routeTask(task: AgentTask): ModelAssignment;
  
  // Управление VRAM (24GB лимит)
  vramManager: VRAMManager;
  
  // Очередь задач
  taskQueue: PriorityQueue<AgentTask>;
}

interface ModelAssignment {
  task: 'code' | 'vision' | 'research' | 'tts' | 'stt' | 'analysis' | 'generation';
  model: string;
  priority: number;
}

// Маппинг задач → моделей
const MODEL_ROUTING = {
  code: 'ollama/qwen2.5-coder-32b',
  vision: 'ollama/stable-diffusion',  // или API
  research: 'ollama/qwen3-32b',
  tts: 'qwen3-tts',  // localhost:8880
  stt: 'faster-whisper',
  analysis: 'ollama/qwen3-32b',
  generation: 'ollama/qwen3-32b',
};
```

**Управление VRAM:**
```typescript
interface VRAMManager {
  totalVRAM: number;  // 24576 MB
  loaded: Map<string, { model: string; vramMB: number; lastUsed: Date }>;
  
  // Загрузка модели (с выгрузкой наименее используемой при нехватке)
  loadModel(model: string): Promise<void>;
  
  // Выгрузка модели
  unloadModel(model: string): Promise<void>;
  
  // Проверка: влезет ли?
  canFit(model: string): boolean;
}
```

**ОГРАНИЧЕНИЕ**: 2×32B модели одновременно НЕ влезут. Оркестратор управляет очередью:
- Задача приходит → проверить есть ли модель в VRAM
- Если нет → выгрузить наименее нужную → загрузить нужную
- Время загрузки ~5-15 сек для 32B модели на NVMe SSD

### 6.5 Внешние ИИ-агенты

| Агент | API | Когда использовать |
|-------|-----|-------------------|
| Perplexity Computer | API | Сложные исследования, мультишаговые задачи |
| Antigravity | GitHub Tasks | Разработка кода |
| Cursor | API / MCP | Редактирование кода |
| OpenClaw | API | Computer-use (браузер, UI automation) |

### 6.6 Коммуникация между агентами

- Агенты общаются между собой по просьбе пользователя
- **Конференции агентов**: несколько агентов обсуждают задачу вместе
- Один агент в фокусе (голосовой диалог), остальные в фоне
- Вызов агента голосом по имени: "Арина, подключись к обсуждению"
- **Взаимный контроль**: агенты проверяют работу друг друга
- **Оркестратор оркестраторов**: главный модуль управляет приоритетами всех

### 6.7 Dashboard оркестратора (отдельный UI)

```typescript
interface OrchestratorDashboard {
  nodes: AgentNode[];       // все агенты как ноды
  connections: Connection[]; // связи между агентами
  taskQueue: TaskItem[];     // очередь задач
  vramStatus: VRAMStatus;    // загрузка GPU
  modelStatus: ModelStatus[]; // какие модели загружены
}

interface AgentNode {
  id: string;
  name: string;
  type: 'local_model' | 'cloud_api' | 'external_agent';
  status: 'idle' | 'working' | 'waiting' | 'error';
  currentTask?: string;
  vramUsageMB?: number;
}
```

Визуализация: карта нодов с линиями связей, статусы (цвета), текущие задачи, загрузка GPU в реальном времени.

---

## Модуль 7: Шелестун — фоновый аналитик

**ТОЛЬКО ДЛЯ ADMIN (Антон)**

### 7.1 Что анализирует

| Источник данных | Как получает | Частота |
|----------------|-------------|---------|
| Telegram переписки | Telegram Exporter (morf3uzzz) → JSON → ingestion | Разовый импорт + периодический |
| Записи звонков | Из петлицы → ChromaDB conversations | Непрерывно |
| История ассистента | ChromaDB conversations | Непрерывно |
| История петлицы | ChromaDB conversations + daily_digests | Непрерывно |
| Samsung Watch 8 (24/7) | Android companion app → REST API → SQLite | Непрерывно |
| Геопозиция | Android app + Google Location History | Непрерывно |
| Умные очки (будущее) | TBD — API очков | Будущее |
| ВКонтакте | VK API (friends, subscriptions, wall) | Периодически |
| Instagram | Instagram Basic Display API / scraping | Периодически |
| Telegram каналы/группы | Telegram Bot API + MTProto | Периодически |
| Перфокарта (привычки) | Google Sheets API | Ежедневно |

### 7.2 Что делает

1. **Карта социальных связей**: кто с кем дружит, как часто общаются, сила связи
2. **Склейка персон**: один человек = Telegram аккаунт + голосовой отпечаток + ВК профиль + Instagram
3. **Хронология общения**: когда, с кем, о чём (timeline)
4. **Дайджесты инсайтов**: еженедельные/ежемесячные отчёты
5. **Стратегический анализ**: паттерны поведения, аномалии, рекомендации
6. **Сохранение**: Obsidian/Стратегическое/ + ChromaDB strategic collection

### 7.3 Telegram Exporter Integration

**Источник**: github.com/morf3uzzz/telegram-exporter-downloads — десктопное приложение (Electron), экспорт в JSON + Markdown.

**Интеграция:**
1. Пользователь экспортирует чаты через Telegram Exporter → JSON файлы
2. VoiceZettel ingestion pipeline: JSON → парсинг сообщений → embedding → ChromaDB
3. Шелестун анализирует: даты, частоту общения, тональность, темы
4. Периодический ре-экспорт для обновления данных

### 7.4 Архитектура Шелестуна

```
┌─────────────────────────────────────┐
│         Shelestun Node              │
│  ┌──────────────────────────────┐   │
│  │  Ollama (Qwen3-32B)         │   │ ← Супер-умная модель
│  │  Автономный процесс         │   │
│  └──────────┬───────────────────┘   │
│             │                       │
│  ┌──────────▼───────────────────┐   │
│  │  Анализатор                  │   │
│  │  - Загружает данные порциями │   │
│  │  - Итеративный анализ       │   │ ← Изучает по много раз
│  │  - Cross-reference источники│   │
│  │  - Генерация инсайтов       │   │
│  └──────────┬───────────────────┘   │
│             │                       │
│  ┌──────────▼───────────────────┐   │
│  │  Планировщик                 │   │
│  │  - Фоновые задачи (cron)    │   │
│  │  - По запросу пользователя  │   │
│  │  - Очередь анализа          │   │
│  └──────────────────────────────┘   │
└─────────────────────────────────────┘
         │              │
    ┌────▼────┐    ┌────▼────┐
    │ChromaDB │    │Obsidian │
    │strategic│    │Стратег. │
    └─────────┘    └─────────┘
```

---

## Модуль 8: Умный дом

### 8.1 Архитектура

```
Голосовая команда ("выключи свет в спальне")
    │
    ▼
VoiceZettel LLM → парсинг intent → SmartHome API
    │
    ├──→ Yandex IoT API (прямой контроль)
    │    endpoint: https://api.iot.yandex.net/v1.0/
    │    OAuth scopes: iot:view, iot:control
    │
    ├──→ Home Assistant (сложные сценарии)
    │    REST API: http://localhost:8123/api/
    │    Интеграция: dext0r/yandex_smart_home (HACS)
    │
    └──→ MQTT (кастомные устройства)
         broker: mosquitto (localhost:1883)
```

### 8.2 Устройства

| Устройство | Протокол | API |
|-----------|----------|-----|
| Yandex Station Max | Yandex IoT | Управление/медиа |
| Zigbee устройства (через Station Max) | Zigbee 3.0 | Yandex IoT (как стандартные устройства) |
| Камеры | HLS stream | `devices.capabilities.video_stream` |
| Датчики (температура, влажность) | Zigbee | Yandex IoT |
| Кондиционер | IR / Zigbee | `devices.capabilities.range` (temp) + `on_off` |
| Умные лампы | Zigbee | `devices.capabilities.color_setting` + `on_off` |
| Умная розетка (перезагрузка ПК) | Zigbee | `devices.capabilities.on_off` |

### 8.3 Сценарии

- **Перезагрузка компьютера**: умная розетка off → 5сек → on
- **Мониторинг помещения**: камера + датчики → уведомление в Telegram
- **Голосовые команды**: через VoiceZettel ИЛИ через Yandex Alice (параллельно)
- **Автоматизация**: Home Assistant rules (время суток, датчики, присутствие)
- **Персонализация**: пользователь придумывает свои сценарии, ИИ помогает настроить

### 8.4 Возможный отдельный агент для умного дома

Агент "Домовой" — специализированный ИИ-агент для умного дома:
- Мониторит датчики в реальном времени
- Предлагает автоматизации на основе паттернов
- Управляет климатом адаптивно
- Реагирует на аномалии (протечка, задымление, неожиданное движение)

---

## Модуль 9: Перфокарта — геймификация жизни

### 9.1 Текущее состояние

**Google Sheet**: `1CUx0OUOhRcpRkaU4XYRgvNBEkEj_jQFyA5UkjxaDBZU`

**Структура** (из анализа):
- **10 вкладок**: Перфокарта (основная), Работа, План, и 7 других
- **~335 строк** данных (февраль 2023 → январь 2025)
- **Столбцы (31+)**: дата, сон (ложиться/встать), оценка дня (1-10), прогресс %
- **Категории**:
  - Physical Level (розовый): 15+ физических активностей
  - Mental Level (голубой): чтение, музыка, голос, видео
- **Формат**: дропдауны "Выполнено"/"Не выполне"/"Отсутствует", условное форматирование
- **Формулы**: расчёт сна, процент выполнения

### 9.2 Расширение в VoiceZettel

```typescript
// Файл: src/modules/perfocard/PerfoCardSync.ts
interface PerfoCardConfig {
  spreadsheetId: string;
  syncInterval: number;  // ms, default 3600000 (1 час)
  autoTrack: {
    sleep: boolean;       // из Samsung Watch
    steps: boolean;       // из Samsung Watch
    heartRate: boolean;   // из Samsung Watch
    meditation: boolean;  // из приложения
  };
}
```

**Автоматизация:**
1. Google Sheets API v4 — чтение/запись данных
2. Автоматическое заполнение из Samsung Watch: сон, шаги, пульс
3. Голосовое заполнение: "Вика, отметь тренировку сегодня"
4. Дайджест прогресса: ежедневный/еженедельный
5. Геймификация: очки, стрики, достижения
6. Интеграция с Шелестуном: анализ привычек во времени

---

## Модуль 10: Samsung Watch 8 интеграция

### 10.1 Архитектура

```
Galaxy Watch 8 (Wear OS 6)
    │
    │ Samsung Health Sensor SDK
    │ (real-time: HR 1Hz, EDA 1Hz, accel 25Hz)
    │
    ▼
Samsung Galaxy Phone (companion)
    │
    │ Samsung Health Data SDK (batch)
    │ (sleep, SpO2, body comp, ECG history)
    │
    │ Custom Android App (relay)
    │
    ▼
HTTPS → VoiceZettel Server
    │
    ├──→ SQLite (health_metrics)
    ├──→ ChromaDB (анализ трендов)
    └──→ Шелестун (корреляции)
```

### 10.2 Доступные метрики

| Категория | Метрики | Источник |
|-----------|---------|----------|
| Сердце | HR (1Hz), IBI, HRV, ECG waveform | Sensor SDK (real-time) / Data SDK (batch) |
| Кислород | SpO2 | Data SDK |
| Стресс | EDA (Watch 8+), stress level | Sensor SDK (real-time) |
| Сон | Фазы сна, храп, SpO2 во сне | Data SDK |
| Тело | Fat %, muscle %, water %, BMI | BIA on-demand |
| Температура | Skin temp + ambient | Continuous + on-demand |
| Активность | Шаги, дистанция, калории, этажи | Data SDK |
| Watch 8 эксклюзив | MF-BIA, Antioxidant Index, AGEs, Vascular Load | Sensor SDK |

### 10.3 Android Companion App

- Kotlin, минимальный UI (background service)
- Samsung Health Data SDK для batch-данных
- Wear OS Data Layer API для real-time streaming от часов
- HTTPS relay: каждые N минут отправка батча + real-time HR/stress через WebSocket
- Нет необходимости в partner approval для dev mode

---

## Модуль 11: Telegram бот администрирования

### 11.1 Функциональность

```typescript
// Файл: src/modules/telegram-bot/TelegramAdmin.ts
// Библиотека: telegraf.js

interface TelegramBotCommands {
  // Статус
  '/status': 'Состояние всех сервисов (CPU, RAM, GPU, disk, uptime)';
  '/health': 'Здоровье пайплайна (STT, LLM, TTS, ChromaDB, Obsidian)';
  
  // Управление
  '/restart <service>': 'Перезагрузить сервис (next, chromadb, ollama, all)';
  '/reboot': 'Перезагрузить весь сервер';
  
  // Пользователи
  '/users': 'Список пользователей, активность';
  '/whitelist add <email>': 'Добавить email в whitelist';
  '/whitelist remove <email>': 'Удалить email';
  
  // Логи
  '/logs <service> <lines>': 'Последние N строк логов';
  '/errors': 'Ошибки за последний час';
  
  // Мониторинг
  '/costs': 'Расходы по провайдерам (сегодня / неделя / месяц)';
  '/tokens': 'Токены: использовано / лимиты';
  
  // Экстренное
  '/kill <process>': 'Убить зависший процесс';
  '/backup now': 'Создать бэкап немедленно';
}
```

### 11.2 Кнопки (Inline Keyboard)

- Быстрые проверки: 🟢 Status | 🔄 Restart | 📊 Costs
- Перезагрузка: Next.js | ChromaDB | Ollama | ALL | REBOOT
- Пользователи: Список | Добавить | Заблокировать

### 11.3 Уведомления

Бот сам пишет в Telegram при:
- Ошибка сервиса (STT/LLM/TTS down)
- Высокая загрузка (CPU >90%, RAM >85%, VRAM >95%)
- Автоматический fallback сработал
- Превышение бюджета API
- Успешный бэкап / ошибка бэкапа

---

## Модуль 12: Админ-панель (Web UI)

### 12.1 Разделы

| Раздел | Содержимое |
|--------|-----------|
| **Dashboard** | Статус сервисов, GPU usage, активные пользователи, расходы сегодня |
| **Пользователи** | Таблица, добавление/блокировка, роли, whitelist |
| **Файловый менеджер** | Дерево файлов + Monaco Editor, превью markdown, Git-панель |
| **Логи** | Фильтрация по event_type, пользователю, времени |
| **Аналитика** | GA4 виджеты + Яндекс Метрика |
| **API & Баланс** | Per-model cost tracking (USD + RUB), графики расходов |
| **Сущности** | Управление типами заметок, toggle вкл/выкл |
| **Настройки** | Конфигурация проекта, API ключи, домен |
| **Оркестратор** | Dashboard агентов (Модуль 6.7) |

### 12.2 Файловый менеджер + Git

- Дерево файлов проекта
- Monaco Editor для редактирования
- Превью markdown
- Git-панель: коммиты, diff, push
- **Comet Agent**: может редактировать файлы → автокоммит → push → GitHub Issue

### 12.3 Логи

```sql
CREATE TABLE activity_logs (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id TEXT REFERENCES users(id),
  event_type TEXT NOT NULL,       -- 'stt', 'llm', 'tts', 'auth', 'error', 'agent', 'lapel'
  event_data JSON NOT NULL,       -- детали события
  tokens_used INTEGER,
  cost_usd REAL,
  provider TEXT,
  latency_ms INTEGER,
  created_at TEXT DEFAULT (datetime('now'))
);

CREATE INDEX idx_logs_type ON activity_logs(event_type);
CREATE INDEX idx_logs_user ON activity_logs(user_id);
CREATE INDEX idx_logs_date ON activity_logs(created_at);
```

---

## Модуль 13: Obsidian интеграция

### 13.1 Структура vault

```
Obsidian/
├── Zettelkasten/
│   ├── Заметки/
│   ├── Идеи/
│   ├── Факты/
│   ├── Персоны/
│   └── Задачи/
├── Архив/
│   ├── Телеграм переписки/
│   └── Старые заметки/
├── Стратегическое/    ← Шелестун пишет сюда
│   ├── Инсайты ИИ/
│   ├── Факты обо мне/
│   └── Аналитика/
└── _templates/
```

### 13.2 Расширяемые сущности

```typescript
interface EntityType {
  id: string;
  name: string;           // "Персона", "Идея", "Факт"
  icon: string;
  template: string;       // markdown template
  fields: EntityField[];
  extractionRules: string; // промпт для LLM извлечения
  isActive: boolean;       // toggle вкл/выкл
  isCustom: boolean;       // пользовательский или системный
}

interface EntityField {
  name: string;
  type: 'text' | 'date' | 'number' | 'tags' | 'relation';
  required: boolean;
}
```

**Базовые сущности**: Заметки, Идеи, Факты, Персоны, Задачи
**Кастомные**: пользователь создаёт свои через админку, toggle вкл/выкл

---

## Модуль 14: Инфраструктура и безопасность

### 14.1 Сеть и доступ

```
Интернет
    │
    │ voicezettel.online → белый IP
    │
    ▼
┌────────────────────────────┐
│ Nginx (reverse proxy)      │
│ - Let's Encrypt SSL/TLS    │
│ - Rate limiting             │
│ - WebSocket proxy           │
│ - Static files cache        │
└────────────┬───────────────┘
             │
             ▼
┌────────────────────────────┐
│ Fail2ban                   │
│ - SSH brute-force          │
│ - API rate abuse           │
│ - Auth failure blocking    │
└────────────┬───────────────┘
             │
             ▼
┌────────────────────────────┐
│ Next.js 15 (port 3000)    │
│ Middleware: auth check     │
└────────────────────────────┘
```

**НЕ Cloudflare** — белый IP напрямую. Заказан у провайдера.

### 14.2 Бэкап

```typescript
interface BackupConfig {
  targets: {
    local: '/backup/voicezettel/';        // локальный SSD
    cloud: 'backblaze-b2://voicezettel/'; // или Yandex Object Storage
    remote: 'ssh://wife-pc/backup/';       // компьютер жены в другом доме
  };
  schedule: '0 3 * * *'; // каждую ночь в 3:00
  what: [
    'sqlite/*.db',        // все SQLite базы
    'chromadb/',           // весь ChromaDB
    'obsidian/',           // весь Obsidian vault
    '.env',                // секреты
    'config/',             // конфигурации
  ];
  retention: {
    daily: 7,    // 7 дневных бэкапов
    weekly: 4,   // 4 недельных
    monthly: 3,  // 3 месячных
  };
}
```

### 14.3 SSL и домен

- Домен: `voicezettel.online`
- DNS A-record → белый IP
- Let's Encrypt через certbot + nginx plugin
- Auto-renewal через cron

### 14.4 PWA

- `manifest.json` для мобильных
- Service Worker для offline fallback
- Push notifications (Telegram бот как альтернатива)

---

## Модуль 15: CI/CD и рабочий цикл

### 15.1 Workflow разработки

```
Perplexity Computer (PM)
    │
    ├─→ Создаёт SPEC файлы (docs/specs/)
    ├─→ Создаёт TASK файлы (.tasks/)
    │
    ▼
Antigravity (Developer)
    │
    ├─→ Читает TASK
    ├─→ Создаёт feature/task-NNN-slug ветку
    ├─→ Кодит в TypeScript, русские комментарии
    ├─→ Тесты: Jest (unit) + API (integration) + Playwright (E2E)
    ├─→ PR → main (squash merge)
    │
    ├─→ Max 3 попытки, потом BLOCKED
    │
    ▼
Main branch
    │
    └─→ Теги: v2.0.0-alpha.N
```

### 15.2 Специальный QA-агент

Для вайбкодинга на уровне качества, недоступном человеку:

```typescript
interface QAAgent {
  // Запускается ПОСЛЕ каждого PR от Antigravity
  runChecks(pr: PullRequest): QAReport;
  
  checks: [
    'typecheck',           // tsc --noEmit
    'lint',                // eslint
    'unit_tests',          // jest
    'integration_tests',   // API tests
    'e2e_tests',           // Playwright
    'security_scan',       // npm audit + custom checks
    'performance_check',   // lighthouse CI
    'dependency_check',    // circular deps, unused imports
    'architecture_check',  // соответствие SPEC
  ];
}
```

### 15.3 Comet Agent (интеграция)

- Может редактировать файлы через админку
- Автокоммит → push в ветку `agent/comet-{timestamp}`
- Создаёт GitHub Issue с описанием изменений
- НЕ пушит в main напрямую

### 15.4 Откат

- Squash merge → один коммит на задачу → чистый `git revert`
- Теги версий: `v2.0.0-alpha.1`, `v2.0.0-alpha.2`, ...
- При критической ошибке: `git revert HEAD` + Telegram уведомление

---

## Модуль 16: Счётчик токенов и Auto-Fallback

### 16.1 Token & Cost Tracking

```sql
CREATE TABLE token_usage (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id TEXT NOT NULL REFERENCES users(id),
  provider TEXT NOT NULL,       -- 'groq', 'openai', 'deepseek', 'elevenlabs', etc.
  model TEXT NOT NULL,          -- 'llama-4-maverick', 'gpt-4o', etc.
  operation TEXT NOT NULL,      -- 'stt', 'llm_input', 'llm_output', 'tts', 'embedding'
  tokens INTEGER NOT NULL,
  cost_usd REAL NOT NULL,
  cost_rub REAL NOT NULL,       -- по текущему курсу
  latency_ms INTEGER,
  created_at TEXT DEFAULT (datetime('now'))
);

CREATE TABLE provider_rates (
  provider TEXT NOT NULL,
  model TEXT NOT NULL,
  operation TEXT NOT NULL,
  cost_per_1m_tokens REAL NOT NULL, -- USD per 1M tokens
  updated_at TEXT DEFAULT (datetime('now')),
  PRIMARY KEY (provider, model, operation)
);
```

### 16.2 Auto-Fallback

При ошибке провайдера — автоматическое переключение на следующий по приоритету:

| Pipeline | Цепочка fallback |
|----------|-----------------|
| STT | Deepgram Nova-3 → Whisper (local) |
| LLM | Groq → OpenAI → DeepSeek → Gemini → Ollama (local) |
| TTS | Cartesia → ElevenLabs → Google → Yandex → Qwen3-TTS → Piper |

- Уведомление пользователю при переключении (toast notification)
- Уведомление в Telegram бот для admin
- Автоматический retry через 5 мин для упавшего провайдера

---

## Модуль 17: RAG Engine (замена NotebookLM)

### 17.1 Почему не NotebookLM

- Нет публичного API (2026)
- Enterprise API ($250+/мес) — только управление notebooks, нет Q&A endpoint
- Browser automation (Playwright) — хрупкий, нарушает ToS

### 17.2 Собственный RAG

```typescript
// Файл: src/modules/rag-engine/RAGEngine.ts
interface RAGEngine {
  // Document ingestion
  ingest(source: DocumentSource): Promise<void>;
  
  // Source-grounded Q&A
  query(question: string, collectionId: string): Promise<RAGResponse>;
  
  // Summary generation
  summarize(collectionId: string): Promise<string>;
}

interface DocumentSource {
  type: 'pdf' | 'url' | 'youtube' | 'text' | 'telegram_export';
  content: string | Buffer;
  metadata: Record<string, any>;
}

interface RAGResponse {
  answer: string;
  sources: { text: string; documentId: string; score: number }[];
  confidence: number;
}
```

**Стек**: LangChain + ChromaDB + Gemini Flash (или Ollama)
- Document ingestion: PDF → text, URL → scrape, YouTube → transcript
- Chunking: RecursiveCharacterTextSplitter (1000 chars, 200 overlap)
- Embedding: text-embedding-3-small (OpenAI) или local (sentence-transformers)
- Retrieval: ChromaDB cosine similarity, top-10
- Generation: LLM с контекстом из retrieved chunks
- Source attribution: цитаты с ссылками на источники

---

## Модуль 18: Аналитика

### 18.1 Web Analytics

- **Google Analytics 4**: gtag.js, pageviews, events, user properties
- **Яндекс Метрика**: параллельно, вебвизор, карта кликов

### 18.2 Custom Analytics (внутренняя)

Через SQLite `activity_logs` + админ-панель:
- Количество запросов по провайдерам
- Средняя латентность по пайплайну
- Расходы по дням/неделям/месяцам
- Активность пользователей
- Топ тем разговоров
- Использование режимов (ассистент / петлица / агент)

---

## Приоритеты MVP

### Волна A (MVP) — ПЕРВЫЙ ДЕПЛОЙ

| # | Модуль | Что включает |
|---|--------|-------------|
| 1 | Auth (Модуль 1) | Google OAuth, whitelist, роли |
| 2 | UI (Модуль 2) | Сфера частиц (базовая), чат, настройки |
| 3 | Voice Pipeline (Модуль 3) | STT + LLM + TTS, fallback, переключение, barge-in |
| 4 | Memory (Модуль 4) | ChromaDB, адаптивный промпт, RAG |
| 5 | Lapel Basic (Модуль 5) | Diarization 20+, идентификация, 3 состояния |
| 6 | Admin Basic (Модуль 12) | Dashboard, пользователи, логи |
| 7 | Infra (Модуль 14) | Nginx, SSL, Fail2ban, белый IP |
| 8 | Token Tracking (Модуль 16) | Счётчик, auto-fallback |

### Волна B — РАСШИРЕНИЕ

| # | Модуль |
|---|--------|
| 9 | Agents Basic (Модуль 6) — создание агентов, базовый оркестратор |
| 10 | Obsidian (Модуль 13) — полная интеграция |
| 11 | Telegram Bot (Модуль 11) — администрирование |
| 12 | Voice Cloning (Модуль 3.4) — ElevenLabs + XTTS |

### Волна C — ПОЛНАЯ СИСТЕМА

| # | Модуль |
|---|--------|
| 13 | Particle Avatar (Модуль 2.2) — morphing, lip-sync, visemes |
| 14 | Mafia Engine (Модуль 5.7) — игровой режим |
| 15 | Agent Orchestrator Full (Модуль 6.4-6.7) — VRAM management, conferences |
| 16 | RAG Engine (Модуль 17) — замена NotebookLM |

### Волна D — ПРОДВИНУТОЕ

| # | Модуль |
|---|--------|
| 17 | Shelestun (Модуль 7) — фоновый аналитик |
| 18 | Smart Home (Модуль 8) — Yandex IoT + Home Assistant |
| 19 | Samsung Watch (Модуль 10) — companion app |
| 20 | Perfocard (Модуль 9) — геймификация |

---

## Зависимости (npm packages)

```json
{
  "dependencies": {
    "next": "^15.0.0",
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "three": "^0.180.0",
    "zustand": "^5.0.0",
    "next-auth": "^5.0.0",
    "better-sqlite3": "^11.0.0",
    "chromadb": "^1.8.0",
    "telegraf": "^4.16.0",
    "@deepgram/sdk": "^3.0.0",
    "openai": "^4.0.0",
    "@google/generative-ai": "^0.20.0",
    "googleapis": "^140.0.0",
    "langchain": "^0.3.0",
    "@langchain/community": "^0.3.0",
    "tailwindcss": "^4.0.0",
    "lucide-react": "^0.400.0"
  },
  "devDependencies": {
    "typescript": "^5.6.0",
    "jest": "^30.0.0",
    "@playwright/test": "^1.48.0",
    "eslint": "^9.0.0",
    "@types/three": "^0.180.0",
    "@types/better-sqlite3": "^7.6.0"
  }
}
```

---

## Ограничения и риски

| Риск | Вероятность | Митигация |
|------|------------|-----------|
| VRAM 24GB не хватит для 2 моделей | Высокая | Очередь задач, выгрузка/загрузка моделей |
| pyannote 4+ одновременных голоса | Средняя | Мафия — по очереди, crosstalk ≤3 норма |
| Deepgram API down | Низкая | Whisper local fallback |
| Белый IP — DDoS | Средняя | Fail2ban + rate limit + Telegram alert |
| Voice cloning юридические риски | Средняя | Только личное использование, локальный инференс |
| NotebookLM API не появится | Высокая | Собственный RAG (Модуль 17) |
| Samsung Health партнёрское одобрение | Средняя | Dev mode для тестирования |
| Telegram Exporter прекратит обновления | Низкая | Формат JSON стабилен, fork при необходимости |
| Производительность Three.js 3000+ частиц + lip-sync | Низкая | GPUComputationRenderer, ShaderMaterial |

---

## Glossary (глоссарий)

| Термин | Определение |
|--------|-----------|
| Barge-in | Механизм перебивания ИИ голосом |
| ChromaDB | Векторная база данных для RAG |
| DER | Diarization Error Rate — метрика точности diarization |
| Diarization | Разделение аудио по говорящим |
| GLSL | OpenGL Shading Language — язык шейдеров |
| MF-BIA | Multi-Frequency Bioelectrical Impedance Analysis |
| Morphing | Плавная трансформация формы (облако → лицо) |
| pyannote | Библиотека для speaker diarization на Python/PyTorch |
| RAG | Retrieval-Augmented Generation — генерация с поиском по базе знаний |
| RVC | Retrieval-Based Voice Conversion |
| Шелестун | Автономный фоновый аналитик (Модуль 7) |
| TalkingHead | Open-source Three.js библиотека для lip-sync аватаров |
| Viseme | Визуальная фонема — форма рта для звука |
| VRAM | Video RAM — память GPU |
| XTTS | Cross-lingual Text-to-Speech (Coqui) |
| Перфокарта | Система трекинга привычек / геймификации жизни |
