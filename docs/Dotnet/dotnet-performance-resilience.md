# Q. Performance, Caching & Resilience
---

> Health checks and container readiness are in [dotnet-testing-ops.md](dotnet-testing-ops.md). `async`/`await` fundamentals are in [../C-Sharp/csharp-async.md](../C-Sharp/csharp-async.md).

## QA — Caching

### Q1. What should you cache, and how do you design a cache key?

**Answer.** **Caching** is keeping a copy of an expensive result so you don't recompute or refetch it.

Good candidates share three traits: **expensive to produce**, **read far more often than written**, and **tolerant of being slightly stale**. Reference data, computed reports, and rendered fragments fit. A user's account balance does not.

**The cache key must include everything the result depends on.** Miss something and you serve one user's data to another:

```csharp
// ❌ two users share one entry
var key = "dashboard";

// ✅ everything the result varies by is in the key
var key = $"dashboard:user:{userId}:tenant:{tenantId}:v2";
```

The `v2` suffix is a useful habit: bump it when the shape of the cached object changes, so old entries are ignored rather than deserialized into the new type and failing.

⚠️ **A missing key component is a security bug, not just a correctness one.** It's the same mistake as a cached page fragment without `vary-by-user`.

---

### Q2. `IMemoryCache` vs `IDistributedCache` — when do you use each?

**Answer.** **`IMemoryCache`** stores objects in the process's own memory. **`IDistributedCache`** stores serialized bytes in a shared external store, usually Redis.

| | `IMemoryCache` | `IDistributedCache` |
|---|---|---|
| Lives | In this process | Outside, shared (Redis, SQL) |
| Speed | Fastest — no network, no serialization | Network hop + serialize |
| Stores | Any object | **`byte[]` only** |
| Survives restart | No | Yes |
| Multiple instances | **Each has its own copy** | All share one |

```csharp
// IMemoryCache — objects, with a built-in get-or-create
var product = await _cache.GetOrCreateAsync($"product:{id}", async entry =>
{
    entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10);
    return await _repo.GetAsync(id);
});

// IDistributedCache — you serialize, and there is no GetOrCreate
var json = await _cache.GetStringAsync($"product:{id}");
```

❌ **The multi-instance trap:** with three servers each holding its own `IMemoryCache`, an update invalidated on server A is still stale on B and C. Users see different data depending on which server answers — intermittent and very hard to reproduce.

**Rule of thumb:** `IMemoryCache` for hot data that tolerates being slightly stale per node; `IDistributedCache` when correctness across instances matters or the data must survive a restart. Many apps want both, which is what `HybridCache` formalizes (Q4).

---

### Q3. What is cache-aside, and what's its main hazard?

**Answer.** **Cache-aside** is the standard caching pattern: check the cache; on a hit return it; on a miss load from the source, put it in the cache, and return it.

```csharp
var cached = await _cache.GetStringAsync(key);
if (cached is not null) return Deserialize(cached);      // hit

var value = await _repo.LoadAsync(id);                    // miss → load
await _cache.SetStringAsync(key, Serialize(value));       // populate
return value;
```

❌ **The hazard is a cache stampede** (also called thundering herd). A popular key expires, and 500 concurrent requests all miss at the same moment — so all 500 hit the database at once, for the same value. The cache was what was protecting the database, and the instant it expires that protection vanishes.

Three defences:

- **Lock per key** so only the first request loads and the rest wait for it. `IMemoryCache.GetOrCreateAsync` does *not* do this by itself.
- **Stagger expiries** with a small random offset, so keys don't all expire together.
- **Use `HybridCache`** (Q4), which has stampede protection built in.

---

### Q4. Absolute vs sliding expiration, and what is `HybridCache`?

**Answer.** **Absolute** expiry removes the entry at a fixed time after it was cached, no matter how often it's read. **Sliding** expiry resets the clock on every read, so a frequently-read entry never expires.

```csharp
entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10);   // gone in 10 min, always
entry.SlidingExpiration = TimeSpan.FromMinutes(10);                 // gone after 10 min idle
```

**Use absolute for data that goes stale** — prices, stock levels, anything where "how old is this?" matters. **Use sliding for session-like data** where the point is to keep what's actively in use.

⚠️ **Sliding alone can keep data forever.** A key read every 9 minutes never expires and never refreshes. Set **both** — sliding for idle eviction plus an absolute cap:

```csharp
entry.SlidingExpiration = TimeSpan.FromMinutes(10);
entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1);   // hard ceiling
```

