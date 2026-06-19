# METAMODEL.md

> **Каноничный документ.** Заменяет 12-файлый архив (`01_module_system_specification` ... `12_platform_communication_model`) и `review_solution_01`. Противоречия с ними разрешаются в пользу этого документа.

---

## TL;DR

4 JSON-сущности: `registry.json`, `worker.json`, `contract.json`, `*.schema.json`. Registry индексирует **артефакты**, не воркеры. Subject НЕ хранится в JSON — вычисляется через Contracts (Go/Elixir) на основе `domain/entity/action` из `contract_mappings` таблицы БД. JSON-файлы = source of truth, БД = applied state. Sync script применяет JSON → БД с preview. Worker = единица развёртывания, самодостаточен, переносим в другой проект.

---

## 1. Четыре сущности

| Сущность | Файл | Назначение |
|----------|------|------------|
| **Registry** | `registry.json` | Каталог артефактов платформы |
| **Worker** | `workers/<name>/worker.json` | Описание воркера: что слушает, что публикует |
| **Contract** | `workers/<name>/contracts/<id>.json` (domain) или `internal/contracts/tables/<table>.json` (table descriptor) | Описание сообщения: type, domain, entity, action |
| **Schema** | `*.schema.json` | JSON Schema для payload сообщения |

### Главный принцип

> Каждый новый файл должен отвечать на вопрос: **«Какую новую информацию он хранит, которую нельзя положить в существующий?»** Если ответ — никакую, файл не нужен.

Никаких `module.json`, `action.json`, `event.json`, `command.json`, `signal.json`, `hook.json`, `listener.json`. Action — часть worker.json (пока 1:1 с worker).

---

## 2. registry.json

Registry — каталог **артефактов**, не воркеров. Future-proof: завтра появятся `service.json`, `projection.json`, `scheduler.json` — Registry о них ничего не знает, просто индексирует.

### Структура

```json
{
  "artifacts": [
    {"type": "worker",   "path": "./workers/notes-vectorize/worker.json"},
    {"type": "worker",   "path": "./workers/chat-message/worker.json"},
    {"type": "table",    "path": "./internal/contracts/tables/chat_messages.json"},
    {"type": "table",    "path": "./internal/contracts/tables/notes.json"},
    {"type": "table",    "path": "./internal/contracts/tables/tasks.json"}
  ]
}
```

### Логика работы

1. Registry читает `registry.json` при старте (или при сборке)
2. Для каждого артефакта читает JSON по `path`
3. Для `worker` артефакта — парсит `worker.json`, индексирует воркер
4. Для `table` артефакта — парсит table descriptor, генерирует 3 CDC contracts (created/updated/deleted)
5. Строит глобальный индекс:
   - `workers: {id → worker}`
   - `contracts: {contract_id → contract}`
   - `dependencies: {(contract_id) → [worker_id, ...]}`
   - `modules: {module_name → [worker_id, ...]}` (виртуальная группировка из поля `module` в worker.json)

### Валидации при сборке

1. Каждый `contract_id` в `worker.json` `subscribes`/`publishes` → есть в индексе contracts
2. Каждый domain contract JSON → owner worker существует и имеет этот `contract_id` в `publishes`
3. Каждый table descriptor JSON → таблица реально существует в `pg_class`
4. Каждый `table_name` с триггером `attach_activity_log` → есть в registry как `type: "table"`
5. Уникальность `contract_id` по всем JSON

---

## 3. worker.json

### Поля

| Поле | Тип | Обязательное | Описание |
|------|-----|--------------|----------|
| `id` | string | ✅ | Уникальный ID воркера (например, `notes-vectorize`) |
| `module` | string | ✅ | Логическая группировка (`notes`, `chat`, `ai`). Не файл — поле |
| `name` | string | ✅ | Человекочитаемое имя (`Vectorize Note`) |
| `description` | string | ✅ | Что делает воркер |
| `subscribes` | string[] | ✅ | Логические ID контрактов, на которые подписан (`["notes.created", "notes.updated"]`) |
| `publishes` | string[] | ✅ | Логические ID контрактов, которые публикует (`["notes.vectorized"]`) |
| `tags` | string[] | ❌ | Метки для UI/поиска (`["ai", "vector"]`) |
| `version` | string | ❌ | Семвер воркера (`1.0.0`) |

