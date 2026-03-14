# TASK_008: Чат-интерфейс с streaming и цветными облачками
**Статус:** READY_FOR_ANTIGRAVITY
**Ветка:** feature/task-008-chat-interface
**Спецификация:** docs/specs/SPEC_001_voicezettel_2_0_architecture.md — Модуль 2.3

## Контекст
Чат-интерфейс показывает диалог с ИИ. В режиме петлицы — сообщения от разных спикеров с цветными облачками. Streaming ответов LLM token by token. Обновление имён спикеров задним числом.

## Задача
1. Компонент чата (на основе speaker1991/voicezettel):
   - Список сообщений с автоскроллом
   - Bubbles: пользователь (справа), ИИ (слева), спикеры петлицы (слева, цветные)
   - Markdown рендеринг в сообщениях
   - Timestamps
2. Streaming отображение:
   - Ответ LLM появляется token by token
   - Индикатор "печатает..." с анимацией
   - Cursor (мигающий) в конце текста при streaming
3. Режим петлицы:
   - Каждый спикер → уникальный цвет (из hash имени/ID)
   - Облачки с именем спикера сверху
   - "Speaker N" для неизвестных → обновление на реальное имя когда узнано
4. Zustand store для сообщений:
   - Массив messages с типами: user, assistant, speaker
   - Метод updateSpeakerName(oldName, newName) — обновить все сообщения

## Файлы для создания/изменения
- `src/components/chat/ChatContainer.tsx` — контейнер чата
- `src/components/chat/MessageBubble.tsx` — облачко сообщения
- `src/components/chat/StreamingIndicator.tsx` — индикатор печати
- `src/components/chat/SpeakerBadge.tsx` — бейдж спикера (имя + цвет)
- `src/stores/chat-store.ts` — Zustand: messages, addMessage, updateSpeakerName
- `src/lib/utils/speaker-colors.ts` — генерация цвета из имени/ID

## Acceptance Criteria
- [ ] Сообщения отображаются с правильным позиционированием (user справа, ИИ слева)
- [ ] Streaming: текст появляется token by token (видно побуквенное появление)
- [ ] Markdown рендерится в сообщениях (жирный, курсив, ссылки, код)
- [ ] В режиме петлицы: разные спикеры имеют разные цвета облачков
- [ ] updateSpeakerName обновляет ВСЕ предыдущие сообщения Speaker N → реальное имя
- [ ] Автоскролл к последнему сообщению
- [ ] Mobile-friendly: корректно выглядит на телефоне
- [ ] Индикатор "печатает" показывается во время streaming

## Стек
React 19, Tailwind CSS, Zustand, react-markdown