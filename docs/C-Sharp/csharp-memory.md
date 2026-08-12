# G. Memory Management & Runtime (CLR)
---

## G1 — Garbage Collection

### Q1. How does the .NET garbage collector work at a high level?

**Answer.** The **garbage collector (GC)** is .NET's automatic memory cleaner. You create objects with `new` and **never free them manually** — the GC figures out which objects are no longer being used and reclaims their memory for you.

How does it decide what's still in use? It starts from a set of **"roots"** (things your program can directly reach — local variables, static fields) and follows every reference from there. Anything it can reach is **kept**; anything it *can't* reach is **garbage** and gets cleaned up.

```csharp
var a = new byte[100];   // reachable through 'a' → kept
a = new byte[100];       // the FIRST array is now unreachable → will be collected
```

The key idea: the GC doesn't watch every object constantly. Every so often, it **traces** from the roots to see what's still reachable. A nice consequence: even if two dead objects point at *each other* (A → B → A), they still get collected — since nothing reachable points to them, tracing never finds them.

---

### Q2. What are GC generations, and why is the GC generational?

**Answer.** The GC sorts objects into three **generations** by age — **gen 0, gen 1, gen 2** — and this is the most important performance idea in the whole GC.

It's based on a simple observation: **most objects die young.** Temporary objects (loop variables, short-lived results) become garbage almost instantly, while long-lived things (caches, app-wide state) tend to keep sticking around. So instead of scanning *everything* every time, the GC focuses on the youngest objects, where most of the garbage is.

Follow one object through its life to see how the three generations work:

- **Gen 0 — where every object is born.** `new` puts the object here. Gen 0 is small, so it fills quickly and gets collected often. When a gen-0 collection runs, anything unreachable is freed on the spot. **Most objects never leave gen 0** — they die here, which is exactly what you want, because cleaning gen 0 is cheap.
- **Gen 1 — the survivors of one collection.** If an object is still referenced when gen 0 is collected, it isn't freed. It is **promoted to gen 1**. Gen 1 is a waiting room: the GC doesn't yet know whether this object is genuinely long-lived or just took slightly longer to die, so it checks it less often than gen 0.
- **Gen 2 — the long-lived objects.** Survive a gen-1 collection and the object is **promoted to gen 2**. This is where caches, statics, and app-wide state end up. There is no gen 3, so objects stay here for the rest of their life. Gen 2 is collected **rarely**, because doing so means scanning the whole heap.

So the path is: **born in gen 0 → survive a collection → promoted to gen 1 → survive again → promoted to gen 2 → stays there.**

Each promotion means "you've proved you're still needed, so I'll stop checking you so often." That is the whole trick: the GC spends most of its effort on gen 0, where nearly everything is garbage, and rarely touches gen 2, where nearly everything is still alive.

```csharp
for (int i = 0; i < 1_000_000; i++)
{
    var tmp = new StringBuilder();   // dies almost immediately → cheap gen-0 garbage
    tmp.Append(i);
}
// A long-lived cache, by contrast, survives and gets promoted to gen 2, where it stays.
```

**Practical takeaway:** keep your objects short-lived so they die cheaply in gen 0. The expensive case is objects that live long enough to reach gen 2 and *then* die, since clearing them needs a full, costly collection.

---

### Q3. When does the garbage collector actually run?

**Answer.** You never decide this — the runtime does. A collection is triggered by one of these:

1. **A generation's budget is full.** This is the normal trigger. Each generation has a size budget, and when an allocation would exceed gen 0's budget, a gen-0 collection runs. It's allocation that triggers the GC, so **an app that allocates nothing never collects.**
2. **The system is low on memory.** The OS signals memory pressure and the runtime responds with a collection.
3. **Someone calls `GC.Collect()`.** Almost always a mistake (Q7).

Which generation gets collected depends on which budget was exceeded. Gen 0 fills fastest, so gen-0 collections are the common case. **Collecting a generation always collects the ones below it** — a gen-2 collection is a *full* collection of gen 0, gen 1, and gen 2.

---

### Q4. What is the Large Object Heap (LOH), and why does it matter?

**Answer.** Objects of **85,000 bytes or more** don't go on the normal heap. They go on a separate one called the **Large Object Heap (LOH)**. Usually that means big arrays or big strings.

