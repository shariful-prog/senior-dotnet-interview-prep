# X. Schema Design & Data Modeling
---

## X0 — Workload First: OLTP vs OLAP

### Q0. Explain OLTP vs OLAP.

They are two different **workloads**, and the difference decides how you model.

- **OLTP (Online Transaction Processing)** — the application database. Many users, short read/write transactions touching a few rows each: place an order, update a profile. Optimized for **writes, concurrency, and integrity**.
- **OLAP (Online Analytical Processing)** — the reporting database. Few users, long read-only queries aggregating millions of rows: revenue by region by quarter. Optimized for **scan speed**.

**Typical databases:**

| | Databases |
|---|---|
| **OLTP** | SQL Server, PostgreSQL, MySQL, Oracle |
| **OLAP** | Snowflake, Amazon Redshift, Google BigQuery, ClickHouse, Azure Synapse |

**Don't report off the OLTP database.** A large analytical scan competes for I/O and locks with transactions and can bring throughput down — a common production incident. Use a read replica, a summary table, or a real warehouse instead.

---

## X1 — Normalization

### Q1. What is normalization and what problem does it solve?

**Normalization** is the database design technique of organizing tables and columns to **eliminate data redundancy** and **prevent modification anomalies**. The primary rule: *Every fact should be stored in exactly one place.*

---

### 🚨 The Problem: Unnormalized (Flat) Table

Consider an unnormalized table where orders, customers, and products are crammed together:

```text
OrderId | CustomerName | CustomerPhone | ProductName | Price  | Qty
1001    | Alice        | 555-1234      | Pen         | 5.00   | 2
1002    | Alice        | 555-1234      | Notebook    | 12.00  | 1
1003    | Bob          | 555-5678      | Pen         | 5.00   | 3
```

This single table suffers from **Data Redundancy** and **Three Major Anomalies**:

#### 1. Data Redundancy (Unnecessary Duplication)
- Alice's phone `555-1234` is duplicated across multiple rows.
- The product price `5.00` for "Pen" is duplicated across multiple rows.
- **Impact:** Wasted storage space and risk of inconsistent data.

#### 2. The 3 Modification Anomalies
- **Update Anomaly (Data Inconsistency):**  
  If Alice changes her phone number, you must update every single row containing Alice. If you update row 1001 but miss row 1002, the database becomes inconsistent (`555-1234` vs `555-9999`).
- **Insertion Anomaly (Unable to insert independent facts):**  
  You want to add a new product ("Eraser") to your catalog. However, you **cannot insert it** because `OrderId` is the Primary Key and no customer has ordered an Eraser yet.
- **Deletion Anomaly (Accidental data loss):**  
  Bob has only placed one order (1003). If Bob cancels and deletes order 1003, you **lose all record of Bob's customer details and phone number** from your database.

---

### ✅ The Solution: Normalized Schema (3NF)

Break the flat table into 4 focused entities, linking them via Foreign Keys:

```sql
-- 1. Customers (stores customer details ONCE)
Customers (CustomerId PK, CustomerName, CustomerPhone)

-- 2. Products (stores product details ONCE)
Products (ProductId PK, ProductName, Price)

-- 3. Orders (stores order header info)
Orders (OrderId PK, CustomerId FK, OrderDate)

-- 4. OrderLines (stores order items)
OrderLines (OrderId FK, ProductId FK, Qty)
```

#### How Normalization Solves the Problems:
* **No Redundancy:** Alice's phone and Pen's price are stored in **only 1 place**.
* **No Update Anomaly:** Updating Alice's phone requires editing **1 single row** in `Customers`.
* **No Insertion Anomaly:** You can add "Eraser" into `Products` without creating a fake order.
* **No Deletion Anomaly:** Deleting Bob's order leaves Bob intact in the `Customers` table.

> **Trade-off:** Normalization guarantees data integrity and prevents bugs, but requires `JOIN` operations to read combined data. (Denormalization deliberately trades integrity for read speed).

---

### Q2. Explain 1NF, 2NF, and 3NF with a worked example.

The classic mnemonic created by Bill Kent summarizes normalization to **3NF**:

> **"Every non-key column must depend on *the key* (1NF), *the whole key* (2NF), and *nothing but the key* (3NF) — so help me Codd."**

---

### The Worked Example: From Messy to 3NF

#### Step 0: Unnormalized Form (UNF)
Here is a raw, unnormalized order log:

```text
OrderId | CustomerName | CustomerCity | Items
1001    | Alice        | Boston       | "Pen x2, Notebook x1"
1002    | Alice        | Boston       | "Stapler x1"
```
* **Problems:** The `Items` column contains multiple values in a single cell (not atomic).

---

#### 1NF (First Normal Form) — *"Depend on the Key"*
* **Rule:** 
  1. Each column cell must contain a single **atomic (indivisible) value**.
  2. No repeating column groups.
  3. Table must have a primary key identifying each row.
* **Fix:** Flatten the comma-separated list into separate rows.

```text
-- Composite PK: (OrderId, ProductName)
OrderId | OrderDate  | CustomerName | CustomerCity | ProductName | Price  | Qty
1001    | 2026-01-05 | Alice        | Boston       | Pen         | 5.00   | 2
1001    | 2026-01-05 | Alice        | Boston       | Notebook    | 12.00  | 1
1002    | 2026-01-07 | Alice        | Boston       | Stapler     | 15.00  | 1
```
* **New Problem:** Notice the Primary Key is now composite `(OrderId, ProductName)`. Some columns depend on only **part** of the key.

---

#### 2NF (Second Normal Form) — *"Depend on the Whole Key"*
* **Rule:** 
  1. Must already be in 1NF.
  2. **No Partial Dependencies:** Every non-key column must depend on the **entire** composite Primary Key, not just part of it. *(Only relevant when PK is composite).*
* **Problem in 1NF Table:**
  - `CustomerName`, `CustomerCity`, and `OrderDate` depend ONLY on `OrderId` (part of PK).
  - `Price` depends ONLY on `ProductName` (part of PK).
  - Only `Qty` depends on **both** `(OrderId + ProductName)`.
* **Fix:** Split data into 3 separate tables based on key dependency.

