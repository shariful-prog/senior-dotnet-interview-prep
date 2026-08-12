# Z. Common Interview Query Problems
---

## Z1 — Ranking & Nth-Highest

### P1. Find the second-highest salary.

**Problem.** Return the second-highest salary from an `Employees` table. If there is no second-highest (e.g. one row, or every row has the same salary), return `NULL` rather than an empty result set. Treat equal salaries as one salary (i.e. the *distinct* 2nd salary, not the 2nd row).

```sql
-- Sample data
-- CREATE TABLE Employees (Id INT, Name VARCHAR(30), Salary INT);
-- INSERT INTO Employees VALUES (1,'A',100),(2,'B',100),(3,'C',90),(4,'D',80);
-- Distinct salaries: 100, 90, 80 -> 2nd highest = 90

-- Approach A (recommended): DENSE_RANK over distinct salary values
SELECT MAX(Salary) AS SecondHighest
FROM (
 SELECT Salary, DENSE_RANK() OVER (ORDER BY Salary DESC) AS rnk
 FROM Employees
) t
WHERE rnk = 2;

-- Approach B (classic subquery, returns NULL when none exists)
SELECT MAX(Salary) AS SecondHighest
FROM Employees
WHERE Salary < (SELECT MAX(Salary) FROM Employees);

-- Approach C (OFFSET/FETCH — needs DISTINCT to skip ties correctly)
SELECT DISTINCT Salary AS SecondHighest
FROM Employees
ORDER BY Salary DESC
OFFSET 1 ROWS FETCH NEXT 1 ROW ONLY;
```

**Why / approach.** The trap is **ties**. With salaries `100,100,90,80`:
- `ROW_NUMBER()` would assign 1,2,3,4 → "row 2" is the *second 100*, which is wrong for "2nd highest salary". Use `DENSE_RANK`, which gives 1,1,2,3 → rank 2 is `90`. `RANK` also works here because we filter on rank = 2 and there is no gap-before issue at rank 2, but **`DENSE_RANK` is the safe default** for "Nth distinct value".
- **Approach B** is the shortest and elegant: it naturally returns `NULL` (not an empty set) when there's no smaller salary, because `MAX` over zero rows is `NULL`. This is the interviewer's favorite "no second salary → NULL" edge case. Downside: it doesn't generalize to Nth cleanly.
- **Approach C** wrapped in a subquery with `ISNULL`/`MAX` if you must guarantee NULL-on-empty, since bare `OFFSET/FETCH` returns an empty set, not NULL.

Windowing is covered in [sql-advanced-queries.md](sql-advanced-queries.md).
**PostgreSQL:** identical window functions; `OFFSET 1 LIMIT 1` instead of `OFFSET…FETCH`.

---

### P2. Find the Nth-highest salary (generalized).

**Problem.** Parameterize P1: return the Nth-highest *distinct* salary. Should return `NULL` if fewer than N distinct salaries exist.

```sql
DECLARE @N INT = 3;

-- Recommended: DENSE_RANK, wrapped so empty -> NULL
SELECT MAX(Salary) AS NthHighest
FROM (
 SELECT Salary, DENSE_RANK() OVER (ORDER BY Salary DESC) AS rnk
 FROM Employees
) t
WHERE rnk = @N;

-- OFFSET/FETCH variant (skip N-1 distinct salaries)
SELECT DISTINCT Salary
FROM Employees
ORDER BY Salary DESC
OFFSET (@N - 1) ROWS FETCH NEXT 1 ROW ONLY;
```

**Why / approach.** `DENSE_RANK() … WHERE rnk = @N` reads as exactly "the Nth distinct salary". Wrapping in `MAX(...)` (over the single matching row or zero rows) guarantees `NULL` instead of an empty set — the classic requirement in LeetCode-style "Nth Highest Salary" problems. The `OFFSET (@N-1)` form is concise but returns empty (not NULL) when N is out of range, so prefer the aggregate wrapper when the spec demands NULL.

**Gotcha.** If the interviewer wants the *Nth row* (allowing duplicate salaries to count separately), swap `DENSE_RANK` for `ROW_NUMBER` and drop `DISTINCT`.

---

### P3. Find the highest-paid employee in each department.

**Problem.** For every department, return the employee(s) with the maximum salary. Handle ties (multiple employees tied at the top → return all of them).

```sql
-- Sample data
-- Employees(Id, Name, DeptId, Salary)

-- Recommended: RANK to keep all ties
WITH r AS (
 SELECT Id, Name, DeptId, Salary,
 RANK() OVER (PARTITION BY DeptId ORDER BY Salary DESC) AS rnk
 FROM Employees
)
SELECT Id, Name, DeptId, Salary
FROM r
WHERE rnk = 1;

-- If you want exactly ONE per department (arbitrary tie-break), use ROW_NUMBER:
WITH r AS (
 SELECT Id, Name, DeptId, Salary,
 ROW_NUMBER() OVER (PARTITION BY DeptId ORDER BY Salary DESC, Id) AS rn
 FROM Employees
)
SELECT * FROM r WHERE rn = 1;
```

**Why / approach.** `PARTITION BY DeptId` restarts the ranking per department — the core idea for all "per group" problems (see Z3). Choose the ranking function by tie policy:
- `RANK() = 1` → **all** employees tied at the max.
- `ROW_NUMBER() = 1` → **exactly one**, broken deterministically by the extra `ORDER BY Id`.

Avoid the naive `WHERE Salary = (SELECT MAX(Salary) FROM Employees e2 WHERE e2.DeptId = e.DeptId)` correlated subquery — it's correct but re-scans per row and reads worse. A single window pass is cleaner and usually faster with a supporting index on `(DeptId, Salary DESC)` — see [sql-indexing-plans.md](sql-indexing-plans.md).

---

### P4. Rank employees by salary with no gaps vs with gaps.

**Problem.** Show the difference between `ROW_NUMBER`, `RANK`, and `DENSE_RANK` on the same data — a common "explain the output" question.

