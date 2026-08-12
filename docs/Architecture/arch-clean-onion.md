# AC. Application Architecture — Clean / Onion / Hexagonal
---

## AC0 — Framing

### Q1. Why do these architectures exist? What problem are they all solving?

**Answer.** Layered, Onion, Clean, and Hexagonal are variations on **one core idea**: keep the **business logic independent of infrastructure and delivery mechanisms** (DB, web framework, message broker, UI). They differ mostly in vocabulary and diagram shape, not in principle.

The problem they solve: in a naïve app, business rules get tangled with EF entities, controllers, and SQL. That makes the rules hard to test (need a DB and a web host), hard to change (swap SQL Server for Postgres, or REST for gRPC, and the domain breaks), and hard to reason about (the "what the app does" is buried in "how it talks to the world").

The shared cure is the **Dependency Inversion Principle** applied at the *architecture* level: point all source-code dependencies **inward, toward the domain**, and let the domain define abstractions (interfaces/ports) that the outer layers implement. See [arch-solid.md](arch-solid.md) Q8.

---

## AC1 — Layered / N-tier

### Q2. What is classic layered (N-tier) architecture and what's its weakness?

**Answer.** The traditional stack: **Presentation → Business/Service → Data Access → Database**, each layer depending on the one below it.

```
Presentation (Controllers / UI)
 ↓ depends on
Business Logic (Services)
 ↓ depends on
Data Access (Repositories / EF)
 ↓ depends on
Database
```

**Weakness:** the **business layer depends on the data-access layer**, which means it (transitively) depends on infrastructure. Your domain rules end up referencing EF entities and repository types directly. Testing business logic drags in the persistence layer; swapping the data store ripples upward. The dependency arrow points the *wrong way* — toward the volatile detail (the DB) instead of away from it.

Onion/Clean fix exactly this by **inverting** the data-access dependency.

---

## AC2 — Onion & Clean Architecture

### Q3. Explain Onion / Clean Architecture and the dependency rule.

**Answer.** Picture concentric circles. The **domain sits at the center**; each ring outward is more concerned with infrastructure and delivery. The one unbreakable law is **The Dependency Rule: source-code dependencies point only inward.** Nothing in an inner circle knows anything about an outer circle.

```
 ┌─────────────────────────────────────────┐
 │ Presentation / Infrastructure (outer) │ ← Controllers, EF, HTTP, files, brokers
 │ ┌───────────────────────────────────┐ │
 │ │ Application (use cases) │ │ ← orchestrates domain; defines interfaces
 │ │ ┌───────────────────────────┐ │ │
 │ │ │ Domain (core) │ │ │ ← entities, value objects, domain rules
 │ │ │ entities, VOs, rules │ │ │ NO dependencies on anything
 │ │ └───────────────────────────┘ │ │
 │ └───────────────────────────────────┘ │
 └─────────────────────────────────────────┘
 dependencies point INWARD ──►
```

- **Domain (core):** entities, value objects, domain services, and the business rules/invariants. **Zero dependencies** — no EF, no ASP.NET, no NuGet infrastructure packages. Pure C#.
- **Application:** use cases / orchestration (e.g. `PlaceOrderHandler`). Depends only on the domain. **Defines the interfaces it needs** (`IOrderRepository`, `IEmailSender`, `IPaymentGateway`) — but not their implementations.
- **Infrastructure:** implements those interfaces with real tech — EF Core repositories, SMTP/SendGrid email, Stripe payment client. Depends *inward* on Application/Domain.
- **Presentation:** controllers, minimal APIs, gRPC — the delivery mechanism. Also outer.

The magic move: `PlaceOrderHandler` (Application) needs to save an order, so it depends on `IOrderRepository` **which lives in Application/Domain**. `EfOrderRepository` (Infrastructure) *implements* it. At runtime DI injects the concrete, but at *compile time* Infrastructure depends on Application, never the reverse. The domain never sees EF.

---

### Q4. What does the dependency inversion look like concretely across projects?

**Answer.** A typical .NET solution:

```
MyApp.Domain // Order, OrderLine, Money (VO), IOrderRepository (interface)
MyApp.Application // PlaceOrderHandler, IEmailSender (interface), DTOs
MyApp.Infrastructure // EfOrderRepository : IOrderRepository, SmtpEmailSender : IEmailSender
MyApp.Api // Controllers / Program.cs — composition root, wires DI
```

Project references (the compile-time arrows):

```
Api ──► Application ──► Domain
Infrastructure ──► Application ──► Domain
Api ──► Infrastructure (only to register services in the composition root)
```

Note **Application and Domain reference nothing outward.** `Infrastructure` points *in* at `Application`. The interface `IOrderRepository` is declared in Domain/Application; the implementation `EfOrderRepository` lives in Infrastructure. This is DIP at the assembly level — the dependency arrow between "policy" and "detail" is inverted.

