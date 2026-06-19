# WIRING_DIAGRAMS.md

> **Сквозные трейсы.** 6 диаграмм показывают, как система работает целиком — от HTTP-запроса до обновления UI, от пустой папки до работающего воркера, от команды пользователя до её завершения.

---

## TL;DR

6 диаграмм:
- **A.** CRUD-путь (PostgREST → DB → WAL → activity_service → UI)
- **B.** Event-driven путь (HTTP → Event Router → Command → Worker → DB → Signal → UI)
- **C.** Добавление нового воркера (от папки до работающей системы)
- **D.** Long-running process (reports.generate → completed / failed)
- **E.** Cross-module communication через entity_links
- **F.** Cache invalidation (WAL → cache_invalidator → Redis → UI pull)

Каждая диаграмма показывает: компоненты, последовательность, payload, тайминги, точки отказа.

---

## Diagram A. CRUD-путь (создание заметки через PostgREST)

### Сценарий

Пользователь создаёт заметку через UI. Простая операция без бизнес-логики — хватает PostgREST.

### Компоненты

```
Next.js → KrakenD → PostgREST → PostgreSQL → activity_logs
                                              ↓
                                          WAL (publication: activity_pub)
                                              ↓
                                    air_activity_service
                                              ↓
                          ┌──────────────────┼──────────────────┐
                          ↓                  ↓                  ↓
                      Redis              NATS              Phoenix WS
                                              ↓                  ↓
                                          Go workers         Next.js
                                              ↓
                                          (нет воркеров
                                           для notes.created
                                           в этом сценарии)
```

### Последовательность (19 шагов)

```
[Next.js]              [KrakenD]            [PostgREST]         [PostgreSQL]         [air_activity]       [NATS]              [Redis]            [air_hb_beta]      [Next.js]

   |                       |                     |                   |                    |                  |                    |                    |                  |
   |-- 1. POST /notes ---->|                     |                   |                    |                  |                    |                    |                  |
   |   {title, content}    |                     |                   |                    |                  |                    |                    |                  |
   |   + JWT cookie        |                     |                   |                    |                  |                    |                    |                  |
   |                       |                     |                   |                    |                  |                    |                    |                  |
   |                       |-- 2. Validate JWT ->|                   |                    |                  |                    |                    |                  |
   |                       |    Extract tenant_id|                   |                    |                  |                    |                    |                  |
   |                       |    workspace_id     |                   |                    |                  |                    |                    |                  |
   |                       |    from claims      |                   |                    |                  |                    |                    |                  |
   |                       |                     |                   |                    |                  |                    |                    |                  |
   |                       |-- 3. Set RLS ctx --->|                   |                    |                  |                    |                    |                  |
   |                       |    Forward POST     |                   |                    |                  |                    |                    |                  |
   |                       |    X-User-Id header |                   |                    |                  |                    |                    |                  |
   |                       |                     |                   |                    |                  |                    |                    |                  |
   |                       |                     |-- 4. INSERT notes->|                   |                  |                    |                    |                  |
   |                       |                     |    SET LOCAL       |                    |                  |                    |                    |                  |
   |                       |                     |    app.current_   |                    |                  |                    |                    |                  |
   |                       |                     |    tenant_id       |                    |                  |                    |                    |                  |
   |                       |                     |    app.current_   |                    |                  |                    |                    |                  |
   |                       |                     |    user_id         |                    |                  |                    |                    |                  |
   |                       |                     |                   |                    |                  |                    |                    |                  |
   |                       |                     |                   |-- 5. INSERT note -->|                  |                    |                    |                  |
   |                       |                     |                   |    row in 'notes'   |                  |                    |                    |                  |
   |                       |                     |                   |                    |                  |                    |                    |                  |
   |                       |                     |                   |-- 6. trg_notes_    ->|                  |                    |                    |                  |
   |                       |                     |                   |    activity AFTER  |                  |                    |                    |                  |
   |                       |                     |                   |    INSERT          |                  |                    |                    |                  |
   |                       |                     |                   |    log_activity()  |                  |                    |                    |                  |
   |                       |                     |                   |                    |                  |                    |                    |                  |
   |                       |                     |                   |-- 7. INSERT       ->|                  |                    |                    |                  |
   |                       |                     |                   |    activity_logs  |                  |                    |                    |                  |
   |                       |                     |                   |    (action=       |                  |                    |                    |                  |
   |                       |                     |                   |     'created',    |                  |                    |                    |                  |
   |                       |                     |                   |     new_data=     |                  |                    |                    |                  |
   |                       |                     |                   |     to_jsonb(NEW))|                  |                    |                    |                  |
   |                       |                     |                   |                    |                  |                    |                    |                  |
   |<-- 8. 201 Created ----|<-- 8. 201 Created --|<-- 8. RETURNING --|                    |                  |                    |                    |                  |
   |   {id, title, ...}    |                     |    id, title      |                    |                  |                    |                    |                  |
   |   (Optimistic UI      |                     |                   |                    |                  |                    |                    |                  |
   |    shows "creating")  |                     |                   |                    |                  |                    |                    |                  |
   |                       |                     |                   |                    |                  |                    |                    |                  |
   |                       |                     |                   |-- 9. WAL stream -->|                  |                    |                    |                  |
   |                       |                     |                   |    (publication:   |                  |                    |                    |                  |
   |                       |                     |                   |     activity_pub)  |                  |                    |                    |                  |
   |                       |                     |                   |    slot:           |                  |                    |                    |                  |
   |                       |                     |                   |     activity_slot  |                  |                    |                    |                  |
   |                       |                     |                   |     _v1            |                  |                    |                    |                  |
   |                       |                     |                   |                    |                  |                    |                    |                  |
   |                       |                     |                   |                    |-- 10. Decode ---->|                    |                    |                  |
   |                       |                     |                   |                    |    pgoutput      |                    |                    |                  |
   |                       |                     |                   |                    |-- 11. Normalize ->|                    |                    |                  |
   |                       |                     |                   |                    |    :insert →     |                    |                    |                  |
   |                       |                     |                   |                    |    "created"     |                    |                    |                  |
   |                       |                     |                   |                    |-- 12. Lookup ---->|                    |                    |                  |
   |                       |                     |                   |                    |    contract_     |                    |                    |                  |
   |                       |                     |                   |                    |    mappings:    |                    |                    |                  |
   |                       |                     |                   |                    |    "notes" →     |                    |                    |                  |
   |                       |                     |                   |                    |    {domain:      |                    |                    |                  |
   |                       |                     |                   |                    |     "notes",    |                    |                    |                  |
   |                       |                     |                   |                    |     entity:      |                    |                    |                  |
   |                       |                     |                   |                    |     "note"}      |                    |                    |                  |
   |                       |                     |                   |                    |-- 13. Build ------>|                    |                    |                  |
   |                       |                     |                   |                    |    subject:      |                    |                    |                  |
   |                       |                     |                   |                    |    event.v1.     |                    |                    |                  |
   |                       |                     |                   |                    |    notes.note.   |                    |                    |                  |
   |                       |                     |                   |                    |    created       |                    |                    |                  |
   |                       |                     |                   |                    |                  |                    |                    |                  |
   |                       |                     |                   |                    |-- 14a. Publish -->|-- 14b. event. ---->|                    |                  |
   |                       |                     |                   |                    |    to NATS       |    v1.notes.note.  |                    |                  |
   |                       |                     |                   |                    |    (async)       |    created         |                    |                  |
   |                       |                     |                   |                    |                  |                    |                    |                  |
   |                       |                     |                   |                    |-- 15a. SET ------>|--------------------|-- 15b. tenant_42:>|                    |                  |
   |                       |                     |                   |                    |    Redis cache   |                    |    notes:<uuid>   |                    |                  |
   |                       |                     |                   |                    |    key:          |                    |    = {id, title,  |                    |                  |
   |                       |                     |                   |                    |    {tenant_id}:  |                    |     content, ...} |                    |                  |
   |                       |                     |                   |                    |    {table}:      |                    |    EX 900         |                    |                  |
   |                       |                     |                   |                    |    {entity_id}   |                    |                    |                    |                  |
   |                       |                     |                   |                    |    new_data      |                    |                    |                    |                  |
   |                       |                     |                   |                    |    EX 900        |                    |                    |                    |                  |
   |                       |                     |                   |                    |                  |                    |                    |                    |                  |
   |                       |                     |                   |                    |-- 16a. Broadcast->|--------------------|--------------------|-- 16b. WS push -->|                  |
   |                       |                     |                   |                    |    to Phoenix    |                    |                    |    signal:        |                  |
   |                       |                     |                   |                    |    Channel       |                    |                    |    "notes.created"|                  |
   |                       |                     |                   |                    |                  |                    |                    |    sanitized:     |                  |
   |                       |                     |                   |                    |                  |                    |                    |    {entity_id,    |                  |
   |                       |                     |                   |                    |                  |                    |                    |     action}       |                  |
   |                       |                     |                   |                    |                  |                    |                    |    (no raw data)  |                  |
   |                       |                     |                   |                    |                  |                    |                    |                    |                  |
   |                       |                     |                   |                    |                  |                    |                    |                    |-- 17. signal -->|
   |                       |                     |                   |                    |                  |                    |                    |                    |    received     |
   |                       |                     |                   |                    |                  |                    |                    |                    |    via WS       |
   |                       |                     |                   |                    |                  |                    |                    |                    |                  |
   |<-- 18. UI transitions to "created" state, ---------------------------------- (if actor_id == current user) ----------------------|                    |                  |
   |    React Query invalidates [notes] query    |                   |                    |                  |                    |                    |                    |                  |
   |                       |                     |                   |                    |                  |                    |                    |                    |                  |
   |-- 19. GET /notes/<id> (cache invalidation ->|                   |                    |                  |                    |                    |                    |                  |
   |    React Query refetch, gets fresh data)    |                   |                    |                  |                    |                    |                    |                  |
   |                       |                     |                   |                    |                  |                    |                    |                    |                  |
   |<-- 19b. From Redis (cache hit, no DB hit) --|<-- 19b. From Redis|<-- 19c. SELECT ---->|                  |                    |                    |                    |                  |
   |    {id, title, content, ...}                |    (cache hit)    |    (cache miss)    |                  |                    |                    |                    |                  |
```

