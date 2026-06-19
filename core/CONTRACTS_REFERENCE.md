# CONTRACTS_REFERENCE.md

> **Каноничный API reference** для библиотеки Contracts в Go и Elixir. Заменяет устаревший код с `Subject(type, table, action)` (3 параметра, table дважды) на актуальный `Subject(type, domain, entity, action)` (4 параметра).

---

## TL;DR

Contracts — единый source of truth для построения subject-ов, имён streams, Redis-ключей, нормализации actions. **Subject никогда не хардкодится** — только через `contracts.*Subject()`. Реализации на Go и Elixir идентичны. Subject строится из 4 параметров: `type`, `domain`, `entity`, `action`. Mapping `table_name → (domain, entity)` берётся из `contract_mappings` таблицы БД.

---

## 1. Роль Contracts

```
Что делает:                 Что НЕ делает:
─────────────               ─────────────
✅ Строит subject            ❌ Не хранит контракты
✅ Строит Redis-ключ         ❌ Не валидирует payload
✅ Возвращает имена streams  ❌ Не публикует в NATS
✅ Нормализует WAL actions   ❌ Не подписывается на NATS
✅ Даёт wildcard-паттерны    ❌ Не знает про tenants
```

Contracts — чистые функции. Состояния нет. Контекста нет. Тестировать тривиально.

---

## 2. Go API

### Файл `internal/contracts/contracts.go`

```go
package contracts

import "fmt"

const Version = "v1"

// Stream names — должны совпадать с тем, что создаёт nats.go при init streams
const (
    StreamCommands = "COMMANDS"
    StreamEvents   = "EVENTS"
    StreamSignals  = "SIGNALS"
    StreamIndexer  = "INDEXER" // aggregate: EVENTS + SIGNALS, для cross-type consumers
)

// Subject строит NATS-subject по формуле: [type].v1.[domain].[entity].[action]
//
//   type   ∈ {"command", "event", "signal"}  (ед.ч.!)
//   domain = логический модуль ("notes", "chat", "tasks")
//   entity = конкретная сущность ("note", "message", "task")
//   action = "created"/"updated"/"deleted" (CDC) или бизнес-глагол ("vectorized", "completed")
//
// Примеры:
//   Subject("event",  "notes", "note", "created")    → "event.v1.notes.note.created"
//   Subject("signal", "notes", "note", "vectorized") → "signal.v1.notes.note.vectorized"
//   Subject("command","chat",  "message", "send")    → "command.v1.chat.message.send"
//
func Subject(msgType, domain, entity, action string) string {
    return fmt.Sprintf("%s.%s.%s.%s.%s", msgType, Version, domain, entity, action)
}

func EventSubject(domain, entity, action string) string {
    return Subject("event", domain, entity, action)
}

func SignalSubject(domain, entity, action string) string {
    return Subject("signal", domain, entity, action)
}

func CommandSubject(domain, entity, action string) string {
    return Subject("command", domain, entity, action)
}

// CacheKey строит Redis-ключ по схеме: {tenant_id}:{table_name}:{entity_id}
//
// ВАЖНО: table_name — имя таблицы as-is из WAL (не domain.entity!).
// Это внутреннее дело cache layer, не меняется при переходе на domain/entity в subject.
//
//   CacheKey("tenant_42", "notes", "uuid-123")      → "tenant_42:notes:uuid-123"
//   CacheKey("tenant_42", "chat_messages", "uuid")  → "tenant_42:chat_messages:uuid"
//
func CacheKey(tenantID, tableName, entityID string) string {
    return fmt.Sprintf("%s:%s:%s", tenantID, tableName, entityID)
}

// NormalizeAction преобразует WAL-атомы в строковый action контракта.
// Нормализация происходит ОДИН РАЗ — при входе в air_activity_service.
//
//   NormalizeAction("insert")  → "created"
//   NormalizeAction("update")  → "updated"
//   NormalizeAction("delete")  → "deleted"
//   NormalizeAction("created") → "created"  (idempotent)
//
func NormalizeAction(raw string) string {
    switch raw {
    case "insert", "created":
        return "created"
    case "update", "updated":
        return "updated"
    case "delete", "deleted":
        return "deleted"
    default:
        return raw
    }
}

// MessageType определяет тип сообщения по action.
// INSERT → event (есть данные нового объекта)
// UPDATE/DELETE → signal (только триггер)
//
//   MessageType("created") → "event"
//   MessageType("updated") → "signal"
//   MessageType("deleted") → "signal"
//   MessageType("vectorized") → "signal"  (domain events — всегда signal)
//
func MessageType(action string) string {
    if action == "created" {
        return "event"
    }
    return "signal"
}

// Wildcard-паттерны для подписок. > только последним токеном в NATS.
//
//   SubAllEvents   = "event.v1.>"
//   SubAllSignals  = "signal.v1.>"
//   SubAllCommands = "command.v1.>"
//
// Фильтрацию по action делай в коде consumer'а или через JetStream FilterSubject.
//
const (
    SubAllEvents   = "event.v1.>"
    SubAllSignals  = "signal.v1.>"
    SubAllCommands = "command.v1.>"
)
```

