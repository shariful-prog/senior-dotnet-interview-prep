# T. SQL Fundamentals & Joins
---
## T1 — SELECT / WHERE / GROUP BY / HAVING & logical query processing

### Q1. What is the *logical* query processing order, and why does it matter?

**Answer.** SQL is declarative — you write the clauses in one order but the engine *logically* evaluates them in another. The logical order (independent of the physical plan the optimizer actually picks) is:

```
1. FROM (and JOIN / ON) -- produce the working row set
2. WHERE -- filter individual rows
3. GROUP BY -- collapse rows into groups
4. HAVING -- filter groups
5. SELECT -- evaluate expressions, assign aliases
6. DISTINCT
7. ORDER BY -- sort the final result
8. TOP / OFFSET-FETCH -- limit
```

Why it matters in practice:

- **You cannot reference a `SELECT` alias in `WHERE`, `GROUP BY`, or `HAVING`** — those clauses are processed *before* `SELECT` runs, so the alias does not exist yet. You *can* use it in `ORDER BY` (processed after `SELECT`).
- **`WHERE` filters rows before grouping; `HAVING` filters groups after.** Put row-level predicates in `WHERE` (cheaper — fewer rows to group) and aggregate predicates in `HAVING`.

```sql
-- ILLEGAL: net_price alias not visible in WHERE
SELECT price * (1 - discount) AS net_price
FROM Orders
WHERE net_price > 100; -- error: invalid column name 'net_price'

-- Correct: repeat the expression, or wrap in a CTE / subquery
SELECT price * (1 - discount) AS net_price
FROM Orders
WHERE price * (1 - discount) > 100
ORDER BY net_price; -- alias is fine in ORDER BY
```

**Gotcha.** This logical order is a *model*, not the execution plan. The optimizer is free to reorder physical operations (e.g. apply a `WHERE` filter via an index seek before a join) as long as results are identical.

---

### Q2. When do you use `WHERE` vs `HAVING`?

**Answer.** `WHERE` filters **individual rows** before any grouping happens. `HAVING` filters **groups** after `GROUP BY` has aggregated them, so it can reference aggregate functions. Prefer `WHERE` whenever the predicate does not involve an aggregate — it reduces the number of rows the group operator has to process, and it can use indexes (be SARGable — see [sql-indexing-plans.md](sql-indexing-plans.md)).

```sql
-- Total spend per customer in 2026, only for orders over $10,
-- and only customers whose 2026 total exceeds $5,000.
SELECT CustomerId, SUM(Amount) AS TotalSpend
FROM Orders
WHERE OrderDate >= '2026-01-01' -- row filter: cheap, index-friendly
 AND OrderDate < '2027-01-01'
 AND Amount > 10 -- row filter
GROUP BY CustomerId
HAVING SUM(Amount) > 5000; -- group filter: needs the aggregate
```

**Gotcha.** Putting a non-aggregate predicate in `HAVING` (`HAVING CustomerId = 5`) is legal but wasteful — you grouped rows you were going to throw away. Keep it in `WHERE`.

---

### Q3. What are the `GROUP BY` rules — which columns must appear where?

**Answer.** In standard SQL and SQL Server, **every column in the `SELECT` list must either be inside an aggregate function or appear in the `GROUP BY` clause.** The reason: after grouping, each group is a single output row, so a non-aggregated, non-grouped column would have multiple candidate values with no rule for which to return.

```sql
-- ERROR: Region is neither aggregated nor grouped
SELECT Region, City, COUNT(*)
FROM Customers
GROUP BY City;
-- Msg 8120: Column 'Region' is invalid in the select list because it is not
-- contained in either an aggregate function or the GROUP BY clause.

-- Fix: group by both, or aggregate Region
SELECT Region, City, COUNT(*) AS Cnt
FROM Customers
GROUP BY Region, City;
```

**PostgreSQL / MySQL note.** PostgreSQL relaxes this in one case: if you `GROUP BY` a table's primary key, you may `SELECT` its other columns un-aggregated (they are functionally dependent on the key). MySQL historically allowed *any* non-grouped column with `ONLY_FULL_GROUP_BY` disabled — returning arbitrary values — a notorious foot-gun that is now off by default. SQL Server never allows this; be explicit.

---

## T2 — JOINs: INNER / LEFT / RIGHT / FULL / CROSS, self, anti (CORE)

### Q4. Walk through every join type and its semantics.

**Answer.** A join combines rows from two tables based on a predicate (`ON`). The type controls what happens to rows that have *no* match on the other side.

| Join | Returns | Venn intuition |
|------|---------|----------------|
| `INNER JOIN` | Only rows with a match on **both** sides | Intersection |
| `LEFT [OUTER] JOIN` | All left rows + matched right rows; unmatched right side is `NULL` | All of left |
| `RIGHT [OUTER] JOIN` | All right rows + matched left rows; unmatched left side is `NULL` | All of right |
| `FULL [OUTER] JOIN` | All rows from both; non-matches filled with `NULL` on the missing side | Union |
| `CROSS JOIN` | Every left row paired with every right row (Cartesian product), no `ON` | Grid |

