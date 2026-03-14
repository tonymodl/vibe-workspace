# TASK_003: UI — Сфера частиц + основной layout
**Статус:** READY_FOR_ANTIGRAVITY
**Ветка:** feature/task-003-ui-particle-sphere
**Спецификация:** docs/specs/SPEC_001_voicezettel_2_0_architecture.md
**Зависит от:** TASK_001
**Референс UI:** https://github.com/speaker1991/voicezettel (компоненты сферы)

## Контекст
Берём готовый интерфейс из speaker1991/voicezettel — сферу из частиц, чат, кнопки.
Но оптимизируем загрузку: CSS-placeholder пока Three.js грузится, начинаем с 500 частиц → до 3000.

## Задача
1. Перенести компонент сферы частиц из speaker1991 в новый проект:
   - Three.js сцена с шариком из частиц
   - Анимации: idle (медленное вращение), listening (пульсация синим), speaking (активная анимация)
2. Оптимизация загрузки:
   - CSS-анимированный placeholder (пульсирующий круг) пока Three.js инициализируется
   - Начать с 500 частиц, плавно добавить до 3000 за 2 секунды
   - Web Worker для вычисления позиций частиц
3. Layout главной страницы:
   - Центр: сфера частиц
   - Снизу: кнопка микрофона (push-to-talk) + переключатель режимов
   - Справа: панель чата (сворачиваемая)
   - Слева: панель настроек (сворачиваемая)
4. Тёмная тема по умолчанию (dark mode)
5. Респонсивность: мобильные и десктоп

## Файлы для создания/изменения
- `src/components/ParticleSphere/` — компонент сферы
- `src/components/ParticleSphere/ParticleSphere.tsx`
- `src/components/ParticleSphere/shaders/` — vertex/fragment шейдеры
- `src/components/ParticleSphere/useParticleAnimation.ts`
- `src/components/Layout/MainLayout.tsx`
- `src/components/Chat/ChatPanel.tsx`
- `src/components/MicButton/MicButton.tsx`
- `src/app/page.tsx` — главная страница

## Acceptance Criteria
- [ ] Сфера частиц рендерится при загрузке страницы
- [ ] CSS-placeholder виден пока Three.js грузится (до 1 сек)
- [ ] Частицы начинаются с 500, плавно добавляются до 3000
- [ ] 3 состояния анимации работают: idle, listening, speaking
- [ ] Кнопка микрофона есть и кликабельна (пока без функционала)
- [ ] Панель чата открывается/закрывается
- [ ] Панель настроек открывается/закрывается
- [ ] Тёмная тема применена
- [ ] На мобильных — панели занимают полный экран
- [ ] FPS сферы >= 30 на среднем устройстве

## Стек
Three.js, @react-three/fiber, @react-three/drei, GLSL шейдеры, Tailwind CSS, shadcn/ui