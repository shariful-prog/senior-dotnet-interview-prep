# P. Auth, Errors & Logging
---

> JWTs, OAuth2, and token storage are in [dotnet-security.md](dotnet-security.md). `ProblemDetails` as an error format is in [dotnet-web-apis.md](dotnet-web-apis.md). This file covers how ASP.NET Core wires these up.

## P1 — Identity in the Pipeline

### Q1. What is `HttpContext.User`, and what are claims?

**Answer.** `HttpContext.User` is a **`ClaimsPrincipal`** — the identity of whoever is making the current request. Authentication middleware builds it; everything downstream reads it.

It contains information about the user, such as:

- User ID
- Username
- Email
- Roles
- Permissions

Example:

```csharp
var userId = User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
var email = User.FindFirst(ClaimTypes.Email)?.Value;
var role = User.FindFirst(ClaimTypes.Role)?.Value;
```

#### What are Claims?

A **claim** is a piece of information about a user stored as a **key-value pair**.

Examples:

| Claim Type | Value |
|------------|-------|
| NameIdentifier | 123 |
| Name | John |
| Email | john@example.com |
| Role | Admin |

Think of claims as the user's identity information.

#### How is `HttpContext.User` set?

When a request arrives:

```text
Client
   │
   │  JWT Token / Cookie
   ▼
Authentication Middleware
   │
   ▼
Validate Token
   │
   ▼
Create ClaimsPrincipal
   │
   ▼
HttpContext.User
   │
   ▼
Controller
```

For example, if the request contains a valid JWT token:

```text
Authorization: Bearer eyJhbGciOi...
```

The JWT Authentication Middleware:

1. Validates the token.
2. Reads the claims from the token.
3. Creates a `ClaimsPrincipal`.
4. Assigns it to `HttpContext.User`.

So in your controller:

```csharp
var name = User.Identity?.Name;
```

or

```csharp
var userId = User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
```

you're reading values that came from the authenticated user's token (or cookie).

---

### Q2. What is an authentication scheme, and what's the difference between Challenge and Forbid?

**Answer.** A **scheme** is a named way of authenticating — "Cookies", "Bearer", "Google". You register each with a name and a handler, and one is the default:

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer()          // scheme name: "Bearer"
    .AddCookie();            // scheme name: "Cookies"
```

Naming them lets one app accept several kinds of credential — a cookie for the MVC pages, a bearer token for the API.

**Challenge vs Forbid** is the 401-vs-403 distinction, and it's a common interview question:

| | Meaning | Status | Typical response |
|---|---|---|---|
| **Challenge** | "I don't know who you are" | **401** | Redirect to login (cookies), or `WWW-Authenticate` (bearer) |
| **Forbid** | "I know who you are, and no" | **403** | An error page or plain 403 |

```csharp
return Challenge();   // not authenticated → prove who you are
return Forbid();      // authenticated, but not allowed
```

The framework picks the right one automatically: an anonymous request to an `[Authorize]` endpoint gets a challenge, while an authenticated user who fails a policy gets a forbid.

❌ Returning 401 for an authorization failure is a common bug — it tells the client to log in again, which won't help, since they're already logged in.

---

## P2 — Authorization

### Q3. What do `[Authorize]` and `[AllowAnonymous]` do?

**Answer.** `[Authorize]` restricts access to **authenticated users only**. If a user is not authenticated, ASP.NET Core blocks the request.

`[AllowAnonymous]` bypasses authorization checks. It allows everyone to access the endpoint, even if the controller has `[Authorize]`.

```csharp
[Authorize]                          // whole controller requires auth
public class AccountController : ControllerBase
{
    [AllowAnonymous]                 // ...except this one
    [HttpPost("login")]
    public IActionResult Login(LoginDto dto) => /* ... */;
}
```

**The safer posture is a fallback policy** — require authentication everywhere, so a new endpoint is protected unless someone deliberately opens it:

```csharp
builder.Services.AddAuthorization(o =>
    o.FallbackPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build());
