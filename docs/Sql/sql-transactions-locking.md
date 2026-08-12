# W. Transactions, Isolation & Locking


## W1 — ACID Properties

### Q1. What does ACID stand for, and what does each property guarantee?

**ACID** is a set of four properties that ensure database transactions are processed **safely, correctly, and reliably**.

- **A — Atomicity (All or Nothing):**  
  A transaction is treated as a single unit of work. Either **all operations succeed**, or **all changes are rolled back** (`ROLLBACK`) if any step fails.  
  *Example:* During a bank transfer, if $100 is deducted from **Account A** but the server crashes before adding it to **Account B**, SQL Server rolls back the transaction so the money is never lost.

- **C — Consistency (Follows the Rules):**  
  A transaction must always leave the database in a **valid state** by following all defined rules, such as primary keys, foreign keys, and check constraints. If any rule is violated, the transaction is cancelled.  
  *Example:* If an account has **$50** and a rule says `Balance >= 0`, trying to transfer **$100** fails because it would make the balance negative. No changes are saved.

- **I — Isolation (No Interference):**  
  Multiple transactions can run at the same time without affecting each other. Each transaction behaves as if it is the only one running.  
  *Example:* Two customers try to buy the last laptop at the same time. Isolation ensures only one purchase succeeds, preventing both customers from buying the same item.

- **D — Durability (Never Lost Once Saved):**  
  Once a transaction is successfully committed (`COMMIT`), the changes are permanently saved, even if the server crashes or loses power immediately afterward.  
  *Example:* A bank transfer is committed successfully. Even if the server crashes a second later, the transfer remains in the database after it restarts.

---


### Q2. How does SQL Server actually deliver each ACID property under the hood?

SQL Server uses specific internal mechanisms to enforce each ACID guarantee:

### ⚙️ How SQL Server Delivers Each Guarantee

1. **Atomicity & Durability → The Transaction Log & Write-Ahead Logging (WAL)**
   - **Write-Ahead Logging (WAL):** SQL Server **always writes changes to the Transaction Log (`.ldf`) on disk first**, before updating the actual data file (`.mdf`).
   - **Atomicity (Undo):** If a transaction fails or is cancelled (`ROLLBACK`), SQL Server reads the transaction log and **undoes** all partial changes.
   - **Durability (Redo):** When you run `COMMIT`, SQL Server immediately flushes the log records to disk. If the server crashes 1ms later, SQL Server reads the log upon reboot and **replays (redoes)** the committed changes.
   - **Checkpoints:** SQL Server periodically saves modified memory pages (RAM) to disk so recovery doesn't take hours replaying old logs.

2. **Consistency → Constraint Enforcement & Schema Rules**
   - SQL Server enforces `PRIMARY KEY`, `FOREIGN KEY`, `NOT NULL`, and `CHECK` constraints at runtime.
   - If a statement violates any constraint, SQL Server immediately aborts the statement and rolls back the transaction, keeping the database valid.

3. **Isolation → Locking Mechanisms & Row Versioning**
   - **Locking (Pessimistic):** SQL Server uses Shared (S) and Exclusive (X) locks to hold rows/tables so concurrent queries don't corrupt each other's data.
   - **Row Versioning (Optimistic - RCSI/Snapshot):** Stores older copies of modified rows in `tempdb` so readers can read past versions without locking or blocking writers.

---

> **Note — Delayed Durability:** SQL Server allows an opt-in mode called `DELAYED_DURABILITY`. It batches log writes in RAM before saving to disk to boost write speed, but trades off durability (you could lose the last few milliseconds of commits if the server power cuts out).

---

## W2 — Isolation Levels (CORE)

### Q3. Name the isolation levels and rank them from least to most isolated.

An **isolation level** is a dial you set: how much do you protect your transaction from other people's changes, and how much speed are you willing to give up for it? More protection = more waiting.

Four levels come from the SQL standard, and SQL Server adds two extra ones that work differently (they keep old copies of rows instead of locking):