### Пример использования

```go
// В processor.go воркера
subject := contracts.SignalSubject("notes", "note", "vectorized")
bus.Publish(ctx, subject, payload)

// В main.go воркера
consumer, _ := nc.JS.CreateOrUpdateConsumer(ctx, "EVENTS", jetstream.ConsumerConfig{
    Name:          "notes-vectorize-events",
    Durable:       "notes-vectorize-events",
    FilterSubject: contracts.EventSubject("notes", "note", "created"),
    // или шире: contracts.SubAllEvents и фильтр в коде
    AckPolicy:     jetstream.AckExplicitPolicy,
    MaxAckPending: 50,
})

// В repository при работе с Redis
key := contracts.CacheKey(tenantID, "notes", noteID)
cache.Set(ctx, key, value, ttl)
```

---

## 3. Elixir API

### Файл `lib/air_moon/contracts.ex`

```elixir
defmodule AirMoon.Contracts do
  @moduledoc """
  Единый источник правды для именования subject-ов, имён streams и Redis-ключей.

  Формула NATS-subject:  [type].v1.[domain].[entity].[action]
  Формула Redis-ключа:   {tenant_id}:{table_name}:{entity_id}

  Правила:
  - tenant_id НИКОГДА не попадает в subject
  - WAL даёт атомы (:insert/:update/:delete) — нормализуем через normalize_action/1
  - INSERT → message_type "event"  (факт создания, slim-данные)
  - UPDATE/DELETE → message_type "signal" (триггер, без бизнес-данных)
  - COMMAND — только от Event Router, air_activity_service их не генерирует
  """

  @version "v1"

  # ── Stream names (JetStream) ─────────────────────────────────────────────────

  @stream_commands "COMMANDS"
  @stream_events    "EVENTS"
  @stream_signals   "SIGNALS"
  @stream_indexer   "INDEXER"  # aggregate: EVENTS + SIGNALS, для cross-type consumers

  def stream(:commands), do: @stream_commands
  def stream(:events),   do: @stream_events
  def stream(:signals),  do: @stream_signals
  def stream(:indexer),  do: @stream_indexer

  # ── Subject builders ─────────────────────────────────────────────────────────

  @doc """
  Строит NATS-subject по формуле: [type].v1.[domain].[entity].[action]

      iex> AirMoon.Contracts.subject("event", "notes", "note", "created")
      "event.v1.notes.note.created"

      iex> AirMoon.Contracts.subject("signal", "notes", "note", "vectorized")
      "signal.v1.notes.note.vectorized"

      iex> AirMoon.Contracts.subject("command", "chat", "message", "send")
      "command.v1.chat.message.send"
  """
  def subject(type, domain, entity, action) do
    "#{type}.#{@version}.#{domain}.#{entity}.#{action}"
  end

  def event_subject(domain, entity, action),   do: subject("event",   domain, entity, action)
  def signal_subject(domain, entity, action),  do: subject("signal",  domain, entity, action)
  def command_subject(domain, entity, action), do: subject("command", domain, entity, action)

  # ── WAL action normalization ──────────────────────────────────────────────────
  # Postgres WAL → атомы (:insert/:update/:delete)
  # Контракт → строки ("created"/"updated"/"deleted")
  # Нормализация ОДИН РАЗ — при входе в air_activity_service.

  def normalize_action(:insert),  do: "created"
  def normalize_action(:update),  do: "updated"
  def normalize_action(:delete),  do: "deleted"
  def normalize_action("insert"), do: "created"
  def normalize_action("update"), do: "updated"
  def normalize_action("delete"), do: "deleted"
  def normalize_action(a) when a in ["created", "updated", "deleted"], do: a
  def normalize_action(a), do: to_string(a)

  # ── Message type ─────────────────────────────────────────────────────────────
  # INSERT  → "event"  (есть данные нового объекта)
  # UPDATE/DELETE → "signal" (только триггер)

  def message_type("created"), do: "event"
  def message_type(_),         do: "signal"

  # ── Wildcard subscriptions ────────────────────────────────────────────────────

  def sub_all_events,   do: "event.#{@version}.>"
  def sub_all_signals,  do: "signal.#{@version}.>"
  def sub_all_commands, do: "command.#{@version}.>"

  # ── Redis key ────────────────────────────────────────────────────────────────
  # Схема: {tenant_id}:{table_name}:{entity_id}
  # table_name — имя таблицы as-is из WAL (не domain.entity!)

  def cache_key(tenant_id, table_name, entity_id) do
    "#{tenant_id}:#{table_name}:#{entity_id}"
  end
end
```

