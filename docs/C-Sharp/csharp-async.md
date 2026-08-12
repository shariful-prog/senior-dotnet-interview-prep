# F. Asynchrony, Threading & Concurrency
---

## F1 — async/await Model

### Q1. What is  `async` and `await`? Is it multithreading?

**Answer.** Two keywords that work together:

- **`async`** is a marker you put on a method. It tells the compiler this method is allowed to use `await`, and it lets the compiler rewrite the method so it can pause and resume (Q2). On its own, `async` does nothing at runtime.
- **`await`** is where the work happens. It says: start this operation, and if it isn't finished yet, **pause here and give the thread back**. When the operation completes, carry on from this line.

An `async` method returns a `Task` (or `Task<T>`), which represents work that will finish later.

```csharp
public async Task<int> GetCountAsync()
{
    var rows = await _db.QueryAsync("SELECT ...");   // pause here, thread is freed
    return rows.Count;                                // resumes here when the data arrives
}
```

**Is it multithreading? No.** They solve different problems:

- **Multithreading** = doing several things at the same time, on several threads.
- **`async`/`await`** = not holding a thread while waiting for something slow to finish.

Take a web server querying a database. **Without async**, the thread calls the database and waits, doing nothing, for tens of milliseconds. Under load you run out of threads and requests queue up. **With async**, the thread is handed back at the `await` and serves other requests until the result arrives.

So `await` does **not** start a new thread. While waiting on I/O, often **no thread is used at all** — the operating system tells you when the data is ready. Threads only come in for **CPU-heavy work** you deliberately move off with `Task.Run` (F2).

The payoff is throughput, not speed: async doesn't make one call faster, it lets the app handle more requests with fewer threads.

---

### Q2. What does the compiler do with an `async` method?

**Answer.** The compiler **rewrites your method** so it can **pause and resume**. Your straightforward code with `await` points gets turned into a behind-the-scenes structure (called a *state machine*) that remembers where it paused and can continue from that exact spot later.

You don't need to memorize the generated code — just understand the *idea*:

```csharp
async Task<int> M()
{
    int a = await FooAsync();   // ← can pause here
    return a + 1;               // ← resumes here later
}
```

When the method hits an `await`, one of two things happens:
- **If the thing is already done** (fast path): it just keeps running, no pause at all.
- **If it's not done yet:** the method **saves its place, returns control to the caller**, and registers a "call me back when finished" note. When the awaited work completes, the method resumes from where it paused.

Each `await` is basically a **bookmark** where the method can pause and later continue. That's the whole model.

---

### Q3. What can you `await`?

**Answer.** You can `await` anything that follows the "awaitable" pattern — most commonly a **`Task`**, **`Task<T>`**, **`ValueTask`**, or **`ValueTask<T>`**. You can even make your own types awaitable.

The technical requirement is that the type has a `GetAwaiter()` method (with `IsCompleted`, `OnCompleted`, and `GetResult`), but in practice you'll almost always be awaiting Tasks that framework methods hand back to you.

```csharp
await SomeMethodThatReturnsTask();
int x = await SomeMethodThatReturnsTaskOfInt();
```

The takeaway: `await` isn't tied to one specific type — it works with anything that "knows how to be awaited." That flexibility is what let Microsoft add `ValueTask` later without changing the language.

---

### Q4. Why is `async void` dangerous, and when is it acceptable?

**Answer.** `async void` (an async method that returns nothing) is a trap for a few reasons:

1. **You can't await it.** The caller gets nothing back, so it can't wait for it to finish or know when it's done.
2. **Exceptions crash your app.** In a normal `async Task` method, an error is stored on the returned Task and surfaces safely when you await it. In an `async void` method, there's no Task to hold the error — so it gets thrown "into the void" and usually **crashes the whole process**. You can't even `try/catch` it from the caller.

```csharp
async void FireAndForget()      // BAD
{
    await Task.Delay(100);
    throw new Exception("boom"); // → crashes the process
}

async Task Safe()               // GOOD
{
    await Task.Delay(100);
    throw new Exception("boom"); // → safely stored on the returned Task
}
```

**The one valid use** is **event handlers**, which are *required* to return `void` (like a button click handler): `async void Button_Click(...)`. Even then, wrap the body in a `try/catch`. For everything else, **return `Task`**.

---

