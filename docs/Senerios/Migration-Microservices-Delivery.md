# RT2. Migration, Microservices & Delivery Scenarios

These are open-ended design questions. The interviewer is checking how you think, not whether you know one right answer. For every one of them, the good answer follows the same shape: **understand the current state → pick a safe order of steps → say how you verify it → say what you do if it goes wrong.**

---

### RT2-1 — Service-to-service communication and fallbacks

### Q1. Service A calls Service B and Service B is down or slow. What do you do?

**Answer.** First, decide whether the call is really needed right now.

**Step 1 — Ask if the call must be synchronous.**

| Call type | Use when | What happens if the other service is down |
|-----------|----------|-------------------------------------------|
| **Synchronous** (HTTP/gRPC) | The caller needs the answer to continue | The caller fails too |
| **Asynchronous** (message queue) | The work can happen slightly later | The message waits in the queue |

Most "fallback" problems are really a sign that the call should have been asynchronous. If Order Service must email a customer, it should publish an `OrderPlaced` message, not call Email Service and wait.

**Step 2 — Add timeouts.** A call with no timeout is the worst case. The caller's threads pile up waiting, and the caller goes down too.

```csharp
builder.Services.AddHttpClient<IPricingClient, PricingClient>(c =>
{
    c.BaseAddress = new Uri("https://pricing-service/");
    c.Timeout = TimeSpan.FromSeconds(3);        // always set one
});
```

**Step 3 — Add retries, but only for safe calls.**

- **Retry** only on transient failures — timeouts, 503, connection reset.
- **Never retry** on 400 or 404. The answer will not change.
- Use **exponential backoff with jitter** so all callers do not retry at the same instant.
- Retries need **idempotency** — the same call twice must not create two orders. Send a unique request key and have the receiver ignore duplicates.

**Step 4 — Add a circuit breaker.** Retrying a service that is fully down makes things worse. A circuit breaker stops calling for a while and fails fast.

**Definition — Circuit breaker.** After a set number of failures, the breaker "opens" and all calls fail instantly without hitting the network. After a wait, it lets one test call through. If that succeeds, it closes again.

```text
Closed  → calls go through normally
Open    → calls fail immediately (service gets time to recover)
Half-open → one test call allowed; success closes, failure re-opens
```

**Step 5 — Decide the fallback.** Only after the above. Options, best first:

| Fallback | Example |
|----------|---------|
| **Cached value** | Show yesterday's price list |
| **Default value** | Show "shipping estimate unavailable" |
| **Degrade the feature** | Hide the recommendations panel, keep checkout working |
| **Queue it for later** | Accept the order, send the email when Email Service returns |
| **Fail the request** | Payment failed — never fake a success |

The last row matters. Never invent a fallback for something that must be correct. A missing recommendation is fine; a fake payment confirmation is not.

**Step 6 — In .NET, use Polly.** Do not hand-write retry loops. Polly is the standard library for this, and since .NET 8 it ships as `Microsoft.Extensions.Http.Resilience`.

**What Polly gives you.** Each of these is called a **strategy**, and you combine them into a **pipeline**:

| Strategy | What it does | Typical setting |
|----------|--------------|-----------------|
| **Timeout** | Cancels a call that takes too long | 2–5 seconds |
| **Retry** | Tries again on transient failure | 3 attempts, exponential backoff + jitter |
| **Circuit breaker** | Stops calling after repeated failures | Break after 50% failures in 30s |
| **Fallback** | Returns a substitute value instead of throwing | Cached or default value |
| **Hedging** | Sends a second call in parallel if the first is slow | For read-only calls only |
| **Rate limiter** | Limits how many calls you make outward | Protects the downstream service |

**The simplest correct setup** — one line, sensible defaults:

```csharp
builder.Services.AddHttpClient<IPricingClient, PricingClient>(c =>
{
    c.BaseAddress = new Uri("https://pricing-service/");
})
.AddStandardResilienceHandler();     // timeout + retry + circuit breaker, in the right order
```

`AddStandardResilienceHandler()` is the recommended starting point. It applies rate limiter → total timeout → retry → circuit breaker → per-attempt timeout, already ordered correctly.

**Customising it** when the defaults do not fit:

```csharp
builder.Services.AddHttpClient<IPricingClient, PricingClient>()
    .AddStandardResilienceHandler(options =>
    {
        options.AttemptTimeout.Timeout = TimeSpan.FromSeconds(3);   // per attempt
        options.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(15); // all attempts

        options.Retry.MaxRetryAttempts = 3;
        options.Retry.BackoffType = DelayBackoffType.Exponential;
        options.Retry.UseJitter = true;                             // avoid retry storms

        options.CircuitBreaker.FailureRatio = 0.5;                  // 50% failures
        options.CircuitBreaker.SamplingDuration = TimeSpan.FromSeconds(30);
        options.CircuitBreaker.BreakDuration = TimeSpan.FromSeconds(15);
    });
```

**Building a pipeline by hand** — for anything that is not an `HttpClient`, such as a database or queue call:

```csharp
var pipeline = new ResiliencePipelineBuilder<Price>()
    .AddRetry(new RetryStrategyOptions<Price>
    {
        MaxRetryAttempts = 3,
        Delay = TimeSpan.FromSeconds(1),
        BackoffType = DelayBackoffType.Exponential,
        UseJitter = true,
        ShouldHandle = new PredicateBuilder<Price>()
            .Handle<HttpRequestException>()
            .Handle<TimeoutRejectedException>()      // only transient failures
    })
    .AddCircuitBreaker(new CircuitBreakerStrategyOptions<Price>
    {
        FailureRatio = 0.5,
        SamplingDuration = TimeSpan.FromSeconds(30),
        BreakDuration = TimeSpan.FromSeconds(15)
    })
    .AddFallback(new FallbackStrategyOptions<Price>
    {
        FallbackAction = _ => Outcome.FromResultAsValueTask(Price.Unavailable)
    })
    .Build();

var price = await pipeline.ExecuteAsync(
    async ct => await _client.GetPriceAsync(sku, ct),
    cancellationToken);
```

