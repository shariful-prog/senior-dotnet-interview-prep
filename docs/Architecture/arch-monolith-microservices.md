# AD. Monolith, Modular Monolith & Microservices
---

## AD1 — The Three Models

### Q1. What is a monolith?

**Answer.** A **monolith** is an application that is built and deployed as **one single unit**. All the code lives in one solution, compiles into one artifact, and runs as one process against one database.

```text
┌─────────────────────────────────┐
│         Shop.Web.dll            │   ← one deployable
│  Catalog · Orders · Payments    │   ← all code inside
└──────────────┬──────────────────┘
               │
        ┌──────▼──────┐
        │  one database│
        └─────────────┘
```

Inside, the code is usually organised into layers (Controllers → Services → Repositories). Those layers are a **convention**, not a wall — any class can call any other class, and any query can join any table.

Important: "monolith" describes **how you deploy it**, not how good it is. A tidy, well-organised monolith is a perfectly respectable architecture.

---

### Q2. What is a modular monolith?

**Answer.** A **modular monolith** is still **one deployable unit**, but the code inside is divided into **modules with enforced boundaries**. Each module owns its own tables and exposes a small public API. Other modules must go through that API — the compiler stops them reaching inside.

```text
┌─────────────────────────────────────────┐
│              Shop.Web.dll               │   ← still ONE deployable
│  ┌─────────┐ ┌─────────┐ ┌───────────┐  │
│  │ Catalog │ │ Orders  │ │ Payments  │  │   ← walls between them
│  └─────────┘ └─────────┘ └───────────┘  │
└──────────────────┬──────────────────────┘
                   │
      ┌────────────▼────────────┐
      │ one DB, schema per module│
      └─────────────────────────┘
```

So it sits between the other two: it differs from a plain monolith by **discipline**, and from microservices by **how it's deployed**. You get clean separation without any network calls.

---

### Q3. What are microservices?

**Answer.** **Microservices** are an architecture where the system is split into **many separately deployable services**. Each service owns its own database, runs as its own process, and can be released without touching the others. They talk over the network — HTTP, gRPC, or messages.

```text
┌──────────┐   ┌──────────┐   ┌──────────┐
│ Catalog  │   │  Orders  │   │ Payments │   ← three deployables
└────┬─────┘   └────┬─────┘   └────┬─────┘
     │              │              │
  ┌──▼──┐        ┌──▼──┐        ┌──▼──┐
  │ DB  │        │ DB  │        │ DB  │       ← one database each
  └─────┘        └─────┘        └─────┘
```

The point of doing this is **independent deployment** — letting many teams ship without coordinating. That's the benefit you're buying, and everything else is the price.

---

### Q4. What is the difference between the three, side by side?

**Answer.**

| | Deployment | Boundaries | Data | Communication |
|---|---|---|---|---|
| **Monolith** | One unit | Convention only | One shared schema | Method calls |
| **Modular Monolith** | One unit | **Enforced** by compiler | One DB, schema per module | Method calls / in-process events |
| **Microservices** | Many units | Enforced by the network | **One DB per service** | HTTP / gRPC / messages |

The useful way to remember it: **a modular monolith has the boundaries of microservices with the deployment of a monolith.**

---

### Q5. What does "independently deployable" mean?

**Answer.** **Independently deployable** means you can release your service on its own — no coordinating with other teams, and without breaking anything already running.

That sounds simple but the bar is high. You need all of these:

- **Your own database.** If another service reads your tables, your schema is public and you can't change it alone.
- **No shared library holding your internals.** If everyone depends on a "Common" project with domain models in it, changing it makes everyone rebuild.
- **Contracts that don't break.** You must be able to deploy while old versions of callers are still running. So add fields, don't remove or rename them.
- **No release meeting.** If shipping needs a coordinated window, it isn't independent.
- **Callers cope when you're down.** They degrade rather than crash.

This matters because independent deployment is the *entire* reason to accept the cost of microservices. If you can't tick these boxes, you've paid the price without getting the benefit — which has a name.

---

### Q6. What is a distributed monolith?

**Answer.** A **distributed monolith** is a system that has been split into separate services, but those services are still so tangled together that they must be **deployed at the same time**. It's the classic failure mode: you get the costs of microservices and none of the benefits.

How to spot one:

