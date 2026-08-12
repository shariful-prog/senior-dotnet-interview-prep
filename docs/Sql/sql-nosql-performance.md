# Y. SQL vs NoSQL & Performance Practices
---

## Y1 — SQL vs NoSQL

### Q1. What is the fundamental difference between a relational database and NoSQL?

A **relational database** (SQL Server, PostgreSQL, MySQL, Oracle) stores data in **tables with a fixed schema** — rows and typed columns, related by keys. Its defining traits:

- **ACID transactions** (Atomicity, Consistency, Isolation, Durability) — strong guarantees.
- **Declarative SQL** — you say *what* you want, the optimizer decides *how*.
- **Normalization + joins** — data lives once and is combined at query time.
- **Schema-on-write** — the shape is enforced when you insert.

**NoSQL** ("Not Only SQL") is an umbrella term for non-relational stores that trade some of those guarantees for **schema flexibility** and **horizontal scalability**. Common traits:

- **Flexible / schema-on-read** — each record can have a different shape.
- **Aggregate-oriented / denormalized** — data you read together is stored together (often duplicated) to avoid joins.
- **Horizontal scale-out** — designed to spread across many commodity nodes.
- **Weaker or tunable consistency** — often eventual consistency for availability.

The key mental model: **relational optimizes for flexible querying of normalized data; NoSQL optimizes for scaling a known access pattern.** Relational databases fit the vast majority of OLTP workloads — do not reach for NoSQL by default.

**Follow-up — "Is NoSQL 'schemaless'?"** No. The schema still exists; it simply moves from the database layer into your **application code**:

- **SQL (*Schema-on-Write*)**: The database acts as a strict guard — if your `Users` table defines `Age INT NOT NULL`, the database **rejects** any record where `Age` is missing or passed as `"thirty"`. Every row is guaranteed to follow the same blueprint.
- **NoSQL (*Schema-on-Read*)**: The database is passive — it blindly accepts whatever you give it. So you can end up with:
    - Doc 1: `{ "name": "Alice", "age": 30 }`
    - Doc 2: `{ "name": "Bob", "user_age": "thirty" }` ← different field name, wrong type, no error from DB.

  But your **application code** still expects a specific shape:
  ```csharp
  string name = user.Name;
  int age = user.Age; // Expects an int — will crash on Doc 2!
  ```
  The database didn't complain — it passed the problem to **your code**. Now your code must be full of null checks and fallback logic to handle missing/renamed/mistyped fields.

- **The Trade-off**: You gain deployment speed (no heavy `ALTER TABLE` migrations), but trade it for application complexity — handling null checks, missing fields, and data drift over time in code. The schema didn't disappear; the **developer** now manages it instead of the database.

---

### Q2. What are the four main families of NoSQL databases, and what is each good for?

| Family | Model | Examples | Sweet spot |
|--------|-------|----------|------------|
| **Document** | JSON/BSON documents, nested, flexible schema | MongoDB, Azure Cosmos DB, Couchbase | Aggregate-oriented data (a whole order + line items in one doc), content/catalogs, evolving schemas |
| **Key-Value** | Opaque value keyed by an ID, ultra-simple | Redis, DynamoDB, Memcached | Caching, session state, feature flags, rate-limit counters — ultra-fast point lookups |
| **Column-Family (wide-column)** | Rows with dynamic wide columns, grouped into families | Cassandra, HBase, ScyllaDB | Write-heavy, time-series, IoT telemetry, event logs at massive scale |
| **Graph** | Nodes + edges (relationships are first-class) | Neo4j, Amazon Neptune | Relationship-heavy queries — social graphs, recommendations, fraud rings, hierarchies |

**How to reason about them:**

- **Document** = "store the aggregate the way I read it." Great when your unit of access is a self-contained object. Weak at ad-hoc cross-document joins.
- **Key-Value** = the simplest, fastest possible store: `GET key` / `SET key value`. No querying by value. This is why Redis dominates caching (cross-ref [.NET performance & caching](../Dotnet/dotnet-performance-resilience.md)).
- **Column-Family** = optimized for enormous write throughput and range scans over a partition key. You design tables **per query**, not per entity.
- **Graph** = when the **relationships** are the point. A "friends-of-friends who like X" query that is a nightmare of self-joins in SQL is a natural traversal in a graph DB.

