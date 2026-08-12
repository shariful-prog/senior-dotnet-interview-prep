# M. ASP.NET Core Fundamentals
---

## M1 — Kestrel & the Hosting Model

### Q1. What is Kestrel, and why does ASP.NET Core ship its own web server?

**Answer.** **Kestrel** is the web server that comes built into ASP.NET Core. Here's the key idea: your app *is* the web server. When you run `dotnet MyApp.dll`, that one process listens for HTTP requests directly — there's no separate server product sitting in front of it.

This is a big change from the old ASP.NET, which was glued to IIS and only ran on Windows. Kestrel is fast and runs the same on Windows, Linux, and macOS.

Why does the framework ship its own server? So it isn't tied to any one server. ASP.NET Core talks to a web server through a small "plug" (`IServer`), and Kestrel is the default plug. You could swap in others (IIS's built-in server, or Windows-only `HTTP.sys`), but Kestrel is what almost everyone uses.

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.WebHost.ConfigureKestrel(options =>
{
 options.Limits.MaxConcurrentConnections = 1000;
 options.Limits.MaxRequestBodySize = 30 * 1024 * 1024; // cap uploads at 30 MB
 options.ListenAnyIP(5000);
 options.ListenAnyIP(5001, listen => listen.UseHttps());
});
var app = builder.Build();
```

Kestrel is fully production-ready and can face the internet directly (it speaks HTTP/1.1, HTTP/2, and HTTP/3). People often put another server in front of it, but that's for convenience (Q2), not because Kestrel isn't good enough.

---

### Q3. In-process vs out-of-process hosting with IIS — what's the difference?

**Answer.** 
ASP.NET Core applications can be hosted in IIS in **two ways**:

### In-Process Hosting (Default)

In **In-Process** hosting, the ASP.NET Core application runs **inside the IIS worker process (`w3wp.exe`)**.

```text
Browser
   │
   ▼
IIS (w3wp.exe)
   │
   ▼
ASP.NET Core App
```

- Only **one process** (`w3wp.exe`)
- IIS handles the HTTP requests directly.
- **No separate `dotnet.exe` or Kestrel process** is used to listen for requests.
- Faster because there is no proxy between IIS and the application.


### Out-of-Process Hosting

In **Out-of-Process** hosting, IIS does **not** run the application directly. Instead, it forwards requests to a separate **Kestrel (`dotnet.exe`)** process.

```text
Browser
   │
   ▼
IIS (w3wp.exe)
   │
   ▼
Kestrel (dotnet.exe)
   │
   ▼
ASP.NET Core App
```

- **Two processes:** `w3wp.exe` and `dotnet.exe`
- IIS acts as a **reverse proxy**.
- Kestrel receives the request and executes the application.
- Slightly slower due to the extra proxy hop.

---

### Q4. What is the Generic Host / `IHost`, and how does `WebApplication` relate to it?

**Answer.** The **Generic Host (`IHost`)** is the foundation of a .NET application. It manages the application's **dependency injection (DI), configuration, logging, and lifetime**.

`WebApplication` is a higher-level API built on top of `IHost` that simplifies creating ASP.NET Core web applications.

#### Example

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.Run();
```
Internally, `WebApplication` creates and configures an `IHost`.

---

### Q5. How does graceful shutdown work, and how do you hook into the application lifetime?

**Answer.** When your app is told to stop (Ctrl+C, a shutdown signal from Kubernetes or a service manager, or an IIS recycle), it doesn't just drop dead. It shuts down **gracefully**: it stops taking new requests, lets the ones already in progress finish (up to 30 seconds by default), and then cleans up. Like a shop that locks the door but still serves the customers already inside.

To run your own code at these moments, use **`IHostApplicationLifetime`**, which gives you three "hooks":

