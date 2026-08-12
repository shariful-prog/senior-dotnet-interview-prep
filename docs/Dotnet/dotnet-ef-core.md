# O. Entity Framework Core
---

## O1 — DbContext, DbSet & the Change Tracker

### Q1. What is a `DbContext`, conceptually — and what design patterns does it implement?

**Answer.** `DbContext` is the central class responsible for interacting with the database. A **`DbContext`** is one job of work against the database. You create it, load data, change it, call `SaveChanges()`, and dispose it.

It gives you two patterns:

- **Unit of Work** — it holds your inserts, updates, and deletes in memory. One `SaveChanges()` sends them all in a single transaction. They all succeed or all fail.
- **Repository Pattern** — each `DbSet<T>` is a collection-like abstraction over one table. You add, remove, and query objects through it and never write SQL yourself.

**What the Repository pattern is.** A repository sits between your application code and the database. Its job is to make a table look like an in-memory collection of objects. Your code says "add this customer" or "give me the customers in Sweden"; the repository works out the SQL, runs it, and turns rows back into objects. The point is that the calling code holds no knowledge of tables, columns, connections, or SQL dialect. Swap the database and that code does not change.

**How `DbContext` implements it.** Each `DbSet<T>` property is the repository for one entity type. It exposes exactly the operations the pattern asks for:

```csharp
db.Customers.Add(customer);        // insert an object into the "collection"
db.Customers.Remove(customer);     // remove one
var c = await db.Customers.FindAsync(5);                 // get by key
var swedes = await db.Customers                          // query by criteria
 .Where(x => x.Country == "SE")
 .ToListAsync();
```

None of those lines mention SQL. `DbSet<T>` implements `IQueryable<T>`, so your `Where` is translated into a `WHERE` clause and executed on the server (Q6), and the returned rows are materialised into `Customer` objects for you.

The two patterns work together: `DbSet<T>` is the repository that stages the changes, and the surrounding `DbContext` is the unit of work that commits them all in one transaction.

---

### Q2. What is a `DbSet<T>` and how does it relate to `IQueryable`?

**Answer.** A **`DbSet<T>`** is your way in to one table. `db.Customers` maps to the Customers table. You use it to query and to stage changes.

It is an `IQueryable<T>`. That means LINQ you write against it becomes SQL and runs on the database, not in your app (Q6).

```csharp
db.Customers.Add(newCustomer);            // stages an INSERT — nothing sent yet
var c = await db.Customers.FindAsync(1);  // checks memory first, then the database
db.Customers.Remove(c);                   // stages a DELETE
```

**`Find(id)` works differently from a LINQ query.** It first looks at entities the context already holds. If the row is there, it returns with no database call. `Single(x => x.Id == 1)` always queries.

One senior point: you usually **do not need a repository wrapper** here. `DbSet<T>` is already a repository. `DbContext` is already a unit of work. An `IRepository<T>` layer on top often repeats both, and it tends to hide useful features like `Include` and projection. Add one only if you want EF kept out of your domain on purpose.

---

### Q3. What does the change tracker actually do, and how does `SaveChanges()` know what SQL to run?

**Answer.** When a tracked query loads an entity, EF saves a **snapshot** of its original values. At `SaveChanges()`, EF compares the entity with that snapshot. It finds which properties differ and writes an `UPDATE` for only those columns.

Every tracked entity has a **state**:

| State | Meaning | SQL on SaveChanges |
|-------|---------|--------------------|
| `Added` | new, staged via `Add` | `INSERT` |
| `Unchanged` | loaded, nothing changed | none |
| `Modified` | differs from the snapshot | `UPDATE` (changed columns only) |
| `Deleted` | staged via `Remove` | `DELETE` |
| `Detached` | this context isn't tracking it | none |

```csharp
var c = await db.Customers.FirstAsync(x => x.Id == 1); // snapshot saved here
c.Email = "new@x.com";                                 // no SQL yet
await db.SaveChangesAsync();                           // UPDATE ... WHERE Id = 1
```

**The key point:** there is no "update" call. You change a normal property, and EF works out the SQL by comparing snapshots. That is what the unit of work does for you — you say what the data should be, not which statements to run.

EF also sorts the statements so parents go in before children, and batches them to cut round-trips.

---

### Q4. Why is `DbContext` not thread-safe, and what breaks if you share one?

**Answer.** A `DbContext` is built to do **one thing at a time**. It keeps notes as you work, and those notes assume only one operation is touching them.

It holds two things that cannot be shared:

- **The change tracker** — the list of loaded entities and what changed on each one. Every query adds to this list.
- **The database connection** — one connection, running one command.

Nothing guards either of them. If two operations run at the same time, one overwrites what the other is in the middle of writing. The list ends up wrong, or the connection gets a second command while the first is still going.

EF does not let that happen quietly. It checks, and throws:

```csharp
// ❌ two queries started at once on one context
await Task.WhenAll(db.Orders.ToListAsync(), db.Customers.ToListAsync());
// → InvalidOperationException: A second operation was started on this context...

// ✅ one finishes, then the next starts
var orders = await db.Orders.ToListAsync();
var customers = await db.Customers.ToListAsync();
```

The difference is small but important. In the bad version both queries are **started** before either finishes. In the good version each `await` waits for the answer before the next line runs.

If you see that error message, the cause is almost always this: one context used by two things at the same time. Common sources are `Task.WhenAll`, `Parallel.ForEach`, a background task handed the same context, or a `DbContext` registered as a singleton instead of scoped.

**The rule:** one context per job of work, one operation at a time. If you genuinely need parallel database calls, give each one its own context from a factory (Q31):

