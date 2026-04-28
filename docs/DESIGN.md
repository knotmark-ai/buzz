# Buzz Design Document

> Buzz is a lightweight, embeddable event engine for Go.
> It provides event bus, complex event processing (CEP), rule compilation (NL → DSL),
> and workflow execution — all without LLM dependency at runtime.

---

## 1. Problem Statement

Modern applications need event-driven automation:
- "If no customer interaction for 30 days, send alert"
- "If 3 consecutive negative sentiments, escalate"
- "When subscription expires within 7 days, trigger renewal workflow"

Current options are either too heavy (Flink, Esper) or too simple (cron + polling).
Buzz fills the gap: **embeddable, composable, and LLM-free at runtime**.

---

## 2. Architecture Overview

```
+------------------------------------------------------------------+
|                          Buzz Engine                              |
|                                                                  |
|  +------------+    +------------+    +-----------+    +--------+ |
|  | Event Bus  |--->| CEP Engine |--->| Workflow  |--->| Action | |
|  | (Pub/Sub)  |    | (NFA)      |    | Engine    |    | Exec   | |
|  +------------+    +-----+------+    +-----------+    +--------+ |
|                          |                                       |
|                    +-----+------+                                |
|                    | Rule Store |                                |
|                    | (AST)      |                                |
|                    +------------+                                |
|                                                                  |
|  +------------------+    +------------------+                    |
|  | NL -> DSL Compiler|    | Infra Adapters  |                    |
|  | (LLM, compile    |    | InMemory / SQS  |                    |
|  |  time only)       |    | Temporal / Kafka|                    |
|  +------------------+    +------------------+                    |
+------------------------------------------------------------------+
```

---

## 3. Core Components

### 3.1 Event Bus

Publish/subscribe for CloudEvents 1.0 format.

```go
// Core interface
type EventBus interface {
    Publish(ctx context.Context, event Event) error
    Subscribe(pattern EventPattern, handler EventHandler) (Subscription, error)
    Unsubscribe(sub Subscription) error
}

// Event follows CloudEvents 1.0 spec
type Event struct {
    ID          string            `json:"id"`
    Type        string            `json:"type"`        // e.g. "fact.asserted"
    Source      string            `json:"source"`      // e.g. "/ontology/facts"
    TenantID    string            `json:"tenantid"`    // mandatory multi-tenant
    Time        time.Time         `json:"time"`
    Data        json.RawMessage   `json:"data"`
    Extensions  map[string]string `json:"extensions,omitempty"`
}
```

**Adapters** (pluggable infrastructure):

| Adapter | Use Case | Phase |
|---------|----------|-------|
| `InMemory` | Development, testing, single-process | 1 |
| `Redis Streams` | Multi-process, persistent | 2 |
| `Kafka / Redpanda` | High-throughput, distributed | 3 |

### 3.2 CEP Engine (Complex Event Processing)

Pattern matching on event streams using NFA (Nondeterministic Finite Automaton).

**6 Primitives:**

| Primitive | Semantics | Example |
|-----------|-----------|---------|
| `filter` | Single event matching condition | `event.type == "fact.asserted" AND data.predicate == "stage"` |
| `sequence` | A then B (ordered) | Login followed by purchase within 1h |
| `parallel` | A and B both (unordered) | Both approval AND payment received |
| `any` | A or B (either) | Email reply OR phone call |
| `absence` | A not happening within window | No interaction for 30 days |
| `repeat` | A happening N times | 3 consecutive errors |

All primitives support `within` time window modifier.
Complex rules are built by nesting primitives.

```go
// Pattern is the compiled rule representation
type Pattern struct {
    ID        string
    TenantID  string
    Kind      PatternKind  // filter, sequence, parallel, any, absence, repeat
    Children  []Pattern    // nested patterns
    Condition Condition    // for filter/leaf nodes
    Window    Duration     // time constraint
    Count     int          // for repeat
}

// CEP engine interface
type CEPEngine interface {
    Register(pattern Pattern, handler MatchHandler) error
    Unregister(patternID string) error
    Process(event Event) error  // feed event into all active NFAs
}
```

**NFA Implementation:**
- Each registered pattern compiles to an NFA instance
- NFA state is per-tenant, persisted for crash recovery
- ~200 lines core NFA logic (intentionally minimal)
- `absence` uses timer-based state expiry

### 3.3 Rule Store

Persistent storage for rule definitions (DSL/AST form).

```go
type RuleStore interface {
    Save(ctx context.Context, rule Rule) error
    Get(ctx context.Context, id string) (Rule, error)
    List(ctx context.Context, tenantID string) ([]Rule, error)
    Delete(ctx context.Context, id string) error
}

type Rule struct {
    ID          string
    TenantID    string
    Name        string
    Description string
    Pattern     Pattern         // compiled CEP pattern
    Actions     []ActionDef     // what to do on match
    Enabled     bool
    CreatedAt   time.Time
    UpdatedAt   time.Time
    Source      RuleSource      // "manual", "nl_compiled", "system"
    DSL         string          // original DSL YAML (for display/edit)
}
```

