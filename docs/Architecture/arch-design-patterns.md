# AB. Design Patterns
---

## AB0 — Framing

### Q1. How should you talk about design patterns in a senior interview?

**Answer.** Patterns are **named solutions to recurring design problems** (from the GoF book), and a shared vocabulary. The junior mistake is treating them as goals ("I used 5 patterns"). The senior signal is:

1. Name the **problem/force** first, then the pattern that resolves it.
2. Show the **idiomatic .NET form** — many GoF patterns are baked into the framework (DI = a factory/service-locator, `IEnumerator` = Iterator, LINQ = Strategy-ish, `IObservable<T>` = Observer, middleware = Chain of Responsibility).
3. Know **when NOT to use it** — every pattern adds indirection. Patterns are discovered by refactoring toward a need, not applied up front.

The three GoF categories: **Creational** (how objects are made — Factory, Builder, Singleton), **Structural** (how objects compose — Adapter, Decorator, Facade), **Behavioral** (how objects interact — Strategy, Observer, Mediator, Chain of Responsibility).

---

## AB1 — Factory patterns (Creational)

### Q2. Distinguish Simple Factory, Factory Method, and Abstract Factory.

**Answer.**

- **Simple Factory** (not strictly GoF) — a single method/class that centralizes object creation, usually switching on a parameter. Hides `new` and the concrete type from the caller.
- **Factory Method** — defines an interface for creating an object but lets *subclasses* decide which class to instantiate. Creation is a `virtual`/`abstract` method overridden per subtype.
- **Abstract Factory** — an interface for creating **families of related objects** without specifying concrete classes (e.g. a `IUiThemeFactory` producing matching `Button` + `Checkbox` for a theme).

```csharp
// Simple Factory — centralize creation, return an abstraction
public static class PaymentProcessorFactory
{
 public static IPaymentProcessor Create(PaymentType type) => type switch
 {
 PaymentType.CreditCard => new CreditCardProcessor(),
 PaymentType.PayPal => new PayPalProcessor(),
 _ => throw new ArgumentOutOfRangeException(nameof(type))
 };
}
```

```csharp
// Factory Method — subclass decides the concrete product
public abstract class Dialog
{
 protected abstract IButton CreateButton(); // the factory method
 public void Render() { var b = CreateButton(); b.Draw(); }
}
public class WindowsDialog : Dialog { protected override IButton CreateButton() => new WindowsButton(); }
public class WebDialog : Dialog { protected override IButton CreateButton() => new HtmlButton(); }
```

```csharp
// Abstract Factory — a family of related products
public interface IUiFactory { IButton CreateButton(); ICheckbox CreateCheckbox(); }
public class MaterialFactory : IUiFactory { /* material button + checkbox */ }
public class FluentFactory : IUiFactory { /* fluent button + checkbox */ }
```

**Why use them:** decouple callers from concrete types (DIP), centralize construction logic, enable swapping families.

**.NET note.** The DI container is effectively a configurable factory. When you need *runtime* selection you inject `IEnumerable<T>` (all implementations) or a `Func<T>` / typed factory rather than hand-rolling a factory class. Prefer the container until it can't express what you need.

---

## AB2 — Strategy (Behavioral)

### Q3. What is the Strategy pattern and when do you reach for it?

**Answer.** Strategy defines a **family of interchangeable algorithms**, encapsulates each behind a common interface, and lets the algorithm vary independently of the client that uses it. It's the OOP way to pass *behavior* as a parameter.

```csharp
public interface IDiscountStrategy { decimal Apply(decimal price); }

public class NoDiscount : IDiscountStrategy { public decimal Apply(decimal p) => p; }
public class PercentDiscount : IDiscountStrategy
{
 private readonly decimal _pct;
 public PercentDiscount(decimal pct) => _pct = pct;
 public decimal Apply(decimal p) => p * (1 - _pct);
}

public class Checkout
{
 private readonly IDiscountStrategy _discount;
 public Checkout(IDiscountStrategy discount) => _discount = discount;
 public decimal Total(decimal subtotal) => _discount.Apply(subtotal);
}
```

**Reach for it when:** you have a `switch`/`if-else` selecting between behaviors, or several variants of one algorithm (pricing, sorting, compression, routing). It's the concrete realization of **OCP** — see [arch-solid.md](arch-solid.md) Q4.