| Level | In plain words | Speed | Safety |
|---|---|---|---|
| **READ UNCOMMITTED** | Reads anything, even half-finished changes | Fastest | Least safe |
| **READ COMMITTED** *(default)* | Waits until a change is saved, then reads it | Fast | Basic |
| **REPEATABLE READ** | Rows you read stay frozen until you finish | Medium | Better |
| **SERIALIZABLE** | Acts as if transactions ran one after another | Slowest | Safest |
| **SNAPSHOT** *(old copies)* | You see the data exactly as it was when you started | Fast (no waiting) | High |
| **READ COMMITTED SNAPSHOT (RCSI)** *(old copies)* | Like READ COMMITTED, but never makes anyone wait | Fast (no waiting) | Basic |

You set it for your connection like this, and it stays until you change it:

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

### Q4. Explain READ UNCOMMITTED and the `NOLOCK` hint — why is it dangerous?

**READ UNCOMMITTED** means "just show me whatever is there right now, don't wait for anyone." It ignores the locks other transactions are holding, so it can read changes that are **not saved yet** — and those changes might get cancelled a second later. Reading data that never really existed is called a **dirty read**.

`WITH (NOLOCK)` is the same thing applied to one table:

```sql
SELECT * FROM Orders WITH (NOLOCK);            -- just this table
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED; -- whole connection
```

**Why it's risky** — people treat `NOLOCK` as a "make it faster" switch, but it's really a "show me possibly wrong data" switch:

- **Dirty reads:** you may see a value that gets rolled back and never actually existed.
- **Missing or duplicated rows:** if the database moves rows around while your query is scanning, you can see the **same row twice** or **miss a row completely** — even rows that were saved long ago. Most people don't expect this part.
- **Read errors:** occasionally the scan just fails with error 601 ("could not continue scan with NOLOCK due to data movement").

*Example:* A customer's order total is being updated from $100 to $500, but the transaction hasn't finished. A `NOLOCK` report reads $500. The transaction then fails and rolls back to $100 — your report now shows a number that was never true.

Only use it for rough dashboards or monitoring where "close enough" is fine. If your real problem is queries waiting on each other, use **RCSI** (Q9) instead — no waiting *and* correct data.

### Q5. What is READ COMMITTED (the default), and what does it prevent vs allow?

**READ COMMITTED** is SQL Server's default. The rule is simple: **you only ever see saved (committed) data**. If someone is in the middle of changing a row, your query waits until they finish.

It reads a row, then immediately lets go — it does **not** keep holding on to it for the rest of your transaction.

- **Stops:** dirty reads (you never see unsaved data).
- **Still allows:**
  - **Non-repeatable read** — you read a row, someone else changes it and saves, you read the same row again and get a different value.
  - **Phantom read** — you run a range query, someone inserts a new matching row, you run it again and get an extra row.

*Example:* You read Product 5's price and get $100. A colleague updates it to $120 and commits. Two seconds later, still inside your transaction, you read it again — now it's $120. Same transaction, two different answers.

Because reading and writing the same row can't happen at the same time, **readers make writers wait and writers make readers wait**. This is the most common cause of slow, "stuck" queries in production — and it's exactly what RCSI fixes.

### Q6. What does REPEATABLE READ add, and what does it still allow?

**REPEATABLE READ** holds on to every row you read **until your transaction ends**. So nobody can change or delete a row you've already looked at. Read it again, get the same answer — hence the name.

- **Stops:** dirty reads and non-repeatable reads.
- **Still allows:** **phantom reads**. It protects the rows you actually touched, but not the *empty space* around them — so someone can still `INSERT` a brand-new row that matches your `WHERE` clause.

*Example:* You run `SELECT * FROM Orders WHERE Amount > 1000` and get 5 rows. Those 5 rows are now protected. But someone inserts a new $2000 order. You re-run the query and get 6 rows — the extra one is the "phantom."

The cost: holding rows longer means other transactions wait longer, so you get more blocking and a higher chance of deadlocks.

### Q7. What does SERIALIZABLE guarantee, and how does it achieve it?

**SERIALIZABLE** is the strictest level. It makes transactions behave as if they ran **one at a time, in a queue**, even though they're actually running together.

It does this by locking **the gaps too**, not just the rows — so if your query says `WHERE Amount > 1000`, nobody can insert a new row in that range while you're working. This is called a **range lock**.

