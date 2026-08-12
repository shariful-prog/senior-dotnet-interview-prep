# V. Advanced Queries (Window Functions, CTEs, Subqueries)
---
## V1 — Window Functions

### Q1. What is a window function, and how does it differ from GROUP BY?

A **Window Function** performs a calculation across a set of related rows (called a **window**) **without collapsing the rows**. Unlike `GROUP BY`, it returns a value for **each row** while still allowing access to the other rows.


That is the whole difference from `GROUP BY`:

| | Rows out |
|---|---|
| `GROUP BY` | one per group — the detail is gone |
| Window function | one per input row — detail kept, value attached |

```sql
-- GROUP BY: one row per department, detail is gone
SELECT DepartmentId, AVG(Salary) AS AvgSalary
FROM Employees
GROUP BY DepartmentId;

-- Window: every employee row kept, dept average attached to each
SELECT EmployeeId, DepartmentId, Salary,
 AVG(Salary) OVER (PARTITION BY DepartmentId) AS DeptAvgSalary
FROM Employees;
```

The second query compares each salary to the department average **on the same row**. With `GROUP BY` you'd need a self-join or subquery to do that.

**Detail plus aggregate together — that's the point of window functions.**

They never add or remove rows.

---

### Q2. Break down the `OVER()` clause: PARTITION BY, ORDER BY, and the frame.

`OVER()` defines the window. It has up to three parts:

```sql
func() OVER (
 PARTITION BY <cols> -- (1) split rows into groups; function restarts per group
 ORDER BY <cols> -- (2) order rows within each partition
 ROWS/RANGE BETWEEN ... AND ... -- (3) frame: which rows within the ordered partition count
)
```

1. **PARTITION BY** — splits rows into groups; the function restarts in each one. Omit it and the whole result is one group.
2. **ORDER BY** — orders rows *within* the partition. Needed by ranking functions and anything order-sensitive (running totals, `LAG`). This is local to the window — unrelated to the query's final `ORDER BY`.
3. **Frame** — narrows which of those ordered rows count. Covered in V2.

```sql
SELECT EmployeeId, DepartmentId, Salary,
 ROW_NUMBER() OVER (PARTITION BY DepartmentId ORDER BY Salary DESC) AS RankInDept
FROM Employees;
-- Each department gets its own 1..n ranking by salary.
```

All three parts are optional, but they build on each other — a frame only means something once there's an `ORDER BY`.

---

### Q3. Compare ROW_NUMBER, RANK, DENSE_RANK, and NTILE. When do the differences matter?

All four are ranking window functions requiring `ORDER BY`. The difference is entirely in how they handle **ties**:

| Function | Behavior on ties | Example values for salaries 100,90,90,80 |
|---|---|---|
| `ROW_NUMBER()` | Always unique 1..n; ties broken **arbitrarily** | 1, 2, 3, 4 |
| `RANK()` | Same rank for ties, then **skips** (gaps) | 1, 2, 2, **4** |
| `DENSE_RANK()` | Same rank for ties, **no gaps** | 1, 2, 2, **3** |
| `NTILE(n)` | Splits partition into `n` roughly equal buckets | with `NTILE(2)`: 1, 1, 2, 2 |

```sql
SELECT Name, Salary,
 ROW_NUMBER() OVER (ORDER BY Salary DESC) AS rn,
 RANK() OVER (ORDER BY Salary DESC) AS rnk,
 DENSE_RANK() OVER (ORDER BY Salary DESC) AS dense,
 NTILE(4) OVER (ORDER BY Salary DESC) AS quartile
FROM Employees;
```

**Which to pick:**

- **`ROW_NUMBER`** — you need exactly one row per key (dedup, latest-row-per-group, pagination). Ties are arbitrary, so add a tiebreaker column to `ORDER BY` for repeatable results.
- **`RANK` / `DENSE_RANK`** — leaderboards, where tied people should share a position. `DENSE_RANK` for "top 3 distinct salary levels", `RANK` for Olympic-style skipping.
- **`NTILE`** — splitting into quartiles or cohorts.