```csharp
var small = new byte[1_000];     // normal heap → starts in gen 0
var big   = new byte[100_000];   // 100,000 bytes → goes on the LOH
```

**Why a separate heap?** After cleaning the normal heap, the GC slides the surviving objects together to close up the gaps. That keeps memory tidy. But sliding a 10 MB array means copying 10 MB, which is slow — so the GC **doesn't move LOH objects at all**.

That leads to the two things worth knowing:

**1. The LOH is only cleaned during a full (gen-2) collection.** It is never part of a cheap gen-0 pass. So allocating lots of big objects forces more of the expensive collections.

**2. It gets fragmented.** Because nothing is moved, freeing a big object leaves a **hole**. A new object can only use that hole if it fits:

```
[ 100 KB used ][ 50 KB hole ][ 200 KB used ]

new byte[40_000]  → fits in the hole ✅
new byte[90_000]  → doesn't fit, needs new memory ❌
```

Over time you can end up with lots of free space split into holes that are all too small to use. You can even get an out-of-memory error while plenty of memory is technically free.

**The fix: reuse big arrays instead of allocating new ones.** `ArrayPool<T>.Shared` lets you rent an array and give it back when you're done:

```csharp
var buffer = ArrayPool<byte>.Shared.Rent(100_000);
try { /* use buffer */ }
finally { ArrayPool<byte>.Shared.Return(buffer); }   // reused, not garbage
```

Keeping your buffer sizes consistent helps too, since same-sized holes get reused instead of piling up.

---

### Q5. Workstation GC vs Server GC — what's the difference and when do you pick each?

**Answer.** These are two "modes" of the GC, tuned for different situations:

- **Workstation GC** — uses **one heap**. Optimized for **low latency and light resource use** — good for desktop and console apps. This is the default there.
- **Server GC** — uses a **separate heap and a dedicated GC thread per CPU core**, so cleanup runs in parallel. This gives much higher **throughput** for busy, multi-core servers, at the cost of using more memory. It's the default for **ASP.NET Core** on multi-core machines.

```xml
<!-- In your .csproj -->
<PropertyGroup>
    <ServerGarbageCollection>true</ServerGarbageCollection>
</PropertyGroup>
```

