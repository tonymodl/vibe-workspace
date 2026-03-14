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
│  │  └──────────┘  └────────────────┘  └──────────────────────────┘ ││
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