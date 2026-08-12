# SQL — Scenario Questions
---

> **Format:** situational questions — *"in production you see X, what happened and what do you do?"* You talk through the cause and the fix rather than writing a puzzle query.
> **Related:** for "write this query" problems see [sql-query-problems.md](sql-query-problems.md). The underlying concepts live in sections [T](sql-fundamentals-joins.md)–[Y](sql-nosql-performance.md).

---

## S1 — Wrong Results

### Q1. A LEFT JOIN report used to show all customers. Now customers with no orders have disappeared. What happened?

Someone put a filter on the **right** table in the `WHERE` clause, which turns a `LEFT JOIN` back into an `INNER JOIN`.

A `LEFT JOIN` keeps customers with no orders by filling the order columns with `NULL`. Then `WHERE o.Status = 'Shipped'` runs *after* the join, and for those customers `o.Status` is `NULL`. `NULL = 'Shipped'` is not true, so the row is dropped — exactly the rows the `LEFT JOIN` was protecting.

**The fix:** move the condition into the `ON` clause.

```sql
-- Bug: customers with no shipped order vanish
LEFT JOIN Orders o ON o.CustomerId = c.Id
WHERE o.Status = 'Shipped';

-- Fix: the filter decides what counts as a match
LEFT JOIN Orders o ON o.CustomerId = c.Id
                  AND o.Status = 'Shipped';
```

The one thing that *is* safe in `WHERE` is `IS NULL`, used deliberately to find customers with no orders.

---

### Q2. Revenue on a report suddenly doubled after someone added a join. Why?

The new join multiplied the rows. This is called **fan-out**.

If you join `Orders` to `OrderLines`, an order with 3 line items becomes 3 rows. `SUM(o.Total)` then adds that order's total three times. The clue is that the number is a rough multiple of the truth, and the aggregated column lives on the parent table.

**The fix depends on why the join is there:**

- Only needed to **filter** ("orders that contain product X") → use `EXISTS` instead, which never duplicates rows.
- Genuinely need child columns → aggregate the child table down to one row per order first, then join.

`COUNT` inflates the same way. Whenever you join to a "many" side, check every existing `SUM` and `COUNT`.

---

### Q3. A dashboard total disagrees with what finance expects, and NULLs are involved. What's the subtlety?

Aggregates **skip NULLs**, and that surprises people.

- `COUNT(*)` counts rows. `COUNT(Discount)` counts only rows where `Discount` is not NULL. Different numbers.
- `AVG(Discount)` divides by the count of **non-NULL** rows. If NULL means "no discount" (really zero), the average comes out too high — those rows should have been counted as 0.
- `SUM` of an all-NULL set returns `NULL`, not 0, and that `NULL` then spreads through any later arithmetic.

```sql
AVG(Discount)              -- ignores NULL rows entirely
AVG(ISNULL(Discount, 0))   -- treats them as zero
```

**The fix:** decide per column whether NULL means "unknown" (skip it) or "zero" (use `ISNULL`), and agree that with whoever owns the number.

---

### Q4. Duplicate rows appeared in a table that was supposed to be unique. How do you handle it?

If duplicates got in, there was **no unique constraint** — uniqueness was only checked in application code, and that check does not hold under concurrency. Two requests can both run "does this exist?", both get "no", and both insert.

**The fix is two steps:**

1. **Remove the existing duplicates.** Use `ROW_NUMBER()` partitioned by the key and delete where `rn > 1`. Decide first which copy to keep — that's a business question, not a technical one.
2. **Add a unique index** on that key so it cannot happen again.

Only the database can guarantee uniqueness across concurrent requests. Also fix the insert path — usually a retried API call that needs to be idempotent.

---

## S2 — Concurrency

### Q5. Two users edit the same record on a form. Both save successfully, but one person's changes vanish. What happened?

This is a **lost update**, and the default isolation level does not prevent it.

User A loads the record. User B loads the same record. A saves. B saves — writing back the fields *as B loaded them*, overwriting A's change. Nobody gets an error.

**The fix is optimistic concurrency.** Add a `rowversion` column and include it in the `WHERE` clause of the update:

```sql
UPDATE Orders
SET Status = @status, RowVer = ...
WHERE Id = @id AND RowVer = @rowVerWhenIRead;
```

If someone changed the row since you read it, the version no longer matches and **zero rows update** — the app detects that and asks the user to reload. EF Core surfaces this as `DbUpdateConcurrencyException`.

Don't hold a database lock while a user fills in a form — they might go to lunch.

---

### Q6. Users occasionally get "deadlock victim" errors. What causes them and how do you reduce them?

A deadlock is a **circular wait** — two transactions each hold a lock the other one needs. SQL Server picks one and kills it. "Occasional" is normal; they only fire when the timing lines up.

The usual cause is **inconsistent ordering**: one piece of code updates table A then B, another updates B then A.

**What to do:**

- Make all code touch tables in the **same order** — this removes the whole category.
- Keep transactions **short**. Long transactions hold locks longer and collide more.
- Check for a **missing index** — a query that scans locks far more rows than it needs to.
- **Retry in the application.** The victim's transaction is rolled back cleanly, so retrying usually just works (`EnableRetryOnFailure` in EF Core).

You can make deadlocks rare, but never impossible, so the retry is not optional.

---

## S3 — Performance

### Q7. A query that was fast last month is slow now, and nobody changed the code. Where do you look?

If the SQL text is identical, then something *around* it changed. Three things to check, in order:

**Data volume.** The table grew. A plan that was fine at 10,000 rows can be wrong at 10 million.

**Statistics.** SQL Server uses statistics to guess how many rows a query will touch. On a fast-growing table these go stale, the guess goes wrong, and it picks a bad plan. `UPDATE STATISTICS` often fixes it immediately — which also confirms the diagnosis.

**Blocking.** Check whether the query is actually *working* or just *waiting* for someone else's lock. If it's waiting, this is a concurrency problem, not a query problem.

Then look at the execution plan and compare **estimated vs actual rows**. A big gap points straight at stale statistics.

---

### Q8. A paged search screen is instant on page 1 but crawls when users jump to page 500. Why?

`OFFSET` does not skip ahead — it **reads every row and throws away the ones before your page**. Page 500 reads 10,000 rows to return 20. The deeper the page, the slower it gets.

**The fix is keyset pagination.** Instead of an offset, remember the last row you showed and ask for rows after it:

```sql
-- Instead of OFFSET 10000
WHERE (OrderDate, Id) < (@lastDate, @lastId)
ORDER BY OrderDate DESC, Id DESC;
```

With an index on those columns, every page costs the same.

**The trade-off:** you get next/previous but not "jump to page 500", because you don't know that page's starting point. That's fine for infinite scroll, and honestly nobody scrolls to page 500 — if they're trying to, they need better search filters.

---

**Cross-references:** indexing and query plans → [sql-indexing-plans.md](sql-indexing-plans.md) · isolation and locking → [sql-transactions-locking.md](sql-transactions-locking.md) · joins and NULLs → [sql-fundamentals-joins.md](sql-fundamentals-joins.md) · pagination → [sql-advanced-queries.md](sql-advanced-queries.md)