```

That's **fail-closed**. Without it, forgetting `[Authorize]` on a new controller silently exposes it — and nothing in a code review makes an *absent* attribute obvious.

---

### Q4. Compare role-based, claims-based, and policy-based authorization.

**Answer.** Three styles, increasing in flexibility. All three end up reading the same `ClaimsPrincipal` — they differ in *what* they check and *where the rule lives*.

**1. Role-based** — checks whether the user holds a named role. The rule lives in the attribute:

```csharp
[Authorize(Roles = "Admin")]                    // "is the user an Admin?"
public IActionResult DeleteUser() => Ok();

[Authorize(Roles = "Admin,Manager")]            // comma = OR (either role passes)
public IActionResult Reports() => Ok();
```

A role is really just a claim of type `role` in the token:

```json
{ "name": "John", "role": "Admin" }
```

Good for simple Admin/Manager/Customer separation. It breaks down as rules grow: you end up with roles like `AdminExceptBilling`, and every rule change means editing attributes and redeploying.

**2. Claims-based** — checks a specific fact about the user rather than a job title. A claim is a key-value pair issued in the token:

```json
{ "permission": "delete_users", "department": "Finance", "country": "SE" }
```

Claims are more granular than roles, so you can grant one capability without inventing a role for it. In ASP.NET Core there is no `[Authorize(Claims = ...)]` attribute — you either check inline, or wrap the claim in a policy (which is why this style blends into the next one):

```csharp
// inline check
if (!User.HasClaim("permission", "delete_users")) return Forbid();

// or declare it once as a policy
options.AddPolicy("CanDelete", p => p.RequireClaim("permission", "delete_users"));
```

**3. Policy-based** — the recommended approach. A policy is a **named rule registered in one place** and referenced by name at the endpoint:

```csharp
// Program.cs — the rule lives here
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("CanApproveInvoice", policy =>
        policy.RequireClaim("permission", "ApproveInvoice"));

    options.AddPolicy("SeniorFinance", policy =>        // combine requirements: AND
        policy.RequireClaim("department", "Finance")
              .RequireRole("Manager"));
});
```

```csharp
[Authorize(Policy = "CanApproveInvoice")]
public IActionResult ApproveInvoice() => Ok();
```

A policy can require claims, roles, or arbitrary logic through a custom `AuthorizationRequirement` and handler — age checks, resource ownership, or a database lookup.

**Why policies win.** The rule is defined once and named, so changing it means editing one line in `Program.cs` instead of hunting through attributes. The controller states the *intent* (`CanApproveInvoice`), not the mechanism, and the same policy can be reused across endpoints and unit tested on its own. Roles and claims still do the work underneath — policies are the layer that keeps them maintainable.

---

### Q5. What is resource-based authorization?

**Answer.** `[Authorize]` runs **before** the action, so it only sees the *user* and the *endpoint*. It cannot express *"you may edit this document only if you own it"*, because the document hasn't been loaded yet.

Resource-based authorization runs the check **after** you load the data, using `IAuthorizationService`:

```csharp
public class DocumentAuthHandler
    : AuthorizationHandler<OperationAuthorizationRequirement, Document>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext ctx,
        OperationAuthorizationRequirement op, Document doc)
    {
        if (op.Name == "Edit" && doc.OwnerId == ctx.User.FindFirstValue(ClaimTypes.NameIdentifier))
            ctx.Succeed(op);                 // pass

        return Task.CompletedTask;           // don't fail explicitly — see below
    }
}
```

```csharp
var doc = await _repo.GetAsync(id);
var result = await _authz.AuthorizeAsync(User, doc, Operations.Edit);
if (!result.Succeeded) return Forbid();      // 403
```

⚠️ Note you call `Succeed` but **never fail explicitly**. You just don't succeed. That's deliberate: several handlers can offer different ways to satisfy the same requirement (an owner *or* an admin), so one saying "no" mustn't veto another saying "yes".

This is the pattern for per-record ownership and multi-tenant isolation — the same problem as IDOR in [dotnet-security.md](dotnet-security.md).

---

## P3 — Exception Handling

### Q6. How do you handle exceptions globally?

**Answer.** Centrally, in one place, for two reasons: **don't leak internals** (a stack trace tells an attacker your framework, versions, and file paths) and **stay consistent** (every error looks the same to clients).

**The modern approach is `IExceptionHandler`** (.NET 8+) — a class per concern, registered in DI:

```csharp
public class NotFoundExceptionHandler : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext ctx, Exception ex, CancellationToken ct)
    {
        if (ex is not EntityNotFoundException) return false;   // not mine — pass along

        ctx.Response.StatusCode = StatusCodes.Status404NotFound;
        await ctx.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Title = "Resource not found",
            Status = 404
        }, ct);

        return true;                                            // handled
    }
}
```

```csharp
builder.Services.AddExceptionHandler<NotFoundExceptionHandler>();   // order matters
builder.Services.AddExceptionHandler<FallbackExceptionHandler>();
builder.Services.AddProblemDetails();