- **They share a database**, or one service reads another's tables.
- **You deploy them together** — a "release train", or "we can't ship A until B ships".
- **A shared domain library** everyone must upgrade at once.
- **Long chains of synchronous calls** — one request hops through five services, so if any is down, the request fails.
- **Changing one service breaks another service's build.**
- **You can't run or test one service on its own.**

What to say in an interview: *"You're paying the full cost of distributed systems and getting nothing back. Either fix the coupling — give each one its own data and versioned contracts — or merge them back together."* Suggesting the merge is a good sign, not an admission of failure.

---

## AD2 — Choosing Between Them

### Q7. What do microservices actually cost you?

**Answer.** Three things change the moment you put a network between two pieces of code:

1. **A method call becomes a network call.** It can now be slow, time out, or fail halfway. Every call site needs a timeout and a retry policy.
2. **A compile error becomes a runtime error.** Rename a field in a monolith and the build fails. Rename it in a service contract and you find out in production.
3. **A transaction becomes a saga.** You can no longer wrap two operations in one database transaction (Q17).

On top of that there's the ongoing running cost, which is what teams underestimate:

- You now **need** central logging, distributed tracing, and per-service metrics. Without them "checkout feels slow" is unanswerable.
- One build pipeline per service.
- Local development needs several services running.
- On-call gets harder — more parts, and failures that only appear when something is half-broken.

---

### Q8. Do microservices make a system more reliable?

**Answer.** **Not by themselves — usually the opposite at first.** This surprises people, so it's worth being able to show the arithmetic.

If one request needs 10 services and each is up 99.9% of the time, they all have to work for the request to succeed:

```text
0.999 ^ 10  =  0.990   →  about 99%
```

That's roughly **10x more downtime** than a single monolith at 99.9%. Splitting things up doesn't add reliability, it adds more things that can fail.

You get reliability back only by deliberately adding **timeouts**, **retries**, **circuit breakers**, **fallbacks**, and **async messaging** so one service being down doesn't fail the whole request. That's engineering work, and it isn't free — which is why this is a strong argument against splitting before you're ready.

---

### Q9. When should you choose a monolith or modular monolith?

**Answer.** Choose one when:

- **The team is small.** Under roughly 15-20 engineers, deployment isn't your bottleneck — writing features is. Microservices tax every feature.
- **You don't understand the domain yet.** Early boundaries are usually wrong, and a wrong boundary is cheap to move inside a monolith and expensive to move between services.
- **Everything scales together.** If nothing needs to scale differently, per-service scaling buys you nothing.
- **You need real transactions.** Money, bookings, stock — one database transaction is worth a lot.
- **You don't have the ops setup yet.** No CI/CD, central logging, or tracing? Get those first. They're a prerequisite, not a side effect.

Moving **back** is also legitimate. If you have 50 services, painful deploys, and a big cloud bill with only a few teams, merging related services into a modular monolith is good engineering.

---

### Q10. When are microservices genuinely the right choice?

**Answer.** When you have a **specific pressure** that a single deployable can't relieve:

- **Many teams blocked on each other.** The classic driver. Ten teams sharing one pipeline spend their time coordinating.
- **One part scales very differently.** Image processing needs 40 machines; the admin screens need one.
- **Different availability needs.** Checkout must never go down; the reporting screen can.
- **A hard compliance boundary** — card data or medical records isolated from everything else.
- **A genuinely different technology need** for one component.

Notice what's *not* on this list: "it's modern", "it'll scale better later", or "big companies do it". If you can't name the pressure, you don't have a reason yet.

---

### Q11. Three developers want microservices from day one. What do you advise?

**Answer.** Talk them out of it — with specifics, not "microservices are bad".

With three developers there's **nobody to be independent from**. What's slowing them down is how fast they can build features. Here's what they'd sign up for:

- Every feature touching two services becomes a contract negotiation with themselves.
- Several services must run just to work locally.
- Debugging needs tracing they haven't set up.
- Anything that was a `JOIN` becomes an API call.
- Every multi-step write becomes a saga.

Suggest instead: **a modular monolith with enforced boundaries**, organised by business area (Catalog, Ordering, Payments) rather than technical layer. Add an architecture test that fails the build when someone crosses a boundary (Q14).

