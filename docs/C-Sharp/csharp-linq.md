# E. LINQ & Functional
---

## E1 — LINQ Fundamentals (Query vs Method Syntax)

### Q1. What is LINQ, and what is it actually built on?

**Answer.** **LINQ** (Language Integrated Query) lets you **query and transform data** using the same readable syntax, no matter where the data lives — a `List` in memory, a database, an XML file, etc.

Under the hood there's nothing magic: the LINQ operations you use (`Where`, `Select`, `OrderBy`, `GroupBy`, …) are just **extension methods** — regular methods that "attach" onto collections. You get access to them with `using System.Linq;`.

```csharp
using System.Linq;

int[] nums = { 1, 2, 3, 4, 5 };
var evens = nums.Where(n => n % 2 == 0)   // keep even numbers
                .Select(n => n * n);       // square them → 4, 16
```

Because they're just methods you can chain together, you can build up a query step by step, and the same query can run in memory *or* be turned into SQL for a database — depending on the data source.

---

### Q2. What is the difference between query syntax and method (fluent) syntax?

**Answer.** They are **two ways of writing the exact same thing**. **Query syntax** looks like SQL (`from … where … select`). **Method syntax** (also called fluent syntax) chains the methods directly with lambdas. The compiler actually **converts query syntax into method syntax** behind the scenes, so there's **no performance difference** — it's purely a style choice.

```csharp
// Query syntax — reads like SQL
var q = from n in nums
        where n % 2 == 0
        orderby n descending
        select n * n;

// Method syntax — the same thing, chained methods
var m = nums.Where(n => n % 2 == 0)
            .OrderByDescending(n => n)
            .Select(n => n * n);
```

One practical difference: **method syntax can do more**. Query syntax only covers common operations (`where`, `select`, `join`, `group`, `orderby`). Things like `Count`, `First`, `Any`, `Distinct`, `Take`, `ToList` have **no query-syntax keyword**, so you end up using method calls anyway. Most teams just standardize on method syntax; query syntax is nicest for complex **joins and grouping**, where it reads more clearly.

---

### Q3. What do `let` and `into` do in query syntax?

**Answer.**
- **`let`** creates a **temporary named value** inside the query so you can compute something once and reuse it (instead of writing it multiple times).
- **`into`** takes the result of a `select` or `group` and **feeds it into a new stage** of the query, so you can keep querying the reshaped result.

```csharp
var query = from w in words
            let len = w.Length          // compute length once, reuse it below
            where len > 3
            orderby len
            select $"{w} ({len})";

// `into` continues the query after grouping
var byFirst = from w in words
              group w by w[0] into g    // g = each group; keep querying it
              where g.Count() > 1
              select new { Letter = g.Key, Count = g.Count() };
```

Think of `let` as "save this in a variable for later in the query," and `into` as "now continue working with the result so far."

---

## E2 — `IEnumerable` vs `IQueryable`

### Q4. What is the difference between `IEnumerable<T>` and `IQueryable<T>`?

**Answer.** The main difference is **where the query is executed**.

#### `IEnumerable<T>`

- Works with **in-memory collections**.
- Query runs **in the application memory**.
- Data is already loaded from the source.

```csharp
var users = db.Users.ToList();

var result = users.Where(x => x.Age > 18);
```

Flow:

```
Database → Load all users → Filter in C# memory
```

#### `IQueryable<T>` Example (EF Core)

- Works with **data sources like databases**.
- Query is built as an **expression tree**.
- The query is translated and executed by the data source (e.g., SQL Server).

Example:

```csharp
var result = db.Users
               .Where(x => x.Age > 18)
               .ToList();
```

Flow:

```
C# LINQ → SQL Query → Database filters data → Return result
```

#### Why Does It Matter?

Suppose the `Employees` table has **1,000,000** rows.

##### Using `IEnumerable<T>`

```csharp
var employees = dbContext.Employees.ToList(); // Loads all rows

var result = employees.Where(e => e.Age > 30);
```

- Loads all 1,000,000 rows into memory.
- Then filters them in C#.
- More memory usage and slower performance.

##### Using `IQueryable<T>`

```csharp
var result = dbContext.Employees.Where(e => e.Age > 30);
```

- SQL Server filters the data.
- Only matching rows are returned.
- Less memory usage and better performance.

---

