# N. Web APIs (MVC / Minimal APIs)
---

## N1 — Controllers vs Minimal APIs

### Q1. What's the difference between MVC controllers and Minimal APIs?

**Answer.** Two ways to write HTTP endpoints. They run on the same ASP.NET Core plumbing — same routing, DI, middleware, auth — but look different.

**Controllers** are classes that group related endpoints. Each endpoint is a method with `[HttpGet]`, `[HttpPost]`, and so on.

**Minimal APIs** define an endpoint in one line, where the handler is a lambda. No class needed.

```csharp
// Controller
[ApiController]
[Route("api/[controller]")]
public class TodosController : ControllerBase
{
    [HttpGet("{id:int}")]
    public ActionResult<TodoDto> Get(int id) => /* ... */;
}

// Minimal API — same route, same DI
app.MapGet("/api/todos/{id:int}", (int id, ITodoService svc) => svc.Find(id));
```

The differences that matter:

| | Controllers | Minimal APIs |
|---|---|---|
| Structure | A class per group | One line per endpoint |
| Validation | Automatic with `[ApiController]` | **Manual** (Q10) |
| Filter pipeline | Full (N5) | Lighter endpoint filters |

**Which to use?** Minimal APIs for small services where you want less ceremony. Controllers for larger apps, where the automatic validation and grouping earn their keep. Neither is meaningfully faster, and you can mix both in one app.

---

### Q2. How do parameters get into a Minimal API handler?

**Answer.** The framework works out where each parameter comes from, based on its type and name. The order it checks:

1. **Route** — the name matches a `{token}` in the URL.
2. **Query string** — simple types like `int` or `string` not found in the route.
3. **Body** — one object type, read from the JSON. Only one allowed.
4. **Services** — anything registered in DI, no attribute needed.
5. **Special types** — `HttpContext`, `CancellationToken`, uploaded files.

```csharp
app.MapPost("/api/todos", (
    CreateTodo dto,           // object → JSON body
    ITodoService svc,         // registered in DI → injected
    CancellationToken ct) => svc.CreateAsync(dto, ct));
```

You can be explicit when the guess would be wrong, or when reading a header:

```csharp
app.MapGet("/api/todos", (
    [FromQuery] int page,
    [FromHeader(Name = "X-Tenant")] string tenant) => /* ... */);
```

The attributes available are `[FromRoute]`, `[FromQuery]`, `[FromHeader]`, `[FromBody]`, `[FromForm]`, and `[FromServices]`.

---

### Q3. What are endpoint groups (`MapGroup`)?

**Answer.** `MapGroup` applies a shared URL prefix and shared settings to several endpoints at once, so you don't repeat them.

```csharp
var todos = app.MapGroup("/api/todos")
    .RequireAuthorization()                  // applies to all three below
    .WithTags("Todos");

todos.MapGet("/{id:int}", GetTodo);          // → GET /api/todos/5
todos.MapPost("/", CreateTodo);              // → POST /api/todos
todos.MapDelete("/{id:int}", DeleteTodo);
```

Without the group you'd write `/api/todos` and `.RequireAuthorization()` on every line.

It's the Minimal API version of putting attributes on a controller class. Groups can also nest, which is how you build something like `/api/v1/todos`.

---

## N2 — Routing

### Q4. Attribute routing vs conventional routing?

**Answer.** Two ways to say which URL goes to which code.

**Conventional routing** defines one URL pattern centrally, and the parts of the URL map to class and method names:

```csharp
app.MapControllerRoute("default", "{controller=Home}/{action=Index}/{id?}");
```

**Attribute routing** puts the route directly on the method:

```csharp
[ApiController]
[Route("api/[controller]")]          // [controller] → "orders" from the class name
public class OrdersController : ControllerBase
{
    [HttpGet("{id:int}")]            // GET api/orders/5
    public IActionResult Get(int id) => /* ... */;

    [HttpGet("{id:int}/items")]      // GET api/orders/5/items
    public IActionResult Items(int id) => /* ... */;
}
```

**APIs use attribute routing**, and `[ApiController]` requires it. Two reasons: API URLs rarely line up with class and method names, and having the route next to the method means you can see what a URL does without hunting through a config file.

Conventional routing is still used for MVC web pages, where URLs do follow a predictable shape.

---

### Q5. What are route constraints?