var app = builder.Build();
app.UseExceptionHandler();
```

Handlers run **in registration order** until one returns `true`. Returning `false` means "not mine" — so put specific handlers first and a catch-all last.

**Two things to get right:**

- ❌ **Put `UseExceptionHandler` first in the pipeline.** It only catches exceptions from middleware *below* it. Registered last, it catches nothing.
- **Log the real exception, return a generic message.** The caller gets "An error occurred" plus a trace ID; your logs get the details. Then a support ticket quoting that ID leads you to the exact failure.

**How this differs from an MVC exception filter:** a filter only catches exceptions from **inside the MVC pipeline** — actions and other filters. It never sees a failure in routing, in earlier middleware, or in a Minimal API. Use a filter only when you need MVC context; otherwise handle it globally.

---

## P4 — Logging

### Q6b. How does logging work in .NET by default, and where do the logs go?

**Answer.** Logging is built in. `WebApplication.CreateBuilder(args)` already registers the logging system, so you inject `ILogger<T>` and write — no setup, no NuGet package:

```csharp
public class OrderService
{
    private readonly ILogger<OrderService> _logger;

    public OrderService(ILogger<OrderService> logger) => _logger = logger;

    public void Ship(int id) => _logger.LogInformation("Order {OrderId} shipped", id);
}
```

The generic parameter matters: `ILogger<OrderService>` sets the entry's **category** to the full type name (`MyApp.Services.OrderService`). That is the string you filter on in configuration.

**Where it goes.** `ILogger` writes nothing itself. It hands the entry to every registered **provider**, and each provider decides the destination. The default template registers four:

| Provider | Destination |
|---|---|
| Console | stdout — the terminal, or `docker logs` |
| Debug | the debugger output window in Visual Studio |
| EventSource | ETW / `dotnet-trace` |
| EventLog | Windows Event Log (Windows only) |

Note what is missing: **there is no file provider.** Out of the box .NET does not write logs to disk at all. Console output vanishes when the process stops. This is deliberate — in containers and on cloud hosts, stdout is collected by the platform (Docker, Kubernetes, Azure App Service). If you want files, you add a provider that writes them (Q6d).

A console entry looks like this — level, category, event ID, then the message:

```text
info: MyApp.Services.OrderService[0]
      Order 42 shipped
```

**Controlling what is written.** Every entry has a **category** (the `ILogger<T>` type name) and a **level**. Before any provider sees it, the logging system checks the configured minimum level for that category and drops the entry if it does not reach it. That minimum lives in `appsettings.json`, so you change what gets logged by editing config — no code change, no rebuild:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "MyApp.Payments": "Debug"
    }
  }
}
```

**How a category is matched.** The keys are treated as **prefixes**, and the longest matching one wins. There is no wildcard syntax — `Microsoft.AspNetCore` already covers every category beneath it. `Default` is the fallback when nothing else matches.

Working through the config above:

| Category writing the entry | Matched by | Minimum level | `LogDebug` | `LogInformation` | `LogWarning` |
|---|---|---|---|---|---|
| `MyApp.Payments.StripeClient` | `MyApp.Payments` | `Debug` | written | written | written |
| `MyApp.Services.OrderService` | `Default` | `Information` | dropped | written | written |
| `Microsoft.AspNetCore.Mvc.Something` | `Microsoft.AspNetCore` | `Warning` | dropped | dropped | written |

So the payments code is turned up loud for debugging, framework noise is held down to real problems, and everything else sits in the middle. That third row is why you stop seeing the per-request `Information` lines from ASP.NET Core once you set it to `Warning`.

**Where the value comes from.** `LogLevel` is ordinary configuration, so the usual precedence applies — later sources override earlier ones:

```text
appsettings.json  →  appsettings.{Environment}.json  →  environment variables
```

`appsettings.Development.json` normally sets `Microsoft.AspNetCore` to `Information`, which is why you see far more output locally than in production. On a running server you can raise a level through an environment variable instead of touching files:

```text
Logging__LogLevel__MyApp__Payments=Trace
```

Double underscore is the separator, because a colon is not legal in environment variable names on all platforms.

**Why filter rather than delete the calls.** A dropped entry costs almost nothing — the level check happens before the message template is formatted (Q7), so an argument is never converted to a string. That is what makes it safe to leave `LogDebug` calls permanently in your code and switch them on only when you are chasing something.

---

### Q6c. What are logging providers, and how do you add or replace them?

**Answer.** A **provider** is a sink — one destination for log entries. `ILogger` is the API you write to; providers are what decide where the text ends up. Every registered provider receives every entry that passes the level filter, so one `LogInformation` call can reach console, a file, and Application Insights at once.

You configure them on `builder.Logging`:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Logging.ClearProviders();        // drop the four defaults
builder.Logging.AddConsole();            // add back only what you want
builder.Logging.AddDebug();
```

`ClearProviders()` is worth knowing. Without it you are **adding** to the defaults, not replacing them, so a common mistake is wiring up Serilog and still getting duplicate console output from the built-in provider.

Common providers beyond the built-in four:

| Provider | Use |
|---|---|
| `AddApplicationInsights()` | Azure telemetry, with traces and metrics |
| `AddOpenTelemetry()` | vendor-neutral export to any OTLP backend |
| `AddSeq()` / `AddSerilog()` | structured log servers |
| `AddAzureWebAppDiagnostics()` | App Service log stream and blob storage |

For the console provider specifically, JSON output is usually what you want in production, since log collectors parse fields rather than scraping text:

```csharp
builder.Logging.AddJsonConsole();   // {"Timestamp":"...","Level":"Information","OrderId":42,...}
```

That single switch is what turns your structured templates (Q7) into queryable fields without adding any third-party library.

---

### Q6d. Why use Serilog or NLog instead of the built-in logger?

**Answer.** The built-in system is a good API with a thin set of destinations. Third-party libraries plug into the same `ILogger<T>` abstraction and supply what the built-in one lacks: **file output with rolling, richer sinks, and enrichment**.

Your application code does not change. You still inject `ILogger<T>` and write templates — only `Program.cs` differs:

```csharp
// Serilog — configure the pipeline, then hand logging over to it
builder.Host.UseSerilog((ctx, cfg) => cfg
    .ReadFrom.Configuration(ctx.Configuration)
    .Enrich.FromLogContext()
    .Enrich.WithMachineName()                     // adds fields to every entry
    .WriteTo.Console()
    .WriteTo.File("logs/app-.log",
        rollingInterval: RollingInterval.Day,     // one file per day
        retainedFileCountLimit: 14)               // keep two weeks, delete older
    .WriteTo.Seq("http://localhost:5341"));
```

`UseSerilog` replaces the provider set, so you do not need `ClearProviders()` here.

What you get over the defaults:

- **Files that manage themselves** — roll by day or size, cap the retained count, so a long-running service does not fill the disk.
- **Many sinks** — files, Seq, Elasticsearch, Datadog, Splunk, email, databases. Well over a hundred exist.
- **Enrichers** — attach machine name, environment, thread ID, or correlation ID to every entry automatically instead of repeating them per call.
- **Sink-specific levels** — everything to file, only `Warning` and above to the paid service you are billed by volume for.

**When the built-in one is enough:** containerised services where the platform already collects stdout, or anything already sending telemetry to Application Insights or an OTLP collector. Adding Serilog to write files inside a container is usually a step backwards — the file disappears with the container.

Because both sit behind `ILogger<T>`, swapping later touches only startup. That is the point of the abstraction.

---

### Q7. What is structured logging, and why not string interpolation?

**Answer.** These two lines print identical text to the console:

```csharp
// ❌ interpolated — you build the string, the logger gets one finished sentence
_logger.LogInformation($"Order {orderId} shipped to {country}");