```sql
-- Table 1: Orders (Key = OrderId)
Orders (OrderId PK, OrderDate, CustomerName, CustomerCity)

-- Table 2: Products (Key = ProductName)
Products (ProductName PK, Price)

-- Table 3: OrderLines (Key = OrderId + ProductName)
OrderLines (OrderId FK, ProductName FK, Qty)
```

---

#### 3NF (Third Normal Form) — *"Depend on Nothing But the Key"*
* **Rule:** 
  1. Must already be in 2NF.
  2. **No Transitive Dependencies:** No non-key column can depend on another non-key column (A → B → C).
* **Problem in 2NF `Orders` Table:**
  - In `Orders (OrderId PK, OrderDate, CustomerName, CustomerCity)`, `CustomerCity` depends on `CustomerName` (a non-key column), NOT directly on `OrderId`.
  - `OrderId` → determines `CustomerName` → determines `CustomerCity`.
* **Fix:** Extract the `Customer` details into their own dedicated table.

```sql
-- 1. Customers Table (Key = CustomerId)
Customers (CustomerId PK, CustomerName, CustomerCity)

-- 2. Orders Table (Key = OrderId)
Orders (OrderId PK, OrderDate, CustomerId FK)

-- 3. Products Table (Key = ProductId)
Products (ProductId PK, ProductName, Price)

-- 4. OrderLines Table (Key = OrderId + ProductId)
OrderLines (OrderId FK, ProductId FK, Qty)
```

---

### 💡 Summary Cheat Sheet

| Normal Form | Elimination Target | Practical Meaning |
| :--- | :--- | :--- |
| **1NF** | Multi-value cells & repeating groups | "Make values atomic; set a primary key." |
| **2NF** | Partial dependencies | "Don't store product or customer info in a join-table." |
| **3NF** | Transitive dependencies | "Don't store customer address in an orders table." |

---

### Q4. What normal form do you target in production, and why not always go higher?

Target **3NF (effectively BCNF)** for transactional/OLTP schemas. It eliminates the redundancy anomalies while keeping the model intuitive.

Reasons not to blindly push higher or lower:

- **Fully normalized = more joins.** Every additional table is a join at read time. On hot read paths this can hurt.
- **Analytics/reporting workloads** deliberately go the other way (denormalized star/snowflake schemas in a data warehouse — mentioned only, out of scope here).
- **Selective denormalization** (X2) is a conscious, measured exception to 3NF, not sloppiness.

Rule of thumb: **normalize until it hurts, denormalize until it works** — start clean at 3NF, then denormalize specific paths when profiling proves it's needed.

---

## X2 — Denormalization & Trade-offs

### Q5. What is denormalization and when is it justified?

**Denormalization** is the intentional, strategic introduction of **data redundancy** into a normalized schema. Instead of keeping every fact in one place, you copy or precalculate data to **eliminate expensive JOINs and speed up read queries**.

> **Rule of thumb:** *Normalize for write integrity; denormalize for read performance.*

---

### 🎯 When is Denormalization Justified?

1. **Precomputing Heavy Aggregates:**  
   Avoiding repeated `SUM()`, `COUNT()`, or `AVG()` calculations over millions of rows on every user request.
2. **Historical Data Snapshots:**  
   Preserving point-in-time data (e.g., an order must keep the price at the time of purchase, even if the catalog price changes later).
3. **High Read-to-Write Ratios:**  
   When a path is read 10,000× per minute but updated once a week (e.g., e-commerce product catalogs).
4. **Avoiding Complex Multi-Table JOINs:**  
   Collapsing 4–5 joined tables into a single flat read table for fast API responses.

---

### 📊 Real-World Table Examples

#### Example 1: Author Info on Comments (Social Media / Blog Feeds)

* **Normalized Approach (3NF):**  
  The `Comments` table only stores `UserId`. To render a comment feed with 50 comments, the API must `JOIN` the `Users` table 50 times (or use an `IN` query) just to fetch the author's name and avatar.
  ```sql
  -- Normalized: Requires joining Users on every comment feed read
  SELECT c.CommentId, c.Text, c.CreatedAt, u.UserName, u.AvatarUrl
  FROM Comments c
  JOIN Users u ON c.UserId = u.UserId
  WHERE c.PostId = 1001;
  ```

* **Denormalized Approach:**  
  Copy `AuthorName` and `AuthorAvatarUrl` directly into the `Comments` table at creation time.
  ```text
  Comments Table:
  CommentId | PostId | UserId | AuthorName | AuthorAvatarUrl  | Text                | CreatedAt
  5001      | 1001   | 42     | "JohnDoe"  | "/avatars/42.png"| "Great article!"    | 2026-01-05
  ```
  * **Why it's justified:** Comment feeds are read thousands of times per minute. Fetching comments requires **zero joins**.
  * **The trade-off:** If John updates his username, an async background job or trigger must update his comments, or you accept that past comments retain his username at the time of posting.

---

#### Example 2: Category Name on Product Catalog Search

* **Normalized Approach (3NF):**  
  `Products` references `Categories` via `CategoryId`. Filtering and displaying product listings requires joining `Categories` on every search request.

* **Denormalized Approach:**  
  Store `CategoryName` directly inside `Products`.
  ```text
  Products Table:
  ProductId | Name        | Price  | CategoryId | CategoryName
  881       | "Wireless Mouse" | 29.99 | 12         | "Electronics & Accessories"
  ```
  * **Why it's justified:** Product listing APIs get huge traffic. Including `CategoryName` on `Products` eliminates joins during search, pagination, and sorting.
  * **The cost:** If a category is renamed, all products under it must be updated in a batch write.


> **Warning:** The hidden cost of denormalization is keeping copies synchronized. Always **normalize first by default**, and only denormalize when real profiling demonstrates a performance bottleneck.

---

### Q6. How do you keep denormalized data consistent? What are indexed/materialized views?

Options to maintain redundant copies, roughly from most to least risky:

1. **Application code** — update all copies in the same transaction. Simple but fragile; every write path must remember, and out-of-band writes bypass it.
2. **Triggers** — the database updates the copy automatically on INSERT/UPDATE/DELETE. Centralized, but adds hidden write cost and complexity.
3. **Indexed views (SQL Server) / materialized views (PostgreSQL)** — let the engine manage the denormalization for you.