### Пример в `air_activity_service`

```elixir
# В NatsPublisher
defp build_subject(message_type, entity_type, action) do
  # entity_type — имя таблицы из WAL ("chat_messages", "notes")
  # Lookup в contract_mappings для получения (domain, entity)
  mapping = ContractMappings.lookup(entity_type)
  # mapping = %{domain: "chat", entity: "message"}

  Contracts.subject(message_type, mapping.domain, mapping.entity, action)
end

# В CacheInvalidatorConsumer
defp process_event(event) do
  action      = Contracts.normalize_action(event[:action])
  tenant_id   = event[:tenant_id]
  entity_type = event[:entity_type]  # имя таблицы
  entity_id   = event[:entity_id]

  key = Contracts.cache_key(tenant_id, entity_type, entity_id)

  case action do
    "created" -> Cache.set(tenant_id, entity_type, entity_id, event[:new_data])
    "updated" -> Cache.set(tenant_id, entity_type, entity_id, event[:new_data])
    "deleted" -> Cache.delete(tenant_id, entity_type, entity_id)
  end
end
```

---

## 4. Subject construction pipeline

### При публикации из WAL (air_activity_service)

```
1. Из WAL получаем:
   - entity_type = "chat_messages"  (имя таблицы)
   - action = :insert
   - new_data = {...}

2. Normalize action:
   action = Contracts.normalize_action(:insert)  # → "created"

3. Determine message type:
   message_type = Contracts.message_type("created")  # → "event"

4. Lookup mapping:
   mapping = ContractMappings.lookup("chat_messages")
   # → %{domain: "chat", entity: "message"}

5. Build subject:
   subject = Contracts.subject("event", "chat", "message", "created")
   # → "event.v1.chat.message.created"

6. Publish to NATS:
   NatsPublisher.publish(subject, payload)
```

### При публикации domain event (Go-воркер)

```go
// Воркер знает domain/entity/action напрямую (он owner контракта)
subject := contracts.SignalSubject("notes", "note", "vectorized")
// → "signal.v1.notes.note.vectorized"

bus.Publish(ctx, subject, payload)
```

Никакого lookup в `contract_mappings` не нужно — воркер сам знает свой контракт.

### При подписке воркера (build-time)

```go
// В main.go при создании consumer
filterSubject := contracts.EventSubject("notes", "note", "created")
// → "event.v1.notes.note.created"

consumer, _ := nc.JS.CreateOrUpdateConsumer(ctx, "EVENTS", jetstream.ConsumerConfig{
    FilterSubject: filterSubject,
    ...
})
```

Если воркер подписан на логический ID из `worker.json` `subscribes`, при старте он делает lookup в `contract_mappings`, получает `(type, domain, entity, action)` и строит subject через `contracts.*Subject()`.

---

## 5. Примеры контрактов

### CDC contract (автоматически из table descriptor)

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