### Пример

```json
{
  "id": "notes-vectorize",
  "module": "notes",
  "name": "Vectorize Note",
  "description": "Creates embeddings for notes after creation or update",
  "subscribes": [
    "notes.created",
    "notes.updated"
  ],
  "publishes": [
    "notes.vectorized"
  ],
  "tags": ["ai", "vector"],
  "version": "1.0.0"
}
```

### Что НЕ входит в worker.json

- ❌ `contracts: [...]` — пути к контрактам. Registry сам находит по `id` из `subscribes`/`publishes`
- ❌ `action: {...}` — отдельное действие. Action — это контракт, описывается в `contract.json`
- ❌ `chain: [...]` — цепочки процессов. Это BPMN, мы от этого уходим
- ❌ `process: {...}` — описание процесса. Process = сам воркер
- ❌ Subject строки — только логические ID

### Module = логическая группировка

`module` — поле, не отдельный файл. Registry по нему строит группы:

```
Notes
├── notes-create
├── notes-vectorize
└── notes-search

Chat
├── chat-message
└── chat-room

AI
└── ai-summary
```

Структура репо **плоская**: `workers/<name>/`. Никаких `modules/notes/` директорий.

### Worker = единица развёртывания

Папка `workers/<name>/` самодостаточна:
- `worker.json` — описание
- `processor.go` — код
- `contracts/` — domain contracts, которые воркер публикует
- `*.schema.json` — JSON Schemas для payload
- `tests/` — тесты
- `README.md` (опционально) — документация

Если скопировать `workers/notes-vectorize/` в другой проект — должно заработать (после `sync-contracts`).

---

## 4. contract.json

### Поля

| Поле | Тип | Обязательное | Описание |
|------|-----|--------------|----------|
| `contract_id` | string | ✅ | Уникальный ID (`notes.created`, `chat.message.created`, `notes.vectorized`) |
| `name` | string | ✅ | Человекочитаемое имя (`Note Created`) |
| `description` | string | ✅ | Что означает событие |
| `type` | string | ✅ | `event` \| `signal` \| `command` |
| `domain` | string | ✅ | Логический модуль (`notes`, `chat`) |
| `entity` | string | ✅ | Сущность (`note`, `message`) |
| `action` | string | ✅ | `created`/`updated`/`deleted` (CDC) или бизнес-глагол (`vectorized`, `completed`) |
| `table` | string\|null | ❌ | Имя таблицы для CDC контрактов (`notes`, `chat_messages`). NULL для domain events |
| `source` | string | ✅ | `cdc` \| `domain` \| `command` — кто публикует |
| `ui_visible` | boolean | ❌ | Показывать в UI (по умолчанию `false`) |
| `schema` | string | ❌ | Путь к JSON Schema относительно contract.json |
| `version` | int | ❌ | Версия контракта (по умолчанию 1) |

### Subject НЕ хранится

Subject вычисляется через `Contracts.Subject(type, domain, entity, action)`:

```
contract_id = "notes.vectorized"
type        = "signal"
domain      = "notes"
entity      = "note"
action      = "vectorized"

→ subject = "signal.v1.notes.note.vectorized"
```

Если меняется naming convention — меняется одна функция в Contracts, ни один JSON не трогается.

### Два класса контрактов

#### Domain contract (в папке воркера)

`workers/notes-vectorize/contracts/notes.vectorized.json`:
```json
{
  "contract_id": "notes.vectorized",
  "name": "Note Vectorized",
  "description": "Embedding completed, vector is ready in Qdrant",
  "type": "signal",
  "domain": "notes",
  "entity": "note",
  "action": "vectorized",
  "table": null,
  "source": "domain",
  "ui_visible": true,
  "schema": "notes.vectorized.schema.json",
  "version": 1
}
```