```csharp
public class ShutdownLogger(IHostApplicationLifetime lifetime, ILogger<ShutdownLogger> log)
 : IHostedService
{
 public Task StartAsync(CancellationToken ct)
 {
 lifetime.ApplicationStarted.Register(() => log.LogInformation("Up and serving"));
 lifetime.ApplicationStopping.Register(() => log.LogInformation("Draining..."));
 lifetime.ApplicationStopped.Register(() => log.LogInformation("Stopped"));
 return Task.CompletedTask;
 }
 public Task StopAsync(CancellationToken ct) => Task.CompletedTask;
}
```

`ApplicationStopping` fires when shutdown *begins* (still finishing in-flight work); `ApplicationStopped` fires once everything is done. You can also call `lifetime.StopApplication()` to shut down from code.

Senior tip for containers: give Kubernetes a shutdown grace period that's a bit **longer** than your app's drain timeout — otherwise it force-kills requests mid-finish. And background jobs should watch their cancellation token so they stop cleanly too (see M6).

---

## M2 — The Request Pipeline & Middleware (CORE)

### Q6. What is middleware, and how does the request pipeline actually work?

**Answer.** Every request passes through a **pipeline** — a line of small components called **middleware**, each doing one job (logging, auth, compression, etc.) before handing off to the next.

Each middleware gets the request and a reference to `next` (the next component in line). It can:

1. Do something **before** calling `next()`,
2. Call `await next(...)` to pass control along,
3. Do something **after** `next()` comes back (as the response heads out),
4. Or **not call `next()` at all**, stopping the request right there.

Picture nested Russian dolls (or the layers of an onion): the request travels *inward* through each layer to the center (your endpoint), then the response travels back *outward* through the same layers in reverse. Code before `next()` runs on the way in; code after runs on the way out.

```csharp
app.Use(async (context, next) =>
{
 var sw = Stopwatch.StartNew(); // BEFORE — on the way in
 await next(context); // hand off to the rest of the pipeline
 sw.Stop(); // AFTER — on the way back out
 logger.LogInformation("{Path} took {Ms}ms", context.Request.Path, sw.ElapsedMilliseconds);
});
```

The single most important thing to remember: **the order you register middleware is the order requests flow through it** (Q9).

---

### Q7. `app.Use` vs `app.Run` vs `app.Map`/`MapWhen` — what's the difference?

**Answer.** These methods are used to configure the **ASP.NET Core request pipeline**.

### `app.Use()`

Adds a **middleware** to the pipeline.

It can perform work **before and after** the next middleware by calling `next()`.

```csharp
app.Use(async (context, next) =>
{
    Console.WriteLine("Before");
    await next();
    Console.WriteLine("After");
});
```
### `app.Run()`

Adds the **terminal middleware**.

It handles the request and **does not call the next middleware**, ending the pipeline.

```csharp
app.Run(async context =>
{
    await context.Response.WriteAsync("Hello World!");
});
```

### `app.Map()`

Branches the pipeline based on the **request path**.

```csharp
app.Map("/admin", adminApp =>
{
    adminApp.Run(async context =>
    {
        await context.Response.WriteAsync("Admin Page");
    });
});
```
Requests to `/admin` use this branch.

### `app.MapWhen()`

Branches the pipeline based on a **custom condition**.

```csharp
app.MapWhen(
    context => context.Request.Query.ContainsKey("debug"),
    appBuilder =>
    {
        appBuilder.Run(async context =>
        {
            await context.Response.WriteAsync("Debug Mode");
        });
    });
```

---

### Q8. What is short-circuiting, and when do you use it?

**Answer.** **Short-circuiting** means a middleware answers the request itself and returns *without* calling `next()` — so the rest of the pipeline never runs. It's like a security guard turning someone away at the door before they ever reach the office.

This is exactly how auth, caching, and validation gates work. For example, an API-key check returns `401` immediately if the key is missing, so the endpoint never runs:

```csharp
app.Use(async (ctx, next) =>
{
 if (!ctx.Request.Headers.ContainsKey("X-Api-Key"))
 {
 ctx.Response.StatusCode = StatusCodes.Status401Unauthorized;
 await ctx.Response.WriteAsync("Missing API key");
 return; // ❌ stop here — do NOT call next()
 }
 await next(ctx); // ✅ all good — carry on
});
```