**Answer.** A **constraint** limits what a route parameter accepts, written as `{param:constraint}`. If the value doesn't fit, the route doesn't match and the router tries the next one.

```csharp
[HttpGet("{id:int}")]        // matches /42, not /abc
[HttpGet("{code:guid}")]     // only well-formed GUIDs
[HttpGet("{id:int:min(1)}")] // matches /42, not /-1
```

Common ones: `int`, `guid`, `bool`, `datetime`, `alpha`, `min(n)`, `max(n)`, `range(a,b)`, `length(n)`, `regex(...)`.

They're useful when two routes could overlap:

```csharp
[HttpGet("products/{id:int}")]     // /products/42    → this one
[HttpGet("products/{slug}")]       // /products/shoes → this one
```

The more specific route wins — literal text beats a parameter, and a constrained parameter beats an unconstrained one.

❌ **A constraint is not validation.** `{id:int}` only means "this is a number". It can still be `-999` or an ID that doesn't exist, so business rules still belong in your handler.

---

## N3 — Model Binding & Validation (CORE)

### Q6. What is model binding, and where does the data come from?

**Answer.** **Model binding** is how ASP.NET Core fills your method parameters from the incoming request.

| Attribute | Reads from |
|---|---|
| `[FromRoute]` | the URL path |
| `[FromQuery]` | the query string |
| `[FromBody]` | the JSON body — only **one** per action |
| `[FromForm]` | form fields and file uploads |
| `[FromHeader]` | a request header |
| `[FromServices]` | DI, not the request |

With `[ApiController]` you rarely write these, because the guesses are sensible — objects come from the body, simple types from the route or query:

```csharp
[HttpPut("api/orders/{id:int}")]
public IActionResult Update(
    int id,                      // from route — the name matches {id}
    [FromQuery] bool notify,     // ?notify=true
    UpdateOrder body)            // from the JSON body — it's an object
{ /* ... */ }
```

One difference worth knowing: a `[FromBody]` object is deserialized in one go by the JSON serializer, while route, query, and form values are matched one property at a time.

---

### Q7. How does validation work with DataAnnotations and `ModelState`?

**Answer.** You put **attributes** on your model describing the rules. After binding, the framework checks them and records any failures in **`ModelState`**.

```csharp
public class CreateOrder
{
    [Required, StringLength(100)]
    public string Customer { get; set; } = default!;

    [Range(1, 1000)]
    public int Quantity { get; set; }

    [EmailAddress]
    public string? NotifyEmail { get; set; }
}
```

`ModelState.IsValid` is true only if binding **and** every rule passed. Without `[ApiController]` you check it yourself:

```csharp
[HttpPost]
public IActionResult Create(CreateOrder dto)
{
    if (!ModelState.IsValid) return BadRequest(ModelState);
    // ...
}
```

Common attributes: `[Required]`, `[Range]`, `[StringLength]`, `[MinLength]`/`[MaxLength]`, `[EmailAddress]`, `[RegularExpression]`, `[Compare]`, `[Url]`.

Note that validation runs **after** binding. So if someone sends `"abc"` where an `int` is expected, that failure is already in `ModelState` before your rules run.

---

### Q8. What does `[ApiController]` do automatically?

**Answer.** It turns on four conveniences so you write less code:

1. **Automatic 400 on invalid input.** If the model is invalid, the framework returns `400 Bad Request` **before your method runs**.
2. **Smarter binding.** Objects come from the body, simple types from route or query — no attributes needed.
3. **Standard error format.** Errors come back as `ProblemDetails` (Q14).
4. **Attribute routing required.**

Point 1 is the one interviewers look for:

```csharp
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    [HttpPost]
    public IActionResult Create(CreateOrder dto)
    {
        // ❌ not needed — [ApiController] already returned 400 if invalid
        // if (!ModelState.IsValid) return BadRequest(ModelState);

        return Ok(/* ... */);
    }
}
```

Writing that `ModelState` check alongside `[ApiController]` is dead code, and a sign someone doesn't know the attribute's behaviour.

You can change how the automatic 400 looks through `ApiBehaviorOptions` if you need a custom error body.

---

### Q9. How do you write custom validation?

**Answer.** Two options, depending on whether the rule covers one field or several.

**One field → a custom `ValidationAttribute`.** Reusable across models:

```csharp
public sealed class NotWeekendAttribute : ValidationAttribute
{
    protected override ValidationResult? IsValid(object? value, ValidationContext ctx)
    {
        if (value is DateTime d && d.DayOfWeek is DayOfWeek.Saturday or DayOfWeek.Sunday)
            return new ValidationResult("Weekends are not allowed.");

        return ValidationResult.Success;
    }
}

public class Booking { [NotWeekend] public DateTime Date { get; set; } }
```

**Several fields → `IValidatableObject`.** For rules that compare properties, which an attribute on one property can't do:

```csharp
public class DateRange : IValidatableObject
{
    public DateTime Start { get; set; }
    public DateTime End { get; set; }

    public IEnumerable<ValidationResult> Validate(ValidationContext ctx)
    {
        if (End < Start)                        // needs both fields
            yield return new ValidationResult("End must be after Start.",
                                              new[] { nameof(End) });
    }
}
```

For larger rule sets many teams use **FluentValidation**, which keeps the rules in a separate class instead of scattering attributes over the model.

---

### Q10. Why doesn't automatic validation work in Minimal APIs?

**Answer.** The automatic 400 belongs to the `[ApiController]` pipeline. Minimal APIs don't run that pipeline, so there's no `ModelState` and **your DataAnnotations attributes are ignored**.

That surprises people: the `[Required]` on your model does nothing, and invalid data reaches your handler.

The fix is an **endpoint filter** that checks the model and returns a 400 itself:

```csharp
public sealed class ValidationFilter<T> : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext ctx, EndpointFilterDelegate next)
    {
        var model = ctx.Arguments.OfType<T>().FirstOrDefault();

        if (model is not null)
        {
            var results = new List<ValidationResult>();
            if (!Validator.TryValidateObject(model, new(model), results, true))
                return Results.ValidationProblem(results.ToDictionary(   // ❌ 400
                    r => r.MemberNames.FirstOrDefault() ?? "",
                    r => new[] { r.ErrorMessage ?? "Invalid" }));
        }

        return await next(ctx);                                          // ✅ carry on
    }
}

app.MapPost("/api/orders", (CreateOrder dto) => Results.Ok())
   .AddEndpointFilter<ValidationFilter<CreateOrder>>();
```

Write it once and add it to a `MapGroup` (Q3) so every endpoint in the group gets it.

---

## N4 — Results & Status Codes

### Q11. What is `ActionResult` vs `IActionResult`?

**Answer.** Both are return types for a controller method.

- **`IActionResult`** is the interface. It says "this method returns some HTTP response" without saying which one. Every helper — `Ok()`, `NotFound()`, `BadRequest()` — returns something that implements it.
- **`ActionResult<T>`** is a generic type that means "either an HTTP response **or** a `T`". You can return both from the same method.

```csharp
// IActionResult — any status code, but the success type is invisible
public IActionResult Get(int id)
{
    var order = _repo.Find(id);
    if (order is null) return NotFound();     // 404
    return Ok(order);                          // 200 — but Swagger sees only "some response"
}

// ActionResult<T> — same freedom, and the type is declared ✅
public ActionResult<OrderDto> Get(int id)
{
    var order = _repo.Find(id);
    if (order is null) return NotFound();     // 404
    return order;                              // 200 — no need to wrap in Ok()
}
```

Two differences in the second version: you can `return order;` directly without `Ok(...)`, and the `OrderDto` in the signature tells Swagger and any generated client code what a success actually returns.

There's also a third option — **returning the type directly**:

```csharp
public OrderDto Get(int id) => _repo.Find(id);    // always 200
```

That works, but you can't return `NotFound()` from it, so it only suits endpoints that can't fail.

**Use `ActionResult<T>`** as your default. It's the only one of the three that gives you both any status code and a documented success type.

---

### Q12. Which status code helper do you use when?

**Answer.** `ControllerBase` gives you a helper for each status code. Picking the right one is what shows REST knowledge:

| Helper | Code | Use for |
|---|---|---|
| `Ok(obj)` | 200 | Success with a body |
| `CreatedAtAction(...)` | **201** | Successful POST that created something |
| `NoContent()` | 204 | Successful PUT/DELETE with nothing to return |
| `BadRequest()` | 400 | The request is malformed |
| `Unauthorized()` | 401 | Not logged in |
| `Forbid()` | 403 | Logged in, but not allowed |
| `NotFound()` | 404 | Doesn't exist |
| `Conflict()` | 409 | Clashes with current state — duplicate, concurrent edit |
| `UnprocessableEntity()` | 422 | Well-formed, but breaks a business rule |