### Q5. How do exceptions flow through `async`/`await`?

**Answer.** When an `async Task` method throws, the error doesn't blow up immediately. Instead it's **stored on the Task**, and it surfaces **when you `await` that Task** — at which point `await` re-throws it, keeping the original error details intact.

```csharp
async Task DoAsync()
{
    throw new Exception("boom");   // stored on the Task, not thrown right now
}

try
{
    await DoAsync();               // ← the exception surfaces HERE
}
catch (Exception ex) { /* handle it */ }
```

One nuance for later: if you run several tasks together with `Task.WhenAll` and more than one fails, **`await` only re-throws the first** error. If you need to see all of them, look at `task.Exception` directly.

```csharp
var t = Task.WhenAll(FailAAsync(), FailBAsync());
try { await t; }
catch
{
    foreach (var ex in t.Exception!.InnerExceptions)   // see ALL failures
        Log(ex);
}
```

This clean behavior is one reason to prefer `await` over blocking calls like `.Result` (which wrap errors in a messier way). See [exceptions](csharp-exceptions.md) for more.

---

### Q6. Does `await` always pause the method?

**Answer.** No. `await` only pauses if the thing you're awaiting **isn't already finished**. If it's already done, the method just keeps running on the same thread with **no pause and no overhead**. This is called the **fast path**.

This matters because lots of async operations often finish instantly — for example, a value that's already in a cache. In those cases `await` costs almost nothing.

```csharp
async ValueTask<int> GetAsync(string key)
{
    if (_cache.TryGetValue(key, out var v))
        return v;                        // already have it — no pause, basically free

    return await LoadFromDbAsync(key);   // only pauses when we actually hit the DB
}
```

This "usually already done" case is exactly why `ValueTask` exists (see F2) — and why sprinkling `async` everywhere is cheap when your operations usually hit a cache.

---

## F2 — Task, ValueTask, Task.Run, WhenAll/WhenAny

### Q1. What is a `Task`?

**Answer.** A `Task` represents an **asynchronous operation** that may complete in the future. It is used with `async` and `await` to perform non-blocking operations.

```csharp
Task<int> t = ComputeAsync();   // already running
int result = await t;           // just wait for it to finish
```

There are handy shortcuts for results you already have: `Task.CompletedTask` (an already-finished task), `Task.FromResult(value)`, etc.

---

### Q2. `Task` vs `ValueTask` — when and why use `ValueTask`?

**Answer.** `Task<T>` is a class, so creating one **allocates memory** every time. `ValueTask<T>` is a lightweight **struct** that can hold either an immediate result *or* a real Task. When a method **usually finishes instantly** (like a cache hit), returning `ValueTask<T>` **skips that memory allocation** on the fast path — which reduces pressure on the garbage collector in hot code.

```csharp
public ValueTask<int> GetAsync(string key)
{
    if (_cache.TryGetValue(key, out var v))
        return new ValueTask<int>(v);           // no allocation — instant result
    return new ValueTask<int>(LoadAsync(key));  // slow path — wraps a real Task
}
```

But `ValueTask` has strict rules — only use it when you've measured that the allocation actually matters:
- **Await it exactly once.** Awaiting it twice causes undefined behavior.
- **Don't block on it, don't store it, don't use it with `WhenAll`.** If you need any of that, call `.AsTask()` to convert it first.

**Default advice:** return **`Task`** in normal code. Only reach for `ValueTask` on a proven hot path.

---

### Q3. What does `Task.Run` do, and when should you (not) use it?

**Answer.** `Task.Run` **pushes work onto a background (thread pool) thread**. Its real purpose is to move **CPU-heavy work** off the current thread — for example, keeping a desktop app's UI responsive while a big calculation runs in the background.

The classic mistake it catches in interviews: **don't wrap I/O calls in `Task.Run`**, especially on a web server.

```csharp
// BAD — wastes a thread just to sit and wait on I/O that's already async
var data = await Task.Run(() => _http.GetStringAsync(url));

// GOOD — the I/O is already async; no extra thread needed
var data = await _http.GetStringAsync(url);
```

Wrapping already-async I/O in `Task.Run` gains nothing and *costs* you an extra thread — on a server it actually **hurts** how many requests you can handle. 

**Rule:** `Task.Run` is for CPU work. On a desktop app it keeps the UI smooth; on a web server you rarely need it at all.