```sql
-- INNER: customers who have placed at least one order
SELECT c.CustomerId, o.OrderId
FROM Customers c
INNER JOIN Orders o ON o.CustomerId = c.CustomerId;

-- LEFT OUTER: ALL customers, with their orders if any (NULLs otherwise)
SELECT c.CustomerId, o.OrderId
FROM Customers c
LEFT JOIN Orders o ON o.CustomerId = c.CustomerId;

-- FULL OUTER: all customers AND all orders, matched where possible
SELECT c.CustomerId, o.OrderId
FROM Customers c
FULL JOIN Orders o ON o.CustomerId = c.CustomerId;

-- CROSS: every size paired with every color (e.g. build a variant matrix)
SELECT s.Size, col.Color
FROM Sizes s
CROSS JOIN Colors col;
```

**Gotcha.** `RIGHT JOIN` is rarely used in practice — any `RIGHT JOIN` can be rewritten as a `LEFT JOIN` by swapping table order, which reads more naturally (you list the "keep everything" table first). Reviewers expect `LEFT`.

---

### Q5. The classic bug: why does filtering the outer table in `WHERE` turn a `LEFT JOIN` into an `INNER JOIN`?

**Answer.** This is the single most common join mistake. A `LEFT JOIN` keeps unmatched left rows by filling the right-side columns with `NULL`. If you then apply a `WHERE` predicate on a **right-table column**, those `NULL`s fail the predicate (`NULL = 'X'` is `UNKNOWN`, not `TRUE`), so the unmatched rows are dropped — silently converting your outer join into an inner join.

The fix: predicates that restrict the **right (optional) table** belong in the **`ON` clause**; predicates on the **left table** belong in **`WHERE`**.

```sql
-- BUG: intended "all customers, plus their 2026 orders",
-- but customers with NO 2026 orders vanish.
SELECT c.CustomerId, o.OrderId
FROM Customers c
LEFT JOIN Orders o ON o.CustomerId = c.CustomerId
WHERE o.OrderDate >= '2026-01-01'; -- NULL OrderDate for unmatched rows -> excluded

-- FIX: move the right-table filter into the ON clause
SELECT c.CustomerId, o.OrderId
FROM Customers c
LEFT JOIN Orders o
 ON o.CustomerId = c.CustomerId
 AND o.OrderDate >= '2026-01-01'; -- now customers with no 2026 order stay, with NULL order
```

Rule of thumb: **`ON` decides what counts as a match; `WHERE` filters the result set.** For the *preserved* (left) table, `ON` vs `WHERE` are equivalent; for the *optional* (right) table they are very different.

---

### Q6. How do you write a self-join (e.g. employee → manager)?

**Answer.** A self-join joins a table to itself using two aliases, treating the same physical table as two logical row sets. The canonical case is a hierarchy where each row references another row in the same table.

```sql
-- Each employee with their manager's name.
-- LEFT JOIN so the CEO (ManagerId IS NULL) is not dropped.
SELECT e.EmployeeId,
 e.Name AS Employee,
 m.Name AS Manager
FROM Employees e
LEFT JOIN Employees m ON m.EmployeeId = e.ManagerId;
```

**Gotcha.** Use `INNER JOIN` here and the top of the hierarchy (no manager) disappears — the same outer-join concern as Q5. For arbitrary-depth hierarchies (full org chart, bill-of-materials), a single self-join only reaches one level; use a **recursive CTE** instead (covered in the CTE/window section).

---

### Q7. What is an anti-join, and what are the two idiomatic ways to write one?

**Answer.** An **anti-join** returns rows from A that have **no** matching row in B — the "in A but not in B" set. There is no `ANTI JOIN` keyword in T-SQL; you express it two ways:

```sql
-- 1) LEFT JOIN ... WHERE right IS NULL (outer-join-and-filter pattern)
SELECT c.CustomerId
FROM Customers c
LEFT JOIN Orders o ON o.CustomerId = c.CustomerId
WHERE o.CustomerId IS NULL; -- kept only where no order matched

-- 2) NOT EXISTS (correlated subquery) -- usually the cleaner, NULL-safe choice
SELECT c.CustomerId
FROM Customers c
WHERE NOT EXISTS (SELECT 1 FROM Orders o WHERE o.CustomerId = c.CustomerId);
```

Both are typically compiled to the same physical *anti-semi-join* operator, so performance is comparable. Prefer `NOT EXISTS` for readability and because it is **NULL-safe** — unlike `NOT IN` (see T6). The corresponding *semi-join* ("rows in A that DO have a match, without duplicating A") is `EXISTS` / `INNER JOIN` + `DISTINCT`.

---

### Q8. How does a one-to-many join affect row cardinality — and how does that surprise aggregates?

**Answer.** A join does not "look up" one value per left row; it **multiplies** rows. If a left row matches *N* right rows, you get *N* output rows. Joining a parent to a one-to-many child multiplies the parent by its child count — and if you then `SUM` a parent-side column, you sum it once *per child*, inflating the total.