```csharp
// Domain (or Application) — owns the abstraction
public interface IOrderRepository
{
 Task<Order?> GetByIdAsync(OrderId id, CancellationToken ct);
 Task AddAsync(Order order, CancellationToken ct);
}

// Application — the use case, depends only on the abstraction
public class PlaceOrderHandler
{
 private readonly IOrderRepository _orders;
 private readonly IPaymentGateway _payments;
 public PlaceOrderHandler(IOrderRepository orders, IPaymentGateway payments)
 => (_orders, _payments) = (orders, payments);

 public async Task<OrderId> Handle(PlaceOrderCommand cmd, CancellationToken ct)
 {
 var order = Order.Create(cmd.CustomerId, cmd.Lines); // domain rules enforced here
 await _payments.ChargeAsync(order.Total, cmd.Card, ct);
 await _orders.AddAsync(order, ct);
 return order.Id;
 }
}

// Infrastructure — the detail, points inward by implementing the interface
public class EfOrderRepository : IOrderRepository
{
 private readonly AppDbContext _db;
 public EfOrderRepository(AppDbContext db) => _db = db;
 public Task AddAsync(Order order, CancellationToken ct) { _db.Orders.Add(order); return _db.SaveChangesAsync(ct); }
 public Task<Order?> GetByIdAsync(OrderId id, CancellationToken ct) => _db.Orders.FindAsync(...).AsTask();
}
```

---

### Q5. What is the "composition root"?

**Answer.** The **single place where the object graph is assembled** — where abstractions are bound to concretes. In ASP.NET Core it's `Program.cs` (the DI registrations). It's the *only* part of the app that's allowed to know all the concrete types.

```csharp
// Program.cs — the composition root (in the outer/Api layer)
builder.Services.AddScoped<IOrderRepository, EfOrderRepository>();
builder.Services.AddScoped<IPaymentGateway, StripePaymentGateway>();
builder.Services.AddScoped<IEmailSender, SendGridEmailSender>();
builder.Services.AddScoped<PlaceOrderHandler>();
```

Keeping wiring in one root is what lets every other layer depend only on abstractions. Container/lifetime detail → [ASP.NET Core notes](../Dotnet/dotnet-aspnetcore-fundamentals.md).

---

## AC3 — Hexagonal / Ports & Adapters

### Q6. What is Hexagonal Architecture, and how does it relate to Clean/Onion?