```csharp
[HttpPost]
public ActionResult<OrderDto> Create(CreateOrder dto)
{
    var created = _svc.Create(dto);
    return CreatedAtAction(nameof(Get), new { id = created.Id }, created);  // 201 + Location
}

[HttpDelete("{id:int}")]
public IActionResult Delete(int id)
    => _svc.Delete(id) ? NoContent() : NotFound();                          // 204 or 404
```

The three most often confused:

- **201 vs 200** — a POST that creates something returns **201**, plus a `Location` header pointing at the new resource.
- **400 vs 422** — 400 means the request couldn't be read; 422 means it was read fine but breaks a rule.
- **401 vs 403** — 401 is "I don't know who you are"; 403 is "I know, and you can't."

---

### Q13. What are `Results` and `TypedResults` in Minimal APIs?

**Answer.** Minimal APIs return `IResult` instead of `IActionResult`. Two classes build them:

- **`Results`** — returns a general `IResult`. `Results.Ok(obj)`, `Results.NotFound()`.
- **`TypedResults`** — returns the **specific** type, like `Ok<OrderDto>`.

**Prefer `TypedResults`**, for two reasons:

```csharp
// Testing — you can check the type and value with no HTTP involved
var result = await GetOrder(5, svc);
var ok = Assert.IsType<Ok<OrderDto>>(result);
Assert.Equal("Bob", ok.Value!.Customer);
```

And Swagger picks up the status code and body type automatically.

When an endpoint can return more than one shape, declare them with `Results<T1, T2>`:

```csharp
app.MapGet("/api/orders/{id:int}",
    Results<Ok<OrderDto>, NotFound> (int id, IOrderService svc) =>
        svc.Find(id) is { } o ? TypedResults.Ok(o) : TypedResults.NotFound());
```

---

### Q14. What is `ProblemDetails`?

**Answer.** A **standard JSON shape for API errors**, defined by RFC 7807. Rather than every endpoint inventing its own error format, they all return the same fields:

```json
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.5.4",
  "title": "Not Found",
  "status": 404,
  "detail": "Order 42 does not exist",
  "traceId": "00-8fa1..."
}
```

The value is consistency: client code can read errors the same way from every endpoint, instead of special-casing each one.

`[ApiController]` already returns this for automatic 400s. To make **all** errors use it, including unhandled exceptions:

```csharp
builder.Services.AddProblemDetails();

var app = builder.Build();
app.UseExceptionHandler();     // a crash → 500 as ProblemDetails, not a stack trace
app.UseStatusCodePages();      // bare status codes → ProblemDetails

// Returning one yourself:
return Results.Problem(title: "Payment failed", statusCode: 402, detail: reason);
```

`ValidationProblemDetails` is the variant with a per-field `errors` list, which is what validation failures return.

---

## N5 — Filters

### Q15. What is a filter, what types are there, and in what order do they run?

**Answer.** A **filter** is a piece of code that runs around your controller action, so you can put shared logic — auth checks, logging, error handling — in one place instead of repeating it in every method.

What makes a filter different from middleware is **when** it runs: after routing has already chosen the action. So a filter knows which action was picked, and can see its bound arguments and `ModelState` (Q16).

**The five types, in the order they run:**

| Order | Type | Runs | Used for |
|---|---|---|---|
| 1 | **Authorization** | First of all | Blocking unauthorized requests |
| 2 | **Resource** | Before and after model binding | Caching, bailing out early |
| 3 | **Action** | Just before and after your method | Reading or changing arguments |
| 4 | **Exception** | When the action throws | Turning errors into responses |
| 5 | **Result** | Around turning your return value into a response | Adding headers, reshaping output |

Authorization comes first for a reason: there's no point binding the model or running the action if the caller isn't allowed in.

```csharp
public sealed class TimingFilter : IAsyncActionFilter
{
    public async Task OnActionExecutionAsync(
        ActionExecutingContext ctx, ActionExecutionDelegate next)
    {
        var sw = Stopwatch.StartNew();       // before the action

        var executed = await next();         // the action runs here

        sw.Stop();                           // after the action
        executed.HttpContext.Response.Headers["X-Elapsed-ms"]
            = sw.ElapsedMilliseconds.ToString();
    }
}
```

