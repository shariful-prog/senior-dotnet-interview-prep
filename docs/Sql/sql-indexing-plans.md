# U. Indexing & Query Plans
---
## U1 — Clustered vs Non-Clustered Indexes

### Q1. What is an index, and what structure does SQL Server use?

**An index is a separate sorted structure that lets the database find rows without reading the whole table** — like the index at the back of a book.

**Structure — B-tree:**

- A root page at the top, a few middle levels, and the data at the bottom (the **leaf level**).
- Every page above the leaf holds values plus pointers to the level below, so a lookup goes root → middle → leaf.
- Values are kept **sorted** — this is what allows a **seek** (jump straight to a value) instead of a **scan** (check every row).

**Why it matters so much:** a table with a few million rows is only **3–4 levels deep**. A seek costs about 4 page reads whether the table holds 10,000 rows or 10 million.

**Two kinds** — the only difference is what sits at the leaf level:

| Type | Leaf level holds |
|---|---|
| **Clustered** | the table's actual data |
| **Non-clustered** | indexed values + a pointer to the real row |

### Q2. What is a clustered index, and what is a heap?

**A clustered index physically sorts the table, and the table lives inside it** — the leaf level of the clustered index *is* your table. There is no separate copy of the data.

**Consequences:**

- **One per table.** You can't physically sort the same rows two ways.
- **Fastest possible lookup by that key** — navigate down the tree and the full row is right there, no second trip.
- **`PRIMARY KEY` creates it automatically** in SQL Server, unless you write `PRIMARY KEY NONCLUSTERED`.
- **Range queries are fast** — matching rows sit physically next to each other.

```sql
CREATE TABLE dbo.Orders (
    OrderId    INT IDENTITY(1,1) PRIMARY KEY,  -- clustered by default
    CustomerId INT NOT NULL,
    OrderDate  DATETIME2 NOT NULL,
    Total      DECIMAL(10,2) NOT NULL
);
```

**A heap is a table with no clustered index.** Rows go wherever there's free space, in no order. Each row is addressed by its physical location (**RID** = file + page + slot).

**Problems with heaps:**

- No way to seek or scan in order without some other index.
- **Forwarding pointers** — if an update makes a row too big for its page, the row moves and leaves a pointer behind. Reads must chase it; the cost accumulates.
- Any query without a useful index becomes a full table scan.

**Valid uses:** bulk-load staging tables, write-once tables read through covering indexes.

**Takeaway:** give nearly every table a clustered index.

### Q3. What is a non-clustered index, and what's at its leaf level?

**A separate B-tree from the table**, sorted by the columns you indexed. Each leaf entry holds **the indexed values plus a pointer back to the real row**.

**What the pointer is:**

| Table type | Pointer stored at leaf |
|---|---|
| Has a clustered index | the **clustered key** |
| Heap | the **RID** |

This is why the clustered key is silently appended to every non-clustered index — and why a wide clustered key makes *all* your indexes wide.

```sql
CREATE NONCLUSTERED INDEX IX_Orders_CustomerId
    ON dbo.Orders(CustomerId);
```

This index is sorted by `CustomerId`; each leaf entry is really `(CustomerId, OrderId)`.

**The consequence to understand:**

```sql
SELECT CustomerId, Total FROM dbo.Orders WHERE CustomerId = 42;
```

The index finds the rows, but `Total` isn't in it — so SQL Server goes **back to the table** for every row found. That return trip is a **key lookup** (Q8).

**Limit:** up to 999 per table. You'd never want close to that (Q14).

### Q4. What makes a good clustered key? Why is a random GUID bad?

A good clustered key is **narrow, unique, unchanging, and always increasing**.

| Requirement | Why |
|---|---|
| **Narrow** | It's copied into every non-clustered index — 4 bytes keeps them all small |
| **Unique** | If not, SQL Server adds a hidden 4-byte **uniquifier** to every row |
| **Never changes** | Changing it moves the row *and* every index pointer to it |
| **Always increasing** | New rows append at the end of the tree instead of squeezing into the middle |

**`INT`/`BIGINT IDENTITY` ticks all four** — hence the standard recommendation.