How to put it: *"We're not saying monolith forever. We're avoiding an expensive, hard-to-reverse decision before we know enough to make it well."*

---

### Q12. What is Conway's Law?

**Answer.** **Conway's Law** says that a system's design ends up mirroring the communication structure of the organisation that built it. Four teams building a compiler will produce a four-pass compiler.

The consequence: if your architecture and your org chart disagree, **the org chart wins**. Split into services owned by teams that must talk constantly, and you get services that must talk constantly.

**The Inverse Conway Maneuver** is using this deliberately — arranging teams to produce the architecture you want. Give each team a business area end to end (one team owns Ordering) rather than a technical layer (a "database team", a "frontend team"). Layer-shaped teams produce layer-coupled software where every feature needs three teams.

The practical rule: **one team owns a service.** Two teams sharing one means coordinating on every change. One team owning several related services is fine.

---

## AD3 — Modular Monolith in Practice

### Q13. What is a module, and what is its public API?

**Answer.** A **module** is a business area with a wall around it. Its **public API** is the small set of interfaces and DTOs other modules are allowed to use. Everything else is `internal` and invisible outside.

```csharp
// Ordering module — the ONLY thing other modules can see
namespace Shop.Ordering.Contracts;

public interface IOrderService
{
    Task<OrderSummary> PlaceOrderAsync(PlaceOrder command, CancellationToken ct);
}

public sealed record PlaceOrder(Guid CustomerId, IReadOnlyList<OrderLine> Lines);
public sealed record OrderSummary(Guid OrderId, decimal Total, string Status);
```

```csharp
// Everything else is internal — invisible outside the assembly
namespace Shop.Ordering.Domain;

internal sealed class Order        // the real entity never leaves the module
{
    private readonly List<OrderLine> _lines = new();
    internal void AddLine(Guid productId, int qty) { /* rules enforced here */ }
}
```

The test for a real boundary: **could you pull this module out into its own service without changing any calling code?** If yes, it's a module. If callers reach into its internals or join to its tables, it's just a folder.

---

### Q14. How do you enforce module boundaries?

**Answer.** Trusting people to remember doesn't survive a deadline. Make the build stop them. Strongest first:

- **Put each module in its own project and mark internals `internal`.** Now the compiler enforces it. Other projects reference only `.Contracts`. This is the highest-value step.
- **Add architecture tests that fail the build**, which catch what `internal` can't (like accidental cycles).
- **Give each module its own database schema**, with a DB user that can't read the others — so a cross-module query fails in development.

```csharp
// Architecture test (NetArchTest) — fails the build on a boundary violation
[Fact]
public void Modules_must_not_reference_each_others_internals()
{
    var result = Types.InAssembly(typeof(Shop.Ordering.Domain.Order).Assembly)
        .That().ResideInNamespace("Shop.Ordering")
        .ShouldNot().HaveDependencyOn("Shop.Inventory.Domain")   // internals, not contracts
        .GetResult();

    Assert.True(result.IsSuccessful, string.Join(", ", result.FailingTypeNames ?? []));
}
```

**.NET note.** `InternalsVisibleTo` is how people cheat. Grant it to the module's *test* project only, never to another module.

---

### Q15. How should modules talk to each other?

**Answer.** Two options, and a simple rule for choosing.

**Call an interface directly** when you're telling a module to do something and need to know it worked. Synchronous, in the same transaction, easy to debug.

**Publish an in-process event** when you're announcing that something happened and don't care who listens. The publisher doesn't know its subscribers.

```csharp
// Command — I need this done, and I need the result
var summary = await _orderService.PlaceOrderAsync(command, ct);

// Event — this happened; whoever cares can react
await _events.PublishAsync(new OrderPlaced(order.Id), ct);
```

The rule: **commands go through interfaces, announcements go through events.**

One thing to watch: **don't let two modules reference each other both ways.** If Ordering calls Inventory and Inventory calls Ordering, you've built one module with extra steps. Replace one direction with an event.

---

### Q16. How do you handle data that several modules need, like `User`?

**Answer.** Each module gets its own **schema** in one database and owns its tables. Nobody reads anyone else's directly.

The tricky part is something like `User`, which everyone seems to need. The mistake is one big shared `User` table everyone joins to.