// ✅ template — you pass the sentence and the values separately
_logger.LogInformation("Order {OrderId} shipped to {Country}", orderId, country);
```

The difference is what gets **stored**. The first version hands over a finished string, so that is all there is:

```text
"Order 42 shipped to SE"
```

The second keeps the pieces apart, so the entry arrives as data with named fields:

```json
{
  "Message":  "Order 42 shipped to SE",
  "Template": "Order {OrderId} shipped to {Country}",
  "OrderId":  42,
  "Country":  "SE"
}
```

That is **structured logging** — the values survive as fields instead of being melted into text.

**Why it matters.** Once `OrderId` is a field, your log system can search it like a database column: `OrderId = 42` finds every step of that one order, and "count errors grouped by Country" is a real query. With the interpolated version you only have text, so you are down to substring matching — and searching for `42` also hits order 4200 and a timestamp ending in 42.

---

### Q8. What are log scopes, and what should you never log?

**Answer.** A **scope** attaches properties to every log entry written inside a block, so you don't repeat them on every call.

```csharp
using (_logger.BeginScope("{OrderId} {UserId}", orderId, userId))
{
    _logger.LogInformation("Validating");     // both properties attached
    _logger.LogInformation("Charging card");  // both attached
    _logger.LogInformation("Shipped");        // both attached
}
```

Now you can filter your logs to one order and see every step, without passing the ID into each message. ASP.NET Core does this for you per request, which is where the automatic request ID comes from.

**The log levels**, and what each is for:

| Level | Use for |
|---|---|
| `Trace` / `Debug` | Development detail. Off in production. |
| `Information` | Normal events worth recording — started, completed |
| `Warning` | Something unexpected but handled — a retry, a fallback |
| `Error` | An operation failed and a user is affected |
| `Critical` | The app can't continue |

Pick the level by who needs to act on the entry, not by how interesting it is. Filtering is per category in configuration (Q6b), so you can turn your own code up to `Debug` without drowning in framework noise.

❌ **What you must never log:** passwords, tokens, connection strings, card numbers, personal data. Logs are shipped to systems with far broader access than your database and kept for a long time.

```csharp
_logger.LogInformation("Login {Email} {Password}", email, password);   // ❌ forever
_logger.LogInformation("Login attempt for {UserId}", user.Id);         // ✅
```

The accidental leaks are the dangerous ones: logging a whole request object, an exception carrying a connection string, or a URL with a token in the query string — which lands in every access log along the way.

---

## P5 — CORS

### Q9. What is CORS, and what is a preflight request?

**Answer.** JavaScript on `https://app.com` calls `https://api.app.com`. The request goes out, the server answers normally — and the browser **refuses to hand the response to your code**:

```text
Access to fetch at 'https://api.app.com/orders' from origin 'https://app.com'
has been blocked by CORS policy
```

**Why browsers do this.** It is the **same-origin policy**. Without it, any site you visited could quietly call your bank's API from your browser — with your cookies attached — and read your balance. So the browser blocks cross-origin reads by default.

An **origin** is scheme + host + port. All three must match, so these are all different origins from `https://app.com`:

```text
https://api.app.com     ❌ different host (a subdomain still counts)
http://app.com          ❌ different scheme
https://app.com:8080    ❌ different port
```

**CORS is how the server opts in.** The server adds response headers saying which origins are allowed; the browser reads them and decides whether to release the response. In ASP.NET Core you declare that as a policy:

```csharp
builder.Services.AddCors(o => o.AddPolicy("app", p => p
    .WithOrigins("https://app.com")          // who may call
    .AllowAnyHeader()
    .AllowMethods("GET", "POST")
    .AllowCredentials()));                   // may cookies be sent?

app.UseCors("app");        // must come before UseAuthorization
```

**The preflight.** For anything beyond a plain GET or form POST, the browser asks permission *first*, using an automatic `OPTIONS` request you never write:

```text
OPTIONS /orders                             ← browser asks first
Origin: https://app.com
Access-Control-Request-Method: DELETE
Access-Control-Request-Headers: authorization

204 No Content                              ← server approves
Access-Control-Allow-Origin: https://app.com
Access-Control-Allow-Methods: GET, POST, DELETE
Access-Control-Allow-Headers: authorization

DELETE /orders/42                           ← only now does the real request go
```

