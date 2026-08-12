# Full-Stack (.NET + Angular + SQL) Scenario-Based Interview Questions & Solutions

This document contains real-world, scenario-based interview questions designed for Senior .NET, Angular, SQL, and Full-Stack Developer roles.

---

## 📌 Section 1: Core System & Architecture Scenarios (Original Questions - Expanded)

### 1. Email Verification Link Expiration
**Scenario:** Your application sends a verification email. The user clicks the link 15 minutes later and sees "Link has expired." How does the server know when the link should stop working?

**Answer:**

* **Stateless Token Approach (JWT / Signed Token):** The link contains an encrypted/signed token containing a payload with `UserId` and `ExpiresAt` (UTC timestamp). The server verifies the signature with its secret key and checks if `DateTime.UtcNow > token.ExpiresAt`. No DB lookup required unless token revocation is needed.
* **Stateful Database Approach:** The link contains a unique GUID (e.g., `https://api.myapp.com/verify?token=abc-123`). The database table `EmailVerificationTokens` stores `Token`, `UserId`, `CreatedAt`, `ExpiresAt`, `IsUsed`. When clicked, the server queries the database:
  ```sql
  SELECT * FROM EmailVerificationTokens WHERE Token = @Token AND IsUsed = 0 AND ExpiresAt > GETUTCDATE();
  ```
* **Security Best Practice:** Burn (invalidate) the token immediately upon first successful use to prevent replay attacks.

---

### 2. Resumable Large File Upload (5 GB Upload Interrupted at 2 GB)
**Scenario:** A user uploads a 5 GB file to file storage. After 2 GB, their internet connection drops. What happens, and how do you handle it without forcing the user to restart from 0 GB?

**Answer:**
* **What happens by default:** Standard multipart HTTP uploads fail completely; the server discards incomplete temporary buffers, and the user must re-upload from 0%.

* **Solution (Chunked & Resumable Uploads):**
  1. **Frontend (Angular):** Slice the 5 GB file into 5 MB to 10 MB chunks using the JavaScript `Blob.slice()` API.
  2. **Protocol (e.g., TUS Protocol / Azure Block Blobs):** 
     - Generate a unique `UploadSessionId` for the file.
     - Upload chunks sequentially or concurrently with chunk index/offset headers.
  3. **Interruption Recovery:** When network restores, Angular queries the server: `GET /upload/status?uploadId=123`.
  4. **Server Response:** Returns current offset (e.g., `Received: 2,147,483,648 bytes`).
  5. **Resume:** Angular resumes uploading starting from chunk index N at offset 2 GB.
  6. **Assembly:** Once all chunks arrive, the server verifies file MD5/SHA256 checksum and merges chunks.

---

### 3. Duplicate Rows Error in SQL Query with ORDER BY
**Scenario:** Why does the following query throw an error or behave unexpectedly?
```sql
SELECT DISTINCT user_id FROM orders ORDER BY created_at;
```

**Answer:**
* **Root Cause:** When `DISTINCT` is specified, SQL Server collapses multiple rows with the same `user_id` into a single result row. If a single `user_id` has multiple orders with different `created_at` values (e.g., Jan 1 and Feb 10), SQL Server does not know which `created_at` value to use to sort that unique `user_id`.

* **SQL Server Error:** `ORDER BY items must appear in the select list if SELECT DISTINCT is specified.`
* **Fix Options:**
  1. Include `created_at` in SELECT (note: this changes distinct grouping):
     ```sql
     SELECT DISTINCT user_id, created_at FROM orders ORDER BY created_at;
     ```
  2. Use `GROUP BY` with an aggregate function (Recommended):
     ```sql
     SELECT user_id FROM orders GROUP BY user_id ORDER BY MIN(created_at);
     ```

---

### 4. Production Debugging: CPU at 100%, Stable Memory, Idle Disk & Network
**Scenario:** In production, CPU hits 100%. Memory is stable, network is normal, disk is idle, and only one service is affected. Where do you start debugging?

**Answer:**
Since **only one service** is affected and CPU is at **100%**, the problem is likely inside that service (e.g., an infinite loop, expensive computation, excessive requests, or thread contention).

### Debugging Steps

1. **Check application logs**
   - Look for exceptions, retries, or unusual activity.

2. **Monitor CPU usage**
   - Use tools like `dotnet-counters`, `dotnet-trace`, or `dotnet-monitor` to identify hot methods and CPU-intensive operations.

3. **Capture a CPU profile**
   - Analyze which methods consume the most CPU.
   - Look for infinite loops, recursive calls, or inefficient algorithms.