### Q5. What is the classic "premature `ToList()` / client vs server evaluation" bug?

**Answer.** This is one of the most common database performance bugs. If you call `.ToList()` (or `AsEnumerable()`) **too early**, the query stops being a database query and everything after it runs **in your app's memory**. So filtering and paging that *should* happen in SQL instead pull the **entire table** into your app first.

```csharp
// BUG: ToList() here runs "SELECT * FROM Orders" — loads the WHOLE table,
//      then filters and takes 10 in memory. Disaster on a big table.
var recent = db.Orders
    .ToList()                              // loads EVERYTHING here
    .Where(o => o.CustomerId == id)
    .Take(10);

// FIX: filter and page FIRST (in SQL), call ToList() LAST
var recent = db.Orders
    .Where(o => o.CustomerId == id)        // becomes SQL WHERE
    .Take(10)                              // becomes SQL TOP/LIMIT
    .ToList();                             // one efficient query
```

**Rule:** do all your filtering, sorting, and paging *before* calling `.ToList()`. Call `.ToList()` **last**, once, so the database returns only what you need.

---

### Q6. What happens if you use an unsupported method inside a database query?

**Answer.** When you write a database (`IQueryable`) query, EF Core has to translate your code into SQL. If you use something it **can't translate** — like a custom C# helper method — it **throws an error** telling you it "could not be translated."

```csharp
// Throws: EF Core doesn't know how to turn Format() into SQL
var q = db.Users.Where(u => Format(u.Name) == "x").ToList();

// Fix: do what SQL can in the database, then switch to memory for the rest
var q = db.Users
    .Where(u => u.IsActive)            // this part runs as SQL
    .AsEnumerable()                    // from here on, runs in memory
    .Where(u => Format(u.Name) == "x") // now the custom method is fine
    .ToList();
```

`AsEnumerable()` marks the point where "database work" ends and "in-memory C# work" begins. Just be aware: everything after it is loaded into memory, so filter as much as you can in SQL first.

> **Note:** older EF Core versions used to *silently* run the untranslatable part in memory, which caused hidden performance problems. Newer versions throw instead, so you notice.

---

### Q7. What is the difference between streaming and buffering operators?

**Answer.** 
### Streaming Operators

- Process elements **one by one**.
- Do not need to load the entire collection into memory.
- Can start returning results immediately.

Examples:

```csharp
.Where()
.Select()
.Take()
```

Example:

```csharp
var result = numbers.Where(x => x > 10);
```

It processes each item as it is requested. 

### Buffering Operators

- Need to read the **entire collection first**.
- Store data in memory before returning results.

Examples:

```csharp
.OrderBy()
.GroupBy()
.ToList()
.ToArray()
```

Example:

```csharp
var result = numbers.OrderBy(x => x);
```

It must see all numbers before it can sort them.

---

## E3 — Common Operators (Grouping, Joins, Projections)

### Q1. Explain `Select` vs `SelectMany`.

**Answer.**
- **`Select`** transforms each item into **one** result (one in, one out).
- **`SelectMany`** is for when each item produces a **whole list**, and you want all those lists **flattened into one**.

```csharp
var people = new[]
{
    new { Name = "Ann", Pets = new[] { "cat", "dog" } },
    new { Name = "Bob", Pets = new[] { "fish" } },
};

// Select → a list of lists (usually NOT what you want)
var nested = people.Select(p => p.Pets);        // [ ["cat","dog"], ["fish"] ]

// SelectMany → one flat list
var flat = people.SelectMany(p => p.Pets);      // "cat", "dog", "fish"
```

Think of `SelectMany` as "get all the pets from everyone, as a single combined list."

---

### Q2. Explain `GroupBy` — what does it return?

**Answer.** `GroupBy` **sorts items into buckets** based on a key. You get back a set of groups; each group has a **`.Key`** (what they have in common) and contains **all the items** in that bucket.

```csharp
var byCategory = products
    .GroupBy(p => p.Category)          // bucket products by category
    .Select(g => new
    {
        Category = g.Key,             // the category name
        Count = g.Count(),            // how many in this category
        Total = g.Sum(p => p.Price)   // total price in this category
    });
```

Unlike SQL's `GROUP BY` (which only gives you summary numbers), LINQ's `GroupBy` **keeps the actual items** in each group, so you can list them out if you want, not just count/sum them.