---

### Q3b. What is the difference between `Task.Run()` and `Task.Factory.StartNew()`? Why is `Task.Run()` preferred in modern C#?

**Answer.** `Task.Run` was introduced in .NET 4.5 as a simplified, developer-friendly wrapper around `Task.Factory.StartNew`.

#### Differences & Gotchas:

1. **Handling Async Delegates (`Func<Task>`)**:
   - `Task.Factory.StartNew(async () => await DoWorkAsync())` returns a nested **`Task<Task>`**! If you `await` it without calling `.Unwrap()`, you only wait for the outer delegate to start, NOT for the inner async work to complete!
   - `Task.Run(async () => await DoWorkAsync())` automatically unwraps the nested task, returning a plain `Task` that can be awaited directly.

2. **Default Options**:
   - `Task.Run` defaults to `TaskScheduler.Default` (the ThreadPool) and `TaskCreationOptions.DenyChildAttach`.
   - `Task.Factory.StartNew` uses the current `TaskScheduler` (which in UI frameworks might be the UI thread sync context!).

| Feature | `Task.Run` | `Task.Factory.StartNew` |
| :--- | :--- | :--- |
| **Async Lambda Support** | ✅ Automatically unwraps `Task<Task>` | ❌ Returns nested `Task<Task>` (must call `.Unwrap()`) |
| **Default Scheduler** | Always ThreadPool (`TaskScheduler.Default`) | Uses `TaskScheduler.Current` by default |
| **Use Case** | 99% of everyday async/CPU background work | Advanced cases (custom schedulers, `LongRunning` threads) |

```csharp
// ❌ Dangerous with StartNew:
Task<Task> nestedTask = Task.Factory.StartNew(async () => await DoAsync()); // Returns Task<Task>!

// ✅ Safe with Task.Run:
Task task = Task.Run(async () => await DoAsync()); // Unwraps automatically!
```

---

### Q4. Explain `Task.WhenAll` and `Task.WhenAny`.

**Answer.**
### `Task.WhenAll`

`Task.WhenAll` waits for **all** tasks to complete before continuing.

```csharp
Task task1 = DownloadFileAsync();
Task task2 = SendEmailAsync();
Task task3 = UpdateDatabaseAsync();

await Task.WhenAll(task1, task2, task3);

Console.WriteLine("All tasks completed.");
```

##### When to Use

Use `Task.WhenAll` when multiple independent tasks can run **in parallel** and you need **all of them** to finish.

### `Task.WhenAny`

`Task.WhenAny` waits until **the first task** completes, then returns that completed task.

```csharp
Task task1 = DownloadFileAsync();
Task task2 = SendEmailAsync();

Task completedTask = await Task.WhenAny(task1, task2);

Console.WriteLine("First task completed.");
```

> **Note:** `Task.WhenAny` does **not** cancel the remaining tasks. They continue running unless you cancel them explicitly.

##### When to Use

Use `Task.WhenAny` when you only need the **first completed result** or want to implement timeouts or race multiple operations.

---

### Q5. What's the difference between `Task.Delay` and `Thread.Sleep`?

**Answer.**
- **`Thread.Sleep(1000)`** **blocks the thread** for 1 second — the thread just sits there, wasted.
- **`Task.Delay(1000)`** waits 1 second **without holding a thread** — you `await` it, and the thread is free to do other work meanwhile.

```csharp
Thread.Sleep(1000);        // blocks and wastes the thread
await Task.Delay(1000);    // frees the thread during the wait
```

In async code, always use `Task.Delay`. Seeing `Thread.Sleep` in an async method is a red flag — it throws away the whole benefit of async. `Task.Delay` can also be cancelled with a `CancellationToken` (see F7).

---

## F3 — ConfigureAwait, Synchronization Context & Deadlocks

### Q1. What is a `SynchronizationContext` and what role does it play in `await`?

**Answer.** A `SynchronizationContext` is basically a rule for **"which thread should the code run on."** Some environments set one up:
- **Desktop UI apps (WPF, WinForms)** have one that sends work back to the **single UI thread** — because you can only touch UI controls from that thread.
- **Classic ASP.NET** (the old .NET Framework) had one tied to the request.

