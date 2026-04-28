# Buzz

> Event engine for real-time event processing, rule matching, and workflow execution.
> Part of the [KnotMark](https://github.com/knotmark) ecosystem (Waggle + Buzz + Honeycomb).

## Overview

Buzz is a lightweight, embeddable event engine that provides:

- **Event Bus** — CloudEvents-compatible event publishing and subscription
- **CEP (Complex Event Processing)** — Pattern matching with 6 primitives: sequence, parallel, any, absence, repeat, filter
- **NL → DSL** — Natural language to rule DSL compilation (LLM at compile time only)
- **Workflow Engine** — Action execution with retry, timeout, and audit logging

## Architecture

```
Events In ──→ Event Bus ──→ CEP Engine (NFA) ──→ Workflow Engine ──→ Actions Out
                                ↑
                          Rule Store
                          (DSL/AST)
```

## Design Principles

- **Zero LLM at runtime** — LLM only involved when creating/modifying rules, never during event matching
- **Embeddable** — Use as a Go library, not a standalone service
- **Pluggable infrastructure** — Swap event bus (InMemory → SQS → Kafka), workflow engine (direct → BullMQ → Temporal)
- **Multi-tenant** — Built-in tenant isolation for events, rules, and state

## Relationship to KnotMark Projects

```
Waggle    — LLM Gateway (bee communication)
Buzz      — Event Engine (bee activity/vibration)  
Honeycomb — Knowledge Platform (bee storage/home)
```

Honeycomb uses Buzz as its event gateway layer. Buzz can also be used independently by any Go project that needs event-driven rule processing.

## Status

Early development. Core abstractions being designed.

## License

TBD