---

### Q3. What is the difference between `Join` and `GroupJoin`?

**Answer.**
- **`Join`** is like an SQL inner join — it matches items from two lists and gives you **one flat row per match**.
- **`GroupJoin`** gives you **one row per item on the left**, each carrying a **list** of its matches on the right (useful for "left join" style results where you want everyone, even with no matches).

```csharp
// Join — one row per matching pair (customers WITH orders only)
var flat = customers.Join(orders,
    c => c.Id, o => o.CustomerId,
    (c, o) => new { c.Name, o.Total });

// GroupJoin — each customer once, with all their orders grouped together
var grouped = customers.GroupJoin(orders,
    c => c.Id, o => o.CustomerId,
    (c, os) => new { c.Name, Orders = os });
```

In real EF Core code you usually don't write joins by hand — you use **navigation properties** like `customer.Orders`. But interviewers like the flat-vs-grouped distinction.

---

### Q4. When do `First`, `FirstOrDefault`, `Single`, and `SingleOrDefault` throw?

**Answer.** They differ in **how many items they expect** and **what they do when there are none**:

| Operator | No matches | One match | Two or more matches |
|---|---|---|---|
| `First` | **throws** | returns it | returns the **first** |
| `FirstOrDefault` | returns default (null/0) | returns it | returns the **first** |
| `Single` | **throws** | returns it | **throws** (expects exactly one) |
| `SingleOrDefault` | returns default | returns it | **throws** |

```csharp
users.First(u => u.Id == id);           // throws if none found
users.FirstOrDefault(u => u.Id == id);  // returns null if none (remember to null-check!)
users.Single(u => u.Id == id);          // throws if none OR if more than one
```

**How to choose:**
- Use **`Single`/`SingleOrDefault`** when there should be **exactly one** result (like looking up by primary key) — it'll catch bugs if duplicates sneak in.
- Use **`First`/`FirstOrDefault`** when "just give me the first match" is fine.
- The `...OrDefault` versions return `null` (for objects) or `0`/`false` (for numbers/bools) instead of throwing.

---

### Q5. Summarize the other everyday operators (`Any`/`All`/`Count`, `Distinct`, `Skip`/`Take`, `Aggregate`).

**Answer.**

- **`Any()`** — "is there at least one?" Prefer **`Any()` over `Count() > 0`** — `Any()` stops as soon as it finds one item, while `Count()` walks through everything.
- **`All(condition)`** — "do *all* items match?"
- **`Count()`** — how many items.
- **`Distinct()`** — removes duplicates. **`DistinctBy(x => x.Email)`** removes duplicates by a specific field.
- **`Skip(n)` / `Take(n)`** — for paging: `Skip((page-1)*size).Take(size)`.
- **`Aggregate`** — a general "combine everything into one value" tool. Usually you'd reach for the simpler `Sum`/`Min`/`Max`/`Average` first; use `Aggregate` for custom combining.

```csharp
bool hasAdmin = users.Any(u => u.IsAdmin);              // stops at first admin
var page = query.Skip(20).Take(10);                     // page 3 of 10
var unique = people.DistinctBy(p => p.Email).ToList();  // one per email
var product = new[] { 1, 2, 3, 4 }.Aggregate((a, b) => a * b);  // 1*2*3*4 = 24
```

---

### Q6. What is projection and why does projection matter for databases?

**Answer.** **Projection** means **selecting only the columns you need** instead of retrieving the entire object. In LINQ, projection is done using the `Select()` method.

```csharp
var employees = dbContext.Employees
    .Select(e => new
    {
        e.Id,
        e.Name
    })
    .ToList();
```

EF Core generates SQL similar to:

```sql
SELECT Id, Name
FROM Employees;
```

Instead of:

```sql
SELECT *
FROM Employees;
```

#### Why Does It Matter?

- Improves query performance
- Reduces data transferred from the database
- Uses less memory
- Returns only the required data

---

## E4 — Delegates (`Func` / `Action` / `Predicate`)

### Q1. What is a delegate?

**Answer.** A **delegate** is a variable that holds a **method** instead of a value. You declare the shape it accepts — the parameter types and the return type — and it can then hold any method of that shape.

Here it lets `OrderService` run something after placing an order, without knowing what that something is:

```csharp
using System;

// The shape: takes a string, returns nothing
public delegate void OrderAction(string orderId);

class OrderService
{
    public void PlaceOrder(string orderId, OrderAction action)
    {
        Console.WriteLine($"Order {orderId} placed.");

        action(orderId);          // run the method that was passed in
    }
}

class EmailService
{
    public void SendConfirmationEmail(string orderId)
    {
        Console.WriteLine($"Confirmation email sent for order {orderId}");
    }
}

class Program
{
    static void Main()
    {
        var orderService = new OrderService();
        var emailService = new EmailService();

        // Pass the method itself — note there are no parentheses after it
        orderService.PlaceOrder("ORD101", emailService.SendConfirmationEmail);
    }
}
```

```text
Order ORD101 placed.
Confirmation email sent for order ORD101
```

**Two things to notice.**

`emailService.SendConfirmationEmail` has **no parentheses**. With `()` you would call the method and pass its result; without them you pass the method itself.

`OrderService` has **no reference to `EmailService`**. It does not know emails exist — it only knows it was handed a method taking a `string`.

**Why that matters.** Requirements change: now you also want an SMS and a log entry. No change to `OrderService`:

```csharp
class SmsService
{
    public void SendSms(string orderId)
    {
        Console.WriteLine($"SMS sent for order {orderId}");
    }
}
```

```csharp
orderService.PlaceOrder("ORD102", smsService.SendSms);
orderService.PlaceOrder("ORD103", logger.LogOrder);
```

Without a delegate, `OrderService` would need `EmailService`, `SmsService`, and the logger injected, plus `if` statements deciding which to call — and it would need editing every time a new action appeared.

❌ **The shape must match exactly.** The compiler enforces it:

```csharp
public bool CheckStock(string orderId, int quantity) { }

orderService.PlaceOrder("ORD104", CheckStock);   // ❌ wrong parameters and return type
```

**In practice you rarely declare your own delegate type.** `OrderAction` above is really just `Action<string>` — .NET's built-in types cover almost every case (Q2).

---


### Q2. What are `Func`, `Action`, and `Predicate`, and when do you use each?

**Answer.** These are **ready-made delegate types** built into .NET. Instead of declaring `delegate void OrderAction(string id)` yourself (Q1), you use one of these.

| Type | Returns | Meaning |
|---|---|---|
| `Action<T>` | nothing (`void`) | "go do something" |
| `Func<T, TResult>` | a value | "go get me something" |
| `Predicate<T>` | `bool` | "is this true?" |

**How to read the generic arguments.** This is the part people get wrong:

```csharp
Action<string>              // takes a string, returns nothing
Action<string, int>         // takes a string and an int, returns nothing

Func<int>                   // takes NOTHING, returns an int
Func<string, int>           // takes a string, returns an int
Func<string, int, bool>     // takes a string and an int, returns a bool
```

**For `Func`, the last type is always the return type** — everything before it is a parameter. `Action` has no return type, so every argument is a parameter.

---

**`Action<T>` — do something, return nothing.** Your `OrderAction` from Q1 was really just `Action<string>`:

```csharp
public void PlaceOrder(string orderId, Action<string> action)
{
    Console.WriteLine($"Order {orderId} placed.");
    action(orderId);
}
```

```csharp
orderService.PlaceOrder("ORD101", emailService.SendConfirmationEmail);
```

Typical uses: callbacks, progress reporting, event handlers, and configuration:

```csharp
// The DI configuration you write every day IS an Action
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString));      // Action<DbContextOptionsBuilder>
```

---

**`Func<T, TResult>` — go get a value.** The single most common use is LINQ, where every operator takes a `Func`:

```csharp
products.Where(p => p.Price < 50m)        // Func<Product, bool>
        .Select(p => p.Name)              // Func<Product, string>
        .OrderBy(n => n.Length);          // Func<string, int>
```

It is also how you defer work — pass the *recipe* rather than the result:

```csharp
// ✅ the value is only computed if the key is missing from the cache
public T GetOrAdd<T>(string key, Func<T> loadIfMissing)
{
    if (_cache.TryGetValue(key, out var found))
        return (T)found;

    var value = loadIfMissing();          // the expensive call happens only here
    _cache[key] = value;
    return value;
}
```

```csharp
var user = cache.GetOrAdd("user:42", () => _db.Users.Find(42));
```