Воркер, у которого этот контракт в `publishes`, — его owner. Registry валидирует ownership.

#### Table descriptor (для CDC)

`internal/contracts/tables/chat_messages.json`:
```json
{
  "table": "chat_messages",
  "domain": "chat",
  "entity": "message",
  "description": "Chat messages",
  "ui_visible": true
}
```

Из этого одного файла sync script генерирует 3 CDC контракта:
- `chat.message.created` (event, cdc)
- `chat.message.updated` (signal, cdc)
- `chat.message.deleted` (signal, cdc)

---

## 5. Schema files (JSON Schema)

`workers/notes-vectorize/contracts/notes.vectorized.schema.json`:
```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "NoteVectorized",
  "type": "object",
  "properties": {
    "entity_id":  {"type": "string", "format": "uuid"},
    "tenant_id":  {"type": "string", "format": "uuid"},
    "vector_dim": {"type": "integer", "minimum": 1}
  },
  "required": ["entity_id", "tenant_id"]
}
```

Standard JSON Schema. Используется для валидации payload при публикации/потреблении.

---

## 6. `contract_mappings` таблица

**Applied state, не source of truth.** Заполняется sync script'ом из JSON-файлов.

### Схема

```sql
CREATE TABLE contract_mappings (
    contract_id  TEXT PRIMARY KEY,
    name         TEXT NOT NULL,
    description  TEXT,
    type         TEXT NOT NULL,              -- 'event' | 'signal' | 'command'
    domain       TEXT NOT NULL,
    entity       TEXT NOT NULL,
    action       TEXT NOT NULL,
    table_name   TEXT,                       -- NULL для domain events
    source       TEXT NOT NULL,              -- 'cdc' | 'domain' | 'command'
    ui_visible   BOOLEAN NOT NULL DEFAULT FALSE,
    schema_path  TEXT,                       -- путь к .schema.json относительно репо
    version      INT NOT NULL DEFAULT 1,
    source_file  TEXT NOT NULL,              -- путь к исходному JSON (для трейсинга)
    applied_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_contract_mappings_table ON contract_mappings (table_name) WHERE table_name IS NOT NULL;
CREATE INDEX idx_contract_mappings_ui ON contract_mappings (ui_visible) WHERE ui_visible = TRUE;
```

### Кто что читает

| Компонент | Что читает | Зачем |
|-----------|------------|-------|
| **Разработчик** | JSON файлы в `workers/*/contracts/` и `internal/contracts/tables/` | Понять, что воркер публикует, без похода в БД |
| **AI-агент** | JSON файлы | Та же цель + генерация новых воркеров |
| **Sync script** | JSON файлы → БД | Применить изменения |
| **Registry (build time)** | БД `contract_mappings` | Построить индекс для UI/поиска/валидации |
| **air_activity_service (runtime)** | БД `contract_mappings` через ETS-кэш | Lookup `(domain, entity)` по `table_name` для построения subject |
| **Go-воркер (runtime)** | БД `contract_mappings` через пул | Lookup subject для публикации domain event |
| **UI (Next.js)** | БД `contract_mappings WHERE ui_visible=true` | Показать события в feed |

---

## 7. Sync script

### Рабочий процесс

```
[запуск: make sync-contracts или CI step]
        │
        ▼
1. Сканировать workers/*/contracts/*.json         → domain contracts
2. Сканировать internal/contracts/tables/*.json   → сгенерировать CDC contracts
3. Прочитать текущее состояние из contract_mappings
        │
        ▼
4. Diff: что добавилось / изменилось / удалилось
        │
        ▼
5. Вывод (preview):
   + notes.vectorized (new, source=domain)
     Subject: signals.v1.notes.note.vectorized
     File: workers/notes-vectorize/contracts/notes.vectorized.json
   
   + chat.message.created (new, source=cdc, from chat_messages.json)
     Subject: events.v1.chat.message.created
   
   ~ notes.reindexed (changed: ui_visible false→true)
   
   - notes.old_action (removed from JSON, was in DB)
        │
        ▼
6. [интерактивный режим] Подтверждение: Apply? [y/N]
   [CI режим] Apply автоматически, fail если есть destructive changes
        │
        ▼
7. INSERT/UPDATE/DELETE в contract_mappings
        │
        ▼
8. air_activity_service подхватывает через LISTEN/NOTIFY (или polling)
```

