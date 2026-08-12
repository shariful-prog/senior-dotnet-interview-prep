# .NET / ASP.NET Core Top 20 Interview Questions (Quick Answers)

> Goal: Fast revision. Each question opens with a one-line definition, then the detail — plus a link to the exact section in the detailed .NET notes.

## Platform & DI

### 1. What is dependency injection?

**Definition:** **Dependency injection** means a class receives the things it depends on from outside — normally through its constructor — instead of creating them itself.

**Detail:** The class then depends on an interface rather than a concrete type, so you can swap the implementation, pass a fake in tests, and keep object lifetime managed in one place. The container wires it all up at startup.

:material-file-document-outline: **Deep dive:** [J1 — Dependency Injection](../Dotnet/dotnet-platform.md#j1--dependency-injection)

---

### 2. What are the three service lifetimes?

**Definition:** **Transient** creates a new instance every time it's resolved. **Scoped** creates one per request. **Singleton** creates one for the whole application.

**Detail:** `DbContext` is the classic scoped service — one per request, so all work in that request shares a change tracker and a transaction. Anything registered as a singleton must be thread-safe, because every request will use the same instance at the same time.

:material-file-document-outline: **Deep dive:** [J1 — Dependency Injection](../Dotnet/dotnet-platform.md#j1--dependency-injection)

---

### 3. What is a captive dependency?

**Definition:** A **captive dependency** is a shorter-lived service trapped inside a longer-lived one — typically a scoped service injected into a singleton.

**Detail:** The singleton grabs one instance at startup and holds it forever, so a `DbContext` meant to live for one request lives for the life of the app. You get stale data, thread-safety bugs, and disposal errors. The fix is to inject `IServiceScopeFactory` and create a fresh scope for each unit of work.

:material-file-document-outline: **Deep dive:** [J1 — Dependency Injection](../Dotnet/dotnet-platform.md#j1--dependency-injection)

---

### 4. What is the difference between .NET Framework, .NET Core, and .NET 5+?

**Definition:** **.NET Framework** is the original Windows-only platform, frozen at 4.8. **.NET Core** was the cross-platform rewrite. **.NET 5+** is the unified continuation of .NET Core.

**Detail:** Modern .NET is cross-platform, open source, and ships every November — even-numbered releases are LTS with three years of support. New work should target current .NET. `.NET Standard` only matters now if you're sharing a library with old Framework code.

:material-file-document-outline: **Deep dive:** [J2 — .NET Framework vs .NET Core vs .NET 5+](../Dotnet/dotnet-platform.md#j2--net-framework-vs-net-core-vs-net-5-net-standard)

---

### 5. What are the CLR, IL, and the JIT?

**Definition:** **IL** is the portable intermediate language your C# compiles into. The **CLR** is the runtime that loads and executes it. The **JIT** compiles IL into native machine code, one method at a time, as it's first called.

**Detail:** Compiling at runtime lets the JIT optimize for the actual CPU it's running on and re-optimize hot paths. **Native AOT** compiles everything ahead of time instead — much faster startup and smaller memory, at the cost of reflection and dynamic loading.

:material-file-document-outline: **Deep dive:** [J1b — The Runtime: CLR, IL & JIT](../Dotnet/dotnet-platform.md#j1b--the-runtime-clr-il--jit)

## ASP.NET Core Fundamentals

### 6. What is middleware?

**Definition:** **Middleware** is a component in the request pipeline. Each one receives the `HttpContext`, can do work before and after calling the next component, and can stop the chain by not calling it.

**Detail:** They run in the order you register them, which matters enormously. Exception handling goes first so it wraps everything; authentication must come before authorization; endpoints go last. Getting the order wrong produces bugs that look like configuration problems.

:material-file-document-outline: **Deep dive:** [M2 — The Request Pipeline & Middleware](../Dotnet/dotnet-aspnetcore-fundamentals.md#m2--the-request-pipeline--middleware-core)

---

### 7. What is the difference between `app.Use`, `app.Run`, and `app.Map`?

**Definition:** `Use` adds middleware that can pass the request along. `Run` is **terminal** — it never calls the next component. `Map` branches the pipeline based on a path.

**Detail:** `Use` receives a `next` delegate and normally calls it. `Run` doesn't, so nothing registered after it will ever execute — registering a `Run` too early silently swallows the rest of your pipeline. `MapWhen` branches on any predicate rather than just a path.

:material-file-document-outline: **Deep dive:** [M2 — The Request Pipeline & Middleware](../Dotnet/dotnet-aspnetcore-fundamentals.md#m2--the-request-pipeline--middleware-core)

---

### 8. What is the Options pattern?

**Definition:** The **Options pattern** binds a section of configuration to a strongly typed class, which you then inject.

**Detail:** Configuration providers layer in order — `appsettings.json`, then the environment-specific file, then environment variables, then command line — with later ones overriding earlier. Three ways to consume it: `IOptions<T>` (singleton, read once), `IOptionsSnapshot<T>` (per scope, picks up changes), and `IOptionsMonitor<T>` (singleton with change notifications).

:material-file-document-outline: **Deep dive:** [M4 — Configuration & the Options Pattern](../Dotnet/dotnet-aspnetcore-fundamentals.md#m4--configuration--the-options-pattern-core)

---

### 9. How do you run background work in ASP.NET Core?

**Definition:** **`BackgroundService`** is a base class for long-running work that starts with the app and stops with it. It's the convenient version of `IHostedService`.

**Detail:** You override `ExecuteAsync` and honour the `CancellationToken` so shutdown is graceful. The catch: it's registered as a **singleton**, so you can't inject a scoped service like `DbContext` directly — create a scope inside each iteration with `IServiceScopeFactory`.

:material-file-document-outline: **Deep dive:** [M6 — IHostedService / BackgroundService](../Dotnet/dotnet-aspnetcore-fundamentals.md#m6--ihostedservice--backgroundservice)

---

### 10. What is Kestrel?

**Definition:** **Kestrel** is the cross-platform web server built into ASP.NET Core. It's the thing actually running your application.

**Detail:** Kestrel can face the internet directly, but a reverse proxy in front (IIS, Nginx) adds process management, port sharing, request filtering, and certificate handling at the edge. In-process hosting on IIS runs your app inside the IIS worker process and is faster than the out-of-process option.

:material-file-document-outline: **Deep dive:** [M1 — Kestrel & the Hosting Model](../Dotnet/dotnet-aspnetcore-fundamentals.md#m1--kestrel--the-hosting-model)

## Web APIs

### 11. What is model binding, and what does `[ApiController]` change?

**Definition:** **Model binding** maps incoming request data — route values, query string, form fields, JSON body — onto your action's parameters. **`[ApiController]`** is an attribute that turns on API-specific conventions.

**Detail:** After binding, validation runs the data annotations and fills `ModelState`. `[ApiController]` then gives you automatic `400` responses when validation fails, inferred binding sources (`[FromBody]` on complex types), and ProblemDetails-shaped errors — so you rarely need to write `if (!ModelState.IsValid)` yourself.

:material-file-document-outline: **Deep dive:** [N3 — Model Binding & Validation](../Dotnet/dotnet-web-apis.md#n3--model-binding--validation-core)

---

### 12. Which status codes should a REST API return?

**Definition:** The status code tells the client **what category of thing happened**: 2xx success, 4xx the caller's fault, 5xx the server's fault.

**Detail:** `200 OK` for a successful read, `201 Created` with a `Location` header for a create, `204 No Content` for a delete, `400` for validation failures, `401` when you don't know who they are versus `403` when you do and they're not allowed, `404` for missing, `409` for a conflict. Use the helpers — `Ok()`, `CreatedAtAction()`, `NotFound()` — rather than raw numbers.

:material-file-document-outline: **Deep dive:** [N4 — Results & Status Codes](../Dotnet/dotnet-web-apis.md#n4--results--status-codes)

---

### 13. What are filters, and how do they differ from middleware?

**Definition:** A **filter** runs inside the MVC pipeline and knows about actions, model state, and the selected endpoint. **Middleware** sits lower down and sees every request.

**Detail:** There are several kinds — authorization, resource, action, exception, and result filters — each hooking a different stage. Use a filter when your cross-cutting concern needs MVC context (like the action arguments); use middleware when it applies to all traffic, including requests that never reach a controller.

:material-file-document-outline: **Deep dive:** [N5 — Filters](../Dotnet/dotnet-web-apis.md#n5--filters)

---

### 14. How should an API handle errors globally?

**Definition:** **Global error handling** means catching unhandled exceptions in one place — exception-handling middleware or `IExceptionHandler` — rather than wrapping every action in try/catch.

**Detail:** Return **ProblemDetails** (RFC 7807) so every error has the same shape and clients can parse it. Log the full exception server-side, but never send stack traces, SQL, or connection strings back to the caller in production.

:material-file-document-outline: **Deep dive:** [N9 — Error Handling & Standards](../Dotnet/dotnet-web-apis.md#n9--error-handling--standards) · [P3 — Exception Handling](../Dotnet/dotnet-auth-logging.md#p3--exception-handling)

## EF Core

### 15. What is the change tracker?

**Definition:** The **change tracker** is the part of `DbContext` that remembers every entity it loaded and what its values were at the time.

**Detail:** When you call `SaveChanges`, EF compares each entity's current values against that original snapshot, works out what actually changed, and generates only the necessary `INSERT`/`UPDATE`/`DELETE` statements — all wrapped in a single transaction, so it's all-or-nothing.

:material-file-document-outline: **Deep dive:** [O1 — DbContext, DbSet & the Change Tracker](../Dotnet/dotnet-ef-core.md#o1--dbcontext-dbset--the-change-tracker)

---

### 16. What is the N+1 problem?

**Definition:** The **N+1 problem** is issuing one query to fetch a list, then one more query per item as you access its related data — 101 round trips for 100 rows.

**Detail:** It's usually caused by lazy loading. Fix it with **eager loading** (`Include()`) to fetch everything in one query, or **explicit loading** when you want control. Often better still: project with `Select()` to exactly the columns you need, which avoids loading whole entities at all.

:material-file-document-outline: **Deep dive:** [O3 — Loading Strategies & the N+1 Problem](../Dotnet/dotnet-ef-core.md#o3--loading-strategies--the-n1-problem)

---

### 17. What does `AsNoTracking()` do?

**Definition:** **`AsNoTracking()`** tells EF not to track the entities a query returns.

**Detail:** Skipping the snapshot and identity resolution means less memory and measurably faster reads, especially on large result sets. Use it for anything read-only. The trade-off: the entities come back detached, so editing them and calling `SaveChanges` does nothing.

:material-file-document-outline: **Deep dive:** [O4 — Change Tracking vs AsNoTracking](../Dotnet/dotnet-ef-core.md#o4--change-tracking-vs-asnotracking)

---

### 18. How do you handle concurrency conflicts in EF Core?

**Definition:** **Optimistic concurrency** means letting two users edit at once, and detecting at save time whether someone else got there first.

**Detail:** Mark a column with `[Timestamp]` or `IsRowVersion()`. EF then adds it to the `WHERE` clause of every `UPDATE`. If the row was changed since you read it, zero rows match and you get `DbUpdateConcurrencyException` — which you resolve by store-wins, client-wins, or merging. Without this you get silent lost updates.

:material-file-document-outline: **Deep dive:** [O7 — Transactions & Concurrency](../Dotnet/dotnet-ef-core.md#o7--transactions--concurrency)

## Security, Performance & Ops

### 19. What is the difference between authentication and authorization?

**Definition:** **Authentication** establishes *who you are*. **Authorization** decides *what you're allowed to do*.

**Detail:** With JWT bearer tokens, the client sends a signed token; the server validates the signature, issuer, audience, and expiry, then builds a `ClaimsPrincipal`. Authorization then applies roles or policies to that principal. In the pipeline `UseAuthentication()` must come before `UseAuthorization()` — you can't check permissions before you know who's asking. Never trust claims without validating the signature.

:material-file-document-outline: **Deep dive:** [S1 — Authentication & JWT](../Dotnet/dotnet-security.md#s1--authentication--jwt) · [P2 — Authorization](../Dotnet/dotnet-auth-logging.md#p2--authorization)

---

### 20. Why use `IHttpClientFactory` instead of `new HttpClient()`?

**Definition:** **`IHttpClientFactory`** creates and pools `HttpClient` instances, managing the underlying connection handlers for you.

**Detail:** Creating a new `HttpClient` per call exhausts sockets, because connections sit in `TIME_WAIT` after disposal. But a single static instance never notices DNS changes. The factory solves both by rotating handlers on a timer, and it gives you one clean place to attach Polly retry and circuit-breaker policies.

:material-file-document-outline: **Deep dive:** [QB — HttpClient](../Dotnet/dotnet-performance-resilience.md#qb--httpclient) · [QC — Resilience](../Dotnet/dotnet-performance-resilience.md#qc--resilience)

---

## Runners-Up (ask-me-next round)

- **Caching: in-memory vs distributed, and invalidation** — [QA](../Dotnet/dotnet-performance-resilience.md#qa--caching)
- **Throughput & performance tuning** — [QD](../Dotnet/dotnet-performance-resilience.md#qd--throughput)
- **`DbContext` lifetime & pooling** — [O8](../Dotnet/dotnet-ef-core.md#o8--dbcontext-lifetime--pooling)
- **Migrations & seeding** — [O5](../Dotnet/dotnet-ef-core.md#o5--migrations)
- **LINQ→SQL translation & client-side evaluation** — [O2](../Dotnet/dotnet-ef-core.md#o2--linqsql-translation--iqueryable-deferred-execution)
- **Relationships & Fluent API configuration** — [O6](../Dotnet/dotnet-ef-core.md#o6--relationships--fluent-configuration)
- **Controllers vs Minimal APIs** — [N1](../Dotnet/dotnet-web-apis.md#n1--controllers-vs-minimal-apis)
- **Routing: conventional vs attribute** — [N2](../Dotnet/dotnet-web-apis.md#n2--routing)
- **Content negotiation & JSON options** — [N6](../Dotnet/dotnet-web-apis.md#n6--content-negotiation--json)
- **API versioning; OpenAPI/Swagger** — [N7](../Dotnet/dotnet-web-apis.md#n7--api-versioning) · [N8](../Dotnet/dotnet-web-apis.md#n8--openapi--swagger)
- **Program.cs & minimal hosting; environments** — [M3](../Dotnet/dotnet-aspnetcore-fundamentals.md#m3--programcs--minimal-hosting--webapplicationbuilder) · [M5](../Dotnet/dotnet-aspnetcore-fundamentals.md#m5--environments)
- **Identity in the pipeline; CORS; rate limiting** — [P1](../Dotnet/dotnet-auth-logging.md#p1--identity-in-the-pipeline) · [P5](../Dotnet/dotnet-auth-logging.md#p5--cors) · [P6](../Dotnet/dotnet-auth-logging.md#p6--middleware--rate-limiting)
- **Structured logging & log levels** — [P4](../Dotnet/dotnet-auth-logging.md#p4--logging)
- **SQL injection, XSS & CSRF; secrets & password hashing** — [S3](../Dotnet/dotnet-security.md#s3--injection) · [S4](../Dotnet/dotnet-security.md#s4--csrf) · [S6](../Dotnet/dotnet-security.md#s6--secrets--passwords)
- **HTTPS, cookies & security headers** — [S5](../Dotnet/dotnet-security.md#s5--https-cookies--headers)
- **Integration testing with `WebApplicationFactory`** — [R1](../Dotnet/dotnet-testing-ops.md#r1--integration-testing)
- **Observability; health checks; Docker & deployment** — [R2](../Dotnet/dotnet-testing-ops.md#r2--observability) · [R4](../Dotnet/dotnet-testing-ops.md#r4--health-checks--observability) · [R3](../Dotnet/dotnet-testing-ops.md#r3--deployment)
- **MVC pattern, Razor & passing data to views** — [V1](../Dotnet/dotnet-mvc-razor.md#v1--the-mvc-pattern) · [V2](../Dotnet/dotnet-mvc-razor.md#v2--razor-syntax) · [V3](../Dotnet/dotnet-mvc-razor.md#v3--passing-data-to-views)
- **Legacy .NET Framework & EF 6** — [Legacy notes](../Dotnet/dotnet-legacy-framework-ef.md)