Had the parameter been `T value` instead of `Func<T>`, the database call would run **every** time, even on a cache hit. That is the reason `Func` exists.

---

**`Predicate<T>` — a yes/no test.** Identical to `Func<T, bool>`, and only used by a few older methods:

```csharp
public bool IsOutOfStock(Product p)
{
    return p.Stock == 0;
}

List<Product> missing = products.FindAll(IsOutOfStock);   // Predicate<Product>
bool anyMissing       = products.Exists(IsOutOfStock);
int firstIndex        = products.FindIndex(IsOutOfStock);
```

**When to use which:**

| You want to... | Use |
|---|---|
| Run an action, ignore the result | `Action<T>` |
| Compute or fetch a value | `Func<T, TResult>` |
| Delay expensive work until needed | `Func<T>` |
| Test a condition | `Func<T, bool>` |
| Call `List<T>.FindAll` / `Exists` / `RemoveAll` | `Predicate<T>` — it is required there |

**Rule of thumb:** use `Func` and `Action` in new code. `Predicate<T>` exists mainly for those `List<T>` methods that predate LINQ — everything modern, including all of LINQ, expects `Func<T, bool>`.

❌ They are **not interchangeable** even though the shape matches. A method expecting `Predicate<Product>` will not accept a `Func<Product, bool>` variable — different types:

```csharp
Func<Product, bool> rule = IsOutOfStock;
products.FindAll(rule);              // ❌ does not compile
products.FindAll(p => rule(p));      // ✅ wrap it
```

---

### Q3. What is a multicast delegate? What happens to the return value and to exceptions?