### Тайминги

| Шаг | Компонент | Latency |
|-----|-----------|---------|
| 1-3 | KrakenD auth + routing | <5ms |
| 4-7 | PostgREST + INSERT + trigger | 5-20ms |
| 8 | Response to client | total ~25ms |
| 9 | WAL write (async) | <1ms |
| 10-13 | air_activity_service decode | 5-15ms |
| 14-16 | NATS + Redis + WS publish (parallel) | 5-10ms |
| 17-19 | UI update via WS + cache refetch | 10-30ms |

**End-to-end: ~50ms** от HTTP-запроса до обновления UI у того же пользователя.

### Payload на 14b (NATS)

```json
{
  "message_id": "evt_01901a5b-...",
  "message_type": "event",
  "version": "v1",
  "tenant_id": "tenant_42_uuid",
  "workspace_id": "ws_abc_uuid",
  "user_id": "profile_xyz_uuid",
  "timestamp": "2026-06-19T12:34:56.789Z",
  "entity_type": "notes",
  "entity_id": "note_uuid_123",
  "action": "created",
  "new_data": {
    "id": "note_uuid_123",
    "title": "My note",
    "status": "active",
    "updated_at": "2026-06-19T12:34:56Z"
  }
}
```

### Точки отказа

| Что упало | Что происходит |
|-----------|----------------|
| PostgREST | 500 клиенту, заметка не создана |
| Trigger log_activity | INSERT откатывается, заметка не создана (триггер AFTER INSERT, ошибка → rollback) |
| air_activity_service | WAL копится, алерт при lag > 1GB. Заметка в БД есть, UI не получит сигнал |
| NATS | air_activity_service retry, event не потерян (WAL хранится). UI не обновится пока NATS не поднимется |
| Redis | Cache miss на следующем GET → идёт в БД. Работает, но медленнее |
| air_hb_beta (WS) | Сигнал в NATS есть. UI обновится при следующем reconnect |

### Что НЕ происходит

- ❌ Никакой воркер не запускается (нет подписчиков на `notes.created` в этом сценарии)
- ❌ Никакая AI-обработка не запускается
- ❌ Никакой domain event не публикуется

Если бы был воркер `notes-vectorize`, он бы запустился на шаге 14b — но это уже Diagram B.

---

## Diagram B. Event-driven путь (команда от пользователя)

### Сценарий

Пользователь нажимает «Generate Report» в UI. Это сложная операция — пойдёт через Event Router как Command, воркер будет работать 10+ секунд.

### Компоненты

```
Next.js → KrakenD → Event Router → NATS (COMMANDS) → Report Worker
                                                          ↓
                                              PostgreSQL (INSERT reports)
                                                          ↓
                                              WAL → air_activity_service
                                                          ↓
                                          ┌───────────────┼───────────────┐
                                          ↓               ↓               ↓
                                      NATS event      Redis cache     Phoenix WS
                                          ↓
                                      UI: "in progress"
                                          
[параллельно, через 10 секунд]
Report Worker → UPDATE reports SET status=completed
                                                          ↓
                                              WAL → air_activity_service
                                                          ↓
                                          ┌───────────────┼───────────────┐
                                          ↓               ↓               ↓
                                      NATS signal      Redis cache     Phoenix WS
                                          ↓
                                      UI: "ready"
                                          
Report Worker → publish domain event: signal.v1.reports.report.completed
                                          ↓
                                      UI: "completed" (через broadcaster)
                                      Search indexer: reindex (если подписан)
```

