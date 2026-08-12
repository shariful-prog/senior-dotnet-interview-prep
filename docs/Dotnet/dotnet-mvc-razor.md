# N2. ASP.NET Core MVC (Views & Razor)

---

## V1 — The MVC Pattern

### Q1. What are the three parts of MVC, and where does the ViewModel fit?

**Answer.** **MVC** splits a page into three jobs:

- **Model** — the data and its rules. Doesn't know a web page exists.
- **View** — the HTML template. Only displays things; makes no decisions.
- **Controller** — takes the request, gets the data, picks the view.

The **ViewModel** is the part interviewers ask about, because it isn't one of the three letters. It's a class shaped for **one screen** — exactly the fields that page needs.

```csharp
// ❌ passing the database entity straight to the view
public IActionResult Details(int id) => View(_db.Users.Find(id));   // exposes PasswordHash!

// ✅ a ViewModel with only what the screen shows
public sealed record UserDetailsVm(string DisplayName, string Email, int OrderCount);

public async Task<IActionResult> Details(int id)
{
    var vm = await _db.Users
        .Where(u => u.Id == id)
        .Select(u => new UserDetailsVm(u.DisplayName, u.Email, u.Orders.Count))
        .FirstOrDefaultAsync();

    return vm is null ? NotFound() : View(vm);
}
```

