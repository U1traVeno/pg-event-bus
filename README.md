# pgebus

[**English**](README.md) | [简体中文](README.zh-CN.md)

Event Bus System Based on PostgreSQL - Lightweight and Reliable Event Processing Framework

## Features

- ✅ **PostgreSQL as the Single Source of Truth** - The database serves as the only queue, ensuring data consistency
- ✅ **NOTIFY/LISTEN Real-Time Notifications** - Leverages PostgreSQL's native features for event pushing
- ✅ **Concurrency Safety** - Ensures no duplicate event processing
- ✅ **Delayed Execution** - Supports scheduled tasks and delayed event handling
- ✅ **Automatic Retry** - Built-in retry mechanism for automatic handling of failures
- ✅ **Hierarchical Event Dispatching** - FastAPI-style router composition via `prefix` and `include_router()` (TODO: add wildcard/prefix matching if needed)
- ✅ **Fully Typed** - Ships with complete type annotations (PEP 561)

## Installation

TODO

## What pgebus IS NOT

pgebus is not intended to replace:

- RabbitMQ
- Redis Streams
- Kafka
- NATS
- Cloud-native message brokers

Specifically, it is not suitable for:

- High-throughput event streaming
- Large fan-out pub/sub workloads
- Cross-datacenter or internet-scale messaging
- Heavy payload delivery

If you need throughput, fan-out, or decoupled consumers, use a real message broker.

## Why pge-bus

You are likely a good fit if:

- Your application is mostly local or monolithic
- You control the PostgreSQL instance
- You have long-running or expensive external jobs
- Losing or duplicating an event would cause real cost
- You already rely on PostgreSQL for correctness

## Why Not Use pgebus

You should probably not use this if:

- Your system is microservice-heavy
- You need horizontal scaling across regions
- You expect burst traffic or high fan-out
- You treat events as best-effort signals

## API Reference

This project intentionally mimics FastAPI's feel:

- Create an `EventRouter()` (often named `event`)
- Register handlers via `@event.on("...")`
- Compose routers via `include_router(...)` (prefix-style)
- Run the system with `await EventSystem(...).start()` / `await ...stop()`
- Publish events via `publish_event(...)`

### Quickstart

```python
import asyncio
from datetime import datetime, timedelta, timezone

from pgebus import EventRouter, EventSystem, Settings, DatabaseSessionManager, publish_event


event = EventRouter()


@event.on("demo.hello", transactional=False)
async def handle_demo(ctx, payload):
    # payload is the raw DB payload dict
    # ctx.session is None unless some handler requires transactional=True
    print("got:", payload)


async def main() -> None:
    settings = Settings(
        database={
            "host": "localhost",
            "port": 5432,
            "user": "postgres",
            "password": "postgres",
            "database": "postgres",
            "application_name": "pgebus",
            "schema": "pgebus",
        },
        event_system={
            "channel": "events",
            "n_workers": 5,
        },
    )

    system = EventSystem(router=event, settings=settings)
    await system.start()

    # Publish from your app code (can be the same process)
    publisher_sm = DatabaseSessionManager(settings.database)
    async with publisher_sm.session() as session:
        await publish_event(
            session,
            event_type="demo.hello",
            payload={"msg": "hi"},
            channel=settings.event_system.channel,
        )
        # IMPORTANT: publish_event does NOT commit
        await session.commit()

    # Delayed execution
    await publish_event(
        session,
        event_type="demo.hello",
        payload={"msg": "later"},
        channel=settings.event_system.channel,
        run_at=datetime.now(timezone.utc) + timedelta(seconds=10),
    )
    await session.commit()

 # ... your app runs ...

    await system.stop()
    await publisher_sm.close()


asyncio.run(main())
```

### Core APIs

#### `EventSystem`

- `EventSystem(router: EventRouter, settings: Settings)`
- `await EventSystem.start()`
- `await EventSystem.stop(wait_for_completion: bool = True, timeout: float | None = 30.0)`

Notes:

- `start()` ensures the configured schema exists (via `CREATE SCHEMA IF NOT EXISTS`).
- `start()` does NOT create tables/migrations for you. (TODO: document recommended migration workflow.)

#### `EventRouter`

- `EventRouter(prefix: str = "")`
- `@router.on(path: str)` registers a handler.
- `router.include_router(other: EventRouter, prefix: str = "")` composes routers.

Dispatch rules:

- Event type matching is currently **exact** (`path == event["type"]`).
- Hierarchical composition is supported via router `prefix` and `include_router(...)` joining with `.`.
- TODO: add prefix/wildcard matching if you want handlers to match `demo.*`-style patterns.

Handler signature:

- `async def handler(ctx: EventContext, payload: dict) -> Any`

About `transactional`:

- `@router.on("...", transactional=True)` means “I require running inside a dispatcher-managed transaction”.
- ❌ `transactional` does not mean auto-commit.
- ❌ `transactional` does not mean one transaction per handler.
- ✅ At most one transaction is opened per event; the dispatcher decides based on all matched handlers.
- Inside handlers, `commit/rollback/close/begin/begin_nested/connection/get_bind/invalidate` are forbidden; use `ctx.session.unsafe` only as an explicit escape hatch.

#### `publish_event`

- `await publish_event(session, event_type: str, payload: dict, channel: str, run_at: datetime | None = None) -> Event`

Notes:

- You must `commit()` (or use `async with session.begin(): ...`) for the event to be visible to workers.
- `payload` should stay small (typically IDs), because PostgreSQL is the source of truth.

## Dependencies

- Python 3.8+
- SQLAlchemy 2.0+ (with asyncio support)
- asyncpg
- PostgreSQL 9.5+

## License

MIT

## Contribution

Contributions are welcome! Please submit a Pull Request or create an Issue.