4. **Check thread activity**
   - Capture a process dump (`dotnet-dump`) and inspect thread stacks.
   - Look for busy loops, deadlocks, or thread contention.

5. **Review recent deployments**
   - Determine whether a recent code change introduced the issue.

6. **Check database and external API calls**
   - Ensure the service isn't repeatedly retrying failed operations or making excessive requests.

7. **Review metrics**
   - Examine request rate, response time, error rate, and dependency latency to identify abnormal behavior.


### .NET Diagnostic Tools

- **`dotnet-counters`** – Shows **live metrics** like CPU, memory, GC, and thread pool usage.
  ```bash
  dotnet-counters monitor -p <PID>
  ```

- **`dotnet-trace`** – Collects a **performance trace** to identify CPU-intensive or slow methods.
  ```bash
  dotnet-trace collect -p <PID>
  ```

- **`dotnet-monitor`** – Collects **logs, metrics, traces, and dumps** from a running application, commonly used in production.
  ```bash
  dotnet-monitor
  ```

- **`dotnet-dump`** – Captures a **process dump** for offline analysis of crashes, hangs, deadlocks, or high CPU.
  ```bash
  dotnet-dump collect -p <PID>
  ```


---

### 5. Securing a Public-Facing Web API
**Scenario:** You have built a REST API for enterprise clients. How do you secure it end-to-end?

**Answer:**

* **Authentication & Authorization:** OAuth2.0 / OpenID Connect (OIDC) using Azure AD, Keycloak, or Duende IdentityServer. Issue short-lived JWT Access Tokens validated via middleware (`AddJwtBearer`).
* **Transport Security:** Mandatory HTTPS/TLS 1.3 with HSTS headers. Enforce TLS cipher suites.
* **Rate Limiting & Throttling:** ASP.NET Core RateLimiting middleware (Fixed window, Sliding window, Token bucket per IP or Client ID) to prevent DoS attacks.
* **Input Validation & Sanitization:** FluentValidation + DTO binding. Never trust user input (protect against SQL Injection, XSS, Command Injection).
* **CORS Policy:** Strict origin restriction (do not use `AllowAnyOrigin()` with credentials).
* **API Gateway & Firewall:** Web Application Firewall (WAF e.g., Azure Front Door / Cloudflare) for DDoS protection, OWASP top 10 filtering, and IP whitelisting.
* **Security Headers:** CSP, X-Frame-Options, X-Content-Type-Options.

---

### 6. Making a Single Page Application (SPA) Faster
**Scenario:** How would you improve the performance and initial load time of a large Angular SPA?

**Answer:**

* **Bundle Optimization:**
  - Route-level Lazy Loading using `loadComponent: () => import(...)`.
  - Tree-shaking unused libraries.
  - Differential loading and modern ES bundle output.
* **Rendering & State Performance:**
  - Change Detection: Set `ChangeDetectionStrategy.OnPush` on all UI components.
  - Use Angular Signals (Angular 16+) for fine-grained reactivity without Zone.js overhead.
  - TrackBy in `*ngFor` / New `@for` syntax with `track`.
* **Network & Asset Loading:**
  - Server-Side Rendering (SSR) & Angular Hydration.
  - Image optimization (`NgOptimizedImage` directive for priority loading & WebP formatting).
  - HTTP Caching, CDN distribution, and Service Workers / PWA caching.
  - Preloading critical modules using custom preloading strategies.

---

## 📌 Section 2: .NET Core & Backend Developer Scenarios

### 7. Dependency Injection Lifetime Mismatch (Captive Dependency Bug)
**Scenario:** A developer injects a `Scoped` service (like EF Core `DbContext`) into a `Singleton` background service (`IHostedService`). What goes wrong at runtime, and how do you fix it?

**Answer:**

* **Problem (Captive Dependency):** The `Singleton` service is instantiated once when the app starts and lives forever. Because it holds a reference to the `Scoped` `DbContext`, the `DbContext` never gets disposed.
* **Consequences:**
  1. **Thread Safety Crashes:** `DbContext` is not thread-safe. Concurrent requests accessing the singleton will throw `InvalidOperationException: A second operation was started on this context before a previous operation completed`.
  2. **Memory Leak & Stale Data:** Change tracker accumulates entities indefinitely, inflating RAM and returning stale entities.