By default, when you `await`, C# **remembers the current context** and **resumes your code on it afterward**. That's why, in a button click handler, the code after `await` can safely update the screen — it was automatically sent back to the UI thread.

```csharp
async void Button_Click(...)
{
    var data = await _http.GetStringAsync(url);   // remembers the UI context
    myLabel.Text = data;                          // resumes on UI thread — safe
}
```

The catch: this "resume on the original thread" behavior is what causes the classic deadlock (Q3). Good news — **ASP.NET Core has no such context**, so it resumes on any available thread and is much less deadlock-prone.

---

### Q2. What does `ConfigureAwait(false)` do, and where should you use it?

**Answer.** `ConfigureAwait(false)` says: **"don't bother going back to the original thread — just resume on any available thread."** This skips some overhead and, importantly, **avoids the deadlock** described in Q3.

**Rule of thumb:** use `ConfigureAwait(false)` in **library code** (code that doesn't care which thread it runs on). Don't use it in UI code that needs to touch controls after the `await`.

```csharp
// Library method — doesn't care which thread it resumes on
public async Task<string> FetchAsync(string url)
{
    var resp = await _http.GetAsync(url).ConfigureAwait(false);
    return await resp.Content.ReadAsStringAsync().ConfigureAwait(false);
}
```

Modern note: **ASP.NET Core has no sync context**, so `ConfigureAwait(false)` does nothing there — but you should still add it in **shared libraries** that might be used by a desktop app someday. In everyday ASP.NET Core code, it's often just left out.

---

### Q3. Walk through the classic `.Result` / `.Wait()` deadlock. Why does it happen?

**Answer.** This is *the* famous async interview question. The deadlock happens when you **block** on an async method (using `.Result` or `.Wait()`) on a thread that must resume its own work — like a UI thread.

Here's the trap, step by step:
1. The UI thread calls `GetDataAsync().Result` — this **blocks the UI thread**, making it wait.
2. Inside, the `await` wants to **resume back on the UI thread** when the web call finishes.
3. But the UI thread is **stuck** at step 1, waiting.
4. So the method can't finish (it needs the UI thread), and the UI thread can't move on (it's waiting for the method). **Both wait forever — deadlock.**

```csharp
// DEADLOCKS on a UI thread:
public string GetData()
{
    return GetDataAsync().Result;   // blocks THIS thread...
}
private async Task<string> GetDataAsync()
{
    return await _http.GetStringAsync("https://...");   // ...needs THIS thread to resume → stuck
}
```

**Fixes:**
1. **Go async all the way** — use `await GetDataAsync()` instead of `.Result`.
2. Add `ConfigureAwait(false)` inside the async method so it doesn't need to come back to the UI thread.

ASP.NET Core doesn't deadlock like this, but the rule stays the same: **never block on async** — see the next question for why it's still bad there.

---

### Q4. If ASP.NET Core has no sync context, is blocking on async safe there?

**Answer.** It won't cause *that specific* deadlock — but blocking is **still wrong** because it causes **thread starvation**. Every blocked thread is a thread sitting uselessly, waiting. Under heavy load, requests grab threads, block, and the server runs out of threads to handle new requests — throughput collapses, and it can look like the app has frozen.

```csharp
public IActionResult Get() => Ok(_service.GetAsync().Result);          // DON'T
public async Task<IActionResult> Get() => Ok(await _service.GetAsync()); // DO
```

So the rule generalizes: **never block on async.** The exact failure differs by environment (deadlock in UI apps, thread starvation on servers), but the fix is always the same — async all the way.

---

### Q5. What is `GetAwaiter()` and `GetResult()`?

**Answer.** `GetAwaiter()` is the method that makes a type awaitable. When you write `await task`, the compiler calls `task.GetAwaiter()` behind the scenes and uses what it returns to pause and resume your method (Q3 in F1). You normally never call it yourself — `await` does it for you.

`GetResult()` is a method on that awaiter. It **blocks the thread** until the work finishes, then gives you the result (or throws if it failed).

```csharp
var result = await GetDataAsync();                    // ✅ normal way — no blocking
var result = GetDataAsync().GetAwaiter().GetResult(); // ❌ blocks the thread
```

So where you see `.GetAwaiter().GetResult()` in real code, it is being used to **call an async method from sync code**. That is the sync-over-async anti-pattern this whole section is about (Q1), with all the same problems: deadlocks in apps with a sync context (Q2), thread starvation on a server (Q4).

**Why do people use it instead of `.Result`?** Only one reason: **cleaner exceptions.** `.Result` and `.Wait()` wrap any error in an `AggregateException`, so you have to dig into `InnerException` to find what actually went wrong. `GetResult()` throws the original exception directly, just like `await` would.

```csharp
try { task.Result; }                        // throws AggregateException — original is nested
catch (AggregateException ex) { }

try { task.GetAwaiter().GetResult(); }      // throws the real exception
catch (InvalidOperationException ex) { }
```

That makes it the **least bad** way to block — but it is still blocking. It doesn't fix deadlocks or thread starvation; it only improves the error message you get.

**When is it acceptable?** The few places where there is no async option: `Main` in older C# versions (before `async Main`), a constructor, `Dispose()`, or a framework callback that must be sync. Everywhere else, **make the caller async instead** (Q3).

---

## F5 — Threads vs Tasks vs Thread Pool

### Q1. What's the difference between a `Thread`, the thread pool, and a `Task`?

**Answer.** These are three levels, from low-level to high-level:

- **Thread** → An actual OS thread that executes code.
- **Task** → A higher-level abstraction that represents a unit of asynchronous work.
- **Thread Pool** → A pool of reusable worker threads managed by .NET.

> **A `Task` is not a thread.** A task may run on a ThreadPool thread, another thread, or may not require a thread while waiting for I/O.


### Thread

A **Thread** is an actual operating system thread.

```csharp
Thread thread = new Thread(() =>
{
    DoWork();
});

thread.Start();
```

- Created explicitly using `new Thread()`
- Expensive to create and destroy
- You are responsible for managing it
- Rarely used in modern .NET applications


### Task

A **Task** represents an asynchronous operation, **not a thread**.

```csharp
Task task = Task.Run(() => DoWork());

await task;
```

A `Task` can:

- Run on a ThreadPool thread (most common)
- Represent an asynchronous I/O operation (no thread while waiting)
- Be completed immediately (`Task.FromResult()`)

The runtime decides **how** to execute the task.


### Thread Pool

The **Thread Pool** is a collection of reusable worker threads maintained by .NET.

Instead of creating a new thread for every task, .NET reuses existing ThreadPool threads.

```csharp
await Task.Run(() =>
{
    DoWork();
});
```

Here:

1. `Task.Run()` creates a **Task**.
2. The task is queued to the **Thread Pool**.
3. A **ThreadPool thread** executes the work.
4. When finished, the thread returns to the pool for reuse.


#### Where Is a Task Created?

When you write:

```csharp
Task task = Task.Run(() => DoWork());
```

- A **Task object** is created in memory.
- The task is queued to the **Thread Pool**.
- A **ThreadPool thread** picks up and executes the work.

So:

- **Task** = Represents the work.
- **ThreadPool thread** = Executes the work.



#### Important Note

With I/O operations:

```csharp
await httpClient.GetAsync(url);

await dbContext.Users.ToListAsync();

await File.ReadAllTextAsync(path);
```

A thread starts the operation, but **no thread is blocked while waiting** for the I/O to complete. When the operation finishes, a ThreadPool thread resumes the async method.

---

### Q2. When would you create a dedicated `Thread` instead of using a `Task`?

**Answer.** Rarely — but there are a few valid cases, mostly for **long-running or blocking** work that shouldn't tie up a shared pool thread:

- **A long-running background loop** that runs for the whole life of the app (like a background poller). Hogging a pool thread forever would starve the pool, so use a dedicated thread.
- **Blocking work you can't make async** — parking it on its own thread avoids using up pool threads.
- **Special needs** — a specific priority or thread configuration.

```csharp
// Long-running dedicated thread (won't starve the pool):
var t = new Thread(RunForever) { IsBackground = true, Name = "poller" };
t.Start();
```

For almost everything else — short bursts of work, handling requests, parallel jobs — **use Tasks / the thread pool.**

---

## F6 — lock, Interlocked, SemaphoreSlim, Mutex

### Q1. What is `lock`, and what does it do?

**Answer.** `lock (obj) { ... }` makes sure **only one thread at a time** can run the code inside the block. It's how you protect shared data from being changed by two threads at once (which causes corruption/race conditions).

```csharp
lock (_gate) { _count++; }   // only one thread increments at a time
```

Best practices interviewers expect:
- **Lock on a private object made just for locking:** `private readonly object _gate = new();`.
- **Never `lock(this)`, `lock(typeof(X))`, or lock on a string** — these are reachable from outside your class, so other code could accidentally lock the same thing and deadlock you.
- **Keep the locked section short**, and never `await` inside a lock (see next question).

---

### Q2. Why can't you `await` inside a `lock`, and what do you use instead?

**Answer.** You **can't `await` inside a `lock`** — the compiler won't allow it. The reason: a `lock` must be released by the **same thread** that took it, but `await` might resume your code on a **different** thread, which would then try to release a lock it never took — broken.

---

## F7 — CancellationToken & Cooperative Cancellation

### Q1. How does cancellation work in .NET? What is "cooperative" cancellation?

**Answer.** .NET cancellation is **cooperative** — nothing forcibly kills a running operation. Instead:
- A **`CancellationTokenSource`** is the "off switch."
- It hands out a lightweight **`CancellationToken`** to the work.
- The work **voluntarily checks** the token and stops when it sees a cancellation request.

If the code never checks the token, nothing happens — cooperation is required.

```csharp
using var cts = new CancellationTokenSource();
Task work = DoWorkAsync(cts.Token);
// later, from elsewhere:
cts.Cancel();   // requests cancellation; the work must notice and stop
```

Two ways the work checks the token:
- **`token.ThrowIfCancellationRequested()`** — throws an exception to stop the operation (common inside loops).
- **`token.IsCancellationRequested`** — a simple true/false you can check to stop gracefully.

The good news: built-in async methods (`HttpClient`, EF Core, `Task.Delay`) accept a token and do the checking for you — so just **pass the token through** to them.

---

### Q2. How do you propagate a token, add a timeout, and combine tokens?

**Answer.**
- **Pass the token down** through every async call — it's only useful if it reaches the code that can actually stop.
- **Add a timeout** with `new CancellationTokenSource(TimeSpan)` or `cts.CancelAfter(...)`.
- **Combine tokens** (e.g. the caller's token *plus* your own timeout) with `CreateLinkedTokenSource` — the combined token trips if *any* of them do.

```csharp
public async Task<string> FetchAsync(string url, CancellationToken ct)
{
    // add a 5-second timeout on top of the caller's token:
    using var timeout = new CancellationTokenSource(TimeSpan.FromSeconds(5));
    using var linked = CancellationTokenSource.CreateLinkedTokenSource(ct, timeout.Token);

    return await _http.GetStringAsync(url, linked.Token);   // pass it down
}
```

In ASP.NET Core, the framework hands you a `CancellationToken` in your action methods that automatically cancels when the client disconnects — just accept it and pass it along.

---

### Q3. How should you handle a cancellation exception? Is cancellation an error?

**Answer.** Cancellation is a **normal, expected outcome — not a bug.** That's why it has its own exception type (`OperationCanceledException`) instead of being treated as a real failure. A cancelled task ends in a "Cancelled" state, distinct from a "Failed" state.

```csharp
try
{
    await LongRunningAsync(token);
}
catch (OperationCanceledException) when (token.IsCancellationRequested)
{
    // expected: someone cancelled — clean up quietly, don't log as an error
}
catch (Exception ex)
{
    // a real failure — log/handle it
}
```

Guidelines: treat cancellation as expected (don't log it as an error), always **dispose the `CancellationTokenSource`** (use `using`), and when *writing* a cancellable method, prefer `ThrowIfCancellationRequested()` so callers get the standard exception they can catch.

---

## F8 — Parallel Programming (`Parallel`, PLINQ)

### Q1. What's the difference between parallelism and async? When is parallel programming the right tool?

**Answer.** People mix these up constantly, but they solve different problems:

- **Async/await** is about **not wasting a thread while waiting** — it's for **I/O work** (database, web, disk). The win is handling *more* work with the same threads.
- **Parallelism** is about **using many CPU cores at once to finish faster** — it's for **CPU-heavy work** (image processing, big calculations). The win is doing *one big job* faster.

```csharp
// I/O-bound → async (no threads wasted while waiting)
await Task.WhenAll(urls.Select(u => httpClient.GetStringAsync(u)));

// CPU-bound → parallel (spread across cores to finish sooner)
Parallel.ForEach(images, img => img.ApplyFilter());
```

**Simple rule:**
- Lots of I/O operations → `async` + `Task.WhenAll`.
- Heavy CPU work you can split up → `Parallel.For`/`ForEach` or PLINQ.

**Important warning:** don't use `Parallel.ForEach` to fire off *I/O* (like web calls) — that wastes threads. Use `Task.WhenAll` for I/O instead.

---

### Q2. How do `Parallel.For` and `Parallel.ForEach` work, and what are the pitfalls?

**Answer.** `Parallel.For` / `Parallel.ForEach` take a loop and **run its iterations across multiple threads at once**, then wait for them all to finish. .NET handles splitting up the work for you. Use them when each iteration is **independent** and **CPU-heavy**.

```csharp
Parallel.For(0, items.Length, i => Process(items[i]));

Parallel.ForEach(orders, new ParallelOptions { MaxDegreeOfParallelism = 4 },
    order => Recompute(order));   // limit to 4 at a time
```

The pitfalls interviewers look for:
- **Shared data races** — since many threads run at once, changing a shared list/counter without protection causes bugs. Use `Interlocked`, a `lock`, or a thread-safe collection.
- **Errors come bundled** — if several iterations throw, you get an `AggregateException` (multiple errors at once).
- **No order guarantee** — results don't come back in input order.
- **Not for I/O** — for async work per item, use `Parallel.ForEachAsync` (.NET 6+) instead.

---

### Q3. What is PLINQ, and when should (and shouldn't) you use it?

**Answer.** **PLINQ (Parallel LINQ)** runs a LINQ query across multiple cores just by adding **`.AsParallel()`**. It's great for **CPU-heavy processing over large in-memory collections**.

```csharp
var results = numbers
    .AsParallel()
    .Where(n => IsPrime(n))   // runs across cores
    .Select(n => n * n)
    .ToArray();
```

**When it helps:** a **large** collection, **expensive** work per item (like checking if a number is prime), and independent items.

**When it hurts** (stick with normal LINQ):
- **Small collections or cheap operations** — the overhead of coordinating threads outweighs the gain.
- **I/O work** — use async instead.
- **On a busy web server** — PLINQ grabs lots of threads, competing with request handling, and can make things *slower* overall.

The senior instinct: **measure it** — PLINQ is easy to add and easy to accidentally make things slower.

---

### Q4. What's the difference between concurrency and parallelism?

**Answer.** Short version:

- **Concurrency** = *dealing with* many things at once. Tasks are started and interleaved, but they don't have to run at the same instant.
- **Parallelism** = *doing* many things at once. Tasks genuinely run at the same instant, on different CPU cores.

The easy example is cooking. **Concurrency** is one cook making two dishes: put the rice on, and while it boils, chop vegetables for the curry. One person, two jobs on the go, switching between them whenever one is waiting. **Parallelism** is two cooks, one making each dish at the same time.

Notice what that means: **concurrency needs only one core.** The single cook never does two things in the same instant — they just never sit idle while the rice boils.

```csharp
// CONCURRENCY — one thread handles 3 downloads by not waiting on any of them
var results = await Task.WhenAll(
    http.GetStringAsync(url1),
    http.GetStringAsync(url2),
    http.GetStringAsync(url3));

// PARALLELISM — the work is split across cores and computed simultaneously
Parallel.ForEach(images, img => img.ApplyFilter());
```

In the first block, all three downloads are **in flight together**, but almost no CPU is used — the thread is released at the `await` and the network does the work. In the second, every core is busy computing.

How they map to C#:

| | Concurrency | Parallelism |
|---|---|---|
| Goal | Stay busy instead of waiting | Finish faster |
| Best for | I/O work — network, database, disk | CPU work — maths, image processing |
| Tool | `async`/`await`, `Task.WhenAll` | `Parallel.For`/`ForEach`, PLINQ |
| Cores needed | Works on one | Needs several to help |

They aren't opposites, and you often use both: a web server handles thousands of requests **concurrently**, and one of those requests might use `Parallel.ForEach` to resize images **in parallel**.

The one-line answer for an interview: **parallelism is about doing work simultaneously; concurrency is about making progress on several things without waiting on any one of them.** Parallelism is a way of achieving concurrency, but not the only one.