**Follow-up — "Where does Cosmos DB fit?"** Azure Cosmos DB is multi-model — it exposes document (SQL/Mongo API), key-value (Table API), column (Cassandra API), and graph (Gremlin API) over one engine, with globally distributed, **tunable consistency** (five levels from strong to eventual).

---

### Q3. When would you actually choose NoSQL over a relational database?

Choose NoSQL when the workload's characteristics line up with a family's strengths **and** relational hits a wall:

**Good reasons to go NoSQL:**
- **Massive horizontal scale / throughput** beyond what one relational box can do (millions of writes/sec) → Cassandra/DynamoDB.
- **Schema volatility** — the shape changes constantly or varies per tenant → document.
- **Aggregate access pattern** — you always read/write one self-contained object → document.
- **Sub-millisecond point lookups / caching** → key-value (Redis).
- **Relationship traversal** as the core query → graph.
- **Known, fixed query patterns** you can design the data model around.

**Bad reasons (red flags in an interview):**
- "It's newer / trendier."
- "I don't want to design a schema" (you still have one, now in code).
- "I need joins and ad-hoc reporting" — that is exactly what relational is *for*.
- Complex multi-entity transactions across the dataset.

**The senior framing — polyglot persistence:** Use the **right store per need**, not one store for everything. A real system might use SQL Server for transactional order data, Redis for caching/sessions, and Elasticsearch for full-text search. The cost of polyglot persistence is operational complexity and keeping stores in sync — so add stores deliberately, not reflexively.

**Follow-up — "You're building a standard e-commerce backend. SQL or NoSQL?"** Relational as the system of record — orders, payments, inventory need ACID and cross-entity integrity. Add Redis for cart/session and product-page caching, and possibly a search index. Relational is the default; NoSQL is a supplement for specific access patterns.

---

### Q4. Explain the CAP theorem and how it relates to choosing a database.

**CAP theorem:** in a distributed data store, during a **network Partition** you can guarantee at most **two** of:

- **C — Consistency:** every read sees the most recent write (all nodes agree).
- **A — Availability:** every request gets a (non-error) response.
- **P — Partition tolerance:** the system keeps working despite dropped/delayed messages between nodes.

Because network partitions **will** happen in any distributed system, **P is not optional** — so the real, practical choice under a partition is **C vs A**:

- **CP** — stay consistent, refuse/block requests that can't be made consistent (e.g., a single-primary relational cluster, MongoDB with majority writes, HBase). Correct but may reject during a partition.
- **AP** — stay available, allow reads/writes that may be stale and reconcile later (e.g., Cassandra, DynamoDB, Riak — **eventual consistency**). Always answers, but may return stale data.

**Tie-in to the SQL/NoSQL choice:**
- Traditional **relational** systems lean **CP** — they prioritize consistency (ACID).
- Many **NoSQL** systems lean **AP** or offer **tunable consistency** — you pick per operation (e.g., Cassandra's `QUORUM` vs `ONE`; Cosmos DB's five consistency levels).

**Nuances a senior should add:**
- CAP only bites **during a partition**. When the network is healthy, you get both C and A — it's not a permanent tax.
- **PACELC** extends CAP: *if Partition then C-vs-A, Else (normal operation) Latency-vs-Consistency.* This captures the everyday trade-off CAP ignores — even with no partition, stronger consistency costs latency.
- Don't overstate it: "ACID vs eventual" is a spectrum, and modern systems (Spanner, Cosmos DB) offer strong consistency at scale with clever engineering.

*(CAP also appears in system design — keep it brief here and cross-reference system-design notes for the deeper distributed-systems treatment.)*

---

## Y2 — Query Optimization Workflow

### Q6. What does `SET STATISTICS IO ON` tell you, and why do you trust logical reads over elapsed time?

`SET STATISTICS IO ON` reports, per table, the page reads a query performed:

- **Logical reads** — pages read **from the buffer pool (memory)**. This is the primary cost metric.
- **Physical reads** — pages read from **disk** (cache misses).
- **Read-ahead reads** — pages prefetched.
- **LOB logical reads** — for large object (varchar(max), etc.) data.
- **Worktable / workfile** reads — spills to tempdb for sorts, hashes, spools.

**Why logical reads over wall-clock time:** elapsed time is **noisy** — it swings with server load, whether data is cached, blocking, and network. **Logical reads are stable and deterministic** for a given plan and data — the same query does the same page reads every run. So when you tune, you compare logical reads: dropping from 50,000 reads to 200 is a real, reproducible win regardless of what else the box is doing.