---

### Q4. (Killer use case) How do you get the "latest row per group" / "top N per group" with a window function?

The canonical pattern: `ROW_NUMBER()` partitioned by the group key, ordered by the tiebreak (e.g., date descending), then filter to `rn = 1` (or `rn <= N`).

```sql
-- Latest order per customer
WITH Ranked AS (
 SELECT o.*,
 ROW_NUMBER() OVER (PARTITION BY CustomerId ORDER BY OrderDate DESC, OrderId DESC) AS rn
 FROM Orders o
)
SELECT * FROM Ranked WHERE rn = 1;
```

For top 3, change the filter to `WHERE rn <= 3`. The `OrderId DESC` tiebreaker keeps results stable when two orders share a date.

**Why the CTE is required:** window functions are computed in the `SELECT` phase, which runs *after* `WHERE`. So `rn` does not exist yet when `WHERE` is evaluated.

```sql
-- ILLEGAL: rn doesn't exist during WHERE
SELECT *, ROW_NUMBER() OVER (...) AS rn
FROM Orders
WHERE rn = 1; -- Error: Invalid column name 'rn'
```

Compute it in an inner query, filter in the outer one — where it's now an ordinary column.

**The alternative is `CROSS APPLY (SELECT TOP 1 ...)`** (Q19). The window version makes one pass over the table; `APPLY` can be much faster when you have few groups and an index on `(GroupKey, OrderDate DESC)`, because it seeks per group instead of ranking everything.

---

## V2 — Window Aggregates: Running Totals, Frames, LAG/LEAD

### Q5. How do you compute a running total (cumulative sum) with a window function?

Use an aggregate (`SUM`) as a window function with an `ORDER BY` and an explicit **frame** of "everything from the start up to the current row":

```sql
SELECT OrderDate, Amount,
 SUM(Amount) OVER (ORDER BY OrderDate, OrderId
 ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS RunningTotal
FROM Orders;
```

`ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` means "the first row in the partition through the current one." Add `PARTITION BY CustomerId` and the total restarts per customer.

**Changing the frame gives you the other common patterns:**

| Frame | Result |
|---|---|
| `UNBOUNDED PRECEDING AND CURRENT ROW` | running total |
| `2 PRECEDING AND CURRENT ROW` | 3-row moving average |
| `1 PRECEDING AND 1 FOLLOWING` | centered 3-row window |
| no `ORDER BY` at all | total for the whole partition on every row |

**Always write the `ROWS` clause explicitly.** Omitting it works but silently changes behaviour — Q6 explains why.

---

### Q6. Explain ROWS vs RANGE and the default-frame gotcha.

Both define the frame; they count differently.

- **`ROWS`** counts **physical rows** — "the 2 rows before this one." Tied values are still separate rows.
- **`RANGE`** counts by **value** — every row sharing the current row's `ORDER BY` value is treated as one peer group and included together.

**The gotcha:** write `ORDER BY` with no frame and you get `RANGE`, not `ROWS`. If the ordering column has duplicates, all tied rows collapse into "current row," so a running total **jumps** at each duplicate instead of stepping row by row:

```sql
-- Two orders on the same date get the SAME running total (both include each other)
SELECT OrderDate, Amount,
 SUM(Amount) OVER (ORDER BY OrderDate) AS RunTotal_RANGE -- default RANGE — surprising
FROM Orders;

-- Fix: ROWS increments strictly row-by-row
SELECT OrderDate, Amount,
 SUM(Amount) OVER (ORDER BY OrderDate
 ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS RunTotal_ROWS
FROM Orders;
```

`RANGE` is also slower — SQL Server may spool the frame to a worktable.

**Rule: always write `ROWS` for running totals and moving windows**, unless you genuinely want peer-group behaviour.

---

### Q7. What do LAG and LEAD do?

They read a value from another row in the same partition — `LAG` looks **backward**, `LEAD` looks **forward**. No self-join needed.

`LAG(col, offset, default)`: `offset` defaults to 1, and `default` is what you get when there's no such row (otherwise `NULL`).