* **Detection:** ASP.NET Core DI automatically throws an exception in Development mode if `ValidateScopes = true` is enabled in `Program.cs`.
* **Fix:** Inject `IServiceScopeFactory` into the Singleton, and create a scope on-demand:
  ```csharp
  public class MyBackgroundWorker : BackgroundService
  {
      private readonly IServiceScopeFactory _scopeFactory;

      public MyBackgroundWorker(IServiceScopeFactory scopeFactory)
      {
          _scopeFactory = scopeFactory;
      }

      protected override async Task ExecuteAsync(CancellationToken stoppingToken)
      {
          using (var scope = _scopeFactory.CreateScope())
          {
              var dbContext = scope.ServiceProvider.GetRequiredService<MyDbContext>();
              await dbContext.Orders.ToListAsync(stoppingToken);
          }
      }
  }
  ```

---

### 8. ThreadPool Starvation & Sync-Over-Async Deadlocks
**Scenario:** An ASP.NET Core Web API under heavy load experiences slow response times and eventual timeouts, but CPU utilization is low (~15%). Logs show many threads blocked on `.Result` or `.Wait()`. What is happening?

**Answer:**

* **Root Cause:** "Sync-over-async" blocking (`Task.Result` or `Task.Wait()`).
* **Mechanism:** When a web request thread hits `.Result`, the underlying async work is queued to the ThreadPool. The calling thread blocks waiting for completion. Under high concurrency, available ThreadPool threads get exhausted (starvation). The ThreadPool slowly injects new threads (1-2 threads per second), causing incoming requests to queue up and time out.
* **Fix:**
  1. Refactor code to be asynchronous all the way down (`async` / `await`).
  2. Never call `.Result`, `.Wait()`, or `Task.Run().Result` inside web handlers.
  3. If integrating legacy sync code, use async wrappers cleanly or configure `ThreadPool.SetMinThreads(100, 100)` as a temporary emergency workaround while refactoring.

---

### 9. Entity Framework Core N+1 Query & Change Tracker Bottleneck
**Scenario:** An API endpoint fetching 1,000 orders with their associated customer details takes 6 seconds and executes 1,001 SQL queries. Memory usage spikes during the request. How do you resolve both issues?

**Answer:**

* **Issue 1 (N+1 Query):** Occurs due to lazy loading or accessing `order.Customer` inside a loop without eager loading. 1 query fetches 1,000 orders, then 1,000 individual queries fetch each customer.
* **Issue 2 (Change Tracker Overhead):** EF tracks all 1,000 entities in memory by default, taking CPU and RAM.
* **Fix (Eager Loading + Projections + AsNoTracking):**
  ```csharp
  // Fast & efficient single SQL JOIN query without tracking overhead
  var result = await _context.Orders
      .AsNoTracking()
      .Where(o => o.Status == OrderStatus.Completed)
      .Select(o => new OrderDto
      {
          OrderId = o.Id,
          TotalAmount = o.TotalAmount,
          CustomerName = o.Customer.Name,
          CustomerEmail = o.Customer.Email
      })
      .ToListAsync();
  ```

---

## 📌 Section 3: Angular & Frontend Developer Scenarios

### 10. Memory Leaks from RxJS Observables
**Scenario:** Users navigating between pages in your Angular SPA report that the app gets progressively slower over an 8-hour shift. Browser DevTools show memory growing from 100 MB to 1.8 GB with thousands of detached DOM nodes. Why?

**Answer:**

* **Root Cause:** Subscriptions to long-lived Observables (e.g., `HttpClient`, `Router.events`, `NgRx store`, `interval()`, `Subject`) inside components that are NOT unsubscribed when the component is destroyed.
* **Why it leaks:** The active subscription holds a reference to the component instance in memory, preventing garbage collection even after the component is removed from the DOM.
* **Modern Fixes (Angular 16+ Best Practices):**
  1. **Use `takeUntilDestroyed()`:**
     ```typescript
     private userService = inject(UserService);
     
     ngOnInit() {
       this.userService.getUpdates()
         .pipe(takeUntilDestroyed(this.destroyRef))
         .subscribe(data => this.data = data);
     }
     ```
  2. **Use the `async` pipe in templates:** (Angular handles subscription & unsubscription automatically).
  3. **Use Signals:** (`toSignal(observable$)` converts observables and manages lifetime automatically).

---

### 11. Race Condition in Live Search / Auto-Suggest Input
**Scenario:** A user types "Angular" in a search input. The UI briefly displays results for "Angular", but then replaces them with results for "Ang". What caused this bug and how do you fix it using RxJS?

**Answer:**