### Последовательность

```
[Next.js]              [KrakenD]            [Event Router]      [NATS]              [Report Worker]     [PostgreSQL]        [air_activity]      [Redis]            [air_hb_beta]      [Next.js]

   |                       |                     |                  |                    |                  |                    |                  |                    |                  |
   |-- 1. POST /events --->|                     |                  |                    |                  |                    |                  |                    |                  |
   |   {event_type:        |                     |                  |                    |                  |                    |                  |                    |                  |
   |    "reports.generate",|                     |                  |                    |                  |                    |                  |                    |                  |
   |    payload: {type,    |                     |                  |                    |                  |                    |                  |                    |                  |
   |    date_range}}       |                     |                  |                    |                  |                    |                  |                    |                  |
   |   + JWT cookie        |                     |                  |                    |                  |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |                    |                  |                    |                  |
   |                       |-- 2. Validate JWT ->|                  |                    |                  |                    |                  |                    |                  |
   |                       |    Extract claims   |                  |                    |                  |                    |                  |                    |                  |
   |                       |-- 3. Forward POST ->|                  |                    |                  |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |                    |                  |                    |                  |
   |                       |                     |-- 4. Validate -->|                    |                  |                    |                  |                    |                  |
   |                       |                     |    event_type    |                    |                  |                    |                  |                    |                  |
   |                       |                     |    not empty     |                    |                  |                    |                  |                    |                  |
   |                       |                     |    UUIDs valid   |                    |                  |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |                    |                  |                    |                  |
   |                       |                     |-- 5. Enrich ----->|                    |                  |                    |                  |                    |                  |
   |                       |                     |    evt.ID =      |                    |                  |                    |                  |                    |                  |
   |                       |                     |    "evt_"+uuid   |                    |                  |                    |                  |                    |                  |
   |                       |                     |    evt.Timestamp |                    |                  |                    |                  |                    |                  |
   |                       |                     |    = now()       |                    |                  |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |                    |                  |                    |                  |
   |                       |                     |-- 6. Resolve --->|                    |                  |                    |                  |                    |                  |
   |                       |                     |    routes.yaml:  |                    |                  |                    |                  |                    |                  |
   |                       |                     |    reports.      |                    |                  |                    |                  |                    |                  |
   |                       |                     |    generate →    |                    |                  |                    |                  |                    |                  |
   |                       |                     |    command.v1.   |                    |                  |                    |                  |                    |                  |
   |                       |                     |    reports.      |                    |                  |                    |                  |                    |                  |
   |                       |                     |    report.       |                    |                  |                    |                  |                    |                  |
   |                       |                     |    generate      |                    |                  |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |                    |                  |                    |                  |
   |                       |                     |-- 7. Publish ---->|                    |                  |                    |                  |                    |                  |
   |                       |                     |    to COMMANDS   |                    |                  |                    |                  |                    |                  |
   |                       |                     |    stream        |                    |                  |                    |                  |                    |                  |
   |<-- 8. 202 Accepted ---|<-- 8. 202 ----------|                  |                    |                  |                    |                  |                    |                  |
   |   {event_id, status:  |                     |                  |                    |                  |                    |                  |                    |                  |
   |    "accepted"}        |                     |                  |                    |                  |                    |                  |                    |                  |
   |   (UI shows           |                     |                  |                    |                  |                    |                  |                    |                  |
   |    "generating...")   |                     |                  |                    |                  |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |                    |                  |                    |                  |
   |                       |                     |                  |-- 9. Deliver ----->|                  |                    |                  |                    |                  |
   |                       |                     |                  |    command to      |                  |                    |                  |                    |                  |
   |                       |                     |                  |    report-worker   |                  |                    |                  |                    |                  |
   |                       |                     |                  |    (queue group)   |                  |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |                    |                  |                    |                  |
   |                       |                     |                  |                    |-- 10. Parse ----->|                    |                  |                    |                  |
   |                       |                     |                  |                    |    command       |                    |                  |                    |                  |
   |                       |                     |                  |                    |    DDD-validate  |                    |                  |                    |                  |
   |                       |                     |                  |                    |    (rights,      |                    |                  |                    |                  |
   |                       |                     |                  |                    |    limits)       |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |                    |                  |                    |                  |
   |                       |                     |                  |                    |-- 11. INSERT ---->|                    |                  |                    |                  |
   |                       |                     |                  |                    |    INTO reports  |                    |                  |                    |                  |
   |                       |                     |                  |                    |    (status=      |                    |                  |                    |                  |
   |                       |                     |                  |                    |     'pending',   |                    |                  |                    |                  |
   |                       |                     |                  |                    |     payload=...) |                    |                  |                    |                  |
   |                       |                     |                  |                    |    SET LOCAL     |                    |                  |                    |                  |
   |                       |                     |                  |                    |    app.current_*|                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |-- 12. trg_ ------->|                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |    reports_       |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |    activity       |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |    AFTER INSERT   |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |    log_activity() |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |-- 13. INSERT ----->|                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |    activity_logs  |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |    (action=       |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |     'created')    |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |                    |                  |                    |                  |
   |                       |                     |                  |                    |<-- 14. Ack ------|                    |                  |                    |                  |
   |                       |                     |                  |<-- 14. Ack --------|                    |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |-- 15. WAL ------->|                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |    stream         |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |                    |-- 16. Decode --->|                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |                    |    + normalize   |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |                    |    + lookup      |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |                    |    mapping       |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |                    |-- 17a. Publish ->|-- 17b. event. --->|                    |                  |
   |                       |                     |                  |                    |                  |                    |    to NATS       |    v1.reports.    |                    |                  |
   |                       |                     |                  |                    |                  |                    |                  |    report.created |                    |                  |
   |                       |                     |                  |                    |                  |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |                    |-- 18a. SET ----->|--------------------|-- 18b. cache -->|
   |                       |                     |                  |                    |                  |                    |    Redis         |                    |    tenant_42:    |                  |
   |                       |                     |                  |                    |                  |                    |                  |                    |    reports:<id>  |                  |
   |                       |                     |                  |                    |                  |                    |                  |                    |    = {status:    |                  |
   |                       |                     |                  |                    |                  |                    |                  |                    |     'pending'}   |                  |
   |                       |                     |                  |                    |                  |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |                    |-- 19a. Broadcast>|--------------------|--------------------|-- 19b. WS ----->|
   |                       |                     |                  |                    |                  |                    |    to Phoenix    |                    |                    |    signal:       |
   |                       |                     |                  |                    |                  |                    |    Channel       |                    |                    |    "reports.     |
   |                       |                     |                  |                    |                  |                    |                  |                    |                    |    created"      |
   |                       |                     |                  |                    |                  |                    |                  |                    |                    |    (sanitized)   |
   |                       |                     |                  |                    |                  |                    |                  |                    |                    |                  |
   |<-- 20. UI: "Report created, status: pending" ----------------------------------------------------------------------------------------------------------------------------------|                    |                  |
   |    (React Query invalidates [reports] query)                                                                                                                                                       |                  |
   |                                                                                                                                                                                                     |                  |
   |                       |                     |                  |                    |                  |                    |                  |                    |                    |                  |
   |                  [Report Worker работает 10 секунд: собирает данные, генерирует PDF, ...]                                                                                                            |                  |
   |                       |                     |                  |                    |                  |                    |                  |                    |                    |                  |
   |                       |                     |                  |                    |-- 21. UPDATE ---->|                    |                  |                    |                  |
   |                       |                     |                  |                    |    reports SET   |                    |                  |                    |                  |
   |                       |                     |                  |                    |    status=       |                    |                  |                    |                  |
   |                       |                     |                  |                    |    'completed',  |                    |                  |                    |                  |
   |                       |                     |                  |                    |    content=...,  |                    |                  |                    |                  |
   |                       |                     |                  |                    |    file_url=...  |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |-- 22. trg_ ------->|                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |    reports_       |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |    activity       |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |    AFTER UPDATE   |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |    log_activity() |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |-- 23. INSERT ----->|                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |    activity_logs  |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |    (action=       |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |     'updated',    |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |    changes=       |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |    {status:...,   |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |     content:...}) |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |-- 24. WAL ------->|                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |                    |-- 25a. Publish ->|-- 25b. signal. -->|                    |                  |
   |                       |                     |                  |                    |                  |                    |                  |    v1.reports.    |                    |                  |
   |                       |                     |                  |                    |                  |                    |                  |    report.updated |                    |                  |
   |                       |                     |                  |                    |                  |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |                    |-- 26a. SET ----->|--------------------|-- 26b. cache -->|
   |                       |                     |                  |                    |                  |                    |    Redis         |                    |    updated       |                  |
   |                       |                     |                  |                    |                  |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |                    |-- 27a. Broadcast>|--------------------|--------------------|-- 27b. WS ----->|
   |                       |                     |                  |                    |                  |                    |                  |                    |                    |    signal:       |
   |                       |                     |                  |                    |                  |                    |                  |                    |                    |    "reports.     |
   |                       |                     |                  |                    |                  |                    |                  |                    |                    |    updated"      |
   |                       |                     |                  |                    |                  |                    |                  |                    |                    |                  |
   |                       |                     |                  |                    |-- 28. Publish --->|                    |                  |                    |                    |                  |
   |                       |                     |                  |                    |    domain event: |                    |                  |                    |                  |
   |                       |                     |                  |                    |    signal.v1.    |                    |                  |                    |                  |
   |                       |                     |                  |                    |    reports.      |                    |                  |                    |                  |
   |                       |                     |                  |                    |    report.       |                    |                  |                    |                  |
   |                       |                     |                  |                    |    completed     |                    |                  |                    |                  |
   |                       |                     |                  |                    |    (in code via  |                    |                  |                    |                  |
   |                       |                     |                  |                    |    contracts.    |                    |                  |                    |                  |
   |                       |                     |                  |                    |    SignalSubject)|                    |                  |                    |                  |
   |                       |                     |                  |<-- 28. published --|                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |                    |                  |                    |                  |
   |                       |                     |                  |                    |                  |                    |                  |                    |                    |                  |
   |<-- 29. UI: signal.v1.reports.report.completed received via WS -----------------------------------------------------------------------------------------------------------------------------------|                    |                  |
   |    UI updates to "Report ready" state                                                                                                                                                                                |                  |
   |    Shows download button for file_url                                                                                                                                                                                |                  |
   |                                                                                                                                                                                                                       |                  |
   |    [Search indexer (separate worker) subscribed to signal.reports.completed]                                                                                                                                          |                  |
   |    [Receives 28, reindexes report in Meilisearch]                                                                                                                                                                     |                  |
```