**`HybridCache`** (.NET 9) is the modern recommendation. It's **two-tier**: an in-process L1 in front of an optional distributed L2. A read checks memory, then Redis, then your factory:

```csharp
var product = await _cache.GetOrCreateAsync(
    $"product:{id}",
    async ct => await _repo.GetAsync(id, ct),
    cancellationToken: ct);
```

That one call gives you in-memory speed, cross-instance sharing, **stampede protection**, and serialization handled for you. It supersedes hand-rolling `IMemoryCache` in front of `IDistributedCache`, and you can add Redis later without touching call sites.

---

### Q5. Response caching vs output caching?

**Answer.** Both cache whole HTTP responses. The difference is **who holds the copy**.

|  | Response caching | Output caching (.NET 7+) |
|---|---|---|
| Copy lives | in the browser / proxy | on your server |
| You control it | no — it's a request via headers | yes, completely |
| Invalidate early | not possible | `EvictByTagAsync` |
| Saves server work | only if the client obeys | always — no handler runs |

**Response caching** just sends `Cache-Control` headers and hopes. A browser or CDN may honour them or ignore them, and once a response is cached out there you cannot pull it back.

**Output caching** keeps the copy in your process, so a hit never reaches your endpoint — and you can evict on demand:

```csharp
builder.Services.AddOutputCache();
app.UseOutputCache();

app.MapGet("/products", GetProducts)
   .CacheOutput(p => p.Expire(TimeSpan.FromMinutes(5)).Tag("products"));

// after a write, drop the stale copy immediately:
await cacheStore.EvictByTagAsync("products", ct);
```

**Use output caching for new work.** Eviction by tag is the deciding feature — without it, stale data sits there until it expires.

⚠️ **Never cache an authenticated response without varying by user.** Output caching skips authenticated requests by default; overriding that is how you serve one user's data to another.

---

## QB — HttpClient

### Q6. What's wrong with `new HttpClient()`, and with one static `HttpClient`?

**Answer.** Two opposite traps, and knowing both is the point of the question.

**`new HttpClient()` per request → socket exhaustion.** Disposing a client doesn't release its socket immediately — TCP holds it in `TIME_WAIT` for up to a couple of minutes. Under load you exhaust the port range and start getting `SocketException`:

```csharp
using var client = new HttpClient();     // ❌ a new socket per call
var res = await client.GetAsync(url);
```

This is the single most common `HttpClient` bug in production, and it looks completely reasonable.

**One static `HttpClient` forever → stale DNS.** That fixes sockets, but the handler caches connections indefinitely and **never picks up DNS changes**. When the upstream fails over to a new IP, your app keeps calling the dead one.

**`IHttpClientFactory` solves both.** It pools handlers, so sockets are reused, and retires each handler after about 2 minutes, so DNS gets re-resolved:

```csharp
builder.Services.AddHttpClient<GitHubClient>(c =>
{
    c.BaseAddress = new Uri("https://api.github.com/");
    c.DefaultRequestHeaders.Add("User-Agent", "MyApp");
});

public class GitHubClient(HttpClient http)      // already configured, injected
{
    public Task<string> GetUserAsync(string login, CancellationToken ct)
        => http.GetStringAsync($"users/{login}", ct);
}
```

That's a **typed client**, the style to prefer — configuration lives with the client and there are no magic strings. (Named clients, `AddHttpClient("github")`, suit one class talking to several APIs.)

❌ **Never `using` an injected `HttpClient`.** The factory owns it; disposing is pointless at best and throws `ObjectDisposedException` at worst. `HttpClient` is thread-safe for concurrent requests, so sharing is fine.

---

### Q7. What is a `DelegatingHandler` for?

**Answer.** It's **middleware for outbound HTTP** — the request-side equivalent of ASP.NET Core middleware. Each handler can change the outgoing request, pass it down the chain, and inspect the response coming back.

It's where cross-cutting HTTP concerns belong, so they don't clutter every call site: auth tokens, correlation IDs, logging.

```csharp
public class AuthHeaderHandler(ITokenProvider tokens) : DelegatingHandler
{
    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken ct)
    {
        var token = await tokens.GetAccessTokenAsync(ct);
        request.Headers.Authorization = new("Bearer", token);

        return await base.SendAsync(request, ct);      // pass down the chain
    }
}

builder.Services.AddTransient<AuthHeaderHandler>();
builder.Services.AddHttpClient<GitHubClient>()
                .AddHttpMessageHandler<AuthHeaderHandler>();
```

Handlers are ordered — the first one added is outermost. This is also the mechanism the resilience libraries use to add retries and circuit breaking (Q9).

