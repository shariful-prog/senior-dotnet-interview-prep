# Architecture Top 20 Interview Questions (Quick Answers)

> Goal: Fast revision. Each question opens with a one-line definition, then the detail — plus a link to the exact section in the detailed architecture notes.
> These are senior/lead questions — interviewers listen for **trade-offs**, so every answer below names a cost, not just a benefit.

## Design Principles

### 1. What does SOLID stand for?

**Definition:** Five design principles: **S**ingle responsibility, **O**pen/closed, **L**iskov substitution, **I**nterface segregation, **D**ependency inversion.

**Detail:** One reason to change; extend without modifying; a subtype must work anywhere its base works; don't force a class to depend on methods it doesn't use; depend on abstractions rather than concrete types. They all serve one goal — isolating change so a new requirement touches one place instead of ten.

:material-file-document-outline: **Deep dive:** [AA0 — Framing](../Architecture/arch-solid.md#aa0--framing)

---

### 2. What is the Liskov Substitution Principle?

**Definition:** **LSP** says that if `S` is a subtype of `T`, you must be able to use an `S` anywhere a `T` is expected without the program breaking.

**Detail:** In practice it means a subclass can't strengthen what it demands or weaken what it promises. The tell in a code review is an override that throws `NotSupportedException`, silently does nothing, or rejects input the base class accepted — the classic `Square extends Rectangle` problem. The fix is usually to split the interface so the capability is explicit.

:material-file-document-outline: **Deep dive:** [AA3 — Liskov Substitution](../Architecture/arch-solid.md#aa3--liskov-substitution-principle-lsp)

---

### 3. What is the difference between DIP, DI, and IoC?

**Definition:** **IoC** is the broad idea that the framework calls your code. **DIP** is a principle about which way dependencies point. **DI** is one technique for achieving DIP.

**Detail:** They get used interchangeably, but they're different levels. The subtle part of DIP is that the **consumer owns the interface** — `IOrderRepository` belongs with the domain code that uses it, not with the EF implementation. You can use a DI container and still violate DIP, if the interface lives beside its implementation.

:material-file-document-outline: **Deep dive:** [AA5 — Dependency Inversion](../Architecture/arch-solid.md#aa5--dependency-inversion-principle-dip)

---

### 4. When does applying SOLID become over-engineering?

**Definition:** **Over-engineering** here means adding an abstraction for a variation that never actually arrives.

**Detail:** One implementation behind an interface, a factory that produces a single type, five files where one would do. Every abstraction has a real cost in indirection and reading time. Apply the principles when a second reason to change actually shows up — the rule of three. Premature abstraction is as much a smell as no structure at all.

:material-file-document-outline: **Deep dive:** [AA6 — Judgment & Trade-offs](../Architecture/arch-solid.md#aa6--judgment--trade-offs)

---

### 5. Should you use the Repository pattern on top of EF Core?

**Definition:** A **repository** abstracts data access behind a collection-like interface; **unit of work** groups changes into one transaction.

**Detail:** Usually no, because EF already gives you both — `DbContext` *is* a unit of work and `DbSet<T>` *is* a repository. Wrapping them adds a layer that either leaks `IQueryable` (so it abstracts nothing) or blocks `Include` and projections (so it costs you performance). Justify it only for genuine swappability or to keep the domain persistence-free. "For testing" is a weak reason — use an integration test.

:material-file-document-outline: **Deep dive:** [AB3 — Repository & Unit of Work](../Architecture/arch-design-patterns.md#ab3--repository--unit-of-work)

---

### 6. Why is Singleton called an anti-pattern?

**Definition:** The **Singleton pattern** is a class that creates and exposes its own single instance through a static property.

**Detail:** The problem isn't having one instance — it's that the class hides a global, mutable dependency that callers can't substitute in tests, and it's easy to get thread-safety wrong. In .NET you want **singleton lifetime**, not the Singleton pattern: register the type with `AddSingleton`, inject it as an interface, and make it thread-safe.

:material-file-document-outline: **Deep dive:** [AB6 — Other Patterns Worth Knowing](../Architecture/arch-design-patterns.md#ab6--other-patterns-worth-knowing)

## Application Architecture

### 7. What is Clean/Onion architecture?

**Definition:** An architecture arranged as concentric layers where **all dependencies point inward**. The domain sits at the centre and knows nothing about databases, HTTP, or frameworks.

**Detail:** The domain defines interfaces like `IOrderRepository`; the infrastructure layer on the outside implements them. That inversion is what lets you test business rules without a database. The cost is more projects, more mapping between layers, and more ceremony for small applications.

:material-file-document-outline: **Deep dive:** [AC2 — Onion & Clean Architecture](../Architecture/arch-clean-onion.md#ac2--onion--clean-architecture)

---

### 8. What is the difference between an anemic and a rich domain model?

**Definition:** An **anemic model** has classes that hold data only, with all behaviour in separate service classes. A **rich model** keeps the rules with the data they protect.

**Detail:** Anemic is workable but tends to scatter the same validation across several services, so an object can exist in an invalid state. A rich model enforces its invariants in its own methods and constructor — `order.AddLine(...)` can refuse. Prefer rich where the domain has real rules; anemic is fine for genuine CRUD.

:material-file-document-outline: **Deep dive:** [AC4 — Domain Model Style](../Architecture/arch-clean-onion.md#ac4--domain-model-style)

---

### 9. What is Vertical Slice Architecture?

**Definition:** **Vertical Slice** organizes code by **feature** rather than by technical layer — each slice owns its request, handler, and data access end to end.

**Detail:** Instead of touching five layers to add a field, you change one folder. The trade-off is some duplication between slices, and less enforced separation. It suits CRUD-heavy applications and teams who find Clean Architecture's ceremony isn't paying for itself.

:material-file-document-outline: **Deep dive:** [AC5 — Alternatives & Pragmatism](../Architecture/arch-clean-onion.md#ac5--alternatives--pragmatism)

## Monolith vs Microservices

### 10. What is the difference between a monolith, a modular monolith, and microservices?

**Definition:** A **monolith** deploys as one unit with boundaries by convention. A **modular monolith** deploys as one unit with **enforced** boundaries. **Microservices** deploy as many units, each owning its own database.

**Detail:** A modular monolith differs from a plain monolith by discipline, and from microservices by deployment. It's the boundaries of microservices with the deployment of a monolith — which is why it's the right default for most systems.

:material-file-document-outline: **Deep dive:** [Q4 — The Three Models](../Architecture/arch-monolith-microservices.md#q4-what-is-the-difference-between-the-three-side-by-side)

---

### 11. What does "independently deployable" mean?

**Definition:** You can release your service on its own, without coordinating with other teams and without breaking anything already running.

**Detail:** The bar is higher than people expect. You need your own database, no shared library holding your internals, contracts that only ever add rather than remove, no coordinated release window, and callers that degrade rather than crash when you're down. Miss any of these and you have the costs of microservices without the benefit.

:material-file-document-outline: **Deep dive:** [Q5 — Independently Deployable](../Architecture/arch-monolith-microservices.md#q5-what-does-independently-deployable-mean)

---

### 12. What is a distributed monolith?

**Definition:** A system split into separate services that are still coupled enough to require **deploying them together**.

**Detail:** The classic failure mode — you pay network latency, partial failure, and extra infrastructure, and get none of the autonomy. Signs: a shared database, release trains, a shared domain library everyone versions together, long synchronous call chains, and services you can't test in isolation. The honest fix is often to merge them back.

:material-file-document-outline: **Deep dive:** [Q6 — Distributed Monolith](../Architecture/arch-monolith-microservices.md#q6-what-is-a-distributed-monolith)

---

### 13. What is a bounded context?

**Definition:** A **bounded context** is an area of the business where one model and one vocabulary stay consistent.

**Detail:** "Customer" in Sales (a lead, a deal stage) and in Support (tickets, an SLA tier) are genuinely different things sharing a name. Bounded contexts are the right seam for services, because they cut where the language changes — which is where the reasons to change differ. This rules out one-service-per-table: `UserService`, `AddressService` aren't business areas, they're tables with HTTP on top.

:material-file-document-outline: **Deep dive:** [Q23 — Bounded Context](../Architecture/arch-monolith-microservices.md#q23-what-is-a-bounded-context) · [Q24 — Bad Boundary Signs](../Architecture/arch-monolith-microservices.md#q24-what-are-the-signs-of-a-bad-boundary)

---

### 14. Three developers want microservices from day one. What do you advise?

**Definition:** Push back — with three developers there's **nobody to be independent from**, which is the only thing microservices buy you.

**Detail:** What's slowing them down is feature throughput, not deployment. They'd be signing up for contract negotiations with themselves, several services running locally, tracing they haven't built, and a saga for every multi-step write. Recommend a modular monolith organized by business area, with an architecture test that fails the build on boundary violations — the seams stay, the cost doesn't.

:material-file-document-outline: **Deep dive:** [Q11 — Three Developers](../Architecture/arch-monolith-microservices.md#q11-three-developers-want-microservices-from-day-one-what-do-you-advise) · [Q9 — When to Stay Monolith](../Architecture/arch-monolith-microservices.md#q9-when-should-you-choose-a-monolith-or-modular-monolith)

## Distributed Data

### 15. What is a saga?

**Definition:** A **saga** carries out a business operation spanning several services as a **sequence of local transactions**, each with a matching undo step.

**Detail:** You need it because there's no shared transaction across services. **Choreography** has services react to each other's events — decoupled, but the flow isn't written down anywhere. **Orchestration** has one coordinator drive the steps — explicit and monitorable, at the cost of an extra component. Crucially, undoing is a *new action*: you don't un-charge a card, you refund it. So put irreversible steps last.

:material-file-document-outline: **Deep dive:** [Q17 — Sagas](../Architecture/arch-monolith-microservices.md#q17-what-is-a-saga)

---

### 16. What is the dual-write problem, and what is the Outbox pattern?

**Definition:** The **dual-write problem** is writing to two systems — your database and a message broker — with no shared transaction. The **Transactional Outbox** solves it by writing the message into a table in the same transaction as the data.

**Detail:** Without it, a crash between the two leaves them silently disagreeing, and retrying doesn't help because you can't tell which half succeeded. With the Outbox it's one database, so it's genuinely all-or-nothing; a background relay then publishes. The cost: slightly later delivery, and duplicates when the relay crashes after sending.

:material-file-document-outline: **Deep dive:** [Q18 — Dual Write](../Architecture/arch-monolith-microservices.md#q18-what-is-the-dual-write-problem) · [Q19 — Transactional Outbox](../Architecture/arch-monolith-microservices.md#q19-what-is-the-transactional-outbox-pattern)

---

### 17. What is idempotency?

**Definition:** An operation is **idempotent** if doing it twice has the same effect as doing it once.

**Detail:** Not optional in a distributed system: brokers guarantee **at-least-once** delivery, so duplicates will arrive. The **Inbox pattern** records each processed message ID in the same transaction as the work, and skips anything already seen. A unique constraint or a conditional update often does the same job more simply. Worth saying: exactly-once delivery doesn't exist — it's at-least-once plus consumers that tolerate repeats.

:material-file-document-outline: **Deep dive:** [Q20 — Idempotency](../Architecture/arch-monolith-microservices.md#q20-what-is-idempotency)

---

### 18. What is eventual consistency?

**Definition:** **Eventual consistency** means that after a change, different parts of the system are briefly out of step and catch up shortly after.

**Detail:** It's the unavoidable result of each service owning its data. Explain it to stakeholders in their own terms — "the warehouse screen picks up the new address a second later" — then ask if that's acceptable. Usually it is. It is *not* for moving money within one account, selling the last item in stock, or permission checks. Keep those inside one service with one transaction.

:material-file-document-outline: **Deep dive:** [Q21 — Eventual Consistency](../Architecture/arch-monolith-microservices.md#q21-what-is-eventual-consistency)

## Judgment

### 19. Do microservices make a system more reliable?

**Definition:** **No — not by themselves.** Splitting a system up adds more things that can fail, and every one of them is needed for the request to succeed.

**Detail:** Ten services at 99.9% each, all required for one request, gives roughly 99% overall — about ten times more downtime than the monolith. You only win reliability back by deliberately adding timeouts, retries, circuit breakers, fallbacks, and async messaging. That's real work, and it's the strongest argument against splitting before you have the platform for it.

:material-file-document-outline: **Deep dive:** [Q8 — Availability Maths](../Architecture/arch-monolith-microservices.md#q8-do-microservices-make-a-system-more-reliable) · [Q7 — What Distribution Costs](../Architecture/arch-monolith-microservices.md#q7-what-do-microservices-actually-cost-you)

---

### 20. What is the Strangler Fig pattern?

**Definition:** **Strangler Fig** replaces a system gradually — a facade in front, capabilities moved across one at a time, old paths deleted once nothing uses them.

**Detail:** Use it to argue against a big-bang rewrite, which stops feature delivery for months, forces you to maintain two systems, always underestimates the undocumented edge cases that *are* the business rules, and offers nothing until the very end. There's also a deeper problem: a tangled system has no clear boundaries yet, so a rewrite means guessing them at the moment you know least.

:material-file-document-outline: **Deep dive:** [Q25 — Strangler Fig](../Architecture/arch-monolith-microservices.md#q25-what-is-the-strangler-fig-pattern) · [Q26 — What to Extract First](../Architecture/arch-monolith-microservices.md#q26-which-part-should-you-extract-first)

---

## Runners-Up (ask-me-next round)

**Principles & patterns**
- **Factory, Strategy, Decorator, Mediator/MediatR** — [AB1](../Architecture/arch-design-patterns.md#ab1--factory-patterns-creational) · [AB2](../Architecture/arch-design-patterns.md#ab2--strategy-behavioral) · [AB5](../Architecture/arch-design-patterns.md#ab5--decorator-structural) · [AB4](../Architecture/arch-design-patterns.md#ab4--mediator-behavioral)
- **Adapter vs Facade vs Proxy; Observer; Builder** — [AB6](../Architecture/arch-design-patterns.md#ab6--other-patterns-worth-knowing)
- **When NOT to use a pattern** — [AB7](../Architecture/arch-design-patterns.md#ab7--judgment)
- **SRP, cohesion & coupling; OCP in C#; ISP** — [AA1](../Architecture/arch-solid.md#aa1--single-responsibility-principle-srp) · [AA2](../Architecture/arch-solid.md#aa2--openclosed-principle-ocp) · [AA4](../Architecture/arch-solid.md#aa4--interface-segregation-principle-isp)
- **Hexagonal / Ports & Adapters; the composition root** — [AC3](../Architecture/arch-clean-onion.md#ac3--hexagonal--ports--adapters)
- **Layered / N-tier and its weakness** — [AC1](../Architecture/arch-clean-onion.md#ac1--layered--n-tier)

**Monolith, modules & boundaries**
- **Conway's Law & the Inverse Conway Maneuver** — [Q12](../Architecture/arch-monolith-microservices.md#q12-what-is-conways-law)
- **Module public API; enforcing boundaries with architecture tests** — [Q13](../Architecture/arch-monolith-microservices.md#q13-what-is-a-module-and-what-is-its-public-api) · [Q14](../Architecture/arch-monolith-microservices.md#q14-how-do-you-enforce-module-boundaries)
- **Module communication: calls vs in-process events** — [Q15](../Architecture/arch-monolith-microservices.md#q15-how-should-modules-talk-to-each-other)
- **Shared entities (`User`, `Product`) across modules** — [Q16](../Architecture/arch-monolith-microservices.md#q16-how-do-you-handle-data-that-several-modules-need-like-user)
- **When microservices are genuinely right** — [Q10](../Architecture/arch-monolith-microservices.md#q10-when-are-microservices-genuinely-the-right-choice)
- **Is a modular monolith a valid end state? Preventing decay** — [Q27](../Architecture/arch-monolith-microservices.md#q27-is-a-modular-monolith-just-a-stepping-stone) · [Q28](../Architecture/arch-monolith-microservices.md#q28-how-do-you-stop-a-modular-monolith-decaying)

**Distributed data**
- **Querying across services when you can't `JOIN`** — [Q22](../Architecture/arch-monolith-microservices.md#q22-how-do-you-query-across-services-when-you-cant-join)

## Judgment Questions (senior/lead rounds)

These have no textbook answer — they test whether you've actually paid the costs. Worth rehearsing a real story for each.

- **When does applying SOLID become over-engineering?** — [AA6](../Architecture/arch-solid.md#aa6--judgment--trade-offs)
- **Isn't Clean Architecture over-engineering for a small app?** — [AC5](../Architecture/arch-clean-onion.md#ac5--alternatives--pragmatism)
- **When is eventual consistency unacceptable?** — [Q21](../Architecture/arch-monolith-microservices.md#q21-what-is-eventual-consistency)
- **What is the Day-2 cost teams underestimate?** — [Q7](../Architecture/arch-monolith-microservices.md#q7-what-do-microservices-actually-cost-you)
- **How do you push back on a premature rewrite?** — [Q25](../Architecture/arch-monolith-microservices.md#q25-what-is-the-strangler-fig-pattern)
- **What is the worst architectural decision you have made?** — *no notes; prepare a real story: what you chose, what it cost, what you'd do now.*

## Scenario Practice

The hardest rounds are scenario-driven — a symptom, and you diagnose. High-value ones:

- **Three devs want microservices on day one** — [Q11](../Architecture/arch-monolith-microservices.md#q11-three-developers-want-microservices-from-day-one-what-do-you-advise)
- **Adding a field breaks 4 services** → wrong boundaries — [Q24](../Architecture/arch-monolith-microservices.md#q24-what-are-the-signs-of-a-bad-boundary)
- **Service crashes after DB write, before publishing** → Outbox — [Q19](../Architecture/arch-monolith-microservices.md#q19-what-is-the-transactional-outbox-pattern)
- **Broker delivers the same event twice** → idempotent consumer — [Q20](../Architecture/arch-monolith-microservices.md#q20-what-is-idempotency)
- **Payment succeeds, inventory reservation fails** → saga compensation — [Q17](../Architecture/arch-monolith-microservices.md#q17-what-is-a-saga)
- **Leadership demands a full rewrite** → Strangler Fig — [Q25](../Architecture/arch-monolith-microservices.md#q25-what-is-the-strangler-fig-pattern)