### Тайминги

| Фаза | Время |
|------|-------|
| 1-8 (HTTP roundtrip) | ~30ms |
| 9-14 (worker receives command, inserts) | ~50ms |
| 15-20 (WAL → UI: "pending") | ~30ms |
| 21 (worker генерирует отчёт) | **5-30 секунд** (бизнес-логика) |
| 22-27 (WAL → UI: "updated") | ~30ms |
| 28 (domain event publish) | ~5ms |
| 29 (UI: "completed") | ~10ms |

**End-to-end: ~30 секунд** (доминирует бизнес-логика генерации отчёта).

### Точка отказа: worker падает во время генерации

- Команда уже Ack'нута (на шаге 14)
- INSERT в `reports` со статусом `pending` уже в БД
- Worker упал → статус остался `pending` навсегда
- **Решение:** cron-воркер раз в 5 минут проверяет `reports WHERE status='pending' AND created_at < NOW() - INTERVAL '10 minutes'`, помечает как `failed` или перезапускает

### Точка отказа: NATS упал после Ack

- Worker сделал INSERT (шаг 11)
- WAL записан, но air_activity_service не может опубликовать (NATS упал)
- Сигнал `reports.created` не уйдёт, UI не узнает о создании
- **Решение:** air_activity_service retry с backoff, NATS поднимется — сигнал уйдёт. WAL хранится в слоте replication, не теряется.

---

## Diagram C. Добавление нового воркера

### Сценарий

Разработчик хочет добавить воркер `notes-vectorize`, который после создания заметки делает embedding и публикует domain event `notes.vectorized`.

### Последовательность

