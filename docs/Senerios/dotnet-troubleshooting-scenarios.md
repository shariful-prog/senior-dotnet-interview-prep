# .NET & SQL Troubleshooting Scenarios (Real-World Performance & Diagnostics)
---

> **Overview:** This document contains practical, real-world troubleshooting scenarios encountered in senior .NET and SQL production environments. Each scenario covers root cause analysis, diagnostic tooling (`dotnet-trace`, `dotnet-dump`, SQL Deadlock Graphs, Application Insights), and concrete technical solutions.

---

### Scenario 1: Investigating High CPU Spikes (99% CPU) in ASP.NET Core

**Problem Statement:** Your ASP.NET Core API CPU utilization suddenly jumps to 99% in production under normal traffic load. The app remains responsive to some requests but latency spikes dramatically. How do you locate the exact method causing the high CPU without restarting the application?

#### Diagnostic & Troubleshooting Steps:
1. **Capture a CPU Trace (Non-destructive):** Use `dotnet-trace` to capture a performance trace from the running process without stopping it:
   ```bash
   dotnet-trace collect -p <PID> --duration 00:00:30 --providers Microsoft-Windows-DotNETRuntime
   ```
2. **Analyze Hot Paths:** Open the trace file in **SpeedScope** (speedscope.app) or **PerfView**. Look at the Flame Graph to identify which call stack consumes the highest percentage of CPU time.
3. **Common Root Causes:**
   - **Regex Catastrophic Backtracking**: Evaluating complex regular expressions against untrusted user inputs (`(a+)+`).
   - **Infinite Loops / Busy Wait**: `while(true)` loops lacking `await Task.Delay()` or proper cancellation tokens.
   - **In-Memory LINQ Sorting**: Calling `.ToList()` before `.OrderBy()` on a 500,000 item collection in memory instead of offloading to SQL.
   - **JSON Serialization of Huge Graphs**: Serializing deeply nested object trees containing circular references or millions of properties.

#### Solution Code Fix:
Offload sorting/filtering to SQL, fix regex patterns with timeouts (`TimeSpan.FromSeconds(1)`), or replace blocking loops with async channels/queues.

---

### Scenario 2: Diagnosing Memory Leaks & Large Object Heap (LOH) Growth

**Problem Statement:** The memory footprint of your ASP.NET Core container steadily grows from 500 MB to 6 GB over 24 hours until the container is terminated by the Kubernetes Out-Of-Memory (OOM) killer.

#### Diagnostic & Troubleshooting Steps:
1. **Trigger Memory Dump:** Capture a core dump when memory reaches 80% threshold:
   ```bash
   dotnet-dump collect -p <PID> --type Full
   ```
2. **Analyze Dump with `dotnet-dump`:**
   ```bash
   dotnet-dump analyze coredump.dmp
   # Run dumpheap to view objects consuming the most memory:
   > dumpheap -stat
   # Run gcroot to trace why a specific object is not garbage collected:
   > gcroot <Object_Address>
   ```
3. **Common Root Causes:**
   - **Static Event Handler Leaks**: Long-lived singletons subscribing to short-lived component events without unsubscribing.
   - **Static Dictionary / Cache Leak**: Adding items to a static `ConcurrentDictionary` without expiration or removal logic.
   - **Large Object Heap (LOH) Fragmentation**: Frequently allocating byte arrays or strings $> 85,000$ bytes (e.g. reading entire file uploads into `byte[]` memory buffers instead of streaming via `Stream`).

#### Solution Code Fix:
Use `IMemoryCache` with `SetAbsoluteExpiration()`, stream files using `Stream` / `PipeReader`, or implement `IDisposable` to unhook event handlers.

---

### Scenario 3: Thread Pool Starvation & Intermittent Request Timeouts

**Problem Statement:** Under peak concurrent load, users experience sudden HTTP 504 timeouts. CPU utilization is low (20-30%), but incoming HTTP request queues keep growing.