Like middleware, the "before" halves run outer to inner and the "after" halves run inner to outer.

---

### Q16. Filter or middleware — which do you use?

**Answer.** **Middleware** runs for every request, before anyone knows which endpoint will handle it. So it has no `ModelState` and no idea which action was chosen.

**Filters** run after the action is chosen, so they can see it.

**The rule:** needs to know about the specific action → filter. Applies to everything → middleware.

```csharp
// Middleware — every request, no action context
app.Use(async (ctx, next) => { /* logging, CORS, compression */ await next(ctx); });

// Filter — knows the action and its arguments
public class AuditFilter : IAsyncActionFilter { /* ... */ }
```

**Injecting services into a filter:** a filter used as a plain attribute can't take constructor dependencies. Use `[ServiceFilter]` (the filter comes from DI, so register it) or `[TypeFilter]` (created with DI, no registration needed):

```csharp
[ServiceFilter(typeof(AuditFilter))]
[HttpPost] public IActionResult Do() => Ok();
```

**Minimal APIs** use `IEndpointFilter` instead — lighter, added with `.AddEndpointFilter<T>()`, and it works on a `MapGroup` too. That's what Q10 uses for validation.

---

## N6 — Content Negotiation & JSON

### Q17. What is content negotiation?

**Answer.** The server picking a response format based on what the client asked for. The client sends an **`Accept`** header:

```
Accept: application/json    → JSON
Accept: application/xml     → XML, if the XML formatter is enabled
```

JSON is the only built-in format. XML is opt-in:

```csharp
builder.Services.AddControllers(o => o.ReturnHttpNotAcceptable = true)
                .AddXmlSerializerFormatters();
```

`ReturnHttpNotAcceptable = true` makes the server return **406 Not Acceptable** when it can't produce the requested format, instead of quietly sending JSON anyway.

On the way in, the **`Content-Type`** header tells the server how to read the request body.

Note this is a controller feature — **Minimal APIs always return JSON.**

---

### Q18. How do you configure `System.Text.Json`?

**Answer.** There are two places, because controllers and Minimal APIs use different paths:

```csharp
// Controllers
builder.Services.AddControllers().AddJsonOptions(o =>
{
    o.JsonSerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
    o.JsonSerializerOptions.Converters.Add(new JsonStringEnumConverter());
    o.JsonSerializerOptions.DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull;
});

// Minimal APIs
builder.Services.ConfigureHttpJsonOptions(o =>
{
    o.SerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
    o.SerializerOptions.Converters.Add(new JsonStringEnumConverter());
});
```

❌ **The gotcha:** configure the wrong one and nothing happens, with no error. If you mix both styles in one app, configure both.

The common settings: camelCase names (already the default), enums as text rather than numbers, and skipping nulls.

---

## N7 — API Versioning

### Q19. Why version an API, and what are the options?

**Answer.** So you can make breaking changes without breaking apps already using your API. You can't make every client update at once, so the old version keeps working while new clients use the new one.

Four ways to say which version you want:

| Style | Example | Trade-off |
|---|---|---|
| **URL** | `/api/v1/orders` | Clearest, easy to test in a browser. Most common. |
| **Query string** | `/api/orders?api-version=1.0` | Cleaner URLs, slightly less obvious |
| **Header** | `X-Api-Version: 1.0` | Clean URLs, hard to test by hand |
| **Media type** | `Accept: application/json;v=1.0` | Most "pure REST", least approachable |

**URL versioning is the usual choice** for public APIs because you can see the version just by looking at the request.

---

### Q20. How do you implement versioning?

**Answer.** With the `Asp.Versioning.*` packages. Register the service, then tag your controllers:

```csharp
builder.Services.AddApiVersioning(o =>
{
    o.DefaultApiVersion = new ApiVersion(1, 0);
    o.AssumeDefaultVersionWhenUnspecified = true;
    o.ReportApiVersions = true;            // tells clients which versions exist
});

[ApiController]
[ApiVersion("1.0", Deprecated = true)]     // still works, but warns clients
[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/orders")]
public class OrdersController : ControllerBase
{
    [HttpGet, MapToApiVersion("2.0")]
    public IActionResult GetV2() => Ok();
}
```

`ReportApiVersions = true` adds response headers listing the available and deprecated versions — the polite way to warn people before you remove one.