```sql
-- Month-over-month revenue delta
SELECT MonthStart, Revenue,
 LAG(Revenue, 1, 0) OVER (ORDER BY MonthStart) AS PrevMonth,
 Revenue - LAG(Revenue, 1, 0) OVER (ORDER BY MonthStart) AS MoMDelta
FROM MonthlyRevenue;
```

Common uses: period-over-period deltas, and detecting where a value changed (`WHERE value <> LAG(value) OVER (...)`).

Add `PARTITION BY` and the comparison stays inside each group — a customer's previous order, not the previous order overall.

---

## V3 — CTEs (WITH) & Readability

### Q8. What is a CTE, and why use one over a nested subquery?

A **Common Table Expression (CTE)** is a **temporary named result set** that exists only for the duration of a single SQL statement. It is defined using the `WITH` keyword and makes complex queries easier to read and maintain. It is mainly a readability tool.

```sql
WITH RecentOrders AS (
 SELECT CustomerId, OrderId, OrderDate
 FROM Orders
 WHERE OrderDate >= DATEADD(DAY, -30, GETDATE())
)
SELECT CustomerId, COUNT(*) AS OrdersLast30Days
FROM RecentOrders
GROUP BY CustomerId;
```

Four reasons to prefer one over a nested subquery:

1. **Readability** — you read top-to-bottom instead of inside-out, and the logic gets a name.
2. **Reference it twice** — a derived table would have to be written out again.
3. **Chaining** — each CTE can reference the previous ones, so a multi-step transformation reads as a pipeline.
4. **Recursion** — the only way to write one (V4).

```sql
WITH a AS ( SELECT ... ),
 b AS ( SELECT ... FROM a WHERE ... ), -- b references a
 c AS ( SELECT ... FROM b JOIN a ... )
SELECT * FROM c;
```

**Scope:** only the statement right after the `WITH` can see it. Nothing is stored, and it's gone afterward.

---

### Q9. Is a CTE materialized? Does referencing it twice run it twice?

**No, it is not materialized — and yes, it can run twice.**

In SQL Server a CTE is **inlined** into the query plan. Treat it as a named subquery, not a temp table: the optimizer expands the definition wherever the name appears and optimizes everything as one query.

So referencing a CTE three times may execute it three times. Nothing is cached. For an expensive definition, that is a real problem:

```sql
WITH Expensive AS (
 SELECT CustomerId, SUM(Amount) AS Total -- costly aggregation
 FROM HugeOrders GROUP BY CustomerId
)
SELECT a.CustomerId
FROM Expensive a JOIN Expensive b ON ...; -- 'Expensive' may run twice
```

**The fix:** if the result must be computed once and reused, use a `#temp` table. That forces materialization and lets you index it.

**You can also `DELETE` through a CTE**, and it changes the real table — the CTE is a window onto it, not a copy. This is how the classic dedup works (Q18):

```sql
WITH ranked AS (
    SELECT Id, Email, ROW_NUMBER() OVER (PARTITION BY Email ORDER BY Id) AS rn
    FROM dbo.Contacts
)
DELETE FROM ranked WHERE rn > 1;   -- deletes from dbo.Contacts itself
```

**PostgreSQL note.** Before version 12, Postgres always materialized CTEs. Since 12 it inlines them like SQL Server, and you can force either behaviour with `AS MATERIALIZED` / `AS NOT MATERIALIZED`. SQL Server has no such keyword.

---

### Q10. CTE vs subquery vs temp table vs view — when do you pick which?

| Construct | Lifetime | Materialized? | Indexable? | Best for |
|---|---|---|---|---|
| **Subquery / derived table** | one statement | no (inlined) | no | simple, single-use inline logic |
| **CTE** | one statement | no (inlined in SQL Server) | no | readability, self-reference/recursion, chaining steps |
| **Temp table `#t`** | session/scope | yes | **yes** (+ stats) | reuse across statements, breaking up huge plans, indexing intermediate results |
| **Table variable `@t`** | batch/scope | yes (in tempdb) | limited (PK/unique only, poor stats pre-2019) | small sets, in functions, when you want no recompiles |
| **View** | persistent object | no (unless indexed view) | indexed views only | reusable named query across many statements/users |