**Order matters.** Polly applies strategies outside-in, so the one added first wraps the others:

```text
Fallback           ← outermost: catches everything below
   └── Circuit breaker
          └── Retry
                 └── Timeout      ← innermost: per single attempt
                        └── your call
```

Read it as: the timeout applies to one attempt, retry repeats those attempts, the breaker counts the failures, and the fallback catches whatever still failed. Adding retry *outside* the breaker instead would retry even while the circuit is open — which defeats the point.

❌ **Common Polly mistakes:**

- **Retrying non-transient errors.** A 400 or 404 will never succeed. Filter with `ShouldHandle`.
- **Retrying non-idempotent calls.** Retrying a `POST /orders` can create three orders. Retry only with an idempotency key.
- **No jitter.** Without it, every caller retries at the same moment and hammers the recovering service.
- **Retry attempts × timeout exceeding the caller's own timeout.** 3 retries × 5s = 15s, but the caller gave up at 10s. Set `TotalRequestTimeout`.
- **A separate pipeline instance per call.** The circuit breaker holds state, so it must be a singleton — the `AddHttpClient` version handles this for you.

**What to also mention:** bulkhead isolation (limit how many threads one dependency can consume), health checks so the load balancer stops routing to a sick instance, and a fallback for the caller's caller — degrade the whole chain gracefully.

---

### Q1b. How do retry and fallback work when you use a message queue instead of HTTP?

**Answer.** Very differently, and this is a common follow-up. With HTTP the caller waits and must handle the failure itself. With a queue, **the message is already stored safely**, so the queue does most of the work for you.

**The key difference:**

| | HTTP call | Message queue |
|---|---|---|
| **If the receiver is down** | Caller fails immediately | Message waits in the queue |
| **Who retries** | The caller, in memory | The broker, by redelivering |
| **If the caller restarts** | Retry state is lost | Message is still there |
| **Retry duration** | Seconds | Minutes, hours, or days |
| **Fallback needed?** | Usually yes | Often not — waiting *is* the fallback |

This is why moving a call to a queue removes most of the fallback problem. The order is accepted, the email is sent whenever Email Service comes back.

**1. Two levels of retry.** Do not confuse them.

```text
┌─ Immediate retry (in the consumer, in memory) ──────────┐
│  3 attempts, milliseconds apart                         │
│  For: a brief blip — a momentary connection drop         │
└─────────────────────────────────────────────────────────┘
                        │ still failing
                        ▼
┌─ Delayed retry (message goes back to the broker) ───────┐
│  Redelivered after 1 min, 5 min, 15 min                 │
│  For: the dependency is properly down                   │
└─────────────────────────────────────────────────────────┘
                        │ still failing
                        ▼
┌─ Dead-letter queue ────────────────────────────────────┐
│  Stop. A human looks at it.                             │
└─────────────────────────────────────────────────────────┘
```

Immediate retries must be few and fast. Holding a message for 15 minutes inside the consumer blocks a worker thread and the broker may think the consumer died and redeliver it anyway.

**In MassTransit:**

```csharp
builder.Services.AddMassTransit(x =>
{
    x.AddConsumer<OrderPlacedConsumer>();

    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.ReceiveEndpoint("order-placed", e =>
        {
            // Level 1 — immediate, in memory
            e.UseMessageRetry(r => r.Interval(3, TimeSpan.FromMilliseconds(500)));

            // Level 2 — put it back on the broker and try later
            e.UseDelayedRedelivery(r => r.Intervals(
                TimeSpan.FromMinutes(1),
                TimeSpan.FromMinutes(5),
                TimeSpan.FromMinutes(15)));

            e.ConfigureConsumer<OrderPlacedConsumer>(context);
        });
    });
});
```

After all intervals are exhausted, MassTransit moves the message to `order-placed_error` automatically.

**2. The dead-letter queue is the real fallback.**

**Definition — Dead-letter queue (DLQ).** A separate queue holding messages that could not be processed after all retries. Nothing is lost; the message waits for investigation.

What matters operationally:

- **Alert when anything lands in the DLQ.** A silent DLQ filling up is a common production failure — the system looks healthy and data is quietly not being processed.
- **Keep the failure reason** with the message: exception, stack trace, attempt count.
- **Be able to replay it.** Once the bug is fixed, move the messages back to the main queue.

**3. Separate the two failure types — this is the important judgement.**

| Failure | Example | What to do |
|---------|---------|------------|
| **Transient** | Database timeout, service restarting | Retry — it will probably work |
| **Permanent** | Invalid message, missing required field, business rule violation | Do **not** retry. Dead-letter it immediately |

Retrying a malformed message 20 times wastes resources and delays every message behind it.

```csharp
public class OrderPlacedConsumer : IConsumer<OrderPlaced>
{
    public async Task Consume(ConsumeContext<OrderPlaced> context)
    {
        // Permanent problem — retrying will never help
        if (context.Message.CustomerId <= 0)
            throw new MessageValidationException("Invalid CustomerId");   // straight to DLQ

        try
        {
            await _payments.ChargeAsync(context.Message.Amount);
        }
        catch (HttpRequestException)
        {
            throw;      // transient — let the retry policy handle it
        }
    }
}
```

Configure the permanent type to skip retries:

```csharp
e.UseMessageRetry(r =>
{
    r.Interval(3, TimeSpan.FromSeconds(1));
    r.Ignore<MessageValidationException>();      // never retry this one
});
```