* **Root Cause:** Network latency variation. The HTTP request for "Ang" was sent first, but took 800ms. The request for "Angular" was sent second, but took 200ms. The first request finished *last*, overwriting the state with outdated results.
* **Fix (RxJS `switchMap` + `debounceTime` + `distinctUntilChanged`):**
  ```typescript
  searchControl.valueChanges.pipe(
    debounceTime(300),                   // Wait 300ms after last keystroke
    distinctUntilChanged(),              // Ignore if query hasn't changed
    switchMap(query =>                   // Cancels previous pending HTTP request!
      this.searchService.search(query)
    )
  ).subscribe(results => {
    this.searchResults = results;
  });
  ```

---

### 12. Angular Heavy Change Detection Lag on Form Inputs
**Scenario:** Typing a single character in a text box inside a complex dashboard component causes a 200ms lag. DevTools Performance tab shows Angular running Change Detection across 500 component instances per keypress. How do you optimize this?

**Answer:**

* **Root Cause:** Angular's default Zone.js change detection runs top-down across the entire component tree on every browser event (keypress, click, setTimeout).
* **Fix Strategy:**
  1. **Switch to `OnPush` Change Detection:**
     ```typescript
     @Component({
       selector: 'app-dashboard',
       changeDetection: ChangeDetectionStrategy.OnPush
     })
     ```
  2. **Never call functions in templates:** Replace `<div>{{ calculateComplexSummary(item) }}</div>` with precomputed property values or a `@Pipe({ pure: true })`.
  3. **Use Angular Signals:** Signals notify Angular of exact dependencies, triggering pinpoint DOM updates without re-checking unchanged components.

---

## 📌 Section 4: SQL & Database Scenarios

### 13. Resolving SQL Server Deadlocks in Production
**Scenario:** Production error logs frequently report: `Transaction (Process ID 64) was deadlocked on lock resources with another process and has been chosen as the deadlock victim. Rerun the transaction.` How do you investigate and fix this?

**Answer:**

* **Understanding Deadlocks:** Process A holds Lock X and requests Lock Y. Process B holds Lock Y and requests Lock X. Neither can proceed.
* **Investigation Steps:**
  1. Capture deadlock graph using SQL Server Extended Events (`xml_deadlock_report` event).
  2. Identify the two SQL statements, index names, and lock types (X = Exclusive, S = Shared).
* **Common Fixes:**
  1. **Consistent Access Order:** Ensure all services update tables in the exact same sequence (e.g., always update `Orders` before `Inventory`, never vice-versa).
  2. **Keep Transactions Short:** Move read queries outside transactions; select only required columns.
  3. **Enable Read Committed Snapshot Isolation (RCSI):**
     ```sql
     ALTER DATABASE MyDb SET READ_COMMITTED_SNAPSHOT ON WITH ROLLBACK IMMEDIATE;
     ```
     This allows readers to read row versions without acquiring Shared (S) locks, eliminating read-write deadlocks.
  4. **Use Covering Indexes:** Prevent Bookmark Lookups during updates.

---

### 14. Deep Pagination Performance Bottleneck
**Scenario:** A report query `SELECT * FROM AuditLogs ORDER BY CreatedDate DESC OFFSET 1000000 ROWS FETCH NEXT 50 ROWS ONLY;` takes 14 seconds to execute on a 50 Million row table. Why is it slow, and how do you optimize it?

**Answer:**

* **Root Cause:** SQL Server must scan the index, read, and discard 1,000,000 rows in memory before returning row 1,000,001 to 1,000,050. Performance degrades linearly ($O(N)$) as offset increases.
* **Optimization (Seek-Based / Keyset Pagination):**
  Instead of `OFFSET`, remember the last `Id` or `CreatedDate` from the previous page:
  ```sql
  -- Fast seek query using index ($O(1)$ complexity)
  SELECT TOP 50 *
  FROM AuditLogs
  WHERE CreatedDate < @LastSeenCreatedDate
  ORDER BY CreatedDate DESC;
  ```

---

### 15. High Write Latency on Table with 100M+ Rows
**Scenario:** Inserting a single record into the `Transactions` table takes 4 seconds during peak write hours. Disk I/O is high. What is causing this and how do you fix it?

**Answer:**
High **Disk I/O** during inserts usually means SQL Server is doing extra work besides writing the new row.

### Possible Causes

- Too many indexes on the table (every insert updates all indexes).
- Locking or blocking from other transactions.
- Triggers executing additional work.
- Slow storage or transaction log bottleneck.
- Large or fragmented indexes.