One subtlety: middleware that ran *before* the short-circuiting one still gets to run its "after `next()`" code on the way out — the onion still unwinds through the layers it already entered. And don't call `next()` after you've decided to short-circuit, or you'll handle the request twice.

---

### Q9. Middleware **order** — walk through the standard ordering and why each position matters. (Classic senior gotcha.)

**Answer.** Because the pipeline is a single line, order is everything. Here's the standard order for a full API/MVC app:

```csharp
app.UseExceptionHandler("/error"); // 1. FIRST: catches errors from everything below it
app.UseHsts(); // (prod) tell browsers "always use HTTPS"
app.UseHttpsRedirection(); // 2. push http → https before doing real work
app.UseStaticFiles(); // 3. serve files like /logo.png early and stop
app.UseRouting(); // 4. FIGURE OUT which endpoint matches
app.UseCors(); // 5. apply cross-origin rules
app.UseAuthentication(); // 6. WHO are you?
app.UseAuthorization(); // 7. are you ALLOWED?
app.MapControllers(); // 8. finally, RUN the endpoint
```

Why this order:

- **Exception handler first** so it wraps everything below and can catch their errors on the way out. Put it last and it can't catch anything.
- **Static files early** so a request for `/logo.png` is served right away without wasting time on routing and auth.
- **Routing before auth** because auth needs to know *which endpoint* was matched (to read its `[Authorize]` rules). So the order is: match the endpoint → check who you are → check if you're allowed → run it.
- **Authentication before authorization** — you have to know *who* someone is before you can check *what they're allowed to do*.
- **CORS in the middle** so cross-origin browser checks pass before authorization might reject the request.

❌ **The classic bug:** putting `UseAuthentication`/`UseAuthorization` in the wrong spot (after `MapControllers`, or before `UseRouting`). Then `[Authorize]` silently does nothing or throws, because the endpoint info it needs isn't ready yet. In the minimal setup, routing and endpoints are added for you automatically, so as long as you don't rearrange things, auth lands in the right place.

---

### Q10. How do you write custom middleware — inline vs a class vs `IMiddleware`?

**Answer.**

There are **three ways** to create custom middleware in ASP.NET Core.

### 1. Inline Middleware

Defined directly in `Program.cs` using `app.Use()`.

```csharp
app.Use(async (context, next) =>
{
    // Before
    await next();
    // After
});
```

**Use when:**
- Simple or one-time middleware
- Small applications

### 2. Middleware Class (Most Common)

Create a class with an `InvokeAsync` method.

```csharp
public class LoggingMiddleware
{
    private readonly RequestDelegate _next;

    public LoggingMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // Before

        await _next(context);

        // After
    }
}
```

Register it:

```csharp
app.UseMiddleware<LoggingMiddleware>();
```

**Use when:**
- Middleware is reusable
- Large applications
- Most common approach

### 3. `IMiddleware`

Implement the `IMiddleware` interface.

```csharp
public class LoggingMiddleware : IMiddleware
{
    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        // Before

        await next(context);

        // After
    }
}
```

Register it:

```csharp
builder.Services.AddTransient<LoggingMiddleware>();

app.UseMiddleware<LoggingMiddleware>();
```

**Use when:**
- Middleware depends heavily on Dependency Injection
- You want the middleware itself to be managed by the DI container

---

### Q11. Why can't you change response headers after writing to the body?

**Answer.** Once the **response body starts being sent to the client**, the HTTP headers have already been sent. At that point, the headers are **locked** and cannot be modified.

```csharp
app.Run(async context =>
{
    await context.Response.WriteAsync("Hello");

    // Throws an exception
    context.Response.Headers["MyHeader"] = "Value";
});
```
This fails because `"Hello"` has already started the response.

### Correct

Set headers **before** writing the response body.