The better way to see it: **there isn't really one `User`.** There's an Ordering `Customer` (shipping address, past orders), a Billing `Payer` (cards, tax ID), and an Identity `User` (login, roles). Same human being, three different models, each owning what it actually needs.

So in practice:

- **One module owns identity** and issues the ID.
- **Other modules store just that ID**, plus any fields they need for their own decisions.
- **Need more? Ask through the API** — or keep a local copy updated by events if you read it constantly.

In a modular monolith you can start simpler (a foreign key, reads through the interface) and denormalise only when it hurts. But keep the *models* separate even while storage is shared — that's what makes splitting cheap later.

---

## AD4 — Distributed Data

### Q17. What is a saga?

**Answer.** A **saga** is a way to carry out a business operation that spans several services when you can't use a single database transaction. It's a **sequence of local transactions**, where each step has a matching **undo step** (called a *compensating action*) in case a later step fails.

```text
Place Order  →  Take Payment  →  Reserve Stock  →  Ship
                                       ✗ fails
                 ← Refund      ←  Cancel Order          ← undo steps run backwards
```

Two ways to run one:

- **Choreography** — each service publishes events and others react. Nicely decoupled, but the overall flow isn't written down anywhere; you have to read every handler to understand it. Fine for two or three steps.
- **Orchestration** — one coordinator explicitly runs the steps. The flow lives in one readable place and is easy to monitor, at the cost of an extra component.

The key point: **undoing is a new action, not a rollback.** You don't un-charge a card, you issue a refund — a new fact in the system. And some things can't be undone at all: an email has been sent, a parcel has left the warehouse. So order your saga to do reversible things first and irreversible things last.

---

### Q18. What is the dual-write problem?

**Answer.** The **dual-write problem** happens when one operation must write to **two different systems** — typically your database and a message broker — with no shared transaction between them. If the process crashes in between, the two disagree and nothing tells you.

```csharp
// BROKEN — two systems, no shared transaction
await _db.SaveChangesAsync(ct);                    // ✅ committed
await _bus.PublishAsync(new OrderPlaced(id), ct);  // ❌ crash here = event lost forever
```

Now the order exists but nobody was told. Retrying doesn't help, because you can't tell which half succeeded.

---

### Q19. What is the Transactional Outbox pattern?

**Answer.** The **Transactional Outbox** solves the dual-write problem by writing the message into an **`Outbox` table in the same transaction** as the data change. It's one database, so it's genuinely all-or-nothing. A background job then reads that table and publishes, marking rows as sent.

```csharp
// CORRECT — one transaction covers both
_db.Orders.Add(order);
_db.Outbox.Add(new OutboxMessage
{
    Type    = nameof(OrderPlaced),
    Payload = JsonSerializer.Serialize(new OrderPlaced(order.Id)),
    OccurredAt = timeProvider.GetUtcNow()
});
await _db.SaveChangesAsync(ct);   // both or neither
```

What it costs: messages arrive slightly later, and you can get **duplicates** — the publisher might send, then crash before marking the row as sent, and send again on restart. Which is exactly why consumers must handle repeats.

---

### Q20. What is idempotency?

**Answer.** An operation is **idempotent** if doing it twice has the same effect as doing it once. Charging a card is not idempotent by default — do it twice and the customer pays twice.

This isn't optional in a distributed system. Message brokers guarantee **at-least-once** delivery, so duplicates *will* arrive: from publisher retries, from a consumer crashing after doing the work but before acknowledging, and from broker redelivery.

The **Inbox pattern** handles it — record the message ID in the same transaction as the work, and skip anything you've already seen:

```csharp
public async Task HandleAsync(ChargePayment msg, CancellationToken ct)
{
    // Same transaction as the work — so "seen it" and "did it" are atomic
    if (await _db.InboxMessages.AnyAsync(m => m.MessageId == msg.MessageId, ct))
        return;                                    // already handled — skip quietly

    _db.InboxMessages.Add(new InboxMessage { MessageId = msg.MessageId });
    _db.Payments.Add(new Payment(msg.OrderId, msg.Amount));
    await _db.SaveChangesAsync(ct);                // both or neither
}
```

Simpler options often work too: a **unique constraint** on `OrderId` does the same job without an inbox table, and a **conditional update** (`WHERE status = 'Pending'`) naturally ignores repeats.

