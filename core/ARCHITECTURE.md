# ARCHITECTURE.md

> **Каноничный документ.** Заменяет `architecture_doc_0.2.2.md`, `ARCHITECTURAL_MANIFEST.md`, `(demo)present_system_for_tech_users.md`. Противоречия с ними разрешаются в пользу этого документа.

---

## TL;DR

Платформа Air Moon — event-driven, multi-tenant, CDC-based. PostgreSQL — источник правды, WAL → `activity_logs` → NATS/Redis. Сервисы общаются через Contracts (event/signal/command). Воркеры только пишут в БД, не публикуют CDC-события сами. RLS включён везде. Формат сообщений — JSON. Subject: `[type].v1.[domain].[entity].[action]` (ед.ч.).

---

## 1. Принципы

| # | Принцип | Суть |
|---|---------|------|
| 1 | **DDD** | Инварианты инкапсулированы в `domain/`. Прикладные сервисы оркестрируют, домен защищает состояние |
| 2 | **CQRS + Sync/Async Split** | Чтение/запись синхронны по HTTP. Побочные эффекты — асинхронно в Event-среде |
| 3 | **Transactional Outbox через CDC** | Воркеры не публикуют события сами. Атомарный INSERT/UPDATE → WAL → `air_activity_service` публикует. WAL — единственный триггер CDC-событий |
| 4 | **Пинг-Фетч (Notification-Driven)** | По WS/NATS летят лёгкие Сигналы. Получатель сам pull-ит данные из Redis/БД |
| 5 | **Zero-Trust Gateway Auth** | KrakenD + Auth Service (Kratos + Casbin). Внутренние сервисы не валидируют токены, RLS на уровне БД |

---

## 2. Стек

| Слой | Технология |
|------|------------|
| Frontend | Next.js |
| API Gateway | KrakenD |
| Auth | Ory Kratos v26 + Casbin v3 (в Auth Service) |
| CRUD | PostgREST + `supabase/postgrest-js` клиент |
| Event Router (бывший Ingestion) | Go + Fiber 3 |
| Auth Service | Go + Chi v5 (standalone репо) |
| Брокер | NATS JetStream |
| Workers | Go (монорепо) |
| Activity Service (WAL → NATS/Redis) | Elixir + GenStage |
| Broadcaster (WS → Next.js) | Elixir + Phoenix (`air_hb_beta`) |
| БД | PostgreSQL 17+ (RLS, партиционирование, pg_cron) |
| Кэш | Redis |
| Векторный поиск | Qdrant |
| Полнотекстовый поиск | Meilisearch |
| Аналитика | QuestDB (изолирована от критического пути) |
| Логирование | zerolog (Go), Logger (Elixir) |

---

## 3. Топология