```
[Developer]            [Filesystem]         [registry.json]      [sync-contracts]      [contract_mappings]  [air_activity]      [NATS]              [Worker process]

   |                       |                     |                    |                    |                  |                    |                    |
   |-- 1. mkdir ---------->|                     |                    |                    |                  |                    |                    |
   |    workers/           |                     |                    |                    |                  |                    |                    |
   |    notes-vectorize/   |                     |                    |                    |                  |                    |                    |
   |                       |                     |                    |                    |                  |                    |                    |
   |-- 2. Write ---------->|                     |                    |                    |                  |                    |                    |
   |    worker.json        |                     |                    |                    |                  |                    |                    |
   |    (id, module,       |                     |                    |                    |                  |                    |                    |
   |    subscribes,        |                     |                    |                    |                  |                    |                    |
   |    publishes)         |                     |                    |                    |                  |                    |                    |
   |                       |                     |                    |                    |                  |                    |                    |
   |-- 3. Write ---------->|                     |                    |                    |                  |                    |                    |
   |    contracts/         |                     |                    |                    |                  |                    |                    |
   |    notes.vectorized.  |                     |                    |                    |                  |                    |                    |
   |    json               |                     |                    |                    |                  |                    |                    |
   |    + schema.json      |                     |                    |                    |                  |                    |                    |
   |    + ui.json          |                     |                    |                    |                  |                    |                    |
   |                       |                     |                    |                    |                  |                    |                    |
   |-- 4. Write ---------->|                     |                    |                    |                  |                    |                    |
   |    processor.go       |                     |                    |                    |                  |                    |                    |
   |    (DI, business      |                     |                    |                    |                  |                    |                    |
   |    logic, publishes   |                     |                    |                    |                  |                    |                    |
   |    via contracts.*)   |                     |                    |                    |                  |                    |                    |
   |                       |                     |                    |                    |                  |                    |                    |
   |-- 5. Write ---------->|                     |                    |                    |                  |                    |                    |
   |    cmd/workers/       |                     |                    |                    |                  |                    |                    |
   |    notes-vectorize/   |                     |                    |                    |                  |                    |                    |
   |    main.go            |                     |                    |                    |                  |                    |                    |
   |                       |                     |                    |                    |                  |                    |                    |
   |-- 6. Update --------->|                     |                    |                    |                  |                    |                    |
   |    registry.json      |                     |                    |                    |                  |                    |                    |
   |    (add artifact)     |                     |                    |                    |                  |                    |                    |
   |                       |                     |                    |                    |                  |                    |                    |
   |-- 7. make ------------>|-------------------->|-------------------->|                    |                  |                    |                    |
   |    sync-contracts     |                     |                    |                    |                  |                    |                    |
   |                       |                     |                    |-- 8. Scan -------->|                  |                    |                    |
   |                       |                     |                    |    workers/*/      |                  |                    |                    |
   |                       |                     |                    |    contracts/*.json|                  |                    |                    |
   |                       |                     |                    |    + internal/     |                  |                    |                    |
   |                       |                     |                    |    contracts/      |                  |                    |                    |
   |                       |                     |                    |    tables/*.json   |                  |                    |                    |
   |                       |                     |                    |                    |                  |                    |                    |
   |                       |                     |                    |-- 9. Read current ->|                  |                    |                    |
   |                       |                     |                    |    state from DB   |                  |                    |                    |
   |                       |                     |                    |                    |                  |                    |                    |
   |                       |                     |                    |<-- 9b. rows -------|                  |                    |                    |
   |                       |                     |                    |                    |                  |                    |                    |
   |                       |                     |                    |-- 10. Diff ------->|                  |                    |                    |
   |                       |                     |                    |    (new/changed/   |                  |                    |                    |
   |                       |                     |                    |    removed)        |                  |                    |                    |
   |                       |                     |                    |                    |                  |                    |                    |
   |<-- 11. Preview -------|--------------------|--------------------|                    |                  |                    |                    |
   |    + notes.vectorized |                     |                    |                    |                  |                    |                    |
   |      (signal, domain) |                     |                    |                    |                  |                    |                    |
   |      Subject:         |                     |                    |                    |                  |                    |                    |
   |      signal.v1.notes. |                     |                    |                    |                  |                    |                    |
   |      note.vectorized  |                     |                    |                    |                  |                    |                    |
   |    Apply? [y/N]:      |                     |                    |                    |                  |                    |                    |
   |                       |                     |                    |                    |                  |                    |                    |
   |-- 12. y ------------->|-------------------->|-------------------->|                    |                  |                    |                    |
   |                       |                     |                    |-- 13. INSERT ----->|                  |                    |                    |
   |                       |                     |                    |    INTO            |                  |                    |                    |
   |                       |                     |                    |    contract_       |                  |                    |                    |
   |                       |                     |                    |    mappings        |                  |                    |                    |
   |                       |                     |                    |                    |                  |                    |                    |
   |                       |                     |                    |-- 14. LISTEN/ ---->|                  |                    |                    |
   |                       |                     |                    |    NOTIFY (or      |                  |                    |                    |
   |                       |                     |                    |    polling)        |                  |                    |                    |
   |                       |                     |                    |                    |                  |                    |                    |
   |                       |                     |                    |                    |-- 15. Reload --->|                    |                    |
   |                       |                     |                    |                    |    ETS cache     |                    |                    |
   |                       |                     |                    |                    |    (now knows    |                    |                    |
   |                       |                     |                    |                    |    "notes" →     |                    |                    |
   |                       |                     |                    |                    |    {notes,note}) |                    |                    |
   |                       |                     |                    |                    |                  |                    |                    |
   |-- 16. make ---------->|                     |                    |                    |                  |                    |                    |
   |    run-worker NAME=   |                     |                    |                    |                  |                    |                    |
   |    notes-vectorize    |                     |                    |                    |                  |                    |                    |
   |                       |                     |                    |                    |                  |                    |                    |
   |                       |                     |                    |                    |                  |                    |-- 17. Start ----->|
   |                       |                     |                    |                    |                  |                    |    process        |
   |                       |                     |                    |                    |                  |                    |                    |
   |                       |                     |                    |                    |                  |                    |    <-- 18. Lookup contract_mappings for
   |                       |                     |                    |                    |                  |                    |        "notes.created" → {event, notes, note, created}
   |                       |                     |                    |                    |                  |                    |        "notes.updated" → {signal, notes, note, updated}
   |                       |                     |                    |                    |                  |                    |                    |
   |                       |                     |                    |                    |                  |                    |    <-- 19. Build subjects via contracts.EventSubject/SignalSubject
   |                       |                     |                    |                    |                  |                    |        event.v1.notes.note.created
   |                       |                     |                    |                    |                  |                    |        signal.v1.notes.note.updated
   |                       |                     |                    |                    |                  |                    |                    |
   |                       |                     |                    |                    |                  |                    |    <-- 20. Create JetStream consumers
   |                       |                     |                    |                    |                  |                    |        EVENTS stream: FilterSubject=event.v1.notes.note.created
   |                       |                     |                    |                    |                  |                    |        SIGNALS stream: FilterSubject=signal.v1.notes.note.updated
   |                       |                     |                    |                    |                  |                    |                    |
   |                       |                     |                    |                    |                  |                    |    <-- 21. Subscribe (consumer.Consume(handler))
   |                       |                     |                    |                    |                  |                    |                    |
   |<-- 22. "notes-vectorize worker ready" ------|--------------------|--------------------|--------------------|--------------------|--------------------|                    |
   |    subjects=[event.v1.notes.note.created, signal.v1.notes.note.updated]                                                                                                                |
   |                                                                                                                                                                                         |
   |    [System is now ready. Next notes.created will trigger vectorization.]                                                                                                                |
```

### Что НЕ нужно было делать

- ❌ Не править `internal/contracts/contracts.go` — функция `Subject()` уже умеет строить любой subject
- ❌ Не править `air_activity_service` — он подхватит новый table descriptor из `contract_mappings`
- ❌ Не править другие воркеры — они подпишутся, если им нужно
- ❌ Не править `cmd/services/event-router/` — он не знает про конкретные воркеры
- ❌ Не править `air_hb_beta` — он подписан на wildcard
- ❌ Не писать SQL миграции под каждый contract — sync-contracts сам обновит `contract_mappings`

---