**Answer.** Hexagonal (Alistair Cockburn's **Ports & Adapters**) is the same inward-dependency idea framed around the boundary between the application and the outside world:

- **Ports** — interfaces the application defines. Two kinds:
 - **Driving/primary ports** — how the outside world *drives* the app (e.g. `IPlaceOrderUseCase`). Called by controllers, CLI, tests, message consumers.
 - **Driven/secondary ports** — what the app *needs* from the outside (e.g. `IOrderRepository`, `IPaymentGateway`).
- **Adapters** — concrete implementations that plug into ports:
 - **Driving adapters** — REST controller, gRPC service, message handler, CLI. They *call* driving ports.
 - **Driven adapters** — EF repository, Stripe client, SMTP sender. They *implement* driven ports.

```
 Driving adapters Driven adapters
 (REST / gRPC / CLI / tests) (EF / Stripe / SMTP / Kafka)
 │ call ▲ implement
 ▼ │
 ┌──────────────── Application core ───────────────┐
 │ driving ports → use cases → driven ports │
 └─────────────────────────────────────────────────┘
```

The "hexagon" shape just signals *many* sides/adapters — no significance to six. The key insight vs a plain layer diagram: the same core can be driven by a REST API *and* a message consumer *and* a test harness, all through the same driving port, and can be backed by EF *or* Dapper *or* an in-memory fake through the same driven port.

**Relationship:** Onion, Clean, and Hexagonal are **the same principle** (dependencies point at the domain; the domain owns abstractions). Clean explicitly names the Application/Use-Case ring; Hexagonal emphasizes the symmetric driving/driven port boundary; Onion emphasizes the concentric layering. In interviews, say "they're variations on the same dependency-inversion idea" — that reads as senior, versus treating them as three unrelated things.

---

## AC4 — Domain model style

### Q7. Anemic vs rich domain model — what's the difference and which is preferred?

**Answer.**
- **Anemic domain model** — entities are bags of public getters/setters with **no behavior**; all logic lives in "service" classes that manipulate the data from outside. Martin Fowler calls it an anti-pattern because it's procedural code wearing an OO costume: the data and the rules that govern it are separated, so invariants can be violated from anywhere.
- **Rich domain model** — entities **own their behavior and protect their invariants**. State changes go through methods that enforce rules; setters are private.

```csharp
// Anemic — anyone can put the order in an invalid state
public class Order
{
 public OrderStatus Status { get; set; }
 public List<OrderLine> Lines { get; set; } = new();
 public decimal Total { get; set; }
}
// ...and a service somewhere:
order.Status = OrderStatus.Shipped; // shipped an order with zero lines? nothing stops it

// Rich — the entity enforces its own rules
public class Order
{
 private readonly List<OrderLine> _lines = new();
 public OrderStatus Status { get; private set; } = OrderStatus.Draft;
 public IReadOnlyList<OrderLine> Lines => _lines;
 public Money Total => _lines.Aggregate(Money.Zero, (sum, l) => sum + l.Subtotal);

 public void AddLine(ProductId product, int qty)
 {
 if (Status != OrderStatus.Draft) throw new InvalidOperationException("Can't modify a submitted order.");
 _lines.Add(new OrderLine(product, qty));
 }

 public void Ship()
 {
 if (_lines.Count == 0) throw new InvalidOperationException("Can't ship an empty order.");
 Status = OrderStatus.Shipped;
 }
}
```

**Preferred:** the rich model, especially with DDD (see [AD3 — Bounded Contexts](arch-monolith-microservices.md#q23-what-is-a-bounded-context)). **Nuance for seniors:** anemic isn't *always* wrong — for a thin CRUD app with no real invariants, a rich model is over-engineering. Match the modeling effort to the domain complexity.

---

## AC5 — Alternatives & pragmatism

### Q8. What is Vertical Slice Architecture, and when might you prefer it over Clean?

**Answer.** Clean/Onion organizes code by **technical layer** (all repositories together, all handlers together). **Vertical Slice Architecture** organizes by **feature**: everything for "Place Order" — the command, handler, validator, DTOs, even the data access — lives in one folder/slice, often with MediatR.

```
Features/
 PlaceOrder/
 PlaceOrderCommand.cs
 PlaceOrderHandler.cs
 PlaceOrderValidator.cs
 PlaceOrderEndpoint.cs
 CancelOrder/
 ...
```

**Why:** most changes are feature-shaped ("add a discount to placing an order"), and a slice keeps everything that changes together in one place — high cohesion, less jumping across layers. Slices can also vary their internal approach (a trivial query can hit EF directly; a complex command can use a rich domain model) instead of forcing every feature through the same layers.

**Trade-off:** less enforced separation, potential duplication across slices, and it can drift without discipline. Many real .NET apps combine them: vertical slices for the application layer, a Clean-style domain core, shared infrastructure. Senior framing: *"Layers optimize for swapping technical concerns; slices optimize for shipping features. Most changes are features, so I lean toward slices, keeping a clean domain core for anything with real invariants."*

---

### Q9. Isn't Clean Architecture over-engineering for a small app?

**Answer.** Often, yes — and saying so is a senior signal. Full Clean Architecture (four projects, interfaces for everything, DTOs at every boundary) has real cost: more projects, more mapping, more indirection. For a small CRUD service or a short-lived internal tool, that ceremony buys little.

Pragmatic guidance:
- **Small/simple app:** a couple of layers (or vertical slices) and EF directly is fine. Don't invert what you'll never swap.
- **Large, long-lived, complex domain with real invariants:** Clean/Onion pays off — the domain stays testable and insulated as the app and team grow.
- **The value is realized over time and team size**, not at first commit. Apply the structure where change and complexity are real, not everywhere by default (same judgment as SOLID — [arch-solid.md](arch-solid.md) Q10).

---

## AC — Cheat Sheet

**One idea, three names:** Onion / Clean / Hexagonal all enforce **dependencies point inward, toward the domain**; the domain owns the abstractions (DIP at architecture scale). Differences are vocabulary/diagram, not principle.

**The Dependency Rule:** source-code dependencies point only inward. Inner layers know nothing about outer layers.

**Rings (inner → outer):**
| Layer | Contains | Depends on |
|-------|----------|-----------|
| **Domain** (core) | Entities, value objects, invariants, domain services | **Nothing** (pure C#) |
| **Application** | Use cases/handlers; **defines** `IOrderRepository`, `IEmailSender` | Domain |
| **Infrastructure** | EF repos, Stripe/SMTP clients — **implement** the interfaces | Application/Domain (inward) |
| **Presentation** | Controllers, minimal APIs, gRPC | Application (inward) |

**.NET project refs:** `Api → Application → Domain`, `Infrastructure → Application → Domain`. Interface in Domain/Application, implementation in Infrastructure. `Api → Infrastructure` only in the **composition root** (`Program.cs`) to register services.

**Hexagonal ports/adapters:** *driving* (primary) ports = how the world drives the app (called by controllers/tests); *driven* (secondary) ports = what the app needs (implemented by EF/Stripe). Same core, many adapters.

**Layered (N-tier) weakness:** business layer depends on data-access → domain drags in infrastructure. Onion inverts that arrow.

**Anemic vs rich:** anemic = getter/setter bags + external services (invariants unprotected); rich = entities own behavior + protect invariants (private setters, methods enforce rules). Prefer rich for real domains; anemic OK for thin CRUD.

**Vertical slice:** organize by feature not layer; high cohesion per feature, great with MediatR. Combine with a clean domain core.

**Over-engineering check:** full Clean = 4 projects + mapping + indirection; worth it for large/complex/long-lived domains, overkill for small CRUD. Structure earns its cost over time and team size.