```
                    ┌─────────────────────────────┐
                    │        Next.js Client        │
                    └──────────────┬──────────────┘
                                   │ HTTP / WebSocket
                                   ▼
                    ┌─────────────────────────────┐
                    │         KrakenD             │
                    │  (JWT, rate-limit, routing) │
                    └──┬─────────┬────────┬───────┘
                       │         │        │
            ┌──────────▼─┐  ┌────▼─────┐  ┌▼───────────────┐
            │Auth Service│  │Event     │  │PostgREST       │
            │(Chi, Kratos│  │Router    │  │(simple CRUD)   │
            │+Casbin)    │  │(Fiber 3) │  │                │
            └──────┬─────┘  └────┬─────┘  └────┬───────────┘
                   │             │             │
                   └─────────────┼─────────────┘
                                 │
                                 ▼
                    ┌─────────────────────────────┐
                    │     PostgreSQL Cluster      │
                    │   (Primary + Read Replica)  │
                    │                             │
                    │  CORE: tenants, workspaces, │
                    │        profiles, teams,     │
                    │        memberships          │
                    │  MODULES: notes, tasks,     │
                    │           chat_rooms, ...   │
                    │  INTEGRATION: entity_links  │
                    │  AUDIT: activity_logs       │
                    │        (partitioned)        │
                    └──────────────┬──────────────┘
                                   │ WAL (publication: activity_pub)
                                   ▼
                    ┌─────────────────────────────┐
                    │   air_activity_service      │
                    │       (Elixir/GenStage)     │
                    │                             │
                    │  Reads WAL slot             │
                    │  Decodes → normalizes       │
                    │  → publishes to NATS        │
                    │  → updates Redis cache      │
                    │  → broadcasts to WS         │
                    └──┬──────────────┬───────────┘
                       │              │
                       ▼              ▼
            ┌─────────────────┐  ┌────────────────────┐
            │  NATS JetStream │  │      Redis         │
            │                 │  │  (entity cache +   │
            │  Streams:       │  │   tenant registry +│
            │  - COMMANDS     │  │   casbin policies) │
            │  - EVENTS       │  └────────────────────┘
            │  - SIGNALS      │
            │  - INDEXER      │
            └────┬─────┬──────┘
                 │     │
        ┌────────┘     └────────────┐
        ▼                           ▼
┌─────────────────┐         ┌────────────────────┐
│   Go Workers    │         │   air_hb_beta      │
│ (NATS consumers)│         │ (Phoenix, WS→Next) │
│                 │         │ + sanitize payload │
│ - vectorizer    │         │ + RBAC on join     │
│ - indexer       │         └────────────────────┘
│ - ai-summary    │
│ - report        │
└────────┬────────┘
         │ HTTP (исходящие вызовы к сервисам)
         ▼
┌─────────────────────────────────────────────────┐
│  Standalone core-сервисы (или в services/       │
│  монорепо):                                      │
│  - ai-provider                                   │
│  - file-storage (обёртка над RustFS)             │
└─────────────────────────────────────────────────┘
```

---

## 4. Компонентный инвентарь

| Компонент | Язык | Ответственность | Репо |
|-----------|------|-----------------|------|
| **KrakenD** | Go (binary) | JWT, rate-limit, tenant routing, SSL termination | external |
| **Auth Service** | Go + Chi | Identity (Kratos), RBAC/ABAC (Casbin), Email MFA, сессии, JWT-cookie | standalone |
| **Event Router** | Go + Fiber | HTTP API для приёма команд от клиентов, публикация в NATS | монорепо `services/event-router/` |
| **PostgREST** | Haskell (binary) | CRUD API для простых таблиц, RLS | external |
| **air_activity_service** | Elixir + GenStage | Чтение WAL, нормализация, публикация в NATS, инвалидация Redis, broadcast WS | standalone |
| **air_hb_beta** (Broadcaster) | Elixir + Phoenix | WS-каналы клиентам, sanitize payload, RBAC на join | standalone |
| **Go Workers** | Go | NATS consumers, бизнес-логика, DDD-домен, пишут в БД | монорепо `workers/<name>/` |
| **Standalone core services** | Go | HTTP/gRPC API для воркеров (ai-provider, file-storage) | standalone или монорепо `services/<name>/` |

---

## 5. Два пути входа

### CRUD-путь (простые операции)

```
Next.js (Optimistic UI)
  → KrakenD (JWT валидация)
  → PostgREST
  → PostgreSQL INSERT/UPDATE/DELETE
  → Триггер log_activity() → activity_logs
  → WAL → air_activity_service
  → Redis (SET/DEL) + NATS (event/signal) + WS broadcast
  → Next.js (signal → pull из Redis через API)
```

**Когда:** простые операции без бизнес-логики (создать заметку, обновить название, удалить задачу, отметить избранное).

### Event-driven путь (сложные операции)

```
Next.js
  → KrakenD
  → Event Router (HTTP POST /events)
  → NATS (command.v1.<domain>.<entity>.<action>)
  → Worker (consumer)
  → PostgreSQL (DDD-валидация → INSERT/UPDATE)
  → Триггер log_activity() → activity_logs
  → WAL → air_activity_service
  → Redis + NATS (event/signal) + WS broadcast
  → Next.js
```

**Когда:** тяжёлые, составные, асинхронные операции (сгенерировать отчёт, обработать документ, AI-задача, каскадное удаление workspace).

