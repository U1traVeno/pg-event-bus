# pgebus

[**English**](README.md) | [简体中文](README.zh-CN.md)

Event Bus System Based on PostgreSQL - Lightweight and Reliable Event Processing Framework

## Features

- ✅ **PostgreSQL as the Single Source of Truth** - The database serves as the only queue, ensuring data consistency
- ✅ **NOTIFY/LISTEN Real-Time Notifications** - Leverages PostgreSQL's native features for event pushing
- ✅ **Concurrency Safety** - Ensures no duplicate event processing
- ✅ **Delayed Execution** - Supports scheduled tasks and delayed event handling
- ✅ **Automatic Retry** - Built-in retry mechanism for automatic handling of failures
- ✅ **Hierarchical Event Dispatching** - Supports prefix matching and hierarchical dispatching of event types
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

## Dependencies

- Python 3.8+
- SQLAlchemy 2.0+ (with asyncio support)
- asyncpg
- PostgreSQL 9.5+

## License

MIT

## Contribution

Contributions are welcome! Please submit a Pull Request or create an Issue.