**A random GUID (`NEWID()`) breaks two of them: it's 16 bytes and it's random.**

Because the values are random, new rows land all over the tree instead of at the end. When the target page is already full, SQL Server does a **page split** — new page, half the rows moved, pointers fixed. That gives you three costs: pages left half empty and out of order (**fragmentation**, so scans read more pages), extra logging and I/O on every split, and 16 bytes instead of 4 copied into **every** non-clustered index.

**Fix:** cluster on `IDENTITY` and keep the GUID as a non-clustered unique key — or use `NEWSEQUENTIALID()`, whose values increase so inserts append.

**Takeaway:** a GUID is fine as a **logical/primary key** or a **non-clustered** key. The damage is specific to making it the **clustered** key.

---

## U2 — Covering Indexes & INCLUDE

### Q5. What is a covering index, and what does INCLUDE do?

**A covering index contains every column one query needs** — everything in `SELECT`, `WHERE`, `JOIN`, `ORDER BY`, `GROUP BY` — so the query is answered entirely from the index with no trip back to the table.

**"Covering" is relative to a query, not a property of the index.** The same index can cover query A and not query B.

**`INCLUDE` is how you build one.** It adds columns stored **only at the leaf level**, outside the sorted part of the tree:

```sql
CREATE NONCLUSTERED INDEX IX_Orders_CustomerId_Incl
    ON dbo.Orders (CustomerId)          -- key column: sorted, searchable
    INCLUDE (OrderDate, Total);         -- payload: available, not searchable
```

**Key column vs INCLUDE column:**

| | Key column | INCLUDE column |
|---|---|---|
| Searchable / sortable (`WHERE`, `ORDER BY`) | Yes | No — output only |
| Stored in the tree's upper levels | Yes | No — leaf only |
| Counts toward the 900-byte / 16-column key limit | Yes | No |
| Can hold `varchar(max)` / large types | No | Yes |

**The rule:** filter or sort on it → **key column**. Only display it → **`INCLUDE`**.

**Result:** `SELECT OrderDate, Total FROM Orders WHERE CustomerId = @c` is now fully covered — one seek, values read straight from the leaf, no lookup.

**Trade-off:** a wider index costs more disk and memory, and **slows every write** that touches those columns.

**Takeaway:** don't `INCLUDE` columns speculatively. If you're building a covering index for a `SELECT *`, fix the `SELECT *` first.

---

## U3 — Execution Plans

### Q6. What is an execution plan, and how do you read one?

SQL is **declarative** — you describe *what* you want. **The execution plan is the strategy the optimizer chose for the *how***: which indexes, which join algorithms, whether to sort, in what order.

**It's cost-based:** the optimizer estimates the cost of candidate plans using **statistics** (Q12) and picks the cheapest one it found before running out of compile time.

**Two versions:**

| Plan | What it shows |
|---|---|
| **Estimated** | What it *would* do, without running. Guessed row counts. |
| **Actual** | The same plan plus **real runtime numbers**. Use this for debugging. |

**Read plans right to left**, top to bottom — data flows from the rightmost operators toward the final `SELECT`.

**Checklist:**

1. **Find the expensive operator** — each shows a cost %. Start with the biggest.
2. **Look for thick arrows** — thickness = row count. A thick arrow where you expected few rows is a problem.
3. **Compare estimated vs actual rows** — estimated 5, actual 500,000 means bad estimates. **This is the most useful diagnostic in the plan.**
4. **Watch for these operators:**
   - **Table / Clustered Index Scan** on a big table for a selective query → missing index or non-SARGable predicate (Q11)
   - **Key Lookup** → index doesn't cover the query (Q8)
   - **Sort** → often removable by indexing in that order; a **warning triangle** means it spilled to `tempdb`
5. **Read the warnings** — implicit conversions, missing statistics, spills.
6. **Distrust the green missing-index hint** — it ignores write cost, ignores existing indexes, and over-includes columns.

---

### Q7. Seek vs scan — when is a scan the right answer?

#### Index Seek

An **Index Seek** uses the index to **directly locate** the required rows without reading the entire index.

#### Index Scan

An **Index Scan** reads **all or most of the index** to find the required rows.

#### Which one is better?