A rule of thumb: a query doing far more logical reads than the number of rows it returns is doing too much work (often a scan that should be a seek, or a bad join order).

---

### Q7. You added the "obvious" index but the query is still slow. What now?

Systematic next steps:

1. **Confirm the index is actually used.** Check the plan — is it a seek on your index, or still a scan? A predicate might be **non-SARGable** so the optimizer can't use the index at all (Y3).
2. **Check for a Key Lookup.** The index may be *seeked* but not **covering** — every matched row jumps to the clustered index for other columns. Make it a covering index with `INCLUDE`d columns.
3. **Stale statistics / bad cardinality estimate.** Actual rows ≫ estimated rows → optimizer chose a bad plan (nested loops on millions of rows). `UPDATE STATISTICS ... WITH FULLSCAN`.
4. **Parameter sniffing** (Y4) — the cached plan was built for atypical parameter values. Symptom: fast for some inputs, slow for others.
5. **The query is doing genuinely too much** — e.g., an implicit sort, a `DISTINCT` hiding a bad join producing duplicates, or a function forcing row-by-row evaluation. Rewrite it.
6. **The index is a poor fit** — wrong leading column for this predicate, or low selectivity (indexing a column with 2 distinct values rarely helps).
7. **Blocking, not the plan** — the query is fast alone but waits on locks under load. Check `sys.dm_exec_requests` wait types; that's a concurrency problem, not a plan problem.

The discipline: re-read the **actual** plan after each change rather than adding indexes hopefully.

---

## Y3 — Common SQL Anti-Patterns

### Q8. What is `SELECT *` and why is it an anti-pattern in application code?

`SELECT *` fetches **every column**. Problems:

- **Wasted I/O and network** — you drag back columns (including big `varchar(max)`/blob data) you never use.
- **Breaks covering indexes** — a narrow covering index can't satisfy `SELECT *`, so the query falls back to a **Key Lookup** or clustered scan.
- **Fragile** — if a column is added/renamed/reordered, results and downstream code (ORM mappings, positional reads) break unexpectedly.
- **Hides intent** — a reader can't tell what the query actually needs.

**Fix:** list exactly the columns you need. This makes covering indexes possible, reduces payload, and documents intent.

```sql
-- Anti-pattern
SELECT * FROM Orders WHERE CustomerId = @id;

-- Better: only what you need → can be fully covered by an index
SELECT OrderId, OrderDate, TotalAmount FROM Orders WHERE CustomerId = @id;
```

*(Ad-hoc exploration in SSMS is fine; the anti-pattern is `SELECT *` baked into application/production queries.)*

---

### Q10. What is the N+1 query problem?

**N+1** is when the app runs **1 query** to fetch a list of parent rows, then **N additional queries** — one per parent — to fetch each parent's children. 100 orders → 1 + 100 = 101 round-trips instead of 1 or 2.

```csharp
// Anti-pattern (lazy loading fires a query per order)
var orders = db.Orders.ToList(); // 1 query
foreach (var o in orders)
 Console.WriteLine(o.Customer.Name); // N queries, one per order
```

It's usually an **ORM lazy-loading** trap and often invisible until you watch the SQL. Round-trips dominate the cost — latency, not row count, kills you.

**Fix:** fetch related data in one shot — eager-load with a join (`Include` in EF Core), a projection, or a batched second query. In EF Core this is `.Include(o => o.Customer)` or a `Select` projection; EF Core can also split into a small fixed number of queries (`AsSplitQuery`).

*(EF Core N+1, `Include`, split queries, and projections → [EF Core notes](../Dotnet/dotnet-ef-core.md).)*

---

## Y4 — Parameterization, Plan Cache & Parameter Sniffing

### Q12. Why do parameterized queries perform better than concatenated SQL?

When SQL Server runs a query, it **compiles an execution plan** (expensive) and **caches** it keyed by the query text. Parameterization changes that in two ways:

**1. Plan reuse.** A **parameterized** query has stable text (`WHERE Id = @id`) so the same cached plan is reused across executions with different values — you pay compilation **once**.

**Ad-hoc concatenated SQL** embeds literals in the text:
```sql
WHERE Id = 42 -- one plan
WHERE Id = 43 -- different text → different plan
```
Every distinct value produces a **new plan**, **bloating the plan cache** with thousands of single-use plans, wasting memory and CPU recompiling. (`OPTIMIZE FOR AD HOC WORKLOADS` mitigates this by caching a stub first, but parameterization is the real fix.)