---

## N8 — OpenAPI / Swagger

### Q21. Swashbuckle vs the built-in OpenAPI support?

**Answer.** Two terms first: **OpenAPI** is a machine-readable description of your API, a JSON file. **Swagger UI** is a web page that turns that file into interactive docs.

**Swashbuckle** was the standard library for years and does both — generates the file and serves the UI.

**.NET 9 added a built-in generator**, `Microsoft.AspNetCore.OpenApi`. It produces the file but **includes no UI**:

```csharp
// Built-in (.NET 9) — generates the description only
builder.Services.AddOpenApi();
var app = builder.Build();
if (app.Environment.IsDevelopment())
    app.MapOpenApi();                  // serves /openapi/v1.json

// Swashbuckle — description + UI in one
builder.Services.AddSwaggerGen();
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}
```

So the modern setup is the built-in generator plus a separate UI (Swagger UI, Scalar). Swashbuckle is still fine, especially on .NET 8.

❌ **Serve the UI in development only** — publishing your whole API surface publicly is rarely what you want.

---

### Q22. How do you make the generated docs accurate?

**Answer.** The docs only know what you tell the framework. By default it can see the success type but not the error cases.

```csharp
// Controllers — declare every response
[HttpGet("{id:int}")]
[ProducesResponseType(typeof(OrderDto), StatusCodes.Status200OK)]
[ProducesResponseType(StatusCodes.Status404NotFound)]
public ActionResult<OrderDto> Get(int id) => /* ... */;

// Minimal APIs
app.MapGet("/api/orders/{id:int}", GetOrder)
   .Produces<OrderDto>(StatusCodes.Status200OK)
   .ProducesProblem(StatusCodes.Status404NotFound)
   .WithName("GetOrder")
   .WithTags("Orders");
```

You get some of this free — `ActionResult<T>` and `TypedResults` (Q13) supply the success type on their own. Turning on `<GenerateDocumentationFile>` also turns your `///` comments into descriptions.

**Why bother?** The description **is** the contract. When it's accurate, other teams can generate typed client code from it, run contract tests, and read docs that match reality. When it's wrong, all three quietly break.

---

## N9 — Error Handling & Standards

### Q23. What is the RFC 7807 `ProblemDetails` standard, and how does the `IExceptionHandler` interface in .NET 8 provide global typed exception handling for APIs?

**Answer.** **RFC 7807 (`ProblemDetails`)** is an IETF machine-readable standard format for specifying errors in HTTP API responses. Instead of returning raw string messages or HTML stack traces, APIs return a consistent JSON schema:

```json
{
  "type": "https://errors.example.com/not-found",
  "title": "Resource Not Found",
  "status": 404,
  "detail": "Order '1042' was not found.",
  "instance": "/api/orders/1042"
}
```

#### Global Exception Handling with `IExceptionHandler` (.NET 8+)
Before .NET 8, global error handling required custom middleware (`app.Use(...)`) or MVC Exception Filters. .NET 8 introduced **`IExceptionHandler`**, a clean, typed DI interface that integrates natively with `app.UseExceptionHandler()`.

```csharp
public class GlobalExceptionHandler(ILogger<GlobalExceptionHandler> logger) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        logger.LogError(exception, "Unhandled exception occurred: {Message}", exception.Message);

        var (statusCode, title) = exception switch
        {
            KeyNotFoundException => (StatusCodes.Status404NotFound, "Resource Not Found"),
            UnauthorizedAccessException => (StatusCodes.Status401Unauthorized, "Unauthorized Access"),
            ArgumentException => (StatusCodes.Status400BadRequest, "Bad Request"),
            _ => (StatusCodes.Status500InternalServerError, "Internal Server Error")
        };

        var problemDetails = new ProblemDetails
        {
            Status = statusCode,
            Title = title,
            Detail = exception.Message,
            Instance = httpContext.Request.Path
        };

        httpContext.Response.StatusCode = statusCode;
        await httpContext.Response.WriteAsJsonAsync(problemDetails, cancellationToken);

        // Return true to signal that this exception has been handled completely
        return true;
    }
}

// Registration in Program.cs:
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails(); // Configures framework-level ProblemDetails formatting

var app = builder.Build();

app.UseExceptionHandler(); // Enables built-in exception handler pipeline using registered IExceptionHandlers
```