**4. Idempotency is not optional here.** Queues guarantee **at-least-once** delivery, not exactly-once. A message *will* be delivered twice — after a retry, a consumer crash, or a broker failover.

```csharp
public async Task Consume(ConsumeContext<OrderPlaced> context)
{
    var messageId = context.MessageId!.Value;

    if (await _processed.ExistsAsync(messageId))
        return;                                    // already handled — do nothing

    await _orders.CreateAsync(context.Message);
    await _processed.AddAsync(messageId);          // same transaction as the work
}
```

Without this, a retry creates a second order. This is the single most common microservice messaging bug.

**5. Poison message and ordering.** A message that always fails will block the queue if you process strictly in order. Two choices:

- Move it to the DLQ quickly and keep going — normal case.
- If order genuinely matters, stop the whole partition and alert. Slower, but correct for things like account balance changes.

**6. When you still need a real fallback with queues.** The queue removes most fallback needs, but not all:

| Situation | Fallback |
|-----------|----------|
| The broker itself is down when you publish | **Outbox pattern** — write the event to a table in the same transaction as your data, publish from there in the background |
| The message is valid but the business rule fails | Publish a compensating event — `OrderFailed` — and undo the earlier steps (saga) |
| Processing is delayed for hours | Notify the user that it is pending, not that it succeeded |
| DLQ keeps filling | Circuit-break the consumer: stop consuming, alert, resume when the dependency is healthy |

The outbox one matters most. Without it, you save the order, the broker is unreachable, and the event is silently lost — the order exists but nothing downstream ever happens.

**Summary of the difference:** with HTTP you design the fallback because the failure is immediate and in your hands. With a queue, the retry ladder and the DLQ *are* the fallback, and your job is to classify failures correctly, make handlers idempotent, and use an outbox so publishing cannot lose the message.

---

### RT2-2 — Migrating .NET Framework 4.8 to .NET Core API + SPA

### Q2. You have a .NET Framework 4.8 application. How do you move it to a .NET Core API with a SPA front end?

**Answer.** Do it incrementally. A full rewrite of a working application usually fails because the business keeps needing changes while you rewrite.

### 1. Assess the current application

- Identify business logic, data access, third-party libraries, and .NET Framework dependencies.
- Check what can be migrated and what must be rewritten.

### 2. Don't rewrite everything

Use an **incremental migration (Strangler Fig Pattern)** instead of a big-bang rewrite.

Keep the existing application running while replacing features one by one.

### 3. Build a new ASP.NET Core Web API

- Create new APIs.
- Implement authentication (JWT/OpenID Connect).
- Keep API contracts stable.

### 4. Extract business logic

Move reusable business logic into shared class libraries so both the old and new applications can use it during migration.

### 5. Build the SPA

Develop an Angular/React application that consumes the new APIs.

Initially, only migrated pages use the SPA; the remaining pages continue using the old application.

### 6. Migrate feature by feature

Example:

```text
Login        → ASP.NET Core API ✓
Customers    → ASP.NET Core API ✓
Orders       → ASP.NET Framework
Reports      → ASP.NET Framework
```

Gradually move each module until everything is migrated.

### 7. Cut over and retire the old application

After all modules are migrated and tested:

- Switch all traffic to the new application.
- Remove the old .NET Framework application.

---

### RT2-3 — Upgrading .NET Core 3.1 to the latest version

### Q3. You have a .NET Core 3.1 API. How do you move it to the latest .NET?

**Answer.** This is much simpler than the 4.8 case — it is the same platform, so it is mostly configuration.

**Step 1 — Upgrade one version at a time, not in one jump.** 3.1 → 6 → 8 → 10. Each step has its own breaking changes, and finding which step broke something is far easier than debugging all of them at once.

Prefer **LTS versions** (even numbers: 6, 8, 10) for production.

**Step 2 — Change the target framework and update packages.**

```xml
<TargetFramework>net8.0</TargetFramework>
```

Update every `Microsoft.*` and `Microsoft.EntityFrameworkCore.*` package to a matching major version. Mismatched versions are the most common cause of startup failures.

**Step 3 — Move `Startup.cs` into `Program.cs`.** From .NET 6 onward the two files are merged.

```csharp
// .NET 8 — one file
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddDbContext<AppDbContext>(o =>
    o.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

The old `Startup.cs` still works if you keep the generic host, so you can postpone this step.

**Step 4 — Fix the known breaking changes.**

| Area | What changed |
|------|--------------|
| **Nullable reference types** | On by default in new templates — expect many warnings |
| **`Newtonsoft.Json`** | Replaced by `System.Text.Json`; date and casing behaviour differs |
| **EF Core** | Client-side evaluation of `Where` no longer silently allowed; some LINQ now throws |
| **`IHostingEnvironment`** | Replaced by `IWebHostEnvironment` |
| **Endpoint routing** | `UseMvc` replaced by `MapControllers` |
| **Middleware order** | `UseRouting` / `UseAuthorization` order is now enforced |

The JSON one causes the most production surprises. `System.Text.Json` is stricter and uses camelCase by default. If clients depend on the old behaviour, either configure it or keep Newtonsoft:

```csharp
builder.Services.AddControllers()
    .AddNewtonsoftJson();          // keep old behaviour while you migrate