```csharp
// ✅ separate context per parallel branch
await Task.WhenAll(
 LoadOrders(factory),
 LoadCustomers(factory));

async Task<List<Order>> LoadOrders(IDbContextFactory<AppDbContext> factory)
{
 await using var db = factory.CreateDbContext();   // its own context and connection
 return await db.Orders.ToListAsync();
}
```

❌ **`async` does not make a context thread-safe.** This trips people up. Awaiting calls one after another is fine — that is just waiting in order. The problem is overlapping them, and `async` makes overlapping easy to write by accident.

---

### Q5. `SaveChanges` vs `SaveChangesAsync` — when and why?

**Answer.** The result is the same. The difference is whether the thread waits.

The sync version blocks the thread until the database replies. The async version hands the thread back to the thread pool while it waits.

This matters because the thread pool is shared and limited. If 200 requests each block a thread on a database call, you run out of threads and throughput drops — even though the CPU is doing nothing ([../C-Sharp/csharp-async.md](../C-Sharp/csharp-async.md)).

In a web app, **use async everywhere** — `ToListAsync`, `FirstOrDefaultAsync`, `SaveChangesAsync` — and pass a `CancellationToken`.

```csharp
await db.SaveChangesAsync(cancellationToken);
```

Async does not make one call faster. It lets the app handle more requests at the same time.

❌ Never call the sync version and block on it from async code (`.Result`, `.Wait()`). That starves the thread pool and can deadlock.

---

## O2 — LINQ→SQL Translation & IQueryable Deferred Execution

> Related: `IQueryable` vs `IEnumerable` in [../C-Sharp/csharp-linq.md](../C-Sharp/csharp-linq.md).

### Q6. How does EF Core turn a LINQ query into SQL?

**Answer.** You write this:

```csharp
var top = await db.Orders
 .Where(o => o.Total > 100)
 .OrderByDescending(o => o.Date)
 .Take(10)
 .ToListAsync();
```

EF sends this to the database:

```sql
SELECT TOP(10) * FROM Orders WHERE Total > 100 ORDER BY Date DESC
```

Each LINQ method became a piece of SQL: `Where` → `WHERE`, `OrderByDescending` → `ORDER BY … DESC`, `Take` → `TOP`. Ten rows come back, not ten million.

**How it does that.** Normally when you call a method, it runs. Here it does not. `db.Orders` is an `IQueryable`, and calling `Where` on an `IQueryable` just **writes down** what you asked for and hands back a new `IQueryable`. Nothing executes:

```csharp
var q = db.Orders.Where(o => o.Total > 100);   // no database call
q = q.OrderByDescending(o => o.Date).Take(10); // no database call
var top = await q.ToListAsync();               // NOW one query runs
```

So by the time you call `ToListAsync()`, EF is holding a written-down list of everything you wanted. It reads that list, builds **one** SQL statement from it, and runs it. The written-down form has a name — an **expression tree** — but the idea is just that your code was saved instead of executed.

**Why it matters.** Because the whole chain is collected before anything runs, the filtering and sorting go into the SQL and the database does the work.

`IEnumerable` is the opposite: it means "data already in memory." If your query becomes `IEnumerable` too early, everything after that runs in your app on rows already loaded — that is the bug in Q8.

---

### Q7. What is deferred execution and when does a query *actually* run?

**Answer.** **Deferred execution** means a LINQ query is not executed when you create it. Instead, EF Core builds the query and waits until you actually need the results. Only then does it generate SQL and send it to the database.

What triggers it:

- **Collections:** `ToList`/`ToListAsync`, `ToArray`, or a `foreach`.
- **Single values:** `First`, `Single`, `Count`, `Any`, `Sum`, `Max`, and their `...Async` forms.

❌ **Each time you enumerate, the query runs again:**

```csharp
var recent = db.Orders.Where(o => o.Date > cutoff);
var count = await recent.CountAsync();    // query #1
var list = await recent.ToListAsync();    // query #2 — runs again
```

If you need the results twice, call `ToList()` once and reuse the list.

Deferral is also what lets you build a query up in steps and still run one statement:

```csharp
var q = db.Orders.AsQueryable();
if (customerId is not null) q = q.Where(o => o.CustomerId == customerId);
if (from is not null) q = q.Where(o => o.Date >= from);
var results = await q.Take(50).ToListAsync();   // one query with every filter
```

---

### Q8. What is the "premature `ToList`" bug, and why is it dangerous?

**Answer.** `ToList()` in the middle of a query stops the SQL translation. Every operator after it runs in memory, on rows already sent over the network.

```csharp
// ❌ downloads the whole table, then filters in memory
var vips = db.Customers.ToList().Where(c => c.Orders.Count > 50);

// ✅ stays IQueryable, so the filter becomes a WHERE clause
var vips = await db.Customers.Where(c => c.Orders.Count() > 50).ToListAsync();
```

The first line runs `SELECT * FROM Customers` with **no WHERE clause**. With two million customers, that sends two million rows so you can throw nearly all of them away.

It passes on a dev database with fifty rows, then fails in production. `ToList().Take(10)` has the same problem: it fetches everything to keep ten.

In code review, **an early `.ToList()` or `.AsEnumerable()` mid-query is a red flag.** It is usually an accident — someone added it to fix a compile error.

---

### Q9. What happens when a LINQ expression *can't* be translated to SQL?

**Answer.** EF throws an `InvalidOperationException` saying the expression "could not be translated."

That is on purpose. Older versions ran it in memory instead, without telling you. Queries looked fine while pulling whole tables across. EF Core 3.0 made it an error so you catch it while writing the code.

The usual cause is calling your own method inside the query. EF can only translate what it recognises:

```csharp
// ❌ throws: EF cannot translate IsPremium()
db.Customers.Where(c => c.IsPremium()).ToList();

// ✅ better: say the same thing in a way EF can translate
db.Customers.Where(c => c.Tier == "Premium").ToList();

// ✅ or run it in memory on purpose, after cutting the rows down
db.Customers.Where(c => c.Tier == "Premium")   // filter on the database FIRST
 .AsEnumerable()                               // then switch to memory
 .Where(c => c.IsPremium())
 .ToList();
```

`AsEnumerable()` is the supported way out. The order matters: filter on the database before you switch, or you have written the Q8 bug on purpose.

---

### Q10. Are EF Core queries safe from SQL injection? How do I see the generated SQL?

**Answer.** **LINQ queries are safe.** Your values are sent as **parameters** (`@p0`, `@p1`), separate from the SQL text. So input like `'; DROP TABLE Users; --` is just a string to search for. It cannot change the query.

Raw SQL is the exception. Use `FromSql`, not `FromSqlRaw` with string concatenation:

```csharp
// ✅ the interpolated value becomes a parameter
db.Users.FromSql($"SELECT * FROM Users WHERE Name = {name}");

// ❌ concatenation drops user input straight into the SQL
db.Users.FromSqlRaw("SELECT * FROM Users WHERE Name = '" + name + "'");
```

Those two look almost the same and are completely different on safety ([dotnet-security.md](dotnet-security.md)).

**To see the SQL:** `ToQueryString()` gives you the SQL for a query without running it. `LogTo` on the options logs every query. `EnableSensitiveDataLogging()` also shows parameter values — ❌ **dev only**, since it writes customer data into your logs.

---

### Q10b. What are compiled queries, and when are they worth it?

**Answer.** Normally, every time EF Core executes a LINQ query, it has to **analyze the expression tree, translate it into SQL, and prepare an execution plan** (EF Core's internal query plan, not SQL Server's execution plan). EF Core caches much of this work automatically, but there is still some overhead.

**Compiled queries** let you precompile a LINQ query once and reuse it many times, avoiding most of that repeated translation overhead.

#### Normal Query

```csharp
public async Task<User?> GetUser(int id)
{
    return await context.Users
        .FirstOrDefaultAsync(u => u.Id == id);
}
```

Every call:

1. Build expression tree
2. Look up or compile EF Core query
3. Generate SQL (or retrieve cached translation)
4. Execute SQL

Although EF Core caches query compilation internally, each execution still performs cache lookup and some processing.

#### Compiled Query

```csharp
private static readonly Func<AppDbContext, int, User?> _getUserById =
    EF.CompileQuery(
        (AppDbContext context, int id) =>
            context.Users.FirstOrDefault(u => u.Id == id));
```

Use it like this:

```csharp
var user = _getUserById(context, 10);
```

The query translation happens **once** when the delegate is created.

Later calls skip most of the translation pipeline.

#### Async Compiled Query

For asynchronous queries:

```csharp
private static readonly Func<AppDbContext, int, IAsyncEnumerable<User>> _usersByAge =
    EF.CompileAsyncQuery(
        (AppDbContext context, int age) =>
            context.Users.Where(u => u.Age >= age));
```

Use:

```csharp
await foreach (var user in _usersByAge(context, 18))
{
    Console.WriteLine(user.Name);
}
```

#### When are compiled queries worth it?

They help when:

- A query is executed **thousands or millions of times**
- The query shape never changes
- The application is performance-critical (high-throughput APIs, background services)

Example:

```csharp
GetUserById(id)
```

called for every incoming API request.

Compiling it once can reduce CPU overhead.

#### When are they **not** worth it?

For most applications:

- Typical CRUD APIs
- Admin dashboards
- Queries executed only occasionally
- Dynamic queries with optional filters

The performance gain is usually very small because the database execution time is often much larger than EF Core's query compilation time.

---

## O3 — Loading Strategies & the N+1 Problem

> This is *the* classic EF interview topic. Know it cold.

### Q11. What is the N+1 problem? How do you detect it?

**Answer.** **N+1** is one query for a list, plus one more query for every item in it. Load 1,000 orders and read each order's customer, and you run 1,001 queries.

What costs you is the number of round-trips, not the amount of data. 1,001 round-trips at 1ms each is over a second of waiting.

```csharp
var orders = await db.Orders.ToListAsync();  // 1 query
foreach (var o in orders)
 Console.WriteLine(o.Customer.Name);        // one query per order — N+1
```

The trouble is that `o.Customer.Name` looks like plain property access. With lazy loading on (Q14), each read runs a query.

**To find it**, check the SQL log (Q10) for the same query repeated with only the parameter changing:

```
SELECT ... FROM Customers WHERE Id = 1
SELECT ... FROM Customers WHERE Id = 2   ← this repetition is the signature
```

It usually gets past testing, because ten seed rows means ten fast queries. The fixes are Q12 and Q16.

---

### Q12. What is eager loading, and how do `Include` / `ThenInclude` work?

**Answer.** **Eager loading** means loading the main entity and its related entities in one query.
 **Eager loading** fetches the related data in the same query. `Include` names what to load, so the loop in Q11 runs no extra queries.

`ThenInclude` loads something on the entity you just included:

```csharp
var orders = await db.Orders
 .Include(o => o.Customer)      // Order → Customer
 .Include(o => o.Lines)         // Order → Lines
 .ThenInclude(l => l.Product)   // each Line → Product
 .ToListAsync();
```

`ThenInclude` continues the chain above it. Here it applies to `Lines`, not to `Order`. To include something else on `Order`, start a new `Include`.

**Filtered includes** (EF Core 5+) cut down what comes back, which helps when a parent has thousands of children:

```csharp
db.Blogs.Include(b => b.Posts.Where(p => p.Published).Take(5));
```

Two limits. `Include` fetches **every column** of everything you include, so it wastes bandwidth when you only need a couple of fields (Q16). And including several collections causes the problem in Q13.

---

