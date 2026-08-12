# H. Exceptions & Error Handling
---

## H1 — try/catch/finally & Exception Propagation

### Q1. What happens when an exception is thrown? How does it propagate?

**Answer.** When you `throw`, normal execution **stops**, and .NET starts looking for something that can handle the error. It searches **up the call stack** — from where the error happened, back through whoever called that method, and so on — looking for a `try/catch` that catches this type of exception.

- If it **finds** a matching `catch`, execution jumps there (running any `finally` blocks along the way).
- If it finds **nothing** anywhere, the exception is "unhandled" and the **app crashes**.

```csharp
void A() => B();          // no catch here → the error passes right through
void B()
{
    try { C(); }
    catch (InvalidOperationException e)   // caught HERE, even though C() threw it
    {
        Console.WriteLine(e.Message);
    }
}
void C() => throw new InvalidOperationException("boom");   // where it started
```

The key idea: an exception **doesn't** get checked and passed up one level at a time by your code. .NET automatically routes it to the nearest matching `catch` — which is why an error thrown deep in your code can be caught far above it, with the in-between methods doing nothing special.

---

### Q2. When does `finally` run — and when does it *not*?

**Answer.** A `finally` block runs **no matter what** — whether the code succeeded, threw an exception, or returned early. That's its whole purpose: **guaranteed cleanup** (closing files, releasing locks) that must happen on every path. (In fact, a `using` block is just a `finally` behind the scenes.)

```csharp
FileStream? fs = null;
try
{
    fs = File.OpenRead(path);
    return Parse(fs);      // finally STILL runs before the method actually returns
}
finally
{
    fs?.Dispose();         // guaranteed cleanup, on success AND on error
}
```

The only times `finally` **doesn't** run are catastrophic ones where the app can't run any more code at all — like `Environment.FailFast()`, a stack overflow, or the process being force-killed. In normal code, treat `finally` as "always runs."

One useful pattern: a `try/finally` with **no `catch`** cleans up but **doesn't swallow the error** — the cleanup runs, then the exception keeps propagating. That's how you say "clean up, but let the error continue."

---

### Q3. What happens if you `return` or `throw` inside `finally`?

**Answer.** **Don't do it** — both cause subtle bugs:

- A `return` in `finally` **overrides** whatever the `try` was going to return, and **silently swallows** any exception that was in flight.
- A `throw` in `finally` **replaces** the original exception, so you **lose the real cause** of the problem.

```csharp
int Bad()
{
    try { return 1; }
    finally { return 2; }   // returns 2 — the "return 1" is thrown away. AVOID
}

int AlsoBad()
{
    try { throw new InvalidOperationException("real cause"); }
    finally { throw new Exception("mask"); }   // "real cause" is LOST. AVOID
}
```

**Rule:** keep `finally` for **cleanup only** — never `return` or `throw` from it.

---

### Q4. How does catch ordering work?

**Answer.** Catch blocks are checked **top to bottom**, and the **first matching one wins**. So you must list them **most specific first, most general last**. If you put a broad type before a more specific one, the specific one can never be reached — and the compiler treats that as an **error**.

```csharp
try { DoWork(); }
catch (FileNotFoundException e) { /* most specific */ }
catch (IOException e)           { /* broader — FileNotFoundException is a kind of IOException */ }
catch (Exception e)             { /* catch-all, last */ }
```

```csharp
// COMPILE ERROR: the second catch can never be reached
try { DoWork(); }
catch (Exception e) { }
catch (IOException e) { }   // unreachable — Exception already caught everything
```

Think of it like a set of filters getting progressively wider — the narrow ones have to come first, or the wide one catches everything before they get a chance.

---

### Q5. When should you catch `Exception` versus a specific type — and when should you *not* catch at all?

**Answer.** Catch the **most specific type you can actually do something about.** Catching the broad `Exception` is usually a smell, because it also swallows bugs you didn't expect (like a null-reference error) — turning a clear crash into silent, broken behavior.

Catch a **specific** type when you have a **real plan** for it: retry a temporary database error, show a friendly message for a validation error, fall back when a web request fails.

```csharp
// GOOD: catch what you can handle, let the rest propagate
try { return await _client.GetAsync(url); }
catch (HttpRequestException ex) { return Fallback(); }   // specific, you have a plan

// BAD: swallowing everything hides real bugs
try { Process(); }
catch { }   // never do this — silent failure, unknown state
```

The **worst thing** you can do is an **empty catch** (`catch { }` with no logging and no rethrow) — it hides the failure and the program keeps running in a broken state.

Broad `catch (Exception)` *is* okay in a couple of places: at the **top level** of a request or background job (to log the error before it crashes everything), and in **background loops** that shouldn't die over one bad item. Even there, you usually **log and rethrow**, not silently swallow.

---

### Q6. How expensive are exceptions, and why "don't use them for control flow"?

**Answer.** **Throwing** an exception is **much slower** than a normal return — it has to capture the stack trace and search for a handler. But note: it's the *throwing* that's expensive. A `try` block that **doesn't** throw costs almost nothing. So the advice isn't "avoid `try`" — it's "**don't throw on paths you expect to happen normally.**"

```csharp
// BAD: uses an exception for an expected case (bad input) — slow if it happens a lot
int Parse(string s)
{
    try { return int.Parse(s); }
    catch (FormatException) { return 0; }   // thrown on EVERY bad input
}

// GOOD: TryParse returns a bool — no exception for the expected "bad input" case
int Parse(string s) => int.TryParse(s, out var n) ? n : 0;
```

This is exactly why .NET gives you `TryParse`, `TryGetValue`, `TryDequeue`, and so on: **expected** failures (bad input, missing key) are handled with return values, and exceptions are saved for the **truly unexpected**. Using exceptions for ordinary "did it work or not" branching also makes code harder to read.