```

**Step 5 — Verify.**

- Run the full test suite.
- Compare JSON responses before and after, field by field.
- Load test — most upgrades improve performance, but confirm it.
- Deploy to staging with production-like data.

**Step 6 — Deploy safely.** Blue-green or canary release so you can roll back by switching traffic, not by redeploying.

**What to also mention:** check that your hosting supports the new runtime (IIS module version, Docker base image tag, Azure App Service stack setting) before you start.

---

### RT2-4 — Monolith to microservices

### Q4. How would you convert a monolith into microservices?

**Answer.** Start by asking whether you should. Then, if yes, split by business capability and do it one service at a time.

**Step 1 — Confirm there is a real reason.** Good reasons:

- One part of the app needs to scale very differently from the rest.
- Teams are blocking each other on one release pipeline.
- One module needs a different technology or release rhythm.

Bad reasons: "microservices are modern", "the monolith is messy". A messy monolith becomes messy microservices, plus network calls.

**Step 2 — Clean up the monolith first — make it a modular monolith.** Draw module boundaries inside the single codebase. Each module gets its own folder, its own tables, and a public interface other modules must call through.

This is the most important step, and it is often enough on its own. If you cannot draw clean boundaries inside one process, you certainly cannot draw them across a network.

**Step 3 — Find the boundaries by business capability, not by technical layer.**

```text
❌ Wrong split (by layer)        ✅ Right split (by capability)

   Controller Service              Orders Service
   Business Service                Payments Service
   Data Service                    Shipping Service
   
   Every request needs             Each service owns one
   all three → no benefit          job end to end
```

DDD **bounded contexts** are the tool for this: a group of rules and data that changes together and has one owner.

**Step 4 — Extract the easiest useful service first.** Pick one that is:

- Loosely coupled already.
- Clearly owned by one team.
- Painful today — high load, or slow releases.

Notifications, reporting, and file processing are common first candidates.

**Step 5 — Split the database.** This is the hard part. Each service owns its own data, and no service reads another's tables.

```text
Before                        After

┌─────────────────┐          ┌─────────┐   ┌──────────┐
│    Monolith     │          │ Orders  │   │ Payments │
│                 │          │  API    │   │   API    │
│  ┌───────────┐  │          └────┬────┘   └────┬─────┘
│  │ One shared│  │               │             │
│  │ database  │  │          ┌────▼────┐   ┌────▼─────┐
│  └───────────┘  │          │ Orders  │   │ Payments │
└─────────────────┘          │   DB    │   │    DB    │
                             └─────────┘   └──────────┘
```

Order of work: pick the tables the service owns → remove cross-module joins by calling the other service or duplicating a small amount of data → then physically separate the database.

**Step 6 — Route traffic gradually with Strangler Fig.** Gateway in front, one route at a time, old monolith still serving the rest.

**Step 7 — Handle the things that get harder.** Say these out loud; they are what the interviewer is listening for:

| Problem | Approach |
|---------|----------|
| **No cross-service transactions** | Saga pattern — a sequence of local transactions with compensating actions |
| **Data consistency** | Accept eventual consistency; publish events |
| **A service is down** | Timeouts, retries, circuit breaker (see RT2-1) |
| **Debugging across services** | Distributed tracing with a correlation ID on every call |
| **Duplicate messages** | Make handlers idempotent |
| **Reporting across services** | A read model or data warehouse fed by events |

❌ **The honest answer to give:** microservices trade a code problem for an operations problem. You get independent deploys and scaling, and you pay with network failures, eventual consistency, and much harder debugging. If the team does not have automated deployment, monitoring, and tracing already, fix that first.

---

### RT2-5 — Choosing monolith, modular monolith, or microservices

### Q5. A new requirement arrives. How do you decide whether it should be a monolith, a modular monolith, or a microservice?

**Answer.** Default to the simplest option and only move up when there is a specific reason.

**The three options:**

| Style | What it is | Deploys as |
|-------|------------|------------|
| **Monolith** | One codebase, one database, no internal boundaries | One unit |
| **Modular monolith** | One codebase and deployment, but strict module boundaries and separate tables per module | One unit |
| **Microservices** | Separate codebases, own databases, own deployments | Many units |

**Step 1 — Start from the modular monolith.** It gives most of the benefit of microservices — clear boundaries, code that is easy to reason about — with none of the network cost. It also keeps the option open: a well-bounded module is straightforward to extract later.

**Step 2 — Ask these questions in order.**

| Question | If yes → |
|----------|----------|
| Does this part need to scale independently? | Microservice |
| Does a separate team own it, with its own release schedule? | Microservice |
| Does it need a different technology or database type? | Microservice |
| Must it stay up when the rest of the app is down? | Microservice |
| Is it tightly tied to existing data with lots of joins? | Keep in the monolith |
| Are the boundaries still unclear? | Keep in the monolith |
| Is the team small, with no CI/CD or monitoring? | Keep in the monolith |

**Step 3 — Apply the practical rules.**

- **Two or more "yes" answers on the top four** — extract a service.
- **Unclear boundaries** — never split. A wrong boundary is far more expensive than a monolith, because fixing it means changing two services, two databases, and their contract.
- **Don't split what changes together.** If two features are always released together, they belong in one unit.

**Step 4 — Consider team size honestly.** A common guideline: **one service per team, not one service per developer.** Five developers running fifteen services spend their time on infrastructure instead of features.

**Step 5 — Give the interviewer a concrete answer.** Example:

> "A new PDF report generator: heavy CPU, runs on a schedule, needs no shared transactions with orders. That scales differently and is loosely coupled, so I would make it a separate service. A new discount rule on checkout touches order data on every request and belongs in the same module as orders."

That contrast — one extracted, one kept — is what shows judgement.

---

### RT2-6 — From requirement to production

### Q6. A fresh requirement comes in. Explain what you do, from planning to production.

**Answer.** Walk through it as a pipeline. Naming the verification and rollback steps is what separates a senior answer.

**1. Understand the requirement.**

- What problem does the user actually have? Ask for the goal, not the requested solution.
- What are the acceptance criteria? Write them down as testable statements.
- Non-functional needs: expected load, response time, data volume, retention, compliance.
- What is explicitly *out* of scope.

**2. Design.**

- Which existing modules or services does it touch?
- Data model changes — new tables, new columns, migrations.
- API contract: endpoints, request and response shapes, error cases.
- Does it need async processing, caching, or a new external integration?
- Security: who is allowed to do this, what data is sensitive.
- Write a short design note and get it reviewed before coding. One page is enough.

**3. Break it into small pieces.** Each piece should be shippable on its own and take a day or two. Big branches that live for weeks cause painful merges.

**4. Build.**

- Feature branch off `main`.
- Write the code with unit tests as you go.
- Keep the business rules in a class you can test with no database.
- Add an EF Core migration for schema changes.
- Put the feature behind a **feature flag** if it is risky or needs a coordinated launch.

**5. Test.**

| Level | Checks |
|-------|--------|
| **Unit** | Business rules, edge cases, no database |
| **Integration** | API + real database, using Testcontainers or a test database |
| **Contract** | The response shape other services depend on has not changed |
| **Manual / QA** | The acceptance criteria from step 1 |
| **Performance** | Only if the feature touches a hot path |

**6. Review.**

- Pull request with a description of what and why.
- CI must pass: build, tests, linter, security scan.
- At least one reviewer.

**7. Deploy.**

- Merge to `main` → CI builds an artifact once.
- Deploy that same artifact to **dev → staging → production**. Never rebuild per environment.
- Run migrations **before** the new code, and keep them backward compatible so the old version still runs. This is what makes rollback possible.
- Use **canary or blue-green** so only a small percentage of traffic hits the new version first.

**8. Verify in production.**

- Watch error rate, latency, and CPU for the first period after release.
- Check the specific logs or metric the feature adds.
- Confirm the acceptance criteria with real traffic.

**9. Have a rollback plan before you deploy.**

- Turn off the feature flag — fastest option, no deployment needed.
- Or route traffic back to the previous version.
- Because migrations were backward compatible, the old code still works against the new schema.

**10. After release.**

- Update the documentation and the API contract.
- Remove the feature flag once it is stable — old flags become dead code.
- Note anything that slowed you down for the next iteration.

---

### RT2-7 — Testing microservices

### Q7. How do you test a microservice?

**Answer.** Test each service on its own as much as possible, and test across services as little as you can get away with. Cross-service tests are slow and fail for unrelated reasons.

**The test pyramid for services:**

```text
        ▲          End-to-end        few    slow, brittle
       ╱ ╲         ─────────────
      ╱   ╲        Contract tests    some
     ╱     ╲       ─────────────
    ╱       ╲      Integration       more
   ╱         ╲     ─────────────
  ╱___________╲    Unit tests        many   fast, reliable
