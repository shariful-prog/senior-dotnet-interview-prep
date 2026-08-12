# SQL Top 20 Interview Questions (Quick Answers)

> Goal: Fast revision. Each question opens with a one-line definition, then the detail — plus a link to the exact section in the detailed SQL notes.

## Fundamentals & Joins

### 1. What is the logical query processing order?

**Definition:** The order SQL **logically** evaluates a query, which is not the order you write it: `FROM` → `WHERE` → `GROUP BY` → `HAVING` → `SELECT` → `ORDER BY`.

**Detail:** This one fact explains a lot of confusing errors. `SELECT` runs almost last, so a column alias you define there isn't available in `WHERE` — but it *is* available in `ORDER BY`, which runs after. It's also why `WHERE` can't see an aggregate: grouping hasn't happened yet.

:material-file-document-outline: **Deep dive:** [T1 — Logical Query Processing](../Sql/sql-fundamentals-joins.md#t1--select--where--group-by--having--logical-query-processing)

---

### 2. What are the join types?

**Definition:** **`INNER`** keeps only matching rows. **`LEFT`** keeps all rows from the left table. **`RIGHT`** keeps all from the right. **`FULL`** keeps everything from both. **`CROSS`** pairs every row with every row.

**Detail:** With an outer join, columns from the non-matching side come back as `NULL`. Two named variations worth knowing: a **self-join** joins a table to itself (employee → manager), and an **anti-join** finds rows with *no* match on the other side.

:material-file-document-outline: **Deep dive:** [T2 — JOINs](../Sql/sql-fundamentals-joins.md#t2--joins-inner--left--right--full--cross-self-anti-core)

---

### 3. Why does a `WHERE` clause turn a `LEFT JOIN` into an `INNER JOIN`?

**Definition:** Because `WHERE` runs **after** the join, and filtering on a column from the right-hand table discards the `NULL` rows the outer join just produced.

**Detail:** The join correctly returns customers with no orders, filling their order columns with `NULL`. Then `WHERE o.Status = 'Shipped'` evaluates to unknown for those rows and drops them — so your outer join quietly becomes an inner one. Fix: move the condition into the `ON` clause, or explicitly allow `NULL` with `OR o.Status IS NULL`. This is the most common "my report lost rows" bug.

:material-file-document-outline: **Deep dive:** [T2 — JOINs](../Sql/sql-fundamentals-joins.md#t2--joins-inner--left--right--full--cross-self-anti-core) · [S1 — Wrong Results](../Sql/sql-scenarios.md#s1--wrong-results)

---

### 4. What is `NULL`, and what is three-valued logic?

**Definition:** **`NULL`** means *unknown* — not zero, not an empty string. **Three-valued logic** means every comparison returns true, false, or **unknown**.

**Detail:** Anything compared to `NULL` is unknown, which is why `= NULL` never matches and you need `IS NULL`. Aggregates silently skip `NULL`s, so `COUNT(col)` and `AVG(col)` can disagree with what people expect. And `NOT IN` against a list containing a `NULL` returns **no rows at all** — use `NOT EXISTS` instead.

:material-file-document-outline: **Deep dive:** [T3 — NULL Handling](../Sql/sql-fundamentals-joins.md#t3--null-handling--three-valued-logic)

---

### 5. What is the difference between `WHERE` and `HAVING`?

**Definition:** **`WHERE`** filters individual rows *before* grouping. **`HAVING`** filters groups *after* aggregation.

**Detail:** So `HAVING` is where you put conditions on aggregates like `COUNT(*) > 5`. Always filter in `WHERE` when you can — it removes rows before the expensive grouping work, rather than after.

:material-file-document-outline: **Deep dive:** [T1 — Logical Query Processing](../Sql/sql-fundamentals-joins.md#t1--select--where--group-by--having--logical-query-processing)

---

### 6. What is the difference between `UNION` and `UNION ALL`?

**Definition:** **`UNION`** combines two result sets and removes duplicates. **`UNION ALL`** combines them and keeps everything.

**Detail:** Removing duplicates requires a sort, so `UNION ALL` is meaningfully faster — use it whenever you know the rows are already distinct. The related operators: **`EXCEPT`** returns rows in the first set but not the second, **`INTERSECT`** returns rows in both.

:material-file-document-outline: **Deep dive:** [T5 — Set Operators](../Sql/sql-fundamentals-joins.md#t5--set-operators-union--union-all--except--intersect)

---

### 7. When do you use `EXISTS` vs `IN` vs `JOIN`?

**Definition:** **`EXISTS`** tests whether any matching row exists. **`IN`** tests membership in a list. **`JOIN`** combines rows from both tables.

**Detail:** `EXISTS` stops at the first match and handles `NULL`s safely. `IN` is fine for short literal lists but breaks badly when negated against `NULL`s. A `JOIN` can **multiply** your rows if the other table has duplicates — so for a pure "does it exist" check, use `EXISTS`, and always prefer `NOT EXISTS` over `NOT IN`.

:material-file-document-outline: **Deep dive:** [T6 — EXISTS vs IN vs JOIN](../Sql/sql-fundamentals-joins.md#t6--exists-vs-in-vs-join)

## Advanced Queries

### 8. What is a window function?

**Definition:** A **window function** calculates a value across a set of related rows while **keeping every row** in the output.

**Detail:** That's the difference from `GROUP BY`, which collapses rows into one per group. So you can show each employee next to their department's average salary in a single pass — no self-join, no subquery. The `OVER()` clause defines the window with `PARTITION BY` and `ORDER BY`.

:material-file-document-outline: **Deep dive:** [V1 — Window Functions](../Sql/sql-advanced-queries.md#v1--window-functions)

---

### 9. What is the difference between `ROW_NUMBER`, `RANK`, and `DENSE_RANK`?

**Definition:** They differ only in how they handle **ties**. `ROW_NUMBER` always gives distinct numbers. `RANK` gives ties the same number then **skips** (1, 1, 3). `DENSE_RANK` gives ties the same number with **no gap** (1, 1, 2).

**Detail:** `ROW_NUMBER` is the workhorse — combined with `PARTITION BY` it answers "the Nth highest" and "the latest row per group". Choose `RANK`/`DENSE_RANK` when ties genuinely should share a position.

:material-file-document-outline: **Deep dive:** [V1 — Window Functions](../Sql/sql-advanced-queries.md#v1--window-functions) · [Z1 — Ranking & Nth-Highest](../Sql/sql-query-problems.md#z1--ranking--nth-highest)

---

### 10. What is a CTE, and what makes a recursive one different?

**Definition:** A **CTE** (`WITH ... AS`) is a named temporary result set that exists for a single statement. A **recursive CTE** refers to itself to walk through hierarchical data.

**Detail:** Regular CTEs mainly buy readability — you can break a complex query into named steps and reference each more than once. A recursive CTE has an *anchor* member and a *recursive* member that joins back to the CTE, stopping when nothing new comes back. It's the standard tool for org charts, folder trees, and generating number sequences.

:material-file-document-outline: **Deep dive:** [V3 — CTEs](../Sql/sql-advanced-queries.md#v3--ctes-with--readability) · [V4 — Recursive CTEs](../Sql/sql-advanced-queries.md#v4--recursive-ctes)

---

### 11. What is a correlated subquery?

**Definition:** A **correlated subquery** references a column from the outer query, so it conceptually runs once **per outer row**.

**Detail:** That's fine for small result sets and slow for large ones. A non-correlated subquery, by contrast, runs once and its result is reused. Correlated subqueries can often be rewritten as a join, a window function, or `CROSS APPLY` — letting the optimizer do the work in one pass instead.

:material-file-document-outline: **Deep dive:** [V5 — Subqueries](../Sql/sql-advanced-queries.md#v5--subqueries-correlated-vs-non-correlated) · [V6 — APPLY](../Sql/sql-advanced-queries.md#v6--apply-cross-apply--outer-apply)

---

### 12. How do you find the second-highest salary, or the latest row per group?

**Definition:** Both are **ranking problems** — you number the rows in the order you care about, then filter on the number.

**Detail:** Second-highest: `DENSE_RANK()` in a CTE filtered to `= 2`, or `OFFSET 1 ROWS FETCH NEXT 1 ROW ONLY`. Latest per group: `ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC)` filtered to `= 1`. The thing to clarify out loud: should ties count as one position or two? That decides `ROW_NUMBER` vs `DENSE_RANK`.

:material-file-document-outline: **Deep dive:** [Z1 — Ranking & Nth-Highest](../Sql/sql-query-problems.md#z1--ranking--nth-highest) · [Z3 — Top-N Per Group](../Sql/sql-query-problems.md#z3--top-n-per-group--latest-row-per-group)

## Indexing & Performance

### 13. What is the difference between a clustered and a non-clustered index?

**Definition:** The **clustered index** *is* the table — it defines the physical order of the rows, so there can only be one. A **non-clustered index** is a separate structure holding the key plus a pointer back to the row.

**Detail:** A table with no clustered index is called a **heap**. A good clustered key is narrow, unique, and ever-increasing — which is why a random GUID is a poor choice: new rows land in the middle, causing page splits and fragmentation.

:material-file-document-outline: **Deep dive:** [U1 — Clustered vs Non-Clustered](../Sql/sql-indexing-plans.md#u1--clustered-vs-non-clustered-indexes)

---

### 14. What is a covering index, and what is a key lookup?

**Definition:** A **key lookup** is the extra trip back to the table when an index doesn't have every column the query needs. A **covering index** contains them all, so no lookup happens.

**Detail:** That extra trip happens once *per row*, which is why it destroys performance at scale. You build a covering index by putting the filter and join columns in the key, and the merely-returned columns in `INCLUDE` — they're stored at the leaf level without bloating the key.

:material-file-document-outline: **Deep dive:** [U2 — Covering Indexes & INCLUDE](../Sql/sql-indexing-plans.md#u2--covering-indexes--include)

---

### 15. What is SARGability?

**Definition:** A predicate is **SARGable** if the database can use an index seek for it. Wrap the column in anything and it stops being SARGable.

**Detail:** The three classic killers: applying a function to the column (`YEAR(OrderDate) = 2024`), a leading wildcard (`LIKE '%smith'`), and implicit type conversion (comparing a `varchar` column to an `nvarchar` parameter). Rewrite so the column stands alone: `OrderDate >= '2024-01-01' AND OrderDate < '2025-01-01'`.

:material-file-document-outline: **Deep dive:** [U4 — Composite Indexes & SARGability](../Sql/sql-indexing-plans.md#u4--composite-indexes--sargability)

---

### 16. Why does a query suddenly get slow with nothing changed?

**Definition:** Almost always **stale statistics** or **parameter sniffing** — the optimizer is working from a bad estimate of how many rows it will get.

**Detail:** Statistics describe data distribution; when they drift, the optimizer picks the wrong plan. Parameter sniffing means the cached plan was built for one parameter value and is terrible for another. Also check data growth past a tipping point, index fragmentation, and blocking. Diagnose with the actual execution plan and `SET STATISTICS IO ON` — trust logical reads over elapsed time.

:material-file-document-outline: **Deep dive:** [U5 — Statistics](../Sql/sql-indexing-plans.md#u5--statistics) · [Y4 — Parameter Sniffing](../Sql/sql-nosql-performance.md#y4--parameterization-plan-cache--parameter-sniffing)

---

### 17. Why not just index everything?

**Definition:** Because every index is a **copy of data that must be kept up to date** on every insert, update, and delete.

**Detail:** Indexes speed up reads and slow down writes. They also consume disk and buffer-pool memory, and give the optimizer more options to get wrong. Index for the queries you actually run, and drop the ones nothing uses.

:material-file-document-outline: **Deep dive:** [U6 — Filtered Indexes & Index Downsides](../Sql/sql-indexing-plans.md#u6--filtered-indexes--index-downsides)

## Transactions & Design

### 18. What does ACID mean, and what are the isolation levels?

**Definition:** **Atomicity** (all or nothing), **Consistency** (constraints always hold), **Isolation** (concurrent transactions don't corrupt each other), **Durability** (once committed, it survives a crash).

**Detail:** Isolation is the one with dials. From least to most isolated: `READ UNCOMMITTED` (allows dirty reads), `READ COMMITTED` (the default), `REPEATABLE READ` (blocks non-repeatable reads), `SERIALIZABLE` (blocks phantoms too). Each step up trades concurrency for safety.

:material-file-document-outline: **Deep dive:** [W1 — ACID Properties](../Sql/sql-transactions-locking.md#w1--acid-properties) · [W2 — Isolation Levels](../Sql/sql-transactions-locking.md#w2--isolation-levels-core)

---

### 19. What is a deadlock, and how does it differ from blocking?

**Definition:** **Blocking** is one transaction waiting for another to release a lock — it resolves itself. A **deadlock** is a cycle where each transaction holds what the other needs, so neither can ever proceed.

**Detail:** SQL Server detects the cycle and kills the cheaper transaction as the **deadlock victim**. Reduce them by accessing tables in a consistent order everywhere, keeping transactions short, and indexing well so you lock fewer rows. Your application should catch error 1205 and retry.

:material-file-document-outline: **Deep dive:** [W5 — Deadlocks](../Sql/sql-transactions-locking.md#w5--deadlocks-core) · [W6 — Blocking vs Deadlock](../Sql/sql-transactions-locking.md#w6--blocking-vs-deadlock-optimistic-vs-pessimistic)

---

### 20. What is normalization, and when would you denormalize?

**Definition:** **Normalization** is organizing tables so each fact is stored in exactly one place. **Denormalization** is deliberately duplicating data to make reads faster.

**Detail:** 1NF: atomic column values. 2NF: no column depending on only part of a composite key. 3NF: no column depending on another non-key column. The goal is that an update can never leave two copies disagreeing. Denormalize when reads dominate — reporting, analytics — and accept that you now own the job of keeping the copies in sync.

:material-file-document-outline: **Deep dive:** [X1 — Normalization](../Sql/sql-schema-design.md#x1--normalization) · [X2 — Denormalization & Trade-offs](../Sql/sql-schema-design.md#x2--denormalization--trade-offs)

---

## Runners-Up (ask-me-next round)

- **`COUNT(*)` vs `COUNT(col)` vs `COUNT(DISTINCT col)`; `DISTINCT` vs `GROUP BY`** — [T4](../Sql/sql-fundamentals-joins.md#t4--aggregates--distinct)
- **`ISNULL` vs `COALESCE` vs `NULLIF`** — [T3](../Sql/sql-fundamentals-joins.md#t3--null-handling--three-valued-logic)
- **`BETWEEN` and the classic date bug; `TOP`, `LIKE`, `CASE`** — [T7](../Sql/sql-fundamentals-joins.md#t7--everyday-t-sql-sorting-top-like-case-between-aliases)
- **Running totals, moving averages & `LAG`/`LEAD`** — [V2](../Sql/sql-advanced-queries.md#v2--window-aggregates-running-totals-frames-laglead) · [Z4](../Sql/sql-query-problems.md#z4--running-totals-moving-averages--period-over-period)
- **Pagination — and why page 500 crawls** — [V7](../Sql/sql-advanced-queries.md#v7--pagination--common-patterns) · [S3](../Sql/sql-scenarios.md#s3--performance)
- **Finding & deleting duplicate rows** — [Z2](../Sql/sql-query-problems.md#z2--duplicates)
- **Gaps & islands; anti-joins; median & percentiles** — [Z5](../Sql/sql-query-problems.md#z5--gaps--islands) · [Z10](../Sql/sql-query-problems.md#z10--anti-join--find-missing) · [Z11](../Sql/sql-query-problems.md#z11--median--percentiles)
- **Execution plans: seek vs scan, join operators** — [U3](../Sql/sql-indexing-plans.md#u3--execution-plans)
- **Primary vs foreign vs surrogate keys; constraints & FK actions** — [X3](../Sql/sql-schema-design.md#x3--keys) · [X4](../Sql/sql-schema-design.md#x4--constraints--fk-actions)
- **Relationships & junction tables; data type selection** — [X5](../Sql/sql-schema-design.md#x5--relationships--junction-tables) · [X6](../Sql/sql-schema-design.md#x6--data-type-selection)
- **Views, stored procedures, functions & triggers** — [X8](../Sql/sql-schema-design.md#x8--programmability-views-procedures-functions-triggers)
- **OLTP vs OLAP** — [X0](../Sql/sql-schema-design.md#x0--workload-first-oltp-vs-olap)
- **`SELECT *` and the N+1 query problem** — [Y3](../Sql/sql-nosql-performance.md#y3--common-sql-anti-patterns)
- **SQL vs NoSQL, and the CAP theorem** — [Y1](../Sql/sql-nosql-performance.md#y1--sql-vs-nosql)
- **Batching & bulk operations; partitioning vs sharding** — [Y5](../Sql/sql-nosql-performance.md#y5--batching--bulk-operations) · [Y6](../Sql/sql-nosql-performance.md#y6--scaling-databases-overview)
- **Transaction control: TRY…CATCH, `XACT_ABORT`, nested transactions** — [W7](../Sql/sql-transactions-locking.md#w7--transaction-control)
- **Lock modes & granularity** — [W4](../Sql/sql-transactions-locking.md#w4--locking-core)

## Scenario Practice

Interviewers often ask these as stories rather than definitions — "the report lost rows", "revenue doubled", "one user's edit vanished", "page 1 is instant but page 500 crawls".
**8 worked scenarios:** [SQL Scenario Questions](../Sql/sql-scenarios.md)
**35 worked query problems:** [Common Interview Query Problems](../Sql/sql-query-problems.md)
