# Legacy .NET Framework & Legacy Entity Framework (EF 6 / EF 4) Interview Questions
---

> **Overview:** This document covers legacy **.NET Framework (4.x and earlier)**, **ASP.NET MVC 5 / Web API 2**, **WCF**, and **Legacy Entity Framework (EF 6 / EF 4 / EDMX / ObjectContext)**. These questions are frequently asked when maintaining legacy enterprise applications or migrating legacy codebases to modern .NET 8+.

---

## Section 1 — .NET Framework Architecture & Web Hosting

### Q1. What is the fundamental architectural difference between .NET Framework (4.8 and earlier) and modern .NET Core / .NET 5+?

**Answer.**

| Architectural Aspect | .NET Framework (4.x and earlier) | Modern .NET Core / .NET 5+ |
| :--- | :--- | :--- |
| **OS Compatibility** | Windows only | Cross-platform (Windows, Linux, macOS) |
| **Hosting Model** | Tightly coupled to IIS (`w3wp.exe`) | Self-hosted via Kestrel (runs behind IIS, Nginx, Apache) |
| **Installation Model** | Machine-wide GAC (Global Assembly Cache) | Side-by-side deployment (self-contained or framework-dependent) |
| **Configuration** | XML-based (`Web.config`, `App.config`) | JSON/Env variables (`appsettings.json`, Secret Manager) |
| **Dependency Injection** | Third-party containers (Autofac, Unity, Ninject) | Built-in native DI container (`IServiceProvider`) |
| **Open Source** | Proprietary Microsoft framework | 100% Open Source |

---

### Q2. What is `Web.config`, how does it differ from `appsettings.json`, and how did configuration compilation work in legacy ASP.NET?

**Answer.** **`Web.config`** is an XML-based configuration file used in legacy ASP.NET (.NET Framework) to configure IIS settings, database connection strings, security settings, app settings, and HTTP modules/handlers.

```xml
<!-- Legacy Web.config -->
<configuration>
  <connectionStrings>
    <add name="DefaultConnection" 
         connectionString="Data Source=.;Initial Catalog=LegacyDb;Integrated Security=True;" 
         providerName="System.Data.SqlClient" />
  </connectionStrings>
  <appSettings>
    <add key="ApiUrl" value="https://api.legacy.com" />
  </appSettings>
  <system.web>
    <compilation debug="true" targetFramework="4.8" />
    <httpRuntime targetFramework="4.8" maxRequestLength="4096" />
  </system.web>
</configuration>
```

#### Reading `Web.config` values in C#:
```csharp
string connStr = ConfigurationManager.ConnectionStrings["DefaultConnection"].ConnectionString;
string apiUrl = ConfigurationManager.AppSettings["ApiUrl"];
```

#### Differences from `appsettings.json`:
1. **Format**: XML with strict schema validation vs flexible JSON.
2. **Hierarchy**: `Web.config` supports hierarchical XML inheritance across IIS parent/child folders.
3. **Environment Transforms**: Legacy projects used XML transforms (`Web.Debug.config` $\rightarrow$ `Web.Release.config`). Modern .NET uses environment-specific JSON files (`appsettings.Development.json`).

---

### Q3. What is `Global.asax` (`HttpApplication`), and what are its key lifecycle events?

**Answer.** **`Global.asax`** (Global Application Class) handles application-level events raised by ASP.NET during the lifecycle of the IIS worker process (`w3wp.exe`).

```csharp
public class MvcApplication : System.Web.HttpApplication
{
    // Fired ONCE when the first request hits the application worker process
    protected void Application_Start()
    {
        AreaRegistration.RegisterAllAreas();
        GlobalConfiguration.Configure(WebApiConfig.Register); // Web API 2 setup
        FilterConfig.RegisterGlobalFilters(GlobalFilters.Filters);
        RouteConfig.RegisterRoutes(RouteTable.Routes);
        BundleConfig.RegisterBundles(BundleTable.Bundles);
    }

    // Fired at the start of EVERY HTTP request
    protected void Application_BeginRequest()
    {
        // Custom header injection or correlation tracking
    }

    // Fired on unhandled exceptions
    protected void Application_Error()
    {
        Exception ex = Server.GetLastError();
        // Log exception to Event Viewer or log file
    }

    // Fired when a new user session is initiated
    protected void Session_Start()
    {
        Session["StartTime"] = DateTime.Now;
    }

    // Fired when the application pool shuts down
    protected void Application_End()
    {
    }
}
```