---

## H2 — Rethrowing: `throw` vs `throw ex`

### Q7. `throw;` vs `throw ex;` — what's the difference, and why does it matter so much?

**Answer.** This is the single most-asked exception gotcha. Inside a `catch`:

- **`throw;`** (by itself) rethrows the same exception and **keeps its original stack trace** — so you can still see where the error actually started.
- **`throw ex;`** rethrows it but **resets the stack trace** to *this* line — you **lose** all information about where it really came from.

```csharp
try { DeepCall(); }
catch (Exception ex)
{
    _logger.Error(ex);
    throw;        // ✅ keeps the ORIGINAL stack trace (points to DeepCall)
    // throw ex;  // ❌ resets the trace to THIS line — the real origin is lost
}
```

In production, `throw ex;` is a debugging nightmare because the error looks like it came from your catch block instead of where it truly happened.

**The rule to state in an interview:** always use **`throw;`**, never **`throw ex;`** (unless you deliberately want to reset the trace, which is rare).

---

## H3 — Custom Exceptions & Throw vs Return a Result

### Q8. How do you design a custom exception properly?

**Answer.** Derive **directly from `Exception`**, and give it the **three standard constructors** so it behaves like a built-in exception. The most important one is the **`(string message, Exception innerException)`** constructor — it lets callers **wrap** a lower-level error without losing the original cause. Add extra properties only when they carry **useful data** the handler can act on (like an ID or error code).

```csharp
public class OrderNotFoundException : Exception
{
    public int OrderId { get; }

    public OrderNotFoundException() { }
    public OrderNotFoundException(string message) : base(message) { }
    public OrderNotFoundException(string message, Exception inner) : base(message, inner) { }

    public OrderNotFoundException(int orderId)
        : base($"Order {orderId} was not found.")
        => OrderId = orderId;
}
```

A couple of naming/design tips: end the type name with **`Exception`**, keep your exception types **shallow** (don't build deep hierarchies), and don't derive from `ApplicationException` (an old, deprecated type).

---

### Q9. When does a custom exception add value versus reusing a framework one?

**Answer.** **Default to the built-in exceptions** — everyone already understands them:
- `ArgumentNullException` / `ArgumentException` / `ArgumentOutOfRangeException` — bad method arguments.
- `InvalidOperationException` — the object is in the wrong state for this call.
- `NotSupportedException` / `NotImplementedException`, `FormatException`, `KeyNotFoundException`, `ObjectDisposedException`.

Create a **custom** exception only when:
1. A caller will want to **catch this specific type** to handle it differently, or
2. It represents a real **business concept** worth naming (like `InsufficientFundsException`) and carries useful data.

```csharp
// Reuse built-in types for generic problems:
if (count < 0)  throw new ArgumentOutOfRangeException(nameof(count));
if (!_isOpen)   throw new InvalidOperationException("Connection is not open.");

// Custom type ONLY because a caller wants to catch THIS specifically:
try { _bank.Withdraw(amount); }
catch (InsufficientFundsException ex) { NotifyUser(ex.Shortfall); }
```

The simple test: *"Will any `catch` block ever want to treat this differently from a generic error?"* If no, use a built-in type with a good message. If yes, make a custom one.

---

### Q10. When should a method throw, and when should it return a value/`Result`?

**Answer.** The dividing line is **expected vs unexpected**:

- **Throw** for genuinely **unexpected** problems the caller can't reasonably foresee — bad arguments, invalid state, a broken invariant, an unrecoverable I/O failure.
- **Return a value** for **expected, routine** outcomes that are just part of normal flow — validation failures, "not found," parse failures, business-rule rejections.

Why not throw for the routine cases? Because throwing is slow, it clutters the flow, and — most importantly — it's **invisible in the method signature**, so callers forget to handle it.

.NET itself follows this: `int.Parse` **throws** (use when the input *should* be valid), while `int.TryParse` **returns a bool** (use when bad input is expected). For richer "expected failure" info, many teams use a **`Result<T>`** type that carries either a value or an error:

```csharp
// Try* pattern — expected failure, no exception:
if (_cache.TryGetValue(key, out var value)) return value;

// Result<T> — makes the failure explicit; the caller can't ignore it:
public Result<Order> GetOrder(int id)
{
    var order = _repo.Find(id);
    return order is null
        ? Result.Fail<Order>($"Order {id} not found")   // expected — not an exception
        : Result.Ok(order);
}
```

**Rules of thumb:** don't throw for things you expect to happen routinely; don't use exceptions just to return data; and since a public method's exceptions are part of its contract, document the ones it can throw.

---

### Q11. What are guard-clause helpers like `ArgumentNullException.ThrowIfNull`, and why prefer them?

**Answer.** Modern .NET (6+) added short **helper methods** that replace the classic `if (x is null) throw new ArgumentNullException(...)` with a single line. Better still, they **figure out the parameter name automatically**, so you don't have to type (and possibly mistype) it.

```csharp
public void Send(Message msg, string recipient, int retries)
{
    ArgumentNullException.ThrowIfNull(msg);                    // name "msg" captured automatically
    ArgumentException.ThrowIfNullOrWhiteSpace(recipient);
    ArgumentOutOfRangeException.ThrowIfNegative(retries);
    ArgumentOutOfRangeException.ThrowIfGreaterThan(retries, 5);
    // ... real work ...
}
```

The family includes `ArgumentNullException.ThrowIfNull`, `ArgumentException.ThrowIfNullOrEmpty` / `ThrowIfNullOrWhiteSpace`, and several `ArgumentOutOfRangeException.ThrowIf...` helpers, plus `ObjectDisposedException.ThrowIf`. Prefer them over hand-written guards: **less code, correct parameter names, and consistent messages.**