Sync script генерирует 3 строки в `contract_mappings`:

| contract_id | type | domain | entity | action | table_name | source | subject |
|-------------|------|--------|--------|--------|------------|--------|---------|
| chat.message.created | event | chat | message | created | chat_messages | cdc | `event.v1.chat.message.created` |
| chat.message.updated | signal | chat | message | updated | chat_messages | cdc | `signal.v1.chat.message.updated` |
| chat.message.deleted | signal | chat | message | deleted | chat_messages | cdc | `signal.v1.chat.message.deleted` |

### Domain contract (от воркера)

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

→ `contract_mappings`:

| contract_id | type | domain | entity | action | table_name | source | subject |
|-------------|------|--------|--------|--------|------------|--------|---------|
| notes.vectorized | signal | notes | note | vectorized | NULL | domain | `signal.v1.notes.note.vectorized` |

### Command contract

`workers/reports-generate/contracts/reports.generate.json`:
```json
{
  "contract_id": "reports.generate",
  "name": "Generate Report",
  "description": "Requests report generation by user",
  "type": "command",
  "domain": "reports",
  "entity": "report",
  "action": "generate",
  "table": null,
  "source": "command",
  "ui_visible": false,
  "schema": "reports.generate.schema.json",
  "version": 1
}
```

→ `contract_mappings`:

| contract_id | type | domain | entity | action | table_name | source | subject |
|-------------|------|--------|--------|--------|------------|--------|---------|
| reports.generate | command | reports | report | generate | NULL | command | `command.v1.reports.report.generate` |

---

## 6. INDEXER pattern (aggregate stream)

### Проблема

Search indexer хочет переиндексировать Meilisearch и при создании сущности (event), и при обновлении (signal). Две подписки — два consumer'а, две горутины, две DLQ.

### Решение

Создаём aggregate stream `INDEXER`:

```bash
# NATS CLI
nats stream create INDEXER \
  --sources EVENTS,SIGNALS \
  --storage file \
  --retention limits \
  --max-age 24h
```

Воркер подписывается на один stream:

```go
consumer, _ := nc.JS.CreateOrUpdateConsumer(ctx, "INDEXER", jetstream.ConsumerConfig{
    Name:          "search-indexer",
    Durable:       "search-indexer",
    FilterSubject: ">",  // все subject-ы из обоих sources
    AckPolicy:     jetstream.AckExplicitPolicy,
    MaxAckPending: 100,
})
```

В handler'е различаем по subject:

```go
handler := func(msg jetstream.Msg) {
    // msg.Subject может быть "event.v1.notes.note.created"
    // или "signal.v1.notes.note.updated"
    parts := strings.Split(msg.Subject, ".")
    msgType := parts[0]  // "event" или "signal"
    domain  := parts[2]
    entity  := parts[3]
    action  := parts[4]
    
    switch action {
    case "created":
        // полный reindex из БД
    case "updated":
        // переиндексация конкретной сущности
    case "deleted":
        // удаление из индекса
    }
    
    msg.Ack()
}
```

### Когда использовать INDEXER

- ✅ Search indexer (слушает и creation, и updates)
- ✅ Analytics aggregator
- ✅ Audit logger (записывает все изменения)
- ❌ Worker, который реагирует только на `*.created` — обычная подписка на `EVENTS` дешевле

---

## 7. Decision tree: CDC vs Domain vs Command

