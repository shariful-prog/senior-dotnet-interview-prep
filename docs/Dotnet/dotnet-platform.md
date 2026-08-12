# J. .NET Platform & Ecosystem
---

## J1 — Dependency Injection

### Q1. What is dependency injection, and what problem does it solve?

**Answer.** **Dependency injection (DI)** is a design pattern in which a class receives its required dependencies from an external source rather than creating them itself.

Why is that better? Three reasons:

- **Loose coupling** — your class depends on an *interface* (like `IEmailSender`) rather than a specific class (`SmtpEmailSender`). You can swap the implementation without touching the class.
- **Testability** — in tests you can pass in a fake email sender instead of really sending emails.
- It's the practical way to follow the **"D" in SOLID** (depend on abstractions, not concrete details).

```csharp
// WITHOUT DI — hard-wired to a specific class, hard to test
public class OrderService
{
    private readonly SmtpEmailSender _email = new();   // locked to SmtpEmailSender
    public void Place(Order o) { _email.Send(o.Email, "Placed"); }
}

// WITH DI — depends on an interface, given from outside
public class OrderService(IEmailSender email)          // primary constructor (C# 12)
{
    public void Place(Order o) { email.Send(o.Email, "Placed"); }
}
```

An **IoC container** (like `Microsoft.Extensions.DependencyInjection`) is just a tool that automates handing out these dependencies. You *could* wire everything up by hand, but the container does it for you and manages how long each object lives.

---

### Q2. How does the built-in DI container work — registration and resolution?

**Answer.** There are two steps:

1. **Registration** — at startup, you tell the container "when someone needs `IOrderRepository`, give them a `SqlOrderRepository`." You do this with `IServiceCollection` (in ASP.NET Core that's `builder.Services`).
2. **Resolution** — later, when your code needs a service, the container **creates it for you** (usually automatically through constructor injection).

```csharp
var builder = WebApplication.CreateBuilder(args);

// Registration — map interfaces to implementations
builder.Services.AddScoped<IOrderRepository, SqlOrderRepository>();
builder.Services.AddSingleton<IClock, SystemClock>();
builder.Services.AddTransient<IEmailSender, SmtpEmailSender>();

var app = builder.Build();

// Resolution — usually automatic; you can also do it manually:
using var scope = app.Services.CreateScope();
var repo = scope.ServiceProvider.GetRequiredService<IOrderRepository>();
```

Tip: use `GetRequiredService<T>()` (throws a clear error if you forgot to register something) rather than `GetService<T>()` (returns `null`, which turns into a confusing crash later). The built-in container is intentionally simple; if you need advanced features, teams sometimes use Autofac — but the built-in one is enough for most apps.

---

### Q3. Explain the three service lifetimes — Transient, Scoped, and Singleton. (The classic DI question.)

**Answer.** The lifetime controls **how often the container makes a new instance**:

- **Transient** — a **brand-new instance every time** it's requested. Use for lightweight, stateless services.
- **Scoped** — **one instance per "scope."** In a web app, a scope = **one HTTP request**, so everything in that request shares the same instance. This is the natural fit for EF Core's **`DbContext`** (one per request).
- **Singleton** — **one instance for the whole app**, shared by everyone. Use for stateless, thread-safe services, caches, and configuration. Because it's shared across all requests and threads, **it must be thread-safe.**

```csharp
services.AddTransient<IGuidGenerator, GuidGenerator>();   // new every time
services.AddScoped<AppDbContext>();                       // one per HTTP request
services.AddSingleton<IMemoryCache, MemoryCache>();       // one for the whole app
```

Simple way to remember it: **Transient = per-use, Scoped = per-request, Singleton = per-app.** (There's no default — you must pick one when registering.)

---

### Q4. What is a captive dependency, and why is it a bug? (The most-asked DI gotcha.)

**Answer.** A **captive dependency** happens when a **long-lived** service grabs hold of a **short-lived** one — most commonly, putting a **Scoped** service inside a **Singleton**.


```csharp
public class EmailService // Singleton
{
    private readonly AppDbContext _dbContext;

    public EmailService(AppDbContext dbContext)
    {
        _dbContext = dbContext;
    }
}
```

Here:

- `EmailService` → **Singleton**
- `AppDbContext` → **Scoped**

The singleton keeps the same `DbContext` instance forever, even though a new `DbContext` should be created for each request.


#### Why Is It a Bug?

- The scoped service lives longer than intended.
- It can use stale or invalid data.
- It may cause thread-safety issues.
- It breaks the expected service lifetime.


#### Correct Lifetime Rule

A service should **only depend on services with the same or longer lifetime**.

```
Singleton  → Singleton ✅

Scoped     → Scoped / Singleton ✅

Transient  → Transient / Scoped / Singleton ✅

Singleton  → Scoped ❌
Singleton  → Transient ❌ (usually a design issue)
```

**Good news:** in Development, ASP.NET Core detects this and throws a clear error ("Cannot consume scoped service from singleton"). But that check is off in production for performance, so don't rely on it — just don't do it.

---

### Q5. If a singleton legitimately needs a scoped service, how do you do it correctly?

**Answer.** Don't inject the scoped service directly. Instead, inject **`IServiceScopeFactory`** (which is safe for singletons to hold), and **create a fresh scope each time** you need the scoped service — then dispose it when done.

```csharp
public class CacheWarmer(IServiceScopeFactory scopeFactory)   // safe for a singleton
{
    public async Task WarmAsync()
    {
        using var scope = scopeFactory.CreateScope();          // fresh scope
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        var data = await db.Products.ToListAsync();
        // scope (and the DbContext) is cleaned up here
    }
}
```

This is the standard pattern inside **background services** (`BackgroundService` / `IHostedService`) — they're singletons, so any database work they do each cycle needs its own fresh scope.

---

### Q6. Compare constructor injection, property injection, and the service locator pattern.

**Answer.** Three ways a class can get its dependencies. Only the first is a good default.

**1. Constructor injection — dependencies arrive as constructor parameters.**

```csharp
public class OrderService(IEmailSender email, IOrderRepository repo)   // ✅
{
    public void Place(Order o)
    {
        repo.Save(o);
        email.Send(o.CustomerEmail, "Order placed");
    }
}
```

This is the default. The dependencies are **explicit** — you can read the constructor and know exactly what this class needs — and **required**, because you can't create the object without them. Testing is easy: pass fakes in.

A bonus: if the parameter list grows huge, that's a useful warning that the class is doing too much.

**2. Property injection — dependencies are set after the object is built.**

```csharp
public class OrderService                                              // ⚠️
{
    public IEmailSender? Email { get; set; }     // must be set separately

    public void Place(Order o) => Email!.Send(o.CustomerEmail, "Order placed");
}

var svc = new OrderService();      // object exists but is only half set up
svc.Place(order);                  // 💥 NullReferenceException
```

The problem is visible in that last line: the object can exist in a broken state. The built-in .NET container **doesn't support this at all**. It's occasionally used for genuinely optional dependencies, like a logger that can default to "do nothing", but it's rare.

**3. Service locator — the class takes the container and asks it for things.**

```csharp
public class OrderService(IServiceProvider sp)                         // ❌
{
    public void Place(Order o)
    {
        var email = sp.GetRequiredService<IEmailSender>();   // hidden dependency
        email.Send(o.CustomerEmail, "Order placed");
    }
}
```

This is an **anti-pattern**. The constructor says this class needs `IServiceProvider`, which tells you nothing — the real dependency on `IEmailSender` is buried inside a method. Two consequences:

- **You can't tell what the class needs** without reading every line of it.
- **Testing is painful** — instead of passing a fake email sender, you have to build a whole container.
- **Missing registrations fail at runtime**, deep inside a method, instead of when the object is created.

| | Dependencies visible? | Can be half-built? | Testing |
|---|---|---|---|
| **Constructor** ✅ | Yes — in the signature | No | Pass fakes directly |
| **Property** ⚠️ | Partly | **Yes** | Remember to set each one |
| **Service locator** ❌ | **No** — hidden in methods | No | Needs a container |

**Use constructor injection.** Touch the container directly only in startup/wiring code, or the rare case where nothing else works — like a singleton that needs a scoped service, where you inject `IServiceScopeFactory` instead.

---

### Q7. Walk through the registration options: factory registration, TryAdd, multiple implementations, and open generics.

**Answer.** Beyond the basic `AddTransient/AddScoped/AddSingleton`, there are a few patterns worth knowing:

**Factory registration** — supply a function to build the service yourself, when construction needs some runtime logic:

```csharp
services.AddSingleton<IPaymentGateway>(sp =>
{
    var cfg = sp.GetRequiredService<IConfiguration>();
    return new StripeGateway(cfg["Stripe:ApiKey"]!);
});
```

**`TryAdd`** — registers a service **only if it isn't already registered**. Used in library setup methods so the library doesn't overwrite what the app already set up.

**Multiple implementations** — register the same interface several times. Then injecting `IEnumerable<T>` gives you **all** of them (great for running a list of validators or handlers):

```csharp
services.AddScoped<IValidator, NameValidator>();
services.AddScoped<IValidator, EmailValidator>();
public class Form(IEnumerable<IValidator> validators) { /* runs all of them */ }
```

**Open generics** — register a generic type once, and the container fills in the specific type as needed (the basis for generic repositories):

```csharp
services.AddScoped(typeof(IRepository<>), typeof(EfRepository<>));
// asking for IRepository<Order> gives you EfRepository<Order>
```

---

### Q8. How does the DI container handle disposal? What's the caveat with transients?

**Answer.** The container **cleans up the objects it created**. If a service implements [`IDisposable`](../C-Sharp/csharp-memory.md), the container disposes it automatically:

- **Scoped** services are disposed when the **scope ends** (end of the HTTP request).
- **Singleton** services are disposed when the **app shuts down**.
- **Transient** disposables are *also* tracked by the scope that created them.

**The caveat:** transient/scoped disposables **pile up in their scope until that scope ends.** So if you resolve a transient disposable from the **root** (the app-wide provider, which never ends), it's held for the **entire app lifetime** — a **memory leak**.

```csharp
// Leak risk: transient disposable from the root provider — held forever
var svc = app.Services.GetRequiredService<ITransientDisposable>();

// Correct: tie it to a scope that ends
using var scope = app.Services.CreateScope();
var svc2 = scope.ServiceProvider.GetRequiredService<ITransientDisposable>();
// disposed when 'scope' is disposed
```

Also note: the container does **not** dispose objects **you** created yourself and handed to it (like `AddSingleton(myInstance)`) — those are yours to manage. In short: *the container disposes what it builds, and nothing else.*

---

### Q9. What are keyed services (introduced in .NET 8)?

**Answer.** Sometimes you have several implementations of the same interface and want to pick a **specific one by name**. Before .NET 8, this was awkward. **.NET 8** added **keyed services**: you attach a key to each registration and ask for one by its key.

```csharp
public interface INotifier { Task SendAsync(string to, string message); }

public class EmailNotifier : INotifier { /* sends an email */ }
public class SmsNotifier   : INotifier { /* sends a text  */ }
```

**Register each one with a key:**

```csharp
builder.Services.AddKeyedSingleton<INotifier, EmailNotifier>("email");
builder.Services.AddKeyedSingleton<INotifier, SmsNotifier>("sms");
```

**Inject a specific one** with `[FromKeyedServices]` — this is the common case:

```csharp
public class WelcomeService([FromKeyedServices("email")] INotifier notifier)
{
    public Task GreetAsync(User u) => notifier.SendAsync(u.Email, "Welcome!");
}
```

**Or pick one at runtime**, when the choice depends on data:

```csharp
public class AlertService(IServiceProvider sp)
{
    public Task AlertAsync(User u, string message)
    {
        var key = u.PrefersSms ? "sms" : "email";        // decided at runtime
        var notifier = sp.GetRequiredKeyedService<INotifier>(key);
        return notifier.SendAsync(u.Contact, message);
    }
}
```

Note the second version needs `IServiceProvider`, which is the service locator pattern from Q6. That's acceptable here because the key genuinely isn't known until runtime — but prefer `[FromKeyedServices]` whenever the choice is fixed.

**Without keyed services** you had to inject all implementations and filter, which is what this replaces:

```csharp
// ❌ the old workaround
public class AlertService(IEnumerable<INotifier> all)
{
    private readonly INotifier _sms = all.First(n => n is SmsNotifier);
}
```

Typical uses: payment providers, cache backends (Redis vs in-memory), or per-tenant implementations.

---

## J1b — The Runtime: CLR, IL & JIT

> Garbage collection is covered in depth in [../C-Sharp/csharp-memory.md](../C-Sharp/csharp-memory.md). This section is about how your code actually gets executed.

### Q9b. What is the CLR, and what does it do?

**Answer.** The **CLR (Common Language Runtime)** is the engine that runs .NET code. Your program does not talk to the operating system directly — it runs *inside* the CLR, which provides the services that make .NET code work.

What it is responsible for:

- **Executing code** — turning IL into machine instructions via the JIT (Q9d).
- **Memory management** — allocating objects and running the garbage collector.
- **Type safety** — verifying that code does not read memory as the wrong type.
- **Exception handling** — the `try`/`catch` mechanism itself.
- **Threading** — the thread pool and synchronisation primitives.
- **Security** — code access checks and assembly verification.

A runtime that does this is called a **managed** environment, and .NET code is called **managed code**. The contrast is C or C++, which compile straight to machine code and manage their own memory — **unmanaged code**.

```csharp
// Managed: the CLR allocates this, tracks it, and frees it for you
var list = new List<int>();

// Unmanaged: you own the handle and must release it yourself
IntPtr handle = CreateFile(...);      // a Win32 call via P/Invoke
```

That distinction is why `IDisposable` exists: the GC knows about managed memory but not about file handles, sockets, or database connections, so you must release those yourself.

The CLR is also what makes .NET **multi-language**. C#, F#, and VB.NET all compile to the same IL, so a C# project can reference an F# library — they meet at the runtime, not the language.

---

### Q9c. What is IL, and what does compilation actually produce?

**Answer.** Compiling C# does **not** produce machine code. It produces **IL (Intermediate Language)** — a CPU-independent instruction set — which is stored in a `.dll` or `.exe` along with metadata describing every type.

There are **two compilation steps**:

```text
C# source  ──[ C# compiler, Roslyn ]──►  IL in a .dll     (at build time)
IL         ──[ JIT compiler in CLR ]──►  machine code     (at run time)
```

So a `.dll` you build is not a native binary. It holds IL plus metadata, and needs a runtime to execute.

This is what makes one build portable: the same `.dll` runs on Windows x64, Linux ARM, or macOS, because the machine-specific step is deferred until the program starts on an actual machine.

**Metadata** is the other half of the file, and it is why reflection works (Q13). The assembly describes its own types — names, members, signatures, attributes — so code can inspect it at runtime. There is no separate header file or type library.

You can read the IL for any assembly with `ildasm`, ILSpy, or `dotnet-ildasm`. It is worth doing once, because it explains things that look like magic in C#:

```text
// C#:  int x = a + b;
ldarg.0        // push a
ldarg.1        // push b
add            // add them
stloc.0        // store into x
```

IL is stack-based, which is why `async`/`await`, `yield return`, and lambdas can be compiled into state machines and closure classes — the compiler rewrites your code into plainer IL constructs.

---

### Q9d. What is the JIT compiler, and why not compile ahead of time?

**Answer.** The **JIT (Just-In-Time) compiler** converts IL into machine code **while your program runs**, one method at a time.

Nothing is compiled until it is first called. When a method runs for the first time, the JIT compiles it, caches the machine code, and every later call jumps straight to the compiled version:

```text
first call to Foo()   → JIT compiles Foo → run machine code   (slower)
later calls to Foo()  → run cached machine code               (full speed)
```

**Why this is not just a cost.** The JIT knows things an ahead-of-time compiler cannot:

- **The exact CPU** it is running on, so it can use instruction sets like AVX-512 that a portable binary could not assume.
- **Actual runtime behaviour.** The **tiered** JIT compiles quickly at first (Tier 0) to start fast, then recompiles hot methods with full optimisation (Tier 1) once it sees they matter.
- **Which branches are actually taken**, enabling inlining and devirtualisation decisions based on real data.

That is why long-running .NET server code can match or beat statically compiled code, despite starting slower.

**The trade-off is startup time.** The first request to a fresh process pays JIT cost for every method on its path — the "cold start" problem in serverless and CLI tools.

Two mitigations:

| | What it does | Trade-off |
|---|---|---|
| **ReadyToRun (R2R)** | ships pre-compiled machine code alongside the IL | bigger files; JIT can still re-optimise |
| **Native AOT** | compiles fully to native at build time, no JIT at all | no reflection-heavy code, no runtime codegen |

```xml
<PublishReadyToRun>true</PublishReadyToRun>
<PublishAot>true</PublishAot>
```

Use **Native AOT** for CLI tools, containers, and serverless where startup dominates. Keep the **JIT** for long-running servers, where its runtime knowledge wins. Native AOT has real restrictions — see Q9e.

---

### Q9e. What is Native AOT, and what are its limitations?

**Answer.** **Native AOT** compiles a .NET application into **native machine code at publish time**, eliminating the need for **JIT (Just-In-Time)** compilation at runtime.

**Default:** .NET uses **JIT** by default. Native AOT must be enabled explicitly.

### How to Enable

Add the following to your `.csproj`:

```xml
<PropertyGroup>
  <PublishAot>true</PublishAot>
</PropertyGroup>
```

Or publish using:

```bash
dotnet publish -c Release -r win-x64 -p:PublishAot=true
```

### Advantages

- Faster startup
- Lower memory usage
- Better cold-start performance

### Limitations

- Limited reflection support
- No runtime code generation
- Some libraries are not AOT-compatible
- Longer publish time
- Potentially larger executable

---

### Q9f. What is an assembly, and what is the difference between an assembly and a namespace?

**Answer.** An **assembly** is the unit of **deployment** — a compiled `.dll` or `.exe` containing IL and metadata. A **namespace** is the unit of **naming** — a prefix that organises type names.

They are independent, which is the point of the question:

```csharp
// One assembly can hold many namespaces
MyApp.dll  →  MyApp.Services, MyApp.Models, MyApp.Data

// One namespace can span several assemblies
System.Collections.Generic  →  System.Private.CoreLib.dll, System.Collections.dll
```

The practical differences:

| | Assembly | Namespace |
|---|---|---|
| Exists at runtime | yes — a loaded file | no — compiled into type names |
| Controls access | yes — `internal` is per assembly | no effect on accessibility |
| How you use it | reference the project or package | `using` directive |

`internal` is the reason this matters. It means "visible within this assembly", so moving a class to a different project can break code that a namespace change would not:

```csharp
internal class Helper { }        // visible only inside its own assembly
```

`InternalsVisibleTo` is the standard exception, used to let a test project see internals:

```xml
<ItemGroup>
  <InternalsVisibleTo Include="MyApp.Tests" />
</ItemGroup>
```

Assemblies are loaded **lazily** — the runtime loads one the first time a type in it is needed, not at startup. A missing dependency therefore often surfaces as a `FileNotFoundException` mid-run rather than at launch.

---

## J2 — .NET Framework vs .NET Core vs .NET 5+; .NET Standard

### Q10. Explain the difference between .NET Framework, .NET Core, and .NET 5+.

**Answer.** These are three chapters of the same story:

- **.NET Framework** (up to **4.8**) — the original, **Windows-only** version. It runs older WinForms, WPF, and classic ASP.NET apps. It's now in **maintenance mode** — no new versions are coming. Only use it to maintain existing apps.
- **.NET Core** (1.0 → 3.1) — a from-scratch rewrite that's **cross-platform** (Windows/Linux/macOS), **open source**, and much faster. This is where the platform's future moved.
- **.NET 5 and later** (5, 6, **8**, **9**, ...) — the **unification**. Microsoft dropped "Core" from the name to signal this is now the *one* .NET going forward. (There's deliberately no ".NET 4," to avoid confusion with Framework 4.x.)

```
.NET Framework 4.8   → Windows-only, legacy, end of the line
.NET Core 1–3.1      → cross-platform rewrite, open source
.NET 5, 6, 8, 9…     → unified, single "modern .NET" going forward
```

**Takeaway:** for anything new, target **modern .NET** (like `net8.0` or `net9.0`). Only touch .NET Framework to maintain a legacy app or use a Windows-only library that was never ported.

---

### Q11. What is .NET Standard, and is it still relevant?

**Answer.** **.NET Standard is not a runtime** — it's a **list of APIs** that every kind of .NET promises to support. Its purpose was **sharing libraries** back when there were many separate .NET flavors (Framework, Core, Xamarin, etc.). A library targeting `netstandard2.0` could be used by *all* of them, because they all supported that shared set of APIs.

```
Library targets: netstandard2.0
Can be used by:  .NET Framework 4.6.1+, .NET Core 2.0+, Xamarin, Unity
```

**Is it still relevant?** Mostly not — now that everything is unified, you can just target `net8.0` directly. You'd still use **`netstandard2.0`** when writing a library that needs to work with **both old .NET Framework apps and modern .NET**. If you don't need old Framework support, skip it and target modern .NET. (Note: `netstandard2.1` does *not* work on .NET Framework — a common gotcha.)

---

### Q12. What are TFMs, LTS vs STS releases, and the runtime vs SDK distinction?

**Answer.** Three separate ecosystem facts worth knowing:

**TFMs (Target Framework Monikers)** — the string in your project file that says what you're building for:
- `net8.0` — cross-platform
- `net8.0-windows` — adds Windows-only features (WinForms, WPF)
- `netstandard2.0` — the shared API spec

**Release schedule** — .NET ships **every November**:
- **Even-numbered = LTS (Long-Term Support), 3 years** — e.g. **.NET 8**, **.NET 10**.
- **Odd-numbered = STS (Standard-Term Support), 18 months** — e.g. **.NET 9**.
- Companies usually stick to **LTS** versions for the longer support.

**Runtime vs SDK:**
- The **Runtime** is what you need to **run** an app.
- The **SDK** is what you need to **build** an app — it includes the runtime *plus* the compiler and the `dotnet` command-line tools.
- Dev machines and build servers need the SDK; production servers can install just the runtime.

```xml
<PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
</PropertyGroup>
```

---

### Q12b. Which C# version goes with which .NET version, and what were the headline features?

**Answer.** The C# language and .NET ship together every November, and the compiler picks the language version from your `TargetFramework` — you do not choose it separately.

| .NET | C# | Headline features |
|---|---|---|
| .NET 6 (LTS) | C# 10 | global `using`, file-scoped namespaces, record structs |
| .NET 7 | C# 11 | raw string literals, required members, generic math |
| .NET 8 (LTS) | C# 12 | **primary constructors**, collection expressions `[1, 2]`, alias any type |
| .NET 9 | C# 13 | `params` collections, new lock type, `field` keyword (preview) |
| .NET 10 (LTS) | C# 14 | extension members, null-conditional assignment |

The three most commonly asked about:

```csharp
// C# 12 — primary constructor: declares and assigns in one line
public class OrderService(IOrderRepository repo)
{
    public void Place(Order o) => repo.Save(o);
}

// C# 12 — collection expressions
int[] a = [1, 2, 3];
List<int> b = [.. a, 4];        // .. spreads another collection

// C# 11 — raw string literals: no escaping needed
var json = """{ "name": "John" }""";
```

**You can override the language version**, but it is rarely a good idea:

```xml
<LangVersion>12.0</LangVersion>   <!-- ⚠️ features needing newer runtime APIs still fail -->
```

The reason: some language features are pure compiler work (primary constructors) and do work on older targets, while others need runtime support (generic math, `Span` improvements) and do not. Targeting the framework is the reliable route.

**For interviews, the safe answer** is that .NET 8 is the widely used LTS with C# 12, and the feature people ask about most is primary constructors, because it changes how dependency injection is written.

---

### Q12c. What recent breaking changes should you know about when upgrading?

**Answer.** A **breaking change** is one where code that compiled and ran on the old version no longer does, or behaves differently. Microsoft documents these per release, and upgrades are mostly smooth — but a few catch people out.

**1. Nullable reference types are on by default** (since .NET 6). New projects get thousands of warnings on old code:

```xml
<Nullable>disable</Nullable>       <!-- ✅ turn off while migrating, then re-enable -->
```

**2. `.NET 8` changed how `DateTime`/`TimeSpan` and number parsing handle some inputs**, following stricter standards. Round-tripping values between versions can differ, so re-verify any code that parses user-supplied dates.

**3. ASP.NET Core changed defaults in .NET 8**:

- `HttpClient` metric and logging names changed, which breaks dashboards.
- The `[FromServices]` behaviour and DI validation became stricter, so some previously tolerated registrations now throw at startup.

**4. `.NET 9` removed and renamed some obsolete APIs**, including older `BinaryFormatter` support — which was already unsafe and is now removed entirely rather than just obsolete:

```csharp
new BinaryFormatter().Serialize(stream, obj);   // ❌ removed — use System.Text.Json
```

**5. EF Core tightens each release.** The common one: queries that used to fall back to client-side evaluation now throw instead, which surfaces as a runtime error on code that worked before (see [dotnet-ef-core.md](dotnet-ef-core.md)).

**How to handle an upgrade** — this is what an interviewer is really asking:

1. Move one **LTS to the next** (6 → 8 → 10), not through every release.
2. Change the `TargetFramework`, build, and fix warnings before running.
3. Read the official breaking-changes list for each version you skip.
4. Rely on your test suite, and check logging and metrics names, which change quietly.

The honest summary: **modern .NET upgrades are usually a one-line change plus warning cleanup.** The painful migration is .NET Framework to modern .NET (Q10), which is a rewrite of the hosting model, not an upgrade.

---

## J3 — Reflection, Attributes & Metadata

### Q13. What is reflection, and what can you do with it?

**Answer.** **Reflection** lets your code **inspect and use type information at runtime** — even for types it didn't know about when it was compiled. Every .NET assembly carries **metadata** (a self-describing list of its types, methods, properties, etc.), and reflection is the API that reads it.

With reflection you can list a type's members, read attributes, **create objects by type name**, and **call methods** dynamically:

```csharp
Type t = typeof(Order);                            // get type info
PropertyInfo[] props = t.GetProperties();          // list its properties
MethodInfo m = t.GetMethod("Total")!;

object? instance = Activator.CreateInstance(t);    // create an instance dynamically
object? result = m.Invoke(instance, null);         // call a method dynamically
```

Reflection is what powers serializers (JSON), DI containers, ORMs (EF Core), and test runners — anything that has to work with types it didn't know about ahead of time.

---

### Q14. What are attributes, and how do you define and read a custom one?

**Answer.** An **attribute** is a label you attach to a class, property, or method to store extra information about it. You've used them already:

```csharp
[Obsolete("Use SendAsync instead")]     // compiler warns when this is called
public void Send() { }

[Required]                              // a validator reads this
public string Email { get; set; } = "";

[HttpGet("/users/{id}")]                // ASP.NET Core reads this to route requests
public User Get(int id) { }
```

**The important part: an attribute does nothing by itself.** It is only data attached to your code. Something else has to look for it and act on it — the compiler for `[Obsolete]`, the validation system for `[Required]`, ASP.NET Core for `[HttpGet]`. If nothing reads your attribute, nothing happens.

### Defining one

An attribute is just a class that inherits from `Attribute`. Constructor parameters become the values you pass when using it:

```csharp
[AttributeUsage(AttributeTargets.Property)]      // where it's allowed
public sealed class MaxLenAttribute(int length) : Attribute
{
    public int Length { get; } = length;
}
```

`[AttributeUsage]` says where it can go. Set it to `AttributeTargets.Property` and the compiler stops anyone putting it on a method.

**Naming rule:** the class ends with `Attribute`, but you leave that off when using it. `MaxLenAttribute` is applied as `[MaxLen(50)]`.

### Using it

```csharp
public class User
{
    [MaxLen(50)] public string Name { get; set; } = "";
    [MaxLen(200)] public string Bio { get; set; } = "";
}
```

At this point nothing happens yet — the labels are just sitting there.

### Reading it

You read attributes with **reflection**, which is how you inspect types at runtime:

```csharp
public static List<string> Validate(object obj)
{
    var errors = new List<string>();

    foreach (var prop in obj.GetType().GetProperties())
    {
        var rule = prop.GetCustomAttribute<MaxLenAttribute>();
        if (rule is null) continue;                      // no label → skip

        var value = prop.GetValue(obj) as string;
        if (value is not null && value.Length > rule.Length)
            errors.Add($"{prop.Name} must be {rule.Length} characters or fewer");
    }

    return errors;
}
```

```csharp
var user = new User { Name = new string('x', 100) };
foreach (var e in Validate(user))
    Console.WriteLine(e);        // Name must be 50 characters or fewer
```

That loop *is* the "something else" that gives the attribute meaning. Built-in validation works the same way, just with more rules.

⚠️ Reflection is slow, so don't run a loop like this on every request without caching the property lookups (Q15).

---

### Q15. Reflection has a reputation for being slow. What's the cost, and what are the alternatives?

**Answer.** Reflection is **much slower than normal code** — looking up members and calling methods dynamically involves extra work the compiler can't optimize away. If you do it once, no big deal. If you do it in a hot loop thousands of times, it hurts.

Ways to fix it, from easiest to fastest:

1. **Cache the results.** The lookups (like `GetProperty`) are the expensive part — do them **once** and store them, instead of every call.
2. **Compiled expressions** — build a fast function from the reflection info once, then reuse it at near-normal speed.
3. **Source generators** — the modern best option: do the work at **compile time** so there's **no reflection at runtime at all**. This is how the fast mode of `System.Text.Json` works.

Also remember **`nameof`** — it gives you a member's name as text with **no reflection at all**, which is perfect for logging and validation messages.

---

## J4 — Serialization (System.Text.Json)

### Q16. What is System.Text.Json, and how does it compare to Newtonsoft.Json?

**Answer.** **System.Text.Json (STJ)** is the **built-in, modern** JSON library, and it's now the **default** in ASP.NET Core. It was designed to be **fast and memory-efficient**. **Newtonsoft.Json** (Json.NET) is the older third-party library that was the standard for years.

```csharp
using System.Text.Json;

var json = JsonSerializer.Serialize(order);            // object → JSON
var back = JsonSerializer.Deserialize<Order>(json);    // JSON → object
```

The main differences:

| | System.Text.Json | Newtonsoft.Json |
|---|---|---|
| Speed | Faster, less memory | Slower |
| Default behavior | Strict (case-sensitive, no comments) | Lenient (forgiving) |
| Features | Leaner, growing | Very rich, lots of options |

**When you'd still pick Newtonsoft:** you need a specific feature STJ doesn't have or handles awkwardly. For most new work, STJ is the right default; reach for Newtonsoft only when a particular need forces it.

---

### Q17. How do you configure and control serialization in System.Text.Json?

**Answer.** Two ways: **options** (for overall behavior) and **attributes** (for individual properties).

**Options** — `JsonSerializerOptions`:

```csharp
var options = new JsonSerializerOptions
{
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase,            // FirstName → firstName
    PropertyNameCaseInsensitive = true,                          // match names loosely when reading
    WriteIndented = true,                                        // pretty-print
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull, // skip null values
    Converters = { new JsonStringEnumConverter() }              // enums as text, not numbers
};
var json = JsonSerializer.Serialize(order, options);
```

**Attributes** — on individual properties:

```csharp
public class Order
{
    [JsonPropertyName("order_id")] public int Id { get; set; }   // custom JSON name
    [JsonIgnore] public string Secret { get; set; } = "";        // never serialized
}
```

A common interview point: by default STJ writes **enums as numbers**. Add `JsonStringEnumConverter` to get readable **names** instead (see [enums](../C-Sharp/csharp-type-system.md)). Also, reuse your `JsonSerializerOptions` object — creating a new one every call is a real performance mistake.

---