## Diagram D. Long-running process с failure handling

### Сценарий

Пользователь запускает AI-анализ документа. Воркер должен:
1. Скачать документ из file-storage
2. Отправить в ai-provider
3. Дождаться результата (до 5 минут)
4. Сохранить результат в БД
5. Опубликовать domain event `documents.analyzed` или `documents.analysis_failed`

### Последовательность (success path)

```
[UI]                  [Event Router]      [NATS]              [AI-Analysis Worker]    [file-storage]    [ai-provider]    [PostgreSQL]      [WAL → NATS]

   |-- 1. POST /events ->|                 |                    |                       |                 |                |                  |
   |   command.v1.       |                 |                    |                       |                 |                |                  |
   |   documents.        |                 |                    |                       |                 |                |                  |
   |   document.analyze  |                 |                    |                       |                 |                |                  |
   |   {document_id}     |                 |                    |                       |                 |                |                  |
   |                     |-- 2. Publish -->|                    |                       |                 |                |                  |
   |<-- 3. 202 ----------|                 |                    |                       |                 |                |                  |
   |   (UI: "analyzing") |                 |                    |                       |                 |                |                  |
   |                     |                 |-- 4. Deliver ----->|                       |                 |                |                  |
   |                     |                 |                    |-- 5. INSERT -------->|                 |                |                  |
   |                     |                 |                    |    document_analyses  |                 |                |                  |
   |                     |                 |                    |    (status=pending)  |                 |                |                  |
   |                     |                 |                    |                       |                 |                |                  |
   |                     |                 |                    |                       |                 |                |                  |
   |                     |                 |                    |                  [WAL → air_activity → NATS event.document_analyses.created]                    |
   |                     |                 |                    |                       |                 |                |                  |
   |<-- 6. WS signal: document_analyses.created --------------------------------------------------------------------------------------------------------------------------------|
   |   (UI: "Analysis started")                                                                                                                                                |
   |                     |                 |                    |                       |                 |                |                  |
   |                     |                 |                    |-- 7. HTTP GET ------->|                 |                |                  |
   |                     |                 |                    |    /documents/{id}    |                 |                |                  |
   |                     |                 |                    |<-- 8. file bytes -----|                 |                |                  |
   |                     |                 |                    |                       |                 |                |                  |
   |                     |                 |                    |-- 9. HTTP POST ------>|                 |                |                  |
   |                     |                 |                    |    /v1/analyze        |                 |                |                  |
   |                     |                 |                    |    {content}          |                 |                |                  |
   |                     |                 |                    |                       |                 |                |                  |
   |                     |                 |                    |                  [AI Provider работает 2-5 минут]                                       |
   |                     |                 |                    |                       |                 |                |                  |
   |                     |                 |                    |<-- 10. Analysis -----|                 |                |                  |
   |                     |                 |                    |    result             |                 |                |                  |
   |                     |                 |                    |                       |                 |                |                  |
   |                     |                 |                    |-- 11. UPDATE -------->|                 |                |                  |
   |                     |                 |                    |    document_analyses  |                 |                |                  |
   |                     |                 |                    |    SET status=        |                 |                |                  |
   |                     |                 |                    |    'completed',       |                 |                |                  |
   |                     |                 |                    |    result=...         |                 |                |                  |
   |                     |                 |                    |                       |                 |                |                  |
   |                     |                 |                    |                  [WAL → air_activity → NATS signal.document_analyses.updated]                  |
   |                     |                 |                    |                       |                 |                |                  |
   |<-- 12. WS signal: document_analyses.updated (status=completed) -----------------------------------------------------------------------------------------------------------|
   |   (UI: "Analysis ready")                                                                                                                                                  |
   |                     |                 |                    |                       |                 |                |                  |
   |                     |                 |                    |-- 13. Publish ------->|                 |                |                  |
   |                     |                 |                    |    domain event:      |                 |                |                  |
   |                     |                 |                    |    signal.v1.         |                 |                |                  |
   |                     |                 |                    |    documents.         |                 |                |                  |
   |                     |                 |                    |    document.analyzed  |                 |                |                  |
   |                     |                 |                    |    (via contracts.   |                 |                |                  |
   |                     |                 |                    |    SignalSubject)    |                 |                |                  |
   |                     |                 |                    |                       |                 |                |                  |
   |<-- 14. WS signal: documents.document.analyzed (domain event) ------------------------------------------------------------------------------------------------------------|
   |   (UI: shows analysis result)                                                                                                                                             |
   |                                                                                                                                                                            |
   |   [Search indexer subscribed to signal.documents.document.analyzed → reindex document]                                                                                   |
```

### Failure path: ai-provider timeout

```
[шаги 1-9 те же]
   |                     |                 |                    |                       |                 |                |                  |
   |                     |                 |                    |                  [AI Provider не отвечает 5 минут]                                       |
   |                     |                 |                    |                       |                 |                |                  |
   |                     |                 |                    |-- 10f. UPDATE ------->|                 |                |                  |
   |                     |                 |                    |    document_analyses  |                 |                |                  |
   |                     |                 |                    |    SET status=        |                 |                |                  |
   |                     |                 |                    |    'failed',          |                 |                |                  |
   |                     |                 |                    |    error=             |                 |                |                  |
   |                     |                 |                    |    'ai_timeout'       |                 |                |                  |
   |                     |                 |                    |                       |                 |                |                  |
   |                     |                 |                    |                  [WAL → NATS signal.document_analyses.updated (status=failed)]                          |
   |                     |                 |                    |                       |                 |                |                  |
   |<-- 11f. WS signal: status=failed ----------------------------------------------------------------------------------------------------------------------|                  |
   |   (UI: "Analysis failed")                                                                                                                                                 |
   |                     |                 |                    |                       |                 |                |                  |
   |                     |                 |                    |-- 12f. Publish ------->|                 |                |                  |
   |                     |                 |                    |    domain event:      |                 |                |                  |
   |                     |                 |                    |    signal.v1.         |                 |                |                  |
   |                     |                 |                    |    documents.         |                 |                |                  |
   |                     |                 |                    |    document.          |                 |                |                  |
   |                     |                 |                    |    analysis_failed    |                 |                |                  |
   |                     |                 |                    |    (with reason)     |                 |                |                  |
   |                     |                 |                    |                       |                 |                |                  |
   |<-- 13f. WS signal: documents.document.analysis_failed (domain event) ---------------------------------------------------------------------------------------------------|
   |   (UI: shows error, offers retry button)                                                                                                                                  |
```

### Failure path: transient error (Nak + retry)