**Storage adapters:**

| Adapter | Use Case | Phase |
|---------|----------|-------|
| `InMemory` | Testing | 1 |
| `PostgreSQL` | Production | 1 |
| `SQLite` | Embedded/edge | 2 |

### 3.4 Workflow Engine

Execute actions when a pattern matches. Supports sequential steps, branching, retry, and timeout.

```go
type WorkflowEngine interface {
    Execute(ctx context.Context, workflow WorkflowDef, input WorkflowInput) (WorkflowResult, error)
    Status(ctx context.Context, runID string) (WorkflowStatus, error)
}

type WorkflowDef struct {
    Steps []Step
}

type Step struct {
    Kind     StepKind  // action, branch, wait, notify
    Action   string    // action name to execute
    Args     map[string]any
    Timeout  Duration
    Retry    RetryPolicy
    // for branch
    Condition string
    Then      []Step
    Else      []Step
}
```

**Adapters:**

| Adapter | Use Case | Phase |
|---------|----------|-------|
| `Direct` | In-process function call | 1 |
| `BullMQ` | Queue-based, persistent | 2 |
| `Temporal` | Distributed, durable workflows | 3 |

### 3.5 Action Registry

Register executable actions that workflows can invoke.

```go
type ActionRegistry interface {
    Register(name string, handler ActionHandler) error
    Execute(ctx context.Context, name string, args map[string]any) (ActionResult, error)
    List() []ActionMeta
}

type ActionHandler func(ctx context.Context, args map[string]any) (ActionResult, error)

// Built-in actions
// "notify"    - send notification (webhook, email, slack)
// "http"      - make HTTP request
// "log"       - write to audit log
// "publish"   - publish another event
```

### 3.6 NL → DSL Compiler

Convert natural language rule descriptions to DSL YAML. **LLM only at compile time.**

```go
type Compiler interface {
    Compile(ctx context.Context, naturalLanguage string, schema CompileSchema) (Rule, error)
    Validate(dsl string) (Pattern, error)  // parse + validate without LLM
}
```

**Pipeline:**
```
Natural Language
    |
    v (LLM call - compile time only)
DSL (YAML)
    |
    v (deterministic parser)
AST
    |
    v (validator: type check, window limits, etc.)
Pattern (registered into CEP engine)
```

**DSL Example:**
```yaml
name: stale_deal_alert
description: "Alert when deal has no stage change for 21 days"
pattern:
  kind: absence
  event:
    type: "fact.asserted"
    condition: "data.predicate == 'stage_changed_to'"
  window: 21d
  filter:
    condition: "data.stage NOT IN ['closed_won', 'closed_lost']"
actions:
  - kind: notify
    to: "$deal.owner"
    template: stale_deal
  - kind: publish
    event:
      type: "alert.stale_deal"
      data:
        deal_id: "$deal.id"
```

---

## 4. Multi-Tenancy

Built-in from day 1. Every event, rule, and NFA state is scoped by `tenantID`.

```
Isolation layers:
1. Event Bus    - events routed by tenantID
2. Rule Store   - rules scoped by tenantID
3. CEP Engine   - NFA instances per tenant
4. Action Exec  - action context carries tenantID
```

---

## 5. Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| NFA over Flink/Esper | Self-implemented ~200 LOC | Lightweight, embeddable, no JVM dependency |
| LLM only at compile time | Zero LLM at runtime | Predictable latency, no cost per event |
| CloudEvents format | Standard 1.0 | Interop with any system |
| Pluggable adapters | Interface-based | Start simple (InMemory), scale later (Kafka/Temporal) |
| Go only | No TypeScript/Python | Single-binary deployment, embeddable as library |

---

## 6. Buzz as Library vs Service

Buzz is designed as a **Go library** first, optionally deployable as a standalone service.

```go
// As embedded library (primary use case)
import "github.com/knotmark-ai/buzz"

engine := buzz.New(buzz.Config{
    EventBus:   buzz.InMemoryBus(),
    RuleStore:  buzz.PostgresStore(db),
    Workflow:   buzz.DirectExecutor(),
})

engine.RegisterAction("notify", myNotifyHandler)
engine.Start(ctx)

// Publish events from your app
engine.Publish(ctx, buzz.Event{
    Type:     "fact.asserted",
    TenantID: "tenant_acme",
    Data:     json.RawMessage(`{"subject": "jeff", "predicate": "email"}`),
})
```

```go
// As standalone HTTP service (optional)
import "github.com/knotmark-ai/buzz/server"

srv := server.New(engine, server.Config{
    Addr: ":9090",
})
srv.ListenAndServe()

// REST API:
// POST /events       - publish event
// POST /rules        - create rule (with NL compilation)
// GET  /rules        - list rules
// GET  /rules/:id    - get rule
// DELETE /rules/:id  - delete rule
```