An **indexed view** in SQL Server is a view with a **unique clustered index** materialized to disk; the engine keeps it in sync automatically on every base-table change. It supports precomputed joins and aggregates:

```sql
CREATE VIEW dbo.vw_CustomerOrderTotals
WITH SCHEMABINDING AS
SELECT o.CustomerId,
 COUNT_BIG(*) AS OrderCount,
 SUM(o.TotalAmount) AS LifetimeValue
FROM dbo.Orders o
GROUP BY o.CustomerId;
GO
CREATE UNIQUE CLUSTERED INDEX IX_vw_CustomerOrderTotals
 ON dbo.vw_CustomerOrderTotals(CustomerId);
```

Requirements are strict: `WITH SCHEMABINDING`, `COUNT_BIG(*)` for aggregates, deterministic expressions only. The upside — always consistent, maintained by the engine, and (Enterprise edition) the optimizer can auto-use it even when you query base tables.

**PostgreSQL** has `CREATE MATERIALIZED VIEW`, but it is **not** auto-maintained — you must `REFRESH MATERIALIZED VIEW [CONCURRENTLY]` on a schedule or trigger. That's a key difference: SQL Server indexed views are always current; Postgres materialized views are point-in-time snapshots.

---

## X3 — Keys

### Q8. What is a primary key and how is it physically implemented in SQL Server?

A **PRIMARY KEY** uniquely identifies each row. It is **UNIQUE + NOT NULL**, and there is **exactly one per table**.

In SQL Server, declaring a PRIMARY KEY creates a **unique clustered index by default** — meaning the table's rows are physically stored in PK order (the clustered index *is* the table). You can override this (`PRIMARY KEY NONCLUSTERED`) if you want the clustering key to be a different column, but the default is clustered.

```sql
CREATE TABLE dbo.Customers (
 CustomerId INT IDENTITY(1,1) PRIMARY KEY, -- clustered by default
 Email NVARCHAR(256) NOT NULL
);
```

Because the PK is usually the clustering key, the **choice of PK affects physical layout and every nonclustered index** (which carry the clustering key as their row locator). This is why a narrow, ever-increasing PK matters — see Q11 on GUIDs. (Cross-ref clustered vs nonclustered → sql-indexing-plans.md.)

---

### Q9. What is a foreign key and what does it enforce?

A **FOREIGN KEY** is a column (or set) in one table that references the PRIMARY KEY (or a UNIQUE key) of another, enforcing **referential integrity**: you cannot insert a child row pointing at a non-existent parent, and you cannot delete/update a parent in a way that orphans children (unless a referential action says otherwise — X4).

```sql
CREATE TABLE dbo.Orders (
 OrderId INT IDENTITY(1,1) PRIMARY KEY,
 CustomerId INT NOT NULL,
 CONSTRAINT FK_Orders_Customers
 FOREIGN KEY (CustomerId) REFERENCES dbo.Customers(CustomerId)
);
```

FKs also document the model and give the optimizer information (e.g., trusted FKs enable join elimination). **Index your foreign keys** — SQL Server does *not* auto-create an index on the FK column, and unindexed FKs cause slow child lookups and parent-delete scans (cross-ref sql-indexing-plans.md, and X7).

---

### Q10. Surrogate key vs natural key — pros, cons, and your default?

- **Natural key** — a column that already has business meaning: email, SSN, ISBN, country code.
- **Surrogate key** — a system-generated value with no business meaning: an IDENTITY `int`/`bigint`, a `SEQUENCE`, or a `uniqueidentifier` (GUID).

| | Natural key | Surrogate key |
|---|---|---|
| Extra column | none | one added |
| Stability | can change (email, name) | never changes |
| Width | can be wide (strings) | narrow (4/8 bytes for int/bigint) |
| Meaning | self-describing | opaque |
| Risk | may leak PII, ambiguous uniqueness | none semantic |

**Problem with natural keys as PK:** business values *change* (people change email/surname), and since the PK propagates into every FK and index, a change cascades painfully. They can also be wide (bloating every nonclustered index) and may **leak PII** (an SSN as a key travels into related tables and logs).

**Default: surrogate PK** (`INT`/`BIGINT IDENTITY`) — stable, narrow, meaningless. Still enforce the natural key with a **UNIQUE constraint** so you don't lose the business uniqueness rule:

```sql
CREATE TABLE dbo.Users (
 UserId INT IDENTITY(1,1) PRIMARY KEY, -- surrogate
 Email NVARCHAR(256) NOT NULL,
 CONSTRAINT UQ_Users_Email UNIQUE (Email) -- natural key still enforced
);
```

Natural keys are fine as PK for small, truly-stable reference tables (e.g., `Currencies(CurrencyCode CHAR(3) PK)`).

---

### Q11. Compare IDENTITY, SEQUENCE, NEWID(), and NEWSEQUENTIALID() for generating surrogate keys.

#### 1. `IDENTITY(seed, increment)`
* **Description:** An auto-incrementing number property attached to **one column in one specific table**. SQL Server assigns the next number automatically upon `INSERT`.
* **Example:**
  ```sql
  CREATE TABLE Users (
      UserId INT IDENTITY(1,1) PRIMARY KEY, -- starts at 1, increments by 1
      UserName NVARCHAR(50)
  );

  INSERT INTO Users (UserName) VALUES ('Alice'); -- UserId = 1
  INSERT INTO Users (UserName) VALUES ('Bob');   -- UserId = 2
  ```

#### 2. `SEQUENCE`
* **Description:** A standalone number generator object **independent of any table** (SQL Server 2012+). Unlike `IDENTITY` (which is locked to one table and only generates on `INSERT`), `SEQUENCE` is a central counter:
  - **Get ID before insert:** Fetch `SELECT NEXT VALUE FOR dbo.MySeq` in code *before* running `INSERT`.
  - **Share across tables:** Multiple tables can draw from the same sequence so IDs never collide.
  - **Control:** Supports cycling back to min value (`CYCLE`) and RAM range caching.
