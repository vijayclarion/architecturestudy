# Modular Monolith — Architecture Reference

A working guide for .NET Core + Clean Architecture. Covers structure, module communication, cross-module queries, failure handling, and the migration path to microservices.

---

## Table of Contents

1. [What It Is](#1-what-it-is)
2. [Decision: Modular Monolith vs Microservices](#2-decision-modular-monolith-vs-microservices)
3. [Solution Structure](#3-solution-structure)
4. [What Goes Where](#4-what-goes-where)
5. [Inter-Module Communication](#5-inter-module-communication)
6. [Scenario A: Write Flow (Order → Stock)](#6-scenario-a-write-flow-order--stock)
7. [Scenario B: Failure & Compensation](#7-scenario-b-failure--compensation)
8. [Scenario C: Cross-Module Read (Join)](#8-scenario-c-cross-module-read-join)
9. [Scenario D: Cross-Module Filter](#9-scenario-d-cross-module-filter)
10. [Sync vs Async — What You Actually Get](#10-sync-vs-async--what-you-actually-get)
11. [The Outbox Pattern](#11-the-outbox-pattern)
12. [Database Strategy](#12-database-strategy)
13. [Enforcing Boundaries](#13-enforcing-boundaries)
14. [Testing](#14-testing)
15. [Background Jobs](#15-background-jobs)
16. [Migrating an Existing Monolith](#16-migrating-an-existing-monolith)
17. [Extraction to Microservice](#17-extraction-to-microservice)
18. [Anti-Patterns](#18-anti-patterns)
19. [Checklist](#19-checklist)

---

## 1. What It Is

One deployable unit. One process. Internally divided into modules with enforced boundaries.

```
┌─────────────────────────────────────────────────┐
│              Single Process (Api.exe)           │
│                                                 │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐     │
│  │  Orders  │   │Inventory │   │  Lookup  │     │
│  │          │   │          │   │          │     │
│  │ Domain   │   │ Domain   │   │ Domain   │     │
│  │ App      │   │ App      │   │ App      │     │
│  │ Infra    │   │ Infra    │   │ Infra    │     │
│  │ PublicApi│   │ PublicApi│   │ PublicApi│     │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘     │
│       │              │              │           │
│       └──────────────┼──────────────┘           │
│                 IEventBus                       │
└─────────────────────┬───────────────────────────┘
                      │
              ┌───────▼────────┐
              │   PostgreSQL   │
              │ ┌────────────┐ │
              │ │ orders.*   │ │  ← schema per module
              │ │ inventory.*│ │
              │ │ lookup.*   │ │
              │ └────────────┘ │
              └────────────────┘
```

**Key distinction:** Clean Architecture runs *inside each module*, not across the whole solution. You don't have one `Application` project — you have `Orders.Application`, `Inventory.Application`, etc.

**The one rule:** a module may only be reached through its `PublicApi` project or through events. Everything else is private.

---

## 2. Decision: Modular Monolith vs Microservices

### Comparison

| Dimension | Modular Monolith | Microservices |
|---|---|---|
| **Deployment** | One unit | Independent per service |
| **Scaling** | Whole app, horizontally | Per-service |
| **Database** | One DB, schema per module | DB per service |
| **Transactions** | Local ACID available | Distributed sagas mandatory |
| **Latency between modules** | In-process (~ns) | Network (~ms) |
| **Debugging** | Single stack trace | Distributed tracing required |
| **Testing** | In-process integration tests | Contract tests, test containers |
| **Refactoring across boundaries** | Compiler catches it | Coordinated deploy, versioned APIs |
| **Ops overhead** | One pipeline, one log stream | Service mesh, discovery, per-service CI |
| **Team autonomy** | Shared repo, shared release | Independent release cadence |
| **Failure blast radius** | Process-wide | Contained per service |
| **Onboarding** | Clone one repo, F5 | Run 12 containers |

### Pros — Modular Monolith

- **Operational simplicity.** One deploy, one connection string, one log stream, one dashboard.
- **Fast development.** Single codebase, IDE navigation works, refactors are compiler-checked.
- **Local transactions.** `SaveChangesAsync()` is ACID. No saga needed for simple cases.
- **No network latency** between modules.
- **Easy testing.** Spin the whole app in-process. No Docker orchestration to run a test.
- **Cheap.** One instance, one database, no message broker required day one.
- **Real boundaries anyway.** You get separation of concerns without distributed cost.
- **Extractable.** Modules with clean edges become services later without a rewrite.

### Cons — Modular Monolith

- **Coupled scaling.** Inventory needs 8x capacity? Deploy 8x of everything.
- **Coupled deployment.** Fix a typo in Lookup, redeploy Orders.
- **Shared runtime.** Memory leak in one module kills the process.
- **Boundary decay.** Nothing physically prevents a bad reference — you must enforce it.
- **Single tech stack.** No Python ML module inside a .NET host.
- **Shared database contention.** One noisy module's queries slow everyone.

### Pros — Microservices

- Independent scaling and deployment
- Fault isolation (one service down ≠ all down)
- Technology heterogeneity
- Team autonomy at scale (Conway's Law working *for* you)

### Cons — Microservices

- Distributed transactions — sagas everywhere, not just where needed
- Network partition handling, retries, circuit breakers, idempotency
- Eventual consistency leaks into UX
- Debugging spans processes — needs distributed tracing infrastructure
- Every cross-service query becomes an API call or data duplication
- Operational cost: service discovery, mesh, per-service pipelines, observability

### Decision Guide

**Choose Modular Monolith when:**
- Team < ~30 engineers
- No proven per-component scaling bottleneck
- You're refactoring an existing tangled monolith
- Latency budget is tight
- You want boundaries before you want distribution

**Choose Microservices when:**
- Multiple teams need independent release cadence
- One component has a genuinely different scaling profile (10x traffic, GPU-bound)
- Regulatory isolation is required
- You already have the ops maturity — tracing, mesh, on-call rotation

**The honest default:** start modular monolith. Extract when a specific, measured pain forces it. Extraction from a well-modularized monolith is straightforward. Un-distributing a premature microservice mesh is not.

> A modular monolith is not a stepping stone you're obligated to leave. Many systems should stay there permanently.

---

## 3. Solution Structure

```
src/
├── Modules/
│   ├── Orders/
│   │   ├── Orders.Domain/                  # entities, value objects, domain events
│   │   │   ├── Order.cs
│   │   │   ├── OrderItem.cs
│   │   │   └── DomainException.cs
│   │   │
│   │   ├── Orders.Application/             # use cases, ports, event handlers
│   │   │   ├── Abstractions/
│   │   │   │   └── IOrderRepository.cs     # port — interface lives HERE
│   │   │   ├── Commands/
│   │   │   │   └── PlaceOrderHandler.cs
│   │   │   ├── Queries/
│   │   │   │   └── OrderQueryService.cs
│   │   │   └── EventHandlers/
│   │   │       ├── StockReservedHandler.cs
│   │   │       └── StockUnavailableHandler.cs
│   │   │
│   │   ├── Orders.Infrastructure/          # EF Core, adapters
│   │   │   ├── OrdersDbContext.cs
│   │   │   ├── EfOrderRepository.cs
│   │   │   ├── Configurations/
│   │   │   └── OrdersModule.cs             # DI registration
│   │   │
│   │   └── Orders.PublicApi/               # ← ONLY project others may reference
│   │       ├── IOrderModuleApi.cs
│   │       └── OrderSummaryDto.cs
│   │
│   ├── Inventory/          [same 4 projects]
│   └── Lookup/             [same 4 projects]
│
├── Shared/
│   ├── Shared.Contracts/                   # event records — zero logic, zero deps
│   │   └── Events.cs
│   └── Shared.Infrastructure/
│       ├── IEventBus.cs
│       ├── IEventHandler.cs
│       ├── InMemoryEventBus.cs
│       └── RabbitMqEventBus.cs             # swap later
│
├── Api/                                    # host — the ONLY place all modules meet
│   ├── Controllers/
│   └── Program.cs                          # composition root
│
└── Tests/
    ├── Orders.UnitTests/
    ├── Orders.IntegrationTests/
    └── Architecture.Tests/                 # boundary enforcement
```

### The Reference Graph

```
Orders.Domain          → (nothing)
Orders.Application     → Orders.Domain, Shared.Contracts, Lookup.PublicApi
Orders.Infrastructure  → Orders.Application, Shared.Infrastructure
Orders.PublicApi       → (nothing, or Shared.Contracts)
Api                    → Orders.Infrastructure, Inventory.Infrastructure, Lookup.Infrastructure
```

**Critical:** `Orders.Application` referencing `Lookup.PublicApi` is legal. Referencing `Lookup.Application` is not — and in .NET, *the reference simply doesn't exist in the .csproj*. The compiler enforces what a linter had to police in other stacks.

---

## 4. What Goes Where

| Layer | Contains | May reference | Never contains |
|---|---|---|---|
| **Domain** | Entities, value objects, domain events, invariants, `DomainException` | Nothing | EF attributes, `DbContext`, HTTP, `IEventBus` |
| **Application** | Command/query handlers, event handlers, **port interfaces** (`IOrderRepository`), DTOs | Own Domain, `Shared.Contracts`, other modules' `PublicApi` | EF Core, SQL, `HttpClient` |
| **Infrastructure** | `DbContext`, repository *implementations*, EF configs, external clients, DI registration | Own Application, `Shared.Infrastructure` | Business rules |
| **PublicApi** | One interface + DTOs. That's it. | Nothing | Any implementation |
| **Shared.Contracts** | Event `record`s | Nothing | Logic, EF, framework types |
| **Api** | Controllers, `Program.cs`, middleware | All modules' Infrastructure | Business logic |

**Ports live in Application, adapters in Infrastructure.** `IOrderRepository` is declared in `Orders.Application/Abstractions/`. `EfOrderRepository` implements it in `Orders.Infrastructure/`. This is dependency inversion — Application defines the contract, Infrastructure satisfies it, and Application never learns EF Core exists.

**PublicApi is deliberately anemic.** If `Lookup.PublicApi` starts growing methods, that's a signal Orders is doing Lookup's job.

---

## 5. Inter-Module Communication

Two mechanisms. Choose by direction of dependency.

| | Direct Call (`PublicApi`) | Event (`IEventBus`) |
|---|---|---|
| **Use for** | Reads — "give me data" | Writes — "this happened" |
| **Coupling** | Caller knows callee exists | Publisher doesn't know subscribers |
| **Return value** | Yes | No |
| **Failure** | Exception propagates to caller | Publish compensating event |
| **Example** | `lookup.GetManyAsync(ids)` | `OrderCreated` |

**Rule of thumb:** if you need an answer, call. If you're announcing a fact, publish.

Never let a module reach into another's `DbContext`, tables, or `Application` layer. Both mechanisms above exist precisely to make that unnecessary.

### Contracts

```csharp
// Shared.Contracts/Events.cs
namespace Shared.Contracts;

public interface IEvent { }

public record OrderCreated(Guid OrderId, IReadOnlyList<OrderLine> Items) : IEvent;
public record StockReserved(Guid OrderId) : IEvent;
public record StockUnavailable(Guid OrderId, Guid ProductId) : IEvent;

public record OrderLine(Guid ProductId, int Qty);
```

Records, not classes. Immutable, value-equality, free `ToString()` for logging. No behaviour — an event is a fact, not an object.

### The Bus

```csharp
// Shared.Infrastructure/IEventBus.cs
public interface IEventBus
{
    Task PublishAsync<T>(T @event, CancellationToken ct = default) where T : IEvent;
}

public interface IEventHandler<in T> where T : IEvent
{
    Task HandleAsync(T @event, CancellationToken ct = default);
}
```

```csharp
// Shared.Infrastructure/InMemoryEventBus.cs
public sealed class InMemoryEventBus(IServiceScopeFactory scopeFactory) : IEventBus
{
    public async Task PublishAsync<T>(T @event, CancellationToken ct = default) where T : IEvent
    {
        // New scope per event: each handler gets its OWN DbContext.
        using var scope = scopeFactory.CreateScope();
        var handlers = scope.ServiceProvider.GetServices<IEventHandler<T>>();

        foreach (var handler in handlers)
            await handler.HandleAsync(@event, ct);
    }
}
```

> **Why the scope matters.** Resolve handlers from the *publisher's* scope and Inventory's handler shares Orders' `DbContext` — two modules writing through one change tracker. Your boundary is intact on paper and gone at runtime.

---

## 6. Scenario A: Write Flow (Order → Stock)

### Domain

```csharp
// Orders.Domain/Order.cs
public enum OrderStatus { Pending, Confirmed, Failed }

public sealed class Order
{
    public Guid Id { get; private set; }
    public Guid LookupId { get; private set; }
    public OrderStatus Status { get; private set; }

    private readonly List<OrderItem> _items = [];
    public IReadOnlyList<OrderItem> Items => _items;

    private Order() { }   // EF Core

    public static Order Create(Guid id, Guid lookupId, IEnumerable<OrderItem> items)
    {
        var list = items.ToList();
        if (list.Count == 0)
            throw new DomainException("Order must contain at least one item");

        var order = new Order { Id = id, LookupId = lookupId, Status = OrderStatus.Pending };
        order._items.AddRange(list);
        return order;
    }

    public void Confirm()
    {
        if (Status is not OrderStatus.Pending)
            throw new DomainException($"Cannot confirm order in state {Status}");
        Status = OrderStatus.Confirmed;
    }

    public void Fail() => Status = OrderStatus.Failed;
}

public sealed record OrderItem(Guid ProductId, int Qty);
```

Private setters, private constructor, static factory. Invariants cannot be bypassed — there is no `order.Status = Confirmed` available to a careless service.

### Port

```csharp
// Orders.Application/Abstractions/IOrderRepository.cs
public interface IOrderRepository
{
    Task<Order?> FindByIdAsync(Guid id, CancellationToken ct);
    Task AddAsync(Order order, CancellationToken ct);
    Task SaveChangesAsync(CancellationToken ct);
}
```

### Command Handler

```csharp
// Orders.Application/Commands/PlaceOrderHandler.cs
public sealed record PlaceOrderCommand(Guid OrderId, Guid LookupId, IReadOnlyList<OrderLine> Items);

public sealed class PlaceOrderHandler(IOrderRepository repo, IEventBus bus)
{
    public async Task<Guid> HandleAsync(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var order = Order.Create(
            cmd.OrderId,
            cmd.LookupId,
            cmd.Items.Select(i => new OrderItem(i.ProductId, i.Qty)));

        await repo.AddAsync(order, ct);
        await repo.SaveChangesAsync(ct);        // ← persist FIRST, as Pending

        await bus.PublishAsync(new OrderCreated(
            order.Id,
            order.Items.Select(i => new OrderLine(i.ProductId, i.Qty)).ToList()), ct);

        return order.Id;                        // still Pending — Inventory decides
    }
}
```

> **Save before publish.** Publish first and the save throws, and Inventory has reserved stock for an order that doesn't exist. (This ordering still has a crash window — see [Outbox](#11-the-outbox-pattern).)

### Inventory Reacts

```csharp
// Inventory.Application/EventHandlers/OrderCreatedHandler.cs
public sealed class OrderCreatedHandler(IStockRepository stock, IEventBus bus)
    : IEventHandler<OrderCreated>
{
    public async Task HandleAsync(OrderCreated e, CancellationToken ct)
    {
        try
        {
            foreach (var line in e.Items)
            {
                var item = await stock.FindAsync(line.ProductId, ct)
                    ?? throw new OutOfStockException(line.ProductId);

                item.Reserve(line.Qty);          // throws if insufficient
            }

            await stock.SaveChangesAsync(ct);    // atomic — Inventory's own tables

            await bus.PublishAsync(new StockReserved(e.OrderId), ct);
        }
        catch (OutOfStockException ex)
        {
            await bus.PublishAsync(new StockUnavailable(e.OrderId, ex.ProductId), ct);
        }
    }
}
```

### Orders Completes

```csharp
// Orders.Application/EventHandlers/StockReservedHandler.cs
public sealed class StockReservedHandler(IOrderRepository repo) : IEventHandler<StockReserved>
{
    public async Task HandleAsync(StockReserved e, CancellationToken ct)
    {
        var order = await repo.FindByIdAsync(e.OrderId, ct);
        if (order is null) return;              // idempotent: unknown id = no-op

        order.Confirm();
        await repo.SaveChangesAsync(ct);
    }
}
```

### The Flow

```
Controller.PlaceOrder()
└─ PlaceOrderHandler.HandleAsync()
   ├─ repo.SaveChangesAsync()               → COMMIT #1: Order = Pending
   └─ bus.PublishAsync(OrderCreated)
      └─ Inventory.OrderCreatedHandler
         ├─ stock.SaveChangesAsync()        → COMMIT #2: stock reserved
         └─ bus.PublishAsync(StockReserved)
            └─ Orders.StockReservedHandler
               └─ repo.SaveChangesAsync()   → COMMIT #3: Order = Confirmed
            ← return
      ← return
   ← return
← 200 OK { status: "Confirmed" }
```

Note: **three separate commits**, nested call stack, one thread. Neither module touched the other's tables.

---

## 7. Scenario B: Failure & Compensation

### Compensating Handler

```csharp
// Orders.Application/EventHandlers/StockUnavailableHandler.cs
public sealed class StockUnavailableHandler(IOrderRepository repo, ILogger<StockUnavailableHandler> log)
    : IEventHandler<StockUnavailable>
{
    public async Task HandleAsync(StockUnavailable e, CancellationToken ct)
    {
        var order = await repo.FindByIdAsync(e.OrderId, ct);
        if (order is null) return;

        order.Fail();                            // ← the compensation
        await repo.SaveChangesAsync(ct);

        log.LogWarning("Order {OrderId} failed: product {ProductId} out of stock",
            e.OrderId, e.ProductId);
    }
}
```

That pair — `OrderCreatedHandler` + `StockUnavailableHandler` — **is the saga.** No orchestrator class needed for two steps.

### State Machine

```
                    ┌─────────┐
                    │ Pending │
                    └────┬────┘
              StockReserved  StockUnavailable
                    ┌────┴────┐
                    ▼         ▼
             ┌───────────┐ ┌────────┐
             │ Confirmed │ │ Failed │
             └───────────┘ └────────┘
```

### Failure Matrix

| Failure point | State after | Consistent? | Fix |
|---|---|---|---|
| Before COMMIT #1 | Nothing saved | ✅ | — |
| Crash between COMMIT #1 and publish | Order Pending, event lost | ❌ **Stuck forever** | [Outbox](#11-the-outbox-pattern) |
| Inventory throws `OutOfStock` | Order Failed | ✅ | Handled by compensation |
| Inventory COMMIT #2 fails | Order Pending, no stock reserved | ❌ | Retry / dead-letter |
| Crash between COMMIT #2 and COMMIT #3 | Stock reserved, Order Pending | ❌ **Stock leak** | Reservation TTL + sweeper |
| `StockReservedHandler` throws | Stock reserved, Order Pending | ❌ **Stock leak** | Compensating `ReleaseStock` |

> **The hole in the naive saga.** `StockReservedHandler` failing has *no* compensation — nothing releases the reserved stock. Fix with either a `ReservationExpired` background sweeper (reservations carry a TTL) or an explicit `OrderConfirmationFailed` → `ReleaseStock` event pair.

### Why Not One Transaction?

You *could* wrap all three commits in a `TransactionScope` — same process, same database, it works.

But then you've built distributed-transaction semantics **across module boundaries**. The day you extract Inventory into its own service, that transaction cannot follow it, and you rewrite failure handling from scratch.

Accept the three commits. Accept the window. That's what the saga is *for*.

### Idempotency

Handlers must tolerate replay. With RabbitMQ or an outbox, at-least-once delivery is a guarantee, not an edge case.

```csharp
public async Task HandleAsync(StockReserved e, CancellationToken ct)
{
    var order = await repo.FindByIdAsync(e.OrderId, ct);
    if (order is null) return;
    if (order.Status is not OrderStatus.Pending) return;   // ← already processed

    order.Confirm();
    await repo.SaveChangesAsync(ct);
}
```

For non-idempotent operations, persist processed message IDs:

```csharp
// inventory.ProcessedMessages (MessageId PK, ProcessedAt)
if (await db.ProcessedMessages.AnyAsync(m => m.MessageId == e.MessageId, ct))
    return;
```

---

## 8. Scenario C: Cross-Module Read (Join)

You cannot `JOIN orders.Orders o ON lookup.Lookups l` — that's a schema-level dependency and it welds the modules together.

Instead: fetch from your own tables, ask the other module for what you need, join in memory.

### The PublicApi

```csharp
// Lookup.PublicApi/ILookupModuleApi.cs
public interface ILookupModuleApi
{
    Task<IReadOnlyDictionary<Guid, LookupDto>> GetManyAsync(
        IReadOnlyList<Guid> ids, CancellationToken ct);

    Task<IReadOnlySet<Guid>> FindIdsByStatusAsync(
        string status, CancellationToken ct);
}

public sealed record LookupDto(Guid Id, string Name, string Status);
```

Note `GetManyAsync`, not `GetAsync`. **The batch shape is the point** — a singular method invites N+1.

### The Query

```csharp
// Orders.Application/Queries/OrderQueryService.cs
public sealed class OrderQueryService(OrdersReadDbContext db, ILookupModuleApi lookup)
{
    // ❌ N+1 — one Lookup round-trip per order
    public async Task<List<OrderDto>> BadListAsync(CancellationToken ct)
    {
        var orders = await db.Orders.AsNoTracking().ToListAsync(ct);

        var result = new List<OrderDto>();
        foreach (var o in orders)
        {
            var l = await lookup.GetManyAsync([o.LookupId], ct);   // ← 500 orders = 500 calls
            result.Add(new OrderDto(o.Id, o.Status, l.GetValueOrDefault(o.LookupId)));
        }
        return result;
    }

    // ✅ Batched — one Lookup round-trip total
    public async Task<List<OrderDto>> ListWithLookupsAsync(CancellationToken ct)
    {
        var orders = await db.Orders.AsNoTracking()
            .Select(o => new { o.Id, o.LookupId, o.Status })
            .ToListAsync(ct);

        var ids = orders.Select(o => o.LookupId).Distinct().ToList();
        var lookups = await lookup.GetManyAsync(ids, ct);

        return orders
            .Select(o => new OrderDto(o.Id, o.Status, lookups.GetValueOrDefault(o.LookupId)))
            .ToList();
    }
}
```

**This is an application-level join, not a SQL join.** Slower than a database join. It buys you a boundary. That's the trade.

### Caching

Lookups are reference data — they change rarely.

```csharp
public sealed class CachedLookupApi(ILookupModuleApi inner, IMemoryCache cache) : ILookupModuleApi
{
    public async Task<IReadOnlyDictionary<Guid, LookupDto>> GetManyAsync(
        IReadOnlyList<Guid> ids, CancellationToken ct)
    {
        var result = new Dictionary<Guid, LookupDto>();
        var missing = new List<Guid>();

        foreach (var id in ids)
        {
            if (cache.TryGetValue($"lookup:{id}", out LookupDto? hit) && hit is not null)
                result[id] = hit;
            else
                missing.Add(id);
        }

        if (missing.Count > 0)
        {
            var fetched = await inner.GetManyAsync(missing, ct);
            foreach (var (id, dto) in fetched)
            {
                cache.Set($"lookup:{id}", dto, TimeSpan.FromMinutes(10));
                result[id] = dto;
            }
        }
        return result;
    }
    // ...
}
```

Register as a decorator (Scrutor):

```csharp
services.AddScoped<ILookupModuleApi, LookupModuleApi>();
services.Decorate<ILookupModuleApi, CachedLookupApi>();
```

Orders never knows caching exists.

---

## 9. Scenario D: Cross-Module Filter

The hardest case. *"Give me all orders where the lookup status is ACTIVE."*

### ❌ Naive — Filter in Memory

```csharp
var orders = await db.Orders.AsNoTracking().ToListAsync(ct);      // 10 million rows
var ids = orders.Select(o => o.LookupId).Distinct().ToList();
var lookups = await lookup.GetManyAsync(ids, ct);

return orders.Where(o => lookups[o.LookupId].Status == "ACTIVE").ToList();
```

Works at 10k rows. **Dies at 10M.** You loaded the entire table to discard 95% of it.

### ✅ Push the Predicate Down

```csharp
public async Task<List<OrderDto>> ListWhereLookupActiveAsync(CancellationToken ct)
{
    // 1. Lookup answers: "which ids match?"
    var activeIds = await lookup.FindIdsByStatusAsync("ACTIVE", ct);

    // 2. Orders answers: "which rows have those ids?" — indexed WHERE ... IN (...)
    var orders = await db.Orders.AsNoTracking()
        .Where(o => activeIds.Contains(o.LookupId))
        .ToListAsync(ct);

    return orders.Select(o => new OrderDto(o.Id, o.Status, null)).ToList();
}
```

Each module queries only its own tables. The **database still does the heavy lifting** via a real indexed `IN (...)`.

EF Core translates `activeIds.Contains(x)` to `WHERE lookup_id IN (@p0, @p1, ...)`.

> **The ceiling.** SQL Server caps parameters around **2,100**. PostgreSQL handles more but degrades. Once `activeIds` exceeds a few thousand, this breaks — and that is your signal to build a read model.

### ✅ Read Model (Denormalized Projection)

Owned by *neither* module. Populated from *both* modules' events. Queried directly.

```csharp
// ReadModels/OrderListItem.cs — schema: readmodel.OrderListItems
public sealed class OrderListItem
{
    public Guid OrderId { get; set; }
    public string OrderStatus { get; set; } = "";
    public Guid LookupId { get; set; }
    public string LookupName { get; set; } = "";
    public string LookupStatus { get; set; } = "";     // ← denormalized copy
    public DateTime CreatedAt { get; set; }
}
```

```csharp
// Projections/OrderListProjection.cs
public sealed class OrderListProjection(ReadModelDbContext db)
    : IEventHandler<OrderCreated>, IEventHandler<LookupStatusChanged>
{
    public async Task HandleAsync(OrderCreated e, CancellationToken ct)
    {
        db.OrderListItems.Add(new OrderListItem
        {
            OrderId = e.OrderId,
            OrderStatus = "Pending",
            CreatedAt = DateTime.UtcNow
        });
        await db.SaveChangesAsync(ct);
    }

    public async Task HandleAsync(LookupStatusChanged e, CancellationToken ct)
    {
        // Fan out the status change to every affected row
        await db.OrderListItems
            .Where(x => x.LookupId == e.LookupId)
            .ExecuteUpdateAsync(s => s.SetProperty(x => x.LookupStatus, e.NewStatus), ct);
    }
}
```

Now the query is trivial and fast:

```csharp
public Task<List<OrderListItem>> ListActiveAsync(CancellationToken ct) =>
    db.OrderListItems.AsNoTracking()
        .Where(x => x.LookupStatus == "ACTIVE")       // single indexed table, no cross-module call
        .OrderByDescending(x => x.CreatedAt)
        .Take(50)
        .ToListAsync(ct);
```

**Cost:** eventual consistency. The read model lags the write model by however long the projection takes. Acceptable for lists, reports, search. **Not** acceptable for the data you make a write decision on.

### Escalation Ladder

```
1. In-memory join (batched)      → default; fine to ~10k rows
2. + cache the PublicApi         → reference data, high read ratio
3. Push predicate down, IN(...)  → until you hit the parameter ceiling
4. Read model / projection       → large result sets, complex filters, reporting
```

Do not start at 4.

> **A word of caution.** Heavy cross-module filtering is a *smell*. If Orders constantly needs Lookup's columns to answer basic questions, the boundary may be drawn in the wrong place. Consider whether they're actually one bounded context. Fighting the boundary with read models is sometimes the right answer — and sometimes it's the wrong boundary.

---

## 10. Sync vs Async — What You Actually Get

### The In-Memory Bus is Synchronous

```csharp
await bus.PublishAsync(new OrderCreated(...), ct);
```

That `await` **blocks the caller.** `PublishAsync` resolves handlers from DI and invokes them directly — same thread, same call stack, same HTTP request.

The entire saga completes before your controller returns. It is a method call wearing an event costume.

### `async/await` ≠ Event-Driven

Two different words that happen to overlap. Distinguish them:

| Property | What it means | In-memory bus |
|---|---|---|
| **Asynchronous (C#)** | Yields the thread during I/O; doesn't block a pool worker | ✅ Yes |
| **Asynchronous (architecture)** | Caller proceeds before callee finishes | ❌ **No** |
| **Decoupled (compile-time)** | Orders doesn't reference Inventory | ✅ Yes |
| **Decoupled (temporal)** | Inventory being slow doesn't slow Orders | ❌ **No** |

**Decoupling is a compile-time property. Async is a runtime property.** The in-memory bus gives you the first for free. It does not give you the second.

You get the thing you actually wanted — delete the Inventory module and Orders still compiles. The dependency is on the *contract*, not the module.

### ❌ Never Fire-and-Forget In-Process

```csharp
_ = bus.PublishAsync(@event, ct);   // no await — DON'T
```

- Unobserved exceptions vanish silently
- The DI scope disposes underneath the handler
- `DbContext` disposed mid-query
- Process shuts down, work evaporates

This is how orders get stuck `Pending` forever with no log line explaining why.

### Genuinely Async: Two Options

**1. Outbox + background worker** (in-process, durable) — see [§11](#11-the-outbox-pattern).

**2. Real broker:**

```csharp
public sealed class RabbitMqEventBus(IChannel channel) : IEventBus
{
    public Task PublishAsync<T>(T @event, CancellationToken ct) where T : IEvent
    {
        channel.BasicPublish(
            exchange: "events",
            routingKey: typeof(T).Name,
            body: JsonSerializer.SerializeToUtf8Bytes(@event));

        return Task.CompletedTask;      // ← returns immediately; broker takes it
    }
}
```

Inventory becomes a consumer on a queue. Different process, potentially different machine. Orders never waits.

### The Cost Nobody Mentions

**Go async and the caller can no longer be told the outcome.**

```csharp
// Synchronous bus — saga is DONE by the time you respond
var id = await handler.HandleAsync(cmd, ct);
var order = await repo.FindByIdAsync(id, ct);
return Ok(order);                          // Status: Confirmed. Real.

// Async bus — order is still Pending, Inventory hasn't run
var id = await handler.HandleAsync(cmd, ct);
return Accepted(new { id, status = "Pending" });    // ← 202, not 200
```

The client must now poll `GET /orders/{id}`, hold a SignalR connection, or wait for a webhook. Your UI needs a pending state. Your tests must wait for eventual consistency instead of asserting immediately.

**That is the real cost of event-driven.** Not the infrastructure — the fact that *"did it work?"* stops having an immediate answer.

### What to Pick

Start with `InMemoryEventBus`. Synchronous. Saga completes inside the request, UI stays simple, integration tests assert directly. You already have decoupling.

Go async when you need it: Inventory calls a slow external warehouse API, or Orders must keep accepting traffic while Inventory is down, or you're preparing to extract Inventory.

The bus swap is **one line in `Program.cs`**, because everything depends on `IEventBus` and nothing depends on `InMemoryEventBus`.

The hard part isn't the swap. It's `200 OK` becoming `202 Accepted`, and every API consumer having to learn about it.

---

## 11. The Outbox Pattern

### The Problem

```csharp
await repo.SaveChangesAsync(ct);              // ✅ committed
                                              // ← 💥 process crashes here
await bus.PublishAsync(new OrderCreated(...), ct);   // never runs
```

Order is `Pending` forever. Inventory never heard about it. **In-memory buses lose events.**

### The Fix: Same Transaction

```csharp
// Orders.Infrastructure/OutboxMessage.cs — schema: orders.OutboxMessages
public sealed class OutboxMessage
{
    public Guid Id { get; set; }
    public string Type { get; set; } = "";        // assembly-qualified name
    public string Payload { get; set; } = "";     // JSON
    public DateTime OccurredAt { get; set; }
    public DateTime? ProcessedAt { get; set; }
    public string? Error { get; set; }
    public int Attempts { get; set; }
}
```

```csharp
public sealed class PlaceOrderHandler(OrdersDbContext db)
{
    public async Task<Guid> HandleAsync(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var order = Order.Create(cmd.OrderId, cmd.LookupId, /* ... */);
        db.Orders.Add(order);

        var evt = new OrderCreated(order.Id, /* ... */);
        db.Outbox.Add(new OutboxMessage
        {
            Id = Guid.NewGuid(),
            Type = typeof(OrderCreated).AssemblyQualifiedName!,
            Payload = JsonSerializer.Serialize(evt),
            OccurredAt = DateTime.UtcNow
        });

        await db.SaveChangesAsync(ct);   // ← ONE COMMIT: order + event, atomic

        return order.Id;                 // return immediately
    }
}
```

### The Processor

```csharp
public sealed class OutboxProcessor(IServiceScopeFactory factory, ILogger<OutboxProcessor> log)
    : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            try
            {
                using var scope = factory.CreateScope();
                var db  = scope.ServiceProvider.GetRequiredService<OrdersDbContext>();
                var bus = scope.ServiceProvider.GetRequiredService<IEventBus>();

                var pending = await db.Outbox
                    .Where(m => m.ProcessedAt == null && m.Attempts < 5)
                    .OrderBy(m => m.OccurredAt)
                    .Take(20)
                    .ToListAsync(ct);

                foreach (var msg in pending)
                {
                    try
                    {
                        var type = Type.GetType(msg.Type)!;
                        var evt  = (IEvent)JsonSerializer.Deserialize(msg.Payload, type)!;

                        await bus.PublishAsync((dynamic)evt, ct);
                        msg.ProcessedAt = DateTime.UtcNow;
                    }
                    catch (Exception ex)
                    {
                        msg.Attempts++;
                        msg.Error = ex.Message;
                        log.LogError(ex, "Outbox message {Id} failed (attempt {N})", msg.Id, msg.Attempts);
                    }
                }

                await db.SaveChangesAsync(ct);
            }
            catch (Exception ex)
            {
                log.LogError(ex, "Outbox processor loop failed");
            }

            await Task.Delay(500, ct);
        }
    }
}
```

**Consequences:**
- Delivery becomes **at-least-once**, never zero. Handlers must be idempotent (see [§7](#7-scenario-b-failure--compensation)).
- Publishing is now genuinely async — controller returns while Inventory hasn't run.
- `200 OK` becomes `202 Accepted`.
- Messages exceeding `Attempts >= 5` need a dead-letter path and an alert.

**When to add it:** skip while learning. Add before production, or the day you move to RabbitMQ — whichever comes first.

---

## 12. Database Strategy

**One database. One schema per module. Separate `DbContext` per module.**

```csharp
// Orders.Infrastructure/OrdersDbContext.cs
public sealed class OrdersDbContext(DbContextOptions<OrdersDbContext> options) : DbContext(options)
{
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<OutboxMessage> Outbox => Set<OutboxMessage>();

    protected override void OnModelCreating(ModelBuilder b)
    {
        b.HasDefaultSchema("orders");        // → orders.Orders, orders.OutboxMessages
        b.ApplyConfigurationsFromAssembly(typeof(OrdersDbContext).Assembly);
    }
}
```

### The Rules

| Rule | Why |
|---|---|
| Schema per module | Costs nothing; makes eventual extraction trivial |
| Separate `DbContext` per module | Prevents cross-module change tracking |
| **No FK across schemas** | An FK is a hard dependency; you can't split it later |
| **No navigation property to another module's entity** | `order.Lookup` welds Orders to Lookup |
| Store the foreign *id*, not the entity | `Guid LookupId` — resolve via `PublicApi` |
| Migrations per module | Each module owns its schema evolution |

```csharp
// ❌ Welds the modules together
public sealed class Order
{
    public Lookup Lookup { get; set; }        // navigation to another module — NO
}

// ✅ Just the id
public sealed class Order
{
    public Guid LookupId { get; private set; }   // resolve via ILookupModuleApi
}
```

EF Core will **happily** let you add the navigation property. Nothing stops you but discipline and the architecture test in §13.

### Migrations

```bash
dotnet ef migrations add AddOrders \
  --project src/Modules/Orders/Orders.Infrastructure \
  --startup-project src/Api \
  --context OrdersDbContext \
  --output-dir Migrations
```

Each module's migrations live in its own Infrastructure project. Apply all at startup or via a migration job.

### When to Split the Database

Only when extracting a module to a service. Splitting earlier costs you local transactions and buys nothing.

---

## 13. Enforcing Boundaries

Project references do most of the work — `Orders.Application.csproj` simply *does not contain* a reference to `Inventory.Application`. The compiler stops you.

But someone will add one "just for this one query." Catch it in CI.

```csharp
// Architecture.Tests/ModuleBoundaryTests.cs
using NetArchTest.Rules;

public class ModuleBoundaryTests
{
    [Fact]
    public void Orders_must_not_depend_on_Inventory_internals()
    {
        var result = Types.InAssembly(typeof(PlaceOrderHandler).Assembly)
            .Should()
            .NotHaveDependencyOnAny(
                "Inventory.Domain",
                "Inventory.Application",
                "Inventory.Infrastructure")
            .GetResult();

        Assert.True(result.IsSuccessful, string.Join(", ", result.FailingTypeNames ?? []));
    }

    [Fact]
    public void Domain_must_not_depend_on_EntityFramework()
    {
        var result = Types.InAssembly(typeof(Order).Assembly)
            .Should()
            .NotHaveDependencyOn("Microsoft.EntityFrameworkCore")
            .GetResult();

        Assert.True(result.IsSuccessful);
    }

    [Fact]
    public void Application_must_not_depend_on_Infrastructure()
    {
        var result = Types.InAssembly(typeof(PlaceOrderHandler).Assembly)
            .Should()
            .NotHaveDependencyOn("Orders.Infrastructure")
            .GetResult();

        Assert.True(result.IsSuccessful);
    }

    [Fact]
    public void Handlers_must_be_sealed()
    {
        var result = Types.InAssembly(typeof(PlaceOrderHandler).Assembly)
            .That().ImplementInterface(typeof(IEventHandler<>))
            .Should().BeSealed()
            .GetResult();

        Assert.True(result.IsSuccessful);
    }
}
```

Run in CI. **Without this, in eighteen months you have a distributed monolith with none of the benefits.**

---

## 14. Testing

Modules are testable in isolation — that's most of the point.

### Unit — Domain

```csharp
[Fact]
public void Cannot_confirm_an_already_failed_order()
{
    var order = Order.Create(Guid.NewGuid(), Guid.NewGuid(), [new OrderItem(Guid.NewGuid(), 1)]);
    order.Fail();

    Assert.Throws<DomainException>(() => order.Confirm());
}
```

No mocks. No DI container. Pure.

### Unit — Application (fake the bus)

```csharp
public sealed class FakeEventBus : IEventBus
{
    public List<IEvent> Published { get; } = [];

    public Task PublishAsync<T>(T @event, CancellationToken ct = default) where T : IEvent
    {
        Published.Add(@event);
        return Task.CompletedTask;
    }
}

[Fact]
public async Task PlaceOrder_publishes_OrderCreated()
{
    var bus = new FakeEventBus();
    var handler = new PlaceOrderHandler(new InMemoryOrderRepository(), bus);

    await handler.HandleAsync(new PlaceOrderCommand(/* ... */), default);

    Assert.Single(bus.Published.OfType<OrderCreated>());
}
```

Orders is tested **without Inventory existing at all.** That's the boundary paying off.

### Integration — Whole Saga, In-Process

```csharp
[Fact]
public async Task Order_is_confirmed_when_stock_available()
{
    await using var app = new WebApplicationFactory<Program>()
        .WithWebHostBuilder(b => b.ConfigureServices(s =>
        {
            s.RemoveAll<DbContextOptions<OrdersDbContext>>();
            s.AddDbContext<OrdersDbContext>(o => o.UseNpgsql(_container.ConnectionString));
        }));

    var client = app.CreateClient();
    var response = await client.PostAsJsonAsync("/orders", new { /* ... */ });

    // With the SYNC in-memory bus, the saga is already complete.
    var order = await response.Content.ReadFromJsonAsync<OrderDto>();
    Assert.Equal("Confirmed", order!.Status);   // ← assert immediately, no polling
}
```

> **This is the practical benefit of the synchronous bus.** With an outbox or RabbitMQ, this test must poll for eventual consistency. Keep the sync bus in tests as long as you can — register `InMemoryEventBus` in your test host even if production uses the outbox.

### Fake a Module's PublicApi

```csharp
public sealed class FakeLookupApi(Dictionary<Guid, LookupDto> data) : ILookupModuleApi
{
    public Task<IReadOnlyDictionary<Guid, LookupDto>> GetManyAsync(
        IReadOnlyList<Guid> ids, CancellationToken ct) =>
        Task.FromResult<IReadOnlyDictionary<Guid, LookupDto>>(
            ids.Where(data.ContainsKey).ToDictionary(id => id, id => data[id]));

    public Task<IReadOnlySet<Guid>> FindIdsByStatusAsync(string status, CancellationToken ct) =>
        Task.FromResult<IReadOnlySet<Guid>>(
            data.Where(kv => kv.Value.Status == status).Select(kv => kv.Key).ToHashSet());
}
```

`PublicApi` being one small interface makes this trivial. If it's hard to fake, it's too big.

---

## 15. Background Jobs

A running monolith usually has plenty. Two rules:

**1. Jobs belong to a module.** Not a global `Jobs/` folder.

```
Modules/Inventory/Inventory.Infrastructure/Jobs/
    ├── ReservationSweeper.cs        # releases expired stock reservations
    └── StockReconciliation.cs
```

**2. Jobs go through the same ports as everything else.**

```csharp
// Inventory.Infrastructure/Jobs/ReservationSweeper.cs
public sealed class ReservationSweeper(IServiceScopeFactory factory) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            using var scope = factory.CreateScope();
            var stock = scope.ServiceProvider.GetRequiredService<IStockRepository>();
            var bus   = scope.ServiceProvider.GetRequiredService<IEventBus>();

            var expired = await stock.FindExpiredReservationsAsync(ct);
            foreach (var r in expired)
            {
                r.Release();
                await bus.PublishAsync(new ReservationExpired(r.OrderId), ct);
            }
            await stock.SaveChangesAsync(ct);

            await Task.Delay(TimeSpan.FromMinutes(1), ct);
        }
    }
}
```

That sweeper is what closes the "stock leak" hole from [§7](#7-scenario-b-failure--compensation).

**Scaling caveat:** run *N* instances of the monolith and *N* copies of every `BackgroundService` run. Either:
- Gate jobs behind config (`RunBackgroundJobs: true` on one instance only), or
- Use a distributed lock (`SELECT ... FOR UPDATE SKIP LOCKED`, Redis lock, Hangfire, Quartz with a DB store)

The outbox processor has the same problem. `FOR UPDATE SKIP LOCKED` on the outbox poll is the standard fix.

---

## 16. Migrating an Existing Monolith

You have a tangled monolith, one database, background jobs, a large test suite. Do **not** start with microservices.

### Order of Operations

**1. Identify seams — don't move code yet.**
Find bounded contexts. Look for tables that are always read together, and code that changes together. Draw the boundaries on paper first.

**2. Create the folder/project structure — move code, change nothing.**
Pure move. No logic changes. Tests must stay green.

**3. Break references one direction at a time.**
Pick the leaf module (fewest dependents — usually Lookup or something reference-data-shaped). Introduce its `PublicApi`. Route callers through it. Delete the direct references.

**4. Add the architecture test *immediately* after each module is clean.**
Otherwise it decays before you finish the next one.

**5. Introduce `IEventBus` for the write paths.**
Replace direct `_inventoryService.Reserve(...)` calls with `OrderCreated` events. This is where you'll discover which "transactions" were secretly relying on the shared `DbContext`.

**6. Split the schema.**
`orders.*`, `inventory.*`. Drop cross-schema FKs — this step *will* surface hidden coupling. Expect it.

**7. Separate `DbContext` per module.**
The hardest step. Anything that was in one transaction across modules now needs a saga.

**8. Add the outbox** when step 7 makes the crash windows real.

### What Not To Do

- Don't rewrite. Move code, then refactor in place.
- Don't split the database first. Split the *code* first.
- Don't do all modules at once. One at a time, green tests between each.
- Don't add events where a direct `PublicApi` read would do.

Your existing test suite is the safety net for all of this. That's an asset, not an obstacle.

---

## 17. Extraction to Microservice

You did the work above. Now Inventory genuinely needs to scale independently.

| Step | What changes |
|---|---|
| 1. Swap the bus | `services.AddSingleton<IEventBus, RabbitMqEventBus>()` — one line |
| 2. Copy the schema | `inventory.*` → new database. It's already isolated. |
| 3. Move the projects | `Inventory.*` → new solution. No code changes; references were already clean. |
| 4. Replace the `PublicApi` | `ILookupModuleApi` implementation becomes an HTTP/gRPC client. **The interface doesn't change.** |
| 5. Add resilience | Polly retries, circuit breaker, timeouts — in the new HTTP adapter |
| 6. Handle the read | Cross-module joins now cross the network. Cache harder, or build a read model. |
| 7. `200` → `202` | Callers must handle eventual consistency |

Steps 1–4 are mechanical **because the boundaries were already real.** Steps 5–7 are the actual cost of distribution, and you pay them either way.

```csharp
// The interface is unchanged. Only the adapter moves.
public sealed class HttpLookupModuleApi(HttpClient http) : ILookupModuleApi
{
    public async Task<IReadOnlyDictionary<Guid, LookupDto>> GetManyAsync(
        IReadOnlyList<Guid> ids, CancellationToken ct)
    {
        var response = await http.PostAsJsonAsync("/lookups/batch", ids, ct);
        response.EnsureSuccessStatusCode();

        var items = await response.Content.ReadFromJsonAsync<List<LookupDto>>(ct);
        return items!.ToDictionary(x => x.Id);
    }
    // ...
}
```

`Orders.Application` recompiles unchanged. That is the entire payoff of the `PublicApi` project.

---

## 18. Anti-Patterns

| ❌ Anti-pattern | Why it's wrong | ✅ Instead |
|---|---|---|
| Direct `DbContext` access across modules | Boundary gone at runtime | `PublicApi` or event |
| Navigation property to another module's entity | Welds schemas; can't split | Store the `Guid` id |
| FK across schemas | Same | Reference by id only |
| One shared `Application` project | Not modular at all | One per module |
| `_ = bus.PublishAsync(e)` fire-and-forget | Swallows exceptions, disposed scope | `await` it, or use the outbox |
| Handlers resolved from publisher's scope | Shared `DbContext` across modules | `CreateScope()` per event |
| Events for reads | Awkward, no return value | Direct `PublicApi` call |
| Direct calls for writes | Recouples modules | Publish an event |
| N+1 across a `PublicApi` | Fine locally, fatal over HTTP later | Batch: `GetManyAsync` |
| Loading all rows to filter in memory | Dies at scale | Push predicate down, then read model |
| `TransactionScope` across modules | Can't survive extraction | Saga + compensation |
| Non-idempotent handlers | At-least-once delivery replays | Guard on state or message id |
| No architecture tests | Boundaries decay silently | NetArchTest in CI |
| Bloated `PublicApi` | Module doing another's job | Question the boundary |

---

## 19. Checklist

### Structure
- [ ] Four projects per module: Domain, Application, Infrastructure, PublicApi
- [ ] Domain references nothing
- [ ] Ports (interfaces) in Application, adapters in Infrastructure
- [ ] `PublicApi` is one interface + DTOs
- [ ] `Program.cs` is the only file that sees every module

### Boundaries
- [ ] No project reference to another module's Domain/Application/Infrastructure
- [ ] NetArchTest suite in CI
- [ ] No navigation properties across modules
- [ ] No FKs across schemas

### Data
- [ ] Schema per module (`orders.*`, `inventory.*`)
- [ ] Separate `DbContext` per module
- [ ] Migrations owned by each module's Infrastructure project

### Communication
- [ ] Reads → direct `PublicApi` call
- [ ] Writes → event
- [ ] `IEventBus` interface; `InMemoryEventBus` implementation
- [ ] `CreateScope()` per event dispatch
- [ ] Every cross-module read is batched

### Correctness
- [ ] Save before publish
- [ ] Every event handler is idempotent
- [ ] Every saga step that can fail has a compensating action
- [ ] Reservation TTL + sweeper for the stock-leak window

### Before Production
- [ ] Outbox pattern for durable publishing
- [ ] `FOR UPDATE SKIP LOCKED` on outbox poll (multi-instance safe)
- [ ] Dead-letter path + alerting for `Attempts >= 5`
- [ ] Background jobs gated or distributed-locked
- [ ] Controllers return `202 Accepted` where the saga is async

---

## Summary

**Choose modular monolith** unless you have a measured, specific reason not to. You get separation of concerns, testable modules, and clean boundaries — without distributed transactions, network latency, or a service mesh.

**The one rule:** a module is reachable only through its `PublicApi` or through events. Enforce it with project references and an architecture test, or it will decay.

**The synchronous in-memory bus is not a compromise.** It gives you compile-time decoupling — the property you actually wanted — while the saga still completes inside the request and your tests assert immediately. Trade it for durability when you need durability, not before.

**Extraction is a migration, not a rewrite** — but only if you did the boundaries properly first. That's the whole bet.