### Q13. What is the cartesian explosion, and how does `AsSplitQuery` fix it?

**Answer.** If you `Include` two or more **collections**, EF writes one SQL statement that joins them. Joining two unrelated collections multiplies the rows.

A blog with 10 posts and 10 tags returns **100 rows**, not 20. Every post is paired with every tag, and the blog's columns repeat in each row. With 100 posts and 50 tags, it is 5,000 rows to deliver 150 things.

`AsSplitQuery()` runs one query per collection and joins them up in memory:

```csharp
var blogs = await db.Blogs
 .Include(b => b.Posts)
 .Include(b => b.Tags)
 .AsSplitQuery()      // separate queries — nothing multiplies
 .ToListAsync();
```

The trade-off: **one query** means a single round-trip but duplicated rows. **Split queries** mean no duplication but several round-trips.

Rule of thumb: two or more collection includes → use split. One collection, or only single-item navigations like `Customer` → one query is fine.

⚠️ Split queries run separately, so someone else can commit a change between them. Use a transaction if you need one consistent picture.

---

### Q14. What is lazy loading, why is it off by default, and why is it dangerous?

**Answer.** **Lazy loading** means related entities are **loaded automatically only when you access a navigation property**, not when the main entity is queried.

```csharp
var customer = await context.Customers.FirstAsync(c => c.Id == 1);

// Executes another SQL query here
var orders = customer.Orders;
```

### Why is it off by default?

EF Core disables lazy loading by default because it can execute **hidden SQL queries** without the developer realizing it.

### Why is it dangerous?

The biggest danger is the **N+1 Query Problem**.

```csharp
var customers = await context.Customers.ToListAsync();

foreach (var customer in customers)
{
    Console.WriteLine(customer.Orders.Count);
}
```

This executes:

- **1 query** to load customers.
- **1 additional query per customer** to load orders.

For 100 customers:

```text
1 + 100 = 101 queries
```

This causes:

- Poor performance
- More database round trips
- Hidden SQL queries
- Hard-to-debug performance issues

### Preferred Alternative

Use **Eager Loading** with `Include()` when you know you'll need related data.

```csharp
var customers = await context.Customers
    .Include(c => c.Orders)
    .ToListAsync();
```

---

### Q15. What is explicit loading?

**Answer.** **Explicit loading** loads a navigation when you ask for it, with a real method call. Same timing as lazy loading, but you can see it in the code.

Use `Entry(entity)` with `Reference` for a single item or `Collection` for many:

```csharp
var order = await db.Orders.FindAsync(id);

await db.Entry(order).Reference(o => o.Customer).LoadAsync();
await db.Entry(order).Collection(o => o.Lines).LoadAsync();

// Query a collection without loading it:
var lineCount = await db.Entry(order).Collection(o => o.Lines).Query().CountAsync();
```

`.Query()` gives you an `IQueryable` for the related rows. You can count or filter them on the database without loading any of them.

Use it when you only **sometimes** need the data — load an order's lines only if the user expands the row. Eager loading would fetch them every time. Lazy loading would work but hide the query.

The limit is that it works one entity at a time, so calling it in a loop rebuilds N+1 by hand. For a list, use `Include` or projection.

---

### Q16. What's the *best* fix when you only need a few columns — projection to a DTO?

**Answer.** Often the best answer is not to load entities at all. `Select` into a DTO asks for only the columns you name:

```csharp
var summaries = await db.Orders
 .Where(o => o.Date > cutoff)
 .Select(o => new OrderSummaryDto
 {
 Id = o.Id,
 CustomerName = o.Customer.Name,   // becomes a JOIN — no Include needed
 LineCount = o.Lines.Count()       // becomes a COUNT — lines never loaded
 })
 .ToListAsync();
```

What that one statement gives you:

- **Only the columns you named** come back.
- **No `Include`** — EF adds the JOIN itself when it sees `o.Customer.Name`.
- **No cartesian explosion** (Q13) — `Count()` becomes an aggregate.
- **No tracking** (Q17), because DTOs are not entities.
- **N+1 cannot happen** — it is one query by design.

**For read-only work — list screens, reports, API responses — projection is the senior default.** It fixes what `Include` fixes, and more.

Use full entities with `Include` when you plan to change the data and save it, since that needs tracking (Q3).

---

## O4 — Change Tracking vs AsNoTracking

### Q17. What does `AsNoTracking()` do, and when should you use it?

**Answer.** A normal query saves a snapshot of every entity it returns so EF can detect changes (Q3). If you are only reading, that work is wasted.

`AsNoTracking()` skips the snapshot and the identity map:

```csharp
var products = await db.Products
 .AsNoTracking()          // read-only: no snapshot
 .Where(p => p.InStock)
 .ToListAsync();
```

**Use it for read-only queries** — GET endpoints, reports, lookups. The gain grows with row count. It is clear on a 5,000-row report and tiny on a single lookup.

You can make it the default for a whole context with `UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking)` and turn tracking on only where you write. Rule of thumb: **track only what you plan to change.**

❌ **What follows from this:** with no snapshot, EF cannot see your edits. Load with `AsNoTracking()`, change a property, call `SaveChanges()`, and nothing happens. There is no error either, which is what makes it hard to spot.

---

### Q18. What's the identity map, and how does `AsNoTrackingWithIdentityResolution` fit in?

**Answer.** The **Identity Map** is an internal data structure maintained by EF Core's **Change Tracker**. It ensures that **within a single `DbContext`, only one object instance exists for each primary key**.

When EF Core materializes an entity from the database, it first checks the Identity Map:

1. If an entity with the same primary key is already tracked, EF Core **returns the existing object**.
2. Otherwise, it **creates a new object**, adds it to the Identity Map, and starts tracking it.

For example, if three orders belong to the same customer:

```text
Order1 ──► Customer(Id=1)
Order2 ──► Customer(Id=1)
Order3 ──► Customer(Id=1)
```

Instead of creating three separate `Customer` objects, EF Core creates **one** `Customer` object and all three orders reference it. This saves memory, keeps navigation properties consistent, and ensures changes are tracked correctly.

#### How does `AsNoTrackingWithIdentityResolution()` fit in?

Normally, `AsNoTracking()` disables **both change tracking and the Identity Map**, so EF Core creates a new object every time it encounters the same row.

`AsNoTrackingWithIdentityResolution()` is a middle ground:

- ❌ No change tracking
- ✅ Uses a **temporary Identity Map** while executing the query
- ✅ Reuses entities with the same primary key
- ❌ Discards the Identity Map after the query completes

Example:

```csharp
var orders = await context.Orders
    .Include(o => o.Customer)
    .AsNoTrackingWithIdentityResolution()
    .ToListAsync();
```

If 100 orders belong to the same customer, EF Core creates **one `Customer` object**, not 100. Once the query finishes, the temporary Identity Map is removed, so the entities are **not tracked** by the `DbContext`.

---

## O5 — Migrations

### Q19. Walk through the code-first migrations workflow.

**Answer.** **Code-first** means your C# model is the source of truth, and the database schema is brought in line with it. **Migrations** are the ordered steps that do that.

The workflow:

1. **Change the model** — add a property, a class, or some configuration.
2. **`dotnet ef migrations add AddCustomerEmail`** — EF compares your model to the **snapshot** file, which records the last known schema, and writes a migration with `Up()` and `Down()`.
3. **Read the migration it generated.** Not optional — see below.
4. **`dotnet ef database update`** — apply it.

```bash
dotnet ef migrations add AddCustomerEmail
dotnet ef migrations remove       # undo the last one, if not applied yet
dotnet ef database update         # apply anything pending
dotnet ef database update Prev    # roll back to a named migration
```

❌ **Never apply a generated migration without reading it.** EF compares schema shapes, so it cannot tell a **rename** from a **drop plus an add**. Rename `Name` to `FullName` and it may drop the `Name` column, losing every value, then add an empty `FullName`. It runs with no error. Change it to `RenameColumn` yourself.

**Commit the snapshot file** with the migration. Without it, the next `add` compares against stale data. It is also the file that conflicts when two branches both add migrations — regenerate instead of merging it by hand.

---

### Q20. How should migrations be applied in production — `Database.Migrate()` at startup or SQL scripts?

**Answer.** Do not use `Database.Migrate()` at startup in production. Three reasons:

- **Instances race each other.** Three containers start together, all try to change the schema, and you get deadlocks or a half-applied migration. It usually works, then fails during a deploy under load.
- **It needs too much permission.** Your app's login must be allowed to alter tables all the time, just so it can migrate at boot.
- **No review, no gate.** The schema change happens during startup, so nobody approves it.

**Better: generate a SQL script and run it as its own deploy step**, with credentials meant for that job. Make it **idempotent** so running it twice is safe:

```bash
dotnet ef migrations script --idempotent -o migrate.sql
```

Now the change is something you can review in a pull request, and it is separate from the app deploy. That also lets you apply the schema change before the new code ships, which is how zero-downtime deploys normally work.

`Database.Migrate()` is fine for local dev, integration tests, and single-instance apps.

❌ `EnsureCreated()` is **not** a migrations tool. It creates the schema with no migration history, so you can never evolve it. Mixing it with `Migrate()` leaves a database that neither one can manage.

---

### Q21. How do you seed data, and how do you handle model drift?

**Answer.** **Data seeding** means inserting initial or default data into the database automatically.

Common examples:

- Roles (`Admin`, `User`)
- Countries
- Status values
- Default settings


### How to Seed Data

Use `HasData()` inside `OnModelCreating()`.

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Role>().HasData(
        new Role { Id = 1, Name = "Admin" },
        new Role { Id = 2, Name = "User" }
    );
}
```

When you create a migration, EF Core generates `InsertData()` statements, and the data is inserted when the migration is applied.

#### What is Model Drift?

**Model drift** happens when the **EF Core model and the database schema are no longer in sync.**

For example:

- A developer manually changes a database table.
- Someone edits the database without creating an EF Core migration.
- A migration wasn't applied to a database.

Now:

```text
EF Core Model  ≠  Database Schema
```

This can cause runtime errors or incorrect behavior.

#### How do you handle model drift?

- **Always make schema changes through EF Core migrations.**
- **Never manually modify the production database schema.**
- Ensure all environments (Development, Test, Production) apply the same migrations.
- Review generated migrations before applying them.

---

### Q22. Why and how would you put migrations in a separate assembly?

**Answer.** In a layered solution, migrations belong in the **data or infrastructure project** next to the `DbContext`, not in the web project. That keeps schema code and design-time tooling out of your startup and domain projects, and it lets several apps share one context.

```csharp
services.AddDbContext<AppDbContext>(o =>
 o.UseSqlServer(conn, sql =>
 sql.MigrationsAssembly("MyApp.Infrastructure")));
```

If the CLI then cannot build your context, add an `IDesignTimeDbContextFactory<AppDbContext>`. This happens a lot when the connection string lives in the web project's config. The tools use that class to build a context without starting your app.

It is also the fix for **"Unable to create an object of type 'AppDbContext'"**. That error means the tooling could not work out how to construct your context, and this class tells it how. Point its connection string at a local dev database, since the file gets committed.

---

## O6 — Relationships & Fluent Configuration

### Q23. How do you configure relationships — conventions vs data annotations vs Fluent API?

**Answer.** EF Core provides **three ways** to configure relationships between entities.

#### 1. Conventions (Automatic)
EF Core automatically detects relationships based on **naming conventions**.

```csharp
public class Customer
{
    public int Id { get; set; }

