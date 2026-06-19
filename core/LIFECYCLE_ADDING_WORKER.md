# LIFECYCLE_ADDING_WORKER.md

> **Практический flow.** Это проверка архитектуры: новый воркер добавляется за 8 понятных шагов без магии. Если не получается — архитектура не работает.

---

## TL;DR

Чтобы добавить новый воркер: создаёшь папку `workers/<name>/`, кладёшь `worker.json` + `processor.go` + `contracts/<id>.json`, запускаешь `make sync-contracts` (preview → apply), запускаешь воркер через `cmd/workers/<name>/main.go`. Готово. Никаких правок в `internal/`, `cmd/services/`, registry.json (только если это первый воркер).

---

## Сценарий

> **Хочу:** после создания заметки автоматически векторизовать её текст в Qdrant, а по факту векторизации опубликовать domain event `notes.vectorized`, на который подпишется search indexer.

---

## Шаг 1. Создать папку воркера

```
workers/notes-vectorize/
```

Структура, которая появится к концу:
```
workers/notes-vectorize/
├── worker.json
├── processor.go
├── domain/
│   └── note.go                      ← доменная сущность (опционально)
├── repository/
│   └── note_repo.go                 ← SQL (опционально)
└── contracts/
    ├── notes.vectorized.json
    ├── notes.vectorized.schema.json
    └── notes.vectorized.ui.json     ← опционально, если ui_visible
```

---

## Шаг 2. Описать domain contract

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

### Что важно

- `contract_id` = `notes.vectorized` — это логический ID, на него будут ссылаться в `worker.json` и в других воркерах через `subscribes`
- `type` = `signal` — domain event от воркера (не CDC, не command)
- `source` = `domain` — публикует воркер в коде, не air_activity_service
- `table` = `null` — нет таблицы `vectorizations`, это чисто доменное событие
- `ui_visible` = `true` — хотим показывать в UI (значит, нужен `ui.json`)

---

## Шаг 3. Описать schema для payload