**In practice it comes down to three questions:**

- Just making one statement readable? → **CTE**
- Need it computed once, indexed, or reused across statements? → **`#temp` table**
- Need it shared across the codebase as a named object? → **View**

The temp table has one non-obvious benefit: materializing a step **breaks the optimizer's cardinality estimation chain**. When one enormous query keeps producing a bad plan, splitting it at a `#temp` table often fixes it outright.

---

## V4 — Recursive CTEs

### Q11. What is a recursive CTE and what are its parts?

A recursive CTE references **itself** to iterate. It's how you query hierarchies — org charts, category trees, bills of materials — and how you generate sequences.

It always has two halves joined by `UNION ALL`:

```sql
WITH cte AS (
 -- ANCHOR member: the starting rows, runs once
 SELECT ...
 UNION ALL
 -- RECURSIVE member: references cte, runs repeatedly on the previous result
 SELECT ... FROM SomeTable JOIN cte ON ...
)
SELECT * FROM cte;
```

**How it runs:** the anchor produces the starting rows. The recursive member then runs against **only the rows the previous iteration produced**, appending each batch. It repeats until an iteration returns **no rows** — that's what stops it.

The standard example, an org chart from `Employees(EmployeeId, Name, ManagerId)`:

```sql
WITH OrgChart AS (
 -- Anchor: the top manager (or a chosen root)
 SELECT EmployeeId, Name, ManagerId, 0 AS Level
 FROM Employees
 WHERE EmployeeId = @RootId

 UNION ALL

 -- Recursive: employees whose manager is already in OrgChart
 SELECT e.EmployeeId, e.Name, e.ManagerId, oc.Level + 1
 FROM Employees e
 INNER JOIN OrgChart oc ON e.ManagerId = oc.EmployeeId
)
SELECT EmployeeId, Name, ManagerId, Level
FROM OrgChart
ORDER BY Level, Name;
```

Each iteration walks one level deeper, and `Level` records the depth.

The same shape generates a sequence, with no table involved:

```sql
WITH Dates AS (
 SELECT CAST('2026-01-01' AS DATE) AS d
 UNION ALL
 SELECT DATEADD(DAY, 1, d) FROM Dates
 WHERE d < '2026-01-31'
)
SELECT d FROM Dates
OPTION (MAXRECURSION 366);
```

---

### Q12. What is MAXRECURSION?

It caps how many levels deep the recursion may go. **The default is 100**, and hitting it raises an error rather than returning partial results.

```sql
SELECT * FROM OrgChart OPTION (MAXRECURSION 1000);
SELECT * FROM Dates OPTION (MAXRECURSION 0); -- 0 = no limit (careful)
```

That default is a safety net, and the thing it protects you from is a **cycle in the data** — if A manages B and B manages A, the recursion never ends on its own.

So: keep a sensible limit in production instead of `0`, and let it fail fast. For a genuinely cyclic graph, carry a path string and exclude nodes you've already visited.

**PostgreSQL note.** The `RECURSIVE` keyword is required there (`WITH RECURSIVE cte AS ...`). There is no `MAXRECURSION`.

---

## V5 — Subqueries: Correlated vs Non-Correlated

### Q13. Explain non-correlated vs correlated subqueries.

- **Non-correlated** — does not reference the outer query. It runs **once**, on its own, and the outer query uses the result.

```sql
-- Runs once; returns the overall average, then compared to each row
SELECT Name, Salary
FROM Employees
WHERE Salary > (SELECT AVG(Salary) FROM Employees);
```

