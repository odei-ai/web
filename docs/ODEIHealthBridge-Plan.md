# ODEI Health Bridge (iOS → Mac) — Functional Plan

Цель: сделать iOS/iPadOS приложение **ODEI** (bundle `ai.odei.app`), которое собирает метрики HealthKit, детектит ключевые события, отправляет их на Mac для вкладки Body, и (опционально) шлёт локальные уведомления пользователю.

## 1) Основные возможности

- HealthKit сбор и стрим:
  - Пульс (HR), HRV, сон, виталы (SpO2/resp/temp/resting HR), активность (шаги/ступени/стэнды/энергия), тренировки, шум.
  - Обновление через HKObserverQuery + anchored queries + периодический refresh.
  - HTTP API совместимое с `odei.apple.health.*`:
    - `GET /health` — агрегат.
    - `GET /tools?name=odei.apple.health.*` — данные под каждый инструмент.
  - Bearer-токен; порт по умолчанию 8777 (настройки в UI).

- События (push на Mac):
  - Wake/start of day: завершение сна + первые шаги/HR после сна → POST `/events` на Mac.
  - High stress: если HR сильно выше базовой зоны + низкий HRV.
  - Loud noise: по audio exposure > порога.
  - (Опция) Длительный сидячий период / шаги низкие.

- Уведомления на iPhone:
  - Локальные нотификации по событиям (стресс/шум/утро/low readiness). Запрос разрешения при первом запуске; тумблеры в UI.

- Сетевой слой:
  - HTTP pull (уже есть).
  - Добавить `/events` (приём на Mac, отправка с iOS).
  - (Опция) WebSocket push позже, если нужно мгновенное обновление.

## 2) Приложение ODEI (iOS)

- Bundle: `ai.odei.app`, название ODEI.
- Экран настроек:
  - Порт, токен.
  - Вкл/выкл нотификаций.
  - Вкл/выкл отправки событий wake/stress/noise.
  - Статус сервера (запущен/ошибка), последний push.
- Бэкграунд:
  - HKObserverQuery на сон/HR/шаги/шум + фоновая доставка.
  - Обновление кеша и пуш на Mac при событиях.
- Без App Store: сборка через Xcode (профиль разработчика).

## 3) Mac (Body) интеграция

- Источник данных: только HTTP/события (без симуляции, без MCP HealthKit).
- Настройки на стороне Mac:
  - `localStorage.odeiHealthEndpoint` (например, `http://<iphone-ip>:8777`).
  - `localStorage.odeiHealthToken` при наличии токена.
- Обработка `/events`:
  - wake → статус “Day started”, лог в Body.
  - stress/noise → лог + подсветка.
  - (Опция) бэйдж/индикатор последнего события.
- Фолбэк: при ошибке соединения Body показывает OFFLINE/ERROR (без симуляции).

## 4) Безопасность

- Bearer-токен обязателен (включаем по умолчанию после настройки).
- Локальная сеть; можно добавить whitelist IP (опция).
- mTLS — отложено (по желанию).

## 5) Разрешения (iOS)

- HealthKit: share/update descriptions (уже в Info.plist).
- Notifications: UNUserNotificationCenter request.
- Background Modes: background fetch/processing (для HK deliveries).

## 6) Шаги реализации

1. Переименовать проект под ODEI (`ai.odei.app`), обновить Info.plist/entitlements.
2. Добавить UI настроек (порт/токен/нотификации/события/статус).
3. Добавить нотификации и тумблеры.
4. Реализовать события: wake/stress/noise → POST `/events`.
5. На Mac: добавить приём `/events` и отображение в Body; оставить HTTP pull для метрик.
6. Тест: сборка на iPhone, Health разрешения, проверка wake + метрик + логов в Body.