```csharp
app.Run(async context =>
{
    context.Response.Headers["MyHeader"] = "Value";

    await context.Response.WriteAsync("Hello");
});
```

---

### Q12. How does exception-handling middleware work, and how does it differ from the developer exception page?

**Answer.** Two tools for handling crashes, and you pick based on environment:

- **`UseDeveloperExceptionPage`** (development only) — shows a detailed error page with the full stack trace and request details. Great for debugging. ❌ **Never** use it in production — it leaks internal details to users. In the minimal setup it's turned on automatically in development.
- **`UseExceptionHandler`** (production) — catches crashes and shows a safe, generic error response without leaking internals. It stashes the real error so your error page can log it.

Both work because they sit at the **top** of the pipeline and wrap everything below in a try/catch:

```csharp
if (app.Environment.IsDevelopment())
 app.UseDeveloperExceptionPage(); // detailed errors for you
else
 app.UseExceptionHandler("/error"); // safe generic errors for users
```

Modern option (.NET 8+): **`IExceptionHandler`** — a class you register in DI (`AddExceptionHandler<T>()` + `app.UseExceptionHandler()`) that returns a clean `ProblemDetails` error body. This is the preferred approach for APIs.

---

## M3 — Program.cs / Minimal Hosting & WebApplicationBuilder

### Q13. Describe the minimal hosting model and the two phases of `Program.cs`.

**Answer.** Since .NET 6, the default is the **minimal hosting model**: one `Program.cs` file, no `Main` method, no `Startup` class. It has two clear phases split by one line.

**Phase 1 — register your services (the "what you have" list), using `builder`:**
```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers(); // register services
builder.Services.AddDbContext<AppDb>(o => o.UseSqlServer(
 builder.Configuration.GetConnectionString("Default")));
builder.Services.Configure<MyOptions>(builder.Configuration.GetSection("MyOptions"));
builder.Logging.AddConsole();
```

**Phase 2 — build the request pipeline (the "how requests flow" part), using `app`:**
```csharp
var app = builder.Build(); // the dividing line — services are now locked in

app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

The dividing line is `builder.Build()`: **before** it you register services; **after** it you set up the pipeline and routes. You can't add more services after `Build()` — the list is sealed. This split does the same job the old `Startup` class did, just in one file.

`WebApplication.CreateBuilder` also sets up sensible defaults for you automatically: the web server, config loading (M4), logging, DI, and the developer error page in development.

---

### Q14. How does minimal hosting relate to the old `Startup.cs` model? (Brief.)

**Answer.** Old apps (before .NET 6) used a **`Startup`** class with two methods: `ConfigureServices` (register services) and `Configure` (build the pipeline). The minimal model just **merges both into one file**: `builder.Services.*` replaces `ConfigureServices`, and the `app.Use*`/`app.Map*` calls replace `Configure`.

Nothing is lost — it's the same thing reorganized, and the old `Startup` style still works if you wire it up. New templates use the minimal model. (You may also hear about OWIN or System.Web — those are old pre-Core history, not relevant to modern ASP.NET Core.)

---

### Q15. Where does DI registration happen, and what does `builder.Services` give you?

**Answer.** `builder.Services` is the **list of services** the app knows how to create. Everything you want the container to hand out later, you register here in Phase 1, each with a lifetime (`AddSingleton`, `AddScoped`, `AddTransient` — see [DI & lifetimes](dotnet-platform.md#j1--dependency-injection), DI & lifetimes):

```csharp
builder.Services.AddScoped<IOrderService, OrderService>();
builder.Services.AddSingleton<IClock, SystemClock>();
builder.Services.AddHttpClient<GitHubClient>(); // a ready-to-use HttpClient
builder.Services.AddControllers(); // registers a bunch of framework services at once
```

When you call `builder.Build()`, it turns that list into the live container, which you can reach afterward as `app.Services`. Each incoming request gets its own "scope," and scoped services live for that one request. Helpers like `AddControllers()` and `AddDbContext()` are just convenient bundles of registrations. This "register here, receive via constructor" flow is what controllers, middleware, and background jobs all rely on.

---

## M4 — Configuration & the Options Pattern (CORE)

### Q16. What is `IConfiguration`, and what is the provider precedence order?

**Answer.** 
`IConfiguration` is the .NET interface used to read application configuration from multiple sources, such as **appsettings.json**, **environment variables**, **User Secrets**, and **command-line arguments**.

ASP.NET Core combines all configuration sources into a single `IConfiguration` object.

### Example

```csharp
string connectionString = configuration.GetConnectionString("DefaultConnection");