---

### Q4. What are `IHttpHandler` and `IHttpModule` in legacy ASP.NET, and how do they map to modern Middleware?

**Answer.**
- **`IHttpModule`**: Intercepts **every** HTTP request flowing into the ASP.NET pipeline. Used for cross-cutting concerns like authentication, logging, or URL rewriting (equivalent to **Middleware** in ASP.NET Core).
- **`IHttpHandler`**: Endpoints that process the request and generate the final HTTP response for specific file extensions or routes (e.g. `.aspx`, `.ashx`, or custom image generators).

```csharp
// Legacy Custom IHttpModule
public class CustomLoggingModule : IHttpModule
{
    public void Init(HttpApplication context)
    {
        context.BeginRequest += (src, args) => {
            // Intercept request
        };
    }
    public void Dispose() { }
}
```

#### Mapping to Modern ASP.NET Core:
Both `IHttpModule` and `IHttpHandler` were replaced by a unified, lightweight **Middleware pipeline** (`app.UseMiddleware()`) and Controllers/Minimal APIs in .NET Core.

---

### Q5. What is IIS Application Pool (AppPool), worker process (`w3wp.exe`), and how did hosting differ in legacy ASP.NET vs .NET Core?

**Answer.**
- **Application Pool (AppPool)**: An isolation boundary in IIS containing one or more web applications. Keeps applications isolated so a crash in one AppPool does not crash others.
- **Worker Process (`w3wp.exe`)**: The Windows process created by IIS to execute the ASP.NET application and host the CLR (Common Language Runtime).

#### Legacy vs Modern Hosting Model:

```text
Legacy .NET Framework (In-Process IIS Hosting):
Client ---> IIS (w3wp.exe) [Runs CLR + ASP.NET Pipeline + App Code inside w3wp.exe]

Modern .NET Core / .NET 8+ (Reverse Proxy Model):
Client ---> IIS / Nginx (Reverse Proxy) ---> Kestrel (Self-Hosted .NET Executable)
```

In legacy ASP.NET, IIS hosted the CLR directly inside `w3wp.exe`. In modern .NET, Kestrel is the lightweight, cross-platform web server that runs the application, while IIS or Nginx acts as a reverse proxy.

---

### Q6. What is WCF (Windows Communication Foundation), what are the ABCs of WCF, and why is it not supported in .NET Core?

**Answer.** **WCF (Windows Communication Foundation)** was the enterprise framework in .NET Framework 3.0–4.8 for building SOAP-based service-oriented applications (SOA).

#### The ABCs of WCF:
1. **Address**: *Where* the service is located (`http://localhost:8080/UserService`).
2. **Binding**: *How* to communicate (protocol & encoding: `BasicHttpBinding`, `NetTcpBinding`, `WSHttpBinding`).
3. **Contract**: *What* operations the service exposes (`[ServiceContract]` interface and `[OperationContract]` methods).

```csharp
[ServiceContract]
public interface IUserService
{
    [OperationContract]
    UserDto GetUser(int id);
}
```

#### Why WCF server is absent in .NET Core:
WCF server relies heavily on Windows IIS features, WS-* security standards, System.IdentityModel, and complex XML configurations. Microsoft replaced WCF with **gRPC** (for high-performance binary RPC) and **ASP.NET Core Web APIs** (for REST/JSON). Community projects like **CoreWCF** allow migrating WCF services to .NET Core today.

---

## Section 2 — Legacy ASP.NET MVC 5 & Web API 2

### Q7. How did ASP.NET Web API 2 differ from ASP.NET MVC 5 before ASP.NET Core?

**Answer.** In legacy .NET Framework, MVC 5 and Web API 2 were **two separate frameworks** with different base classes, routing systems, and namespaces:

| Feature | ASP.NET MVC 5 | ASP.NET Web API 2 |
| :--- | :--- | :--- |
| **Base Class** | `System.Web.Mvc.Controller` | `System.Web.Http.ApiController` |
| **Return Type** | `ActionResult` (`ViewResult`, `JsonResult`) | `IHttpActionResult` (`Ok()`, `NotFound()`) or raw object |
| **Routing Registry** | `RouteTable.Routes` | `GlobalConfiguration.Configuration.Routes` |
| **Target Audience** | HTML Views (Razor rendering) | RESTful JSON / XML web services |

#### Unified in .NET Core:
ASP.NET Core combined MVC Controllers and Web API Controllers into a **single unified framework** inheriting from `ControllerBase` or `Controller`.

---

### Q8. What is `DependencyResolver` in ASP.NET MVC 5 / Web API 2?

**Answer.** Legacy .NET Framework had **no built-in Dependency Injection container**. Developers relied on third-party IoC containers (Autofac, Unity, Ninject, Castle Windsor) and registered them using `DependencyResolver`:

```csharp
// Legacy ASP.NET MVC 5 DI Registration
var container = new UnityContainer();
container.RegisterType<IUserRepository, UserRepository>();

// Tell MVC to use Unity for instantiating controllers
DependencyResolver.SetResolver(new UnityDependencyResolver(container));
```

---

### Q9. What are Action Filters in legacy ASP.NET MVC 5 (`IActionFilter`, `IAuthorizationFilter`)?

**Answer.** Action Filters in MVC 5 executed custom logic before or after a controller action executed.

```csharp
public class CustomLogAttribute : ActionFilterAttribute
{
    public override void OnActionExecuting(ActionExecutingContext filterContext)
    {
        // Code executed BEFORE action method runs
    }

    public override void OnActionExecuted(ActionExecutedContext filterContext)
    {
        // Code executed AFTER action method finishes
    }
}

// Applied onto Controller or Action:
[CustomLog]
public ActionResult Index() => View();
```

---

## Section 3 — Legacy Entity Framework (EF 6 / EF 4 / EDMX)

### Q10. What is `ObjectContext`, how does it differ from `DbContext` (EF 4.1+), and when is `ObjectContext` used?

**Answer.** 
- **`ObjectContext`**: The original, low-level Entity Framework context class introduced in EF 1.0/EF 4. Provides direct access to EF internals (`ObjectSet<T>`, metadata workspace, explicit relationship manager, `AddObject`, `Attach`).
- **`DbContext`**: Introduced in EF 4.1 as a **simplified facade wrapper around `ObjectContext`**. Designed for everyday development with a cleaner API (`DbSet<T>`).

```csharp
// 1. Legacy ObjectContext (EF 4 and lower)
using (var context = new MyObjectContext())
{
    ObjectSet<User> users = context.CreateObjectSet<User>();
    User u = users.FirstOrDefault(x => x.Id == 1);
    context.Attach(u);
    context.SaveChanges();
}

// 2. DbContext (EF 4.1+ and EF 6)
using (var context = new MyDbContext())
{
    User u = context.Users.FirstOrDefault(x => x.Id == 1);
    context.SaveChanges();
}
```

#### Accessing Underlying `ObjectContext` from `DbContext`:
```csharp
// Access ObjectContext when low-level metadata or explicit ObjectQuery access is required
var objectContext = ((IObjectContextAdapter)dbContext).ObjectContext;
```

---

### Q11. What is an EDMX file (`.edmx`), and what are its 3 internal XML layers (SSDL, CSDL, MSL)?

**Answer.** An **`.edmx` file** (Entity Data Model XML) is an XML schema file used in legacy EF Database First / Model First workflows to define the mapping between a relational database and C# entity classes.

```text
        ┌──────────────────────────────────────────────┐
        │       CSDL (Conceptual Schema Definition)     │  <-- C# Entity Classes
        └──────────────────────┬───────────────────────┘
                               │
        ┌──────────────────────▼───────────────────────┐
        │       MSL (Mapping Specification Language)   │  <-- Column-to-Property Mappings
        └──────────────────────┬───────────────────────┘
                               │
        ┌──────────────────────▼───────────────────────┐
        │       SSDL (Store Schema Definition Language)│  <-- Database Tables & Columns
        └──────────────────────────────────────────────┘
```