**Answer.** A delegate can actually hold **more than one method** at once — you add them with `+=`. When you call the delegate, it runs **all of them, in order**. This is called a multicast delegate (and it's how events work).

```csharp
Action notify = () => Console.WriteLine("A");
notify += () => Console.WriteLine("B");   // now holds A and B
notify += () => Console.WriteLine("C");
notify();                                 // prints A, B, C
```

Two things to watch out for:
- **Return values:** if the methods return something, you only get back the **last one's** result — the rest are thrown away. So multicast really only makes sense for `void` methods (`Action`) and events.
- **Exceptions:** if one method **throws**, the chain **stops right there** — the ones after it never run.

If you need every method to run and collect their results, you can loop through them manually with `GetInvocationList()`.

---

### Q4. Delegate vs interface — when would you use one over the other?

**Answer.** Both let you plug in custom behavior. The choice comes down to **how much** behavior:

- Use a **delegate** when it's just **one method / one callback** — quick, works great with lambdas. Perfect for LINQ filters, event handlers, "run this when done" callbacks.
- Use an **interface** when you need **several related methods**, need to **hold state**, or want a type to formally declare "I implement this contract" (easier to mock in tests, too).

```csharp
// Delegate — one simple behavior
void Retry(int times, Action work) { /* … */ }

// Interface — a richer contract with multiple methods
interface IValidator<T> { bool IsValid(T item); string Describe(); }
```

Simple version: **one function → delegate; a bundle of related methods → interface.**

---

## E5 — Lambdas & Closures

### Q1. What is a lambda, and how does it relate to delegates and expression trees?

**Answer.** A **lambda** is a small **anonymous function** written inline: `(inputs) => result`. It has no name; you just write it where you need it. It usually gets stored in a `Func`/`Action` delegate.

```csharp
Func<int, int> square = x => x * x;   // a lambda stored in a Func
Console.WriteLine(square(5));         // 25
```

One subtlety worth knowing: the *same* lambda can become one of two things depending on what you assign it to:
- assigned to a **`Func`/`Action`** → it becomes **runnable code**.
- assigned to an **`Expression<...>`** → it becomes **data describing the code**, which databases (EF Core) can translate into SQL. (More on this in E7.)

```csharp
Func<int, int> runnable = x => x * x;             // code you can run
Expression<Func<int, int>> asData = x => x * x;   // data describing "x * x"
```

---

### Q2. What is a closure? What exactly does it capture?

**Answer.** A **closure** is a lambda that **uses a variable from the surrounding code**. The important part: it captures the **variable itself, not a copy of its value**. So if that variable changes later, the lambda sees the new value.

```csharp
int factor = 2;
Func<int, int> multiply = n => n * factor;   // uses the variable "factor"
Console.WriteLine(multiply(5));              // 10
factor = 3;                                  // change factor afterward...
Console.WriteLine(multiply(5));              // 15 — the lambda sees the change!
```

This surprises people who expect it to "lock in" `factor = 2`. It doesn't — it stays connected to the live variable. This behavior is behind the loop gotcha in the next question.

---

### Q3. Explain the closure-over-loop-variable gotcha. How does `for` differ from `foreach`?

**Answer.** This is a classic trap. If you create lambdas inside a loop and they capture the **loop variable**, they can all end up sharing the **same** variable — and all see its **final** value.

- A **`for`** loop uses **one shared variable** for all iterations → all the lambdas see the last value. **This still bites people today.**
- A **`foreach`** loop (since C# 5) gives each iteration its **own fresh variable** → `foreach` lambdas are **safe**.

```csharp
// for — one shared "i": all lambdas print 3, 3, 3
var forFns = new List<Func<int>>();
for (int i = 0; i < 3; i++)
    forFns.Add(() => i);
// forFns → 3, 3, 3   (oops)

// Fix: copy into a fresh local inside the loop, capture THAT
for (int i = 0; i < 3; i++)
{
    int copy = i;
    forFns.Add(() => copy);   // each lambda captures its own copy
}
// → 0, 1, 2

// foreach — safe since C# 5 (fresh variable each time)
var each = new List<Func<int>>();
foreach (var x in new[] { 0, 1, 2 })
    each.Add(() => x);
// → 0, 1, 2
```

**When in doubt:** copy the loop variable into a fresh local inside the loop body before capturing it.

---

### Q4. How can closures cause memory leaks or hidden allocations, and how do `static` lambdas help?

**Answer.** Capturing variables has two hidden costs:

1. **Keeping things alive longer than expected.** A captured variable stays alive as long as the lambda does. If the lambda captures `this` (an object) and gets attached to something long-lived (like an event), that whole object can't be cleaned up — a **memory leak**.
2. **Extra allocations.** Each time you create a lambda that captures variables, .NET allocates a little hidden object to hold them. In performance-critical loops this adds up.

**`static` lambdas** (C# 9+) let you say "this lambda captures nothing" — the compiler will **error** if you accidentally use an outside variable. This prevents accidental captures (and their leaks/allocations).

```csharp
Func<int, int> pure = static x => x * x;   // guaranteed to capture nothing
// static x => x * factor;   // ERROR: can't use "factor" in a static lambda
```

You mostly only worry about this in hot paths or with long-lived event subscriptions; for everyday code, capturing is fine.

---

## E6 — Events (`event` keyword, EventHandler pattern)

### Q1. What is an `event`, and how does it differ from a plain public delegate field?

**Answer.** An **event** is a **guarded delegate** that implements the **publisher-subscriber pattern**. It allows other classes to subscribe to notifications but prevents them from invoking or replacing the delegate.

> **Purpose:** Notify subscribers when something happens while protecting the delegate from external misuse.

### Public Delegate Field

```csharp
public Action<string> MessageReceived;
```

External code can:

```csharp
service.MessageReceived += LogMessage;      // ✔ Subscribe
service.MessageReceived -= LogMessage;      // ✔ Unsubscribe
service.MessageReceived("Hello");           // ✔ Invoke
service.MessageReceived = null;             // ✔ Replace/Clear
```

Anyone can invoke or overwrite the delegate, which breaks encapsulation.

### Event

```csharp
public event Action<string> MessageReceived;
```

External code can only:

```csharp
service.MessageReceived += LogMessage;      // ✔ Subscribe
service.MessageReceived -= LogMessage;      // ✔ Unsubscribe
```

These are **not allowed**:

```csharp
service.MessageReceived("Hello");           // ❌ Compile error
service.MessageReceived = null;             // ❌ Compile error
```

Only the declaring class can raise the event:

```csharp
MessageReceived?.Invoke("Hello");
```

---

### Q2. What is the standard .NET event pattern (`EventHandler` / `EventHandler<T>`)?

**Answer.** .NET has a standard convention so all events look the same:
- Declare events using **`EventHandler`** (no extra data) or **`EventHandler<TEventArgs>`** (with data).
- The handler signature is always `(object? sender, SomeEventArgs e)` — who raised it, and any details.
- Trigger it from a **`protected virtual OnSomething`** method so subclasses can extend it.

```csharp
public class OrderService
{
    public event EventHandler<OrderPlacedEventArgs>? OrderPlaced;

    protected virtual void OnOrderPlaced(OrderPlacedEventArgs e)
        => OrderPlaced?.Invoke(this, e);

    public void Place(Order o)
    {
        // … save the order …
        OnOrderPlaced(new OrderPlacedEventArgs(o.Id));
    }
}

public class OrderPlacedEventArgs(int orderId) : EventArgs
{
    public int OrderId { get; } = orderId;
}
```

Following this `(sender, args)` convention means tools and other code know what to expect from any event.

---

### Q3. Why raise events with `Handler?.Invoke(this, e)`?

**Answer.** Two reasons, both about safety:

1. **No subscribers = null.** If nobody has subscribed, the event is `null`, and calling it directly would crash. The `?.` skips the call when it's null.
2. **Thread safety.** Writing `Handler?.Invoke(...)` safely grabs the current handlers first, then calls them. This avoids a rare crash where another thread unsubscribes the last handler at the exact wrong moment.

```csharp
protected virtual void OnChanged() => Changed?.Invoke(this, EventArgs.Empty);
```

So `Handler?.Invoke(this, e)` is just the correct, safe way to raise an event.

---

### Q4. What is the event memory-leak gotcha, and how do you avoid it?

**Answer.** When you subscribe with `publisher.SomeEvent += handler`, the **publisher now holds a reference to the subscriber**. If the publisher lives a long time and the subscriber **never unsubscribes**, the subscriber **can't be garbage-collected** — it stays in memory forever (and its handler keeps getting called on a "dead" object). This is a very common real-world leak.

```csharp
class Subscriber
{
    public Subscriber(Publisher p) => p.Event += OnEvent;  // p now references THIS
    void OnEvent(object? s, EventArgs e) { … }
    // If we never do  p.Event -= OnEvent  and p lives on, THIS never gets cleaned up.
}
```

**How to avoid it:**
- **Unsubscribe** when you're done — in `Dispose`, on page close, etc. Use `-=` with the same handler you subscribed with. (Note: you can't unsubscribe an inline lambda — store it in a variable or use a named method so you can remove it later.)
- Be extra careful with **static or long-lived events** — they keep subscribers alive for the entire app's lifetime.

---

## E7 — Expression Trees

### Q1. What is an expression tree, and how does it differ from a delegate?

**Answer.** Normally a lambda becomes **runnable code** — you can call it, but you can't see *inside* it. An **expression tree** is the same lambda captured as **data you can inspect** — a structured description of "a comparison of `n` and `5` using greater-than."

The compiler picks which one you get based on the type you assign to:

```csharp
Func<int, bool> code = n => n > 5;                 // runnable code (can only call it)
Expression<Func<int, bool>> tree = n => n > 5;     // data (can look inside it)

// You can examine the tree:
var body = (BinaryExpression)tree.Body;
Console.WriteLine(body.NodeType);   // GreaterThan
Console.WriteLine(body.Left);       // n
Console.WriteLine(body.Right);      // 5
```

The difference in one line: a delegate tells you *"run this and get an answer"*; an expression tree tells you *"here's what the code says"* — so another tool can read it and turn it into something else, like SQL.

---

### Q2. Why do expression trees matter — where are they actually used?

**Answer.** The main place is **database queries with EF Core**. When you write `db.Orders.Where(o => o.Total > 100)`, EF Core doesn't *run* your lambda — it **reads the expression tree and turns it into SQL** (`WHERE Total > 100`). This is only possible because the code arrived as inspectable data, not compiled code.

```csharp
// EF Core reads this as data and translates it to SQL — it never runs the lambda in C#
IQueryable<Order> q = db.Orders.Where(o => o.Total > 100 && o.CustomerId == id);
```

This is also *why* using a custom C# method in a database query fails: EF Core doesn't know how to translate it into SQL.

Other places expression trees show up:
- **Mapping libraries** (like AutoMapper) use them to connect columns to properties.
- **Validation libraries** (like FluentValidation) use `x => x.Name` to grab a property's name.

For most everyday coding, you just need to understand the EF Core connection — the rest is library internals.

---