* **Example:**
  ```sql
  -- Create an independent Sequence object (this is NOT a table!)
  CREATE SEQUENCE dbo.GlobalSeq START WITH 1000 INCREMENT BY 1;

  -- 1. Fetch next number standalone (no table involved at all!)
  SELECT NEXT VALUE FOR dbo.GlobalSeq; -- Returns: 1000

  -- 2. Or share the SAME sequence across two different tables:
  CREATE TABLE StoreOrders (OrderId INT DEFAULT (NEXT VALUE FOR dbo.GlobalSeq) PRIMARY KEY);
  CREATE TABLE OnlineOrders (OrderId INT DEFAULT (NEXT VALUE FOR dbo.GlobalSeq) PRIMARY KEY);
  ```

#### 3. `NEWID()`
* **Description:** Generates a completely **random** 16-byte GUID (`uniqueidentifier`). Globally unique, but **causes severe index fragmentation** if used as a Clustered Primary Key because rows are inserted at random disk locations.
* **Example:**
  ```sql
  SELECT NEWID(); 
  -- Returns: '6F9619FF-8B86-D011-B42D-00C04FC964FF' (random value every execution)
  ```

#### 4. `NEWSEQUENTIALID()`
* **Description:** Generates GUIDs in **chronological / sequential order**. Preserves global uniqueness while keeping disk inserts append-only, preventing index fragmentation. *(Must be used as a column `DEFAULT`).*
* **Example:**
  ```sql
  CREATE TABLE EventLogs (
      LogId UNIQUEIDENTIFIER DEFAULT NEWSEQUENTIALID() PRIMARY KEY,
      Message NVARCHAR(255)
  );

  INSERT INTO EventLogs (Message) VALUES ('Server Started');
  -- LogId = 'A1F82142-6B9E-EE11-8B41-00155D012001' (increases sequentially per machine)
  ```

---

**Summary Guidance:**
- Default to **`IDENTITY`** or **`SEQUENCE`** for narrow, fast, sequential integer PKs.
- Use **`SEQUENCE`** if you need the ID *before* inserting or across multiple tables.
- Use **`NEWSEQUENTIALID()`** if you require GUID PKs without index fragmentation.
- Never use random **`NEWID()`** as a Clustered Primary Key.

---

### Q12. What is a composite key and when would you use one?

A **composite key** is a key made of **two or more columns** together. Common uses:

- **Junction tables** for many-to-many — the PK is the pair of FKs, e.g. `(StudentId, CourseId)` (see X5).
- **Natural composite identity** — e.g. `(TenantId, LocalId)` in multi-tenant designs, or `(OrderId, LineNumber)` for order lines.

```sql
CREATE TABLE dbo.OrderLines (
 OrderId INT NOT NULL,
 ProductId INT NOT NULL,
 Qty INT NOT NULL,
 CONSTRAINT PK_OrderLines PRIMARY KEY (OrderId, ProductId),
 CONSTRAINT FK_OL_Orders FOREIGN KEY (OrderId) REFERENCES dbo.Orders(OrderId),
 CONSTRAINT FK_OL_Products FOREIGN KEY (ProductId) REFERENCES dbo.Products(ProductId)
);
```

Trade-offs: composite keys are meaningful and avoid an extra column, but they **propagate all their columns into every child FK and nonclustered index** (widening them), and **column order matters** for the supporting index's usability. Many teams add a surrogate PK to junction tables *and* a UNIQUE constraint on the pair — but a plain composite PK is perfectly correct and often leaner. Also recall: **partial dependency (2NF)** is only a concern when the key is composite (Q2).

---

## X4 — Constraints & FK Actions

### Q13. Why enforce constraints in the database when the application already validates?

Because the database is the **last line of defense** for data integrity, and application validation is not sufficient on its own:

- **Multiple writers** — other apps, services, ETL jobs, ad-hoc scripts, and DBAs all touch the same tables. App validation only protects the one code path that runs it.
- **Bugs and races** — application checks can have gaps or concurrency holes; a `UNIQUE` constraint or `CHECK` is enforced atomically by the engine.
- **The optimizer trusts constraints** — trusted CHECK/FK constraints enable optimizations like join and partition elimination.
- **Self-documenting** — constraints declare the rules of the data model in one authoritative place.

Validate in the app for good UX (fast feedback), but **also** enforce at the DB for correctness. They're complementary, not either/or.

---

### Q14. Explain UNIQUE, CHECK, DEFAULT, and NOT NULL. How does UNIQUE differ from a primary key?

```sql
CREATE TABLE dbo.Products (
 ProductId INT IDENTITY(1,1) PRIMARY KEY,
 Sku NVARCHAR(40) NOT NULL,
 Price DECIMAL(10,2) NOT NULL DEFAULT 0,
 IsActive BIT NOT NULL DEFAULT 1,
 CONSTRAINT UQ_Products_Sku UNIQUE (Sku),
 CONSTRAINT CK_Products_Price CHECK (Price >= 0)
);
```

- **NOT NULL** — the column must have a value; forbids missing data.
- **DEFAULT** — supplies a value when none is given on insert (`GETUTCDATE()`, `0`, `1`, etc.).
- **CHECK** — enforces a domain/business rule via a boolean predicate (`Price >= 0`, `EndDate >= StartDate`, `Status IN ('New','Paid','Shipped')`). Note: a CHECK passes when the expression is `TRUE` **or `UNKNOWN`** — so `CHECK (Price >= 0)` does *not* block `NULL` prices; add `NOT NULL` if you mean it.
- **UNIQUE** — guarantees no duplicate values in the column(s).

**UNIQUE vs PRIMARY KEY:**

| | PRIMARY KEY | UNIQUE |
|---|---|---|
| Per table | exactly one | many allowed |
| NULLs | not allowed | **one NULL allowed** (SQL Server) |
| Default index | clustered | nonclustered |
| Purpose | the row's identity | an alternate/business uniqueness rule |