```
[шаги 1-4 те же]
   |                     |                 |                    |                       |                 |                |                  |
   |                     |                 |                    |-- 5t. INSERT -------->|                 |                |                  |
   |                     |                 |                    |    fails: DB          |                 |                |                  |
   |                     |                 |                    |    connection         |                 |                |                  |
   |                     |                 |                    |    timeout            |                 |                |                  |
   |                     |                 |                    |                       |                 |                |                  |
   |                     |                 |                    |<-- 5t. error          |                 |                |                  |
   |                     |                 |                    |                       |                 |                |                  |
   |                     |                 |                    |-- 6t. msg.Nak() ----->|                 |                |                  |
   |                     |                 |                    |    (transient,        |                 |                |                  |
   |                     |                 |                    |    will retry)        |                 |                |                  |
   |                     |                 |                    |                       |                 |                |                  |
   |                     |                 |<-- 6t. Nak --------|                       |                 |                |                  |
   |                     |                 |    (NATS will      |                       |                 |                |                  |
   |                     |                 |    redeliver in    |                       |                 |                |                  |
   |                     |                 |    ack_wait +      |                       |                 |                |                  |
   |                     |                 |    backoff)        |                       |                 |                |                  |
   |                     |                 |                    |                       |                 |                |                  |
   |                     |                 |-- 7t. Redeliver ->|                       |                 |                |                  |
   |                     |                 |    (after 30s)     |                       |                 |                |                  |
   |                     |                 |                    |-- 8t. INSERT (retry) ->|                |                |                  |
   |                     |                 |                    |    success            |                 |                |                  |
   |                     |                 |                    |                       |                 |                |                  |
   |                     |                 |                    |    [continue normal flow]               |                |                  |
```

### Failure path: permanent business error (Ack without retry)

```
[шаги 1-4 те же]
   |                     |                 |                    |                       |                 |                |                  |
   |                     |                 |                    |-- 5p. Validate ----->|                 |                |                  |
   |                     |                 |                    |    document_id       |                 |                |                  |
   |                     |                 |                    |    not found         |                 |                |                  |
   |                     |                 |                    |    in tenant         |                 |                |                  |
   |                     |                 |                    |                       |                 |                |                  |
   |                     |                 |                    |-- 6p. log.Warn() ---->|                 |                |                  |
   |                     |                 |                    |    "document not     |                 |                |                  |
   |                     |                 |                    |    found"            |                 |                |                  |
   |                     |                 |                    |                       |                 |                |                  |
   |                     |                 |                    |-- 7p. msg.Ack() ----->|                 |                |                  |
   |                     |                 |                    |    (permanent, no    |                 |                |                  |
   |                     |                 |                    |    retry)            |                 |                |                  |
   |                     |                 |                    |                       |                 |                |                  |
   |                     |                 |<-- 7p. Ack --------|                       |                 |                |                  |
   |                     |                 |    (no redelivery) |                       |                 |                |                  |
   |                     |                 |                    |                       |                 |                |                  |
   |   (UI waits, eventually shows "document not found" via separate status check)                                                                       |
```

### Правила выбора стратегии

| Тип ошибки | Пример | Стратегия | Domain event? |
|------------|--------|-----------|---------------|
| Transient | DB timeout, AI provider timeout, Redis down | `Nak()` + retry | ❌ Нет |
| Permanent business | Document not found, no rights, invalid ID | `Ack()` + log | ❌ Нет |
| Long-running success | Report generated, analysis done | `Ack()` + UPDATE + publish `*.completed` | ✅ Да |
| Long-running failure | AI provider failed after 5 min | `Ack()` + UPDATE status=failed + publish `*.failed` | ✅ Да |

---

## Diagram E. Cross-module communication через entity_links

### Сценарий

Пользователь привязывает заметку к задаче. Заметка в модуле `notes`, задача в модуле `tasks`. Модули не знают друг о друге — связь через `entity_links`.

### Компоненты

```
UI → KrakenD → PostgREST → entity_links (INTEGRATION table)
                                ↓
                            WAL → air_activity_service
                                ↓
                            NATS: signal.entity_links.created
                                ↓
                            UI: "Link created"
```

### SQL

```sql
-- entity_types реестр (заполнен один раз)
-- 4 = chat_room, 5 = note, 6 = task, 7 = project

-- Создать связь: note → task
INSERT INTO entity_links (
    tenant_id, workspace_id,
    source_type_id, source_id,
    target_type_id, target_id,
    relation_type,
    meta
) VALUES (
    'tenant_42_uuid', 'ws_abc_uuid',
    5, 'note_uuid_123',           -- source: note
    6, 'task_uuid_456',           -- target: task
    'attached',                   -- relation_type
    '{"context": "reference"}'::jsonb
);
```

### Запросы

```sql
-- Все задачи, привязанные к заметке
SELECT t.*
FROM entity_links l
JOIN tasks t ON t.id = l.target_id
WHERE l.source_type_id = 5
  AND l.source_id = $1     -- note_uuid
  AND l.target_type_id = 6
  AND l.tenant_id = $2;

-- Все заметки, привязанные к задаче
SELECT n.*
FROM entity_links l
JOIN notes n ON n.id = l.source_id
WHERE l.target_type_id = 6
  AND l.target_id = $1     -- task_uuid
  AND l.source_type_id = 5
  AND l.tenant_id = $2;

-- Все сущности, связанные с проектом (любой тип)
SELECT
    et.code AS entity_type,
    el.target_id AS entity_id,
    el.relation_type
FROM entity_links el
JOIN entity_types et ON et.id = el.target_type_id
WHERE el.source_type_id = 7
  AND el.source_id = $1     -- project_uuid
  AND el.tenant_id = $2;
```

### Сценарий: cascade при удалении

```
1. Пользователь удаляет заметку note_uuid_123
   ↓
2. DELETE FROM notes WHERE id = 'note_uuid_123'
   ↓
3. Триггер log_activity → activity_logs (action=deleted, entity_type=notes)
   ↓
4. WAL → air_activity_service → NATS signal.notes.note.deleted
   ↓
5. Cleanup worker (подписан на signal.notes.note.deleted):
   - DELETE FROM entity_links WHERE source_type_id=5 AND source_id=$1
   - DELETE FROM entity_links WHERE target_type_id=5 AND target_id=$1
   - (или soft delete: UPDATE entity_links SET meta=meta||'{"deleted":true}'::jsonb)
   ↓
6. entity_links изменён → WAL → NATS signal.entity_links.updated
   ↓
7. UI видит: связи удалены, обновляет интерфейс
```

### Почему entity_links, а не FK

| Подход | Плюс | Минус |
|--------|------|-------|
| Прямой FK `notes.task_id` | Просто | Модуль notes знает про tasks — нарушение независимости |
| Таблица `note_tasks` | Явно | Для каждой пары модулей своя таблица → комбинаторный взрыв |
| **`entity_links` (универсальная)** | **Модули независимы, одна таблица на все связи** | JOIN с entity_types, чуть сложнее SQL |

### Когда entity_links НЕ подходит

- Иерархия внутри одного модуля (`chat_rooms` → `chat_messages`): прямой FK `chat_messages.room_id`
- Сильная связь с обязательной целостностью (`users` → `profiles`): прямой FK
- Производительные запросы с частым JOIN: материализованное представление или кэш

