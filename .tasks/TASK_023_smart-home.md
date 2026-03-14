# TASK_023: Умный дом — Yandex IoT + Home Assistant
**Статус:** READY_FOR_ANTIGRAVITY
**Ветка:** feature/task-023-smart-home
**Спецификация:** docs/specs/SPEC_001_voicezettel_2_0_architecture.md — Модуль 8

## Контекст
Управление умным домом голосом через VoiceZettel. Yandex IoT API для прямого контроля, Home Assistant для сложных сценариев. Устройства: камеры, датчики, AC, лампы, розетки, Yandex Station Max (Zigbee gateway).

## Задача
1. Yandex IoT API клиент:
   - OAuth авторизация (scopes: iot:view, iot:control)
   - GET /v1.0/user/info — список устройств
   - POST /v1.0/devices/actions — управление устройствами
   - Камеры: получение HLS stream URL
2. Home Assistant REST API клиент (опциональный middleware):
   - Подключение к localhost:8123
   - Управление entities
   - Создание автоматизаций
3. Intent Parser:
   - Голосовая команда → LLM парсинг → intent + target device
   - "Выключи свет в спальне" → {action: 'turn_off', device: 'light_bedroom'}
   - "Какая температура дома?" → {action: 'query', device: 'sensor_temp'}
4. Устройства:
   - Yandex Station Max (медиа + Zigbee gateway)
   - Zigbee: лампы, розетки, датчики (через Station Max → Yandex IoT)
   - Камеры: HLS stream + screenshot
   - AC: температура + on/off
5. Перезагрузка ПК через умную розетку:
   - off → wait 5sec → on
   - Через Telegram бот (/restart-pc)

## Файлы для создания/изменения
- `src/modules/smart-home/yandex-iot.ts` — Yandex IoT API клиент
- `src/modules/smart-home/home-assistant.ts` — Home Assistant клиент
- `src/modules/smart-home/intent-parser.ts` — парсинг голосовых команд
- `src/modules/smart-home/device-registry.ts` — реестр устройств
- `src/modules/smart-home/scenarios.ts` — пользовательские сценарии
- `src/app/api/smart-home/route.ts` — API: управление устройствами
- `src/app/admin/smart-home/page.tsx` — UI: список устройств, управление

## Acceptance Criteria
- [ ] Yandex IoT: получение списка устройств через API
- [ ] Yandex IoT: управление устройствами (вкл/выкл, температура)
- [ ] Голосовая команда → intent parsing → действие с устройством
- [ ] Камеры: получение HLS URL для просмотра
- [ ] AC: управление температурой через API
- [ ] Перезагрузка ПК: розетка off → 5с → on
- [ ] UI: список устройств со статусами
- [ ] Home Assistant: подключение (опционально, для сложных сценариев)
- [ ] Unit тесты: intent parsing, device actions

## Стек
Yandex IoT API (REST), Home Assistant API, LLM Router, MQTT (опционально)
