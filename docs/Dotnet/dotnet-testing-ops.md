# R. Testing, Observability & Deployment
---

> Unit-testing basics — AAA, mocking, test doubles, faking the clock — are in [../C-Sharp/csharp-practices.md](../C-Sharp/csharp-practices.md). Configuration and hosting are in [dotnet-aspnetcore-fundamentals.md](dotnet-aspnetcore-fundamentals.md). This file covers what's specific to testing and running an ASP.NET Core app.

## R1 — Integration Testing

### Q1. What is `WebApplicationFactory`, and why use it?

**Answer.** `WebApplicationFactory<Program>` starts your **whole app in-process**, using the same `Program` path production uses, but hosted on an in-memory test server instead of Kestrel. It hands you an `HttpClient` whose requests never touch the network — they go straight into the real request pipeline.

```csharp
public class ProductsApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public ProductsApiTests(WebApplicationFactory<Program> factory)
        => _client = factory.CreateClient();

    [Fact]
    public async Task Get_unknown_product_returns_404()
    {
        var response = await _client.GetAsync("/products/9999");
        Assert.Equal(HttpStatusCode.NotFound, response.StatusCode);
    }
}
```

Because the app is real, this exercises **routing, middleware order, model binding, filters, auth, and DI wiring** — all the things a unit test mocks away. It catches a route that no longer matches, a middleware you short-circuited, an `[Authorize]` you forgot, a JSON casing change.