1. **CSDL (Conceptual Schema)**: Describes the C# domain entity classes, properties, and navigation properties.
2. **SSDL (Store Schema)**: Describes the database schema (tables, columns, data types, primary/foreign keys).
3. **MSL (Mapping Schema)**: Defines how CSDL properties map to SSDL database columns.

⚠️ **Why EDMX was removed in EF Core**: EDMX files caused massive merge conflicts in source control (Git), loaded slowly for large databases, and required visual designer tools in Visual Studio. EF Core dropped EDMX in favor of pure Code First / Scaffolding (`dotnet ef dbcontext scaffold`).

---

### Q12. What are the 3 workflows in legacy EF 6 (Database First, Model First, Code First)?

**Answer.**

1. **Database First**: You design the SQL database first. Visual Studio generates the `.edmx` file and auto-generates T4 template C# entity classes.
2. **Model First**: You draw entities in a Visual Studio EDMX visual designer canvas. Visual Studio generates SQL DDL scripts to create the database.
3. **Code First**: You write C# POCO classes and `DbContext`. EF generates the database schema using migrations (`Add-Migration`).

---

### Q13. How did Change Tracking and Lazy Loading work in EF 6 vs EF Core (Virtual properties & Proxies)?

**Answer.** In EF 6, **Lazy Loading** was enabled by default if navigation properties were marked `virtual`:

```csharp
public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; }
    
    // Virtual enables Dynamic Proxy generation for Lazy Loading!
    public virtual ICollection<Order> Orders { get; set; }
}
```

#### How Dynamic Proxies work:
At runtime, EF 6 dynamically generates a wrapper class (`CustomerProxy`) inheriting from `Customer`. When accessing `customer.Orders`, the proxy intercepts the property getter, executes a SQL query to fetch orders from the database automatically, and populates the collection.

⚠️ **Gotcha (N+1 Query Problem)**: Iterating over 100 customers and accessing `customer.Orders` inside a loop fires **101 separate SQL queries**!
Fix: Use **Eager Loading** with `.Include(c => c.Orders)`.

---

### Q14. What are Code First Migrations in EF 6, and how is `__MigrationHistory` structured?

**Answer.** In legacy EF 6 Code First, migrations were managed via Package Manager Console commands:

```powershell
Enable-Migrations              # Enables migrations folder & Configuration.cs
Add-Migration InitialCreate    # Generates a C# migration file
Update-Database                # Applies pending migrations to SQL database
```

#### The `__MigrationHistory` Table:
EF 6 creates a special database system table named `dbo.__MigrationHistory` containing:
- `MigrationId` (timestamp + migration name)
- `Model` (compressed binary GZIP byte array of the EDMX metadata model snapshot)
- `ProductVersion` (EF version, e.g. `6.4.4`)

Every time `Update-Database` runs, EF 6 compares the current model snapshot stored in `__MigrationHistory` with the target model to generate delta DDL changes.

---

### Q15. What are `DbContext` lifetime management best practices in legacy .NET Framework applications?

**Answer.** In legacy .NET Framework applications (which lacked built-in DI scopes):

1. **Web Applications (ASP.NET MVC 5 / Web API 2)**: `DbContext` was instantiated **per HTTP Request** (using IoC containers like Autofac registered as `InstancePerRequest()`).
2. **Desktop / Console Apps**: `DbContext` was used in short-lived `using` blocks per unit of work.

```csharp
// ✅ Correct: Short-lived DbContext unit of work
using (var db = new AppDbContext())
{
    var user = db.Users.Find(userId);
    user.LastLogin = DateTime.UtcNow;
    db.SaveChanges();
}
```

⚠️ **Anti-Pattern (Static / Long-Lived DbContext)**: Keeping a single `DbContext` instance alive globally causes:
- Memory leaks (unbounded ChangeTracker memory growth).
- Stale data (cached entity instances never re-querying the database).
- Thread-safety exceptions (`DbContext` is NOT thread-safe).