### How to Fix

1. Check for **blocking/locks** (`sp_who2`, Activity Monitor, DMVs).
2. Review and remove **unused or excessive indexes**.
3. Optimize or remove expensive **triggers**.
4. Check **transaction log** and storage performance; ensure log file isn't auto-growing frequently.
5. Rebuild/reorganize fragmented indexes if necessary.


## Large Table, Few Indexes, Inserts Still Slow

### Check for Blocking

The insert may be waiting for another transaction holding a lock.

```sql
EXEC sp_who2;
```

or query the DMVs to identify blocking sessions.

### Check Transaction Log

Every insert is written to the transaction log first.

If the log disk is slow or the log file is frequently auto-growing, inserts become slow.

### Check Disk Performance

High disk I/O may indicate:

- Slow storage
- Storage latency
- Saturated disk

Use Performance Monitor or SQL Server wait statistics.

### Check Wait Statistics

Find what SQL Server is waiting on.

```sql
SELECT *
FROM sys.dm_os_wait_stats;
```

---

## 📌 Section 5: Full-Stack & Integration Scenarios

### 16. Secure Token Authentication Architecture (JWT + Refresh Token Rotation)
**Scenario:** Design a secure authentication mechanism for an Angular frontend consuming a .NET Core API that satisfies:
- User stays logged in seamlessly without typing credentials every 15 mins.
- Protects against XSS and CSRF token theft.

**Answer:**

* **Architecture:**
  1. **Access Token (Short-lived - 15 mins):** Stored in Angular memory (Service variable or RxJS state). Sent in HTTP header: `Authorization: Bearer <token>`.
  2. **Refresh Token (Long-lived - 7 days):** Stored in an `HttpOnly`, `Secure`, `SameSite=Strict` cookie set by the server. JavaScript cannot access this cookie (protects from XSS).
* **Refresh Flow in Angular:**
  - An Angular `HttpInterceptor` catches `401 Unauthorized` responses.
  - Interceptor pauses pending requests and calls `POST /api/auth/refresh` (cookie sent automatically by browser).
  - API validates refresh token, issues new Access Token + new Refresh Token (**Refresh Token Rotation**).
  - Interceptor retries original failed request with the new Access Token.

---

### 17. Concurrent Editing & Lost Updates (Optimistic Concurrency Control)
**Scenario:** Two users open Customer #500 in Angular simultaneously. User A edits Phone Number and saves at 10:00:01 AM. User B edits Address and saves at 10:00:02 AM. User A's changes are overwritten and lost. How do you prevent this?

**Answer:**

* **Solution (Optimistic Concurrency Control):**
  1. **Database / EF Core Model:** Add a `RowVersion` timestamp column:
     ```csharp
     public class Customer
     {
         public int Id { get; set; }
         public string Phone { get; set; }
         public string Address { get; set; }
         
         [Timestamp]
         public byte[] RowVersion { get; set; }
     }
     ```
  2. **Update Execution:** EF Core automatically generates SQL:
     ```sql
     UPDATE Customers SET Phone = @Phone WHERE Id = @Id AND RowVersion = @OriginalRowVersion;
     ```
  3. **Handling Conflict:**
     - If User B tries to save with an outdated `RowVersion`, 0 rows match the `WHERE` clause.
     - EF Core throws `DbUpdateConcurrencyException`.
  4. **Frontend Experience (Angular):** API returns `409 Conflict`. Angular displays a visual diff showing User A's changes alongside User B's changes, allowing User B to merge or overwrite explicitly.

---

### 18. Real-Time Dashboard: WebSockets (SignalR) vs SSE vs Polling
**Scenario:** You need to build a live financial ticker dashboard in Angular backed by .NET Core. Updates occur every 500ms for 10,000 connected users. Which tech stack do you choose and why?

**Answer:**

* **Comparison:**
  * **Short Polling:** HTTP requests every 500ms $\rightarrow$ Severe server overload (10,000 users * 2 req/sec = 20,000 req/sec HTTP overhead). **Unusable.**
  * **Server-Sent Events (SSE):** HTTP persistent connection, unidirectional (server to client). Lightweight, but binary streaming is complex.
  * **WebSockets / ASP.NET Core SignalR (Recommended):** Full-duplex persistent connection with fallback transports (WebSockets, Server-Sent Events, Long Polling).
* **Scalability Solution for SignalR:**
  - Scale out .NET Core backend nodes behind a Load Balancer using **Azure SignalR Service** or **Redis Backplane** to sync state across nodes.