It sits between a unit test (fast but can't see the framework) and hitting a deployed server (slow, flaky, needs infrastructure).

⚠️ **One setup gotcha:** with top-level statements the generated `Program` class is internal, so the test project can't see it. Add a marker at the bottom of `Program.cs`:

```csharp
public partial class Program { }   // makes Program visible to tests
```

That's the pattern Microsoft's own templates use.

---

### Q2. How do you replace real dependencies in an integration test?

**Answer.** Override the services **after** the app has registered its own, so yours win. The hook is `ConfigureTestServices`:

```csharp
public class TestAppFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureTestServices(services =>
        {
            services.RemoveAll<IEmailSender>();                       // drop the real one
            services.AddSingleton<IEmailSender, FakeEmailSender>();   // add the fake
        });
    }
}
```

❌ **Use `ConfigureTestServices`, not `ConfigureServices`.** Only the former runs *after* `Program`'s registrations, so it's the only one where your override actually wins.

Two details:

- **`RemoveAll<T>()` first.** Otherwise you have two registrations, and code that injects `IEnumerable<IEmailSender>` gets both.
- **For outbound HTTP**, replace the `HttpMessageHandler` rather than mocking `HttpClient` — `HttpClient` isn't designed to be mocked.

For a one-off variation, `factory.WithWebHostBuilder(b => b.ConfigureTestServices(...))` customises a single test instead of the whole class.

---

### Q3. What database should integration tests use?

**Answer.** ❌ **Not the EF Core InMemory provider.** It isn't a relational database: it doesn't enforce foreign keys or unique constraints, has no real transactions, and translates LINQ differently. Tests pass against it and the same code fails in production — a `GroupBy` or string function with no SQL translation throws only against a real provider.

Better options, in order of fidelity:

| Option | Fidelity | Trade-off |
|---|---|---|
| **Real DB in a container** (Testcontainers) | Highest — your actual engine | Needs Docker, slower |
| **SQLite in-memory** | Good — a real relational engine | Different dialect from your production DB |
| **EF InMemory** | Poor | Only for code that just needs *a* `DbContext` |

**Testcontainers** starts a real database in Docker from your test code, waits until it's healthy, and gives you the connection string:

```csharp
public class DbFixture : IAsyncLifetime
{
    private readonly PostgreSqlContainer _db = new PostgreSqlBuilder()
        .WithImage("postgres:16-alpine")
        .Build();

    public string ConnectionString => _db.GetConnectionString();

    public Task InitializeAsync() => _db.StartAsync();
    public Task DisposeAsync() => _db.DisposeAsync().AsTask();
}
```

Each run gets a clean instance on a random port, so tests are reproducible. Containers are slow to start, so share one per test class and apply migrations once.

**The rule: test against the same database engine you ship on.**

---

### Q4. How do you test endpoints that need authentication?

**Answer.** Register a **fake authentication handler** that builds whatever identity the test needs. Your real authorization logic then runs against a controlled user — you're testing your `[Authorize]` rules, not the token plumbing.

```csharp
public class TestAuthHandler(IOptionsMonitor<AuthenticationSchemeOptions> o, ILoggerFactory l,
                            UrlEncoder e) : AuthenticationHandler<AuthenticationSchemeOptions>(o, l, e)
{
    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        var claims = new[] { new Claim(ClaimTypes.Name, "test"), new Claim(ClaimTypes.Role, "Admin") };
        var principal = new ClaimsPrincipal(new ClaimsIdentity(claims, "Test"));

        return Task.FromResult(AuthenticateResult.Success(
            new AuthenticationTicket(principal, "Test")));
    }
}
```

```csharp
builder.ConfigureTestServices(services =>
{
    services.AddAuthentication("Test")
            .AddScheme<AuthenticationSchemeOptions, TestAuthHandler>("Test", null);
});
```

Now every request is authenticated as an Admin. Vary the claims per test to check that a non-admin gets 403.

**The alternative** is scraping a real token from a test identity provider — slower and more brittle, and it tests someone else's code.

---

### Q5. How do you keep integration tests isolated when they share a database?

**Answer.** The problem: tests that write data see each other's leftovers, so they pass alone and fail together — or pass in one order and fail in another.

Three approaches, best first:

- **Transaction per test.** Begin a transaction, run the test, roll it back. Fast and completely clean, but it doesn't work if the code under test manages its own transactions.
- **Reset between tests.** Truncate the tables, or use a tool like Respawn to wipe data while keeping the schema. Reliable and works everywhere.
- **Unique data per test.** Give each test its own tenant or a GUID-based key so they can't collide. Needs discipline, but allows parallel runs.

❌ **What doesn't work is relying on test order.** xUnit runs test classes in parallel by default, so anything sharing mutable state will fail intermittently — the worst kind of failure, because a re-run often passes.

Also note **`WebApplicationFactory` is expensive to start**, so share it with `IClassFixture` rather than building one per test.

---

## R2 — Observability

### Q6. What are the three pillars of observability?

**Answer.** Three kinds of telemetry that answer different questions:

| | What it is | Answers |
|---|---|---|
| **Logs** | Timestamped records of events | *What happened in this one request?* |
| **Metrics** | Numbers aggregated over time | *How is the system doing overall?* |
| **Traces** | The path of one request across services | *Where did this request spend its time?* |

They're complementary. A **metric** tells you error rates jumped at 14:00. A **trace** shows the slow call is to the payments service. A **log** tells you exactly which record failed and why.

The practical difference is cost: metrics are cheap and aggregated, so you keep them for everything. Logs are expensive per event, so you sample or filter. Traces are per-request and usually sampled.

---

### Q7. How do you emit custom metrics and traces in .NET?

**Answer.** **Metrics** use `Meter` and instruments. Create the meter once, then record values:

```csharp
public class OrderMetrics
{
    private readonly Counter<long> _created;
    private readonly Histogram<double> _duration;

    public OrderMetrics(IMeterFactory factory)
    {
        var meter = factory.Create("MyApp.Orders");
        _created = meter.CreateCounter<long>("orders.created");
        _duration = meter.CreateHistogram<double>("orders.duration", unit: "ms");
    }

    public void OrderCreated(string channel)
        => _created.Add(1, new KeyValuePair<string, object?>("channel", channel));
}
```

Pick the instrument by what you're measuring: **Counter** for things that only go up (orders, errors), **Histogram** for distributions (durations, sizes), **Gauge** for a current value (queue depth).

⚠️ **Keep tag values low-cardinality.** A tag like `channel` has a handful of values — fine. Tagging with a user ID or an order ID creates a separate time series per value and will overwhelm your metrics backend.

**Traces** use `ActivitySource`. An `Activity` is one span of work, and .NET propagates the trace context across HTTP calls automatically:

```csharp
private static readonly ActivitySource Source = new("MyApp.Orders");

public async Task<Order> CreateAsync(CreateOrder dto)
{
    using var activity = Source.StartActivity("CreateOrder");
    activity?.SetTag("order.channel", dto.Channel);

    var order = await _repo.SaveAsync(dto);

    activity?.SetTag("order.id", order.Id);
    return order;
}
```

The `using` matters — disposing the activity is what records the duration.

---

### Q8. What is OpenTelemetry, and why standardize on it?

**Answer.** **OpenTelemetry (OTel)** is a vendor-neutral standard for collecting logs, metrics, and traces. You instrument your code once against its API, then export to whatever backend you use.

```csharp
builder.Services.AddOpenTelemetry()
    .WithMetrics(m => m
        .AddAspNetCoreInstrumentation()      // request rates and durations, free
        .AddMeter("MyApp.Orders")            // your own meters
        .AddOtlpExporter())
    .WithTracing(t => t
        .AddAspNetCoreInstrumentation()      // incoming requests
        .AddHttpClientInstrumentation()      // outgoing calls
        .AddSource("MyApp.Orders")
        .AddOtlpExporter());
```

**Why it matters:** without it, switching from one observability vendor to another means re-instrumenting your entire codebase. With OTel you change the exporter. That's real leverage, because observability vendors are expensive and you will renegotiate.

The built-in instrumentation is the best value — `AddAspNetCoreInstrumentation` gives you request rates, durations, and status codes for every endpoint with no code changes.

**Tying logs to traces:** OTel attaches the current trace ID to log records automatically, so your backend can jump from a slow trace straight to the logs for that exact request. That link is what makes debugging a distributed system tractable — without it you're grepping timestamps across services.

---

### Q9. How does HTTP Logging work in ASP.NET Core (`AddHttpLogging` middleware), and how do you configure request/response body logging, headers, and security redactions?

**Answer.** ASP.NET Core provides built-in HTTP logging via `AddHttpLogging()` service registration and `UseHttpLogging()` middleware. It logs HTTP request and response information (headers, path, duration, status codes, and optional request/response bodies) for debugging, auditing, and diagnostics.

#### 1. Service Registration & Middleware Configuration:

```csharp
var builder = WebApplication.CreateBuilder(args);

// Register HTTP Logging service with options
builder.Services.AddHttpLogging(options =>
{
    // Specify which fields to log (RequestPath, RequestHeaders, ResponseStatusCode, ResponseBody, etc.)
    options.LoggingFields = HttpLoggingFields.RequestPath
                          | HttpLoggingFields.RequestHeaders
                          | HttpLoggingFields.ResponseStatusCode
                          | HttpLoggingFields.ResponseBody;

    // Set body limits (in bytes) to prevent logging massive payloads into memory
    options.RequestBodyLogLimit = 4096;
    options.ResponseBodyLogLimit = 4096;

    // Explicitly add allowed headers to log
    options.RequestHeaders.Add("X-Correlation-ID");
    options.RequestHeaders.Add("User-Agent");
    
    // Redact sensitive headers (logged as "[Redacted]")
    options.RequestHeaders.Add("Authorization");
});

var app = builder.Build();

// ⚠️ Place UseHttpLogging early in the pipeline (after UseRouting, before UseEndpoints/Controllers)
app.UseHttpLogging();

app.MapControllers();
app.Run();
```

#### 2. Key Considerations & Production Best Practices:
- **Performance Overhead**: Logging request and response bodies buffers payload streams into memory. In high-throughput production services, limit `LoggingFields` to headers/path metadata, or enable body logging selectively.
- **Security & PII Redaction**: Never log unredacted sensitive headers (`Authorization`, `Cookie`, `X-Api-Key`) or bodies containing passwords or PII.
- **Endpoint Customization**: Use the `[HttpLogging]` attribute or `.WithHttpLogging()` endpoint metadata to enable or disable HTTP logging per controller or Minimal API route.

---

## R3 — Deployment

### Q10. What does `dotnet publish` produce?

**Answer.** A folder ready to deploy. The main choice is whether the .NET runtime ships with it.

| | Framework-dependent | Self-contained |
|---|---|---|
| Runtime | Must be installed on the machine | Included in the output |
| Size | Small (a few MB) | Large (60 MB+) |
| Updates | Patch the runtime separately | Rebuild and redeploy to patch |

```bash
dotnet publish -c Release                                      # framework-dependent
dotnet publish -c Release -r linux-x64 --self-contained true    # self-contained
```

**Framework-dependent is the normal choice**, especially in containers, where the base image supplies the runtime. Self-contained suits a machine you don't control or where you can't install anything.

Three further options, each with a real caveat:

- **Single-file** — bundles everything into one executable. Convenient; it just extracts at startup, so it isn't smaller.
- **Trimming** — removes unused framework code to shrink the output. ⚠️ It can break **reflection**, because the trimmer can't see a type only referenced by name. Test the trimmed build properly.
- **Native AOT** — compiles ahead of time to a native binary. Very fast startup and low memory, which suits serverless. ⚠️ No runtime code generation, so reflection-heavy libraries won't work.

---

### Q11. Walk through a production Dockerfile for an ASP.NET Core app.

**Answer.** The key idea is a **multi-stage build**: compile in an image that has the SDK, then copy only the output into a small runtime image. The SDK is around 800 MB and you don't want it in production.

```dockerfile
# ---- build stage ----
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src

COPY *.csproj ./                 # copy csproj first — see why below
RUN dotnet restore

COPY . .                         # then the rest of the source
RUN dotnet publish -c Release -o /app --no-restore

# ---- runtime stage ----
FROM mcr.microsoft.com/dotnet/aspnet:9.0
WORKDIR /app
COPY --from=build /app .

USER $APP_UID                    # don't run as root
EXPOSE 8080
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

The details that matter:

- **Copy `.csproj` and restore before copying the source.** Docker caches layers, so as long as your dependencies haven't changed, a code edit reuses the cached restore. Copy everything first and you re-download packages on every build.
- **`aspnet` runtime image, not `sdk`** — a fraction of the size and a much smaller attack surface.
- **Don't run as root.** `USER $APP_UID` uses the non-root user the Microsoft images provide.
- **Add a `.dockerignore`** for `bin/`, `obj/`, and `.git` — otherwise you copy local build output into the image and can invalidate the cache constantly.

**Port configuration:** since .NET 8 the default container port is **8080**, not 80, because the non-root user can't bind low ports. Override with `ASPNETCORE_HTTP_PORTS` or `ASPNETCORE_URLS`.

**What makes it orchestrator-ready:**

```csharp
app.MapHealthChecks("/health/live");    // is the process up?
app.MapHealthChecks("/health/ready");   // can it serve traffic — DB reachable?
```

Kubernetes needs both, and they differ: failing **liveness** restarts the pod, failing **readiness** just takes it out of the load balancer. Wire a dependency check to readiness, not liveness, or a brief database blip will restart every pod you have.

Also honour **graceful shutdown** — on `SIGTERM` the host stops taking new requests and drains in-flight ones. Give Kubernetes a grace period slightly longer than your drain timeout, or it kills requests mid-flight.

---

## R4 — Health Checks & Observability

### Q12. How do Health Checks work in ASP.NET Core (`Liveness` vs `Readiness` probes for Kubernetes), and how do custom `IHealthCheck` implementations check DB and Redis availability?


**Answer.** ASP.NET Core provides an **infra-ready Health Check framework** (`Microsoft.Extensions.Diagnostics.HealthChecks`). In containerized environments (Kubernetes, AWS ECS), health checks allow orchestrators to monitor process status and route traffic intelligently.

#### Liveness vs Readiness Probes

| Health Check Probe | Kubernetes Purpose | Failure Action | What it should test |
|---|---|---|---|
| **Liveness Probe** (`/health/live`) | Is the app process alive and un-deadlocked? | **Restarts the container** | Basic in-process check (returns 200 OK if server responds). **Do NOT check DB here!** |
| **Readiness Probe** (`/health/ready`) | Can the app process serve HTTP traffic? | **Removes pod from Load Balancer** | Downstream dependencies: SQL Database, Redis cache, RabbitMQ connection. |

#### Custom `IHealthCheck` Implementation

```csharp
public class RedisHealthCheck(IConnectionMultiplexer redis) : IHealthCheck
{
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            var db = redis.GetDatabase();
            var ping = await db.PingAsync();

            return ping < TimeSpan.FromSeconds(2)
                ? HealthCheckResult.Healthy($"Redis ping fast: {ping.TotalMilliseconds}ms")
                : HealthCheckResult.Degraded($"Redis ping slow: {ping.TotalMilliseconds}ms");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("Redis cluster unreachable", ex);
        }
    }
}

// Registration in Program.cs:
builder.Services.AddHealthChecks()
    .AddCheck<RedisHealthCheck>("redis", tags: ["ready"])
    .AddNpgSql(builder.Configuration.GetConnectionString("Default")!, tags: ["ready"]);

// Endpoint mapping:
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false // Liveness: basic process check only (no tags)
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready") // Readiness: check DB & Redis
});
```

❌ **The Classic Production Disaster**: Putting database connectivity checks inside the **Liveness** probe. If SQL Server experiences a 10-second failover or network blip, Kubernetes marks every pod as un-live and restarts all application containers simultaneously, escalating a minor DB blip into a total cluster crash-loop.