**Оба пути сходятся в `activity_logs`.** WAL гарантирует: сигнал = факт успешной записи.

---

## 6. Multi-tenant

### Иерархия

```
Tenant (организация)
  └── Workspace (рабочее пространство)
        └── Team (команда)
              └── Profile (пользователь)
```

### JWT Claims

```json
{
  "profile_id":   "uuid",
  "tenant_id":    "uuid",
  "workspace_id": "uuid",
  "roles":        ["owner"],
  "aal":          "aal1|aal2",
  "session_id":   "uuid",
  "iss":          "auth",
  "sub":          "profile_uuid"
}
```

Переключение tenant → сброс workspace на default, перевыпуск JWT-cookie.

### Изоляция

- **Физическая**: per-tenant PostgreSQL инстанс (3 уровня: Shared / Dedicated / Dedicated+Replica)
- **Логическая**: RLS на каждой таблице через `app.current_tenant_id` контекст
- **Auth**: Kratos identity, Casbin RBAC с доменами (`sub, dom, obj, act`)
- **Subject rule**: `tenant_id` **НИКОГДА** в NATS subject, только в payload/Redis-key/БД

---

## 7. DB-архитектура

### Три уровня таблиц

```
CORE (auth-домен)
  tenants, workspaces, profiles, teams, memberships,
  roles, permissions, sessions, mfa_settings, otp_codes,
  security_events, casbin_policies

MODULES (бизнес-домены)
  notes, tasks, chat_rooms, chat_messages, chat_members,
  documents, projects, goals, files, ...

INTEGRATION (клей)
  entity_types (id, code)
  entity_links (source_type, source_id, target_type, target_id, relation_type, meta)
```

### `entity_links` — универсальный клей между модулями

Модули НЕ знают друг о друге. Связи только через `entity_links`:

```sql
CREATE TABLE entity_links (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    workspace_id    UUID NOT NULL,
    source_type_id  SMALLINT NOT NULL REFERENCES entity_types(id),
    source_id       UUID NOT NULL,
    target_type_id  SMALLINT NOT NULL REFERENCES entity_types(id),
    target_id       UUID NOT NULL,
    relation_type   TEXT NOT NULL,  -- attached, related, parent, child, owner, participant
    meta            JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

Заменяет `entity_files`, `entity_tags`, `entity_chats`, `entity_notes`, `entity_tasks`, и т.п.

### `entity_types` — реестр типов

```sql
CREATE TABLE entity_types (
    id    SMALLINT PRIMARY KEY,
    code  TEXT UNIQUE NOT NULL
);

-- 1=workspace, 2=team, 3=profile, 4=chat_room, 5=note, 6=task, 7=project, ...
```

Строки запрещены (избегаем `chat` vs `ChatRoom` vs `chat-room`). Только коды из реестра.

---

## 8. Базовый шаблон таблицы

**Каждая бизнес-сущность** содержит:

```sql
id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
tenant_id       UUID NOT NULL,
workspace_id    UUID NOT NULL,
meta            JSONB NOT NULL DEFAULT '{}',
created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
```

### Когда добавлять

- Самостоятельные сущности: `notes`, `tasks`, `chat_rooms`, `documents`, `projects`
- Дочерние сущности (`chat_messages`, `chat_members`, `task_comments`): **тоже добавляем** — денормализация для RLS/аудита/экспорта tenant'а без JOIN на родителя

### Опциональные поля

- `sort_order BIGINT NOT NULL DEFAULT 0` — только где реально нужен drag-and-drop / ручная сортировка
- `deleted_at TIMESTAMPTZ` — для soft delete
- `created_by UUID`, `updated_by UUID` — для аудита

---

## 9. `activity_logs` — ядро системы

```sql
CREATE TABLE activity_logs (
    id           UUID        NOT NULL DEFAULT gen_random_uuid(),
    tenant_id    UUID        NOT NULL,           -- NO FK намеренно
    workspace_id UUID,                           -- NO FK. NULL = tenant-level
    entity_type  TEXT        NOT NULL,           -- имя таблицы ('notes', 'chat_messages')
    entity_id    UUID        NOT NULL,
    action_type  TEXT        NOT NULL
        CHECK (action_type IN ('created','updated','deleted','viewed','shared','archived')),
    actor_id     UUID,                           -- NULL = system
    actor_type   TEXT        NOT NULL DEFAULT 'human'
        CHECK (actor_type IN ('human','system','agent','ai_assistant','api','webhook')),
    old_data     JSONB,                          -- для важных событий
    new_data     JSONB,                          -- полное состояние ПОСЛЕ
    changes      JSONB,                          -- diff: {"field": {"old":"x","new":"y"}}
    request_id   UUID,                           -- correlation id
    source       TEXT        NOT NULL DEFAULT 'api'
        CHECK (source IN ('api','webhook','cron','migration','import','sync','system')),
    ip_address   INET,
    user_agent   TEXT,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);