```
Нужно опубликовать событие. Какой тип?

┌──────────────────────────────────────────────────────┐
│ Это команда от пользователя/UI?                      │
│ (UI инициирует действие, ждёт результат)             │
└──────────────────────────────────────────────────────┘
              │
              ▼ YES → COMMAND
              │       contract: command.v1.<domain>.<entity>.<action>
              │       publisher: Event Router (по HTTP от actor'а)
              │       example: command.v1.reports.report.generate
              │
              ▼ NO
┌──────────────────────────────────────────────────────┐
│ Это факт INSERT в БД?                                │
│ (новая сущность создана)                             │
└──────────────────────────────────────────────────────┘
              │
              ▼ YES → EVENT (CDC)
              │       contract: event.v1.<domain>.<entity>.created
              │       publisher: air_activity_service (автоматически из WAL)
              │       example: event.v1.notes.note.created
              │       ⚠️ Воркер НЕ публикует это сам!
              │
              ▼ NO
┌──────────────────────────────────────────────────────┐
│ Это факт UPDATE/DELETE в БД?                         │
│ (существующая сущность изменилась)                   │
└──────────────────────────────────────────────────────┘
              │
              ▼ YES → SIGNAL (CDC)
              │       contract: signal.v1.<domain>.<entity>.updated|deleted
              │       publisher: air_activity_service (автоматически из WAL)
              │       example: signal.v1.notes.note.updated
              │       ⚠️ Воркер НЕ публикует это сам!
              │
              ▼ NO
┌──────────────────────────────────────────────────────┐
│ Это факт завершения бизнес-процесса,                 │
│ НЕ отражённый в БД?                                  │
│ (вектор в Qdrant, индекс в Meilisearch, long-running)│
└──────────────────────────────────────────────────────┘
              │
              ▼ YES → SIGNAL (DOMAIN)
              │       contract: signal.v1.<domain>.<entity>.<custom_action>
              │       publisher: воркер в коде через contracts.SignalSubject()
              │       example: signal.v1.notes.note.vectorized
              │                signal.v1.reports.report.completed
              │                signal.v1.reports.report.generation_failed
              │
              ▼ NO
              ┌──────────────────────────────────────────┐
              │ Возможно, событие не нужно публиковать.  │
              │ Может, это просто UPDATE в БД,           │
              │ и signal.updated уже сгенерится из WAL?  │
              └──────────────────────────────────────────┘
```

---

## 8. Backward compatibility

### Как переименовать contract

Сценарий: `notes.vectorized` → `notes.embedding_ready` (решили использовать более точное имя).

**Шаг 1.** Переименовать файл:
```
workers/notes-vectorize/contracts/notes.vectorized.json
→ workers/notes-vectorize/contracts/notes.embedding_ready.json
```

**Шаг 2.** Внутри файла обновить `contract_id` и `action`:
```json
{
  "contract_id": "notes.embedding_ready",
  "action": "embedding_ready",
  ...
}
```

**Шаг 3.** В `worker.json` обновить `publishes`:
```json
"publishes": ["notes.embedding_ready"]
```

**Шаг 4.** В `processor.go` обновить вызов:
```go
subject := contracts.SignalSubject("notes", "note", "embedding_ready")
```

**Шаг 5.** Запустить sync-contracts:
```
- notes.vectorized (removed)
+ notes.embedding_ready (new)
Apply? [y/N]: y
✓ DELETE FROM contract_mappings WHERE contract_id = 'notes.vectorized'
✓ INSERT INTO contract_mappings (contract_id, ...) VALUES ('notes.embedding_ready', ...)
```

**Шаг 6.** Обновить подписчиков (воркеры, у которых `notes.vectorized` в `subscribes`).

### Версионирование контракта

Если меняется payload schema, но не имя:

```json
{
  "contract_id": "notes.vectorized",
  "version": 2,
  ...
}
```

В `contract_mappings`:
```sql
UPDATE contract_mappings SET version = 2, schema_path = '...', updated_at = NOW()
WHERE contract_id = 'notes.vectorized';
```

Подписчики могут читать `version` из payload и решать, как парсить.

### Параллельные версии

Если нужно поддерживать старых подписчиков, можно временно публиковать оба:

```go
// Публикуем и старую, и новую версию
oldSubject := contracts.SignalSubject("notes", "note", "vectorized") // v1
newSubject := contracts.SignalSubject("notes", "note", "vectorized") // v2 (тот же subject, другой payload)

bus.Publish(ctx, oldSubject, oldPayload)
bus.Publish(ctx, newSubject, newPayload)
```

Версия в payload:
```json
{
  "version": 2,
  "entity_id": "...",
  ...
}
```

Подписчик читает `version` и решает, как парсить.

---

## 9. Migration со старой формулы subject

### Было (старый код)

```go
// 3 параметра, table дважды
func Subject(msgType, table, action string) string {
    return fmt.Sprintf("%s.%s.%s.%s.%s", msgType, Version, table, table, action)
}
// → "events.v1.notes.notes.created" (table дважды, множественное в type)
```

