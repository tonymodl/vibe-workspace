# TASK_004: Сфера частиц Three.js + три режима анимации
**Статус:** READY_FOR_ANTIGRAVITY
**Ветка:** feature/task-004-particle-sphere
**Спецификация:** docs/specs/SPEC_001_voicezettel_2_0_architecture.md — Модуль 2.1

## Контекст
Сфера частиц — центральный визуальный элемент VoiceZettel. Берём из speaker1991/voicezettel и оптимизируем. Три режима с уникальными анимациями: Ассистент, Петлица, Агент. Переключение свайпом.

## Задача
1. Перенести Three.js сферу частиц из speaker1991/voicezettel
2. Оптимизация загрузки:
   - CSS placeholder (SVG gradient) до загрузки WebGL
   - Lazy init Three.js после первого взаимодействия
   - Progressive particles: 500 → 1000 → 3000 (300ms ступеньки)
   - Web Worker для расчёта позиций (если OffscreenCanvas поддерживается)
   - Кеширование скомпилированных шейдеров
3. Три режима анимации:
   - **Ассистент**: стандартная пульсирующая сфера, idle breathing (sin wave)
   - **Петлица**: "ушная раковина", цветные потоки (по 1 цвету на спикера)
   - **Агент**: placeholder для будущего morphing (пока просто другой цвет/форма)
4. Переключение режимов: горизонтальный свайп (touch + mouse drag)
   - Плавный морфинг при переходе (0.5с lerp)
   - Индикатор текущего режима (dots внизу)
5. Audio-reactive: Web Audio API AnalyserNode → uniform в шейдер
   - Bass → scale пульсация
   - Treble → скорость частиц
   - Level → общая интенсивность
6. GLSL ShaderMaterial с uniform'ами: uAudioLevel, uAudioBass, uAudioTreble, uBreathPhase, uMode

## Файлы для создания/изменения
- `src/components/particle-system/ParticleSphere.tsx` — React обёртка
- `src/components/particle-system/three-scene.ts` — Three.js сцена
- `src/components/particle-system/shaders/vertex.glsl` — vertex shader
- `src/components/particle-system/shaders/fragment.glsl` — fragment shader
- `src/components/particle-system/audio-analyzer.ts` — Web Audio API
- `src/components/particle-system/mode-controller.ts` — переключение режимов
- `src/stores/ui-store.ts` — Zustand: текущий режим (assistant/lapel/agent)

## Acceptance Criteria
- [ ] Сфера рендерится с 3000 частицами без просадок FPS (<16ms frame time)
- [ ] CSS placeholder показывается до загрузки WebGL (нет белого экрана)
- [ ] Три режима визуально отличаются (разные анимации/цвета)
- [ ] Свайп влево/вправо переключает режимы с плавным морфингом
- [ ] Audio-reactive: сфера реагирует на звук микрофона в реальном времени
- [ ] Idle breathing работает когда нет звука
- [ ] Zustand store хранит текущий режим, синхронизирован с UI
- [ ] Mobile touch + desktop mouse drag оба работают
- [ ] Progressive loading: частицы появляются ступеньками (видно визуально)

## Стек
Three.js 0.180+, GLSL, Web Audio API, React 19, Zustand