```sql
SELECT Name, Salary,
 ROW_NUMBER() OVER (ORDER BY Salary DESC) AS rn,
 RANK() OVER (ORDER BY Salary DESC) AS rnk,
 DENSE_RANK() OVER (ORDER BY Salary DESC) AS drnk
FROM Employees;
-- Salaries 100,100,90,80 ->
-- rn : 1,2,3,4 (unique, arbitrary among ties)
-- rnk: 1,1,3,4 (ties share, then SKIP -> gap)
-- drnk:1,1,2,3 (ties share, NO gap)
```

**Why / approach.** Memorize: **ROW_NUMBER** = always unique; **RANK** = ties share, leaves gaps; **DENSE_RANK** = ties share, no gaps. Use ROW_NUMBER for pagination and "pick one per group", RANK/DENSE_RANK for leaderboards and "Nth distinct value".

---

## Z2 — Duplicates

### P5. Find duplicate rows.

**Problem.** Given `Contacts(Email, ...)`, list the email addresses that appear more than once, with their counts.

```sql
SELECT Email, COUNT(*) AS Cnt
FROM Contacts
GROUP BY Email
HAVING COUNT(*) > 1;
```

**Why / approach.** `GROUP BY` the duplicate key(s), then filter groups with `HAVING COUNT(*) > 1`. `HAVING` filters *after* aggregation (`WHERE` can't reference `COUNT`). For "duplicate on the whole row", list every column in the `GROUP BY`.

**Gotcha.** `NULL`s form their own group in `GROUP BY` (all NULLs are grouped together in SQL Server/PostgreSQL), so `NULL` emails would be counted as duplicates of each other — usually add `WHERE Email IS NOT NULL` if that's not intended.

---

### P6. Delete duplicate rows keeping one.

**Problem.** The table has exact/near-duplicate rows (same `Email`, different or same `Id`). Delete duplicates, keeping the row with the **smallest `Id`** per email. Table has no reliable unique constraint yet.

```sql
-- Recommended: ROW_NUMBER in a CTE, then DELETE where rn > 1
WITH ranked AS (
 SELECT Id, Email,
 ROW_NUMBER() OVER (PARTITION BY Email ORDER BY Id) AS rn
 FROM Contacts
)
DELETE FROM ranked
WHERE rn > 1;
```

```sql
-- If there is NO unique id at all (true duplicate rows across ALL columns),
-- you can still delete via ROW_NUMBER because the CTE is updatable:
WITH ranked AS (
 SELECT *, ROW_NUMBER() OVER (PARTITION BY Col1, Col2, Col3 ORDER BY (SELECT NULL)) AS rn
 FROM T
)
DELETE FROM ranked WHERE rn > 1;
```

**Why / approach.** This is *the* signature interview task. `ROW_NUMBER() OVER (PARTITION BY dupcols ORDER BY id)` numbers each duplicate group 1,2,3…; row 1 is the keeper, so `DELETE … WHERE rn > 1` removes the rest. The trick that surprises people: **in T-SQL you can `DELETE FROM` the CTE directly** — it deletes from the underlying table.

`ORDER BY (SELECT NULL)` is the idiom for "no meaningful order" when there's no id to pick a keeper by (arbitrary but valid).

**PostgreSQL:** the CTE-DELETE form isn't allowed; use `DELETE … USING` or `ctid`:
```sql
DELETE FROM contacts a
USING contacts b
WHERE a.email = b.email AND a.id > b.id; -- keep the smallest id
-- or delete by ctid when no id exists at all
```

**Follow-up — how do you delete *millions* of duplicates without wrecking production?** A single `DELETE` of millions of rows is a long transaction: it takes escalating locks (often a full table lock), blocks everyone, bloats the log, and rolls back for hours if it fails. **Delete in batches** so each transaction is short:

```sql
WHILE 1 = 1
BEGIN
    WITH ranked AS (
        SELECT Id, ROW_NUMBER() OVER (PARTITION BY Email ORDER BY Id) AS rn
        FROM dbo.Contacts
    )
    DELETE TOP (5000) FROM ranked WHERE rn > 1;   -- short, lock-friendly transaction

    IF @@ROWCOUNT = 0 BREAK;
    -- optional: WAITFOR DELAY '00:00:01' to let other work through
END
```

Each batch commits independently, so locks are released, the log can truncate between batches, and you can stop or resume safely. Keep batches under the **~5,000-lock escalation threshold** ([sql-transactions-locking.md](sql-transactions-locking.md) Q16), and make sure an index supports the `PARTITION BY` column or every batch re-scans the table.

**The alternative for a truly huge cleanup: copy-and-swap.** When you're deleting *most* of the table, `SELECT` the survivors into a new table, then swap names — a minimally-logged bulk insert plus a metadata rename beats deleting 90% of the rows row by row. Afterwards, **add the `UNIQUE` constraint/index** that should have existed, so duplicates can't come back — otherwise you'll be running this script again. (Batching trade-offs: [sql-nosql-performance.md](sql-nosql-performance.md) Q16.)

---

### P7. Find rows that are duplicated on a key but differ on another column.

**Problem.** Find emails used by more than one distinct `Name` (data-quality check).

```sql
SELECT Email
FROM Contacts
GROUP BY Email
HAVING COUNT(DISTINCT Name) > 1;
```

**Why / approach.** `COUNT(DISTINCT Name) > 1` per email finds keys with conflicting values — subtly different from P5's `COUNT(*) > 1`, which just counts row repetitions. Interviewers use this to check you know `COUNT(DISTINCT …)`.

---

## Z3 — Top-N Per Group / Latest Row Per Group

### P8. Get the latest order for each customer.

**Problem.** From `Orders(Id, CustomerId, OrderDate, Amount)`, return the single most-recent order per customer (break date ties by highest `Id`).

```sql
-- Approach A (recommended, general): ROW_NUMBER in a CTE
WITH ranked AS (
 SELECT *,
 ROW_NUMBER() OVER (PARTITION BY CustomerId
 ORDER BY OrderDate DESC, Id DESC) AS rn
 FROM Orders
)
SELECT Id, CustomerId, OrderDate, Amount
FROM ranked
WHERE rn = 1;

-- Approach B: CROSS APPLY (TOP 1) — great when driving from a small Customers table
SELECT c.CustomerId, o.Id, o.OrderDate, o.Amount
FROM Customers c
CROSS APPLY (
 SELECT TOP (1) *
 FROM Orders o
 WHERE o.CustomerId = c.CustomerId
 ORDER BY o.OrderDate DESC, o.Id DESC
) o;
```

**Why / approach.**
- **ROW_NUMBER** (A) is the go-to when you want to scan `Orders` once and slice per group; wrap in a CTE because you can't put a window function in a `WHERE` directly.
- **CROSS APPLY TOP 1** (B) shines when you have a driving table (`Customers`) and a covering index on `Orders(CustomerId, OrderDate DESC, Id DESC)` — the optimizer does an index seek + top per customer, which can be dramatically faster than ranking all rows when each customer has many orders (a "top-N per group" seek). `APPLY` is explained in [sql-fundamentals-joins.md](sql-fundamentals-joins.md).

Avoid the correlated-subquery form `WHERE OrderDate = (SELECT MAX(OrderDate) …)` — it breaks on date ties (returns 2 rows) and needs a second tie-break condition.

---

### P9. Get the top 3 products by sales in each category.

**Problem.** `Products(Id, CategoryId, Name, Sales)` — return the three best-selling products per category (allow ties to spill past 3? Here: exactly top 3 rows).

```sql
WITH ranked AS (
 SELECT *,
 ROW_NUMBER() OVER (PARTITION BY CategoryId ORDER BY Sales DESC) AS rn
 FROM Products
)
SELECT CategoryId, Name, Sales
FROM ranked
WHERE rn <= 3
ORDER BY CategoryId, rn;
```

**Why / approach.** Same shape as P8 but `WHERE rn <= N`. Choose the ranking function by tie policy:
- `ROW_NUMBER <= 3` → exactly 3 rows (arbitrary among ties at the boundary).
- `DENSE_RANK <= 3` → top-3 *distinct* sales values, may return more than 3 rows if ties exist.

The `CROSS APPLY … TOP (3)` alternative from P8 also generalizes and is the fastest path when categories are few but products per category are many (index seek per category).

---

### P10. First and last event per user in one pass.

**Problem.** For each user, get the timestamp of their first and last login.

```sql
SELECT UserId,
 MIN(LoginTime) AS FirstLogin,
 MAX(LoginTime) AS LastLogin
FROM Logins
GROUP BY UserId;
```

**Why / approach.** When you only need boundary *values* (not the whole boundary row), plain `MIN`/`MAX` with `GROUP BY` beats window functions — one aggregate pass, no ranking. Reach for ROW_NUMBER (P8) only when you need other columns *from* the first/last row.

---

## Z4 — Running Totals, Moving Averages & Period-over-Period

### P11. Running (cumulative) total of sales by date.

**Problem.** `Sales(SaleDate, Amount)` — show each day's amount plus the running total up to and including that day.

```sql
SELECT
    SaleDate,
    SUM(Amount) AS DailyAmount,
    SUM(SUM(Amount)) OVER (ORDER BY SaleDate) AS RunningTotal
FROM Sales
GROUP BY SaleDate;
```

---

### P12. 7-day moving average.

**Problem.** Compute a trailing 7-row moving average of daily amount.

```sql
SELECT SaleDate, Amount,
 AVG(Amount) OVER (ORDER BY SaleDate
 ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS Mov7
FROM Sales
ORDER BY SaleDate;
```

---

### P13. Cumulative percentage of total.

**Problem.** For each product, show its share of total sales and the cumulative share (for a Pareto/80-20 chart).

```sql
SELECT Name, Sales,
 CAST(100.0 * Sales / SUM(Sales) OVER () AS DECIMAL(5,2)) AS PctOfTotal,
 CAST(100.0 * SUM(Sales) OVER (ORDER BY Sales DESC
 ROWS UNBOUNDED PRECEDING)
 / SUM(Sales) OVER () AS DECIMAL(5,2)) AS CumulativePct
FROM Products
ORDER BY Sales DESC;
```

**Why / approach.** `SUM(Sales) OVER ()` with an **empty** `OVER()` = grand total on every row (no partition, no order = whole set). Divide the running sum by it for cumulative %. Use `100.0` (not `100`) to force decimal division — integer division is a frequent gotcha in SQL Server.

---

### P14. Month-over-month growth with LAG.

**Problem.** `Monthly(YearMonth, Revenue)` — show each month's revenue, the prior month's revenue, and % growth.

```sql
SELECT YearMonth, Revenue,
 LAG(Revenue) OVER (ORDER BY YearMonth) AS PrevRevenue,
 CAST(100.0 * (Revenue - LAG(Revenue) OVER (ORDER BY YearMonth))
 / NULLIF(LAG(Revenue) OVER (ORDER BY YearMonth), 0) AS DECIMAL(6,2)) AS GrowthPct
FROM Monthly
ORDER BY YearMonth;
```

**Why / approach.** `LAG(x) OVER (ORDER BY …)` fetches the previous row's value without a self-join — the idiomatic period-over-period tool (`LEAD` for the next row). Wrap the denominator in `NULLIF(prev, 0)` to avoid divide-by-zero, and expect `NULL` growth for the first month (no prior). `LAG(Revenue, 1, 0)` supplies a default of 0 instead of NULL if desired.

**PostgreSQL:** identical (`LAG`/`LEAD` are standard).

---

### P15. Difference from the group average.

**Problem.** Show each employee's salary and how far it is from their department's average.

```sql
SELECT Name, DeptId, Salary,
 AVG(Salary) OVER (PARTITION BY DeptId) AS DeptAvg,
 Salary - AVG(Salary) OVER (PARTITION BY DeptId) AS DiffFromAvg
FROM Employees;
```

**Why / approach.** A windowed aggregate with `PARTITION BY` but **no `ORDER BY`** returns the group aggregate on *every* detail row — you keep row-level detail *and* the group stat side by side, which a plain `GROUP BY` cannot do without a join. This "detail + group aggregate together" pattern is one of the most useful window-function tricks.

---

## Z5 — Gaps & Islands

### P16. Find missing numbers in a sequence (gaps).

**Problem.** `Seq(N)` holds 1,2,3,5,6,9. Find the missing values (4,7,8) — assuming a contiguous expected range from `MIN` to `MAX`.

```sql
-- Numbers present: 1,2,3,5,6,9 -> missing 4,7,8
WITH bounds AS (SELECT MIN(N) AS lo, MAX(N) AS hi FROM Seq),
nums AS (
 SELECT lo AS N FROM bounds
 UNION ALL
 SELECT N + 1 FROM nums, bounds WHERE N + 1 <= hi
)
SELECT n.N AS MissingValue
FROM nums n
LEFT JOIN Seq s ON s.N = n.N
WHERE s.N IS NULL
OPTION (MAXRECURSION 0);
```

**Why / approach.** Generate the full expected range with a recursive CTE (or a numbers/tally table), then anti-join against what exists (`LEFT JOIN … IS NULL`). `OPTION (MAXRECURSION 0)` lifts the 100-level recursion cap for long ranges. A tally table is faster than recursion for big ranges — see P24.

**Simpler "gap edges only":** to report just where gaps start/end, `SELECT N + 1 AS GapStart FROM Seq WHERE N + 1 NOT IN (SELECT N FROM Seq)` — cheaper if you don't need every missing value listed.

---

### P17. Group consecutive runs (islands).

**Problem.** `Seq(N)` = 1,2,3,5,6,9. Collapse into consecutive ranges: (1–3), (5–6), (9–9).

```sql
WITH grp AS (
 SELECT N,
 N - ROW_NUMBER() OVER (ORDER BY N) AS island
 FROM Seq
)
SELECT MIN(N) AS RangeStart, MAX(N) AS RangeEnd, COUNT(*) AS Len
FROM grp
GROUP BY island
ORDER BY RangeStart;
```

**Why / approach.** The **islands trick**: for a run of consecutive integers, `value − ROW_NUMBER()` is *constant* (both increase by 1 in lockstep), so that difference becomes a group key. Any break in the sequence changes the difference, starting a new group. Then `GROUP BY` the computed key and take `MIN`/`MAX`. This value−rownum idea is the single most reused gaps-and-islands technique.

---

### P18. Longest streak of consecutive login days per user.

**Problem.** `Logins(UserId, LoginDate)` (one row per user per day). Find each user's longest streak of consecutive days.

```sql
WITH d AS ( -- distinct days per user
 SELECT DISTINCT UserId, LoginDate FROM Logins
),
grp AS (
 SELECT UserId, LoginDate,
 DATEADD(DAY,
 -ROW_NUMBER() OVER (PARTITION BY UserId ORDER BY LoginDate),
 LoginDate) AS island
 FROM d
)
SELECT UserId, MAX(streak) AS LongestStreak
FROM (
 SELECT UserId, island, COUNT(*) AS streak
 FROM grp
 GROUP BY UserId, island
) s
GROUP BY UserId;
```

**Why / approach.** Same islands trick, adapted to dates: subtract a *row number of days* from each date, so consecutive days map to the same anchor date (`island`). `PARTITION BY UserId` runs it per user. Then count rows per island (= streak length) and take the max per user. **Dedup first** (`DISTINCT`) so multiple logins on the same day don't break the row-number lockstep.

---

## Z6 — Self-Join Problems

### P19. Employees who earn more than their manager.

**Problem.** `Employees(Id, Name, Salary, ManagerId)` — list employees whose salary exceeds their manager's.

```sql
SELECT e.Name AS Employee, e.Salary AS EmpSalary,
 m.Name AS Manager, m.Salary AS MgrSalary
FROM Employees e
JOIN Employees m ON e.ManagerId = m.Id
WHERE e.Salary > m.Salary;
```

**Why / approach.** Self-join the table to itself: alias `e` = employee, `m` = manager, joined on `e.ManagerId = m.Id`, then compare salaries. Using an **inner join** naturally excludes the CEO (whose `ManagerId` is `NULL`), which is usually correct. Self-joins are in [sql-fundamentals-joins.md](sql-fundamentals-joins.md).

---

### P20. Find pairs of employees in the same department (no self-pairs, no duplicates).

**Problem.** List all distinct pairs of employees who share a department.

```sql
SELECT a.Name AS Emp1, b.Name AS Emp2, a.DeptId
FROM Employees a
JOIN Employees b ON a.DeptId = b.DeptId
 AND a.Id < b.Id;
```

**Why / approach.** The join condition `a.Id < b.Id` (strict less-than) does two jobs at once: it removes **self-pairs** (a row paired with itself, where `a.Id = b.Id`) and it removes **mirror duplicates** ((A,B) and (B,A) collapse to one). This `a.id < b.id` idiom is the canonical way to enumerate unordered pairs.

---

### P21. Show each employee alongside their manager's name (one-level hierarchy).

**Problem.** List every employee with their manager's name; employees with no manager should still appear.

```sql
SELECT e.Name AS Employee,
 ISNULL(m.Name, '(none)') AS Manager
FROM Employees e
LEFT JOIN Employees m ON e.ManagerId = m.Id;
```

**Why / approach.** **LEFT** join (not inner) so the top of the tree — whose `ManagerId` is `NULL` — is retained; the inner join in P19 would silently drop them. For a **multi-level** hierarchy (full org chart, any depth) you need recursion — P29 works the same problem both ways.

---

## Z7 — Date/Time Problems

### P22. Count orders per day.

**Problem.** `Orders(Id, OrderDateTime, Amount)` — number of orders per calendar day.

```sql
SELECT CAST(OrderDateTime AS DATE) AS OrderDay, COUNT(*) AS Orders
FROM Orders
GROUP BY CAST(OrderDateTime AS DATE)
ORDER BY OrderDay;
```

**Why / approach.** `CAST(… AS DATE)` strips the time so all timestamps in a day collapse to one group. **Do not** `GROUP BY` a function that isn't SARGable in a `WHERE` filter — grouping is fine, but if you also *filter* a date range, filter on the raw column (`WHERE OrderDateTime >= @from AND OrderDateTime < @to`) so an index can be used, rather than wrapping the column in `CAST`. Indexing/SARGability: [sql-indexing-plans.md](sql-indexing-plans.md).

**PostgreSQL:** `GROUP BY OrderDateTime::date` or `date_trunc('day', OrderDateTime)`.

---

### P23. Fill in missing dates (days with zero orders).

**Problem.** Report every day in a range with its order count, showing `0` for days that had none.

```sql
DECLARE @from DATE = '2026-01-01', @to DATE = '2026-01-31';

WITH cal AS (
 SELECT @from AS d
 UNION ALL
 SELECT DATEADD(DAY, 1, d) FROM cal WHERE d < @to
)
SELECT cal.d AS OrderDay, COUNT(o.Id) AS Orders
FROM cal
LEFT JOIN Orders o ON CAST(o.OrderDateTime AS DATE) = cal.d
GROUP BY cal.d
ORDER BY cal.d
OPTION (MAXRECURSION 0);
```

**Why / approach.** A `GROUP BY` on the orders alone can only produce days that *have* orders — missing days simply don't exist. Generate a **calendar** (recursive CTE here; a persisted `Calendar`/numbers table is better in production) and `LEFT JOIN` orders onto it. Use `COUNT(o.Id)` (counts non-NULL matches → 0 for empty days), **not** `COUNT(*)` (would count the calendar row → 1). This COUNT(column) vs COUNT(*) distinction is a favorite gotcha.

---

### P24. Generate a numbers/tally table on the fly.

**Problem.** You need N sequential integers without a physical table (foundation for calendars, gap-filling, splitting).

```sql
WITH nums AS (
 SELECT TOP (1000)
 ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS n
 FROM sys.all_objects a CROSS JOIN sys.all_objects b
)
SELECT n FROM nums;
```

**Why / approach.** Cross-joining a system catalog view against itself yields plenty of rows cheaply; `ROW_NUMBER()` numbers them and `TOP (N)` caps the count. This **set-based tally** beats a recursive CTE for large ranges (no per-row recursion, no `MAXRECURSION`). `ORDER BY (SELECT NULL)` = "any order".

**PostgreSQL:** far simpler — `SELECT n FROM generate_series(1, 1000) AS n;`.

---

### P25. Find overlapping date ranges.

**Problem.** `Bookings(Id, RoomId, StartDate, EndDate)` — find pairs of bookings for the same room whose date ranges overlap (double-booking detection).

```sql
SELECT a.Id AS BookingA, b.Id AS BookingB, a.RoomId
FROM Bookings a
JOIN Bookings b
 ON a.RoomId = b.RoomId
 AND a.Id < b.Id
 AND a.StartDate < b.EndDate
 AND b.StartDate < a.EndDate;
```

**Why / approach.** Two ranges overlap **iff** `A.start < B.end AND B.start < A.end` — the standard interval-overlap predicate. `a.Id < b.Id` avoids self-pairs and mirror duplicates (as in P20). Use `<` vs `<=` depending on whether adjacent, touching ranges (end == next start) count as overlapping — clarify with the interviewer; for `[start, end)` half-open intervals, `<` is correct.

---

### P26. Records active on a given date (temporal/SCD lookup).

**Problem.** `PriceHistory(ProductId, Price, ValidFrom, ValidTo)` — get the price in effect on a specific date.

```sql
DECLARE @asOf DATE = '2026-06-15';

SELECT ProductId, Price
FROM PriceHistory
WHERE @asOf >= ValidFrom
 AND @asOf < ValidTo; -- half-open [ValidFrom, ValidTo)
```

**Why / approach.** Use a **half-open interval** `ValidFrom <= @asOf < ValidTo` so consecutive versions (where one row's `ValidTo` equals the next's `ValidFrom`) never both match — `BETWEEN` (inclusive on both ends) would return two rows on the boundary date. If `ValidTo` is nullable for the current row, coalesce: `@asOf < ISNULL(ValidTo, '9999-12-31')`.

---

## Z8 — Pivot / Unpivot

### P27. Pivot monthly sales into columns.

**Problem.** `Sales(Region, SaleMonth, Amount)` with month numbers 1–12 → one row per region with a column per quarter/month.

```sql
-- Approach A (recommended, portable): CASE + aggregate
SELECT Region,
 SUM(CASE WHEN SaleMonth = 1 THEN Amount ELSE 0 END) AS Jan,
 SUM(CASE WHEN SaleMonth = 2 THEN Amount ELSE 0 END) AS Feb,
 SUM(CASE WHEN SaleMonth = 3 THEN Amount ELSE 0 END) AS Mar
FROM Sales
GROUP BY Region;

-- Approach B: PIVOT operator (SQL Server specific)
SELECT Region, [1] AS Jan, [2] AS Feb, [3] AS Mar
FROM (SELECT Region, SaleMonth, Amount FROM Sales) src
PIVOT (SUM(Amount) FOR SaleMonth IN ([1],[2],[3])) AS p;
```

**Why / approach.** **CASE + aggregate (A) is the recommended default**: portable across all engines, and flexible — you can mix aggregates (`SUM` for one column, `COUNT` for another), add computed expressions, and combine with other logic. **PIVOT (B)** is more compact but SQL-Server-only, one aggregate at a time, and requires you to hard-code the column list. For a *dynamic* column set (unknown months), both need dynamic SQL — build the `IN (...)`/`CASE` list from a query and `EXEC sp_executesql`.

**PostgreSQL:** no `PIVOT` operator — use CASE (A), or the cleaner `SUM(Amount) FILTER (WHERE SaleMonth = 1)`, or the `crosstab()` function from the `tablefunc` extension.

---

### P28. Unpivot columns back into rows.

**Problem.** `Wide(Region, Jan, Feb, Mar)` → tall `(Region, Month, Amount)`.

```sql
-- Approach A (recommended, portable): CROSS APPLY VALUES
SELECT w.Region, u.Month, u.Amount
FROM Wide w
CROSS APPLY (VALUES ('Jan', w.Jan), ('Feb', w.Feb), ('Mar', w.Mar)) u(Month, Amount);

-- Approach B: UNPIVOT operator
SELECT Region, Month, Amount
FROM Wide
UNPIVOT (Amount FOR Month IN (Jan, Feb, Mar)) AS u;
```

**Why / approach.** **CROSS APPLY (VALUES …) (A) is the modern, flexible choice** — it handles `NULL`s (UNPIVOT silently drops NULL cells), lets columns have different types via casting, and mixes with expressions. **UNPIVOT (B)** is terser but rigid (all unpivoted columns must share a type and it discards NULLs). `CROSS APPLY` is covered in [sql-fundamentals-joins.md](sql-fundamentals-joins.md).

**PostgreSQL:** no `UNPIVOT`; use `LATERAL (VALUES …)` (equivalent of CROSS APPLY) or `unnest`.

---

## Z9 — Hierarchy / Recursive

### P29. Find the hierarchy of employees and managers.

**Problem.** `Employees(Id, Name, ManagerId)` — show the reporting hierarchy: who reports to whom, and how deep each person sits in the org chart.

Sample data:

```text
Id | Name    | ManagerId
1  | Alice   | NULL       <- CEO
2  | Bob     | 1
3  | Carol   | 1
4  | David   | 2
5  | Emma    | 4
```

**Note the shape of the table.** It joins to *itself* — `ManagerId` points at another row's `Id` in the same table. That is what makes this either a self-join or a recursive query.

---

**Approach 1 — Self-join (one level)**

If you only need each employee beside their direct manager, a self-join does it:

```sql
SELECT e.Name AS Employee,
       ISNULL(m.Name, '(no manager)') AS Manager
FROM Employees e
LEFT JOIN Employees m ON e.ManagerId = m.Id;
```

```text
Employee | Manager
Alice    | (no manager)
Bob      | Alice
Carol    | Alice
David    | Bob
Emma     | David
```

Two aliases of the same table: `e` is the employee, `m` is the manager. Use a **`LEFT JOIN`** — Alice's `ManagerId` is `NULL`, and an inner join would silently drop the CEO.

**The limit:** one join gives one level. The manager's manager needs a second join, and a tree of unknown depth cannot be written this way at all.

```sql
-- Two levels needs another join, and you still can't handle "any depth"
LEFT JOIN Employees m2 ON m.ManagerId = m2.Id;
```

---

**Approach 2 — Recursive CTE (any depth)**

When you don't know how deep the tree goes, recursion walks the whole thing:

```sql
WITH tree AS (
    -- Anchor: start at the top
    SELECT Id, Name, ManagerId, 0 AS Depth,
           CAST(Name AS VARCHAR(1000)) AS Path
    FROM Employees
    WHERE ManagerId IS NULL

    UNION ALL

    -- Recursive: everyone reporting to a row already found
    SELECT e.Id, e.Name, e.ManagerId, t.Depth + 1,
           CAST(t.Path + ' > ' + e.Name AS VARCHAR(1000))
    FROM Employees e
    JOIN tree t ON e.ManagerId = t.Id
)
SELECT Depth, Name, Path
FROM tree
ORDER BY Path
OPTION (MAXRECURSION 0);
```

```text
Depth | Name  | Path
0     | Alice | Alice
1     | Bob   | Alice > Bob
2     | David | Alice > Bob > David
3     | Emma  | Alice > Bob > David > Emma
1     | Carol | Alice > Carol
```

**How it runs.** The anchor picks the CEO. The recursive member joins `Employees` back to the rows the previous pass produced — pass 1 finds Bob and Carol, pass 2 finds David, pass 3 finds Emma. It stops when a pass returns nothing. `Depth` counts the level; `Path` builds up the chain of names.

---

**Which one to use**

| | Self-join | Recursive CTE |
|---|---|---|
| Levels covered | One per join | Any depth |
| Need to know depth upfront | Yes | No |
| Typical use | "employee and their manager" | full org chart, all reports |

**Two things to watch on the recursive version:**

- **`CAST(... AS VARCHAR(1000))` in the anchor is required.** Without it the anchor's column is sized to the first name, and the recursive part fails with a type-length mismatch.
- **`MAXRECURSION` defaults to 100 levels**, then errors. `OPTION (MAXRECURSION 0)` removes the cap — but if the data has a cycle (A manages B, B manages A) the query never ends, so keep a limit unless you trust the data.

**Variations.** To walk *up* to the CEO instead of down, flip the join to `e.Id = t.ManagerId` and anchor at the employee. For one subtree only, anchor at `WHERE Id = @root`.

Recursive CTEs are covered in [sql-advanced-queries.md](sql-advanced-queries.md).

**PostgreSQL:** `WITH RECURSIVE tree AS (...)` — the `RECURSIVE` keyword is required; no `MAXRECURSION` (use `CYCLE` clause or a depth guard).

---

### P30. Build a full path / breadcrumb string.

**Problem.** For a category tree `Categories(Id, Name, ParentId)`, output each category's full path like `Electronics > Phones > Android`.

```sql
WITH tree AS (
 SELECT Id, Name, ParentId,
 CAST(Name AS VARCHAR(1000)) AS Path
 FROM Categories
 WHERE ParentId IS NULL -- roots
 UNION ALL
 SELECT c.Id, c.Name, c.ParentId,
 CAST(t.Path + ' > ' + c.Name AS VARCHAR(1000))
 FROM Categories c
 JOIN tree t ON c.ParentId = t.Id
)
SELECT Id, Name, Path
FROM tree
ORDER BY Path
OPTION (MAXRECURSION 0);
```

**Why / approach.** The same recursive walk as P29 — only the table differs (a category tree instead of employees). The `Path` column is built the same way: concatenate the parent's path, a separator, and the current name at each level. `ORDER BY Path` then gives a natural tree order, with children sorted under their parent. The `CAST(… AS VARCHAR(1000))` in the anchor is required for the same reason as in P29.

---

## Z10 — Anti-Join / Find-Missing

### P31. Customers with no orders.

**Problem.** `Customers(Id, Name)`, `Orders(Id, CustomerId)` — list customers who have never placed an order.

```sql
-- Approach A (recommended): NOT EXISTS
SELECT c.Id, c.Name
FROM Customers c
WHERE NOT EXISTS (
 SELECT 1 FROM Orders o WHERE o.CustomerId = c.Id
);

-- Approach B: LEFT JOIN ... IS NULL (anti-join)
SELECT c.Id, c.Name
FROM Customers c
LEFT JOIN Orders o ON o.CustomerId = c.Id
WHERE o.CustomerId IS NULL;

-- Approach C: NOT IN (DANGER with NULLs — see below)
SELECT c.Id, c.Name
FROM Customers c
WHERE c.Id NOT IN (SELECT CustomerId FROM Orders);
```

**Why / approach.** **Prefer `NOT EXISTS` (A).** It's NULL-safe and typically produces an efficient anti-semi-join plan.
- **`LEFT JOIN … IS NULL` (B)** is equally correct and sometimes chosen for readability; ensure the `IS NULL` test is on a non-nullable/right-side key.
- **`NOT IN` (C) is the trap:** if *any* row in the subquery has `CustomerId = NULL`, `NOT IN` returns **no rows at all** (because `x <> NULL` is `UNKNOWN`). Interviewers love this. Only safe if the subquery column is guaranteed non-nullable.

EXISTS/joins: [sql-fundamentals-joins.md](sql-fundamentals-joins.md).

---

### P32. Rows in table A not in table B (set difference).

**Problem.** Find `ProductId`s present in `CatalogA` but not in `CatalogB`.

```sql
-- Approach A: EXCEPT (set-based, dedups, NULL-safe)
SELECT ProductId FROM CatalogA
EXCEPT
SELECT ProductId FROM CatalogB;

-- Approach B: NOT EXISTS (keeps duplicates from A, more flexible)
SELECT a.ProductId
FROM CatalogA a
WHERE NOT EXISTS (SELECT 1 FROM CatalogB b WHERE b.ProductId = a.ProductId);
```

**Why / approach.** **`EXCEPT` (A)** is the cleanest when comparing whole rows / column lists: it treats `NULL`s as equal (so it's NULL-safe) and **removes duplicates** (returns a distinct set). **`NOT EXISTS` (B)** is better when you need to (a) keep duplicate rows from A, (b) return columns *not* used in the comparison, or (c) compare on a subset of columns. Rule of thumb: `EXCEPT` for "distinct set difference on these exact columns", `NOT EXISTS` for row-level filtering with extra columns.

**PostgreSQL:** `EXCEPT` is identical; `EXCEPT ALL` keeps duplicates.

---

### P33. Find missing IDs in a sequence (gaps in an identity column).

**Problem.** `Invoices(Id)` should be contiguous but has gaps. List the missing IDs between `MIN` and `MAX`.

```sql
WITH nums AS (
 SELECT TOP ((SELECT MAX(Id) FROM Invoices))
 ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS n
 FROM sys.all_objects a CROSS JOIN sys.all_objects b
)
SELECT n AS MissingId
FROM nums
WHERE n >= (SELECT MIN(Id) FROM Invoices)
 AND NOT EXISTS (SELECT 1 FROM Invoices i WHERE i.Id = nums.n);
```

**Why / approach.** Same anti-join idea as P16 but sourced from a **tally table** (P24) rather than recursion — better for large ID ranges. Generate all integers in range, then `NOT EXISTS` against the actual IDs. If you only need where gaps *start*, the lightweight version: `SELECT Id + 1 FROM Invoices i WHERE NOT EXISTS (SELECT 1 FROM Invoices n WHERE n.Id = i.Id + 1)` (each existing row whose successor is missing).

---

## Z11 — Median & Percentiles

### P34. Compute the median salary.

**Problem.** Return the median salary (average of the two middle values for an even count).

```sql
-- Approach A (recommended, SQL Server 2012+): PERCENTILE_CONT
SELECT DISTINCT
 PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY Salary) OVER () AS MedianSalary
FROM Employees;

-- Approach B (portable / older versions): ROW_NUMBER + COUNT
WITH ordered AS (
 SELECT Salary,
 ROW_NUMBER() OVER (ORDER BY Salary) AS rn,
 COUNT(*) OVER () AS cnt
 FROM Employees
)
SELECT AVG(CAST(Salary AS DECIMAL(18,2))) AS MedianSalary
FROM ordered
WHERE rn IN ((cnt + 1) / 2, (cnt + 2) / 2);
```

**Why / approach.**
- **`PERCENTILE_CONT(0.5)` (A)** interpolates and correctly averages the two middle values for even counts. In SQL Server it's a **window function** — it must be written with `WITHIN GROUP (…) OVER (…)` and returns a value *per row*, hence the `DISTINCT` (or wrap in a subquery) to collapse to one. Add `OVER (PARTITION BY DeptId)` for a per-department median.
- **Approach B** works on any version: order rows, then pick the middle one (odd count) or the two middle ones (even count) via the `rn IN ((cnt+1)/2, (cnt+2)/2)` trick — for odd `cnt` both expressions equal the same row; for even `cnt` they select the two central rows, and `AVG` averages them. `PERCENTILE_DISC(0.5)` (no interpolation) picks an actual existing value instead.

**PostgreSQL:** `PERCENTILE_CONT` is an **ordered-set aggregate** used *without* `OVER`: `SELECT percentile_cont(0.5) WITHIN GROUP (ORDER BY salary) FROM employees;` — cleaner than SQL Server (no `DISTINCT` needed).

---

### P35. Compute quartiles / arbitrary percentiles.

**Problem.** Return the 25th, 50th, and 75th percentile of salary.

```sql
SELECT DISTINCT
 PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY Salary) OVER () AS P25,
 PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY Salary) OVER () AS P50,
 PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY Salary) OVER () AS P75
FROM Employees;
```

**Why / approach.** Same as P34 with different fractions. For assigning each row to a quartile *bucket* instead of computing the cut points, use `NTILE(4) OVER (ORDER BY Salary)` — it splits rows into 4 roughly equal groups (useful for "top quartile of customers"). Note `NTILE` divides by *row count*, not by value ranges, so groups differ from percentile thresholds.

**PostgreSQL:** `percentile_cont(ARRAY[0.25,0.5,0.75]) WITHIN GROUP (ORDER BY salary)` returns all three in one array — very concise.

---

### Cheat sheet (last-minute — whole section)

| Problem type | Go-to technique | Key snippet |
|---|---|---|
| Nth-highest (distinct) | `DENSE_RANK` filtered to N, wrap in `MAX` for NULL-on-empty | `DENSE_RANK() OVER (ORDER BY x DESC)` |
| 2nd-highest, shortest | `MAX(x) WHERE x < (SELECT MAX(x))` | naturally NULL-safe |
| Max per group | `RANK()=1` (all ties) or `ROW_NUMBER()=1` (one) | `PARTITION BY grp ORDER BY x DESC` |
| Find duplicates | `GROUP BY key HAVING COUNT(*) > 1` | `COUNT(DISTINCT c)>1` for conflicting values |
| Delete dupes keep one | `ROW_NUMBER` in CTE, `DELETE … WHERE rn>1` | `PARTITION BY dupcols ORDER BY id` |
| Latest / top-N per group | `ROW_NUMBER … WHERE rn<=N` **or** `CROSS APPLY (TOP N …)` | APPLY wins with big groups + index |
| Running total | `SUM() OVER (ORDER BY … ROWS UNBOUNDED PRECEDING)` | always specify `ROWS` |
| Running total per day | `GROUP BY date` + `SUM(SUM(x)) OVER (…)` | window runs after `GROUP BY` |
| Moving average | `AVG() OVER (… ROWS BETWEEN n PRECEDING AND CURRENT ROW)` | n rows, not n days |
| Period-over-period | `LAG()/LEAD() OVER (ORDER BY period)` | `NULLIF(prev,0)` for growth % |
| Detail + group aggregate | windowed agg with `PARTITION BY`, no `ORDER BY` | `AVG(x) OVER (PARTITION BY g)` |
| Grand total on each row | `SUM(x) OVER ()` | empty `OVER()` = whole set |
| Missing numbers (gaps) | tally/recursive range `LEFT JOIN … IS NULL` | or `NOT EXISTS` |
| Consecutive runs (islands) | `value − ROW_NUMBER()` = group key | dates: `DATEADD(DAY,-rn,date)` |
| Longest streak | islands trick + `COUNT` per island, `MAX` | dedup days first |
| Earn > manager | self-join `e.ManagerId=m.Id`, compare | inner join drops CEO |
| Unordered pairs | self-join `a.Id < b.Id` | kills self + mirror pairs |
| Overlapping ranges | `a.start < b.end AND b.start < a.end` | `<` for half-open intervals |
| Active on date (SCD) | `@d >= ValidFrom AND @d < ValidTo` | half-open avoids double-match |
| Count per day + zeros | calendar table `LEFT JOIN`, `COUNT(o.Id)` | not `COUNT(*)` |
| Pivot | `SUM(CASE WHEN … THEN x END)` (portable) or `PIVOT` | CASE more flexible |
| Unpivot | `CROSS APPLY (VALUES …)` (portable) or `UNPIVOT` | APPLY keeps NULLs |
| Subtree / descendants | recursive CTE, anchor + `UNION ALL` recursive | `OPTION (MAXRECURSION 0)` |
| Path / breadcrumb | recursive CTE carrying concatenated string | `CAST(... AS VARCHAR(n))` in anchor |
| No orders / anti-join | `NOT EXISTS` (NULL-safe) | avoid `NOT IN` with nullable col |
| Set difference | `EXCEPT` (distinct) or `NOT EXISTS` (keeps dups) | EXCEPT is NULL-safe |
| Median | `PERCENTILE_CONT(0.5) WITHIN GROUP (…) OVER ()` + `DISTINCT` | ROW_NUMBER/COUNT for legacy |
| Percentiles / buckets | `PERCENTILE_CONT(p)` (cut points) / `NTILE(k)` (buckets) | NTILE splits by row count |

**Ranking-function ties recap:** `ROW_NUMBER` = unique (pagination, pick-one); `RANK` = ties share + gap after (leaderboards, "all tied at top"); `DENSE_RANK` = ties share, no gap ("Nth distinct value").

**Top NULL/tie traps interviewers probe:** `NOT IN` + NULL → empty result; `COUNT(*)` vs `COUNT(col)` after LEFT JOIN; integer division (`100/3`); `BETWEEN` on adjacent temporal ranges (double-match); omitting `ROWS` → `RANGE` peer-summing bug; `ROW_NUMBER` vs `DENSE_RANK` for Nth-highest with duplicate salaries.