#### Root Cause:
**Sync-over-Async Blocking Calls** (e.g. `task.Result`, `task.Wait()`, `Task.Run().Result`). When a thread blocks waiting for an async task, it surrenders execution. Under heavy load, all ThreadPool worker threads become blocked. Because the .NET ThreadPool injects new threads slowly (typically 1–2 threads per second), request processing starves.

#### Solution Code Fix:
1. **Remove All Sync-Over-Async Calls:** Replace `.Result` and `.Wait()` with `await` end-to-end:
   ```csharp
   // ❌ Bad: Causes ThreadPool Starvation
   var data = _repository.GetDataAsync().Result;

   // ✅ Good: Non-blocking async
   var data = await _repository.GetDataAsync();
   ```
2. **Emergency Mitigation:** Increase minimum ThreadPool worker threads in `Program.cs`:
   ```csharp
   ThreadPool.SetMinThreads(workerThreads: 200, completionPortThreads: 200);
   ```

---

### Scenario 4: SQL Parameter Sniffing & Performance Degradation

**Problem Statement:** A SQL query executes in 30ms for 99% of requests, but occasionally hangs for 45 seconds when called with specific user parameters.

#### Root Cause:
**Parameter Sniffing**. When SQL Server compiles a stored procedure or parameterized query for the *first* time, it creates an execution plan optimized for that initial parameter value. If the initial parameter matched 1 row, SQL Server chooses an Index Seek. If a subsequent parameter matches 1,000,000 rows, SQL Server reuses the Index Seek plan, causing millions of random I/O reads instead of a Table Scan.

#### Solution & SQL Fixes:
1. **Use `OPTIMIZE FOR UNKNOWN` or `RECOMPILE`:**
   ```sql
   -- Force SQL Server to use average distribution statistics
   SELECT * FROM Orders WHERE Status = @Status
   OPTION (OPTIMIZE FOR UNKNOWN);
   ```
2. **Assign Parameters to Local Variables:**
   ```sql
   CREATE PROCEDURE GetOrdersByStatus @Status VARCHAR(20)
   AS
   BEGIN
       DECLARE @LocalStatus VARCHAR(20) = @Status;
       SELECT * FROM Orders WHERE Status = @LocalStatus;
   END
   ```

---

### Scenario 5: Database Connection Pool Exhaustion

**Problem Statement:** Your application logs show `System.InvalidOperationException: Timeout expired. The timeout elapsed prior to obtaining a connection from the pool.`

#### Root Cause:
ADO.NET connection pool depletion caused by leaked or long-lived database connections.

#### Common Causes & Fixes:
1. **`DbContext` Lifetime Mismatch**: Registering `DbContext` as Singleton or creating raw `SqlConnection` without `using` statements.
   ```csharp
   // ❌ Bad: Leaks connection if exception occurs
   var conn = new SqlConnection(connStr);
   conn.Open();
   // missing conn.Dispose()

   // ✅ Good: Automatic disposal
   await using var conn = new SqlConnection(connStr);
   await conn.OpenAsync();
   ```
2. **Long-Running HTTP Transactions**: Keeping an open DB transaction while calling an external third-party API inside a controller action.

---

### Scenario 6: Troubleshooting SQL Deadlocks in High-Concurrency Systems

**Problem Statement:** Users intermittently receive HTTP 500 errors. Database logs reveal `SqlException: Transaction (Process ID 54) was deadlocked on lock resources with another process and has been chosen as the deadlock victim.`

#### Diagnostic & Fix Strategy:
1. **Capture Deadlock Graph:** Enable Extended Events in SQL Server to capture the XML Deadlock Graph.
2. **Identify Access Sequence Mismatch:**
   - Transaction A locks `Table X` then requests `Table Y`.
   - Transaction B locks `Table Y` then requests `Table X`.