Worth adding in an interview: **"exactly once" delivery doesn't exist.** What you build is at-least-once delivery plus consumers that don't mind repeats, which amounts to the same thing.

---

### Q21. What is eventual consistency?

**Answer.** **Eventual consistency** means that after a change, the different parts of the system are briefly out of step, and catch up a moment later. It's the unavoidable result of each service owning its own data.

Explaining it to a non-technical stakeholder — skip CAP theorem, use their world: *"When you update your address, it saves straight away. The warehouse screen picks it up a second or two later. In that gap they might still see the old one."* Then ask the real question: **"Is that okay here?"**

Usually it is. Sometimes it really isn't:

- **Moving money within one account** — a balance shouldn't go negative because two withdrawals both read a stale number.
- **The last item in stock** — overselling costs real money.
- **Permission checks** — revoked access needs to be revoked now.

For those, keep the whole operation **inside one service with one transaction**. The senior move is knowing which parts of the business must be exactly right, and drawing boundaries so those stay in one piece.

---

### Q22. How do you query across services when you can't `JOIN`?

**Answer.** Once each service owns its data you lose `JOIN`s and foreign keys. Four options, cheapest first:

- **Ask each service and combine in code.** Fine for a handful of rows. Falls apart when you need to filter, sort, or page across services, because you'd have to fetch everything first.
- **Keep a local copy** of what you need, updated by events. Fast local reads, at the price of being slightly behind.
- **Build a read model** — a separate store fed by events from several services, shaped for exactly the screen that needs it. This is the answer for "orders filtered by customer city, sorted by name, 50 per page" spanning two services.
- **Use a reporting database** for analytics. Don't build reports on your live system.

For **integrity**, the database can't enforce it any more. So the owning service validates on write, everyone else stores just the ID, and instead of hard-deleting rows others point at, you soft-delete or publish a "this was deleted" event.

---

## AD5 — Boundaries & Migration

### Q23. What is a bounded context?

**Answer.** A **bounded context** is an area of the business where one model and one vocabulary stay consistent. The classic example: "Customer" in Sales (a lead, a deal stage) and "Customer" in Support (tickets, an SLA tier) are genuinely different things that happen to share a name.

Bounded contexts are the **right place to split services**, because they're where the language changes — and where the language changes is where the *reasons to change* differ.

This rules out the common mistake: **one service per database table.** `UserService`, `AddressService`, `OrderLineService` aren't business areas, they're tables with HTTP bolted on. You end up with chatty call chains and sagas for what should have been one local operation.

To find them, listen for the same word meaning different things in different teams. Event Storming — putting the business process on a wall as a series of events — surfaces them quickly.

---

### Q24. What are the signs of a bad boundary?

**Answer.** The clearest sign: **services you always deploy together.** If adding a field to `Order` means Payment, Inventory, and Notification all need redeploying, those four are really one thing cut into four pieces.

Others:

- **Lots of back-and-forth calls** for one operation — you split something that belonged together.
- **Sagas everywhere.** If most operations need compensation, you've cut through something that should have been one transaction.
- **A shared database** — the boundary isn't real.
- **A service that just passes things through**, adding a network hop and no behaviour.
- **The same two or three services in every pull request.**

The fix is often to **merge them back**, which people resist because it feels like going backwards. It isn't.

---

### Q25. What is the Strangler Fig pattern?

**Answer.** The **Strangler Fig** is a way to replace a system gradually instead of rewriting it all at once. You put a facade in front of the old system, move one capability at a time to the new implementation, and delete the old path once nothing uses it.

```text
        ┌─────────┐
        │ Facade  │
        └────┬────┘
    ┌────────┴────────┐
    ▼                 ▼
┌────────┐      ┌──────────┐
│  Old   │      │   New    │   ← capabilities move across one at a time
│ system │      │ service  │
└────────┘      └──────────┘
```

Why not a big-bang rewrite: feature work stops for months while the business still needs changes, you maintain two systems at once, and the old system's undocumented behaviour (the weird edge cases that turn out to *be* the business rules) is always underestimated. Worst of all, there's nothing to show until the very end and no way back.

There's a further problem: a tangled system has no clear internal boundaries, so you don't yet know what the services should be. Rewriting means guessing at the moment you know least.

Worth adding: *"and a modular monolith might be the right destination — most of the pain here is coupling, and we can fix coupling without going distributed."*