- **Correlated** — references a column from the outer query, so conceptually it runs **once per outer row**. (The optimizer usually rewrites it as a join, but that's the mental model.)

```sql
-- References e.DepartmentId from the outer query → per-row
SELECT Name, Salary
FROM Employees e
WHERE Salary > (SELECT AVG(Salary) FROM Employees e2 WHERE e2.DepartmentId = e.DepartmentId);
-- "employees earning above their own department's average"
```

**The tell for correlation:** the inner query mentions an outer alias (`e.DepartmentId`).

**One gotcha worth knowing.** A subquery used as a single value must return **one row**. If it returns more, you get a runtime error:

> *Subquery returned more than 1 value.*

If it returns **zero** rows you get `NULL` — no error, which is worse, because comparisons against it silently become unknown. Fix it with `TOP 1 ... ORDER BY`, an aggregate like `MAX`, or `EXISTS` if you only care whether a row exists.

---

### Q14. When do you prefer EXISTS over IN?

`EXISTS` returns true as soon as the subquery produces **one row**, then stops. It ignores what the subquery selects, so `SELECT 1` is idiomatic.

```sql
-- Customers who have placed at least one order
SELECT c.Name
FROM Customers c
WHERE EXISTS (SELECT 1 FROM Orders o WHERE o.CustomerId = c.CustomerId);
```

For the positive case, `EXISTS` and `IN` perform about the same — the optimizer treats both as semi-joins. Pick whichever reads better.

**The negative case is where it matters.** `NOT IN` has a classic bug: if the subquery returns even one `NULL`, the whole thing returns **no rows at all**, because every comparison becomes `UNKNOWN`.

```sql
-- Returns nothing if any CustomerId in Orders is NULL
SELECT Name FROM Customers
WHERE CustomerId NOT IN (SELECT CustomerId FROM Orders);

-- Correct: NULL-safe
SELECT Name FROM Customers c
WHERE NOT EXISTS (SELECT 1 FROM Orders o WHERE o.CustomerId = c.CustomerId);
```

**Rule: use `NOT EXISTS` for anti-joins.** It is NULL-safe and usually gets a better plan.

---

## V6 — APPLY (CROSS APPLY / OUTER APPLY)

### Q15. What does APPLY do, and when can a JOIN not replace it?

`APPLY` runs the right side **once per left row**, and the right side **can reference that row's columns**. A normal `JOIN` can't do this — its two sides must be independently computable.

- **`CROSS APPLY`** behaves like an **INNER JOIN**: left rows are dropped when the right side returns nothing.
- **`OUTER APPLY`** behaves like a **LEFT JOIN**: left rows are kept, with `NULL`s.

```sql
-- Top 2 most recent orders per customer (right side depends on c.CustomerId)
SELECT c.CustomerId, c.Name, o.OrderId, o.OrderDate
FROM Customers c
CROSS APPLY (
 SELECT TOP 2 o.OrderId, o.OrderDate
 FROM Orders o
 WHERE o.CustomerId = c.CustomerId -- correlation to outer row
 ORDER BY o.OrderDate DESC
) o;
-- OUTER APPLY instead → also lists customers with zero orders (NULL order columns)
```

**Three cases where you have no alternative:**

1. **Top-N per group** — the correlated `TOP` above. With an index on `(CustomerId, OrderDate DESC)` this is often the fastest approach.
2. **Calling a table-valued function per row**, since it needs a column as its argument:

```sql
SELECT o.OrderId, parts.*
FROM Orders o
CROSS APPLY dbo.SplitOrderLines(o.OrderId) AS parts;
```

3. **Unpivoting a few columns into rows:**

```sql
SELECT o.OrderId, v.Phase, v.Ts
FROM Orders o
CROSS APPLY (VALUES ('Created', o.CreatedAt), ('Shipped', o.ShippedAt)) AS v(Phase, Ts)
WHERE v.Ts IS NOT NULL;
```

**PostgreSQL note.** The equivalent is `LATERAL` — `CROSS JOIN LATERAL` and `LEFT JOIN LATERAL ... ON TRUE`.

---

## V7 — Pagination & Common Patterns

### Q16. How does OFFSET/FETCH pagination work, and what's its problem?

`OFFSET ... FETCH` is the standard paging mechanism (SQL Server 2012+). It **requires an `ORDER BY`**:

```sql
SELECT OrderId, OrderDate, Amount
FROM Orders
ORDER BY OrderDate DESC, OrderId DESC -- must be deterministic
OFFSET @PageSize * (@PageNumber - 1) ROWS
FETCH NEXT @PageSize ROWS ONLY;
```

**The problem is deep pages.** `OFFSET n` doesn't skip ahead — it reads the first `n` rows and throws them away. Page 1 is cheap; page 10,000 reads and discards ~10,000 × PageSize rows first, so late pages get progressively slower.

There's a second issue: if rows are inserted between two requests, everything shifts down, so the user sees a **duplicate row** on the next page or **misses one** entirely.

#### Solution: Keyset (Cursor) Pagination

Instead of using page numbers, remember the **last value** from the previous page (usually the primary key or a unique indexed column).

Example:

```sql
-- First page
SELECT TOP (10) *
FROM Orders
ORDER BY OrderId;
```

Suppose the last `OrderId` returned is **100**.

Next page:

```sql
SELECT TOP (10) *
FROM Orders
WHERE OrderId > 100
ORDER BY OrderId;
```

SQL Server performs an **Index Seek**, making performance consistent even for very large datasets.

---

### Q17. What is keyset pagination, and why is it better?

Instead of counting off an offset, **keyset (or "seek") pagination remembers the last row of the previous page** and asks for rows after it:

```sql
-- Page 1
SELECT TOP (@PageSize) OrderId, OrderDate, Amount
FROM Orders
ORDER BY OrderDate DESC, OrderId DESC;

-- Next page: pass the last row's (OrderDate, OrderId) as @lastDate, @lastId
SELECT TOP (@PageSize) OrderId, OrderDate, Amount
FROM Orders
WHERE (OrderDate < @lastDate)
 OR (OrderDate = @lastDate AND OrderId < @lastId) -- tie-break on the key
ORDER BY OrderDate DESC, OrderId DESC;
```

This fixes both `OFFSET` problems:

- **Every page costs the same.** With an index on `(OrderDate DESC, OrderId DESC)` each page is one seek plus a short range read. Page 10,000 is as fast as page 1.
- **No shifting rows**, because you page relative to a real key rather than a position.

**The trade-off:** you only get next/previous — you cannot jump to "page 500," since you don't know that page's starting key. That makes it ideal for infinite scroll and APIs, and unsuitable for numbered page links.

---

### Q18. How do you delete duplicate rows, keeping one per key?

`ROW_NUMBER()` partitioned by whatever defines a duplicate, then delete everything with `rn > 1`:

```sql
WITH d AS (
 SELECT ROW_NUMBER() OVER (
 PARTITION BY Email -- what makes a "duplicate"
 ORDER BY CreatedAt ASC, Id ASC -- which one to KEEP (rn=1)
 ) AS rn
 FROM Users
)
DELETE FROM d WHERE rn > 1;
```

The window's `ORDER BY` decides which copy survives — here the earliest created. Swap the `DELETE` for `SELECT *` to check what you're about to remove first.

---

### Q19. Top-N-per-group: window function or APPLY?

Both solve it; they differ in how they read the table.

```sql
-- (A) Window: one pass, ranks every row, then filters
WITH r AS (
 SELECT o.*, ROW_NUMBER() OVER (PARTITION BY CustomerId ORDER BY OrderDate DESC) AS rn
 FROM Orders o
)
SELECT * FROM r WHERE rn <= 3;

-- (B) APPLY: seeks per group, reads only what it needs
SELECT c.CustomerId, o.*
FROM Customers c
CROSS APPLY (
 SELECT TOP 3 * FROM Orders o
 WHERE o.CustomerId = c.CustomerId
 ORDER BY o.OrderDate DESC
) o;
```

**Choose the window** when you're reading most of the table anyway, or there's no separate table of groups to drive from. It's one pass and simpler to read.

**Choose `APPLY`** when there are **few groups** and an index on `(CustomerId, OrderDate DESC)`. It seeks straight to each customer's newest 3 rows rather than ranking millions of rows to discard nearly all of them — often dramatically faster.

---

### Q20. How do `LAG()` and `LEAD()` window functions work?

**`LAG()`** accesses data from a previous row in the result set, and **`LEAD()`** accesses data from a subsequent row, without requiring a self-join.

```sql
SELECT 
    SalesDate,
    Amount,
    LAG(Amount, 1, 0) OVER (ORDER BY SalesDate) AS PreviousDayAmount,
    Amount - LAG(Amount, 1, 0) OVER (ORDER BY SalesDate) AS DayOverDayChange
FROM DailySales;
```

- **`LAG(col, offset, default_value)`**: `offset` defaults to 1. `default_value` (e.g. 0) is returned if there is no previous row (instead of `NULL`).
- **Use Cases**: Day-over-day sales trends, session duration calculation between user click events, or finding gaps between sequential timestamps.

---

### Q21. How do Window Frames (`ROWS BETWEEN ...`) work for running totals and moving averages?

A **Window Frame** explicitly defines the subset of rows within a partition evaluated by an aggregate function (`SUM`, `AVG`, `COUNT`).

```sql
-- 1. Running Total (Cumulative Sum)
SELECT 
    OrderDate,
    Amount,
    SUM(Amount) OVER (
        ORDER BY OrderDate
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS CumulativeTotal
FROM Orders;

-- 2. 3-Day Moving Average
SELECT 
    OrderDate,
    Amount,
    AVG(Amount) OVER (
        ORDER BY OrderDate
        ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
    ) AS MovingAvg3Days
FROM DailyMetrics;
```

> **Performance Tip**: Always specify `ROWS BETWEEN ...` explicitly when using `ORDER BY` with aggregates. By default, SQL Server uses `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`, which forces spooling to `tempdb` and performs significantly worse than `ROWS`.

---

### Q22. How does the `MERGE` (UPSERT) statement work, and what is its concurrency gotcha?

The **`MERGE`** statement performs `INSERT`, `UPDATE`, or `DELETE` operations in a single atomic SQL statement based on matching conditions between a target table and a source dataset.

```sql
MERGE TargetTable WITH (HOLDLOCK) AS target
USING SourceTable AS source
ON (target.Id = source.Id)
WHEN MATCHED THEN
    UPDATE SET target.Name = source.Name, target.Price = source.Price
WHEN NOT MATCHED BY TARGET THEN
    INSERT (Id, Name, Price) VALUES (source.Id, source.Name, source.Price);
```

#### The Concurrency Gotcha:
`MERGE` is **NOT inherently race-condition safe** under default isolation levels (`READ COMMITTED`). Without explicit locking hints, two concurrent `MERGE` statements inserting the same key will check target existence simultaneously, fail to match, and both attempt an `INSERT`, resulting in a **Primary Key / Unique Constraint Violation** or **Deadlock**.

**Fix**: Always append the `WITH (HOLDLOCK)` hint to the target table to take range locks and enforce serialization.

---

### Q23. What makes a query Non-SARGable, and how do functions on indexed columns ruin performance?

A query is **SARGable** (Search Argument Able) if the engine can use an Index Seek to locate rows. Wrapping an indexed column inside a function or mathematical operation turns it into a **Non-SARGable** query, forcing a full Table Scan / Index Scan.

```sql
-- ❌ Non-SARGable: Function YEAR() operates on indexed column CreatedAt (Full Index Scan)
SELECT * FROM Orders WHERE YEAR(CreatedAt) = 2024;

-- ✅ SARGable: Column is isolated on one side of operator (Index Seek)
SELECT * FROM Orders 
WHERE CreatedAt >= '2024-01-01' AND CreatedAt < '2025-01-01';

-- ❌ Non-SARGable: Wildcard prefix
SELECT * FROM Users WHERE Email LIKE '%@gmail.com';

-- ✅ SARGable: Suffix wildcard allows Index Seek
SELECT * FROM Users WHERE Email LIKE 'john%';
```