### Portability flow

```
[Копируешь workers/notes-vectorize/ в другой проект]
        │
        ▼
[Запускаешь sync-contracts]
        │
        ▼
[Script видит новый JSON, не находит в БД]
        │
        ▼
[Выводит:]
  Found new contracts:
    + notes.vectorized (signal, domain)
      Source: workers/notes-vectorize/contracts/notes.vectorized.json
      Subject will be: signals.v1.notes.note.vectorized
      
  Initialize? [y/N/p(review)]
        │
        ▼
[При подтверждении — INSERT в contract_mappings]
[air_activity_service подхватывает]
```

### Sync script — реализация

Может быть написан на Go (как часть `cmd/tools/sync-contracts/main.go`) или на bash+jq. Рекомендуется Go, т.к.:
- Переиспользует `internal/contracts` для построения subject в preview
- Может валидировать JSON против схемы
- Один бинарник, нет зависимостей

---

## 8. UI: Option C

### `ui_visible` в contract.json

Быстрый фильтр. Если `true` — событие показывается в UI.

```json
{
  "contract_id": "notes.vectorized",
  "ui_visible": true,
  ...
}
```

### `ui.json` (опционально)

Детали отображения. Только когда `ui_visible: true`.

`workers/notes-vectorize/contracts/notes.vectorized.ui.json`:
```json
{
  "icon": "check-circle",
  "category": "AI",
  "template": "Note {entity_id} was vectorized",
  "color": "#10b981",
  "priority": "info",
  "actions": ["view_note", "reindex"]
}
```

### Логика

- `ui_visible: false` (или отсутствует) → событие НЕ показывается в UI, `ui.json` не нужен
- `ui_visible: true` → событие показывается, Registry требует `ui.json` рядом (валидирует при сборке)
- UI-слой Next.js тянет из Registry только события с `ui_visible: true` + их `ui.json`

### Почему так

- Быстрый фильтр по булеву флагу (не надо парсить весь `ui.json` для каждого события)
- Чистый contract.json для не-UI событий
- Расширяемость: в `ui.json` можно добавлять любые поля (hooks, templates, permissions) не трогая contract
- Registry валидирует консистентность

---

## 9. Subject construction pipeline

### При публикации (воркер или activity_service)

```
1. Имеем contract_id ("notes.vectorized")
   ↓
2. Lookup в contract_mappings по contract_id
   → {type: "signal", domain: "notes", entity: "note", action: "vectorized"}
   ↓
3. Вызываем contracts.Subject(type, domain, entity, action)
   → "signal.v1.notes.note.vectorized"
   ↓
4. Публикуем в NATS
```

### При потреблении (воркер)

```
1. В worker.json указано "subscribes": ["notes.created"]
   ↓
2. При старте воркера — lookup в contract_mappings
   → {type: "event", domain: "notes", entity: "note", action: "created"}
   ↓
3. Вызываем contracts.EventSubject("notes", "note", "created")
   → "event.v1.notes.note.created"
   ↓
4. NATS Subscribe с этим subject
```

### Activity_service при публикации из WAL

```
1. Из WAL получаем entity_type="chat_messages", action="created"
   ↓
2. Lookup в contract_mappings по table_name="chat_messages"
   → {domain: "chat", entity: "message"}
   ↓
3. message_type = Contracts.message_type("created") = "event"
   ↓
4. subject = Contracts.Subject("event", "chat", "message", "created")
   → "event.v1.chat.message.created"
   ↓
5. NatsPublisher.publish(subject, payload)
```

---

## 10. Module = логическая группировка

### Не файл, поле

`module` в `worker.json` — это строка. Registry по ней строит группы.