    public ICollection<Order> Orders { get; set; }
}

public class Order
{
    public int Id { get; set; }

    public int CustomerId { get; set; }   // Foreign Key

    public Customer Customer { get; set; } // Navigation Property
}
```

EF Core automatically recognizes:

- `CustomerId` → Foreign Key
- `Customer` → Navigation Property
- One `Customer` has many `Orders`

No configuration is required.

**Best for:** Simple relationships.

#### 2. Data Annotations

Configure relationships using attributes in the entity class.

Example:

```csharp
public class Order
{
    public int Id { get; set; }

    [ForeignKey(nameof(Customer))]
    public int CustomerRefId { get; set; }

    public Customer Customer { get; set; }
}
```

Here the foreign key doesn't follow EF Core's naming convention (`CustomerId`), so we explicitly tell EF Core which property is the foreign key.

Common attributes:

- `[ForeignKey]`
- `[InverseProperty]`
- `[Required]`

**Best for:** Simple configurations directly on the model.

#### 3. Fluent API (Recommended)

Configure relationships inside `OnModelCreating()`.

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Order>()
        .HasOne(o => o.Customer)
        .WithMany(c => c.Orders)
        .HasForeignKey(o => o.CustomerId);
}
```

This configures:

- One `Order` has one `Customer`
- One `Customer` has many `Orders`
- `CustomerId` is the foreign key

You can also configure:

- Cascade delete
- Required/optional relationships
- Composite keys
- Many-to-many relationships

**Best for:** Complex or production applications.

#### Priority

If multiple configurations exist, EF Core applies them in this order:

```text
Fluent API
    ↑
Data Annotations
    ↑
Conventions
```

**Fluent API has the highest priority.**

---

### Q24. How do you model one-to-many, many-to-many, and one-to-one relationships?

**Answer.**

**One-to-many** — one customer, many orders. A collection on the "one" side, plus a navigation and a foreign key on the "many" side. Conventions usually set this up for you.

**Many-to-many** — since EF Core 5, put a collection on **each** side and EF creates and manages the join table. You write no join class:

```csharp
public class Post { public List<Tag> Tags { get; set; } = []; }
public class Tag { public List<Post> Posts { get; set; } = []; }
```

Adding a link is just `post.Tags.Add(tag)`. **But** if the join needs columns of its own, like who added the tag and when, you need a real join class. The question to ask is whether the relationship itself carries data.

**One-to-one** — one user, one profile. You have to say which side holds the foreign key, because EF cannot work it out:

```csharp
mb.Entity<User>()
 .HasOne(u => u.Profile)
 .WithOne(p => p.User)
 .HasForeignKey<Profile>(p => p.UserId);   // the generic says which side it is
```

EF puts a unique constraint on that key. That is what makes it one-to-one instead of one-to-many.

---

### Q25. What are delete behaviors, owned types, and keyless entities?

**Answer.** Three unrelated mapping features that often come up together.

#### 1. Delete Behaviors

**Delete behavior** defines what happens to child entities when the parent entity is deleted.

Example:

```text
Customer
    │
    └── Orders
```

```csharp
modelBuilder.Entity<Order>()
    .HasOne(o => o.Customer)
    .WithMany(c => c.Orders)
    .OnDelete(DeleteBehavior.Cascade);
```

Common options:

- **Cascade** – Deletes child records automatically.
- **Restrict** – Prevents deleting the parent if children exist.
- **SetNull** – Sets the foreign key to `NULL` (FK must be nullable).
- **NoAction** – Leaves enforcement to the database.

#### 2. Owned Types

An **Owned Type** is a value object that **belongs to another entity** and cannot exist independently.

Example:

```csharp
public class Customer
{
    public int Id { get; set; }
    public Address Address { get; set; }
}

modelBuilder.Entity<Customer>()
    .OwnsOne(c => c.Address);
```

`Address` is stored as part of the `Customer` and doesn't have its own identity.


#### 3. Keyless Entities

A **Keyless Entity** is an entity **without a primary key**.

```csharp
modelBuilder.Entity<SalesReport>()
    .HasNoKey();
```

They're typically used for:

- Database views
- Reports
- Raw SQL query results

They are **read-only** and cannot be inserted, updated, or deleted.

---

## O7 — Transactions & Concurrency

### Q26. How does EF Core handle transactions — and when do you need an explicit one?

**Answer.** **Every `SaveChanges()` is already a transaction.** EF wraps all the statements from that call together, so twenty inserts across five tables either all land or none do. Most of the time you write no transaction code.

You only need an explicit one when a single piece of work spans **more than one `SaveChanges()`** and they must succeed or fail together:

```csharp
await using var tx = await db.Database.BeginTransactionAsync();
try
{
 db.Accounts.Add(account);
 await db.SaveChangesAsync();       // save #1
 db.AuditLog.Add(logEntry);
 await db.SaveChangesAsync();       // save #2 — both in one transaction
 await tx.CommitAsync();
}
catch { await tx.RollbackAsync(); throw; }
```

Before writing this, check whether you need two saves at all. Often you can add both entities and save once. Then it is atomic for free and the transaction code goes away.

Keep transactions **short**. Do not call a web service or wait for user input inside one. You hold database locks the whole time, which is how one slow dependency takes down the whole app.

---

### Q27. What is the execution strategy / `EnableRetryOnFailure`, and how does it interact with explicit transactions?

**Answer.** Some database failures are **temporary** — a connection blip, a deadlock, or cloud throttling. The same call would probably work a second later.

An **execution strategy** retries those for you:

```csharp
o.UseSqlServer(conn, sql => sql.EnableRetryOnFailure(maxRetryCount: 5));
```

Turn it on for any cloud database.

