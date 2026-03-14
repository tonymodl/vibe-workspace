# TASK_007: Панель настроек — голосовой пайплайн + голоса + сущности
**Статус:** READY_FOR_ANTIGRAVITY
**Ветка:** feature/task-007-settings-panel
**Спецификация:** docs/specs/SPEC_001_voicezettel_2_0_architecture.md (Модули 2.5, 4.2, 9)
**Зависит от:** TASK_003, TASK_004

## Контекст
Полная переработка панели настроек из speaker1991. Новый порядок блоков:
Виджеты → Голосовой пайплайн → Системный промпт → Сущности.
Все должно переключаться НА ХОДУ без перезагрузки.

## Задача
1. Блок "Виджеты" (верх): переключатели сущностей (заметки, задачи и т.д.)
2. Блок "Голосовой пайплайн" (НОВЫЙ, под виджетами):
   - Пресеты: 4 карточки (Молния/Оптимальный/Эмоциональный/Локальный) с иконками и описаниями
   - Мозги: dropdown с иконкой провайдера (Groq, OpenAI, Gemini, Ollama)
   - Озвучка: dropdown + кнопка превью голоса (play sample)
   - Голос: 3 кнопки (Мужской / Женский / Клонированный) + надпись "или попроси голосом"
3. Блок "Системный промпт": textarea с базовым промптом + список директив пользователя (с кнопкой удалить)
4. Блок "Сущности": список типов (заметки, идеи, факты, персоны, задачи) + кнопка "Добавить сущность" + toggle вкл/выкл для каждой
5. Все изменения применяются мгновенно через Zustand store → VoicePipelineManager

## Файлы
- `src/components/Settings/SettingsPanel.tsx`
- `src/components/Settings/WidgetsBlock.tsx`
- `src/components/Settings/VoicePipelineBlock.tsx`
- `src/components/Settings/SystemPromptBlock.tsx`
- `src/components/Settings/EntitiesBlock.tsx`
- `src/components/Settings/PresetCard.tsx`
- `src/components/Settings/VoiceSelector.tsx`
- `src/stores/settingsStore.ts`

## Acceptance Criteria
- [ ] Порядок блоков: Виджеты → Голосовой пайплайн → Промпт → Сущности
- [ ] Клик на пресет меняет мозги+озвучку без перезагрузки
- [ ] Dropdown мозгов показывает иконки провайдеров и меняет LLM на лету
- [ ] Dropdown озвучки + превью голоса
- [ ] Кнопки голоса (М/Ж/К) переключают голос
- [ ] Директивы пользователя отображаются с кнопкой удаления
- [ ] Сущности: toggle вкл/выкл работает
- [ ] Кнопка "Добавить сущность" открывает форму
- [ ] Дизайн красивый и удобный, в стиле dark theme

## Стек
shadcn/ui (Select, Switch, Tabs, Card, Textarea), Zustand, Tailwind CSS