**Rule of thumb:** **Server GC** for high-traffic multi-core servers; **Workstation GC** for desktop apps or small containers with few CPU cores (where Server GC's per-core heaps would waste memory).

---

### Q6. What is background/concurrent GC?

**Answer.** The expensive full (gen-2) collection normally has to **pause your whole app** while it works. On a big heap that pause could be noticeable — bad for interactive apps.

**Background GC** (on by default) fixes this by doing most of that expensive work on a **separate background thread while your app keeps running**. The quick gen-0/gen-1 collections still cause tiny pauses, but the big full-heap work mostly happens in the background — so pauses stay short.

There are also "latency modes" you can set to hint the GC to avoid big pauses during a critical window (like rendering a frame), but for most apps the defaults are fine — this is an advanced tuning knob, not something you'll reach for often.

---

### Q7. Why should you almost never call `GC.Collect()`?

**Answer.** Because the GC **tunes itself** and knows far more about what's going on than you do. Calling `GC.Collect()` by hand usually **makes things worse**:

- It forces a **full (expensive) collection** at a random moment.
- It **messes up the generational system** — objects that would have died cheaply in gen 0 get promoted to gen 2 by the premature collection, making them *more* expensive to clean up later.
- It causes a pause you didn't actually need.

```csharp
GC.Collect();   // almost always wrong in real code
```

The rare valid uses: getting a clean baseline right before measuring memory in a profiler, or very specialized teardown. If you're reaching for `GC.Collect()` to "fix" a memory problem, the real cause is almost always a **memory leak** (see G3) or too many allocations — fix that instead.

---

### Q8. How do value types vs reference types affect GC pressure?

**Answer.** Every **class** (reference type) you create with `new` is an object **on the heap** that the GC has to track and eventually clean up. A **struct** (value type) used as a local variable lives **on the stack** (or inline inside its container) and costs the GC **nothing** — it just disappears when the method ends. So using structs for small, short-lived data reduces **GC pressure** (how hard and how often the GC has to work).

```csharp
// Class: each element is a separate heap object the GC must track
var points = new List<PointClass>();

// Struct: elements live inline in the array — no per-element GC cost
struct PointStruct { public int X, Y; }
var fast = new List<PointStruct>();
```

Two cautions: **boxing** a struct (see [csharp-type-system.md](csharp-type-system.md) A2) *does* put it on the heap, undoing the benefit; and **large structs** are expensive to copy, so keep structs small. Overall, **reducing allocations is usually a bigger win than tuning the GC** itself.

---

## G2 — `IDisposable`, `using`, Dispose Pattern, Finalizers

### Q1. What problem does `IDisposable` solve? The GC frees memory — why do we need it?

**Answer.** The GC only cleans up **memory**. It does **not** handle other kinds of resources that aren't memory — like open **files, network connections, and database connections**. And the GC runs **on its own schedule**, so it has no idea you're done with a database connection *right now* and someone else is waiting for it.

**`IDisposable`** gives you **cleanup at a precise, known moment**. You (or a `using` block) call `Dispose()` to release those resources **immediately**, instead of waiting for the GC to get around to it eventually.

```csharp
// The connection is released the instant we're done — not "eventually"
using var conn = new SqlConnection(cs);
conn.Open();
// ... use it ...
// Dispose() runs here, at the closing brace — connection returned to the pool NOW
```

So the split is simple: **GC handles memory, on its own schedule; `IDisposable` handles everything else (files, connections), right when you say so.**

---

### Q2. Explain `using` statement vs `using` declaration.

**Answer.** Both guarantee `Dispose()` gets called — even if an error is thrown — so resources always get cleaned up. The difference is just *when*:

- **`using` statement** (with braces) — disposed at the **closing brace** of the block. Use it to clean up as soon as you're done.
- **`using` declaration** (C# 8, no braces) — disposed at the **end of the method**. Less nesting, cleaner-looking code.

```csharp
// using statement — disposed at the inner closing brace
using (var file = File.OpenRead(path))
{
    Read(file);
}   // file.Dispose() here

// using declaration — disposed at the end of the method
using var file2 = File.OpenRead(path);
Read(file2);
// file2.Dispose() runs at method end
```

If you have several `using` declarations, they're disposed in **reverse order** (last one first) — which is exactly right when later resources depend on earlier ones.

---

### Q3. Walk me through the dispose pattern.

**Answer.** Most of the time, disposing is simple: implement `Dispose()` and clean up your resources inside it. The important thing to know is that there are **two situations**:

**Situation 1 — your class just holds other disposable things** (a `FileStream`, an `HttpClient`, etc.). This is by far the most common case. Just dispose them:

```csharp
public class ReportGenerator : IDisposable
{
    private readonly FileStream _file;
    private bool _disposed;

    public void Dispose()
    {
        if (_disposed) return;   // safe to call twice
        _file.Dispose();         // dispose what we own
        _disposed = true;
    }
}
```

**Situation 2 — your class directly holds a raw operating-system resource** (a low-level handle). This is rare in normal app code. Here you *also* want a **finalizer** as a safety net in case someone forgets to call `Dispose()`. The full pattern looks like this:

```csharp
public class NativeBuffer : IDisposable
{
    private IntPtr _handle;   // a raw unmanaged resource
    private bool _disposed;

    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);   // we cleaned up — no need for the finalizer
    }

    protected virtual void Dispose(bool disposing)
    {
        if (_disposed) return;
        if (disposing)
        {
            // dispose any managed objects here
        }
        // always release the raw resource
        if (_handle != IntPtr.Zero) { Native.Close(_handle); _handle = IntPtr.Zero; }
        _disposed = true;
    }

    ~NativeBuffer() => Dispose(false);   // safety net if Dispose() was forgotten
}
```

The honest senior takeaway: **you almost never need the full version.** In modern .NET, if you need a raw handle, wrap it in a `SafeHandle` (see Q5) and you're back to the simple case. Situation 1 covers nearly all real code.

---

### Q4. What's `GC.SuppressFinalize` for?

**Answer.** `GC.SuppressFinalize(this)` tells the GC: *"don't bother running this object's finalizer — I already cleaned everything up in `Dispose()`."* You only call it in classes that **have a finalizer**.

Why it matters: objects **with finalizers are more expensive to clean up**. Normally they need **two GC cycles** to fully go away (one to run the finalizer, another to reclaim the memory), and they survive an extra generation. `SuppressFinalize` skips all that, so a properly disposed object gets cleaned up in the normal, cheap way.

```csharp
public void Dispose()
{
    Dispose(true);
    GC.SuppressFinalize(this);   // skip the expensive finalizer path
}
```

**Rule:** if your class has a finalizer, call `SuppressFinalize` in `Dispose()`. If it has no finalizer, you don't need it.

---

### Q6. What is `IAsyncDisposable` / `await using`, and when do you need it?

**Answer.** Regular `Dispose()` is **synchronous**. But some cleanup needs to be **async** — like flushing data over a network or committing an async transaction. Doing that synchronously would block a thread (the sync-over-async problem — see [csharp-async.md](csharp-async.md) F4). **`IAsyncDisposable`** solves this with a `DisposeAsync()` method you use via **`await using`**.

```csharp
public sealed class AsyncBufferedWriter : IAsyncDisposable
{
    private readonly Stream _stream;

    public async ValueTask DisposeAsync()
    {
        await _stream.FlushAsync();     // async cleanup, no blocking
        await _stream.DisposeAsync();
    }
}

// Usage:
await using var writer = new AsyncBufferedWriter(stream);
// DisposeAsync() is awaited at the end of scope
```

**When to use it:** when your cleanup genuinely does async work. You'll see it on things like EF Core's `DbContext` and `Stream`. Don't add it just for style — plain synchronous cleanup should stay as regular `IDisposable`.

---

### Q7. What are the rules for a well-behaved `Dispose()`?

**Answer.** Three rules interviewers expect:

1. **Safe to call more than once (idempotent).** `Dispose()` might get called twice (e.g. once by you, once by a `using`). A simple guard flag makes the second call do nothing.
2. **Don't throw exceptions from `Dispose()`.** Disposal often runs during error cleanup, and throwing there can hide the original error. Log cleanup failures instead of letting them escape.
3. **Dispose things in the right order** — clean up things that depend on others first (e.g. a writer before the stream it writes to).

```csharp
public void Dispose()
{
    if (_disposed) return;              // (1) safe to call twice
    try
    {
        _writer?.Dispose();            // (3) writer first...
        _stream?.Dispose();            //     ...then the stream underneath
    }
    catch (Exception ex)
    {
        _log.Warn(ex, "cleanup failed"); // (2) don't let it escape
    }
    _disposed = true;
}
```

Bonus: once an object is disposed, calling its methods should throw `ObjectDisposedException` so misuse fails clearly. .NET 8 has a helper: `ObjectDisposedException.ThrowIf(_disposed, this)`.

---

### Q8. What actually happens if you forget to call `Dispose()`?

**Answer.** The GC will still reclaim the object's **memory** eventually — so you rarely leak *memory*. The real problem is the **other resource** (the file, the connection) stays held longer than it should:

So forgetting `Dispose()` mostly hurts by **holding scarce resources too long** — which is the whole reason `IDisposable` exists. This is why calling `using` should be automatic, and why analyzers warn you about undisposed objects.

---

## G3 — Memory Leaks in Managed Code

### Q1. How does a memory leak happen in .NET, and why does the GC allow it?

**Answer.** A .NET leak happens when an object is **still reachable but you will never use it again**. Something in your code is still holding a reference to it, so it stays in memory forever.

```csharp
static readonly List<Customer> _cache = new();   // static → lives for the whole app

void HandleRequest(Customer c)
{
    _cache.Add(c);      // added, never removed → this customer can never be collected
}
```

Every request adds a customer that is never taken out. The list keeps growing, memory keeps climbing, and nothing is ever freed.

**Why does the GC allow it?** Because by its own rules, nothing is wrong. The GC's only job is to free objects that **cannot be reached** from a root — a static field, a local variable, an active reference (G1 Q1). It walks the references and asks one question: *can I still get to this object?*

For that customer, the answer is yes: `_cache` is static, so it is a root, and it points straight at the object. The GC cannot tell the difference between "the app still needs this" and "the developer forgot to remove it." Both look identical — a live reference.

So the GC is working exactly as designed. **The leak is in your code, not in the GC.**

That is why a .NET leak is the opposite of a C++ leak:

| | C++ leak | .NET leak |
|---|---|---|
| Cause | You lost the pointer, so you can't free it | You **kept** a reference you should have dropped |
| Object is | Unreachable | Reachable |
| GC's view | — | "Still in use, keep it" |

**The fix is always the same: break the reference** so the object becomes unreachable.

### The common ways this happens

**1. Event handlers you never unsubscribe.** This is the most common leak in .NET. When you do `publisher.Event += handler`, the **publisher stores a reference to the subscriber**. So a long-lived publisher keeps every subscriber alive:

```csharp
public class Screen
{
    public Screen(AppState state)
    {
        state.Changed += OnChanged;    // ❌ 'state' now holds a reference to this Screen
    }
    void OnChanged(object? s, EventArgs e) { }
}
```

Open and close that screen 100 times and all 100 stay in memory, because the long-lived `AppState` still points at each one. Fix: unsubscribe in `Dispose()`.

```csharp
public void Dispose() => _state.Changed -= OnChanged;   // ✅ reference broken
```

**2. A cache or collection that only grows.** Anything you add to and never remove from. Static makes it worse (it lives for the whole app), but an instance collection on a long-lived object leaks just the same:

```csharp
_dictionary[key] = value;    // ❌ nothing ever removes old keys
```

Fix: use a cache with eviction — `MemoryCache` with a size limit or expiry — or remove entries when you're done.

**3. Timers you never stop.** A running timer is held by the runtime, and it holds your callback, which holds the object the callback belongs to:

```csharp
_timer = new Timer(OnTick, null, 0, 1000);   // ❌ keeps this object alive forever
```

Fix: `_timer.Dispose()` when you're finished.

**4. Captured variables in a lambda.** A lambda captures the variables it uses. If that lambda is stored somewhere long-lived, everything it captured stays alive too:

```csharp
var bigData = new byte[50_000_000];
_callbacks.Add(() => Console.WriteLine(bigData.Length));  // ❌ 50 MB kept alive by the lambda
```

Fix: don't capture more than you need, and clear the list when done.

**5. Long-lived objects holding short-lived ones.** A singleton that stores something per-request — a `DbContext`, a user object, an HTTP context. The singleton lives forever, so whatever it grabbed lives forever too. See the captive dependency problem in [.NET DI](../Dotnet/dotnet-ef-core.md).

**6. Unmanaged resources without `Dispose()`.** File handles, sockets, native memory. The GC doesn't manage these at all, so leaving them undisposed leaks resources the GC can never reclaim (G2).

---

The pattern behind all six: **something long-lived is holding on to something short-lived.** When you look for a leak, that's the question to ask — *what is still pointing at this?*

The symptom is memory that climbs over time and never comes back down, even after a full collection.

---

### Q5. How do you detect and diagnose a memory leak in production?

**Answer.** A simple, systematic approach:

1. **Confirm it's really a leak.** Watch memory over time under steady load. A real leak keeps **climbing and never drops back down**, even after the GC runs. Tools like `dotnet-counters` show the heap size trend.
2. **Take two heap snapshots and compare them.** Capture memory, say, 10 minutes apart under load, then **diff** them to see **which types keep piling up**. Tools: **dotMemory**, **PerfView**, **dotnet-gcdump**.
3. **Find what's holding it.** For the leaking type, ask the tool for the **"path to root"** — this tells you *which* reference (a static, an event, a timer) is keeping it alive. That's your leak.
4. **Fix and verify** — break the reference (unsubscribe, dispose, evict), then re-check that the type stops accumulating.

```text
# Tools you'd name in an interview:
dotnet-counters   → live memory trend
dotnet-gcdump     → heap snapshot to diff
PerfView / dotMemory → snapshot diff + path-to-root
```

**Two helpful tools for *preventing* leaks:** `WeakReference<T>` lets you reference an object **without keeping it alive** (great for caches). `ConditionalWeakTable<TKey, TValue>` lets you attach extra data to an object without keeping it alive. Both are handy when unsubscribing cleanly isn't practical.

---