string apiKey = configuration["ApiKey"];
```
### Provider Precedence (Highest → Lowest)

1. Command-line arguments
2. Environment variables
3. User Secrets (Development only)
4. `appsettings.{Environment}.json`
5. `appsettings.json`

If the same key exists in multiple providers, the **higher-precedence provider overrides the lower one**.

---

### Q17. How do nested keys work, and how do you write them as environment variables?

**Answer.** Configuration values can be organized in nested sections.


```json
{
  "Email": {
    "Host": "smtp.gmail.com",
    "Port": 587
  }
}
```
Access them:
```csharp
string host = builder.Configuration["Email:Host"];
int port = builder.Configuration.GetValue<int>("Email:Port");
```
---

### Q18. How do you bind configuration to a strongly-typed object?

**Answer.** Instead of reading loose string keys all over your code, you can load a settings section into a small class (a plain object with matching properties). Three ways:

```csharp
public class SmtpOptions { public string Host { get; set; } = ""; public int Port { get; set; } }

// 1. Get<T> — quick one-off, returns a new object:
var smtp = builder.Configuration.GetSection("Smtp").Get<SmtpOptions>();

// 2. Bind — fill in an object you already have:
var smtp2 = new SmtpOptions();
builder.Configuration.GetSection("Smtp").Bind(smtp2);

// 3. Configure<T> — register it so DI hands it out (the usual choice):
builder.Services.Configure<SmtpOptions>(builder.Configuration.GetSection("Smtp"));
```

Property names match keys ignoring case. Option 3 is what you almost always want, because it plugs into the **Options pattern** (next) and delivers settings to your classes through DI with proper reload behavior.

---

### Q20. How do you validate options, and what is `ValidateOnStart`?

**Answer.** The Options system can check your settings are valid, so a bad value fails loudly instead of causing a confusing crash deep inside a request later. You can use validation attributes, a small check function, or a dedicated validator class:

```csharp
builder.Services.AddOptions<SmtpOptions>()
 .Bind(builder.Configuration.GetSection("Smtp"))
 .ValidateDataAnnotations() // honors [Required], [Range], etc.
 .Validate(o => o.Port is > 0 and < 65536, "Port out of range")
 .ValidateOnStart(); // check at startup, not on first use
```

By default, validation runs **lazily** — the first time something reads the value, which could be mid-request in production. **`ValidateOnStart()`** moves that check to app startup, so if the config is wrong, the app **refuses to start** instead of failing later. Senior habit: for important settings, always pair `ValidateDataAnnotations().ValidateOnStart()` so a typo in a connection string crashes the deploy, not the first customer.

---

## M5 — Environments

### Q21. How do environments work — `ASPNETCORE_ENVIRONMENT`, `IWebHostEnvironment`, and per-environment config?

**Answer.** The current environment is just a name set by the **`ASPNETCORE_ENVIRONMENT`** environment variable. The three common names are **Development**, **Staging**, and **Production** (if unset, it's Production). This name controls three things:

1. **Which settings file loads** — `appsettings.{Environment}.json` layers on top of the base file (M4).
2. **How your code behaves** — you check `app.Environment` to branch:

```csharp
if (app.Environment.IsDevelopment())
 app.UseDeveloperExceptionPage(); // detailed errors, dev only