`entity_links` — для **межмодульных** связей, где модули не должны знать друг о друге.

---

## Diagram F. Cache invalidation flow

### Сценарий

Пользователь обновляет заметку. UI должен получить свежие данные. Cache invalidation — критический путь.

### Компоненты

```
Next.js → KrakenD → PostgREST → PostgreSQL (UPDATE notes)
                                          ↓
                                      activity_logs (trigger)
                                          ↓
                                      WAL
                                          ↓
                          air_activity_service (GenStage pipeline)
                                          ↓
                          ┌───────────────┴───────────────┐
                          ↓                               ↓
                  CacheInvalidatorConsumer           NatsConsumer
                  (sync Redis DEL/SET)               (publish to NATS)
                          ↓                               ↓
                      Redis                          air_hb_beta
                  (cache updated)                        ↓
                                                  WS broadcast to Next.js
                                                          ↓
                                                  Next.js: signal received
                                                  React Query: invalidate
                                                          ↓
                                                  Next.js: GET /notes/{id}
                                                          ↓
                                                  KrakenD → PostgREST
                                                          ↓
                                                  PostgreSQL (or Redis cache hit)
```

### Жизненный цикл cache key

```
Состояние 1: Cache HIT (note в кэше, актуальный)
─────────────────────────────────────────────────
Redis: tenant_42:notes:note_uuid_123 = {id, title:"Old", content:"...", updated_at:"T0"}

GET /notes/note_uuid_123
  → KrakenD → PostgREST
  → PostgREST: check Redis cache (HIT)
  → Return {title:"Old", ...} from cache
  → Latency: 2ms

Состояние 2: UPDATE в БД
─────────────────────────────────────────────────
PUT /notes/note_uuid_123 {title:"New"}
  → PostgREST → UPDATE notes SET title='New' WHERE id=...
  → Trigger log_activity → activity_logs (action=updated, new_data={title:"New",...})
  → WAL stream

Состояние 3: air_activity_service обрабатывает WAL event
─────────────────────────────────────────────────────────
WAL event received
  → CacheInvalidatorConsumer:
    - action = "updated"
    - Cache.set(tenant_id, "notes", entity_id, new_data)  ← OVERWRITE
    - Redis: tenant_42:notes:note_uuid_123 = {id, title:"New", content:"...", updated_at:"T1"}
    - EX 900 (15 min TTL)
  
  → NatsConsumer (parallel):
    - subject = signal.v1.notes.note.updated
    - publish to NATS

Состояние 4: UI получает signal
─────────────────────────────────────────────────
air_hb_beta (subscribed to signal.v1.>) receives signal
  → sanitize payload (drop raw, old_values)
  → broadcast to Phoenix Channel "activity:tenant:tenant_42_uuid"
  → Next.js receives via WebSocket:
    {action:"updated", entity_type:"notes", entity_id:"note_uuid_123", changed_fields:["title"]}

Состояние 5: UI refetch
─────────────────────────────────────────────────
Next.js React Query:
  → invalidateQueries({queryKey: ["notes", "note_uuid_123"]})
  → GET /notes/note_uuid_123
  → KrakenD → PostgREST
  → PostgREST: check Redis cache (HIT, already updated by CacheInvalidator)
  → Return {title:"New", ...} from cache
  → Latency: 2ms
  → UI shows "New"
```

### Критический момент: race condition

```
T0: UI sends PUT /notes/{id} {title:"New"}
T1: PostgREST UPDATE notes SET title='New'
T2: Trigger → activity_logs
T3: WAL write
T4: air_activity_service reads WAL
T5: CacheInvalidator: Redis SET (cache updated)
T6: NatsConsumer: publish signal
T7: air_hb_beta: broadcast WS
T8: UI receives signal
T9: UI sends GET /notes/{id}
T10: PostgREST check Redis (HIT, already updated at T5)
T11: Return fresh data
```

**Гарантия:** T5 (cache update) всегда **до** T6 (NATS publish) → T8 (UI signal) всегда **до** T11 (UI refetch). Это гарантирует, что когда UI делает refetch, кэш уже свежий.

В `air_activity_service` это обеспечивается `BroadcastDispatcher` — каждый consumer получает событие независимо, но `CacheInvalidatorConsumer` всегда отрабатывает `SET` перед публикацией в NATS (в рамках своего handle_events).

### Точка отказа: Redis упал

```
T4: air_activity_service reads WAL
T5: CacheInvalidator: Redis SET → ERROR (Redis down)
    → CacheInvalidator log error, but NOT crash
    → Событие считается обработанным (Ack)
    → Кэш не обновлён

T6: NatsConsumer: publish signal → OK
T7: air_hb_beta: broadcast WS → OK
T8: UI receives signal → OK
T9: UI GET /notes/{id}
T10: PostgREST check Redis → MISS (или stale data)
T11: PostgREST SELECT from PostgreSQL → fresh data
T12: Return fresh data, cache warmup
```

UI всё равно получит свежие данные — просто медленнее (cache miss → DB hit). Cache invalidation **не блокирует** бизнес-поток.

### Точка отказа: air_activity_service упал

```
T0-T3: UPDATE в БД, WAL записан
T4: air_activity_service не читает WAL
    → WAL копится в slot activity_slot_v1
    → Алерт при lag > 1GB

UI:
  - Optimistic UI показывает "New" (из HTTP response)
  - Другие пользователи не видят обновление
  - При refetch — PostgREST вернёт из Redis СТАРЫЕ данные (кэш не инвалидирован)

Восстановление:
  - air_activity_service поднимается
  - Читает накопленный WAL
  - CacheInvalidator обновляет Redis
  - NatsConsumer публикует signals
  - UI получает signals и обновляется
```

WAL lag monitoring — критичен. Алерт при `lag_bytes > 1GB` → немедленный retry air_activity_service.

---

## Сводка таймингов

| Сценарий | End-to-end | Узкое место |
|----------|------------|-------------|
| A. CRUD через PostgREST | ~50ms | PostgREST + DB |
| B. Event-driven (Command → Worker → DB → Signal) | ~150ms (без бизнес-логики) | Worker processing |
| C. Добавление воркера | ~30 секунд (sync + restart) | Developer writes code |
| D. Long-running (5 минут AI) | ~5 минут | AI provider |
| E. entity_links CRUD | ~50ms (как обычный CRUD) | PostgREST + DB |
| F. Cache invalidation | ~30ms после UPDATE | air_activity_service |

---

## Cross-references

- **Архитектура компонентов** → `ARCHITECTURE.md`
- **Типы сообщений, subject formula** → `COMMUNICATION_MODEL.md`
- **Метамодель, sync script** → `METAMODEL.md`
- **Пример добавления воркера с кодом** → `LIFECYCLE_ADDING_WORKER.md`
- **Contracts API** → `CONTRACTS_REFERENCE.md`