```sql
-- BUG: OrderTotal is on Orders (1 row), but the join to OrderLines (many rows)
-- duplicates OrderTotal per line, so SUM double/triple-counts it.
SELECT o.CustomerId, SUM(o.OrderTotal) AS Spend -- INFLATED
FROM Orders o
JOIN OrderLines ol ON ol.OrderId = o.OrderId
GROUP BY o.CustomerId;

-- FIX A: don't join the many-side if you don't need it
SELECT CustomerId, SUM(OrderTotal) AS Spend
FROM Orders
GROUP BY CustomerId;

-- FIX B: if you need both grains, pre-aggregate the child, then join
SELECT o.CustomerId,
 SUM(o.OrderTotal) AS Spend,
 SUM(l.LineCount) AS Lines
FROM Orders o
JOIN (SELECT OrderId, COUNT(*) AS LineCount
 FROM OrderLines GROUP BY OrderId) l ON l.OrderId = o.OrderId
GROUP BY o.CustomerId;
```

**Gotcha.** Fan-out from a many-to-many join is worse — a 3× and a 4× join produce 12× the base rows. Always know the *grain* (one row per what?) of every table in the join and of the final result. When counts look 2–3× too high, suspect an unintended one-to-many.

---

### Q9. How do you join on multiple columns, and when must you?

**Answer.** When the relationship key is composite (or you are matching on a natural key across systems), extend the `ON` predicate with `AND`. Every column of the compound key must be in the `ON`, or you get an unintended partial-key join that fans out.

```sql
-- Composite key: match on tenant AND product
SELECT p.Name, i.QtyOnHand
FROM Products p
JOIN Inventory i
 ON i.TenantId = p.TenantId
 AND i.ProductId = p.ProductId;
```

**Gotcha.** Forgetting one part of a composite key (`ON i.ProductId = p.ProductId` only, dropping `TenantId`) matches the same product across *all* tenants — a data-leak-grade bug and a cardinality explosion. Also mind collation/type mismatches on join columns: an implicit conversion (e.g. `varchar` join to `nvarchar`) can defeat index seeks and force scans (see [sql-indexing-plans.md](sql-indexing-plans.md)).

---

### Q10. What is the difference between a `CROSS JOIN` and a `CROSS APPLY`?

**Answer.** In a `CROSS JOIN` the two sides know nothing about each other. SQL Server takes every row on the left, pairs it with every row on the right, and returns all combinations. 10 rows on the left and 5 on the right gives 50 rows.

`CROSS APPLY` (a T-SQL feature) is different: the right side is a query that runs **once per left row**, and it is allowed to look at the values of that row. So you write `WHERE o.CustomerId = c.CustomerId` inside it, which a `CROSS JOIN` cannot do.

That makes `CROSS APPLY` the natural tool for "give me the top N per group" and for calling a table-valued function with a column from each row.

```sql
-- The 3 most recent orders per customer
SELECT c.CustomerId, o.OrderId, o.OrderDate
FROM Customers c
CROSS APPLY (SELECT TOP (3) OrderId, OrderDate
 FROM Orders o
 WHERE o.CustomerId = c.CustomerId -- allowed: sees the current customer
 ORDER BY o.OrderDate DESC) o;
```

Here the inner query runs for each customer and returns at most 3 rows for that customer.

One rule to remember: if the inner query returns no rows, `CROSS APPLY` drops the left row entirely — it behaves like an inner join. Use `OUTER APPLY` when you want to keep the customer anyway with `NULL` order columns; that is the `LEFT JOIN` version.

**PostgreSQL note.** The same feature is called `LATERAL` — `CROSS JOIN LATERAL` for `CROSS APPLY`, and `LEFT JOIN LATERAL ... ON true` for `OUTER APPLY`.

---

## T3 — NULL handling & three-valued logic

### Q11. What does `NULL` mean, and what is three-valued logic?

**Answer.** `NULL` is not a value — it means **"unknown / missing / not applicable."** Because a comparison with an unknown is itself unknown, SQL uses **three-valued logic (3VL)**: every predicate evaluates to `TRUE`, `FALSE`, or `UNKNOWN`. Any arithmetic or comparison involving `NULL` yields `UNKNOWN` (or `NULL`).

```sql
NULL = NULL -> UNKNOWN -- NOT true!
NULL <> 5 -> UNKNOWN
NULL + 10 -> NULL
```

Consequences:

