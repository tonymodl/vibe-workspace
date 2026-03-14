# TASK_009: Панель настроек — пресеты, провайдеры, голос
**Статус:** READY_FOR_ANTIGRAVITY
**Ветка:** feature/task-009-settings-panel
**Спецификация:** docs/specs/SPEC_001_voicezettel_2_0_architecture.md — Модуль 2.4

## Контекст
Панель настроек — центр управления голосовым пайплайном. Пользователь свободно выбирает приоритеты моделей, пресеты, голос ассистента. Имя ассистента меняется в любой момент.

## Задача
1. Layout панели (порядок блоков сверху вниз):
   - Виджеты (toggle компоненты вкл/выкл)
   - Голосовой пайплайн (STT / LLM / TTS)
   - Системный промпт (текстовое поле + история директив)
   - Сущности (типы заметок с toggle)
2. Пресеты голосового пайплайна:
   - ⚡ Молния (Groq + Cartesia — максимальная скорость)
   - ⚖️ Оптимальный (GPT-4o + ElevenLabs — баланс)
   - 🎭 Эмоциональный (GPT-4o + ElevenLabs с эмоциями)
   - 💻 Локальный (Ollama + Qwen3-TTS — полностью локально)
3. Выбор провайдеров:
   - LLM: drag-and-drop список приоритетов с иконками
   - TTS: выпадающий список + кнопка превью (прослушать голос)
   - STT: переключатель Deepgram / Whisper
4. Настройки голоса:
   - Переключатель: Мужской / Женский / Клонированный
   - Кнопка "Попросить голосом" (запись сэмпла для клонирования)
5. Имя ассистента: текстовое поле, изменяемое в любой момент
6. Сохранение настроек: Zustand persist + SQLite users.settings_json

## Файлы для создания/изменения
- `src/components/settings/SettingsPanel.tsx` — главный компонент
- `src/components/settings/PresetSelector.tsx` — выбор пресета
- `src/components/settings/ProviderPriority.tsx` — drag-and-drop приоритеты
- `src/components/settings/VoiceSelector.tsx` — выбор голоса + превью
- `src/components/settings/SystemPrompt.tsx` — системный промпт + директивы
- `src/components/settings/EntityToggle.tsx` — toggle сущностей
- `src/stores/settings-store.ts` — Zustand: все настройки
- `src/app/api/settings/route.ts` — сохранение/загрузка настроек

## Acceptance Criteria
- [ ] Все 4 блока отображаются в правильном порядке
- [ ] Пресеты переключают набор провайдеров одним кликом
- [ ] Drag-and-drop для LLM приоритетов работает (desktop + mobile)
- [ ] Превью TTS голоса: клик → озвучка тестовой фразы
- [ ] Имя ассистента сохраняется и используется как триггер
- [ ] Настройки persist между сессиями (Zustand + SQLite)
- [ ] Переключение провайдера НЕ перезагружает страницу
- [ ] Системный промпт: текстовое поле + список активных директив

## Стек
React 19, Tailwind CSS, shadcn/ui, dnd-kit (drag-and-drop), Zustand