3. **Fixes:**
   - **Enforce Strict Table Access Order**: Update both queries/transactions to access `Table X` first, then `Table Y`.
   - **Enable Read Committed Snapshot Isolation (RCSI)**: Prevents read locks from blocking write locks in SQL Server:
     ```sql
     ALTER DATABASE [MyDatabase] SET READ_COMMITTED_SNAPSHOT ON WITH ROLLBACK IMMEDIATE;
     ```
   - **Implement Polly Retry Policy**: Catch `SqlException` error code 1205 and retry with exponential backoff.

---

### Scenario 7: Resolving EF Core N+1 Queries & Slow Serialization

**Problem Statement:** An API endpoint returning a list of 100 Orders takes 6 seconds to respond. SQL Profiler shows 101 queries executed for a single HTTP request.

#### Root Cause:
**N+1 Query Problem**. Querying parent entities (`Orders`) and accessing un-fetched navigation properties (`Order.Customer`) inside a `foreach` loop triggers 1 query for the parent list + 100 separate queries for each customer.

#### Solution Code Fix:
```csharp
// ❌ Bad: Triggers 101 database roundtrips
var orders = await _context.Orders.ToListAsync();
foreach (var o in orders) {
    var name = o.Customer.Name; // Fires query per iteration!
}

// ✅ Good: Eager Loading with Include (1 Single SQL JOIN Query)
var orders = await _context.Orders
    .Include(o => o.Customer)
    .AsNoTracking()
    .Select(o => new OrderDto {
        OrderId = o.Id,
        CustomerName = o.Customer.Name
    })
    .ToListAsync();
```

---

### Scenario 8: Cascading Failures from Third-Party API Timeouts

**Problem Statement:** Your application calls an external Payment Gateway API. When the Payment Gateway experiences 10-second latencies, your entire application crashes, causing timeouts across unrelated features.

#### Root Cause:
All application HTTP worker threads are blocked waiting for external API HTTP calls to finish, exhausting sockets and thread pool workers.

#### Solution Code Fix (Resilience with Polly):
Use **Polly Circuit Breaker** and **Timeout** policies via `IHttpClientFactory`:

```csharp
builder.Services.AddHttpClient("PaymentGateway", client =>
{
    client.BaseAddress = new Uri("https://api.payments.com");
    client.Timeout = TimeSpan.FromSeconds(3); // Hard 3s timeout
})
.AddTransientHttpErrorPolicy(policy => policy.CircuitBreakerAsync(
    handledEventsAllowedBeforeBreaking: 5,
    durationOfBreak: TimeSpan.FromSeconds(30) // Trip circuit for 30s after 5 failures
));
```

---

### Scenario 9: App Is Fast Locally but Times Out in Staging/Production

**Problem Statement:** An endpoint executes in 50ms on a developer machine but times out (30s+) in Staging.

#### Root Causes & Investigation Checklist:
1. **Data Volume Discrepancy**: Local database has 500 rows (Index Scan takes 1ms). Staging database has 10,000,000 rows (Index Scan causes full disk scan). Check missing indexes using Execution Plan in Staging.
2. **Network Latency / Geographic Separation**: App server and DB server are in different cloud regions or availability zones.
3. **Database Parameter Collation**: Staging database uses case-sensitive collation, forcing full string scans on `WHERE LOWER(Email) = ...` queries.

---

### Scenario 10: Zero-Downtime Deployment & DB Migration Lock Contention

**Problem Statement:** Running EF Core `context.Database.Migrate()` automatically at application startup during CI/CD deployment causes database table locks, resulting in HTTP 500 errors for live users.

#### Solution (Expand-Contract Migration Strategy):
1. **Never drop or rename columns directly in live production databases.**
2. **Phase 1 (Expand)**: Add new nullable columns or new tables in Database Migration. Deploy backend code that writes to *both* old and new columns.
3. **Phase 2 (Data Backfill)**: Run asynchronous background job to backfill old data into new columns.
4. **Phase 3 (Contract)**: Deploy backend code reading exclusively from new columns. In a subsequent release, drop old unused columns.