If the server does not approve, the real request is **never sent**. Your API never sees it, which is why nothing appears in your server logs when CORS is misconfigured.

What triggers a preflight: `PUT`, `PATCH`, `DELETE`, a `Content-Type` of `application/json`, or any custom header such as `Authorization`. In other words, nearly every real API call.

**Three things people get wrong:**

- ❌ **`AllowAnyOrigin()` with `AllowCredentials()` throws at startup.** It would mean "any website may send this user's cookies and read the reply" — exactly the attack the same-origin policy exists to stop. With credentials you must name the origins.
- ❌ **CORS is not security for your API.** Only browsers enforce it. curl, Postman, and any server-side HTTP client ignore the headers completely. Authentication and authorization are what protect your endpoints.
- ❌ **A CORS failure looks like a network error**, not a 403. The status code is often fine — the browser simply withholds the response. Check the browser console for the CORS message, not your server logs.

---

## P6 — Middleware & Rate Limiting

### Q10. Middleware, filters, or `DelegatingHandler` — how do you choose?

**Answer.** Three extension points, and the choice comes down to **what the code needs to see**.

| | Runs on | Sees | Use for |
|---|---|---|---|
| **Middleware** | Every inbound request | No MVC context | Logging, exceptions, CORS, headers |
| **Filter** | After an action is chosen | Action, arguments, `ModelState` | Per-action rules, validation |
| **`DelegatingHandler`** | Outbound `HttpClient` calls | The outgoing request | Auth tokens, retries, correlation IDs |

**The rule:** inbound and framework-agnostic → **middleware**. Inbound and needs MVC context → **filter** ([dotnet-web-apis.md](dotnet-web-apis.md)). Outbound → **`DelegatingHandler`** ([dotnet-performance-resilience.md](dotnet-performance-resilience.md)).

Custom middleware looks like this:

```csharp
public class CorrelationIdMiddleware(RequestDelegate next)
{
    public async Task InvokeAsync(HttpContext ctx, ILogger<CorrelationIdMiddleware> log)
    {
        var id = ctx.Request.Headers["X-Correlation-Id"].FirstOrDefault()
                 ?? Guid.NewGuid().ToString();

        ctx.Response.Headers["X-Correlation-Id"] = id;

        using (log.BeginScope("{CorrelationId}", id))
            await next(ctx);
    }
}
```

❌ **Per-request services go on `InvokeAsync`, not the constructor.** The middleware instance is created **once** for the app's lifetime, so a `DbContext` injected into the constructor is captured forever — the captive dependency bug ([dotnet-ef-core.md](dotnet-ef-core.md)).

---

### Q11. How does rate limiting work in .NET?

**Answer.** Rate limiting caps how many requests a caller can make. Without it, login endpoints can be brute-forced and one client can starve everyone else.

.NET 7+ has it built in, with four algorithms:

| Algorithm | Behaviour |
|---|---|
| **Fixed window** | N per fixed interval. Simple, but allows a burst at the boundary |
| **Sliding window** | N over a rolling interval. Smoother |
| **Token bucket** | Tokens refill steadily; allows bursts up to the bucket size |
| **Concurrency** | Caps requests *in flight* rather than per time |

```csharp
builder.Services.AddRateLimiter(o =>
{
    o.RejectionStatusCode = StatusCodes.Status429TooManyRequests;

    o.AddPolicy("per-user", ctx => RateLimitPartition.GetFixedWindowLimiter(
        partitionKey: ctx.User.FindFirstValue(ClaimTypes.NameIdentifier)
                      ?? ctx.Connection.RemoteIpAddress?.ToString()
                      ?? "anonymous",
        _ => new FixedWindowRateLimiterOptions { PermitLimit = 100, Window = TimeSpan.FromMinutes(1) }));
});

app.UseRateLimiter();
app.MapPost("/login", Login).RequireRateLimiting("per-user");
```

**Partitioning is the important part** — that `partitionKey` gives each user their own budget. Without it the limit is global, so one noisy client locks out everyone.

Reject with **429 Too Many Requests**, and add a `Retry-After` header so well-behaved clients know when to come back.

⚠️ **The built-in limiter is per-instance.** With three servers and a 100/minute limit, a user actually gets 300. For a true cluster-wide limit you need a shared store or enforcement at the gateway.