`workers/notes-vectorize/contracts/notes.vectorized.schema.json`:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "NoteVectorizedPayload",
  "type": "object",
  "properties": {
    "entity_id":  {"type": "string", "format": "uuid"},
    "tenant_id":  {"type": "string", "format": "uuid"},
    "vector_dim": {"type": "integer", "minimum": 1, "maximum": 4096}
  },
  "required": ["entity_id", "tenant_id"]
}
```

---

## Шаг 4. Описать UI-детали (опционально)

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

Нужен только если `ui_visible: true`. Registry валидирует наличие при сборке.

---

## Шаг 5. Написать `worker.json`

`workers/notes-vectorize/worker.json`:

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

### Что важно

- `subscribes` содержит **логические ID** (`notes.created`, `notes.updated`), не subject строки
- `notes.created` и `notes.updated` — это CDC контракты, они описаны в `internal/contracts/tables/notes.json`
- `publishes` — наш новый domain contract
- `module: "notes"` — логическая группировка, Registry сам соберёт все `notes-*` воркеры в группу

---

## Шаг 6. Убедиться, что CDC contracts для `notes` существуют

Если таблицы `notes` ещё нет в `internal/contracts/tables/notes.json`, создай:

`internal/contracts/tables/notes.json`:
```json
{
  "table": "notes",
  "domain": "notes",
  "entity": "note",
  "description": "Notes table",
  "ui_visible": true
}
```

Это сгенерирует 3 CDC контракта:
- `notes.created` (event, cdc)
- `notes.updated` (signal, cdc)
- `notes.deleted` (signal, cdc)

И добавить в `registry.json`:
```json
{
  "artifacts": [
    {"type": "worker", "path": "./workers/notes-vectorize/worker.json"},
    {"type": "table",  "path": "./internal/contracts/tables/notes.json"}
  ]
}
```

(Если `registry.json` уже существует — просто добавь новую строку.)

---

## Шаг 7. Запустить sync-contracts

```bash
make sync-contracts
```

Вывод (preview):
```
Scanning workers/*/contracts/*.json...
Scanning internal/contracts/tables/*.json...

New contracts:
  + notes.vectorized (signal, domain)
    Subject: signal.v1.notes.note.vectorized
    File: workers/notes-vectorize/contracts/notes.vectorized.json
    UI: yes (notes.vectorized.ui.json found)
    Schema: notes.vectorized.schema.json

  + notes.created (event, cdc, from notes.json)
    Subject: event.v1.notes.note.created
    File: internal/contracts/tables/notes.json

  + notes.updated (signal, cdc, from notes.json)
    Subject: signal.v1.notes.note.updated

  + notes.deleted (signal, cdc, from notes.json)
    Subject: signal.v1.notes.note.deleted

Validation:
  ✓ Worker 'notes-vectorize' subscribes references valid contracts
  ✓ Worker 'notes-vectorize' publishes references valid contracts
  ✓ Table 'notes' exists in pg_class
  ✓ Table 'notes' has trigger trg_notes_activity

Apply? [y/N/p(review)]: y
✓ Inserted 4 rows into contract_mappings
✓ Notified air_activity_service (LISTEN/NOTIFY)
```

`contract_mappings` теперь содержит:

| contract_id | type | domain | entity | action | table_name | source | ui_visible |
|-------------|------|--------|--------|--------|------------|--------|------------|
| notes.created | event | notes | note | created | notes | cdc | true |
| notes.updated | signal | notes | note | updated | notes | cdc | true |
| notes.deleted | signal | notes | note | deleted | notes | cdc | true |
| notes.vectorized | signal | notes | note | vectorized | NULL | domain | true |

---

## Шаг 8. Написать код воркера

### `workers/notes-vectorize/processor.go`

```go
package notesvectorize

import (
    "context"
    "encoding/json"
    "fmt"

    "air-core-platform/internal/contracts"
    "air-core-platform/internal/event"
    "air-core-platform/internal/logger"

    "github.com/nats-io/nats.go/jetstream"
    "github.com/rs/zerolog"
)

type Processor struct {
    log    zerolog.Logger
    repo   NoteRepo
    ai     AIClient
    qdrant QdrantClient
    bus    Bus
}

type NoteRepo interface {
    GetNote(ctx context.Context, tenantID, noteID string) (*Note, error)
}

type AIClient interface {
    Embed(ctx context.Context, text string) ([]float32, error)
}

type QdrantClient interface {
    Upsert(ctx context.Context, id string, vector []float32, payload map[string]any) error
}

type Bus interface {
    Publish(ctx context.Context, subject string, payload any) error
}

type Note struct {
    ID       string `json:"id"`
    TenantID string `json:"tenant_id"`
    Title    string `json:"title"`
    Content  string `json:"content"`
}

func NewProcessor(log zerolog.Logger, repo NoteRepo, ai AIClient, qdrant QdrantClient, bus Bus) *Processor {
    return &Processor{log: log, repo: repo, ai: ai, qdrant: qdrant, bus: bus}
}

// Process — вызывается на каждое сообщение из NATS.
// subject построен заранее в main.go и передан как-то.
// Здесь уже распакованный event.Event.
func (p *Processor) Process(ctx context.Context, evt *event.Event) error {
    p.log.Info().
        Str("event_id", evt.ID).
        Str("entity_id", evt.EntityID).
        Str("action", evt.Action).
        Msg("processing note vectorization")

    // 1. Загрузить заметку из БД
    note, err := p.repo.GetNote(ctx, evt.TenantID, evt.EntityID)
    if err != nil {
        return fmt.Errorf("load note: %w", err) // transient — Nak, retry
    }

    // 2. Получить embedding
    text := note.Title + "\n" + note.Content
    if len(text) == 0 {
        p.log.Warn().Str("note_id", note.ID).Msg("empty note, skip")
        return nil // Ack, не retry
    }

    vec, err := p.ai.Embed(ctx, text)
    if err != nil {
        return fmt.Errorf("ai embed: %w", err) // transient — Nak, retry
    }

    // 3. Сохранить в Qdrant
    if err := p.qdrant.Upsert(ctx, note.ID, vec, map[string]any{
        "tenant_id": note.TenantID,
        "title":     note.Title,
    }); err != nil {
        return fmt.Errorf("qdrant upsert: %w", err) // transient — Nak, retry
    }

    // 4. ОПУБЛИКОВАТЬ domain event (через contracts, не хардкод!)
    subject := contracts.SignalSubject("notes", "note", "vectorized")
    payload := map[string]any{
        "entity_id":  note.ID,
        "tenant_id":  note.TenantID,
        "vector_dim": len(vec),
    }

    if err := p.bus.Publish(ctx, subject, payload); err != nil {
        // Не возвращаем ошибку — Qdrant уже обновлён, сигнал не критичен
        p.log.Warn().Err(err).Msg("publish notes.vectorized failed (non-fatal)")
    }

    p.log.Info().
        Str("note_id", note.ID).
        Int("vector_dim", len(vec)).
        Msg("note vectorized")

    return nil
}
```

### `cmd/workers/notes-vectorize/main.go`

```go
package main

import (
    "context"
    "os"
    "os/signal"
    "syscall"

    "air-core-platform/internal/config"
    "air-core-platform/internal/db"
    "air-core-platform/internal/logger"
    "air-core-platform/internal/natsconn"
    "air-core-platform/internal/worker"
    "air-core-platform/workers/notes-vectorize"

    "github.com/nats-io/nats.go/jetstream"
)

func main() {
    cfg := config.Load("notes-vectorize")
    log := logger.New(cfg.ServiceName, cfg.LogLevel)
    ctx := context.Background()

    // Infrastructure
    nc, err := natsconn.Connect(cfg.NatsURL, log)
    if err != nil {
        log.Fatal().Err(err).Msg("nats failed")
    }
    defer nc.Close()

    pool, err := db.Connect(ctx, cfg.DatabaseURL, log)
    if err != nil {
        log.Fatal().Err(err).Msg("db failed")
    }
    defer pool.Close()

    // Wire dependencies (DI)
    repo := notesvectorize.NewNoteRepo(pool)
    ai := notesvectorize.NewAIClient(cfg.AIProviderURL)
    qdrant := notesvectorize.NewQdrantClient(cfg.QdrantURL)
    bus := notesvectorize.NewNATSBus(nc.JetStream)

    proc := notesvectorize.NewProcessor(log, repo, ai, qdrant, bus)

    // Subscribe via contracts
    // Воркер подписан на 2 subject: notes.created (event) и notes.updated (signal)
    subjects := []string{
        contracts.EventSubject("notes", "note", "created"),
        contracts.SignalSubject("notes", "note", "updated"),
    }

    consumer, err := nc.JS.CreateOrUpdateConsumer(ctx, "EVENTS", jetstream.ConsumerConfig{
        Name:          "notes-vectorize-events",
        Durable:       "notes-vectorize-events",
        FilterSubject: "event.v1.notes.note.>", // слушаем только events для notes
        AckPolicy:     jetstream.AckExplicitPolicy,
        MaxAckPending: 50,
    })
    if err != nil {
        log.Fatal().Err(err).Msg("events consumer failed")
    }

    // Для signals — отдельный consumer на SIGNALS stream
    signalsConsumer, err := nc.JS.CreateOrUpdateConsumer(ctx, "SIGNALS", jetstream.ConsumerConfig{
        Name:          "notes-vectorize-signals",
        Durable:       "notes-vectorize-signals",
        FilterSubject: "signal.v1.notes.note.updated",
        AckPolicy:     jetstream.AckExplicitPolicy,
        MaxAckPending: 50,
    })
    if err != nil {
        log.Fatal().Err(err).Msg("signals consumer failed")
    }

    // Базовый воркер
    w := worker.New(log)

    // Универсальный handler: распаковывает event.Event, вызывает processor
    handler := func(msg jetstream.Msg) {
        evt, err := event.Parse(msg.Data())
        if err != nil {
            log.Error().Err(err).Msg("parse event failed")
            msg.Term() // битый JSON, не ретраить
            return
        }

        if err := proc.Process(ctx, evt); err != nil {
            log.Error().Err(err).Str("event_id", evt.ID).Msg("process failed")
            msg.Nak() // transient — retry
            return
        }

        msg.Ack()
    }

    if err := w.Start(consumer, handler); err != nil {
        log.Fatal().Err(err).Msg("events worker start failed")
    }
    if err := w.Start(signalsConsumer, handler); err != nil {
        log.Fatal().Err(err).Msg("signals worker start failed")
    }

    log.Info().Strs("subjects", subjects).Msg("notes-vectorize worker ready")

    sig := make(chan os.Signal, 1)
    signal.Notify(sig, syscall.SIGINT, syscall.SIGTERM)
    <-sig

    w.Shutdown()
}
```

---

## Шаг 9. Запустить воркер

```bash
make run-worker NAME=notes-vectorize
# или
go run ./cmd/workers/notes-vectorize
```

Вывод:
```
INFO notes-vectorize worker ready subjects=[event.v1.notes.note.created signal.v1.notes.note.updated]
```

---

## Шаг 10. Проверка end-to-end

### Создать заметку через PostgREST

```bash
curl -X POST http://localhost:3000/notes \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{"title": "Test note", "content": "Hello world"}'
```

### Что происходит

```
1. KrakenD → PostgREST → INSERT INTO notes
2. Триггер log_activity() → INSERT в activity_logs
3. WAL → air_activity_service
4. CacheInvalidator: SET tenant_42:notes:<uuid> {...}
5. NatsConsumer: publish event.v1.notes.note.created
6. notes-vectorize воркер получает event.v1.notes.note.created
7. processor.Process():
   - repo.GetNote → {"id":"...", "title":"Test note", "content":"Hello world"}
   - ai.Embed → [0.1, 0.2, ...]
   - qdrant.Upsert
   - bus.Publish(signal.v1.notes.note.vectorized, {...})
8. air_hb_beta получает signal.v1.notes.note.vectorized → broadcast WS
9. Next.js видит signal → обновляет UI ("Vectorized" badge)
10. Search indexer (если есть) тоже подписан → обновляет Meilisearch
```

### Проверить NATS subjects

```bash
nats sub "signal.v1.notes.note.>"
# Получишь: signal.v1.notes.note.vectorized с payload
```

---

## Что НЕ нужно было делать

Весь фокус в том, чего **не надо** делать при добавлении нового воркера:

- ❌ Не править `internal/contracts/contracts.go` — функция `Subject` уже умеет строить любой subject
- ❌ Не править `air_activity_service` — он сам подхватит новый table descriptor из `contract_mappings`
- ❌ Не править другие воркеры — они подпишутся, если им нужно
- ❌ Не править `cmd/services/event-router/` — он не знает про конкретные воркеры
- ❌ Не править `air_hb_beta` — он подписан на wildcard
- ❌ Не писать SQL миграции под каждый contract — sync-contracts сам обновит `contract_mappings`
- ❌ Не хардкодить subject строки в коде — только через `contracts.*Subject()`

---

## Эволюция воркера

### Добавить новый триггер (например, `notes.deleted` для cleanup Qdrant)

1. В `worker.json` добавить `"notes.deleted"` в `subscribes`
2. В `processor.go` добавить ветку обработки `if evt.Action == "deleted" { ... cleanup ... }`
3. `make sync-contracts` — не нужен, contract уже есть
4. Перезапустить воркер

### Добавить новый domain event (например, `notes.reindexed`)

1. Создать `contracts/notes.reindexed.json`
2. Создать `contracts/notes.reindexed.schema.json`
3. В `worker.json` добавить `"notes.reindexed"` в `publishes`
4. В `processor.go` вызвать `contracts.SignalSubject("notes", "note", "reindexed")` + `bus.Publish()`
5. `make sync-contracts` → preview → apply
6. Перезапустить воркер

### Переименовать domain event (`notes.vectorized` → `notes.embedding_ready`)

1. Переименовать `contracts/notes.vectorized.json` → `contracts/notes.embedding_ready.json`
2. Внутри поменять `contract_id` и `action`
3. В `worker.json` обновить `publishes`
4. В `processor.go` обновить вызов `contracts.SignalSubject("notes", "note", "embedding_ready")`
5. `make sync-contracts` → preview покажет:
   ```
   - notes.vectorized (removed)
   + notes.embedding_ready (new)
   ```
6. Apply — sync script сделает `DELETE` + `INSERT` в `contract_mappings` (или `UPDATE` с bump version)

---

## Чеклист добавления воркера

- [ ] Создал `workers/<name>/` папку
- [ ] Написал `worker.json` (id, module, name, description, subscribes, publishes, tags)
- [ ] Создал domain contracts в `workers/<name>/contracts/*.json` для каждого `publishes`
- [ ] Создал `*.schema.json` для каждого domain contract
- [ ] Создал `*.ui.json` для каждого `ui_visible: true` контракта
- [ ] Убедился, что все `subscribes` ссылаются на существующие контракты (CDC table descriptors в `internal/contracts/tables/` или domain contracts других воркеров)
- [ ] Если есть новая таблица — добавил table descriptor в `internal/contracts/tables/` и зарегистрировал в `registry.json`
- [ ] Добавил воркер в `registry.json` как `{"type": "worker", "path": "..."}`
- [ ] Написал `processor.go` (DI, бизнес-логика, публикация через `contracts.*`)
- [ ] Написал `cmd/workers/<name>/main.go` (инициализация, подписки, graceful shutdown)
- [ ] Запустил `make sync-contracts` → проверил preview → apply
- [ ] Запустил воркер `make run-worker NAME=<name>`
- [ ] Создал тестовую сущность → увидел весь flow в логах

---

## Cross-references

- **Как строятся subject-ы** → `METAMODEL.md` § 9 (Subject construction pipeline)
- **Правила публикации domain events** → `COMMUNICATION_MODEL.md` § 2 (Двухслойная модель)
- **Contracts API** → `CONTRACTS_REFERENCE.md`
- **Сквозные трейсы** → `WIRING_DIAGRAMS.md`
