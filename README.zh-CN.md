# pgebus

[English](README.md) | **简体中文**

基于 PostgreSQL 的事件总线系统 - 轻量级且可靠的事件处理框架

## 功能特性

- ✅ **PostgreSQL 作为唯一数据源** - 数据库作为唯一的队列，确保数据一致性
- ✅ **NOTIFY/LISTEN 实时通知** - 利用 PostgreSQL 的原生功能进行事件推送
- ✅ **并发安全** - 确保事件不会被重复处理
- ✅ **延迟执行** - 支持定时任务和延迟事件处理
- ✅ **自动重试** - 内置重试机制，自动处理失败
- ✅ **分层事件分发** - FastAPI 风格的路由组合（`prefix` + `include_router()`）（TODO：如需通配符/前缀匹配可补充）
- ✅ **完整类型标注** - 内置完整类型注解（PEP 561）

## 安装

待补充

## pgebus 不是什么

pgebus 并不打算替代以下工具：

- RabbitMQ
- Redis Streams
- Kafka
- NATS
- 云原生消息代理

具体来说，它不适用于以下场景：

- 高吞吐量的事件流处理
- 大规模的发布/订阅工作负载
- 跨数据中心或互联网级别的消息传递
- 大型负载的消息传递

如果你需要高吞吐量、大规模发布/订阅或解耦的消费者，请使用专业的消息代理。

## 为什么使用 pgebus

以下情况可能适合使用 pgebus：

- 你的应用主要是本地或单体架构
- 你可以控制 PostgreSQL 实例
- 你有长时间运行或高成本的外部任务
- 丢失或重复事件会带来实际成本
- 你已经依赖 PostgreSQL 来保证数据正确性

## 为什么不使用 pgebus

以下情况可能不适合使用 pgebus：

- 你的系统是微服务架构
- 你需要跨区域的水平扩展
- 你需要处理突发流量或大规模发布/订阅
- 你将事件视为尽力而为的信号

## API 参考

本项目刻意模仿 FastAPI 的使用手感：

- 创建一个 `EventRouter()`（通常命名为 `event`）
- 使用 `@event.on("...")` 注册处理函数
- 使用 `include_router(...)` 进行路由组合（类似 prefix）
- 使用 `await EventSystem(...).start()` / `await ...stop()` 管理生命周期
- 使用 `publish_event(...)` 发布事件

### 快速开始

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

### 核心 API

#### `EventSystem`

- `EventSystem(router: EventRouter, settings: Settings)`
- `await EventSystem.start()`
- `await EventSystem.stop(wait_for_completion: bool = True, timeout: float | None = 30.0)`

说明：

- `start()` 会确保配置的 schema 存在（`CREATE SCHEMA IF NOT EXISTS`）。
- `start()` 不会帮你创建数据表/迁移。（TODO：补充推荐的迁移方式。）

#### `EventRouter`

- `EventRouter(prefix: str = "")`
- `@router.on(path: str)` 注册处理函数。
- `router.include_router(other: EventRouter, prefix: str = "")` 组合路由。

分发规则：

- 当前事件类型匹配为**精确匹配**（`path == event["type"]`）。
- “分层/前缀”的路由组织已支持：通过 router 的 `prefix` 与 `include_router(...)` 以 `.` 拼接。
- TODO：如果你希望处理函数支持 `demo.*` 这类“按前缀/通配符匹配”，可在路由器里新增匹配策略。

处理函数签名：

- `async def handler(ctx: EventContext, payload: dict) -> Any`

关于 `transactional`：

- `@router.on("...", transactional=True)` 表示“我要求在 dispatcher 管理的事务中运行”。
- ❌ `transactional` 不是自动 commit。
- ❌ `transactional` 不是每个 handler 一个事务。
- ✅ 最多只会开一个事务；是否开启由 dispatcher 根据所有 handlers 决定。
- handler 中禁止 `commit/rollback/close/begin/begin_nested/connection/get_bind/invalidate`；如需原始能力可用 `ctx.session.unsafe`（不推荐）。

#### `publish_event`

- `await publish_event(session, event_type: str, payload: dict, channel: str, run_at: datetime | None = None) -> Event`

说明：

- 调用方必须 `commit()`（或用 `async with session.begin(): ...`）后，事件才会真正入库并可被 worker 消费。
- `payload` 建议保持精简（通常只放业务 ID），因为 PostgreSQL 是唯一事实来源。

## 依赖

- Python 3.8+
- SQLAlchemy 2.0+（支持 asyncio）
- asyncpg
- PostgreSQL 9.5+

## 许可证

MIT

## 贡献

欢迎贡献！请提交 Pull Request 或创建 Issue。
