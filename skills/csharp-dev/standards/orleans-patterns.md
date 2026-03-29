# Standard: Orleans Grain Patterns

> **Audience:** AI agents writing, reviewing, or refactoring Orleans grain code.
> **Prerequisite:** Read `skills/csharp-dev/standards/idiomatic-csharp.md` for general C# rules.
> **Orleans version:** 8+ (ships with .NET 9/10). Legacy `Grain<T>` patterns are superseded.

## Core Principle

Grains are **virtual actors** with single-threaded execution. Design for **immutable messages, explicit state persistence, and forward-compatible serialisation**. Keep grains small and focused — one responsibility per grain type.

---

## 1. Modern Serialisation

**Rule:** Use `[GenerateSerializer]` on all types that cross grain boundaries. Mark every serialised member with `[Id(n)]`. Never use `[Serializable]` or BinaryFormatter.

### Bad

```csharp
[Serializable]
public class OrderMessage
{
    public string OrderId { get; set; } = "";
    public decimal Total { get; set; }
}
```

### Good

```csharp
[GenerateSerializer]
[Immutable]
public sealed record OrderMessage(
    [property: Id(0)] string OrderId,
    [property: Id(1)] decimal Total);
```

### Guidelines

- **`[Id(n)]` is mandatory** — Orleans source-generates serialisers from these. Omitting them is a runtime error.
- **Never reuse or reorder IDs** — append new fields with the next available ID for backward compatibility.
- **`[Immutable]`** on all message types — Orleans skips defensive copies, improving throughput.
- **Records over classes** — immutability by construction.

---

## 2. Grain Interface Versioning

**Rule:** Always apply `[Alias("fully.qualified.name")]` to grain interfaces. This decouples the CLR type name from the wire identity, enabling safe renames and versioning.

### Bad

```csharp
public interface IOrderGrain : IGrainWithStringKey
{
    Task<OrderState> GetState();
}
```

### Good

```csharp
[Alias("myapp.grains.IOrderGrain")]
public interface IOrderGrain : IGrainWithStringKey
{
    Task<OrderState> GetState();
}
```

### Guidelines

- Use a stable dot-separated name (namespace-like). Once published, never change it.
- Apply `[Alias]` to grain interfaces **and** to serialised types that appear in state or messages.

---

## 3. State Management — IPersistentState<T>

**Rule:** Inject `IPersistentState<T>` via constructor with `[PersistentState]` attribute. Do not inherit `Grain<T>` — that pattern is legacy.

### Bad (legacy)

```csharp
public class OrderGrain : Grain<OrderState>, IOrderGrain
{
    public override Task OnActivateAsync()
    {
        // state available via this.State
    }
}
```

### Good

```csharp
public sealed class OrderGrain(
    [PersistentState("order", "store")] IPersistentState<OrderState> state)
    : Grain, IOrderGrain
{
    public Task<OrderState> GetState() =>
        Task.FromResult(state.State);

    public async Task Submit(OrderMessage message)
    {
        state.State = state.State with { Status = OrderStatus.Submitted };
        await state.WriteStateAsync();
    }
}
```

### Guidelines

- **First arg** is the state name (unique within the grain). **Second arg** is the storage provider name (configured in silo).
- **`state.State with { ... }`** — always copy-on-write via record `with` expressions.
- **Call `WriteStateAsync()` explicitly** — Orleans does not auto-persist. Be intentional about when state is saved.
- **Multiple state objects** are fine — inject several `IPersistentState<T>` parameters for different concerns.
- **State types must be `[GenerateSerializer]`** with `[Id(n)]` on all members.

---

## 4. Grain State as Records

**Rule:** Grain state types are `[GenerateSerializer]` records with sensible defaults. Use `with` expressions for all mutations.

```csharp
[GenerateSerializer]
public sealed record OrderState
{
    [Id(0)] public string OrderId { get; init; } = "";
    [Id(1)] public OrderStatus Status { get; init; } = OrderStatus.Draft;
    [Id(2)] public ImmutableList<LineItem> Items { get; init; } = [];
    [Id(3)] public DateTimeOffset Created { get; init; } = DateTimeOffset.UtcNow;
}
```

### Guidelines

- **Default values** so a freshly activated grain has valid initial state.
- **`ImmutableList<T>`** or `ImmutableArray<T>` for collections in state — prevents accidental mutation without persistence.
- **Avoid nullable reference types in state** — use empty defaults instead.

---

## 5. Grain Design Principles

**Rule:** One grain = one entity = one responsibility. Keep grain methods small and focused.

### Guidelines

- **Grain keys** map to business identity — order ID, user ID, device ID. Don't use GUIDs as keys unless the business concept has no better identity.
- **Prefer `IGrainWithStringKey`** — most flexible, maps naturally to business identifiers.
- **No blocking calls** — all grain methods return `Task` or `Task<T>`. Never call `.Result` or `.Wait()`.
- **Reentrancy is opt-in** — grains are single-threaded by default. Use `[Reentrant]` on the class only when you've verified no state corruption is possible. For individual methods, use `[AlwaysInterleave]`.
- **Timers** — use `this.RegisterGrainTimer()` (replaces legacy `RegisterTimer`). Timer callbacks run on the grain's activation context.
- **OneWay** — mark fire-and-forget methods with `[OneWay]` to avoid waiting for completion.

---

## 6. Grain Placement

**Rule:** Use placement attributes only when you have a measured reason. The default (`RandomPlacement`) works for most cases.

| Attribute | When to use |
|-----------|-------------|
| `[PreferLocalPlacement]` | Grain is always called from the same silo (co-locate for latency) |
| `[HashBasedPlacement]` | Need deterministic placement for stateless partitioning |
| `[ActivationCountBasedPlacement]` | Spread load evenly across silos |

---

## 7. Streams

**Rule:** Use `IAsyncStream<T>` with explicit subscriptions for pub/sub between grains. Always handle stream errors.

```csharp
// Producer
var stream = this.GetStreamProvider("sms")
    .GetStream<OrderEvent>(StreamId.Create("orders", orderId));
await stream.OnNextAsync(new OrderSubmitted(orderId));

// Consumer — in OnActivateAsync
var stream = this.GetStreamProvider("sms")
    .GetStream<OrderEvent>(StreamId.Create("orders", orderId));
await stream.SubscribeAsync(OnOrderEvent);
```

### Guidelines

- **Name stream providers explicitly** — configure in silo builder, reference by string name.
- **`StreamId.Create(namespace, key)`** — use meaningful namespaces for discoverability.
- **Idempotent handlers** — streams provide at-least-once delivery. Design accordingly.

---

## Self-Check Table

| # | Smell | Fix |
|---|-------|-----|
| 1 | `[Serializable]` or BinaryFormatter | `[GenerateSerializer]` + `[Id(n)]` |
| 2 | Missing `[Id(n)]` on serialised members | Add sequential IDs, never reuse |
| 3 | `Grain<T>` inheritance for state | `IPersistentState<T>` via constructor injection |
| 4 | Mutable message types (classes with setters) | `[Immutable]` sealed records |
| 5 | Missing `[Alias]` on grain interface | Add stable dot-separated alias |
| 6 | `.Result` or `.Wait()` in grain code | `await` all the way |
| 7 | Auto-persist assumption (no `WriteStateAsync`) | Explicit `WriteStateAsync()` calls |
| 8 | `List<T>` in grain state | `ImmutableList<T>` or `ImmutableArray<T>` |