**The part that catches people out:** with retries on, `BeginTransaction` throws. EF can retry a single `SaveChanges` because it knows what was in that call. In a manual transaction, though, the unit of work is your own code — several saves plus whatever else. If the second save fails, EF has no idea how to safely redo the first, so it refuses instead of retrying half the work.

The fix is to give the strategy the whole block:

```csharp
var strategy = db.Database.CreateExecutionStrategy();
await strategy.ExecuteAsync(async () =>
{
 await using var tx = await db.Database.BeginTransactionAsync();
 // ... multiple SaveChanges ...
 await tx.CommitAsync();
});
```

⚠️ That block can run more than once, so it has to be safe to repeat. No sending email or charging a card inside it.

---

### Q28. Explain optimistic concurrency with a `RowVersion` token. How do you resolve a conflict?

**Answer.** The problem: two users open the same product at 10:00. One sets the price to 50 and saves. The other sets it to 60 and saves a second later. With no protection the second save wins silently, and the first user's change is gone with nobody told. That is a **lost update**.

**Optimistic** means EF takes no locks. It assumes clashes are rare, lets both users work freely, and detects the clash at save time.

**Both — you need the property and the configuration.**

The property goes on your entity class:

```csharp
public class Product
{
 public int Id { get; set; }
 public decimal Price { get; set; }
 public byte[] RowVersion { get; set; } = default!;   // the token
}
```

Then you mark it as the concurrency token, either way:

```csharp
// Option A — Fluent API, in OnModelCreating (preferred)
mb.Entity<Product>().Property(p => p.RowVersion).IsRowVersion();

// Option B — data annotation, on the property itself
[Timestamp]
public byte[] RowVersion { get; set; } = default!;
```

Pick one, not both. `[Timestamp]` is less typing; Fluent API is preferred on real projects because it keeps mapping out of your domain classes and can do things annotations cannot. Either way the migration creates a SQL Server `rowversion` column, and **the database changes that value automatically on every update** — you never set it yourself.

**How the check works.** Because EF knows the original value it loaded (the snapshot from Q3), it adds that value to the `WHERE` clause:

```sql
UPDATE Products SET Price = 60
WHERE Id = 7 AND RowVersion = 0x0000000000000ABC   -- the value we loaded
```

If nobody else saved, the row still has that `RowVersion`, one row updates, and the database issues a new token. If someone saved first, the row's `RowVersion` has already changed, so **zero rows match**. EF sees 0 rows affected where it expected 1 and throws `DbUpdateConcurrencyException`.

**Resolving the conflict.** The exception hands you the entry, from which you can read both versions:

```csharp
catch (DbUpdateConcurrencyException ex)
{
 var entry = ex.Entries.Single();
 var mine = entry.CurrentValues;                              // what this user typed
 var theirs = await entry.GetDatabaseValuesAsync();           // what is stored now

 // store wins — discard this edit, show them the current data
 entry.CurrentValues.SetValues(theirs);
 entry.OriginalValues.SetValues(theirs);   // accept the new token so a retry can work
}
```

Three strategies:

- **Store wins** — discard this edit, reload, ask the user to redo it. Safest, and the usual default.
- **Client wins** — overwrite with this user's values, as above but keeping `mine`. Fine for low-stakes fields.
- **Merge** — show both values side by side and let the user decide. Best experience, most work.

❌ **Do not** catch the exception and just retry the save in a loop. That is client-wins by accident, and it reintroduces the exact lost update you added the token to prevent.

---

## O8 — DbContext Lifetime & Pooling

> Related: DI lifetimes in [dotnet-platform.md](dotnet-platform.md).

### Q29. What lifetime should `DbContext` have, and what is the captive-dependency trap?

**Answer.** **Scoped** — one context per request. `AddDbContext<T>()` does that by default, and it matches the unit of work idea from Q1. The context is created for the request, tracks changes, saves once, and is disposed at the end.

```csharp
services.AddDbContext<AppDbContext>(o => o.UseSqlServer(conn)); // scoped by default
```

**The classic bug is putting a scoped context into a singleton.** A singleton is built once and keeps the first context it is given for the life of the app. That is a **captive dependency**, and it goes wrong two ways:

- **It is never disposed**, so the change tracker keeps every entity it has ever loaded. Memory grows, and stale data keeps coming back from the identity map.
- **Every request shares one context**, so requests running at the same time all hit the same object that is not thread-safe. That is the Q4 failure.

The general rule: **something long-lived must never hold on to something short-lived.** Background services count as singletons here ([dotnet-aspnetcore-fundamentals.md](dotnet-aspnetcore-fundamentals.md), Q23).

If a singleton needs database access, give it `IDbContextFactory<T>` (Q31) so it can create contexts instead of holding one.

---

### Q30. What is `AddDbContextPool` and what are its caveats?

**Answer.** `AddDbContextPool` keeps a pool of contexts to reuse. A request borrows one, and when it is disposed EF resets it — clearing the change tracker — and puts it back.

```csharp
services.AddDbContextPool<AppDbContext>(o => o.UseSqlServer(conn), poolSize: 128);
```

That saves the cost of building a context, which only matters on a busy service handling many small requests.

❌ **The catch that really bites:** the reset clears **EF's** state, not yours. Your own fields on the context survive into the next request:

```csharp
public string? CurrentTenantId { get; set; }  // ❌ not cleared by pooling
```

Set that per request, and pooling hands it over still filled in to the next request — which might be a **different tenant**. It is hard to reproduce, because it depends on the order requests arrive in.

There is another limit. The pool controls construction, so your constructor can only take `DbContextOptions`. You cannot inject the current user or a tenant provider, which is exactly what multi-tenancy needs.

**Use plain `AddDbContext` unless you have measured context creation as a real bottleneck.**