---

## QC — Resilience

### Q8. What are the core resilience patterns?

**Answer.** Any call across a network can be slow, flaky, or down. Five patterns, usually layered:

| Pattern | What it does |
|---|---|
| **Timeout** | Bounds the wait, so a hung dependency can't tie up your threads |
| **Retry** | Re-attempts transient failures, with **backoff and jitter** |
| **Circuit breaker** | Stops calling a dependency that's clearly down (Q9) |
| **Bulkhead** | Caps concurrent calls, so one slow dependency can't consume everything |
| **Fallback** | Returns a degraded response — cached value, empty list — instead of an error |

A typical outbound call composes them: **timeout → retry → circuit breaker → fallback**.

⚠️ **Retry needs backoff and jitter.** Retrying immediately, in lockstep across every instance, turns a brief blip into a self-inflicted outage. Backoff spaces attempts out; jitter stops them synchronizing.

❌ **And retries are only safe for idempotent operations.** `GET`, `PUT`, and `DELETE` can be repeated harmlessly. `POST` usually can't: a "create order" that timed out **may have already succeeded** — the failure was in the *response*, not the operation. Retrying blindly creates a duplicate order or a double charge.

The fix is an **idempotency key** the server dedupes on:

```csharp
request.Headers.Add("Idempotency-Key", key);   // a retry is now safe
```

No library can know this for you — Polly can't tell whether your POST tolerates a retry.

---

### Q9. Explain the circuit breaker states, and how do you add resilience in .NET?

**Answer.** A circuit breaker stops you hammering a dependency that's already failing. Three states:

- **Closed** — normal. Calls flow through and failures are counted.
- **Open** — the failure threshold was crossed, so calls **fail instantly** without a network attempt, for a set break duration.
- **Half-open** — after the break, a few trial calls go through. Success closes the circuit; failure re-opens it.

**Why it matters:** without it, every call to a dead dependency waits for its timeout, so your threads pile up on requests that cannot succeed — and your app dies alongside the dependency. Failing fast keeps you responsive and gives the struggling service room to recover.

**In .NET, use `Microsoft.Extensions.Http.Resilience`**, which wires Polly into `IHttpClientFactory`:

```csharp
builder.Services
    .AddHttpClient<GitHubClient>(c => c.BaseAddress = new Uri("https://api.github.com/"))
    .AddStandardResilienceHandler();      // ✅ production-grade defaults in one line
```

That single call adds a rate limiter, total timeout, retry with backoff and jitter, circuit breaker, and per-attempt timeout — sensibly tuned. Use `AddResilienceHandler` with a custom pipeline only when you've measured that the defaults don't fit.

---

## QD — Throughput

### Q10. Beyond async, what levers matter for throughput?

**Answer.** Async frees threads during I/O waits ([../C-Sharp/csharp-async.md](../C-Sharp/csharp-async.md)), but it doesn't reduce how much work you do. Three things that matter more once async is in place:

**1. Pagination.** Never return an unbounded collection. `GET /orders` with no limit works on dev data and dies on production data — big queries, big serialization, big allocations, all for a client that shows twenty rows.

```csharp
public async Task<IActionResult> Get(int page = 1, int size = 50)
{
    size = Math.Min(size, 100);                 // ✅ cap it — clients will ask for 10,000
    var items = await _repo.PageAsync(page, size);
    return Ok(items);
}
```

Cap the page size server-side. For deep pages, prefer **keyset pagination** (`WHERE Id > lastSeenId`) over `Skip(n)`, because the database still has to count through everything you skipped.

**2. Streaming large results.** Returning a huge list materializes all of it in memory before anything is sent. `IAsyncEnumerable<T>` streams row by row, so memory stays flat and the client starts receiving data immediately:

```csharp
public async IAsyncEnumerable<OrderDto> Export()
{
    await foreach (var o in _repo.StreamAllAsync())
        yield return o;                          // serialized as it goes
}
```

**3. Allocations and GC pressure.** Every allocation is cheap; collecting millions of them is not. The wins are avoiding per-request allocation in hot paths, reusing large buffers with `ArrayPool<T>` instead of allocating new arrays (which land on the Large Object Heap), and using `Span<T>` for parsing without intermediate strings ([../C-Sharp/csharp-memory.md](../C-Sharp/csharp-memory.md)).

**Measure before optimizing any of this.** The usual real cause of a slow endpoint is a missing database index or an N+1 query ([dotnet-ef-core.md](dotnet-ef-core.md)) — not allocations.