```

**1. Unit tests — the business rules.** No database, no HTTP. Fast, and they are where most of your coverage should be.

```csharp
[Fact]
public void Order_over_10000_gets_discount()
{
    var order = new Order(20000);

    order.ApplyBulkDiscount();

    Assert.Equal(19000, order.Amount);
}
```

**2. Integration tests — the service with its real database.** Use `WebApplicationFactory` plus a real database in a container.

```csharp
public class OrderApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public OrderApiTests(WebApplicationFactory<Program> factory)
        => _client = factory.CreateClient();

    [Fact]
    public async Task Post_order_returns_created()
    {
        var response = await _client.PostAsJsonAsync("/api/orders", new { Amount = 500 });

        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
    }
}
```

Use **Testcontainers** for a real SQL Server or PostgreSQL in Docker. The EF Core in-memory provider does not behave like a real database — it accepts queries and constraint violations that SQL Server rejects.

**3. Contract tests — the agreement between two services.** This is the one people forget, and it is the most valuable in a microservice system.

**Definition — Contract test.** The consumer writes down what it expects from the provider's API. The provider runs that expectation as part of its own build. If the provider breaks the shape, its build fails — before deployment.

```text
Orders Service (consumer)          Pricing Service (provider)
        │                                    │
   "I expect GET /price/{id}          runs the consumer's
    to return { amount: number }"  ──►  expectations in CI
                                             │
                                    breaks the build if
                                    the shape changed
```

Pact is the common tool. Without contract tests, you only find a broken contract in a shared environment, days later.

**4. Component tests — the service with its dependencies faked.** Run the real service, but replace the other services with stubs (WireMock). This is where you test your **failure handling** — make the stub return 500 or a timeout and check that the retry and circuit breaker behave.

**5. End-to-end tests — a real user journey across services.** Keep a handful, covering only critical paths like signup and checkout. They are valuable but slow and flaky; a large E2E suite becomes a suite nobody trusts.

**6. Also test in production.**

- **Health checks** — `/health` endpoints so the orchestrator can restart a sick instance.
- **Synthetic monitoring** — a scheduled fake transaction that alerts if it fails.
- **Distributed tracing** — correlation IDs so one request can be followed across services.

**What to also mention:** test data setup is the hardest practical problem. Each test should create the data it needs and clean up after itself; shared test data causes tests that pass alone and fail together.

---

### RT2-8 — Testing a PySpark + .NET API ETL application

### Q8. How do you test an ETL application built from PySpark jobs and a .NET API?

**Answer.** ETL is different from a normal API: the risk is **wrong data**, not a failed request. A job can succeed and still be broken. So the tests focus on the transformation logic and on validating the data at each stage.

**1. Test the transformation logic on its own (PySpark).** Split each transformation into a plain function that takes a DataFrame and returns a DataFrame. Then test it with a tiny hand-written DataFrame — a few rows, not a real file.

```python
def apply_discount(df):
    return df.withColumn(
        "final_amount",
        when(col("amount") > 10000, col("amount") * 0.95).otherwise(col("amount"))
    )

def test_discount_applies_over_threshold(spark):
    input_df = spark.createDataFrame(
        [(1, 20000.0), (2, 5000.0)],
        ["id", "amount"]
    )

    result = apply_discount(input_df).collect()

    assert result[0]["final_amount"] == 19000.0
    assert result[1]["final_amount"] == 5000.0