---

### Q26. Which part should you extract first?

**Answer.** Something **low-risk with an obvious payoff** — not the most important thing. Look for:

- **Few things depending on it**, especially few things calling into it.
- **A boundary that's already clean.**
- **An actual reason** — it scales differently, releases on a different rhythm, or a team is ready to own it.
- **Not on the critical path**, so a mistake degrades the site instead of taking it down.

Good first candidates: notifications and email, reporting and search, image or file processing, PDF generation. They're asynchronous, tolerant of delay, often scale differently, and hold no critical business rules.

Bad first candidate: the core thing (`Order` in e-commerce). Most coupled, most business rules, worst possible place to learn distributed systems.

---

### Q27. Is a modular monolith just a stepping stone?

**Answer.** **No — it's a perfectly good place to stop.** Most systems never need microservices, and a modular monolith with real boundaries gives you most of what people actually want: clear ownership, components you can test in isolation, and a codebase you can navigate — without distributed transactions, stale data, or needing a platform team.

Treating it as "just temporary" causes a specific problem: people stop maintaining the boundaries ("we'll sort that out when we split"), the boundaries rot, and splitting becomes just as hard as it would have been from a tangled mess.

Better framing: *"Build it like this is the final answer. If we ever do need to split, it'll be easy — but that's a bonus, not the plan."*

---

### Q28. How do you stop a modular monolith decaying?

**Answer.** Architecture doesn't collapse at once — it erodes through small exceptions that each seemed fine at the time. What actually helps:

- **Automated enforcement** (Q14). Anything CI doesn't check will drift. By far the most valuable one.
- **Make the correct thing the easy thing** — a template for a new module, a documented way to call across modules.
- **Make exceptions visible** — a short written decision record to add a cross-module dependency. Not a ban, just a speed bump so it's a decision instead of a reflex.
- **Don't let a "Common" project exist.** Shared projects collect everything and quietly re-couple all your modules. Copy a small class rather than sharing a domain concept.
- **Watch for warning signs** — modules that always change together, a growing shared project, cross-schema queries appearing.

---

## AD — Cheat Sheet

**The three models**

| | Deploy | Boundaries | Data | Best when |
|---|---|---|---|---|
| Monolith | One unit | Convention | Shared schema | Small team, domain unclear |
| Modular Monolith | One unit | **Enforced** (`internal`, arch tests) | Schema per module | Most systems, most of the time |
| Microservices | Many | Network + contract | **One DB each** | Many teams, different scaling, ops in place |

**Independent deployment needs:** own database · no shared internals library · non-breaking contracts · no release meeting · callers survive you being down. Miss any → **distributed monolith**.

**Distributed monolith signs:** shared DB · deploy together · shared domain library · long sync chains · can't test alone.

**What distribution costs:** method call → network call · compile error → runtime error · transaction → saga. Plus tracing, pipelines, harder local dev and on-call.

**Availability maths:** 10 services × 99.9%, all needed → ≈ 99%. Splitting *lowers* reliability until you add timeouts, circuit breakers, fallbacks, async.

**Boundaries:** split by bounded context, never one-service-per-table. One team owns a service. Bad signs: deploy together, chatty calls, sagas everywhere, same services in every PR. Fix by **merging** — not a failure.

**Distributed data**
- No shared transaction → **saga**. Choreography = decoupled but hard to follow · orchestration = clear but extra component. Undo is a new action; irreversible steps last.
- Writing to DB + broker → **Outbox** (one transaction). Costs latency and duplicates.
- Duplicates will happen → **idempotent consumer** (Inbox / unique key / conditional update).
- **"Exactly once" isn't real** — at-least-once delivery + consumers that tolerate repeats.
- Queries across services → combine in code (small) · local copy (hot reads) · read model (filter/sort/page) · reporting DB (analytics).

**Migration:** Strangler Fig, never big bang. Extract low-risk first (notifications, reporting, images), never the core aggregate.

**The line to land:** *"Start with a modular monolith with real boundaries; pull a service out when there's a specific reason that justifies the cost."*

**Related:** SOLID at service level → [arch-solid.md](arch-solid.md) · patterns → [arch-design-patterns.md](arch-design-patterns.md) · Clean/Onion → [arch-clean-onion.md](arch-clean-onion.md)