Two reasons this matters. **Security** — an entity exposes fields the user should never see or set (that's over-posting, Q12). **Performance** — projecting inside the query means the database returns three columns instead of whole tables.

Keep controllers **thin**: get data, pick a result. Business logic belongs in a service, where it can be tested without the MVC machinery.

---

### Q2. How does MVC differ from Razor Pages?

**Answer.** Same framework, same routing and DI — different code organisation.

**MVC** groups many endpoints in one controller, with views in a separate folder tree. `UserController` handles `Index`, `Details`, `Edit`, `Delete`, and their views live in `Views/User/`.

**Razor Pages** puts one page's code and markup together: `Index.cshtml` plus `Index.cshtml.cs`. Handlers are named by verb instead of routed by attribute:

```csharp
public class EditModel : PageModel
{
    [BindProperty] public UserEditVm Input { get; set; } = new();

    public async Task OnGetAsync(int id) => Input = await LoadAsync(id);

    public async Task<IActionResult> OnPostAsync()
    {
        if (!ModelState.IsValid) return Page();
        await SaveAsync(Input);
        return RedirectToPage("Index");
    }
}
```

Microsoft recommends **Razor Pages for new page-based apps**, MVC when many actions genuinely share one controller. Both mix freely with API controllers in one app.

❌ **Interview trap:** "Razor Pages replaced MVC." No — Razor Pages is *built on* MVC and uses the same binding, validation, and Razor engine. Different organisation, not a successor.

---

### Q3. What happens between a request arriving and HTML going out?

**Answer.** For `GET /Products/Details/5`:

1. **Middleware** runs — HTTPS redirect, static files, then routing.
2. **Routing** matches the URL to `ProductsController.Details` and stores `id = 5`.
3. **Auth** runs — after routing, so `[Authorize]` on the action can be read.
4. **The controller is created**, with its dependencies from the request's DI scope.
5. **Model binding** fills the parameters from route, query, and body.
6. **Validation** runs and fills `ModelState` — but does **not** stop the action.
7. **Filters** run around the action.
8. **The action returns an `IActionResult`** — a *description* of what to do, not the response.
9. **The result executes** — the view engine finds the `.cshtml`, renders it, writes the HTML.
10. **The response unwinds** back out through filters and middleware.

**The step people miss is 8 vs 9.** `return View(vm)` renders nothing — it returns a `ViewResult`, and rendering happens later.

That matters: the view runs **after** your action returned, so a `DbContext` you disposed is already gone by then, and a crash mid-render happens after the headers are sent — so no clean error page (Q14).

---

## V2 — Razor Syntax

### Q4. How does Razor prevent XSS?

**Answer.** Razor **HTML-encodes every `@` expression automatically**. If `Model.Name` contains `<script>alert(1)</script>`, the browser receives escaped text and displays it as characters instead of running it.

That one default removes the most common XSS hole in server-rendered apps.

```cshtml
@Model.Bio              @* ✅ encoded — safe *@
@Html.Raw(Model.Bio)    @* ❌ NOT encoded — raw HTML injected as-is *@
```

`Html.Raw` is only safe when **you** wrote the string, or it has been through a real HTML sanitizer. "An admin typed it" is not a sanitizer — a compromised admin account then owns every visitor's session.

❌ **The subtle one:** HTML encoding is the **wrong** encoding inside a `<script>` block.

```cshtml
<script>var user = "@Model.Name";</script>          <!-- ❌ a quote breaks out -->
<script>var user = @Json.Serialize(Model.Name);</script>   <!-- ✅ -->
```

Encoding has to match the context you're injecting into. Razor's default protects HTML, not JavaScript, URLs, or CSS. Add a Content-Security-Policy header so a missed spot still can't run script ([dotnet-security.md](dotnet-security.md)).

---

## V3 — Passing Data to Views

### Q5. Compare `ViewData`, `ViewBag`, `TempData`, and a strongly-typed model.

**Answer.** Four ways to get data into a view. Only one is a good default.

| | Type | Lives for | Compile-checked |
|---|---|---|---|
| **Model** | Your class | This request | ✅ Yes |
| **`ViewData`** | String-keyed dictionary | This request | ❌ No |
| **`ViewBag`** | `dynamic` over `ViewData` | This request | ❌ No |
| **`TempData`** | Dictionary (cookie/session) | **Until read, across one redirect** | ❌ No |

**Use a strongly-typed model** for the page's data — you get IntelliSense and compile errors instead of a blank spot in the HTML after a rename.

`ViewData` and `ViewBag` are the **same storage** with two faces, so `ViewData["Title"]` and `ViewBag.Title` are one slot. Fine for incidental things like a page title the layout reads.

**`TempData` is the one interviewers probe:** it survives **one redirect**, which is exactly what a "Saved successfully" message needs after a POST:

```csharp
[HttpPost]
public async Task<IActionResult> Edit(UserEditVm vm)
{
    if (!ModelState.IsValid) return View(vm);
    await _svc.SaveAsync(vm);

    TempData["Flash"] = "Saved.";              // survives the redirect
    return RedirectToAction(nameof(Index));
}
```

The mechanism: entries are **deleted when read**. `Peek` reads without consuming, `Keep` makes it survive another request. Values must be serializable, so store strings and IDs, not objects.

---

### Q6. What is session state, and how do you set it up?

**Answer.** **Session** is server-side storage for one visitor that survives across their requests. The browser only holds a cookie with an **opaque ID** — the data itself stays on the server. That's the difference from a cookie, where the data travels with every request.

Unlike old ASP.NET, **session is off by default**. Three pieces to turn it on:

```csharp
builder.Services.AddDistributedMemoryCache();   // 1. the store (dev only — see Q8)

builder.Services.AddSession(o =>                // 2. session services
{
    o.IdleTimeout = TimeSpan.FromMinutes(20);
    o.Cookie.HttpOnly = true;                   // JS can't read the session ID
    o.Cookie.IsEssential = true;                // survives GDPR consent gating
});

var app = builder.Build();
app.UseRouting();
app.UseSession();          // 3. BEFORE anything that reads session
app.MapControllers();
```

Session is built on **`IDistributedCache`**, so it has no storage of its own — it borrows the caching abstraction. That's what makes Q10's fix a one-line change.

**The API only stores `byte[]`**, with wrappers for strings and ints:

```csharp
HttpContext.Session.SetString("Theme", "dark");
HttpContext.Session.SetInt32("CartId", 42);

var theme = HttpContext.Session.GetString("Theme");   // null if absent
HttpContext.Session.Remove("Theme");
```

For objects, serialize to JSON yourself — `SetString(key, JsonSerializer.Serialize(value))`.

Two behaviours that surprise people. **Session loads lazily** — no cookie is issued until you actually write a key, so anonymous visitors don't each get a session. And **`IdleTimeout` is sliding**, not absolute: 20 minutes of *inactivity* expires it, but an active user's session lives indefinitely.

❌ **Ordering gotcha:** `UseSession()` must come **before** anything that touches `HttpContext.Session`, or you get `InvalidOperationException: Session has not been configured`.

---

### Q7. When do you use session versus `TempData`, a cookie, a cache, or the database?

**Answer.** All five outlive one request. Picking wrong is what interviewers look for.

| | Stored | Lives for | Good for |
|---|---|---|---|
| **`TempData`** | Cookie or session | **One redirect** | Flash messages (Q7) |
| **Session** | Server | Sliding timeout | Wizard step, anonymous cart |
| **Cookie** | Browser | Whatever you set | Small prefs — theme, locale |
| **Cache** | Server, shared | TTL, evictable | Expensive derived data |
| **Database** | Disk | Forever | Anything that must survive |

**Good session use:** a multi-step wizard, an **anonymous** cart, a "return to this URL after login" value.

**Use the database instead** when the data matters. A signed-in user's cart belongs in the database — they expect it on their phone too. Session data disappears on timeout, on eviction, and on restart. **It is not durable storage**, even on Redis.

❌ **Never put security state in session** — no `Session["IsAdmin"] = true`. Authorization belongs in claims, checked per request ([dotnet-auth-logging.md](dotnet-auth-logging.md)).

The senior framing: **session is convenient and quietly expensive.** Every read is a serialize plus a network hop, and it makes your app stateful, which complicates scaling (Q8). Modern APIs and SPAs mostly avoid it — state lives in the client or the database. A design leaning hard on session is worth questioning.

---

### Q8. What breaks with session when you scale to multiple servers?

**Answer.** `AddDistributedMemoryCache()` — the line every tutorial shows — stores sessions **in that process's memory**, despite the name. One server: fine. Three servers behind a load balancer:

- The first request hits server A, which creates the session in A's memory.
- The next request goes to server B, which finds nothing and returns an **empty session**.
- The cart empties at random, the wizard resets, logins seem to drop.

It's intermittent and load-dependent, so it usually passes testing and fails in production. The same thing happens on a **single** server at every deploy or app-pool recycle — memory is gone, so all sessions vanish.

**The fix — a real shared store.** Because session sits on `IDistributedCache` (Q8), only this registration changes:

```csharp
builder.Services.AddStackExchangeRedisCache(o =>        // ✅ shared by all instances
{
    o.Configuration = builder.Configuration.GetConnectionString("Redis");
});
builder.Services.AddSession(/* unchanged */);
```

SQL Server (`AddDistributedSqlServerCache`) also works if you'd rather not add Redis.

⚠️ **Sticky sessions are a crutch.** Pinning each user to one instance hides the problem, but you get uneven load, sessions still die when that instance recycles, and you can't scale elastically. A stopgap, not a design.

One related trap: **Data Protection keys default to per-machine storage.** Auth cookies and antiforgery tokens are encrypted with them, so instance B can't decrypt what A issued — you get random logouts and "antiforgery token could not be decrypted". Fix with a shared key ring plus `SetApplicationName` ([dotnet-security.md](dotnet-security.md)).

---

### Q9. What's the difference between a view, a partial view, and a View Component?

**Answer.** Three ways to render HTML, differing in whether they bring their own logic and whether they include the layout.

**A view** (`View("Name", model)`) is a full page. It runs `_ViewStart`, so it gets the layout.

**A partial view** is a reusable fragment. It does **not** run `_ViewStart`, so **no layout** — which is exactly what an AJAX call needs. It has no logic of its own; you hand it everything.

```cshtml
<partial name="_OrderRow" model="order" />
```

**A View Component** is a partial *with* its own class, DI, and async work. Use it when the fragment fetches its own data — a cart summary, a nav tree.

```csharp
public class CartSummary(ICartService carts) : ViewComponent
{
    public async Task<IViewComponentResult> InvokeAsync()
        => View(await carts.GetForUserAsync(UserClaimsPrincipal));
    // → Views/Shared/Components/CartSummary/Default.cshtml
}
```
```cshtml
<vc:cart-summary />
```

**The rule:** needs its own data → View Component. Just renders what you pass → partial.

❌ The anti-pattern is a partial that uses `@inject` to query a `DbContext`. That puts a database call inside rendering, after the action returned, where you can't test it or handle its failure (Q3). That's a View Component's job.

---

### Q10. How does `asp-for` decide the `name`, and why does it matter?

**Answer.** `asp-for="Address.City"` emits `name="Address.City"` and `id="Address_City"`. The **`name`** is what matters: it's the key the browser posts, and model binding uses that exact path to rebuild your object on the way back.

That's why the names must line up. For collections, the index is part of the name:

```cshtml
@for (var i = 0; i < Model.Lines.Count; i++)
{
    <input asp-for="Lines[i].Sku" />   @* → name="Lines[0].Sku", name="Lines[1].Sku" *@
}
```

❌ **The classic bug: using `@foreach` instead of `@for`.** There's no index, so every input renders as `name="Sku"`. The collection posts back as `null` or one mangled item — and it fails **silently**.

One more: `asp-for` takes its value from **`ModelState` first**, then the model. That's deliberate — after a failed POST you want the user's rejected input back, not the old database value. It's also why "the field won't update after I changed it in the action" happens; `ModelState.Remove("Field")` is the fix.

---

## V5 — Forms & Security

### Q11. Why redirect after a successful POST?

**Answer.** This is **POST-Redirect-GET (PRG)**, the standard shape of every form action:

```csharp
[HttpPost]
public async Task<IActionResult> Create(CreateUserVm vm)
{
    if (!ModelState.IsValid) return View(vm);              // ✅ failure → same view
    var id = await _svc.CreateAsync(vm);

    TempData["Flash"] = "User created.";
    return RedirectToAction(nameof(Details), new { id });  // ✅ success → redirect
}
```

Three problems it solves. **Double submission** — render HTML straight from the POST and the browser's URL is still the POST endpoint, so F5 creates a second record. **A meaningful URL** — the user lands on `/Users/Details/42`, which is bookmarkable. **Back-button sanity** — no stale POST in history.

Note the asymmetry, which is what people get wrong: **success redirects, failure returns the view.** On failure you must `return View(vm)` so the form keeps the user's input and your validation messages render. `ModelState` does **not** survive a redirect — which is exactly why `TempData` exists for the success message.

---

### Q12. What is over-posting, and how do you prevent it?

**Answer.** **Over-posting** is when an attacker adds fields to a form POST that your form never rendered, and model binding sets them anyway. Binding matches by **name** — it has no idea which inputs your view actually drew.

```csharp
public class User { public int Id; public string Email; public bool IsAdmin; }

[HttpPost]
public IActionResult Edit(User user)      // ❌ binding the entity directly
{
    _db.Update(user);
    _db.SaveChanges();                     // attacker posts IsAdmin=true → they're an admin
}
```

Your form showed one email box. The attacker adds `IsAdmin=true` with any HTTP tool and it binds. This is one of the most common real vulnerabilities in MVC codebases.

**The fix: bind to a ViewModel with only the fields you allow** (Q1). If `IsAdmin` isn't on the ViewModel, there's nothing to over-post:

```csharp
public sealed class UserEditVm { public int Id { get; set; } public string Email { get; set; } = ""; }

[HttpPost]
public async Task<IActionResult> Edit(UserEditVm vm)
{
    if (!ModelState.IsValid) return View(vm);

    var user = await _db.Users.FindAsync(vm.Id);
    if (user is null) return NotFound();

    user.Email = vm.Email;                 // ✅ copy only what you meant to allow
    await _db.SaveChangesAsync();
    return RedirectToAction(nameof(Index));
}
```

`[BindNever]` and `TryUpdateModelAsync` are alternatives, but they're easy to forget when someone adds a property later.

❌ **What does *not* fix it:** anti-forgery tokens (that's CSRF, a different attack — Q13), or simply not rendering the input. Binding is name-based; invisibility is not protection.

---

### Q13. Explain CSRF and how anti-forgery tokens work.

**Answer.** **CSRF** is: your user is logged in, visits a malicious page, and that page submits a form to your site. The browser **automatically attaches their cookies**, so your server sees a fully authenticated request the user never intended.

The defence is a secret the attacker's page can't read: one value in a cookie, a matching one in the form. A cross-site page can make the browser *send* your cookie, but it can't *read* your cookies or HTML, so it can't produce the form field.

```cshtml
<form asp-action="Delete" method="post">   @* ✅ token injected automatically *@
    <button>Delete</button>
</form>
```

The specifics that get asked:

- **The `form` Tag Helper adds the token automatically** for any non-GET form.
- **Injection is automatic; validation is not.** Use **`[AutoValidateAntiforgeryToken]` globally** so every POST/PUT/DELETE is checked. That beats per-action attributes, where forgetting one is a silent hole.
- **AJAX must send it explicitly** as the `RequestVerificationToken` header — there's no form to carry it.
- **GET must stay safe.** A token protects nothing if `GET /Users/Delete/5` deletes a user; an `<img src>` anywhere triggers it.
- **`SameSite=Lax`** (the modern default) blocks much of this, but isn't complete. Keep the token.

Worth saying out loud: **cookie-authenticated endpoints need CSRF protection; `Authorization: Bearer` endpoints don't**, because the browser never attaches that header on its own.

---

## V7 — Performance

### Q14. What are the main performance pitfalls in MVC views?

**Answer.** The pitfall unique to views is that **rendering happens after your action returned** (Q3), so work that sneaks into a view runs where you have no error handling and no tests.

**1. Queries in the view.** The worst is lazy loading — `@foreach (var line in Model.Order.Lines)` fires a query per iteration, an **N+1** that never appears in the controller ([dotnet-ef-core.md](dotnet-ef-core.md)). If the action disposed the `DbContext`, you get an `ObjectDisposedException` **mid-response**, so the exception handler can't even show a clean error page. Fix: project into a ViewModel in the action.

**2. Sync-over-async.** `@someTask.Result` blocks a thread-pool thread inside rendering. Razor supports `await` — use it, or better, do the work in the action.

**3. Not caching what doesn't change.** Nav menus and category trees get rebuilt every request. Use the `<cache>` Tag Helper, or **output caching** (.NET 7+, the modern option).

⚠️ **Watch the `vary-by` keys** — a fragment cached without `vary-by-user` will show one user's data to another. That's a security bug wearing a performance costume.

**4. Partials in tight loops.** Each render re-enters the view engine. Fine for a dozen rows, measurable for thousands.

The highest-value habit: **make the action produce a fully-loaded ViewModel**, so the view can only do string interpolation. Everything above follows from that.