```

Use a local Spark session in a pytest fixture. Keep these tests row-level and small — they run in seconds and catch most logic bugs.

**2. Test the edge cases that break real pipelines.** These are what actually fail in production:

- Nulls in a column you assumed was always populated.
- Duplicate rows from a re-delivered source file.
- Wrong data type — a number arriving as a string.
- Empty input file.
- Unicode and encoding issues.
- Dates in a different timezone or format.
- A schema change in the source — a new or renamed column.

**3. Validate the data, not just the code.** Add checks that run as part of the job and fail it when the data is wrong.

| Check | Example |
|-------|---------|
| **Row count** | Output rows within an expected range of input rows |
| **Not null** | Key columns are never null |
| **Uniqueness** | No duplicate business keys |
| **Range** | Amounts are not negative; dates are not in the future |
| **Referential** | Every `customer_id` exists in the customer table |
| **Reconciliation** | Sum of amounts in equals sum of amounts out |

The reconciliation check is the most valuable one. Tools: **Great Expectations** or **Soda** for Python, or plain assertion queries.

**4. Test the schema explicitly.** Assert the output schema — column names, types, nullability. A silent schema change is the most common way a downstream consumer breaks.

**5. Test the .NET API separately.** Normal unit and integration tests, as in RT2-7. Two things specific to ETL:

- If the API triggers a job, test that it enqueues correctly and returns immediately — do not run the job in the test.
- If the API reads the ETL output, test it against a small known dataset.

**6. Test idempotency — the ETL-specific requirement.** Jobs get re-run after failures. Running the same job twice with the same input must not double the data.

```python
def test_job_is_idempotent(spark):
    run_job(input_path, output_path)
    first = spark.read.parquet(output_path).count()

    run_job(input_path, output_path)          # run again
    second = spark.read.parquet(output_path).count()

    assert first == second                    # no duplication
```

Use a merge/upsert on a business key rather than an append, or partition overwrite by date.

**7. Integration test the whole pipeline on a small dataset.** Real files in, real database out, but a handful of rows. Compare against an expected output file. Docker Compose or Testcontainers gives you the database.

**8. Monitor in production.**

- Log row counts in and out for every run.
- Alert on the job failing **and** on it finishing suspiciously fast or with far fewer rows.
- Track how long each stage takes so you notice slow growth.
- Keep the failed input file so you can replay it.

❌ **The mistake to avoid:** testing only with the happy-path sample file. Production data is messy — nulls, duplicates, encoding problems — and those cases are where ETL jobs actually break.

---

### RT2-9 — Building 5 microservices with integration

### Q9. You need to build an app made of 5 microservices. How do you do it, including the integration between them?

**Answer.** Use a concrete example so the answer is not abstract. Say an e-commerce app:

```text
                    ┌─────────────┐
   Client ──────────►   Gateway   │  auth, routing, rate limit
                    └──────┬──────┘
          ┌────────┬───────┼───────┬────────┐
          ▼        ▼       ▼       ▼        ▼
      Identity  Catalog  Orders Payments Shipping
          │        │       │       │        │
        ┌─▼─┐    ┌─▼─┐   ┌─▼─┐   ┌─▼─┐    ┌─▼─┐
        │DB │    │DB │   │DB │   │DB │    │DB │   one DB each
        └───┘    └───┘   └───┘   └───┘    └───┘
                          │       │        │
                    ┌─────▼───────▼────────▼─────┐
                    │      Message Broker        │  events
                    └────────────────────────────┘
```

**Step 1 — Define what each service owns.** One clear responsibility, and its own data.

| Service | Owns | Never touches |
|---------|------|---------------|
| **Identity** | Users, roles, tokens | Order data |
| **Catalog** | Products, prices, stock levels | Customer data |
| **Orders** | Orders, order lines, status | Payment details |
| **Payments** | Transactions, refunds | Product data |
| **Shipping** | Shipments, tracking | Payment details |

**Step 2 — Choose how they talk. Two mechanisms, used for different things.**

| Mechanism | Use for | Example |
|-----------|---------|---------|
| **Synchronous** (HTTP/gRPC) | The caller needs an answer now | Orders asks Catalog "is this in stock?" |
| **Asynchronous** (events) | Telling others something happened | Orders publishes `OrderPlaced` |

The rule: **query synchronously, notify asynchronously.** If Orders had to call Payments, Shipping, and Email synchronously, one slow service would slow down checkout and any one being down would break it.

**Step 3 — Design the event flow for the main use case.** Place an order:

```text
1. Client → Gateway → Orders API
2. Orders  → Catalog (sync)  "is SKU-1 in stock?"
3. Orders  → saves order as Pending
4. Orders  → publishes OrderPlaced ─────────┐
                                            ▼
5. Payments   subscribes → charges → publishes PaymentSucceeded
                                            │
6. Orders     subscribes → marks order Paid │
7. Shipping   subscribes → creates shipment │
8. Notification subscribes → sends email ───┘
```

The client gets a response after step 4. Everything else happens in the background. Checkout stays fast, and Shipping being down does not block an order.

**Step 4 — Handle failure across services with a Saga.**

**Definition — Saga.** A business transaction spread across services, done as a sequence of local transactions. If one step fails, earlier steps are undone with **compensating actions** rather than a rollback.

```text
Happy path:  Reserve stock → Charge card → Create shipment

Payment fails:
             Reserve stock → Charge card ✗
                    │
                    ▼
             Release stock (compensating action)
             Mark order as Failed
             Notify the customer