**2. SQL injection prevention.** Parameters are sent **separately** from the query text, so user input can never be interpreted as SQL. Concatenation is the classic injection vector.

```csharp
// concatenation — injectable + plan-cache bloat
var sql = "SELECT * FROM Users WHERE Name = '" + name + "'";

// parameterized — reused plan + safe
cmd.CommandText = "SELECT Id, Name FROM Users WHERE Name = @name";
cmd.Parameters.Add("@name", SqlDbType.NVarChar).Value = name;
```

**EF Core / Dapper parameterize automatically** — LINQ predicates and Dapper's anonymous-object params become `@p0`, `@p1`, so you get both benefits for free (just don't hand-concatenate with `FromSqlRaw` + string building).

*(SQL injection in depth → [.NET security](../Dotnet/dotnet-security.md). EF parameterization → [EF Core O2](../Dotnet/dotnet-ef-core.md#o2--linqsql-translation--iqueryable-deferred-execution).)*

---

### Q13. What is parameter sniffing? Why is it both good and bad?

**Parameter sniffing** is the optimizer's behavior of **"sniffing" the parameter values on the first execution** and building a plan optimized for **those specific values** — then caching and reusing that plan for all later executions.

**Why it's usually good:** a plan tailored to real values (using column statistics) is far better than a generic guess. Most of the time you *want* it.

**Why it can be bad — the classic symptom:** with **skewed data**, the plan that's optimal for the *first* parameter can be **terrible** for others.

Example: an `Orders` table where 99% of rows are `Status = 'Closed'` and 1% are `'Open'`.
- If the first call passes `'Open'` (few rows), the optimizer picks a **nested-loop + index seek** plan — perfect for a few rows.
- That plan is cached. The next call passes `'Closed'` (millions of rows) and reuses the seek/loop plan → millions of individual lookups → agonizingly slow. A **scan/hash** would have been right.
- Reverse the first call and you get the opposite problem.

This is the **"fast for some inputs, slow for others"** signature — and it can flip after a plan-cache eviction, a stats update, or a server restart, making it maddening to reproduce.

---

### Q14. How do you mitigate a bad parameter-sniffing situation?

Options, roughly in order of how targeted they are:

- **`OPTION (RECOMPILE)`** — compile a **fresh plan every execution** using the actual runtime values. Best per-value plan, but pays compilation cost each run. Great for rare, expensive, skewed queries; bad for high-frequency OLTP.
 ```sql
 SELECT ... FROM Orders WHERE Status = @status OPTION (RECOMPILE);
 ```
- **`OPTIMIZE FOR (@p = value)`** — build the cached plan for a **specific representative value** you choose.
 ```sql
 ... WHERE Status = @status OPTION (OPTIMIZE FOR (@status = 'Closed'));
 ```
- **`OPTIMIZE FOR UNKNOWN`** — ignore the sniffed value and use the column's **average density** to build an "average" plan — no single input is great, but none is catastrophic.
- **Query Store plan forcing** — spot a good historical plan in Query Store and **force** it. Powerful for production incidents; no code change.
- **Local variables** — assigning the parameter to a local var inside the proc disables sniffing (acts like `OPTIMIZE FOR UNKNOWN`); an older trick, prefer the explicit hints.
- **Refactor** — split into separate procs/queries per value class, or add a filtered index for the skewed subset.

**Interview-grade answer:** name the symptom ("fast for some params, slow for others → likely parameter sniffing on skewed data"), then reach for `OPTION (RECOMPILE)` for a rare heavy query or **Query Store plan forcing** for a live incident, and consider refactoring if it's chronic.

---

## Y5 — Batching & Bulk Operations

### Q15. Why is batching important, and how do you send many rows efficiently?

The dominant cost of database work from an app is often the **network round-trip** — each statement is a request/response with latency. Inserting 10,000 rows as 10,000 separate `INSERT` calls means 10,000 round-trips; batch them and you collapse that into a handful.

**Techniques, smallest to largest scale:**

- **Batch multiple statements** in one command / one round-trip.
- **Table-Valued Parameters (TVPs)** — pass a whole set of rows as a single parameter, then `INSERT ... SELECT FROM @tvp` or `MERGE` server-side. Great for hundreds–low thousands of rows.
 ```sql
 CREATE TYPE dbo.OrderLineType AS TABLE (
 ProductId INT, Qty INT, Price DECIMAL(10,2));
 -- proc takes @lines dbo.OrderLineType READONLY, then:
 INSERT INTO OrderLines (ProductId, Qty, Price)
 SELECT ProductId, Qty, Price FROM @lines;
 ```
- **`MERGE` / set-based upsert** — insert-or-update a whole batch in one statement instead of per-row check-then-write. *(Note: `MERGE` has had concurrency/bug caveats historically; some teams prefer an explicit `UPDATE` then `INSERT` — mention you're aware.)*
- **`SqlBulkCopy`** — the fastest path for large loads (tens of thousands+); streams rows via the bulk API, bypassing per-row overhead. Use for ETL/imports.

**On the .NET side:**
- **EF Core `AddRange` + one `SaveChanges`** batches inserts (EF Core auto-batches multiple statements per round-trip).
- **EF Core 7+ `ExecuteUpdate` / `ExecuteDelete`** run **set-based** bulk update/delete in a single SQL statement — no load-into-memory-then-save. Use these instead of loading entities to modify them.
 ```csharp
 // One UPDATE statement, no entities loaded
 db.Orders.Where(o => o.Status == "Pending" && o.Created < cutoff)
 .ExecuteUpdate(s => s.SetProperty(o => o.Status, "Expired"));
 ```

*(EF bulk operations, `AddRange`, `ExecuteUpdate/Delete`, and third-party bulk-insert libraries → [EF Core notes](../Dotnet/dotnet-ef-core.md).)*

---

### Q16. What's the trade-off between one big transaction and many small ones when processing a large batch?

Wrapping a huge operation in **one transaction** vs **chunking** it into smaller committed batches:

**One big transaction:**
- All-or-nothing atomicity — clean rollback on failure.
- **Locks held for the whole duration** → blocking, potential lock escalation to a table lock, deadlocks.
- **Transaction log growth** — the log can't truncate until commit; a huge txn can blow up the log file.
- On failure, **rollback is expensive** and re-runs the whole thing.

**Chunked (e.g., 5,000 rows per committed batch in a loop):**
- Locks released between batches → far less blocking.
- Log can truncate between commits (in simple/bulk-logged recovery) → bounded log growth.
- On failure, only the current chunk is lost; you can resume.
- **Not globally atomic** — partial completion is possible, so the operation must be **idempotent/resumable** (e.g., process by an ID range or a "processed" flag).

**Senior guidance:** for large maintenance/ETL operations, **chunk it** — pick a batch size (commonly 1k–10k rows), commit per batch, and make it resumable. Reserve one big transaction for when true all-or-nothing atomicity across the whole set is a hard requirement.

```sql
-- Chunked delete pattern
WHILE 1 = 1
BEGIN
 DELETE TOP (5000) FROM Logs WHERE Created < @cutoff;
 IF @@ROWCOUNT = 0 BREAK;
 -- optional: WAITFOR DELAY to ease pressure
END
```

---

## Y6 — Scaling Databases (Overview)

### Q17. What's the difference between vertical and horizontal scaling for a database?

- **Vertical scaling (scale up)** — give the **single server** more resources: CPU, RAM, faster disk. Simplest — no app changes, no distributed-systems complexity, ACID stays trivial. But it has a **hard ceiling** (biggest box you can buy) and gets expensive at the top, and it's a single point of failure.
- **Horizontal scaling (scale out)** — add **more servers** and spread load/data across them: read replicas, partitioning, sharding, caching. Effectively unbounded scale and better fault tolerance, but introduces **replication lag, eventual consistency, cross-node joins/transactions, and operational complexity.**

**Order of operations a senior recommends:** exhaust the cheap wins first — **tune queries and indexes** (Y2), then **scale up**, then **add caching**, then **read replicas**, and only reach for **sharding** when nothing else suffices. Sharding is powerful but the most invasive; don't lead with it.

---
### Q19. Explain partitioning vs sharding — they're often confused.

Both split a big table into smaller pieces, but at **completely different levels**. Think of it with a **library analogy**:

**Partitioning = One library, books organized into labeled shelves**

You have **one big library** (one database server). Your `Orders` table has 500 million rows. Instead of one giant pile, you split it into **sections by year** — all inside the **same building**.

**Important: This is physical partitioning managed by SQL Server, not you.** You do NOT create separate tables like `Orders_2022`, `Orders_2023`. You still have **one `Orders` table**. You tell SQL Server *how* to split it using a **partition function** and **partition scheme**, and the engine **physically stores the rows in separate internal filegroups** behind the scenes.

```sql
-- Step 1: Define HOW to split (by year boundaries)
CREATE PARTITION FUNCTION pf_OrderDate (DATE)
AS RANGE RIGHT FOR VALUES ('2022-01-01', '2023-01-01', '2024-01-01');

-- Step 2: Define WHERE to store each partition (filegroups)
CREATE PARTITION SCHEME ps_OrderDate
AS PARTITION pf_OrderDate TO (fg_2021, fg_2022, fg_2023, fg_2024);

-- Step 3: Create the table ON the partition scheme
CREATE TABLE Orders (
    OrderId INT, OrderDate DATE, Amount DECIMAL(10,2)
) ON ps_OrderDate(OrderDate);
```

**To you and your app, it's still just `SELECT * FROM Orders`** — one table, one connection string. But under the hood, SQL Server physically stores 2022 rows in one filegroup, 2023 in another, etc.

- The optimizer knows which partition to look at. Query for 2024 orders? It **skips** 2022 and 2023 entirely → **partition elimination**.
- You can **archive** old data instantly with `SWITCH` partition (moves an entire year's data to an archive table in milliseconds — no row-by-row delete).
- Your app doesn't even know partitions exist.

```sql
-- Your app just queries normally — the engine handles partition elimination automatically
SELECT * FROM Orders WHERE OrderDate >= '2024-01-01' AND OrderDate < '2025-01-01'
-- ↑ SQL Server only scans the 2024 partition, ignores the rest
```

**Sharding = Multiple libraries in different cities**

Your single library can't hold all the books anymore. So you build **separate libraries in different cities** (separate database servers):
- Library Dhaka (Shard 1): Customers A–M
- Library Chittagong (Shard 2): Customers N–Z

Each library is a **completely independent database** with its own storage, CPU, and RAM.

- Your **app must know which library to visit** — a routing layer decides: "Customer 'Rahman' → starts with R → go to Shard 2."
- **Massive scale** — you can add more cities (shards) as you grow.
- **But:** if you need data from *both* libraries at once (e.g., "total revenue across all customers"), you must ask **both**, then merge the results yourself → this is the pain.

**Side-by-side comparison:**

| Aspect | Partitioning | Sharding |
| :--- | :--- | :--- |
| **Where** | One table split into pieces **within one DB** | Data split **across multiple independent DBs/servers** |
| **Analogy** | One library, organized shelves | Multiple libraries in different cities |
| **App awareness** | Transparent — app doesn't know | App **must** route to the correct shard |
| **Purpose** | Manageability + query performance (partition elimination) | Massive horizontal scale (storage + write throughput) |
| **Joins** | Normal — all data is in the same DB | **Cross-shard joins are painful** — scatter-gather + merge in app |
| **Transactions** | Normal ACID — single DB | Cross-shard transactions are **very hard** — lose single-node ACID |
| **Complexity** | Low — mostly a DBA task | High — shard key choice, routing, rebalancing |

**The hard part of sharding — choosing the shard key:**
- Must spread load **evenly** (avoid hot shards) and align with your **most common query** so most queries hit one shard.
- **Cross-shard joins and transactions are hard/expensive** — queries spanning shards must scatter-gather and merge in the app.
- **Rebalancing** (adding new shards) is painful once live. Hash/consistent-hashing helps.

**Interview one-liner:** *Partitioning splits a table within one DB for manageability and faster queries; sharding splits data across many DBs for massive scale — but cross-shard joins and transactions become the hard problem.*

*(Sharding implementation deep dive is a system-design topic — this is an overview.)*

---

### Q20. Where does caching fit in a database scaling strategy?

**What is caching?** Storing frequently accessed data in a **faster layer** (usually in-memory) so that repeated reads don't hit the database every time. This reduces DB load and speeds up response times dramatically.

**SQL Server has built-in caching:**

- **Buffer Pool** — SQL Server automatically caches **data pages** (rows/indexes) in RAM. When you query a table, it reads from disk *once* and keeps those pages in memory. Subsequent queries for the same data read from RAM, not disk. You don't configure this — SQL Server manages it automatically and uses as much RAM as you give it.
- **Plan Cache** — SQL Server caches **compiled execution plans**. The first time a parameterized query runs, it compiles a plan (expensive). Every subsequent execution with different parameter values **reuses** the same cached plan (cheap). This is why parameterized queries are faster than ad-hoc SQL.

These are **automatic** — you don't write code for them. But they only help within SQL Server itself. If your app makes 10,000 requests/sec for the same product catalog, SQL Server still processes 10,000 queries even if the data is in the Buffer Pool.

**Application-level caching (Redis / Memcached):**

This is where you put a **distributed in-memory store** (typically **Redis**) between your app and the database. The app checks Redis first; if the data is there, the query **never reaches SQL Server at all**.

**Cache-aside pattern (the most common):**

```
1. App receives request: "Get product #42"
2. App checks Redis: "Do I have product #42?"
   ├── YES (cache hit)  → Return instantly. DB is never touched.
   └── NO  (cache miss)  → Continue to step 3.
3. App queries SQL Server → gets product #42
4. App stores the result in Redis with a TTL (e.g., 5 minutes)
5. Returns the result to the user
6. Next request for #42 within 5 min → served from Redis (step 2 = hit)
```

**What to cache:** product catalogs, config/settings, user sessions, dashboard aggregates, lookup tables — anything **read frequently but written rarely**.

**The hard problem — cache invalidation:**

- If someone updates the product price in SQL Server, Redis still serves the **old price** until the TTL expires → **stale data**.
- **Fix:** invalidate (delete) the Redis key on every write to that data, or use short TTLs for frequently changing data. Accept that the read side is **eventually consistent** within the TTL window.

**Cache stampede:** if a popular key expires and 1,000 requests arrive at the same time, they all miss Redis and slam SQL Server simultaneously. Fix with **locking** (only one request fetches from DB, others wait for it to populate the cache) or staggered TTLs.

*(Redis, cache-aside, and `IDistributedCache` in .NET → [.NET performance & caching](../Dotnet/dotnet-performance-resilience.md).)*

---

### Q21. Walk me through how you'd diagnose and fix a slow query in production.

Think of it like being a **doctor** — you don't jump to surgery. You start with the cheapest, least invasive fix and escalate only if it doesn't work.

**Step 1: Find the slow query (Which patient is sick?)**

You can't fix what you can't see. Use monitoring tools to identify the exact query:

- **SQL Server:** Query Store, `sys.dm_exec_query_stats`, Extended Events.
- **APM tools:** Application Insights, Datadog, New Relic — they show you which DB calls are slow from the app side.

**Step 2: Read the Execution Plan (Run the X-ray)**

Run `SET STATISTICS IO ON` and look at the **actual execution plan**. You're looking for red flags:

- **Table Scan / Index Scan** instead of Index Seek → the query is reading the *entire* table instead of jumping directly to the rows it needs. Like reading an entire book to find one sentence instead of using the index at the back.
- **Key Lookup** → the index found the row, but then had to go back to the main table to grab extra columns. Wasteful extra trip.
- **Sort / Hash spills to disk** → not enough memory; the engine spills to slow disk.
- **Estimated vs Actual rows are wildly different** → stale statistics are misleading the optimizer.

**Step 3: Fix it — cheapest fix first, then escalate**

1. **Add or improve an index** — This is the **#1 fix** for most slow queries. If the plan shows a scan, create a covering index (includes the columns the query needs). This alone fixes ~70% of slow queries. **Free and biggest ROI.**

2. **Rewrite the query to be SARGable** — If you're wrapping an indexed column in a function, the index can't be used:
    ```sql
    -- ❌ BAD: index on OrderDate is useless here
    WHERE YEAR(OrderDate) = 2024

    -- ✅ GOOD: index can now seek directly
    WHERE OrderDate >= '2024-01-01' AND OrderDate < '2025-01-01'
    ```

3. **Kill the N+1 problem** — If your app fires 100 separate queries in a loop instead of one batched query, fix it with eager loading (`Include()` in EF) or a single `JOIN`.

4. **Fix parameter sniffing** — If the query is "fast for some users, slow for others," the execution plan was cached for the *first* user's data distribution and is terrible for everyone else. Fix with `OPTION (RECOMPILE)` or Query Store plan forcing.

5. **Add caching (Redis)** — For read-heavy, hot data that doesn't change every second (product catalogs, config, dashboards), put a cache in front. The query doesn't even hit the DB on cache hits.

6. **Scale up the server** — More RAM / CPU. Simple, buys headroom, but has a ceiling and costs money.

7. **Add read replicas** — Route read queries to replica servers, freeing the primary for writes. Accept a small replication lag (eventual consistency).

8. **Partition / Shard** — Split large tables within one DB (partitioning) or across multiple DBs (sharding). **Most invasive, last resort.**

**The meta-point interviewers want to hear:** Don't jump to exotic scaling to hide an unindexed query or an N+1. **"Measure first, fix the query, then scale"** — and match the tool to whether the bottleneck is reads, writes, storage, or latency.

---

### Cheat sheet (last-minute — whole section)

**SQL vs NoSQL**
- **Relational** = ACID, fixed schema, joins, SQL, schema-on-write. Default for OLTP.
- **NoSQL families:** **Document** (MongoDB/Cosmos — JSON aggregates, flexible schema) · **Key-Value** (Redis/DynamoDB — fast lookups, caching/session) · **Column-family** (Cassandra/HBase — write-heavy, time-series) · **Graph** (Neo4j — relationship traversal).
- NoSQL trade: schema flexibility + horizontal scale **vs** weaker/tunable consistency, limited joins/txns, denormalized/duplicated data, **query patterns known upfront**. "Schemaless" = schema moved into app code.
- **Choose NoSQL** for massive scale, known access pattern, schema volatility, or relationship-first — **not** by default. **Polyglot persistence** = right store per need.
- **CAP:** under a **Partition**, pick **C or A**. Relational ≈ **CP**; many NoSQL **AP** / tunable. P is mandatory (partitions happen). **PACELC** adds latency-vs-consistency in normal operation.

**Optimization workflow**
1. **Find** — Query Store, `sys.dm_exec_query_stats`, Extended Events, APM.
2. **Read plan** + `SET STATISTICS IO/TIME ON` — look for scans, **key lookups**, sorts/spills, estimated-vs-actual row gaps.
3. **Fix** — covering index, SARGable rewrite, reduce rows early, update stats.
4. **Measure** again (trust **logical reads** over wall-clock).
- Biggest wins: **missing index** or **non-SARGable predicate**.

**Anti-patterns → fix**
- `SELECT *` → list columns (enables covering index).
- Function/expression on indexed column → **non-SARGable scan** → rewrite as range/constant-side.
- Implicit conversion (`nvarchar` param vs `varchar` col) → scan → match types.
- `LIKE '%x'` (leading wildcard) → can't seek.
- **N+1** → eager-load / `Include` / project (../dotnet.md O).
- `OR` defeating indexes → sometimes `UNION`. Huge `IN` → TVP.
- Missing FK/join indexes; over-indexing (write tax); RBAR/cursors → **set-based**; filter in app → push to SQL.

**Parameterization & sniffing**
- Parameterized → **plan reuse** (no cache bloat) + **injection-safe**. Ad-hoc concatenation → one plan per literal.
- **Parameter sniffing** = plan built for **first** param's values, cached; skewed data → **"fast for some, slow for others."**
- Fixes: `OPTION (RECOMPILE)`, `OPTIMIZE FOR (…)` / `UNKNOWN`, **Query Store plan forcing**, refactor.

**Batching & bulk**
- Round-trips are the cost. **TVPs** (100s–1000s rows), `MERGE`/set-based upsert, **`SqlBulkCopy`** (large loads), EF `AddRange`, **EF7 `ExecuteUpdate`/`ExecuteDelete`**.
- Big txn = clean rollback but locks + log growth. **Chunk** large ops (1k–10k/commit), make resumable.

**Scaling**
- **Vertical** (scale up) = simple, has ceiling. **Horizontal** = replicas/partition/shard/cache.
- **Read replicas** = offload reads; async → **replication lag / eventual consistency** (read-your-writes risk). SQL Server Always On readable secondaries.
- **Partitioning** = split table **within one DB** (partition elimination, manageability). **Sharding** = split across **many DBs** (scale; hard shard-key choice, cross-shard joins/txns painful).
- **Caching** (Redis, cache-aside + TTL) in front = highest read leverage; invalidation is the hard part. **CQRS** = read/write model split.
- **Ladder:** tune query/index → fix sniffing → cache → scale up → replicas → partition → shard. **Measure first.**
