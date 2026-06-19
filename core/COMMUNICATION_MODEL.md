# COMMUNICATION_MODEL.md

> **Каноничный документ.** Заменяет `event_core.md`, `overview.md`, раздел «Message Architecture» из 12-файлого архива. Противоречия с ними разрешаются в пользу этого документа.

---

## TL;DR

Три типа сообщений: **Command** (намерение), **Event** (факт создания, из WAL), **Signal** (изменение состояния, из WAL или от воркера). Subject: `[type].v1.[domain].[entity].[action]` (ед.ч.). Воркеры не публикуют CDC-события — только `air_activity_service` из WAL. Воркеры могут публиковать domain events (signal) для факта завершения long-running операций. JSON везде. `tenant_id` никогда в subject.

---

## 1. Три типа сообщений

| Тип | Смысл | Кто публикует | Кто слушает |
|-----|-------|---------------|-------------|
| **Command** | «Сделай это» — намерение выполнить действие | Event Router (по HTTP от actor'а) | Один воркер (queue) |
| **Event** | «Это создано» — факт появления нового объекта с данными | `air_activity_service` (автоматически из WAL на INSERT) | Любое количество сервисов |
| **Signal** | «Тут изменилось» — лёгкий триггер без бизнес-данных | `air_activity_service` (из WAL на UPDATE/DELETE) **ИЛИ** воркер (domain event для факта завершения) | Frontend + Push workers + другие воркеры |

### Жёсткое правило WAL → message_type

```
INSERT  → action="created"  → message_type="event"
UPDATE  → action="updated"  → message_type="signal"
DELETE  → action="deleted"  → message_type="signal"
```

Нормализация происходит **один раз** — на входе в `air_activity_service` через `Contracts.normalize_action/1`. `:insert → "created"`, `:update → "updated"`, `:delete → "deleted"`.

### Кто что НЕ публикует

- ❌ Воркеры не публикуют CDC-события (event для INSERT, signal для UPDATE/DELETE). Только `air_activity_service` из WAL.
- ❌ WAL Reader не генерирует commands. Commands только от Event Router.
- ❌ Никто не публикует «event для приказа» (например, `event.generate_report`). Для приказов — command.

---

## 2. Двухслойная модель событий

### Слой 1: CDC (Change Data Capture)

Автоматически генерируется из WAL. Гарантированный минимум.

| Действие в БД | Type | Action | Кто публикует |
|---------------|------|--------|---------------|
| INSERT | event | `created` | air_activity_service |
| UPDATE | signal | `updated` | air_activity_service |
| DELETE | signal | `deleted` | air_activity_service |

3 стандартных action'а. Никаких других в CDC-слое.

### Слой 2: Domain events (от воркеров)

Воркер публикует осознанно в коде, для факта завершения бизнес-процесса, который **не отражается в БД** или **является долгим**.

| Сценарий | Type | Action | Пример subject |
|----------|------|--------|----------------|
| Векторизация завершена | signal | `vectorized` | `signal.v1.notes.note.vectorized` |
| Отчёт сгенерирован | signal | `completed` | `signal.v1.reports.report.completed` |
| AI-анализ провалился (long-running) | signal | `generation_failed` | `signal.v1.reports.report.generation_failed` |
| Документ переиндексирован | signal | `reindexed` | `signal.v1.documents.document.reindexed` |

### Когда domain event, когда нет

**Domain event НУЖЕН** если:
- Long-running процесс (10+ секунд), пользователь ждёт результат → `*.completed` / `*.failed`
- Бизнес-факт, не отражённый в БД (вектор в Qdrant, индекс в Meilisearch) → `*.vectorized`, `*.indexed`
- Событие потребляет другой воркер для запуска следующего шага

**Domain event НЕ НУЖЕН** если:
- Простое изменение в БД (изменили title) → автоматически сгенерится `signal.*.updated`
- Создание сущности → автоматически сгенерится `event.*.created`
- Удаление → автоматически сгенерится `signal.*.deleted`

### Как публикует воркер

```go
// В processor.go воркера
subject := contracts.SignalSubject("notes", "note", "vectorized")
payload := map[string]any{
    "entity_id":  note.ID,
    "tenant_id":  msg.TenantID,
    "vector_dim": len(vec),
}
return p.bus.Publish(ctx, subject, payload)
```

Subject строится через `contracts.SignalSubject(domain, entity, action)` — никогда не хардкод строки.

---

## 3. Subject formula

```
[type].v1.[domain].[entity].[action]
```

### Компоненты

| Компонент | Значения | Описание |
|-----------|----------|----------|
| `type` | `command` \| `event` \| `signal` | Единственное число! |
| `version` | `v1` | Текущая версия |
| `domain` | `notes`, `chat`, `tasks`, `documents`, `reports`, ... | Логический модуль |
| `entity` | `note`, `message`, `task`, `document`, `report`, ... | Конкретная сущность |
| `action` | `created`/`updated`/`deleted` (CDC) или бизнес-глагол (`vectorized`, `completed`, `send`) | Что произошло |

### Примеры

```
command.v1.chat.message.send
event.v1.notes.note.created
signal.v1.notes.note.updated
signal.v1.notes.note.deleted
signal.v1.notes.note.vectorized          ← domain event от воркера
signal.v1.reports.report.completed       ← long-running success
signal.v1.reports.report.generation_failed  ← long-running failure
command.v1.documents.document.reindex
```

### Default правило

Для простых таблиц (где `domain == entity`): `domain = entity = table name as-is`.

```
events.v1.notes.note.created    ← таблица 'notes', domain=notes, entity=note
                                 (по умолчанию domain=entity=table, но мы пишем явно)
```

Реально в коде: если `table = "notes"`, то subject будет `events.v1.notes.notes.created`. **Это ок.** Никакой автоконвертации множественного в единственное.

### Когда domain ≠ entity

Для таблиц с составной структурой (`chat_messages`, `chat_rooms`, `chat_members`):

В `internal/contracts/tables/chat_messages.json`:
```json
{
  "table": "chat_messages",
  "domain": "chat",
  "entity": "message"
}
```

→ subject: `event.v1.chat.message.created`

Mapping хранится в `contract_mappings` таблице БД (см. `METAMODEL.md`).

---

## 4. NATS Streams

| Stream | Subjects | Retention | Назначение |
|--------|----------|-----------|------------|
| `COMMANDS` | `command.v1.>` | WorkQueue (1 consumer per command) | Приказы от Event Router |
| `EVENTS` | `event.v1.>` | 24h+ | Факты создания |
| `SIGNALS` | `signal.v1.>` | 24h+ | Изменения состояния |
| `INDEXER` | aggregate: `event.v1.>` + `signal.v1.>` | — | Для cross-type consumers (meili_indexer) |

### INDEXER pattern

Если воркер должен слушать и events, и signals (например, search indexer — переиндексировать и при создании, и при обновлении), используй aggregate stream `INDEXER` вместо двух отдельных подписок.

---

## 5. Wildcard-подписки

**Только последние `>` токены:**

```
events.v1.>          ← все events
signals.v1.>         ← все signals
commands.v1.>        ← все commands
```

### Запрещено

```
events.v1.>.created          ← НЕВАЛИДНО в NATS (> только последним токеном)
events.v1.notes.note.created ← хардкод, используй contracts.EventSubject()
```

### Фильтрация по action

Делается **в коде consumer'а**, не в subject:

```go
// Воркер хочет только events.v1.notes.note.created
// Подписка: events.v1.>
// Фильтр в коде:
if msg.Subject != contracts.EventSubject("notes", "note", "created") {
    msg.Ack()
    return
}
```

Или использовать queue group с `FilterSubject` в JetStream consumer config — NATS отфильтрует на своей стороне.

---

## 6. Tenant rules

### Главное правило

> **`tenant_id` никогда не попадает в NATS subject.** Только в payload, Redis-key, БД.

✅ Правильно: `event.v1.notes.note.created` (tenant_id в payload)
❌ Неправильно: `event.v1.tenant_42.notes.note.created`

### Почему

- 10 000 tenants = 10 000 уникальных топиков = NATS умрёт
- Имена инстансов БД (`pg-host-1`) — детали инфраструктуры, воркеру всё равно
- Имена получателей (`to.notification_service`) — завтра это же событие нужно аналитике, не переписывай

### Где `tenant_id` живёт

| Место | Пример |
|-------|--------|
| JWT Claims | `{"tenant_id": "uuid", ...}` |
| NATS Payload | `{"tenant_id": "uuid", "entity_id": "...", ...}` |
| Redis key | `tenant_42:notes:uuid-123` |
| PostgreSQL | `tenant_id UUID NOT NULL` в каждой таблице |
| activity_logs | `tenant_id UUID NOT NULL` |

---

## 7. Redis cache

### Entity cache key

```
{tenant_id}:{table_name}:{entity_id}
```

`table_name` — имя таблицы as-is из WAL (`notes`, `chat_messages`, `tasks`). **Не меняется** при переходе на domain/entity в subject.

### Примеры

```
abc123:notes:5efd6848-...
abc123:chat_messages:1e2f3a4b-...
abc123:tasks:9a1b2c3d-...
```

### WAL → Redis rules

| Действие | Redis команда |
|----------|---------------|
| INSERT | `SET key new_values EX ttl` (5-15 min) |
| UPDATE | `SET key new_values EX ttl` (сразу актуально, без запроса в БД) |
| DELETE | `DEL key` |

### Другие Redis ключи

| Key | Назначение | TTL |
|-----|------------|-----|
| `tenant:{id}:db_host` | Tenant Registry — хост БД | permanent |
| `tenant:{id}:tier` | Уровень (shared/dedicated) | permanent |
| `tenant:{id}:reader_url` | URL Phoenix Reader | permanent |
| `casbin:{tenant_id}:policy` | RBAC-политики Casbin | hot reload |
| `dedup:{event_id}` | Идемпотентность воркеров (SetNX) | 5 min |
| `rl:{user_id}` | Rate limit счётчик (KrakenD) | 1 min |

---

## 8. Failure handling

### Три класса ошибок

| Класс | Пример | Что делаем | Публикуем signal? |
|-------|--------|------------|-------------------|
| **Transient** | DB упала, AI provider timeout, Redis недоступен | `msg.Nak()` → NATS retry → после max_deliver в DLQ | ❌ Нет |
| **Permanent business** | Пустой текст, нет прав, невалидный ID | `msg.Ack()` (не ретраим) + лог в Sentry | 🟡 Только для long-running |
| **Long-running failure** | AI generation упала через 30 сек | `msg.Ack()` + publish `*.failed` signal | ✅ Да |

### Subject naming для failure

```
signal.v1.reports.report.generation_failed
```

Формула: `signal.v1.[domain].[entity].[action]_failed`. Не отдельный тип, просто action с суффиксом `_failed`.

### Когда публиковать failure signal

- ❌ Короткие процессы (vectorize, index, notify) — пользователь не видит «векторизуется сейчас», failure ему не виден. Только лог.
- ✅ Long-running процессы (generate_report, ai_analysis) — пользователь ждёт результат, должен знать если упало.

### Default правило

> Thin event philosophy: events — это факты, не статус-репорты.

UI не получает failure signal для коротких процессов. Это операционная проблема, не бизнес-факт.

---

## 9. Цепочки реакций

```
COMMAND может породить EVENT
  ↓
event.*.created → воркер обрабатывает → INSERT в БД → event.*.created

EVENT может породить SIGNAL
  ↓
event.note.created → vectorizer → сохраняет в Qdrant → publishes signal.note.vectorized

SIGNAL может породить COMMAND
  ↓
signal.document.updated → ai_agent решает → publishes command.document.reindex
```

### Пример полной цепочки

```
1. UI → POST /events с command.v1.reports.report.generate
2. Report Worker → INSERT в reports (status=pending)
   → air_activity_service из WAL → event.v1.reports.report.created
   → UI видит "в процессе"
3. Report Worker работает 10 минут
4. Report Worker → UPDATE reports SET status=completed, content=...
   → air_activity_service из WAL → signal.v1.reports.report.updated
5. Report Worker → publishes signal.v1.reports.report.completed (domain event)
   → UI видит "готово"
6. Search Indexer (подписан на signal.reports.completed) → индексирует в Meilisearch
```

Шаги 4 и 5 — это два разных signal. 4 — автоматический из WAL (произошёл UPDATE). 5 — domain event от воркера (факт завершения процесса).

---

## 10. Запрещённые паттерны

### ❌ Command вместо Event

```
❌ command.user.created
✅ event.user.created (из WAL при INSERT в users)
```

### ❌ Signal вместо Event

```
❌ signal.document.created
✅ event.document.created (из WAL при INSERT)
```

### ❌ Event для приказа

```
❌ event.generate_report
✅ command.reports.report.generate
```

### ❌ Цепочки в манифесте

```json
// ❌ ПЛОХО — это BPMN-движок
{
  "chain": ["vectorize", "classify", "summarize", "notify"]
}

// ✅ ПРАВИЛЬНО — Manifest описывает только "что слушаем / что запускаем"
{
  "subscribes": ["notes.created"],
  "publishes":  ["notes.vectorized"]
}
```

Что делать после запуска процесса — живёт в коде processor.go.

### ❌ Хардкод subject в коде

```go
// ❌ ПЛОХО
nats.Publish("events.v1.notes.note.created", payload)

// ✅ ПРАВИЛЬНО
subject := contracts.EventSubject("notes", "note", "created")
nats.Publish(subject, payload)
```

### ❌ tenant_id в subject

```
❌ events.v1.tenant_42.notes.note.created
✅ events.v1.notes.note.created (tenant_id в payload)
```

### ❌ Множественное число в type

```
❌ signals.v1.notes.note.updated
❌ events.v1.notes.note.created
❌ commands.v1.chat.message.send

✅ signal.v1.notes.note.updated
✅ event.v1.notes.note.created
✅ command.v1.chat.message.send
```

---

## 11. Worker — правила взаимодействия

### Что может воркер

- ✅ Подписываться на events/signals/commands через логические ID в `worker.json`
- ✅ Писать в БД (через свой repository layer)
- ✅ Публиковать domain events (signals) через `contracts.SignalSubject()` + `bus.Publish()`
- ✅ Делать исходящие HTTP-вызовы к `services/` монорепо или standalone core-сервисам (ai-provider, file-storage)
- ✅ Читать из Redis (cache lookup)
- ✅ Использовать `contracts.*Subject()` для построения любых subject-ов

### Что не может воркер

- ❌ Иметь HTTP-сервер (внешний API) — это зона `services/`
- ❌ Публиковать CDC-события (event для INSERT, signal для UPDATE/DELETE) — это делает только `air_activity_service`
- ❌ Хардкодить subject строки — только через `contracts.*`
- ❌ Читать чужие таблицы напрямую — только через `entity_links` или HTTP API чужого сервиса
- ❌ Знать про другой воркер — только через NATS

---

## 12. Формула платформы

```
COMMAND  → Что нужно сделать
EVENT    → Что появилось
SIGNAL   → Что изменилось

Worker   → Как реагировать
Process  → Что выполнять
Contracts → Как общаться
Registry → Что существует
```

---

## Cross-references

- **Subject construction pipeline (JSON → contract_mappings → Contracts.Subject())** → `METAMODEL.md`
- **Contracts API (Go + Elixir)** → `CONTRACTS_REFERENCE.md`
- **Пример: воркер публикует domain event** → `LIFECYCLE_ADDING_WORKER.md`
- **Сквозные трейсы с цепочками** → `WIRING_DIAGRAMS.md`