Neither is always better. It depends on the query and the amount of data being retrieved.

- ✅ **Index Seek** is usually better when retrieving a **small number of rows**.
- ✅ **Index Scan** is better when retrieving **most of the rows**, the table is **small**, or scanning is cheaper than many seeks and lookups.

---

### Q8. What is a key lookup, why does it hurt, and how do you fix it?

**A key lookup is the return trip from Q3** — the non-clustered index found the rows but is missing a column the query needs, so SQL Server goes back to the clustered index to fetch it.

**Why it hurts: the lookup runs once per row.**

- 10 rows → unnoticeable.
- 100,000 rows → 100,000 separate random reads, often costing more than the original seek.

This is the classic "worked in dev, died in production" bug — fine on 100 test rows, catastrophic on a million real ones.

**In the plan:** a **Key Lookup** operator joined to your seek by a **Nested Loops**. (On a heap it's a **RID Lookup** — same thing.)

**The fix — cover the query:**

```sql
CREATE NONCLUSTERED INDEX IX_Orders_CustomerId_Incl
    ON dbo.Orders (CustomerId)
    INCLUDE (Total);        -- Total now at the leaf → no lookup
```

**Other valid answers:**

- **Select fewer columns** — often it was a stray `SELECT *`.
- **Do nothing** — if the query runs twice a day, an extra index isn't worth the write cost.

**The tipping point.** Since each lookup is a random read, the optimizer weighs *N lookups* against *one scan*. Past a certain fraction of rows — often around a quarter of the table — it **ignores your index entirely** and scans.

That's why a query sometimes "refuses to use" a perfectly good index. Two cases:

| Estimate | Meaning | Action |
|---|---|---|
| **Correct** | It really does return that many rows — the scan is right | Build a covering index to keep the seek path |
| **Wrong** | Stale stats or non-SARGable predicate | Fix the estimate; the seek returns |

### Q9. Explain the three join operators and when each is used.

The optimizer picks based on input sizes and whether inputs are sorted or indexed.

| Operator | How it works | Best when |
|---|---|---|
| **Nested Loops** | For each outer row, look up matches on the inner side | **Small outer** input + **indexed inner** side. Bad if both are large — work multiplies. |
| **Merge Join** | Both inputs already sorted on the join key; walk them in step | Sorting is **free** (both sides from indexes on that column). Otherwise Sorts must be added. |
| **Hash Match** | Build a hash table from the smaller input, probe with the larger | **Large, unsorted** inputs with no useful index. Needs a memory grant; **spills to `tempdb`** if too small. |

**Common follow-up:** *"You expected Nested Loops with a seek but got a Hash Match — why?"*

- **No index on the inner join column**, so it can't seek, or
- **Row estimates are so high** that hashing genuinely looks cheaper.

Check estimated vs actual rows to tell which.

---

## U4 — Composite Indexes & SARGability

### Q10. Explain the left-prefix rule, and how to order composite index columns.

**A composite index has several key columns in a fixed order** — `INDEX(a, b, c)` is sorted by `a`, then by `b` within equal `a`, then by `c` within equal `(a, b)`.

**The phone book analogy:** sorted by (LastName, FirstName), you can instantly find "Smith" or "Smith, John". But finding **everyone named John** means reading the whole book — it was never sorted by first name.

**The left-prefix rule:** an index can only seek when your `WHERE` covers a **leading run of columns from the left**.

| `WHERE` clause | Seek on `INDEX(a,b,c)`? |
|---|---|
| `a = @x` | Yes |
| `a = @x AND b = @y` | Yes |
| `a = @x AND b = @y AND c = @z` | Yes |
| `b = @y` | **No** — not a leading column → scan |
| `c = @z` | **No** → scan |
| `a = @x AND c = @z` | Partly — seeks on `a`, then filters `c` (can't seek `c`, `b` was skipped) |

**Choosing the order, in priority:**

1. **Equality columns first, then range columns.** Once you hit a range (`>`, `<`, `BETWEEN`), nothing after it can seek.
   For `WHERE Status = 'Open' AND OrderDate >= @d` → index `(Status, OrderDate)`, **not** `(OrderDate, Status)`.
2. **Match what queries actually filter on**, so common predicates form the leading columns.
3. **Match `ORDER BY`** where possible — the optimizer can then **skip the Sort entirely**.
4. **Selectivity is only a tiebreaker.** Among equality columns the more selective one narrows faster, but getting the *prefix* right matters far more.

**Takeaway:** one well-ordered composite index often replaces several single-column indexes, and may cover the query too.

### Q11. What is SARGability, and how does implicit conversion break it?

**SARGable = "Search ARGument-able"** — a predicate the optimizer can satisfy with a **seek**.

**The rule: leave the indexed column bare.** Compare it directly to a value. Wrap it in a function or do arithmetic on it and you get a **scan**.

**Why:** the index is sorted by the column's *actual values*, not by `f(value)`. An index on `OrderDate` is sorted by dates, so it can't locate rows where `YEAR(OrderDate) = 2024` — it would have to compute `YEAR()` on every row, which is a scan.

**The common offenders:**

```sql
-- BAD: function on the column
WHERE YEAR(OrderDate) = 2024
-- GOOD: range on the bare column
WHERE OrderDate >= '2024-01-01' AND OrderDate < '2025-01-01'

-- BAD: arithmetic on the column
WHERE Total * 1.1 > 100
-- GOOD: move the maths to the constant side
WHERE Total > 100 / 1.1

-- BAD: wrapping the column
WHERE ISNULL(Status, 'X') = 'Open'
-- GOOD: leave it bare (handle NULLs separately)
WHERE Status = 'Open'

-- BAD: leading wildcard — no starting point to seek to
WHERE Name LIKE '%son'
-- GOOD: trailing wildcard can seek
WHERE Name LIKE 'John%'
```

The `LIKE` pair makes the concept concrete: `'John%'` gives the index a place to jump to; `'%son'` doesn't, so matching rows could be anywhere.

**The sneaky one — implicit type conversion.**

If the parameter type doesn't match the column type, SQL Server converts one of them. If it converts **the column**, that's a function on the column → scan, and nothing in your SQL looks wrong.

The classic case is an **`NVARCHAR` parameter against a `VARCHAR` column**. Type precedence says `VARCHAR` converts up to `NVARCHAR`, so **the column** gets converted:

```sql
-- Code is VARCHAR; @p arrives as NVARCHAR (the .NET default)
WHERE Code = @p        -- becomes CONVERT(NVARCHAR, Code) = @p → full scan
```

**This is a common EF Core footgun** — .NET strings map to `NVARCHAR` by default, so your C# looks fine and the query scans a ten-million-row table.

| | |
|---|---|
| **Symptom** | A query that should seek is scanning; plan shows a **`PlanAffectingConvert`** warning |
| **Fix** | Match the parameter type to the column (`SqlDbType.VarChar`, or configure the EF column type), or cast the **parameter** — never the column |

See [EF Core notes](../Dotnet/dotnet-ef-core.md) section O for the EF Core side.

---

## U5 — Statistics

### Q12. What are statistics, and why does a query suddenly get slow with no code change?

**Statistics describe how data is distributed in a column.** The core is a **histogram** — up to 200 steps showing where values cluster — plus the row count and average duplication.

**What they're for:** estimating **how many rows a predicate will return, before running the query.** That single estimate drives every decision — seek or scan, which index, which join algorithm, how much memory to grant for sorts.

**The chain: good statistics → good row estimates → good plans.** A wrong estimate makes everything downstream wrong.

SQL Server maintains them automatically by default (`AUTO_CREATE_STATISTICS`, `AUTO_UPDATE_STATISTICS`).

**"It was fast last month, nobody changed anything, now it's slow"** — three suspects:

**1. Stale statistics.** Auto-update only fires after enough rows change. On a large, steadily growing table the histogram drifts behind reality, so the optimizer plans for data you had months ago and picks a scan where a seek is now right.

```sql
UPDATE STATISTICS dbo.Orders;                  -- default: samples a fraction, fast
UPDATE STATISTICS dbo.Orders WITH FULLSCAN;    -- reads every row: exact, expensive
```

Sampling is usually fine, but on **skewed data** a small sample gives a misleading histogram — use `FULLSCAN` on critical tables.

**2. Parameter sniffing.** The plan was compiled and cached for one parameter value and is now reused for a very different one. A plan that's perfect for one customer's orders is terrible for all orders. Deep dive in [sql-nosql-performance.md](sql-nosql-performance.md).

**3. Data volume crossed a threshold.** Nothing is stale — the table simply grew. A plan that was correct at 10,000 rows can be wrong at 10 million (the key-lookup tipping point in Q8 is exactly this).

---

## U6 — Filtered Indexes & Index Downsides

### Q13. What is a filtered index, and when is it useful?

**A filtered index covers only some rows** — those matching a `WHERE` clause on the index itself.

```sql
-- Soft delete: index only the live rows
CREATE NONCLUSTERED INDEX IX_Orders_Active
    ON dbo.Orders (CustomerId)
    WHERE IsDeleted = 0;
```

**Benefits:** smaller, cheaper to maintain, and **more accurate statistics** for that subset.

**Good fits:**

| Scenario | Filter |
|---|---|
| **Soft deletes** | `WHERE IsDeleted = 0` — if 90% of rows are deleted, the index is a tenth the size |
| **Mostly-empty columns** | `WHERE SomeColumn IS NOT NULL` |
| **Queue tables** | `WHERE Status = 'Pending'` — nobody queries the millions of completed rows |

**Two gotchas:**

- The query's predicate must be **compatible with the filter**, or the optimizer won't use the index.
- **Parameterised queries sometimes can't use them at all** — the parameter value isn't known at compile time, so SQL Server can't prove the filter applies. May need `OPTION (RECOMPILE)` or a literal.

PostgreSQL calls the same feature a **partial index**.

### Q14. What are the downsides of indexes — why not index everything?

**An index is a read optimisation paid for with slower writes and more space.** It's a trade, not a free win.

**The ongoing costs:**

| Cost | Detail |
|---|---|
| **Slower writes** | Every `INSERT` adds a row to **every** index; every `DELETE` removes from all of them; an `UPDATE` maintains each index whose columns changed. Ten indexes = ten structures to sync on every write. |
| **Disk & memory** | Each index is a full copy of its columns, competing for buffer-pool space. Unused indexes evict data you do use. |
| **Fragmentation** | Inserts into full pages cause **page splits** → pages half empty and out of order → scans read more. |
| **Maintenance** | Rebuilds and statistics updates consume time and resources. |
| **Slower compiles** | More overlapping indexes = more options to evaluate, and occasionally a worse choice. |

**The discipline: index for the queries you actually run, not the ones you imagine.**

**Two DMVs do the work:**

```sql
sys.dm_db_missing_index_details   -- indexes SQL Server wishes existed
sys.dm_db_index_usage_stats       -- which indexes are actually used
```

**Takeaway:** the second one is the more valuable and the one people forget. Unused indexes cost writes and memory while contributing nothing — find them and drop them.

### Q15. How would you approach a slow query in production?

This is what the rest of the section prepares you for. **A structured answer beats a list of tricks.**

**1. Confirm what's slow.** Get the specific query and its **actual** execution plan. Don't optimise from "the app feels slow."

**2. Read the plan** (Q6):

- **Estimated vs actual rows** — a big gap means bad statistics or a non-SARGable predicate. **Fix this first**; everything else follows from the estimate.
- A **scan on a big table** where you expected few rows.
- A **Key Lookup** running many times.
- **Sort spills** and warning icons.

**3. Check the `WHERE` is SARGable** (Q11). A function on a column or an accidental type mismatch disables the index no matter how well designed it is. This is a code fix, and it's free.

**4. Check the right index exists** — and in the right column order (Q10). Equality columns first, then range, matching `ORDER BY` where possible.

**5. Consider covering the query** (Q5) if a key lookup is doing real damage — but weigh the write cost.

**6. Update statistics** if the estimates were wrong (Q12).

**7. Measure again.** Compare before and after. If it didn't improve, revert it.

**The point to land:** understand *why* it's slow before changing anything. Adding indexes hopefully, one after another, is how a table ends up with forty indexes and a write workload that crawls.

---
