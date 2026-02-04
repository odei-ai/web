# Архитектура агентов ODEI (русский обзор)

Документ описывает четыре ключевых агента конституционного стека ODEI, их ответственность, доступные инструменты и порядок взаимодействия. Используется исключительно для операционного понимания; все записи в граф выполняются через MCP-инструменты, перечисленные ниже.

## Общие принципы

- Каждый агент работает в своей рабочей папке (`/ODEI/{Discuss|Decisions|Execute|Mind}`) и запускает собственный MCP-сервер.
- Для чтения/аналитики используются stage-специфичные сервера (`*-neo4j`). Для канонических записей задействуется центральный сервер `odei-neo4j`.
- Все пишущие инструменты требуют корректного `provenance` и, при необходимости, `idempotencyKey`.
- Агенты передают результаты друг другу по конвейеру: **Discuss → Decisions → Execute → Mind → (обратно в Discuss)**.

---

## Discuss — Конституционный страж и партнёр

**Основная задача:** формирование и обслуживание «верхних» слоёв графа (ценности, принципы, визии, лестница целей), защита от перегрузки и эмоциональных перекосов.

### Зона ответственности

- Слои Foundation (Values, Principles, Guardrails), Direction (Vision), Goals (Life → Day).
- Проверка и поддержание связей между слоями: `SUPPORTED_BY`, `ENFORCED_BY`, `ALIGNS_WITH`, `SERVES`, `SUPPORTED_BY` (Goal→Goal), а также человеческие связи (`HOLDS_VALUE`, `FOLLOWS_PRINCIPLE`, `RESPECTS_GUARDRAIL`, `OPERATES_IN`).
- Управление рисками «evening critic», sopliv-целями и перегрузкой (>85% расписания).
- Подготовка структурированного handoff для Decisions.

### Ключевые инструменты

- **Discovery (discuss-neo4j):** `discuss_semantic_search`, `discuss_hybrid_search`, `discuss_goal_ladder`, `discuss_workload_assess`, весь блок `discuss_graph_*`, `discuss_pattern_detection`.
- **Writes (odei-neo4j):** `value/principle/guardrail/vision/goal.create|update.v1`, `relationship.create.v1`, а также чтения `foundation.list`, `vision.list`, `graph.getAll`, `schema.inventory`.

### Хэнд-оффы

- Передаёт Decisions информацию о созданных/изменённых узлах, связях, загрузке и наблюдениях.
- В случае нерешённых противоречий может инициировать повторное обсуждение утром (метки `created_evening`, `review_morning`).

---

## Decisions — Стратегический ROI-движок

**Основная задача:** превращать конституционные намерения в стратегию (Objectives, Key Results, Initiatives) и фиксировать решения с 5D ROI.

### Зона ответственности

- Слой Strategy и Decision: создание/обновление Objectives, Key Results, Initiatives, Decision, Evidence.
- Расчёт 5D ROI (финансы, время, выравнивание, риск, энергия) и поддержание портфеля инициатив.
- Планирование по циклам (неделя/месяц/квартал/год), контроль загрузки и активных инициатив (≤5 одновременно).

### Ключевые инструменты

- **Discovery (decisions-neo4j):** `decisions_semantic_search`, `decisions_hybrid_search`, `decisions_goal_ladder`, `decisions_workload_assess`, блок `decisions_graph_*`, `decisions_pattern_detection`, `decisions_decisions.backlog/metrics`.
- **Writes (odei-neo4j):** `objective/key_result/initiative/decision/evidence.create|update.v1`, `relationship.create.v1` (`HAS_KEY_RESULT`, `SERVES`, `ADVANCES`, `SUPPORTED_BY`, `HAS_PROJECT`), чтения `strategy.list`, `execution.list`, `vision.list`, `graph.getAll`, `schema.inventory`.

### Хэнд-оффы

- Передаёт Execute выбранные решения и требования к выполнению (инициативы, KPI, бюджет/ресурсы).
- Обратно получает фидбек по отклонениям от плана (через Execute/ Mind) для пересчёта ROI или корректировок.

---

## Execute — Командир исполнения

**Основная задача:** декомпозировать решения в проекты/задачи, управлять расписанием и статусами исполнения.

### Зона ответственности

- Слои Tactics и Execution: Projects, Areas, Systems, Processes, Tasks, TimeBlocks, WorkSessions, Actions.
- Поддержание связей `HAS_PROJECT`, `HAS_TASK`, `SERVES`, `SCHEDULED_AS`, `LOGGED_AS`.
- Контроль загрузки, соблюдение guardrail-ограничений и своевременная эскалация блокеров.

### Ключевые инструменты

- **Discovery (execute-neo4j):** `execute_semantic_search`, `execute_hybrid_search`, `execute_goal_ladder`, `execute_workload_assess`, блок `execute_graph_*`, `odei.neo4j.decisions.backlog.v1`, `odei.neo4j.execute.tasksToday/workSessions`.
- **Writes (odei-neo4j):** `project/area/system/process/task/time_block/work_session/action.create|update.v1`, `relationship.create.v1`, чтения `tactics.list`, `execution.list`, `graph.getAll`, `schema.inventory`.

### Хэнд-оффы

- Передаёт Decisions/Mind отчёты о статусе инициатив, выполнении задач, рисках и отклонениях от расписания.
- Запрашивает Decisions при необходимости пересчёта приоритета/ROI или таргетов.

---

## Mind — Аналитик обратной связи

**Основная задача:** собирать метрики, наблюдения, инсайты и паттерны, закрывая цикл обучения и улучшений.

### Зона ответственности

- Слои Track и Mind: Metric, Observation, Insight, Pattern, Evidence, Source, Note.
- Формирование связей `TRACKS`, `OBSERVES`, `INFORMS`, `APPLIED_TO`, `TRIGGERS`, `SUPPORTED_BY`.
- Подготовка отчётов о соответствии фактов прогнозам, рекомендации по корректировке конституции/стратегии.

### Ключевые инструменты

- **Discovery (mind-neo4j):** `mind_semantic_search`, `mind_hybrid_search`, `mind_goal_ladder`, `mind_workload_assess`, блок `mind_graph_*`, `mind_recentInsights`, `mind_patternStats`.
- **Writes (odei-neo4j):** `metric/observation/insight/pattern/evidence/source/note.create|update.v1`, `relationship.create.v1`, чтения `track.list`, `mind.list`, `vision.list`, `graph.getAll`, `schema.inventory`.

### Хэнд-оффы

- Возвращает Discuss рекомендации по изменению конституции (веса ценностей, guardrails).
- Передаёт Decisions данные об эффективности стратегий, предлагает пересмотр ROI-весов.
- Информирует Execute о паттернах продуктивности/выгорания и рекомендует корректировки расписания.

---

## Канал передачи данных

1. **Discuss** создаёт/обновляет конституционные узлы → делает handoff с ID и контекстом.
2. **Decisions** строит OKR, принимает решения, фиксирует ROI → handoff в Execute.
3. **Execute** декомпозирует и исполняет → формирует отчёты, возвращает статус в Decisions и данные в Mind.
4. **Mind** анализирует, фиксирует инсайты → отправляет рекомендации обратно в Discuss/Decisions/Execute.

Таким образом, каждый агент работает автономно в своей зоне, но совместно поддерживает непрерывный цикл обсуждение → решение → исполнение → обучение. Мужчины (Discuss/Decisions) обеспечивают направление, женщины (Execute/Mind) ведут реализацию и рефлексию. Любые записи в Neo4j проходят исключительно через авторизованные MCP-инструменты.