```json
// workers/notes-vectorize/worker.json
{
  "id": "notes-vectorize",
  "module": "notes",
  ...
}

// workers/notes-search/worker.json
{
  "id": "notes-search",
  "module": "notes",
  ...
}
```

Registry:
```
Modules:
  notes:
    - notes-vectorize
    - notes-search
    - notes-create
  chat:
    - chat-message
    - chat-room
  ai:
    - ai-summary
```

### Валидация

Registry при сборке проверяет:
- Опечатки (`note` vs `notes`) — если значение `module` встречается только у одного воркера, возможно опечатка (warning)
- Консенсус: если `module: "notes"` есть у 3 воркеров, а `module: "note"` у одного — flag на ревью

### Структура репо

Плоская, без `modules/` директории:

```
workers/
├── notes-create/
├── notes-vectorize/
├── notes-search/
├── chat-message/
├── chat-room/
├── ai-summary/
└── ...
```

Группировка — в данных, не в папках.

---

## 11. Process Registry — виртуальный

Process Registry **не отдельный файл**. Строится из `publishes`/`subscribes` полей worker.json.

Registry индекс:
```
Processes:
  - id: vectorize-note
    worker: notes-vectorize
    module: notes
    subscribes: [notes.created, notes.updated]
    publishes:  [notes.vectorized]
    
  - id: chat-message
    worker: chat-message
    module: chat
    subscribes: [chat.message.send (command)]
    publishes:  []
```

Для AI-агента: можно спросить «какие процессы слушают `notes.created`?» — Registry ответит из индекса.

---

## 12. Финальная структура репозитория

```
air-core-platform/
├── cmd/
│   ├── services/
│   │   └── event-router/
│   │       └── main.go
│   ├── workers/
│   │   ├── notes-vectorize/
│   │   │   └── main.go
│   │   ├── chat-message/
│   │   │   └── main.go
│   │   └── ...
│   └── tools/
│       └── sync-contracts/
│           └── main.go              ← sync script
│
├── internal/
│   ├── config/
│   ├── logger/                      ← zerolog
│   ├── db/
│   ├── natsconn/
│   ├── contracts/
│   │   ├── contracts.go             ← Subject(), EventSubject(), CacheKey(), ...
│   │   └── tables/                  ← table descriptors
│   │       ├── chat_messages.json
│   │       ├── notes.json
│   │       └── tasks.json
│   ├── cache/
│   ├── worker/                      ← базовый скелет
│   └── event/
│
├── services/                        ← HTTP-сервисы монорепо (внешний транспорт)
│   └── event-router/
│
├── workers/                         ← NATS-воркеры
│   ├── notes-vectorize/
│   │   ├── worker.json
│   │   ├── processor.go
│   │   └── contracts/
│   │       ├── notes.vectorized.json
│   │       ├── notes.vectorized.schema.json
│   │       └── notes.vectorized.ui.json
│   ├── chat-message/
│   │   ├── worker.json
│   │   ├── processor.go
│   │   └── contracts/
│   │       └── chat.message.sent.json
│   └── ...
│
├── migrations/
│   ├── 000_foundation.sql
│   ├── 001_tenants.sql
│   ├── 002_profiles.sql
│   ├── ...
│   ├── 008_activity_logs.sql
│   ├── 009_activity_trigger.sql
│   ├── 010_logical_replication.sql
│   ├── 011_contract_mappings.sql    ← таблица applied state
│   └── 012_entity_links.sql
│
├── registry.json                    ← каталог артефактов
├── go.mod
├── go.sum
├── Makefile                         ← make sync-contracts, make run-worker NAME=...
└── README.md
```

---

## Cross-references

- **Subject formula, message types, CDC vs Domain** → `COMMUNICATION_MODEL.md`
- **Полная архитектура, DB-слой, activity_logs** → `ARCHITECTURE.md`
- **Сквозной пример добавления воркера** → `LIFECYCLE_ADDING_WORKER.md`
- **Contracts API (Go + Elixir)** → `CONTRACTS_REFERENCE.md`