### Стало (актуальный код)

```go
// 4 параметра, domain и entity раздельно
func Subject(msgType, domain, entity, action string) string {
    return fmt.Sprintf("%s.%s.%s.%s.%s", msgType, Version, domain, entity, action)
}
// → "event.v1.notes.note.created" (ед.ч., domain/entity разделены)
```

### Что нужно сделать при миграции

1. **Обновить `contracts.go` и `contracts.ex`** на новую сигнатуру
2. **Заполнить `contract_mappings` таблицу** через sync script (читает JSON-и, строит subject по новой формуле)
3. **Обновить `air_activity_service`** — `NatsPublisher` теперь делает lookup mapping по `entity_type` (имя таблицы) → `(domain, entity)`, вызывает `Contracts.subject/4`
4. **Обновить Go-воркеры** — все вызовы `contracts.EventSubject(table, action)` заменить на `contracts.EventSubject(domain, entity, action)`
5. **Обновить JetStream consumers** — `FilterSubject` строится через `contracts.*Subject(domain, entity, action)`
6. **Пересоздать NATS streams** с новыми subject patterns:
   - `EVENTS` → subjects `event.v1.>` (было `events.v1.>`)
   - `SIGNALS` → subjects `signal.v1.>` (было `signals.v1.>`)
   - `COMMANDS` → subjects `command.v1.>` (было `commands.v1.>`)
7. **Обновить `air_hb_beta`** — подписка на `signal.v1.>` и `event.v1.>` (ед.ч.)

### Что НЕ меняется

- Redis cache key: `{tenant_id}:{table_name}:{entity_id}` — table_name остаётся
- `cache_invalidator_consumer.ex` — не трогаем
- `channel_consumer.ex` — не трогаем
- `producer.ex` — не трогаем
- WAL decoding — не трогаем, из WAL приходит `entity_type` = имя таблицы

---

## 10. Anti-patterns

### ❌ Хардкод subject строки

```go
// ПЛОХО
nats.Publish("event.v1.notes.note.created", payload)

// ХОРОШО
subject := contracts.EventSubject("notes", "note", "created")
nats.Publish(subject, payload)
```

### ❌ Множественное число в type

```go
// ПЛОХО
contracts.Subject("events", "notes", "note", "created")
// → "events.v1.notes.note.created" (мн.ч.)

// ХОРОШО
contracts.Subject("event", "notes", "note", "created")
// → "event.v1.notes.note.created" (ед.ч.)
```

### ❌ tenant_id в subject

```go
// ПЛОХО
contracts.EventSubject("tenant_42.notes", "note", "created")
// → "event.v1.tenant_42.notes.note.created"

// ХОРОШО
contracts.EventSubject("notes", "note", "created")
// tenant_id в payload
```

### ❌ Использование `table_name` в subject

```go
// ПЛОХО (если table_name = "chat_messages")
contracts.EventSubject("chat_messages", "chat_messages", "created")
// → "event.v1.chat_messages.chat_messages.created"

// ХОРОШО (через lookup mapping)
mapping := ContractMappings.LookupByTable("chat_messages")
// → {domain: "chat", entity: "message"}
contracts.EventSubject(mapping.domain, mapping.entity, "created")
// → "event.v1.chat.message.created"
```

### ❌ Wildcard в середине

```go
// ПЛОХО — NATS не поддерживает
"event.v1.>.created"

// ХОРОШО
contracts.SubAllEvents  // "event.v1.>"
// + фильтр по action в коде
```

### ❌ Subject в JSON-файле контракта

```json
// ПЛОХО — дублирование, при смене naming convention надо переписывать сотни файлов
{
  "contract_id": "notes.created",
  "subject": "event.v1.notes.note.created"
}

// ХОРОШО — subject вычисляется
{
  "contract_id": "notes.created",
  "type": "event",
  "domain": "notes",
  "entity": "note",
  "action": "created"
}
```

---

## Cross-references

- **Метамодель контрактов** → `METAMODEL.md`
- **Subject formula, message types** → `COMMUNICATION_MODEL.md`
- **Сквозной пример воркера** → `LIFECYCLE_ADDING_WORKER.md`
- **Архитектура БД и activity_logs** → `ARCHITECTURE.md`