- To test for null you **must** use `IS NULL` / `IS NOT NULL`; `= NULL` never matches. (SQL Server's legacy `SET ANSI_NULLS OFF` changed this — it is deprecated; never rely on it.)
- **`WHERE` keeps only rows where the predicate is `TRUE`.** Rows where it is `UNKNOWN` are excluded — so a filter like `WHERE Status <> 'Closed'` silently drops rows where `Status IS NULL`.

```sql
-- Drops rows with NULL Status, which is often NOT what you meant
WHERE Status <> 'Closed'
-- Include them explicitly:
WHERE (Status <> 'Closed' OR Status IS NULL)
```

**Gotcha.** `CHECK` constraints behave oppositely to `WHERE`: a `CHECK` *passes* unless it evaluates to `FALSE`, so `UNKNOWN` (from a NULL) satisfies the constraint.

---

### Q12. How do NULLs behave inside aggregate functions?

**Answer.** All aggregates except `COUNT(*)` **ignore NULLs**:

- `COUNT(*)` counts **rows**, NULLs included.
- `COUNT(col)` counts **non-NULL** values of `col`.
- `SUM`, `AVG`, `MIN`, `MAX` skip NULLs entirely.

The subtle trap is `AVG`: it divides the sum of non-NULLs by the **count of non-NULLs**, not by the total row count. So NULLs change the denominator.

```sql
-- Column Bonus: {100, 200, NULL}
SELECT COUNT(*) AS Rows, -- 3
 COUNT(Bonus) AS NonNull, -- 2
 SUM(Bonus) AS Total, -- 300
 AVG(Bonus) AS Avg; -- 150 (300/2, NOT 300/3 = 100)

-- If you want NULL treated as 0 in the average, coalesce first:
SELECT AVG(COALESCE(Bonus, 0)) AS AvgTreatingNullAsZero; -- 100 (300/3)
```

**Gotcha.** `SUM` over a set that is entirely NULL (or empty) returns `NULL`, not `0` — wrap in `COALESCE(SUM(x), 0)` if you need a zero.

---

### Q13. Compare `ISNULL`, `COALESCE`, and `NULLIF`.

**Answer.**

- **`ISNULL(a, b)`** — T-SQL-specific, two arguments; returns `b` if `a` is NULL. The result type and length follow the **first** argument, which can silently truncate.
- **`COALESCE(a, b, c, …)`** — ANSI standard, N arguments; returns the first non-NULL. Result type is the **highest data-type precedence** among the arguments. Portable and more flexible — prefer it.
- **`NULLIF(a, b)`** — returns `NULL` if `a = b`, else `a`. Handy to turn a sentinel into NULL, or to guard against divide-by-zero.

```sql
SELECT ISNULL(MiddleName, 'N/A'); -- SQL Server only
SELECT COALESCE(MobilePhone, HomePhone, 'none'); -- first non-null of several

-- NULLIF to avoid divide-by-zero: 0 -> NULL -> the division yields NULL, not an error
SELECT Revenue / NULLIF(OrderCount, 0) AS AvgOrderValue;
```

**Gotchas.**
- `ISNULL` truncation: `ISNULL(CAST('ab' AS varchar(2)), 'default')` can truncate `'default'` to 2 chars because the type comes from the first arg. `COALESCE` does not have this issue.
- `COALESCE` is expanded by the engine into a `CASE` expression, so a non-deterministic or subquery argument may be **evaluated more than once** — beware side effects and cost.

---

### Q14. Why does `NOT IN` with a list that contains a `NULL` return no rows?

**Answer.** This is the flagship NULL trap and it ties directly to T6. `x NOT IN (a, b, NULL)` expands to `x <> a AND x <> b AND x <> NULL`. That last term is always `UNKNOWN`, and `TRUE AND UNKNOWN = UNKNOWN`, so the whole predicate can never be `TRUE`. Result: **the query returns zero rows** whenever the subquery/list contains even one NULL.

```sql
-- If ANY BannedId is NULL, this returns NOTHING (silently wrong)
SELECT * FROM Users
WHERE UserId NOT IN (SELECT BannedId FROM BanList);

-- NULL-safe rewrite: NOT EXISTS
SELECT * FROM Users u
WHERE NOT EXISTS (SELECT 1 FROM BanList b WHERE b.BannedId = u.UserId);
```

`IN` (positive) is unaffected in the same dangerous way — a NULL in the list just never produces a match — but `NOT IN` breaks catastrophically. **Always prefer `NOT EXISTS`** for exclusion. See Q22.

---

## T4 — Aggregates & DISTINCT

### Q15. Distinguish `COUNT(*)`, `COUNT(col)`, and `COUNT(DISTINCT col)`.

**Answer.**

- `COUNT(*)` — number of rows in the group, NULLs included. Fastest general row count.
- `COUNT(col)` — number of rows where `col IS NOT NULL`.
- `COUNT(DISTINCT col)` — number of distinct non-NULL values of `col`.

```sql
-- Orders table with some NULL PromoCode, repeated CustomerId
SELECT COUNT(*) AS AllRows,
 COUNT(PromoCode) AS OrdersWithPromo,
 COUNT(DISTINCT CustomerId) AS UniqueCustomers,
 COUNT(DISTINCT PromoCode) AS DistinctPromosUsed -- excludes NULL
FROM Orders;
```

**Gotcha.** To count rows matching a condition, use conditional aggregation rather than several queries:

```sql
SELECT SUM(CASE WHEN Status = 'Paid' THEN 1 ELSE 0 END) AS Paid,
 SUM(CASE WHEN Status = 'Pending' THEN 1 ELSE 0 END) AS Pending,
 COUNT(*) AS Total
FROM Invoices;
-- (COUNT(CASE WHEN cond THEN 1 END) works too, since COUNT ignores the NULL else.)
```

---

### Q16. `DISTINCT` vs `GROUP BY` — when are they equivalent, and which should you use?

**Answer.** For pure de-duplication of a column set with no aggregates, `SELECT DISTINCT a, b` and `SELECT a, b … GROUP BY a, b` produce the same result and usually the same plan. Use `GROUP BY` when you also need aggregates; use `DISTINCT` when you only want unique rows and it reads more clearly.

```sql
-- Equivalent
SELECT DISTINCT Region, City FROM Customers;
SELECT Region, City FROM Customers GROUP BY Region, City;

-- Only GROUP BY can also aggregate:
SELECT Region, City, COUNT(*) AS Cnt
FROM Customers GROUP BY Region, City;
```

**Gotchas.**
- `DISTINCT` applies to the **entire** `SELECT` list, not one column — `SELECT DISTINCT a, b` dedups on the pair `(a, b)`, not on `a`. There is no "distinct on a, any b" in standard SQL (PostgreSQL has `DISTINCT ON (a)` for exactly that).
- `DISTINCT` is not free — it forces a sort or hash to remove duplicates. If the data is already unique (e.g. selecting a primary key), a stray `DISTINCT` just adds cost. Don't sprinkle it to "fix" duplicate rows caused by a bad join (Q8) — fix the join.

---

### Q17. How do you concatenate grouped values into a single string?

**Answer.** Use **`STRING_AGG`** (SQL Server 2017+), the standard way to aggregate strings within a group, with an optional `WITHIN GROUP (ORDER BY …)` for deterministic ordering.

```sql
-- One row per customer, comma-separated list of their product names
SELECT o.CustomerId,
 STRING_AGG(p.Name, ', ') WITHIN GROUP (ORDER BY p.Name) AS Products
FROM Orders o
JOIN Products p ON p.ProductId = o.ProductId
GROUP BY o.CustomerId;
```

Like other aggregates, `STRING_AGG` **skips NULLs**. On older SQL Server (pre-2017) the idiom was the `FOR XML PATH('')` + `STUFF` hack — know it exists but reach for `STRING_AGG` today. **PostgreSQL note.** Equivalent is `STRING_AGG(expr, ', ' ORDER BY …)` (same name, slightly different ordering syntax), or `ARRAY_AGG` for an array.

---

## T5 — Set operators: UNION / UNION ALL / EXCEPT / INTERSECT

### Q18. `UNION` vs `UNION ALL` — what's the difference and which is faster?

**Answer.** Both stack the rows of two result sets vertically. **`UNION` removes duplicates** (across the combined set), which requires a distinct/sort or hash step and is therefore costly. **`UNION ALL` keeps every row**, does no dedup, and is significantly faster. **Use `UNION ALL` whenever you know the inputs cannot overlap** (or you want the duplicates) — which is most of the time.

```sql
-- Dedups across both sets (extra sort/hash cost)
SELECT City FROM Customers
UNION
SELECT City FROM Suppliers;

-- Faster; use when overlap is impossible or duplicates are wanted
SELECT OrderId, 'Online' AS Channel FROM OnlineOrders
UNION ALL
SELECT OrderId, 'InStore' AS Channel FROM StoreOrders;
```

**Gotcha.** Reaching for `UNION` "just to be safe" on large sets is a common, silent performance drain. Prove overlap is possible before paying for dedup.

---

### Q19. What are the column-compatibility and `ORDER BY` rules for set operators?

**Answer.** All set operators (`UNION [ALL]`, `EXCEPT`, `INTERSECT`) require the operand queries to have:

1. the **same number of columns**, and
2. **compatible (implicitly convertible) data types** positionally — matching is by *position*, not by name.

The output column names come from the **first** query. **`ORDER BY` is allowed only once, at the very end**, and applies to the whole combined result (it cannot appear on the individual operands, except inside a `TOP`/`OFFSET` subquery).

```sql
SELECT Id, Name FROM A
UNION ALL
SELECT Id, Title FROM B -- matched by position: A.Name <-> B.Title
ORDER BY Name; -- single ORDER BY, at the end, using first-query names
```

**Gotcha.** Because matching is positional, swapping two columns in one operand compiles fine but silently mixes data. And a type mismatch (e.g. `int` vs `varchar` in the same position) either forces a conversion or errors.

---

### Q20. What do `EXCEPT` and `INTERSECT` do?

**Answer.** They are set difference and set intersection over **distinct** rows (comparing the full row, and treating NULLs as equal for this purpose — unlike `=`).

- **`EXCEPT`** — rows in the first query that are **not** in the second (set difference), de-duplicated.
- **`INTERSECT`** — rows present in **both** queries, de-duplicated.

```sql
-- Customers who have never placed an order (compare full row / key column)
SELECT CustomerId FROM Customers
EXCEPT
SELECT CustomerId FROM Orders;

-- Customers who are also suppliers (by id)
SELECT PartyId FROM Customers
INTERSECT
SELECT PartyId FROM Suppliers;
```

Note `EXCEPT`/`INTERSECT` compare **entire rows** and are NULL-aware (two NULLs match), which makes `EXCEPT` a tidy NULL-safe alternative to `NOT IN` for whole-row exclusion. **PostgreSQL note.** Same operators, and Postgres additionally offers `EXCEPT ALL` and `INTERSECT ALL` (multiset semantics that keep duplicate multiplicities); SQL Server has no `ALL` variant for these two.

---

## T6 — EXISTS vs IN vs JOIN

### Q21. Compare `EXISTS`, `IN`, and `JOIN` for testing membership — when is each best?

**Answer.** All three can express "rows in A that have a related row in B," but their semantics and best fit differ.

- **`EXISTS` (semi-join)** — a correlated subquery that **short-circuits on the first matching row**; it never materializes or de-duplicates the inner set. It answers a pure yes/no. Best for correlated conditions and large/duplicated inner sets, and it never inflates A's cardinality.
- **`IN` (subquery)** — conceptually materializes the list of values, then checks membership. Clean for simple, uncorrelated "value in this set" tests. Watch the NULL trap on the `NOT` form (Q14/Q22).
- **`JOIN`** — combines rows and can **return columns from B**. But an `INNER JOIN` to a one-to-many B **duplicates** A's rows (Q8), so you often need `DISTINCT` — at which point `EXISTS` is usually cleaner and cheaper if you don't actually need B's columns.

```sql
-- Customers who have at least one order:

-- EXISTS: semi-join, no duplication, stops at first match
SELECT c.CustomerId FROM Customers c
WHERE EXISTS (SELECT 1 FROM Orders o WHERE o.CustomerId = c.CustomerId);

-- IN: fine for a simple uncorrelated set
SELECT c.CustomerId FROM Customers c
WHERE c.CustomerId IN (SELECT o.CustomerId FROM Orders o);

-- JOIN: needs DISTINCT because a customer with 5 orders returns 5 rows
SELECT DISTINCT c.CustomerId FROM Customers c
JOIN Orders o ON o.CustomerId = c.CustomerId;
```

**Performance.** Modern optimizers (SQL Server, PostgreSQL) frequently compile `EXISTS` and `IN` to the *same* semi-join plan, so they are often equivalent. Use `JOIN` when you need B's columns; use `EXISTS` for existence tests, especially when the subquery is correlated, the inner set is large or duplicated, or NULLs are in play.

---

### Q22. `NOT EXISTS` vs `NOT IN` — why prefer `NOT EXISTS`?

**Answer.** Because **`NOT IN` breaks with NULLs and `NOT EXISTS` does not.** As shown in Q14, if the `NOT IN` subquery returns even one NULL, the entire predicate can only be `UNKNOWN`, so the query returns **zero rows** — a silent, data-dependent bug that hides until a NULL sneaks into the source. `NOT EXISTS` uses `IS`-style matching internally and is completely NULL-safe.

```sql
-- Risky: returns NOTHING if any CustomerId in Orders is NULL
SELECT c.CustomerId FROM Customers c
WHERE c.CustomerId NOT IN (SELECT o.CustomerId FROM Orders o);

-- Safe: correct regardless of NULLs -> prefer this
SELECT c.CustomerId FROM Customers c
WHERE NOT EXISTS (SELECT 1 FROM Orders o WHERE o.CustomerId = c.CustomerId);

-- If you must use NOT IN, defend it explicitly:
SELECT c.CustomerId FROM Customers c
WHERE c.CustomerId NOT IN (SELECT o.CustomerId FROM Orders o
 WHERE o.CustomerId IS NOT NULL);
```

`NOT EXISTS` also tends to plan better as an **anti-semi-join** and short-circuits on the first match. **Rule: default to `NOT EXISTS` for exclusion.** (`EXCEPT` is another NULL-safe option for whole-row difference — Q20.)

**Gotcha.** The `LEFT JOIN … WHERE b.key IS NULL` anti-join (Q7) is also NULL-safe and equivalent, but reads less clearly and is easy to break by filtering the wrong column. `NOT EXISTS` is the senior default.

---

## T7 — Everyday T-SQL: sorting, TOP, LIKE, CASE, BETWEEN, aliases

### Q23. How do you sort results, and what is the "default" sort order?

**Answer.** Sorting is `ORDER BY`, and the single most important point is the one juniors get wrong: **without an explicit `ORDER BY`, SQL Server makes no guarantee about row order at all.** There is no "default sort." Rows may come back in clustered-index order today and in a different order tomorrow — a plan change, parallelism, or a page split is enough to change it. If order matters, say so.

Within `ORDER BY`, the per-column default is **`ASC`** (ascending); `DESC` must be spelled out on **each** column that needs it.

```sql
SELECT Name, Salary, HireDate
FROM Employees
ORDER BY Salary DESC,      -- DESC applies to Salary only
         HireDate,          -- ASC (the default)
         Name;

-- Sort by an expression or a SELECT alias (both legal — ORDER BY runs after SELECT)
SELECT Name, Salary * 1.1 AS Projected
FROM Employees
ORDER BY Projected DESC;
```

**Gotchas.**

- **`ORDER BY` in a view/CTE/derived table does not stick.** `ORDER BY` inside a subquery is only legal alongside `TOP`/`OFFSET`, and even then it defines *which* rows are picked, **not** the order they're returned in. The outer query needs its own `ORDER BY`. Same for a view: sorting inside the view definition guarantees nothing to the consumer.
- **NULL placement.** SQL Server sorts `NULL` **first** in `ASC` and treats it as the lowest value; there is no `NULLS LAST` clause. Emulate it with a leading sort key: `ORDER BY CASE WHEN col IS NULL THEN 1 ELSE 0 END, col`. **PostgreSQL note:** it sorts NULLs **last** in `ASC` by default and does support `NULLS FIRST` / `NULLS LAST` — a real portability difference.
- **`ORDER BY` ordinal** (`ORDER BY 2`) works but is fragile — it silently re-points if the `SELECT` list changes. Name the column.

---

### Q24. How do you return only the first N rows — and what makes it deterministic?

**Answer.** `TOP (n)` in T-SQL, or the ANSI-standard `OFFSET … FETCH` (SQL Server 2012+). The critical rule: **`TOP` without `ORDER BY` is non-deterministic** — you get *some* n rows, not the "first" or "best" n, and the set can change between runs.

```sql
-- Top 10 by salary (deterministic ordering key)
SELECT TOP (10) Name, Salary
FROM Employees
ORDER BY Salary DESC;

-- ANSI form — the only one that also skips rows
SELECT Name, Salary
FROM Employees
ORDER BY Salary DESC
OFFSET 0 ROWS FETCH NEXT 10 ROWS ONLY;

-- WITH TIES: also return anyone tied with the 10th row's salary
SELECT TOP (10) WITH TIES Name, Salary
FROM Employees
ORDER BY Salary DESC;      -- may return 11+ rows

SELECT TOP (5) PERCENT Name FROM Employees ORDER BY Salary DESC;
```

- **`WITH TIES`** requires `ORDER BY` and returns *extra* rows that tie on the ordering key — useful for "top 10 earners" where cutting a tie arbitrarily would be wrong.
- **`PERCENT`** takes a percentage of the result set, rounded up.
- **Ties are the real trap.** With duplicate `Salary` values, `TOP (10) ORDER BY Salary DESC` picks an arbitrary winner among the tied rows. Add a **tiebreaker** to make it stable: `ORDER BY Salary DESC, EmployeeId`.

`TOP` also applies to `INSERT`/`UPDATE`/`DELETE` (`DELETE TOP (1000) FROM …`), which is the basis of batched DML — see [sql-nosql-performance.md](sql-nosql-performance.md) Q16. For paging, `OFFSET/FETCH` degrades on deep pages; keyset pagination is the fix ([sql-advanced-queries.md](sql-advanced-queries.md) Q16–Q17). **PostgreSQL note:** `LIMIT 10 OFFSET 0`; no `TOP`, no `WITH TIES` (use `RANK()` filtering, or `FETCH … WITH TIES` in PG13+).

---

### Q25. What are wildcards, and how does `LIKE` use them?

**Answer.** `LIKE` does pattern matching on strings using wildcards:

| Wildcard | Meaning | Example |
|---|---|---|
| `%` | zero or more characters | `LIKE 'Jo%'` → John, Jones, Jo |
| `_` | **exactly one** character | `LIKE 'J_n'` → Jan, Jon (not Joan) |
| `[…]` | one character from the set/range | `LIKE '[JM]ones'` → Jones, Mones |
| `[^…]` | one character **not** in the set | `LIKE '[^A]%'` → doesn't start with A |

```sql
WHERE Name LIKE 'Jo%'            -- prefix: SARGable, can seek
WHERE Name LIKE '%son'           -- leading wildcard: NOT SARGable, forces a scan
WHERE Code LIKE '[A-Z][0-9][0-9]'-- one letter then two digits

-- Searching for a literal % or _ : use ESCAPE (or brackets)
WHERE Discount LIKE '50!%' ESCAPE '!'   -- matches the text "50%"
WHERE Col LIKE '[_]tmp'                 -- literal underscore
```

**Gotchas.**

- **A leading `%` kills index seeks** — the engine can't navigate a B-tree without a known prefix, so `LIKE '%term%'` scans. This is the #1 performance issue with `LIKE`; for real text search use **full-text search** or a computed/reversed column. See SARGability in [sql-indexing-plans.md](sql-indexing-plans.md) Q22.
- **`_` and `%` inside user input are wildcards, not literals.** A search box passing `50%` or `a_b` straight into `LIKE` silently matches far too much — escape user input.
- **Case sensitivity follows the collation**, not `LIKE`. On a default case-insensitive collation `LIKE 'jo%'` matches "John". Don't wrap the column in `UPPER()` to force it — that breaks SARGability.

---

### Q26. How do you write conditional logic in a query?

**Answer.** `CASE` — an expression (it returns a value), not a control-flow statement, so it can appear anywhere an expression is legal: `SELECT`, `WHERE`, `ORDER BY`, `GROUP BY`, `HAVING`, even inside aggregates. Two forms:

```sql
-- Searched CASE (evaluates conditions in order) — the flexible form
SELECT Name,
       CASE WHEN Salary >= 100000 THEN 'Senior'
            WHEN Salary >=  60000 THEN 'Mid'
            ELSE 'Junior'
       END AS Band
FROM Employees;

-- Simple CASE (equality against one expression only)
SELECT CASE Status
            WHEN 1 THEN 'Active'
            WHEN 2 THEN 'Suspended'
            ELSE 'Unknown'
       END AS StatusName
FROM Accounts;
```

Common senior uses: **conditional aggregation** (Q15), **pivoting** without `PIVOT` ([sql-query-problems.md](sql-query-problems.md) P27), custom sort priority (`ORDER BY CASE WHEN Status='Urgent' THEN 0 ELSE 1 END, DueDate`), and guarding division (`CASE WHEN d = 0 THEN NULL ELSE n / d END`).

**Gotchas.**
- **No `ELSE` → unmatched rows get `NULL`,** silently. Always ask whether you want a default.
- **First match wins**, so order matters — reversing the two `WHEN`s above would band everyone as 'Mid'.
- **A single return type.** All `THEN`/`ELSE` branches are reconciled to the highest-precedence type; mixing `'N/A'` with an `int` throws a conversion error. Cast deliberately.
- `CASE` is not a substitute for `COALESCE`/`ISNULL` on a simple NULL check (Q13), and short-circuiting is **not** guaranteed with aggregates in the branches.

---

### Q27. Explain `BETWEEN` — and the classic date bug.

**Answer.** `BETWEEN a AND b` is shorthand for `>= a AND <= b` — **both endpoints inclusive**. It's readable and fully SARGable (a range seek), so it's the right tool for numeric and date ranges.

```sql
WHERE Salary BETWEEN 50000 AND 80000     -- includes exactly 50000 and 80000
WHERE Salary NOT BETWEEN 50000 AND 80000
```

**The classic bug.** On a `datetime`/`datetime2` column, `BETWEEN '2026-01-01' AND '2026-01-31'` **silently drops almost all of January 31st** — the upper bound is midnight `2026-01-31 00:00:00`, so an order timestamped `2026-01-31 09:15` is excluded. Interviewers love this one.

```sql
-- WRONG: loses everything after midnight on the 31st
WHERE OrderDate BETWEEN '2026-01-01' AND '2026-01-31';

-- RIGHT: half-open range [start, end) — correct at any precision, still SARGable
WHERE OrderDate >= '2026-01-01' AND OrderDate < '2026-02-01';
```

**Use the half-open `>= … < …` pattern for all datetime ranges.** It's immune to precision (`datetime`'s 3.33 ms rounding, `datetime2(7)`'s 100 ns), needs no `23:59:59.997` fudge, and survives a column type change. Reserve `BETWEEN` for `date`-typed columns and discrete integers.

**Gotchas.**
- **`a` must be ≤ `b`** — `BETWEEN 80000 AND 50000` returns nothing rather than erroring.
- **NULL propagates** — if any operand is NULL the result is `UNKNOWN` and the row is filtered out (Q11).
- Don't wrap the column to fix ranges (`WHERE YEAR(OrderDate) = 2026`) — that's non-SARGable; use the half-open range.

---

### Q28. What is an alias and where must you use one?

**Answer.** An alias is a temporary name for a **column** (`AS`) or a **table/subquery** (a table alias). Column aliases name the output; table aliases disambiguate and shorten qualified references.

```sql
SELECT e.Name          AS EmployeeName,   -- column alias (AS is optional but clearer)
       m.Name          AS ManagerName,
       e.Salary * 12   AS AnnualSalary    -- REQUIRED: expressions have no natural name
FROM Employees      AS e                  -- table alias
LEFT JOIN Employees AS m ON m.EmployeeId = e.ManagerId;
```

Aliases are **mandatory** in three places: self-joins (the same table twice — Q6), any derived table (`FROM (SELECT …) AS t` is a syntax error without the alias), and computed/aggregate output columns you intend to reference or display.

**Gotchas.**
- **You cannot reference a `SELECT` alias in `WHERE`, `GROUP BY`, or `HAVING`** — those run before `SELECT` (Q1). Repeat the expression, or wrap it in a CTE/derived table. `ORDER BY` is the exception.
- **Once a table is aliased, the alias replaces the name** — `FROM Employees AS e` makes `Employees.Name` invalid; you must write `e.Name`.
- Aliases in a `LIKE`/quoted form (`AS 'Employee Name'`) are legal but prefer `AS [Employee Name]`; single-quoted aliases are deprecated.
- **Always alias every table in a multi-table query.** Unqualified columns in a join are ambiguous the moment someone adds a same-named column to the other table — a real-world source of breakage.

---

### Q29. Which aggregate functions are there, and what should you know about each?

**Answer.** The core five plus the ones seniors are expected to know:

- **`COUNT`** — rows (`COUNT(*)`) or non-NULL values (`COUNT(col)`); see Q15.
- **`SUM`** / **`AVG`** — numeric only. Both **ignore NULLs**, which is why `AVG` divides by the non-NULL count, not the row count (Q12).
- **`MIN`** / **`MAX`** — work on numbers, strings, and dates (comparison order follows collation for strings).
- **`STRING_AGG`** — string concatenation per group (Q17).
- **`STDEV` / `STDEVP` / `VAR` / `VARP`** — sample vs population statistics.
- **`COUNT_BIG`** — returns `bigint`; use it when a count can exceed 2.1 billion (`COUNT` overflows and errors).

```sql
SELECT DepartmentId,
       COUNT(*)      AS Headcount,
       SUM(Salary)   AS Payroll,
       AVG(Salary)   AS AvgSalary,     -- ignores NULL salaries in the denominator
       MIN(HireDate) AS FirstHire,
       MAX(Salary)   AS TopSalary
FROM Employees
GROUP BY DepartmentId;
```

**Gotchas.**

- **`AVG` on an integer column does integer division** — `AVG(Score)` over 7, 8 returns `7`, not `7.5`. Cast first: `AVG(CAST(Score AS DECIMAL(10,2)))`. A frequent silent-wrong-answer bug.
- **All aggregates except `COUNT(*)` skip NULLs** — so `SUM` over all-NULL rows is `NULL`, not `0`. Wrap with `COALESCE(SUM(x), 0)` when a zero is wanted.
- Aggregates can't be nested directly (`SUM(MAX(x))` is illegal) — use a CTE/derived table, or a window function ([sql-advanced-queries.md](sql-advanced-queries.md)).
- `DISTINCT` works inside most aggregates: `SUM(DISTINCT Amount)`, `AVG(DISTINCT x)`.

---