- **Stops:** dirty reads, non-repeatable reads, **and** phantom reads. Everything.
- **Cost:** the most locking, so the most waiting and the highest deadlock risk. It also needs a good index on your `WHERE` column — without one, SQL Server can't lock a narrow range and may end up locking the whole table.

*Example:* "Book the last available seat" logic. You check for free seats and insert a booking. SERIALIZABLE guarantees nobody sneaks in a booking between your check and your insert, so you can't oversell the seat.

Use it only when you truly need that guarantee — not as a default.

---

## W4 — Locking (CORE)

### Q8. Explain the main lock modes: Shared (S), Exclusive (X), and Update (U).

A **lock** is a claim a transaction puts on a row so others know what it's doing. Think of it like a sign on a door. There are three main kinds:

- **Shared (S) — "I'm reading this."**
  Many transactions can read the same row at the same time, so lots of S locks can sit on one row happily. Under READ COMMITTED it's released the moment the read finishes; under REPEATABLE READ and SERIALIZABLE it's held until the transaction ends.

- **Exclusive (X) — "I'm changing this, keep out."**
  Taken for `INSERT`, `UPDATE`, and `DELETE`. Nobody else can have *any* lock on that row while an X lock is on it — not even a reader. It's held until the transaction ends.

- **Update (U) — "I'm reading this, and I'll probably change it."**
  Taken while SQL Server is searching for the rows an `UPDATE` will modify. Only **one** U lock can exist on a row at a time. It tolerates existing readers, but not another U or an X. When the transaction actually writes, the U **turns into an X**.

**Why does U exist?** It prevents a very common deadlock.

Imagine both transactions do "read the row, then update it." Both take an S lock to read. Then both try to upgrade to X — but each has to wait for the *other* to drop its S lock first. Neither ever will. Stuck forever.

The U lock fixes this by only allowing one "I intend to update" claim at a time. The second transaction waits early, at the reading stage, instead of getting stuck later. SQL Server does this automatically for `UPDATE` and `DELETE`; you can ask for it yourself with the `UPDLOCK` hint when you read a row and plan to write it.

### Q9. What is lock granularity, and what's the trade-off?

**Granularity** just means: how big a chunk does the lock cover? SQL Server can lock a single **row**, an 8 KB **page** (a group of rows stored together), or an entire **table**. (There are a few sizes in between, but these three are what matter.)

The trade-off is always the same — small locks are friendly but expensive to manage:

- **Row locks** → best for sharing. Other transactions can work on neighbouring rows freely. But each lock uses about 96 bytes of memory and some CPU to track, so ten thousand row locks is real overhead.
- **Table locks** → almost free to manage (just one lock), but nobody else can touch the table at all.

SQL Server picks a starting size automatically based on how many rows the query will touch, and it can switch to a bigger lock partway through.

---

## W5 — Deadlocks (CORE)

### Q10. What is a deadlock, and how does SQL Server resolve one?

A **deadlock** is when two transactions are each waiting for something the other one is holding — so neither can ever move.

It's like two people in a narrow corridor, each refusing to step aside until the other does. Nobody is going anywhere without outside help.

```sql
-- Session 1 -- Session 2
BEGIN TRAN;
UPDATE Accounts SET ... WHERE Id='A'; -- X on A
 BEGIN TRAN;
 UPDATE Accounts SET ... WHERE Id='B'; -- X on B
UPDATE Accounts SET ... WHERE Id='B'; -- waits for S2's X on B
 UPDATE Accounts SET ... WHERE Id='A'; -- waits for S1's X on A
-- circular wait: DEADLOCK
```

SQL Server runs a background check (about every 5 seconds, more often when deadlocks are frequent) looking for exactly this situation. When it finds one, it has to break the tie: it picks one transaction as the **victim**, cancels it, rolls it back completely, and sends that session **error 1205**:

> *Transaction (Process ID N) was deadlocked on lock resources with another process and has been chosen as the deadlock victim. Rerun the transaction.*

Which one loses? Normally whichever is **cheapest to undo** (the one that has done the least work). The survivor carries on as if nothing happened.

If you want to influence the choice, you can:

```sql
SET DEADLOCK_PRIORITY LOW;  -- volunteer this transaction to be the victim
```