```

### Ключевые решения

- **Партиционирование по месяцам** (одна партиция = один месяц)
- **Retention: 13 месяцев** в PostgreSQL, потом `DROP PARTITION`
- **NO FK** на `tenant_id`/`workspace_id`/`actor_id` — журнал фактов переживает удаление сущностей
- **RLS включён** на самой таблице журнала
- **pg_cron**: 25-го числа 03:00 — `create_activity_log_partitions()` (текущий + следующий месяц), 1-го числа 04:00 — `drop_old_activity_log_partitions()`

### Универсальный триггер `log_activity()`

Один триггер на все entity-таблицы. Подключение одной строкой:

```sql
SELECT attach_activity_log('notes');
SELECT attach_activity_log('chat_messages');
```

Триггер:
- INSERT → `new_data = to_jsonb(NEW)`, action='created'
- UPDATE → вычисляет diff через `jsonb_object_agg`, пропускает если ничего не изменилось
- DELETE → `old_data = to_jsonb(OLD)`, action='deleted'
- Читает `app.current_user_id`, `app.current_actor_type`, `app.current_source` из SET LOCAL контекста

**Требование к таблице:** поля `id`, `tenant_id`, `workspace_id` обязательны.

---

## 10. WAL / CDC Setup

### PostgreSQL настройки

```sql
ALTER SYSTEM SET wal_level = logical;
ALTER SYSTEM SET max_replication_slots = 10;
ALTER SYSTEM SET max_wal_senders = 10;
ALTER SYSTEM SET wal_compression = on;
ALTER SYSTEM SET max_wal_size = '4GB';
ALTER SYSTEM SET checkpoint_completion_target = 0.9;
-- Требует рестарта PostgreSQL
```

### Publication + Slot

```sql
CREATE PUBLICATION activity_pub FOR TABLE activity_logs;
SELECT pg_create_logical_replication_slot('activity_slot_v1', 'pgoutput');
```

`air_activity_service` слушает только `activity_logs` через слот `activity_slot_v1`. poll_interval: 100ms.

### Мониторинг WAL lag (критично)

```sql
SELECT
    slot_name,
    active,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)) AS lag_human,
    pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn) AS lag_bytes
FROM pg_replication_slots
WHERE slot_name = 'activity_slot_v1';
-- Алерт: lag_bytes > 1GB → Reader упал, диск заполняется
```

### `air_activity_service` — Supervision Tree

```
AirActivityService.Supervisor (one_for_one)
├── Repo
├── Phoenix.PubSub
├── Endpoint
├── Activity.Producer              ← GenStage, BroadcastDispatcher
├── Activity.NatsConsumer         ← → NATS (events/signals)
├── Activity.ChannelConsumer      ← → Phoenix Channels (внутренний WS)
├── Activity.CacheInvalidatorConsumer ← → Redis
├── NatsConnection
├── RedisConnection
└── ReplicationSupervisor
    └── ReplicationConsumer       ← читает WAL, последний стартует
```

`BroadcastDispatcher` — каждый consumer получает каждое событие независимо. Изоляция отказов: NATS тормозит → Redis и WS продолжают работать.

### Обработка события

```
WAL событие
   ↓
1. Декодировать pgoutput → Elixir map
   ↓
2. Trim: убрать old_values и тяжёлые поля
   ↓