```

There is no distributed transaction to roll back, so every step needs a defined undo.

**Step 5 — Get the integration details right.** These are what the interviewer is checking:

- **Idempotency.** Messages get delivered twice. Every handler must check "have I already processed this message ID?"
- **Outbox pattern.** Saving the order and publishing the event must not be two separate risky steps. Write the event to an `Outbox` table inside the same database transaction as the order, and have a background worker publish from that table. Otherwise the order saves and the event is lost.
- **Retries and dead-letter queue.** After N failures, park the message for a human instead of retrying forever.
- **Message versioning.** Add fields, never remove or rename them. Old consumers must keep working.
- **Correlation ID.** Generated at the gateway, passed through every call and message, logged everywhere.

**Step 6 — Add the shared infrastructure.**

| Concern | Tool |
|---------|------|
| **Gateway** | YARP, Azure API Management |
| **Auth** | One identity provider, JWT validated at the gateway |
| **Messaging** | RabbitMQ, Azure Service Bus, Kafka + MassTransit |
| **Service discovery** | Kubernetes DNS, or configuration |
| **Config and secrets** | Azure App Configuration, Key Vault |
| **Logging** | Structured logs to one place — Seq, ELK |
| **Tracing** | OpenTelemetry + Jaeger or Application Insights |
| **Health checks** | `/health` on every service |

**Step 7 — Build it in a sensible order.** Do not build five services in parallel and integrate at the end.

1. Gateway and Identity — everything needs auth.
2. Catalog — mostly read-only, simplest.
3. Orders — the core, integrating with Catalog.
4. Payments — with the saga.
5. Shipping — event subscriber only.

Get one full path working end to end before adding the next service. Integration problems found early are cheap.

**Step 8 — Local development matters.** Five services plus a broker and databases must run on a developer machine. **Docker Compose** or **.NET Aspire** — Aspire is built for exactly this and gives you a dashboard, service discovery, and telemetry wiring for free.

---

### RT2-10 — Security from front end to database

### Q10. Front end to back end to database — how do you secure your services?

**Answer.** Go layer by layer, and assume every layer can be attacked. The principle is **defence in depth** — no single control is the only thing protecting you.

**1. Front end.**

- Store tokens carefully. An **HttpOnly, Secure, SameSite cookie** cannot be read by JavaScript, so an XSS bug cannot steal it. `localStorage` can.
- Never keep secrets or API keys in front-end code — it is fully visible to the user.
- Escape all rendered user content to prevent XSS. Angular and React escape by default; the risk is `innerHTML` and `bypassSecurityTrustHtml`.
- Set a **Content Security Policy** header to limit which scripts can run.
- Validate input for user experience, but never trust it — validate again on the server.

**2. In transit.**

- HTTPS everywhere, including between internal services.
- **HSTS** header so the browser refuses plain HTTP.
- TLS 1.2 minimum.

**3. Gateway / API.**

- **Authentication** — who are you. Validate the JWT: signature, issuer, audience, expiry.
- **Authorization** — what may you do. Check it on **every** endpoint, per resource.

**Definition — the difference.** Authentication proves identity. Authorization decides permission. Passing the first says nothing about the second.

```csharp
[Authorize(Roles = "Admin")]
[HttpDelete("{id}")]
public async Task<IActionResult> Delete(int id)
{
    var order = await _repo.GetByIdAsync(id);

    // ❌ Being an Admin is not enough — check ownership too
    if (order.TenantId != CurrentUser.TenantId)
        return Forbid();

    // ...
}
```

That ownership check is the most commonly missed control. Without it, changing the ID in the URL exposes another user's data — **IDOR** (insecure direct object reference).

- **Rate limiting** per user and per IP, to slow brute force and abuse.
- **Validate all input** — model validation, length limits, allowed values.
- **CORS** — list allowed origins explicitly. Never `AllowAnyOrigin` with credentials.
- Return generic error messages. Stack traces tell an attacker about your internals.
- Do not log tokens, passwords, or card numbers.

**4. Service to service.**

- Do not treat "inside the network" as trusted — **zero trust**. Every internal call authenticates too.
- Use client credentials tokens or **mTLS** between services.
- Give each service its own identity with only the permissions it needs.

**5. Database.**

- **Parameterised queries always.** EF Core and Dapper do this; string concatenation does not.

```csharp
// ❌ SQL injection
var sql = $"SELECT * FROM Users WHERE Email = '{email}'";

// ✅ Parameterised
var user = await _db.Users.FirstOrDefaultAsync(u => u.Email == email);
```

- **Least privilege.** The application's database user should not be `db_owner`. It needs read/write on its tables, not schema rights.
- **Encrypt at rest** — TDE — and encrypt sensitive columns individually.
- **Hash passwords** with bcrypt, Argon2, or ASP.NET Core Identity's hasher. Never encrypt them, never store them plain, never use plain MD5 or SHA-256.
- No shared connection strings in source control — use Key Vault or managed identity.
- The database should not be reachable from the internet. Private network only.

**6. Secrets and configuration.**

- Key Vault or a secrets manager. Never `appsettings.json` in git.
- Managed identity where the cloud supports it, so there is no secret at all.
- Rotate keys, and have a plan for when one leaks.

**7. Monitoring.**

- Log every authentication and authorization failure.
- Alert on unusual patterns — many 401s, one user hitting many resource IDs.
- Keep an audit trail of who changed what.

**8. The whole picture.**

```text
Browser
   │  HTTPS, HttpOnly cookie, CSP
   ▼
Gateway
   │  validate JWT, rate limit, CORS
   ▼
API
   │  authorize per resource, validate input
   ▼
Service layer
   │  mTLS / client credentials between services
   ▼
Database
      parameterised queries, least privilege, encrypted at rest