---

### Q31. When do you use `IDbContextFactory` / `AddDbContextFactory`?

**Answer.** When one-context-per-request does not fit and you need to create contexts yourself:

- **Background services and singletons** — there is no request, so there is no scope. Injecting a context straight in is the Q29 bug.
- **Parallel work** — `Task.WhenAll` over separate database calls needs one context per task (Q4).
- **Blazor Server** — a connection can stay open for hours, and components can trigger overlapping work. One scoped context would be both long-lived and shared.

```csharp
services.AddDbContextFactory<AppDbContext>(o => o.UseSqlServer(conn));

public class ReportJob(IDbContextFactory<AppDbContext> factory)
{
 public async Task RunAsync()
 {
 await using var db = await factory.CreateDbContextAsync();  // yours to dispose
 // ... one job of work ...
 }
}
```

Two rules. **Dispose what you create** — use `await using` every time. And in a long-running loop, create a context **per pass**, or the change tracker grows all night and you have rebuilt the captive dependency by hand.

---

### Q32. How do EF Core Interceptors (`SaveChangesInterceptor`, `DbCommandInterceptor`) work, and how are they used for audit logging (`CreatedAt`, `UpdatedAt`), soft-deletes, or multi-tenant filtering?

**Answer.** **EF Core Interceptors** allow you to hook into, observe, or modify EF Core operations before or after they execute. Unlike DbContext overrides, interceptors are reusable, decoupled classes registered during options setup (`optionsBuilder.AddInterceptors(...)`).

Two primary types of interceptors:
- **`SaveChangesInterceptor`**: Intercepts `DbContext.SaveChanges()` and `SaveChangesAsync()`. Perfect for modifying entity states right before they are written to the database.
- **`DbCommandInterceptor`**: Intercepts raw ADO.NET SQL command creation, execution, and scalar queries. Used for query logging, SQL comment injection (tagging queries with correlation IDs), or custom command timeouts.

```csharp
public class AuditAndSoftDeleteInterceptor : SaveChangesInterceptor
{
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken cancellationToken = default)
    {
        if (eventData.Context is null) return base.SavingChangesAsync(eventData, result, cancellationToken);

        var entries = eventData.Context.ChangeTracker.Entries();

        foreach (var entry in entries)
        {
            // 1. Automatic Timestamp Auditing
            if (entry.Entity is IAuditable auditable)
            {
                if (entry.State == EntityState.Added)
                {
                    auditable.CreatedAt = DateTimeOffset.UtcNow;
                }
                else if (entry.State == EntityState.Modified)
                {
                    auditable.UpdatedAt = DateTimeOffset.UtcNow;
                }
            }

            // 2. Soft-Delete Transformation (Turn State.Deleted into State.Modified)
            if (entry.Entity is ISoftDeletable softDeletable && entry.State == EntityState.Deleted)
            {
                entry.State = EntityState.Modified;
                softDeletable.IsDeleted = true;
                softDeletable.DeletedAt = DateTimeOffset.UtcNow;
            }
        }

        return base.SavingChangesAsync(eventData, result, cancellationToken);
    }
}

// Registration in Program.cs / AddDbContext:
builder.Services.AddSingleton<AuditAndSoftDeleteInterceptor>();

builder.Services.AddDbContext<AppDbContext>((sp, options) =>
{
    var interceptor = sp.GetRequiredService<AuditAndSoftDeleteInterceptor>();
    options.UseSqlServer(connectionString)
           .AddInterceptors(interceptor);
});
```

**Key Takeaway.** Interceptors keep cross-cutting concerns (auditing, tenant scoping, soft-deletes) out of your application logic. Note that when using singleton interceptors with DI, ensure they do **not** inject scoped dependencies (like `DbContext` or `IHttpContextAccessor`) in their constructor—resolve scoped dependencies via `eventData.Context` instead.

---

### Q33. What are Shadow Properties and Value Converters in EF Core, and when should you use them?

**Answer.** Both features give you fine-grained control over how database columns map to C# entity domain models without corrupting clean domain boundaries.

#### 1. Shadow Properties
A **Shadow Property** is a property that exists in the EF Core conceptual model and database schema, but is **not defined** as a C# field/property on the entity class.

**Use cases**: Audit columns (`LastModifiedBy`), foreign keys where navigation properties exist but exposed ID properties are unwanted, or tenancy discriminator keys.

```csharp
// Configure in OnModelCreating:
modelBuilder.Entity<Order>()
    .Property<DateTime>("LastUpdated"); // Shadow property!

// Setting a shadow property value before saving:
context.Entry(order).Property("LastUpdated").CurrentValue = DateTime.UtcNow;

// Querying by shadow property:
var recentOrders = await context.Orders
    .Where(o => EF.Property<DateTime>(o, "LastUpdated") > cutoffDate)
    .ToListAsync();
```

#### 2. Value Converters
A **Value Converter** transforms data between the database column format and the C# entity property type when reading or writing.

**Use cases**: Storing strongly-typed IDs (e.g. `UserId` wrapper over `Guid`), storing enums as strings in DB (`string` vs `int`), or encrypting sensitive string fields (`SSN`) at rest.

```csharp
// Store Enum as String in DB
modelBuilder.Entity<Order>()
    .Property(o => o.Status)
    .HasConversion<string>(); // "Pending", "Shipped" instead of 0, 1

// Custom Converter: List<string> to JSON column string
var jsonConverter = new ValueConverter<List<string>, string>(
    v => JsonSerializer.Serialize(v, (JsonSerializerOptions)null!),
    v => JsonSerializer.Deserialize<List<string>>(v, (JsonSerializerOptions)null!) ?? new());

modelBuilder.Entity<User>()
    .Property(u => u.Roles)
    .HasConversion(jsonConverter);
```