The **NULL behavior is a common gotcha**: SQL Server treats NULLs as equal for uniqueness, so a `UNIQUE` column allows only **one** NULL. **PostgreSQL** (and the SQL standard) treats NULLs as distinct, so a UNIQUE column allows **many** NULLs (Postgres 15+ can opt into SQL Server's behavior with `UNIQUE NULLS NOT DISTINCT`). To allow multiple NULLs in SQL Server, use a **filtered unique index**: `CREATE UNIQUE INDEX ... WHERE Sku IS NOT NULL`.

---

### Q15. Explain foreign key referential actions (ON DELETE / ON UPDATE) and their dangers.

When a referenced parent row is deleted or its key updated, the FK's **referential action** decides what happens to child rows:

- **NO ACTION** (default) — reject the operation if it would orphan children (raises an error). Equivalent in effect to RESTRICT in SQL Server, with a subtle timing difference.
- **CASCADE** — propagate: `ON DELETE CASCADE` deletes children too; `ON UPDATE CASCADE` updates their FK value.
- **SET NULL** — set the child's FK column to `NULL` (the column must be nullable).
- **SET DEFAULT** — set the child's FK to its DEFAULT value (which must reference an existing parent).

```sql
CREATE TABLE dbo.OrderLines (
 OrderId INT NOT NULL,
 ...
 CONSTRAINT FK_OL_Orders FOREIGN KEY (OrderId)
 REFERENCES dbo.Orders(OrderId)
 ON DELETE CASCADE -- delete lines when the order is deleted
);
```

**Dangers:**

- **Cascade-delete surprise** — `ON DELETE CASCADE` down a deep chain can silently wipe out huge subtrees of data from one `DELETE`. Many teams disable cascades and delete explicitly in code (or use soft delete, X7) precisely to avoid accidental mass deletes.
- **Multiple cascade paths error** — SQL Server **forbids** two cascade paths that reach the same table (e.g., a table reachable via two different cascading FKs). You get *"may cause cycles or multiple cascade paths"* at creation time. The fix is to set one path to `NO ACTION` and handle it manually (e.g., in a trigger or app code). This limitation is specific to SQL Server; PostgreSQL is more permissive here.

---

## X5 — Relationships & Junction Tables

### Q16. How do you model one-to-many, one-to-one, and many-to-many relationships?

**One-to-many** — the most common. Put the **FK on the "many" side.** One customer → many orders; the FK (`CustomerId`) lives on `Orders`.

```sql
Customers(CustomerId PK, ...)
Orders(OrderId PK, CustomerId FK NOT NULL, ...) -- FK on the many side
```

**One-to-one** — two ways:
- **Shared primary key** — the child's PK *is* also an FK to the parent (`UserProfile.UserId` is both PK and FK to `Users.UserId`). Cleanest.
- **FK + UNIQUE** — an FK column with a UNIQUE constraint forcing at most one match.

Used to split a wide table (rarely-used or large columns in a separate table), or for optional 1:1 extension data.

```sql
Users(UserId PK, ...)
UserProfiles(UserId PK, -- shared PK = also the FK
 Bio NVARCHAR(MAX),
 CONSTRAINT FK_Profile_User FOREIGN KEY (UserId) REFERENCES Users(UserId));
```

**Many-to-many** — needs a **junction (join/bridge/associative) table** (Q17).
---

### Q17. What is a junction table and how do you design one?

A **junction table** (a.k.a. join / bridge / associative / link table) resolves a **many-to-many** relationship by sitting between the two tables, holding a **foreign key to each** and typically a **composite primary key** of those two FKs (which prevents duplicate pairings).

Students Courses: a student takes many courses; a course has many students.

```sql
CREATE TABLE dbo.Enrollments (
 StudentId INT NOT NULL,
 CourseId INT NOT NULL,
 EnrolledAt DATETIME2(0) NOT NULL DEFAULT SYSUTCDATETIME(), -- payload column
 Grade CHAR(2) NULL,
 CONSTRAINT PK_Enrollments PRIMARY KEY (StudentId, CourseId), -- composite PK
 CONSTRAINT FK_Enr_Students FOREIGN KEY (StudentId) REFERENCES dbo.Students(StudentId),
 CONSTRAINT FK_Enr_Courses FOREIGN KEY (CourseId) REFERENCES dbo.Courses(CourseId)
);
```

Key design points:

- Composite PK `(StudentId, CourseId)` enforces "a student can't enroll in the same course twice."
- The junction table can carry its **own payload** (`EnrolledAt`, `Grade`) — the relationship itself has attributes. When it does, in EF Core you model it as an **explicit join entity**.
- **Index the second FK too.** The composite PK gives you an index leading with `StudentId`; add a nonclustered index leading with `CourseId` so lookups from the course side are efficient (cross-ref sql-indexing-plans.md).

---

### Q18. How do you model a self-referencing relationship (e.g., an org hierarchy)?

A **self-referencing** (recursive) relationship has an FK pointing back to the **same table's** PK — the many-to-one from a child to its parent within one table.

```sql
CREATE TABLE dbo.Employees (
 EmployeeId INT IDENTITY(1,1) PRIMARY KEY,
 Name NVARCHAR(100) NOT NULL,
 ManagerId INT NULL, -- NULL = top of the tree
 CONSTRAINT FK_Emp_Manager FOREIGN KEY (ManagerId)
 REFERENCES dbo.Employees(EmployeeId)
);
```

`ManagerId` is nullable so the CEO (root) has no parent. Note this is exactly why `ON DELETE CASCADE` on a self-reference triggers SQL Server's cascade-cycle error (Q15) — you'd handle deletes manually.

You traverse the hierarchy with a **recursive CTE** (`WITH ... AS (... UNION ALL ...)`), or use the built-in **`hierarchyid`** data type for materialized-path style storage. (Recursive CTEs → cross-ref sql query/CTE section.)

---

## X6 — Data Type Selection
### Q20. How do you choose types for money, dates, integers, and GUIDs?

**Money → `DECIMAL(p, s)` (a.k.a. `NUMERIC`), never `FLOAT`/`REAL`.** Float is binary floating-point and **cannot represent** most decimal fractions exactly, causing rounding errors that are unacceptable for currency. Use `DECIMAL(19,4)` or `DECIMAL(19,2)` depending on precision needs. The `MONEY` type exists but has caveats — limited scale (4 dp) and it can lose precision in intermediate division — so most seniors prefer explicit `DECIMAL`. (Cross-ref [C# strings & dates](../C-Sharp/csharp-strings-dates.md) for `decimal` in .NET — map SQL `DECIMAL` C# `decimal`.)

**Dates/times:**
- **`date`** — date only.
- **`datetime2(n)`** — preferred over the legacy **`datetime`**: wider range, higher precision, and it's the SQL-standard type. `datetime` has ~3.33 ms rounding and a smaller range — avoid it in new schemas.
- **`datetimeoffset`** — datetime2 **plus** a timezone offset; use when you must preserve the originating offset. Best practice for most apps is to store UTC in `datetime2` and convert at the edges. (Cross-ref [C# strings & dates](../C-Sharp/csharp-strings-dates.md) for `DateTime`/`DateTimeOffset`.)

**Integers — right-size:** `tinyint` (0–255), `smallint`, `int` (±2.1B), `bigint` (±9.2 quintillion). Use `int` for most keys; jump to `bigint` when you'll exceed 2.1B rows or IDs (high-volume event/log tables). Oversizing wastes space in every row and index; undersizing risks a painful migration when the identity overflows.

**GUID as key** — `uniqueidentifier` is 16 bytes. Fine as a key when you need distributed generation, but see Q11: random GUIDs (`NEWID()`) make a terrible **clustered** key. Prefer sequential generation or keep GUIDs nonclustered.

---

### Q21. What sizing mistakes hurt performance? (MAX types, over-wide columns)

- **`VARCHAR(MAX)` / `NVARCHAR(MAX)` in indexes** — MAX (LOB) columns **cannot be part of an index key** and can only be trailing `INCLUDE`d columns; storing large values off-row also slows scans. Don't reach for MAX unless values genuinely can exceed 8000/4000 bytes. Give real columns a real bounded length.
- **`varchar(4000)` "just in case"** — over-wide columns hurt in two ways. First, **memory-grant estimation**: SQL Server estimates memory for sorts/hashes based on **declared** size (it assumes ~half the max length), so a `varchar(4000)` holding 20-char values inflates memory grants and can spill to tempdb. Second, wide columns bloat rows and indexes. **Size columns to the actual domain** (`varchar(256)` for email, `char(3)` for currency code), not to the maximum imaginable.
- **`char` vs `varchar`** — use fixed-length `char(n)` only for values that are genuinely always the same length (`char(2)` state code, `char(1)` flag); it pads to full width, wasting space for variable data. Use `varchar` for variable-length text.

---

## X7 — Practical Patterns

### Q22. What is soft delete? How do you implement it well, and what are the trade-offs?

**Soft delete** marks a row as deleted with a flag (or timestamp) instead of physically removing it — the row stays but is filtered out of normal queries.

```sql
ALTER TABLE dbo.Orders ADD IsDeleted BIT NOT NULL DEFAULT 0;
-- or a nullable DeletedAt DATETIME2 (NULL = live, timestamp = when deleted)

-- filtered index so live-row queries stay fast and the index stays small
CREATE NONCLUSTERED INDEX IX_Orders_Live
 ON dbo.Orders (CustomerId)
 WHERE IsDeleted = 0;
```

**Implementation notes:**
- Use a **filtered index** `WHERE IsDeleted = 0` so indexes cover only live rows.
- In **EF Core**, use a **global query filter** (`HasQueryFilter(e => !e.IsDeleted)`) so every query auto-excludes deleted rows (cross-ref [EF Core](../Dotnet/dotnet-ef-core.md)). Remember to `IgnoreQueryFilters()` for admin/restore views.
- **UNIQUE constraints break** with soft delete: a "deleted" row still occupies the unique value, blocking reuse. Solution: a **filtered unique index** `WHERE IsDeleted = 0`.

**Pros:** recoverable (undo/restore), preserves audit/history and referential integrity (FKs still resolve), no cascade-delete data loss.
**Cons:** tables grow forever (need archiving), every query must remember the filter (a forgotten filter leaks deleted rows), unique constraints and FKs need special handling, and it complicates true "right to be forgotten" (GDPR) deletion.

---

### Q23. How do you implement audit columns and point-in-time history (temporal tables)?

**Audit columns** — the lightweight approach: track who/when on each row.

```sql
CREATE TABLE dbo.Products (
 ProductId INT IDENTITY(1,1) PRIMARY KEY,
 Name NVARCHAR(100) NOT NULL,
 CreatedAt DATETIME2(3) NOT NULL DEFAULT SYSUTCDATETIME(),
 CreatedBy NVARCHAR(128) NOT NULL,
 ModifiedAt DATETIME2(3) NULL,
 ModifiedBy NVARCHAR(128) NULL
);
```

This tells you the last state's author/time but **not the full history** of changes.

**Temporal tables (system-versioned)** — when you need **point-in-time** history, SQL Server maintains an automatic history table for you:

```sql
CREATE TABLE dbo.Products (
 ProductId INT PRIMARY KEY,
 Name NVARCHAR(100) NOT NULL,
 Price DECIMAL(10,2) NOT NULL,
 ValidFrom DATETIME2 GENERATED ALWAYS AS ROW START NOT NULL,
 ValidTo DATETIME2 GENERATED ALWAYS AS ROW END NOT NULL,
 PERIOD FOR SYSTEM_TIME (ValidFrom, ValidTo)
) WITH (SYSTEM_VERSIONING = ON (HISTORY_TABLE = dbo.ProductsHistory));
```

Every UPDATE/DELETE automatically writes the previous version to the history table. You then query any point in time:

```sql
SELECT * FROM dbo.Products FOR SYSTEM_TIME AS OF '2026-01-01T00:00:00';
```

Use audit columns for simple "who touched this last"; use **temporal tables** when you need full, tamper-resistant change history and time-travel queries. (PostgreSQL has no built-in equivalent — you use triggers + a history table, or extensions.)

---

### Q24. What's your indexing strategy when designing a schema? And how do you handle optimistic concurrency?

**Indexing strategy (design-time baseline — cross-ref sql-indexing-plans.md for depth):**

1. **PK → clustered index** on a narrow, ever-increasing key (int/bigint IDENTITY). This is your default clustering key.
2. **Index every foreign key.** SQL Server does *not* auto-index FK columns; unindexed FKs cause slow joins and, worse, force **scans of the child table on every parent delete** (to check for references). This is the single most common schema-level indexing miss.
3. **Enforce UNIQUE** on natural/business keys (also gives you an index).
4. **Index by the actual query workload, not blindly** — add nonclustered indexes for the real `WHERE` / `JOIN` / `ORDER BY` columns your queries use, and use `INCLUDE` columns for covering. Don't over-index: every index taxes writes and storage. Design a *baseline* (PK + FKs + uniqueness), then tune from measured query patterns.
5. **Filtered indexes** for soft-delete (`WHERE IsDeleted = 0`) and sparse/optional columns.

**Optimistic concurrency** — add a **`rowversion`** (a.k.a. `timestamp`) column: an auto-incrementing 8-byte value SQL Server bumps on every update. The app reads it, then includes it in the `WHERE` of its update; if another writer changed the row first, the rowversion won't match, zero rows update, and you detect the conflict.

```sql
ALTER TABLE dbo.Products ADD RowVer rowversion; -- auto-maintained by the engine

UPDATE dbo.Products SET Price = @NewPrice
WHERE ProductId = @Id AND RowVer = @OriginalRowVer; -- 0 rows affected = conflict
```

EF Core maps this with `[Timestamp]` / `IsRowVersion()` and throws `DbUpdateConcurrencyException` on mismatch (cross-ref sql-transactions-locking.md and [EF Core O7](../Dotnet/dotnet-ef-core.md#o7--transactions--concurrency)). **PostgreSQL** has no `rowversion`; use the system `xmin` column or a manual integer version column bumped on each update.

---

## X8 — Programmability: Views, Procedures, Functions, Triggers

### Q25. What is a view, and what can and can't it do?

A **view** is a named, stored `SELECT` — a virtual table. It stores **no data** (an *indexed* view is the exception, Q6); each reference re-runs the underlying query, and the optimizer expands the definition into the calling query and optimizes the whole thing together.

```sql
CREATE VIEW dbo.vActiveCustomerOrders
AS
SELECT c.CustomerId, c.Name, o.OrderId, o.OrderDate, o.Total
FROM dbo.Customers c
JOIN dbo.Orders    o ON o.CustomerId = c.CustomerId
WHERE c.IsDeleted = 0;
```

---

### Q26. Stored procedure vs function — what's the difference and when do you use each?

The core mental model:
* **Stored Procedure = Action / Doer** (Modifies data, runs business workflows, manages transactions).
* **Function = Calculator / Reader** (Computes a value or table, used directly inside `SELECT`/`WHERE` queries, read-only).

---

### 🔑 Key Differences at a Glance

| Feature | Stored Procedure | Function (UDF) |
| :--- | :--- | :--- |
| **How to Call** | Standalone command using `EXEC ProcName` | Inside queries: `SELECT dbo.MyFunc()` |
| **Modifies Data?** | **YES** (`INSERT`, `UPDATE`, `DELETE`) | **NO** (Read-only; cannot modify tables) |
| **Transactions** | **YES** (`BEGIN TRAN`, `COMMIT`, `ROLLBACK`) | **NO** (Cannot start/manage transactions) |
| **Return Value** | Optional (Returns status, output params, or query results) | **Mandatory** (Must return a single value or table) |
| **Error Handling** | **YES** (`TRY...CATCH` blocks) | **NO** (`TRY...CATCH` not allowed) |

---

### 📝 Code Examples

#### 1. Stored Procedure Example (Modifies Data & Manages Transactions)
```sql
CREATE PROCEDURE dbo.CreateUser 
    @UserName NVARCHAR(50), 
    @NewUserId INT OUTPUT
AS
BEGIN
    BEGIN TRANSACTION;
    INSERT INTO dbo.Users (UserName) VALUES (@UserName);
    SET @NewUserId = SCOPE_IDENTITY();
    COMMIT TRANSACTION;
END;

-- Execution: Standalone statement
EXEC dbo.CreateUser @UserName = 'Alice', @NewUserId = @Id OUTPUT;
```

#### 2. Function Example (Calculates a Value inside a Query)
```sql
CREATE FUNCTION dbo.CalculateTax (@Amount DECIMAL(10,2))
RETURNS DECIMAL(10,2)
AS
BEGIN
    RETURN @Amount * 0.15; -- 15% Tax
END;

-- Execution: Used inside a SELECT query
SELECT OrderId, Total, dbo.CalculateTax(Total) AS Tax 
FROM dbo.Orders;
```

---

### 🎯 When to Use Each

* **Use a Stored Procedure when:**
  - Performing data mutations (`INSERT`, `UPDATE`, `DELETE`).
  - Executing complex multi-step business logic requiring transactions and error handling (`TRY...CATCH`).

* **Use a Function when:**
  - You need a reusable calculation inside a `SELECT`, `WHERE`, or `JOIN` clause.
  - You need an **Inline Table-Valued Function (ITVF)** to act as a parameterized view you can `JOIN` against.

---

### Q27. What are triggers and why would you use one?

A **trigger** is code that the engine fires **automatically** in response to an event on a table — you never call it, and callers can't tell it ran. That implicitness is both its power and its danger.

**DML triggers** (`INSERT`/`UPDATE`/`DELETE`) are the common case; SQL Server also has **DDL triggers** (fire on `CREATE`/`ALTER`/`DROP` — used for schema-change auditing or to block changes) and **logon triggers**.

Inside a DML trigger you get two **pseudo-tables** holding the affected rows:

| Operation | `inserted` | `deleted` |
|---|---|---|
| `INSERT` | new rows | empty |
| `DELETE` | empty | removed rows |
| `UPDATE` | new values | old values |

```sql
CREATE TRIGGER dbo.trProducts_Audit
ON dbo.Products
AFTER UPDATE
AS
BEGIN
    SET NOCOUNT ON;
    INSERT dbo.ProductAudit (ProductId, OldPrice, NewPrice, ChangedAt, ChangedBy)
    SELECT i.ProductId, d.Price, i.Price, SYSUTCDATETIME(), SUSER_SNAME()
    FROM inserted i
    JOIN deleted  d ON d.ProductId = i.ProductId   -- set-based: handles multi-row updates
    WHERE i.Price <> d.Price;                      -- only real changes
END;
```

**Justified uses:** auditing that must not be bypassable, maintaining a denormalized/aggregate column (Q6), enforcing a cross-row/cross-table rule a `CHECK` constraint can't express, and making a multi-table view updatable (Q28).

**Why seniors are wary.** Triggers are **hidden work on every write** — invisible in the calling code, easily forgotten during debugging ("why is this insert slow / why did this column change?"), they run **inside the caller's transaction** (so they extend its duration and lock footprint, and a failure rolls the caller back), and they're a classic **deadlock** contributor. Prefer a constraint if a constraint can do it; prefer explicit application code or a temporal table (Q23) for history when you can.

**The #1 trigger bug: assuming one row.** A trigger fires **once per statement, not once per row**. This is the mistake to call out in an interview:

```sql
-- BROKEN: silently corrupts data on any multi-row UPDATE
DECLARE @Id INT = (SELECT ProductId FROM inserted);   -- arbitrary single row!
UPDATE dbo.Stock SET Qty = Qty - 1 WHERE ProductId = @Id;
```

Always write **set-based** trigger logic joining `inserted`/`deleted`, as in the example above.
---

### Q28. `AFTER` vs `INSTEAD OF` triggers — what's the difference?

Two firing modes, and the distinction is precise:

- **`AFTER`** (a.k.a. `FOR`, the default) — fires **after** the DML has been applied and constraints/FKs have been validated, but **inside the same transaction**, so it can still `ROLLBACK`. Use for auditing and follow-on work. **Tables only** — not allowed on views.
- **`INSTEAD OF`** — fires **in place of** the DML; the original operation **never happens** unless the trigger performs it itself. Runs **before** constraint checking. Allowed on **tables *and* views** — this is the only way to make a multi-table view updatable.

```sql
-- INSTEAD OF: makes a 2-table view updatable by routing the write
CREATE TRIGGER dbo.trCustomerFull_Update
ON dbo.vCustomerFull                    -- a view joining Customers + Addresses
INSTEAD OF UPDATE
AS
BEGIN
    SET NOCOUNT ON;
    UPDATE c SET c.Name = i.Name
    FROM dbo.Customers c JOIN inserted i ON i.CustomerId = c.CustomerId;

    UPDATE a SET a.City = i.City
    FROM dbo.Addresses a JOIN inserted i ON i.CustomerId = a.CustomerId;
END;

-- INSTEAD OF DELETE turning a hard delete into a soft delete (Q22)
CREATE TRIGGER dbo.trOrders_SoftDelete
ON dbo.Orders
INSTEAD OF DELETE
AS
UPDATE o SET o.IsDeleted = 1, o.DeletedAt = SYSUTCDATETIME()
FROM dbo.Orders o JOIN deleted d ON d.OrderId = o.OrderId;
```

**Key differences to state:**

| | `AFTER` | `INSTEAD OF` |
|---|---|---|
| **Timing** | After the DML + constraint checks | Replaces the DML, before checks |
| **Original operation** | Already happened | Happens only if you write it |
| **On views** | No | **Yes** (the main use case) |
| **Per table/event** | Many allowed | **Only one** per event per table/view |
| **Typical use** | Auditing, cascading updates | Updatable views, soft delete, validation/redirect |

---

### Q29. Compare `#TempTable`, `@TableVariable`, and `CTE` — when do you use each?

| Feature | `#TempTable` | `@TableVariable` | `CTE` (Common Table Expression) |
| :--- | :--- | :--- | :--- |
| **Storage Location** | `tempdb` (disk) | `tempdb` (in-memory buffer, spills to `tempdb`) | In-memory query expression (No physical storage) |
| **Scope** | Current Session / Connection | Local batch or procedure | Single SQL statement |
| **Statistics** | ✅ Full Column Statistics | ❌ No Statistics (Assumes 1 row in SQL 2017 & earlier) | N/A (Derived from underlying tables) |
| **Indexes** | ✅ Clustered & Non-Clustered Indexes | ⚠️ Primary Key & Unique Constraints only | N/A |
| **Transaction Rollback** | ✅ Rolled back on `ROLLBACK TRAN` | ❌ **NOT** rolled back on transaction rollback | N/A |
| **Reusability** | Multiple queries within session | Multiple queries within batch | Single immediate query only |

> **Rule of Thumb**:
> - Use **`CTE`** for readable recursion or single-query modularization.
> - Use **`@TableVariable`** for small datasets ($< 100$ rows) passed into functions or short scripts.
> - Use **`#TempTable`** for large intermediate datasets ($> 1000$ rows) where you need indexes, statistics, or multiple steps.

---

### Q30. How do you execute Dynamic SQL safely using `sp_executesql` and `QUOTENAME()` to prevent SQL Injection?

Executing dynamic SQL strings constructed via raw string concatenation (`EXEC('SELECT * FROM Users WHERE Name = ''' + @name + '''')`) exposes the database to severe **SQL Injection** attacks.

```sql
-- ❌ Dangerous (SQL Injection Risk via String Concatenation):
-- EXEC('SELECT * FROM Users WHERE Email = ''' + @userEmail + '''');

-- ✅ Safe (Parameterized Dynamic SQL with sp_executesql):
DECLARE @sql NVARCHAR(MAX);
DECLARE @params NVARCHAR(MAX);

SET @sql = N'SELECT UserId, Name, Email FROM dbo.Users WHERE Email = @TargetEmail AND IsActive = @ActiveFlag';
SET @params = N'@TargetEmail NVARCHAR(100), @ActiveFlag BIT';

EXEC sp_executesql 
    @stmt = @sql,
    @params = @params,
    @TargetEmail = 'alice@example.com',
    @ActiveFlag = 1;
```

#### Handling Dynamic Identifiers (Table & Column Names):
Parameters (`@TargetEmail`) cannot be used for SQL identifiers (table names or column names). Use **`QUOTENAME()`** to safely wrap dynamic table/column names in brackets:

```sql
DECLARE @tableName NVARCHAR(128) = N'Users; DROP TABLE Orders;--';
DECLARE @safeTableName NVARCHAR(258) = QUOTENAME(@tableName); // Converts to [Users; DROP TABLE Orders;--]

DECLARE @query NVARCHAR(MAX) = N'SELECT COUNT(*) FROM ' + @safeTableName;
EXEC sp_executesql @query; // Safely fails table lookup without executing malicious SQL!
```