3. Параллельно (BroadcastDispatcher):
   ├── Redis: SET {tenant_id}:{entity_type}:{entity_id} new_values EX ttl
   │          (для DELETE → DEL key)
   ├── NATS: publish event.v1.{domain}.{entity}.created
   │               OR signal.v1.{domain}.{entity}.updated|deleted
   └── Phoenix Channel: broadcast "new_activity" (для прямых WS-клиентов)
   ↓
4. Подтвердить LSN
```

**Subject construction:** через `Contracts.subject(type, domain, entity, action)` + lookup в `contract_mappings` для получения `(domain, entity)` из `entity_type` (имя таблицы). См. `COMMUNICATION_MODEL.md` и `METAMODEL.md`.

---

## 11. Standalone сервисы vs монорепо

### Монорепо `air-core-platform/`

```
air-core-platform/
├── cmd/
│   ├── services/                  ← main.go для HTTP-сервисов
│   │   └── event-router/
│   └── workers/                   ← main.go для NATS-воркеров
│       ├── notes-vectorize/
│       ├── chat-message/
│       └── ...
├── internal/                      ← shared infra
│   ├── config/
│   ├── logger/                    ← zerolog
│   ├── db/                        ← pgxpool
│   ├── natsconn/
│   ├── contracts/                 ← Go contracts (Subject, CacheKey, Streams)
│   │   └── tables/                ← table descriptors (*.json)
│   ├── cache/
│   ├── worker/                    ← базовый скелет воркера
│   ├── event/                     ← контракт Event
│   └── ...
├── services/                      ← HTTP-сервисы монорепо (внешний транспорт)
│   └── event-router/
└── workers/                       ← NATS-воркеры (без внешнего API)
    ├── notes-vectorize/
    │   ├── worker.json
    │   ├── processor.go
    │   └── contracts/
    └── ...
```

### Standalone репозитории

| Сервис | Почему отдельный |
|--------|------------------|
| **Auth Service** | Критическое ядро, минимум зависимостей (Chi вместо Fiber), долгоживущий код |
| **air_activity_service** | Elixir, отдельная операционка (BEAM) |
| **air_hb_beta** | Elixir, отдельная операционка |
| **ai-provider** | Может быть standalone core, Go, HTTP API |
| **file-storage** | Может быть standalone core, Go, обёртка над RustFS |

### Правило: что где живёт

- **`workers/`** — NATS-only воркеры. Не имеют внешнего API, но **могут** делать исходящие HTTP-вызовы к `services/` или standalone core-сервисам
- **`services/`** в монорепо — единственное место, где может быть коннект во внешний мир (HTTP/gRPC/WebSocket серверы)
- **Standalone репо** — для критического ядра (Auth) или компонентов на другом языке (Elixir)

---

## 12. Эволюция и масштабирование

| Стадия | Триггер | Что делаем |
|--------|---------|------------|
| **0. Фундамент** | До продакшена | `tenant_id` везде, RLS, KrakenD + Tenant Router, activity_logs партиции, WAL monitoring |
| **1. Первое давление** | 10-50k активных пользователей | Read Replica для крупных tenants, Phoenix Reader с реплики, метрики по воркерам |
| **2. Рост** | 50-150k пользователей | QuestDB per-tenant для холодной истории (>13 мес), батч-воркер PG→QuestDB, автоскейлинг воркеров по NATS consumer lag |
| **3. Highload** | 150-300k+ пользователей | Вертикалка Primary + NVMe, Redis PubSub между нодами broadcaster, переход провижининга с bash на Go, Prometheus + Grafana |

---

## Cross-references

- **Коммуникационная модель (event/signal/command, subject formula)** → `COMMUNICATION_MODEL.md`
- **Метамодель (worker.json, contract.json, registry.json, contract_mappings)** → `METAMODEL.md`
- **Пример добавления нового воркера** → `LIFECYCLE_ADDING_WORKER.md`
- **Contracts API (Go + Elixir)** → `CONTRACTS_REFERENCE.md`
- **Сквозные трейсы** → `WIRING_DIAGRAMS.md`