The lowest priority always loses, no matter which was cheaper.

Important mindset: you can't eliminate deadlocks completely. The correct approach is for your application to **catch error 1205 and try again**.

### Q11. What are the common causes of deadlocks?

- **Locking things in different orders** — one transaction touches A then B, another touches B then A. **This is by far the most common cause.**
- **Both readers trying to become writers** — two transactions read the same row, then both try to update it, each waiting for the other to let go. (This is the exact problem the **U** lock prevents.)
- **Lock escalation** — one transaction grabs the whole table while another is holding rows inside it.
- **Missing indexes** — with no index, a query scans and locks far more rows than it needs to, so collisions become much more likely.
- **Long transactions** — the longer you hold locks, the more chances there are to collide with someone.
- **Foreign keys without indexes** — deleting a parent row means checking child rows; with no index on the child, that check locks a lot more than it should.

### Q12. How do you prevent (reduce) deadlocks?

You can't make them impossible, but you can make them rare:

- **Always touch tables in the same order** everywhere in your code (e.g. always Accounts before Orders). This one habit removes the most common deadlock entirely.
- **Keep transactions short.** Open late, commit fast. Never do application logic, API calls, or wait for a user inside a transaction.
- **Turn on RCSI.** Reads stop taking locks, so read-vs-write deadlocks mostly disappear. And avoid SERIALIZABLE unless you genuinely need it — its gap locks make deadlocks much more likely.
- **Use `UPDLOCK` when you read a row you're about to update.** This takes the "one at a time" U lock straight away instead of a shared lock that has to upgrade later:

 ```sql
 BEGIN TRAN;
 SELECT @bal = Balance FROM Accounts WITH (UPDLOCK) WHERE Id='A'; -- U lock now
 UPDATE Accounts SET Balance = @bal - 100 WHERE Id='A'; -- converts U→X, no deadlock race
 COMMIT;
 ```

- **Add the indexes you're missing** so queries lock a handful of rows instead of scanning the table. Don't forget indexes on foreign key columns.
- **Update in batches** so you never trigger escalation.

### Q13. How do you detect and diagnose deadlocks?

The good news: SQL Server already records every deadlock for you, with no setup required.