```

**What to also mention:** the **OWASP Top 10** as the checklist you work against, dependency scanning for vulnerable NuGet packages, and that broken access control is the number one issue in practice — not exotic attacks.

---

### RT2-11 — Signature and certificate validation app

### Q11. You need to design an app that validates signatures and certificates. How would you do it, and where could AI services help?

**Answer.** Separate the question into two very different problems, because they need opposite approaches.

| Type | What it is | How to validate |
|------|------------|-----------------|
| **Digital signature** | Cryptographic signature on a document or file | Maths — either valid or not. **Never AI.** |
| **Handwritten signature** | An image of a person's signature | Comparison — a confidence score. AI helps here. |

Interviewers often mix these deliberately. Naming the difference is the first mark.

---

#### Part 1 — Digital signature and certificate validation

This is cryptography. The answer is a definite yes or no, and AI has no role in it.

**What must be checked, in order:**

| Check | Question |
|-------|----------|
| **Signature validity** | Does the signature match the document using the public key? |
| **Document integrity** | Has a single byte changed since signing? |
| **Certificate chain** | Does it chain up to a trusted root CA? |
| **Validity period** | Is it within `NotBefore` and `NotAfter`? |
| **Revocation** | Has it been revoked — CRL or OCSP? |
| **Key usage** | Is this certificate allowed to be used for signing? |
| **Timestamp** | Was it signed while the certificate was still valid? |

The revocation and timestamp checks are the ones people miss. A certificate valid today may have been revoked yesterday; a signature made two years ago should still verify even though the certificate has since expired — which is what a trusted timestamp proves.

**In .NET:**

```csharp
using var cert = new X509Certificate2("signer.cer");

var chain = new X509Chain();
chain.ChainPolicy.RevocationMode = X509RevocationMode.Online;      // check OCSP/CRL
chain.ChainPolicy.RevocationFlag = X509RevocationFlag.EntireChain;
chain.ChainPolicy.VerificationFlags = X509VerificationFlags.NoFlag;

bool chainOk = chain.Build(cert);

if (!chainOk)
{
    foreach (var status in chain.ChainStatus)
        Console.WriteLine($"{status.Status}: {status.StatusInformation}");
}
```

Verifying a signed payload:

```csharp
using var rsa = cert.GetRSAPublicKey()!;

bool valid = rsa.VerifyData(
    documentBytes,
    signatureBytes,
    HashAlgorithmName.SHA256,
    RSASignaturePadding.Pkcs1);
```

**Architecture:**

```text
Client ──► API ──► Validation Service
                        │
        ┌───────────────┼────────────────┐
        ▼               ▼                ▼
   Crypto check    Chain build      OCSP / CRL
   (signature)     (trust)          (revocation)
        │               │                │
        └───────────────┼────────────────┘
                        ▼
                  Audit log (immutable)
                        │
                        ▼
              Result: Valid / Invalid + reason
```

**Design points to mention:**

- **Never** store private keys in the application. Use an HSM, Azure Key Vault, or the OS certificate store.
- **Audit everything** — what was validated, when, by whom, and the outcome. This app is usually part of a compliance process, so the log is as important as the result.
- **Cache OCSP responses** for a short period; they are network calls on a hot path.
- **Fail closed.** If revocation cannot be checked, treat it as invalid rather than assuming valid — the safe default for security decisions.
- For PDFs, use a library such as iText or PDFsharp rather than parsing signature structures yourself.
- Standards to name: **X.509** (certificate format), **PKCS#7 / CMS** (signature format), **PAdES** (PDF signatures), **XAdES** (XML), **RFC 3161** (timestamping).

---

#### Part 2 — Handwritten signature verification

This is image comparison, and the output is a **confidence score**, not a yes or no. AI belongs here.

**Flow:**

```text
Scanned document
       │
       ▼
1. Detect the signature region        ← object detection
       │
       ▼
2. Clean the image                    ← deskew, remove lines, normalise
       │
       ▼
3. Extract features                   ← CNN embedding
       │
       ▼
4. Compare with the stored reference  ← similarity score
       │
       ▼
5. Score above threshold?
       │
   ┌───┴────┬─────────────┐
   ▼        ▼             ▼
 Accept   Reject    Send to human review
 (>95%)   (<60%)      (60–95%)
```

**Where AI services genuinely help:**

| Task | Service | Why |
|------|---------|-----|
| **Reading the document** | Azure AI Document Intelligence | OCR, field extraction, and it detects signature presence |
| **Finding the signature** | Azure AI Vision / custom object detection | Locates the region on the page |
| **Comparing signatures** | Custom CNN model, or a signature-verification API | Similarity scoring is a learned task |
| **Detecting tampering** | Image forensics models | Spots copy-paste or edited regions |
| **Classifying documents** | Document Intelligence custom model | Routes each type to the right validation rules |
| **Summarising results** | An LLM | Turns a validation report into readable text for a reviewer |

**Where AI must not be used:**

- ❌ Verifying a **cryptographic** signature. That is deterministic maths. An AI answer would be a probability where a certainty is required and available.
- ❌ Deciding certificate trust or revocation.
- ❌ Being the sole approver on a high-value decision. Keep a human in the loop above a threshold.

**Design points:**

- **Always return a score plus a reason**, never just "rejected". The reviewer needs to know why.
- **Three-way outcome** — accept, reject, and human review. A two-way threshold either rejects valid signatures or accepts forgeries.
- **Log the model version** with every decision. When the model is retrained, you must be able to explain past results.
- **Store the reference signatures encrypted.** They are biometric data, which in the EU falls under GDPR Article 9 special categories.
- **Measure the right error.** For fraud, a false accept is far more costly than a false reject. Tune the threshold accordingly, and report both rates.

**The combined answer:** one API with two pipelines behind it — a deterministic crypto pipeline whose answer is final, and an AI-assisted image pipeline whose answer is a score with a human fallback. Keeping those separate, and being explicit that AI never touches the cryptographic path, is the point of the question.