**.NET idiom.** A `Func<...>` delegate is often a lighter-weight Strategy when the algorithm is a single method with no state — `Checkout(Func<decimal,decimal> discount)`. Reach for the interface form when the strategy has state, multiple methods, or needs DI.

---

### Q4. Strategy vs State — what's the difference?

**Answer.** Structurally near-identical (both delegate to a swappable object behind an interface); the *intent* differs:
- **Strategy** — the client picks the algorithm; strategies are independent and unaware of each other.
- **State** — the object's behavior changes as its internal state changes, and states often trigger *transitions* to other states. The object swaps its own strategy over its lifetime.

---

## AB3 — Repository & Unit of Work

### Q5. What are the Repository and Unit of Work patterns?

**Answer.**
- **Repository** — mediates between the domain and data-mapping layer, exposing a collection-like interface for aggregate access (`Add`, `Remove`, `GetById`, query methods) while hiding the persistence mechanism.
- **Unit of Work** — tracks a set of changes across repositories and commits them as one atomic transaction (`SaveChanges` / `Commit`).

```csharp
public interface IOrderRepository
{
 Task<Order?> GetByIdAsync(int id);
 Task AddAsync(Order order);
 void Remove(Order order);
}

public interface IUnitOfWork
{
 IOrderRepository Orders { get; }
 Task<int> SaveChangesAsync(); // commits all tracked changes atomically
}
```

**Purpose:** testability (mock the repo), a persistence-ignorant domain (DIP), and a single seam for querying rules.

---

### Q6. Should you add Repository/Unit of Work on top of EF Core? (the debate)

**Answer.** This is a deliberate senior trap — there's a strong "no" camp and a strong "it depends" camp. Show you understand both.

