# TASK_018: Particle Avatar — morphing, lip-sync, visemes
**Статус:** READY_FOR_ANTIGRAVITY
**Ветка:** feature/task-018-particle-avatar
**Спецификация:** docs/specs/SPEC_001_voicezettel_2_0_architecture.md — Модуль 2.2

## Контекст
Визуал агента: облако из тысяч частиц формирует образ (лицо/фигуру). Lip-sync через visemes — рот из частиц двигается синхронно с речью. Morphing: облако → сфера → лицо. Audio-reactive + idle анимации.

## Задача
1. Particle Morphing System:
   - Morph targets: cloud (random), sphere, face, custom
   - Плавная интерполяция (lerp, 0.5-1с)
   - GLSL vertex shader: `mix(posA, posB, uMorphProgress)`
   - GPUComputationRenderer для >3000 частиц
2. Lip-sync через visemes:
   - OVR viseme set: aa, E, I, O, U, PP, SS, TH, CH, FF, kk, nn, RR, DD, sil
   - TTS провайдер возвращает viseme timestamps (ElevenLabs, Cartesia)
   - Fallback: простой phoneme mapper из текста
   - Viseme → displacement vectors для частиц в области рта
   - GLSL uniform'ы: uViseme_Aa, uViseme_Ee, etc.
   - Плавная интерполяция между visemes (60fps)
3. Audio-Reactive:
   - Web Audio API AnalyserNode → FFT данные
   - Bass → scale пульсация
   - Treble → скорость/яркость частиц
   - Voice level → общая интенсивность
4. Idle анимации:
   - Breathing: sin(time * 0.5) — расширение/сжатие
   - Blink: случайный интервал 3-7с, 150ms длительность
   - Subtle sway: Perlin noise
5. setMood() system:
   - neutral, happy, sad, angry, thinking
   - Mood влияет на цвет, скорость, форму частиц

## Файлы для создания/изменения
- `src/components/particle-system/ParticleAvatar.ts` — основной компонент
- `src/components/particle-system/morph-engine.ts` — morphing между формами
- `src/components/particle-system/lip-sync.ts` — viseme-based lip-sync
- `src/components/particle-system/mood-controller.ts` — setMood() система
- `src/components/particle-system/idle-animations.ts` — breathing, blink, sway
- `src/components/particle-system/shaders/avatar-vertex.glsl` — vertex shader с visemes
- `src/components/particle-system/shaders/avatar-fragment.glsl` — fragment shader
- `src/components/particle-system/morph-targets/` — JSON файлы позиций (sphere, face, etc.)

## Acceptance Criteria
- [ ] Morphing: облако → сфера → лицо (плавная анимация)
- [ ] Lip-sync: рот из частиц двигается синхронно с TTS аудио
- [ ] 15 visemes корректно отображаются (визуально различимы)
- [ ] Audio-reactive: частицы реагируют на звук в реальном времени
- [ ] Idle: breathing + blink работают когда ИИ молчит
- [ ] setMood('happy') визуально меняет аватар (цвет, скорость, форма)
- [ ] 3000+ частиц: FPS >55 на RTX 4090 (и >30 на средних GPU)
- [ ] Fallback lip-sync из текста работает когда TTS не даёт visemes

## Стек
Three.js 0.180+, GLSL, GPUComputationRenderer, Web Audio API