else
{
 app.UseExceptionHandler("/error"); // safe errors everywhere else
 app.UseHsts();
}
```

Helpers: `IsDevelopment()`, `IsProduction()`, `IsStaging()`, `IsEnvironment("QA")`. The environment object also tells you paths like `ContentRootPath` and `WebRootPath`.

3. **`launchSettings.json`** — a **development-only** file used by `dotnet run` and Visual Studio to set the environment name and launch URLs. It is **not deployed** and does nothing in production. In production you set `ASPNETCORE_ENVIRONMENT` through the real host (container setting, service config, App Service setting).

❌ **Gotcha:** the helper methods ignore case, but the config file name (`appsettings.Development.json`) must match the exact casing on Linux, which is case-sensitive. Get the casing wrong and your override quietly won't load.

---

## M6 — IHostedService / BackgroundService

### Q22. `IHostedService` vs `BackgroundService` — what are they and how do you run background work?

**Answer.** These are for **background work** that runs alongside your app for its whole life (polling a queue, cleanup jobs, etc.).

**`IHostedService`** is the basic contract with two methods: `StartAsync` (runs when the app starts) and `StopAsync` (runs during shutdown). The host starts them all on startup and stops them on shutdown.

**`BackgroundService`** is a helper base class for the most common case: a job that loops. You just override one method, `ExecuteAsync`, with your loop — it handles the start/stop plumbing for you.

```csharp
public class QueueProcessor(ILogger<QueueProcessor> log) : BackgroundService
{
 protected override async Task ExecuteAsync(CancellationToken stoppingToken)
 {
 while (!stoppingToken.IsCancellationRequested) // ✅ stop cleanly on shutdown
 {
 log.LogInformation("Polling...");
 await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
 }
 }
}

builder.Services.AddHostedService<QueueProcessor>();
```

### Why Both `IHostedService` and `BackgroundService`?

`BackgroundService` is built **on top of** `IHostedService`.

If you implement `IHostedService`, you must manage everything yourself.

```csharp
public class MyService : IHostedService
{
    public Task StartAsync(CancellationToken token)
    {
        // Start background task
    }

    public Task StopAsync(CancellationToken token)
    {
        // Stop background task
    }
}
```

This gives you **full control**, but requires more code.

With `BackgroundService`, Microsoft handles `StartAsync()` and `StopAsync()` for you. You only write the background work.

```csharp
public class Worker : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            // Background work
        }
    }
}
```

### When to Use

- **`BackgroundService`** → Most long-running background tasks (recommended).
- **`IHostedService`** → When you need custom startup/shutdown logic or don't need a continuously running worker.


---

### Q23. The scoped-service gotcha: how do you use a `DbContext` inside a `BackgroundService`?

**Answer.** Background services are **singletons** (one instance for the whole app), but `DbContext` and most repositories are registered **scoped** (one per operation). Injecting a scoped `DbContext` into a singleton's constructor traps a single copy forever — a **captive dependency**, which causes threading errors and breaks how `DbContext` is meant to be used ([DI & lifetimes](dotnet-platform.md#j1--dependency-injection), DI lifetimes; [dotnet-ef-core.md](dotnet-ef-core.md), DbContext lifetime).

The fix: inject **`IServiceScopeFactory`** and create a fresh scope for each unit of work, then get the `DbContext` from that scope:

```csharp
public class OrderSweeper(IServiceScopeFactory scopeFactory) : BackgroundService
{
 protected override async Task ExecuteAsync(CancellationToken stoppingToken)
 {
 while (!stoppingToken.IsCancellationRequested)
 {
 using (var scope = scopeFactory.CreateScope()) // ✅ fresh scope each loop
 {
 var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
 await db.Orders.Where(o => o.IsStale).ExecuteDeleteAsync(stoppingToken);
 } // scope ends here → DbContext is cleaned up
 await Task.Delay(TimeSpan.FromMinutes(1), stoppingToken);
 }
 }
}
```

Create the scope **inside** the loop (once per pass), not once around the whole loop — otherwise you're back to one long-lived `DbContext` that piles up data and leaks memory.

---