**The case against:** `DbContext` **is already** a Unit of Work, and `DbSet<T>` **is already** a repository (both patterns are literally named in EF's own design docs). Wrapping them adds a layer that:
- leaks anyway (you end up exposing `IQueryable` or a pile of bespoke query methods),
- hides EF features you actually need (`Include`, projections, `AsNoTracking`, split queries),
- is often justified by "swap the ORM someday," which almost never happens.

**The case for a repository abstraction:**
- **Testability** without a real DB (though EF Core's InMemory/SQLite-in-memory providers weaken this argument),
- a place to **centralize query logic** and enforce aggregate boundaries (DDD),
- keeping EF types out of the domain/application layer (Clean Architecture purity),
- swapping EF for Dapper on hot paths behind the same interface.

**The senior answer:** "I don't wrap EF by default — `DbContext` already gives me UoW + repository. I introduce a **thin, aggregate-focused repository** only when I need to keep infrastructure types out of my domain (Clean/Onion), enforce aggregate boundaries, or centralize complex queries — and I return domain types/DTOs, never `IQueryable`, from it. A generic `IRepository<T>` with `GetAll/Find/Add` over every entity is usually an anti-pattern — it's a leaky abstraction that just renames `DbSet`." Full EF-side detail → [EF Core notes](../Dotnet/dotnet-ef-core.md).

---

## AB4 — Mediator (Behavioral)

### Q7. What problem does the Mediator pattern solve, and how does MediatR use it?

**Answer.** Mediator **reduces coupling between components by having them communicate through a central mediator** instead of referring to each other directly. Instead of N×N direct references, each component talks only to the mediator.

**MediatR** (the popular .NET library) applies this to in-process messaging: controllers/handlers send a **request** object; MediatR routes it to the single **handler** for that request type. The sender doesn't know the handler.

```csharp
// Request (a command or query)
public record CreateOrderCommand(int CustomerId, List<int> ProductIds) : IRequest<int>;

// Handler — the only thing that knows how to fulfill it
public class CreateOrderHandler : IRequestHandler<CreateOrderCommand, int>
{
 private readonly IOrderRepository _repo;
 public CreateOrderHandler(IOrderRepository repo) => _repo = repo;

 public async Task<int> Handle(CreateOrderCommand cmd, CancellationToken ct)
 {
 var order = new Order(cmd.CustomerId, cmd.ProductIds);
 await _repo.AddAsync(order);
 return order.Id;
 }
}

// Controller: thin — just sends the request
[HttpPost]
public async Task<IActionResult> Create(CreateOrderCommand cmd)
 => Ok(await _mediator.Send(cmd));
```

**Why it's popular:** it gives thin controllers, one handler per use case (SRP), and a **pipeline** for cross-cutting concerns via `IPipelineBehavior` (validation, logging, transactions, caching) — the Decorator/Chain idea applied to requests. It's the usual vehicle for **CQRS** — see [AD4 — Distributed Data](arch-monolith-microservices.md#ad4--distributed-data).

**Gotcha / pushback.** Critics note MediatR can become an indirection tax: jumping "send → find handler" hurts navigability, and for simple CRUD it's ceremony over calling a service method. Senior take: "great for a command/query app with real cross-cutting pipeline needs; overkill for a small CRUD API."

---

## AB5 — Decorator (Structural)

### Q8. What is the Decorator pattern? Give a .NET example.

**Answer.** Decorator **attaches additional behavior to an object dynamically by wrapping it in an object of the same interface.** It's composition-based extension — an OCP-friendly alternative to subclassing for adding responsibilities (logging, caching, retry, validation).

```csharp
public interface IOrderService { Task<Order> GetAsync(int id); }

public class OrderService : IOrderService // the real component
{
 public Task<Order> GetAsync(int id) { /* hit the DB */ }
}

public class CachingOrderService : IOrderService // decorator: same interface, wraps inner
{
 private readonly IOrderService _inner;
 private readonly IMemoryCache _cache;
 public CachingOrderService(IOrderService inner, IMemoryCache cache)
 { _inner = inner; _cache = cache; }

 public Task<Order> GetAsync(int id) =>
 _cache.GetOrCreateAsync($"order:{id}", _ => _inner.GetAsync(id))!; // add caching, delegate rest
}
```

You can stack them: `LoggingOrderService → CachingOrderService → OrderService`, each adding one concern. **.NET everywhere:** `Stream` (`BufferedStream`/`GZipStream` wrap another `Stream`), and DI libraries (Scrutor's `Decorate<T>`) register decorators. This is also how MediatR pipeline behaviors work conceptually.

**vs Proxy/Adapter:** Decorator *adds behavior*, same interface; **Adapter** *changes the interface* to match what a client expects; **Proxy** controls *access* (lazy load, remoting, security) with the same interface.

---

## AB6 — Other patterns worth knowing

### Q9. Adapter and Facade — what's the difference?

**Answer.**
- **Adapter** — converts one interface into another a client expects; wraps an incompatible/legacy/3rd-party type so it fits your abstraction. One-to-one, interface *translation*.
- **Facade** — provides a *simplified* interface over a complex subsystem (many classes). One-to-many, complexity *hiding*.

```csharp
// Adapter: 3rd-party logger → our ILogger abstraction
public class SerilogAdapter : IAppLogger
{
 private readonly Serilog.ILogger _serilog;
 public SerilogAdapter(Serilog.ILogger s) => _serilog = s;
 public void Log(string msg) => _serilog.Information(msg); // translate the call
}
```

---

### Q10. Observer — and its .NET forms.

**Answer.** Observer defines a **one-to-many dependency** so that when one object (subject) changes state, all its dependents (observers) are notified automatically — the basis of event-driven/pub-sub within a process.

.NET gives you several idiomatic forms so you rarely hand-roll it:
- **`event` / `EventHandler`** — the classic language-level Observer.
- **`IObservable<T>` / `IObserver<T>`** — the reactive interfaces (Rx).
- **In-process messaging** — MediatR *notifications* (`INotification` → many `INotificationHandler`s) for one-to-many domain events.

For *cross-process* pub/sub you move to a broker — see [AD4 — Distributed Data](arch-monolith-microservices.md#ad4--distributed-data).

---

### Q11. Why is Singleton often called an anti-pattern? What's the .NET-correct way?

**Answer.** The *pattern* (one instance, global access) is fine; the classic **implementation** (a static `Instance` property) is the problem:
- **Hidden global state** — callers depend on it invisibly, hurting testability (can't substitute it) and creating spooky coupling.
- **Thread-safety bugs** if lazily initialized without care.
- It's really two responsibilities: doing its job *and* managing its own lifetime (violates SRP).

**.NET-correct approach:** don't hand-roll a static Singleton. Register the type with **singleton lifetime in the DI container** (`services.AddSingleton<IClock, SystemClock>()`). You get one instance, but it's injected via an abstraction — testable, no global static, lifetime managed by the container. If you *must* hand-roll, use `Lazy<T>` or a static readonly field (the latter is thread-safe by the CLR's type-initialization guarantees).

```csharp
// If you truly need a hand-rolled singleton, this is thread-safe and lazy:
public sealed class Config
{
 private static readonly Lazy<Config> _instance = new(() => new Config());
 public static Config Instance => _instance.Value;
 private Config() { }
}
```

Singleton **lifetime** pitfalls (capturing a scoped/transient dependency = captive dependency) → [ASP.NET Core notes](../Dotnet/dotnet-aspnetcore-fundamentals.md).

---

### Q12. Builder and the Options pattern — when?

**Answer.** **Builder** constructs a complex object step by step, separating construction from representation — useful when a constructor would have many (especially optional) parameters, or construction has ordering/validation steps.

.NET is full of builders: `StringBuilder`, `WebApplicationBuilder`, `HostBuilder`, EF's `ModelBuilder`, the fluent `DbContextOptionsBuilder`. The **Options pattern** (`IOptions<T>`) is the idiomatic way to bind and inject strongly-typed configuration — related in spirit (assemble a config object), detail in [ASP.NET Core notes](../Dotnet/dotnet-aspnetcore-fundamentals.md).

```csharp
var app = WebApplication.CreateBuilder(args) // Builder pattern in the framework
 .Build();
```

---

## AB7 — Judgment

### Q13. When should you NOT use a design pattern?

**Answer.** Patterns solve problems you actually have; applied speculatively they're just complexity:
- **No variation exists yet** — a Strategy/Factory with one implementation is premature. Wait for the second case (rule of three).
- **The framework already provides it** — don't hand-roll a Singleton, Iterator, or Observer that the BCL/DI gives you.
- **Simple CRUD** — Mediator + Repository + generic abstractions over a small app is ceremony that slows everyone down.
- **The pattern hides something you need** — a generic repository that hides EF's `Include`/projections costs more than it saves.

The senior framing: *"Patterns are discovered by refactoring toward a real need, not chosen up front from a catalog. The goal is the simplest design that absorbs the change we can actually see coming."* Over-patterning is as much a code smell as no structure at all.

---

## AB — Cheat Sheet

**Categories:** Creational (make objects) · Structural (compose objects) · Behavioral (objects interact).

| Pattern | Type | Solves | .NET form / example |
|---------|------|--------|---------------------|
| **Simple Factory** | Creational | Centralize `new`, return abstraction | `switch` returning an interface; DI container |
| **Factory Method** | Creational | Subclass picks the concrete | `virtual Create...()` overridden per subtype |
| **Abstract Factory** | Creational | Family of related objects | `IUiFactory` → matching button+checkbox |
| **Builder** | Creational | Step-by-step complex construction | `StringBuilder`, `WebApplicationBuilder` |
| **Singleton** | Creational | One instance | **`AddSingleton` in DI**, not static `Instance` |
| **Adapter** | Structural | Translate an incompatible interface | wrap 3rd-party logger as `IAppLogger` |
| **Facade** | Structural | Simplify a complex subsystem | one entry class over many |
| **Decorator** | Structural | Add behavior by wrapping (same interface) | `Stream` wrappers, caching/logging decorators |
| **Proxy** | Structural | Control access (lazy/remote/security) | EF lazy-loading proxies |
| **Strategy** | Behavioral | Interchangeable algorithms | `IDiscountStrategy`, or a `Func<>` |
| **Observer** | Behavioral | One-to-many notification | `event`, `IObservable<T>`, MediatR notifications |
| **Mediator** | Behavioral | Decouple via a central hub | **MediatR** send/handler + pipeline |
| **Chain of Responsibility** | Behavioral | Pass request along handlers | ASP.NET Core **middleware** |
| **Iterator** | Behavioral | Sequential access | `IEnumerator<T>` / `yield return` |

**Repository/UoW:** `DbContext` = UoW, `DbSet<T>` = repository already. Add a *thin, aggregate* repo only for domain purity / query centralization; never expose `IQueryable`; avoid generic `IRepository<T>` over everything. Debate detail → [EF Core notes](../Dotnet/dotnet-ef-core.md).

**Strategy vs State:** same structure; Strategy = client picks algorithm, State = object swaps its own behavior + transitions.
**Adapter vs Decorator vs Proxy:** Adapter changes interface · Decorator adds behavior (same interface) · Proxy controls access (same interface).

**When NOT to:** no variation yet · framework already provides it · simple CRUD · the pattern hides features you need. Patterns are *refactored toward*, not chosen up front.