- **The `system_health` session** is running by default and captures an `xml_deadlock_report` for every deadlock that happens. So you can go look at deadlocks from last week without having turned anything on in advance. (`system_health` is an Extended Events session — Extended Events is SQL Server's built-in event-tracking system.)
- **The deadlock graph** is the report itself. Open it in SSMS and you get a picture showing both transactions, which rows/tables each one held, which one each was waiting for, the actual SQL they ran, and which one got killed. **This is the thing you actually want** — it shows you the exact ordering that caused the cycle, which tells you what to fix.
- **`sys.dm_tran_locks`** and **`sys.dm_os_waiting_tasks`** show what's locked and waiting *right now*. Useful for live blocking, less so for a deadlock that already happened.
- **Trace flags 1204 / 1222** dump deadlock details into the error log. Older approach — mention it if asked, but the deadlock graph is the modern answer.

The workflow: read the deadlock graph → spot the two objects being locked in opposite orders → fix the order, add an index, or switch on RCSI.

### Q14. How should the application handle deadlocks?

Here's the key insight: **error 1205 is a temporary problem, not a bug in your data.** The victim was rolled back completely and cleanly — no half-finished work was left behind. The two transactions simply collided on timing, and that timing probably won't repeat.

So the fix is straightforward:

- **Catch error 1205 and run the whole transaction again.** Wait a moment first, and cap it at a few attempts so you don't retry forever. Retrying is completely safe here because the rollback was total — you're not at risk of applying something twice.
- **In EF Core, you get this for free** by enabling retries on the SQL Server provider:

  ```csharp
  options.UseSqlServer(conn, o => o.EnableRetryOnFailure());
  ```

  It automatically retries temporary failures, deadlocks included.
- **Don't retry everything.** Only temporary errors deserve a retry. If the failure was a constraint violation or bad input, retrying will just fail again the same way — that's a real error to surface, not to paper over.

---

## W6 — Blocking vs Deadlock; Optimistic vs Pessimistic

### Q15. What's the difference between blocking and a deadlock?

People use these two words as if they mean the same thing, but they're very different situations.

**Blocking is normal.** One transaction wants a row that another one is using, so it **waits its turn**. The moment the first transaction commits or rolls back, the waiter carries on. Nothing is broken — this happens constantly in every busy database, and it fixes itself.

Blocking only becomes a *problem* when the wait is too long — usually because the blocker is a slow transaction, is missing an index, or escalated to a table lock. Then queries start timing out.

**A deadlock is broken.** It's *circular*: A is waiting on B **and** B is waiting on A. It will never sort itself out, no matter how long you wait. SQL Server has to step in and kill one of them (error 1205).

**The easy way to remember it:** blocking is a **queue** — annoying, but you'll get served. A deadlock is a **standoff** — nobody gets served until someone is removed.

How to investigate each:

| | Blocking | Deadlock |
|---|---|---|
| Resolves itself? | Yes | Never |
| Look at | `sys.dm_exec_requests` → `blocking_session_id` | the deadlock graph |
| Does RCSI help? | Yes, a lot | Only read/write ones |

That last point matters in interviews: **RCSI wipes out most blocking** because readers stop waiting on writers. But it doesn't help when **two writers** deadlock — writers still take exclusive locks no matter what.

### Q16. Optimistic vs pessimistic concurrency (database side) — when do you use each?

Both answer the same question — "what if two people want to change the same row?" — but they start from opposite assumptions.

**Pessimistic — "clashes are likely, so I'll lock it now."** You claim the row the moment you read it, and hold it until you're done:

```sql
BEGIN TRAN;
SELECT * FROM Seats WITH (UPDLOCK, ROWLOCK) WHERE Id = @id; -- lock it now
-- ...decide, then...
UPDATE Seats SET Status='Booked' WHERE Id = @id;
COMMIT;
```

(In PostgreSQL this is `SELECT ... FOR UPDATE`.)

Nobody can overwrite your work — but you're holding a lock, so others wait, and deadlocks become possible. Best for **hot rows that lots of people fight over**: booking the last seat, decrementing stock, updating a counter.

**Optimistic — "clashes are rare, so I'll just check at the end."** You lock nothing. Instead you add a `rowversion` column (SQL Server bumps it automatically on every change), remember its value when you read, and check it's still the same when you write:

```sql
UPDATE Products
SET Price = @newPrice, /* rowversion auto-updates */
WHERE Id = @id AND RowVer = @rowVerReadEarlier; -- 0 rows affected ⇒ someone else changed it
```

The clever part is the `AND RowVer = @rowVerReadEarlier`. If somebody else changed the row while you were thinking, its version no longer matches, so **your update matches zero rows**. That "0 rows affected" is your signal that you lost the race — you then tell the user, reload, or merge and retry.

Because you never hold a lock between reading and writing, this scales beautifully. Best for **normal web CRUD**, where two people editing the same record at the same second is unusual.

*Example:* Two admins open the same product page. The first saves a new price. When the second saves, their version value is stale, 0 rows update, and they get "this record was changed by someone else" instead of silently wiping out the first edit.

This is exactly what EF Core does with `[Timestamp]` / `IsConcurrencyToken` and `DbUpdateConcurrencyException`.

**Rule of thumb:** optimistic for data that's mostly read and rarely fought over; pessimistic for hot rows where losing the race and redoing the work would be worse than just waiting your turn.

---

## W7 — Transaction Control

### Q17. Show correct transaction control with error handling (BEGIN/COMMIT/ROLLBACK + TRY…CATCH + XACT_ABORT).

This is the pattern to memorise — it's the safe way to write any transaction in T-SQL:

```sql
SET XACT_ABORT ON; -- on any runtime error, auto-abort the whole transaction
BEGIN TRY
 BEGIN TRAN;
 UPDATE Accounts SET Balance = Balance - 100 WHERE Id='A';
 UPDATE Accounts SET Balance = Balance + 100 WHERE Id='B';
 COMMIT;
END TRY
BEGIN CATCH
 IF XACT_STATE() <> 0 -- there's an open/uncommittable transaction
 ROLLBACK;
 THROW; -- re-raise the original error to the caller
END CATCH;
```

What each piece is doing:

- **`SET XACT_ABORT ON`** — always use this. Without it, some errors only cancel the *one statement* that failed and leave your transaction sitting open, still holding locks. With it on, any error reliably kills the whole transaction. (It's already on automatically inside triggers.)

- **`XACT_STATE()`** — asks "is there a transaction, and can I still commit it?" It returns:
  - `1` = transaction open and fine to commit
  - `-1` = transaction open but doomed; rollback is your only option
  - `0` = no transaction at all

  You check it because calling `ROLLBACK` when there's nothing to roll back is itself an error.

- **`THROW`** — re-raises the original error so your application actually finds out it failed. Without this you'd silently swallow the problem and the caller would think everything worked.

### Q18. Does SQL Server support nested transactions? (The classic gotcha.)

**No. Despite how the syntax looks, SQL Server does not really have nested transactions** — and this catches people out constantly.

You *can* write `BEGIN TRAN` inside another `BEGIN TRAN`, and `@@TRANCOUNT` (a counter of how many you've opened) goes up each time. But the nesting is an illusion:

- **Inner `COMMIT`s don't commit anything.** They just decrease the counter by 1. Only the *outermost* `COMMIT` — the one that brings the counter back to 0 — actually saves your work.
- **Any plain `ROLLBACK` undoes everything.** It doesn't matter how deep you are; a `ROLLBACK` without a savepoint name throws away the entire transaction, all the way out, and resets the counter to 0.

```sql
BEGIN TRAN; -- @@TRANCOUNT = 1
 INSERT INTO T VALUES(1);
 BEGIN TRAN; -- @@TRANCOUNT = 2 ("nested")
 INSERT INTO T VALUES(2);
 ROLLBACK; -- @@TRANCOUNT = 0 — BOTH inserts rolled back!
COMMIT; -- ERROR 3902: no matching BEGIN TRANSACTION
```

Two safe habits:

1. **Check `@@TRANCOUNT` when the procedure starts** to see whether a transaction already exists, and use a **savepoint** rather than a bare `ROLLBACK` if it does.
2. **Better: don't manage transactions inside procedures at all.** Let whoever starts the transaction be the one who ends it.

> **PostgreSQL is different here** — it has genuine sub-transactions through savepoints (`SAVEPOINT` / `RELEASE` / `ROLLBACK TO`), so inner work really can be rolled back on its own. It doesn't have SQL Server's "the inner COMMIT is fake" trap.


**Why this bites in real life:** you write a stored procedure with its own `BEGIN TRAN` / `ROLLBACK`. Someone else calls it from inside *their* transaction. Your `ROLLBACK` silently destroys all of their work too — and then their `COMMIT` fails with error 3902.

Two safe habits:

1. **Check `@@TRANCOUNT` when the procedure starts** to see whether a transaction already exists, and use a **savepoint** rather than a bare `ROLLBACK` if it does.
2. **Better: don't manage transactions inside procedures at all.** Let whoever starts the transaction be the one who ends it.

> **PostgreSQL is different here** — it has genuine sub-transactions through savepoints (`SAVEPOINT` / `RELEASE` / `ROLLBACK TO`), so inner work really can be rolled back on its own. It doesn't have SQL Server's "the inner COMMIT is fake" trap.

### Q28. Why must transactions be kept short?

Because a transaction **holds on to its locks the entire time it's open** — it only lets go when you commit or roll back. Everything bad follows from that.

The longer it stays open:

- **More people wait on it.** Anyone needing those rows queues up behind you, and eventually starts timing out.
- **Deadlocks get more likely** — the longer you hold locks, the more chances there are to collide with someone.
- **The transaction log can't be cleaned up.** SQL Server can't truncate the log past the oldest transaction still running, so one forgotten transaction can grow your log file until the disk fills.
- **The version store keeps growing** if you're using RCSI or SNAPSHOT, putting pressure on `tempdb`.

**The rule:** never put application logic, API/HTTP calls, or anything waiting on a human inside a transaction. The worst version of this is a transaction left open while a user stares at a confirmation dialog — you've handed a person the power to block your database.

**Do it in this order:** read and validate everything *first*, then open the transaction, run just the inserts/updates, and commit straight away. Read outside, write inside.

---