---

## 7. Project Structure

```
buzz/
├── buzz.go              # Engine: top-level orchestrator
├── event.go             # Event, EventBus interface
├── pattern.go           # Pattern, PatternKind, Condition
├── cep.go               # CEP engine (NFA implementation)
├── rule.go              # Rule, RuleStore interface
├── workflow.go          # WorkflowEngine, Step, WorkflowDef
├── action.go            # ActionRegistry, ActionHandler
├── compiler.go          # NL -> DSL compiler interface
│
├── bus/                 # EventBus adapters
│   ├── memory.go        # InMemory (Phase 1)
│   ├── redis.go         # Redis Streams (Phase 2)
│   └── kafka.go         # Kafka/Redpanda (Phase 3)
│
├── store/               # RuleStore adapters
│   ├── memory.go        # InMemory (testing)
│   ├── postgres.go      # PostgreSQL (production)
│   └── sqlite.go        # SQLite (embedded/edge)
│
├── executor/            # WorkflowEngine adapters
│   ├── direct.go        # In-process (Phase 1)
│   └── temporal.go      # Temporal (Phase 3)
│
├── compiler/            # NL -> DSL implementation
│   ├── parser.go        # YAML DSL parser
│   ├── validator.go     # AST validation
│   └── llm.go           # LLM-based NL compilation
│
├── server/              # Optional HTTP service
│   ├── server.go
│   └── handlers.go
│
├── docs/
│   └── DESIGN.md        # This file
│
└── examples/
    ├── basic/           # Minimal usage example
    └── honeycomb/       # How Honeycomb integrates Buzz
```

---

## 8. Phased Implementation

### Phase 1: Core (MVP)

```
- [ ] Event type + CloudEvents format
- [ ] InMemory EventBus
- [ ] CEP Engine with filter + absence primitives
- [ ] NFA core (~200 LOC)
- [ ] Rule type + InMemory RuleStore
- [ ] PostgreSQL RuleStore
- [ ] Direct WorkflowEngine (in-process)
- [ ] ActionRegistry with built-in actions (notify, http, log, publish)
- [ ] DSL parser + validator (no NL compilation yet)
- [ ] Multi-tenant isolation
- [ ] Basic tests
```

### Phase 2: Production Ready

```
- [ ] Full 6 CEP primitives (sequence, parallel, any, repeat)
- [ ] Redis Streams EventBus adapter
- [ ] NFA state persistence (crash recovery)
- [ ] NL -> DSL compiler (LLM integration via Waggle)
- [ ] Rule versioning
- [ ] Metrics + observability (OpenTelemetry)
- [ ] HTTP server mode
```

### Phase 3: Scale

```
- [ ] Kafka/Redpanda EventBus adapter
- [ ] Temporal WorkflowEngine adapter
- [ ] SQLite RuleStore (edge deployment)
- [ ] Rule management UI (optional)
- [ ] Performance benchmarks
```

---

## 9. Integration with Honeycomb

Honeycomb uses Buzz as its event gateway layer. The integration is one-directional:
**Buzz knows nothing about Honeycomb's Ontology.** Honeycomb publishes events and registers rules.

```go
// In Honeycomb's initialization
import "github.com/knotmark-ai/buzz"

// Create Buzz engine
buzzEngine := buzz.New(buzz.Config{
    EventBus:  buzz.InMemoryBus(),
    RuleStore: buzz.PostgresStore(honeycombDB),
    Workflow:  buzz.DirectExecutor(),
})

// Register Honeycomb-specific actions
buzzEngine.RegisterAction("assert_fact", func(ctx context.Context, args map[string]any) (buzz.ActionResult, error) {
    // Write fact to Ontology Engine
    return ontologyEngine.AssertFact(ctx, args)
})

buzzEngine.RegisterAction("notify_agent", func(ctx context.Context, args map[string]any) (buzz.ActionResult, error) {
    // Push notification to Agent Runtime
    return agentRuntime.Notify(ctx, args)
})

// In Ontology Engine's fact write path
func (e *OntologyEngine) WriteFact(ctx context.Context, fact Fact) error {
    // 1. Write to facts table
    err := e.db.Insert(fact)

    // 2. Publish event to Buzz
    buzzEngine.Publish(ctx, buzz.Event{
        Type:     "fact.asserted." + fact.PredicateID,
        TenantID: fact.TenantID,
        Data:     marshalFact(fact),
    })

    return err
}
```

---

## 10. Relationship to Other Projects

```
Waggle (LLM Gateway)
  └── Used by: Buzz (NL -> DSL compiler calls LLM via Waggle)
  └── Used by: Honeycomb (Agent Runtime calls LLM via Waggle)

Buzz (Event Engine)
  └── Uses: Waggle (for NL compilation only)
  └── Used by: Honeycomb (as event gateway)

Honeycomb (Knowledge Platform)
  └── Uses: Waggle (for LLM calls)
  └── Uses: Buzz (for event processing)
```
