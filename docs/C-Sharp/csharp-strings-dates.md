# L. Strings, Dates, Numerics & Complexity
---

## L1 — Strings

### Q1. Why is `string` immutable in .NET, and what does that cost?

**Answer.** **Immutable** means a string's characters can never change after it's created. Every operation that looks like a change — `+`, `Replace`, `ToUpper`, `Substring` — actually returns a **new** string and leaves the original alone.

```csharp
var a = "hello";
var b = a.ToUpper();     // b is a NEW string
Console.WriteLine(a);    // "hello" — unchanged
```

**Why it was designed this way:**

- **Thread-safe automatically.** Nothing can change, so many threads can read the same string with no locks.
- **Safe as a dictionary key.** A string's hash code never changes, so it can't get lost in a `Dictionary` after being stored.
- **Shared literals.** The runtime keeps one copy of `"hello"` no matter how many times it appears in your code.

**The cost is in loops.** Every `+` allocates a new string and copies everything so far, so building a string in a loop gets slower the longer it gets:

```csharp
// ❌ 10,000 items = 10,000 strings allocated, each one bigger than the last
string csv = "";
foreach (var item in items) csv += item + ",";

// ✅ one buffer that grows
var sb = new StringBuilder();
foreach (var item in items) sb.Append(item).Append(',');
string result = sb.ToString();
```

For a fixed handful of pieces (`a + b + c`), `+` is fine — the compiler turns it into a single operation. Use `StringBuilder` when a loop is involved (see [csharp-modern.md](csharp-modern.md), I3).

---

### Q2. What does `string.Length` actually count?

**Answer.** It counts **`char` values, not visible characters.** A `char` is 16 bits, which isn't always enough for one symbol, so some characters take two `char`s.

```csharp
"abc".Length        // 3 ✅ as expected
"😀".Length          // 2 ❗ one emoji, two chars
"café".Length       // 4 or 5, depending on how the é was written
```

The emoji is a single character that doesn't fit in 16 bits, so it's stored as **two** `char` values. `Length` counts storage slots, not what you see on screen.

When does this matter? Anywhere you slice or count text that might contain emoji, or scripts outside the Latin alphabet:

```csharp
var s = "😀abc";
s.Substring(0, 1);        // ❌ half an emoji — produces a broken character
```

For text a user typed, iterate by **grapheme** (what a person would call one character):

```csharp
var e = StringInfo.GetTextElementEnumerator("😀abc");
while (e.MoveNext()) Console.WriteLine(e.GetTextElement());   // 😀, a, b, c
```

For most business code — IDs, codes, ASCII — `Length` is fine. Reach for the above when handling arbitrary user text.

---

### Q3. What's the difference between ordinal and culture-sensitive string comparison?

**Answer.** **Ordinal** compares the raw character values — a straight numeric comparison. **Culture-sensitive** compares by language rules, the way a dictionary in that language would sort.

```csharp
"a".Equals("A", StringComparison.Ordinal)            // false — different values
"a".Equals("A", StringComparison.OrdinalIgnoreCase)  // true
```

They differ because languages have their own rules. The classic example is Turkish, where lowercase `i` uppercases to `İ` (dotted), not `I`:

```csharp
// On a Turkish machine:
"file".ToUpper() == "FILE"            // ❌ false! ToUpper() gives "FİLE"

"file".ToUpperInvariant() == "FILE"   // ✅ true everywhere
```

This is a real bug: code that checks file extensions or header names with `ToUpper()` works on your machine and fails for users with a different system language.

**The rule:**

| Comparing | Use |
|---|---|
| IDs, file paths, keys, header names, anything internal | **`Ordinal`** / `OrdinalIgnoreCase` |
| Text shown to users — sorting a list of names | **culture-sensitive** (the default) |

```csharp
if (path.EndsWith(".json", StringComparison.OrdinalIgnoreCase))  // ✅ safe
if (path.EndsWith(".json"))                                      // ⚠️ culture-sensitive
```

Ordinal is also **faster**, since it skips all the language rules.

---

## L2 — DateTime, DateTimeOffset & Time Zones

### Q4. What is `DateTime.Kind`, and why does it cause bugs?

**Answer.** `Kind` is a flag on every `DateTime` saying what the value means: `Utc`, `Local`, or `Unspecified`.

```csharp
DateTime.Now         // Kind = Local
DateTime.UtcNow      // Kind = Utc
new DateTime(2026, 1, 1)   // Kind = Unspecified ← the dangerous one
```

The problem is that `Unspecified` is the default, and .NET **guesses** what it means depending on the method you call. A value read from a database usually comes back `Unspecified`, so:

```csharp
var fromDb = new DateTime(2026, 1, 1, 12, 0, 0);   // Unspecified
var utc = fromDb.ToUniversalTime();                // assumes it's LOCAL → shifts the time!
```

If that value was already UTC, you've just moved it by your timezone offset — silently, with no error. It works on a developer machine in UTC and breaks in production elsewhere.

**The fix:** be explicit about what you have.

```csharp
var utc = DateTime.SpecifyKind(fromDb, DateTimeKind.Utc);   // ✅ state the truth
```

Better still, use `DateTimeOffset` (Q5), which can't be ambiguous.

---

### Q5. When should you use `DateTimeOffset` instead of `DateTime`?

**Answer.** `DateTimeOffset` stores the time **plus its offset from UTC**, so it always identifies an unambiguous moment. `DateTime` doesn't, unless you're careful about `Kind`.

```csharp
DateTime      dt = new(2026, 1, 1, 12, 0, 0);              // 12:00 — where? unclear
DateTimeOffset dto = new(2026, 1, 1, 12, 0, 0, TimeSpan.FromHours(2));
// 12:00 at UTC+2 — precisely one moment in time
```

**Use `DateTimeOffset` for anything that happened at a point in time:** created-at, logged-at, ordered-at. It's the right default for timestamps.

**Use `DateTime` when the time is a description, not a moment** — like "the shop opens at 9am", which is 9am wherever the shop is, on any day.

**The golden rules for a real application:**

1. **Store UTC.** Every timestamp in the database is UTC.
2. **Convert to local only when displaying**, using the user's timezone.
3. **Never store the user's local time** — offsets change twice a year with daylight saving, so the value becomes ambiguous.

```csharp
// Store
order.CreatedAt = DateTimeOffset.UtcNow;

// Display
var tz = TimeZoneInfo.FindSystemTimeZoneById("Europe/Oslo");
var local = TimeZoneInfo.ConvertTime(order.CreatedAt, tz);
```

⚠️ **Daylight saving is where this bites.** When clocks go back, 02:30 happens **twice** in some countries; when they go forward, 02:30 **doesn't exist at all**. Storing UTC avoids both problems, because UTC has no daylight saving.

---

### Q6. What do `DateOnly` and `TimeOnly` solve?

**Answer.** They represent a date without a time, and a time without a date. Before .NET 6 you had to use `DateTime` for both and ignore the half you didn't want.

```csharp
// ❌ before — a birthday with a meaningless midnight attached
DateTime birthday = new(1990, 5, 12, 0, 0, 0);

// ✅ now
DateOnly birthday = new(1990, 5, 12);
TimeOnly openingTime = new(9, 0);
```

The old approach caused real bugs, because that midnight was a real time that could shift across timezones and turn a birthday into the previous day. A `DateOnly` has no time, so it cannot move.

Use `DateOnly` for birthdays, invoice dates, and holidays. Use `TimeOnly` for opening hours and schedules.

---

## L3 — Numbers & Precision

### Q7. Why can't you use `double` for money?

**Answer.** Because `double` stores numbers in binary, and most decimal fractions have no exact binary form. The values are tiny fractions off, and those errors add up.

```csharp
double a = 0.1 + 0.2;
Console.WriteLine(a == 0.3);      // false!
Console.WriteLine(a);             // 0.30000000000000004
```

That is not a .NET bug — it's how binary floating point works everywhere. With money it means totals that are a cent out, and invoices that don't balance.

**Use `decimal` for money.** It stores numbers in base 10, so decimal fractions are exact:

```csharp
decimal a = 0.1m + 0.2m;
Console.WriteLine(a == 0.3m);     // ✅ true
```

| Type | Stores | Use for |
|---|---|---|
| `double` | Binary, approximate, fast | Measurements, graphics, science |
| `decimal` | Base 10, exact, slower | **Money**, anything a person checks |

Note the `m` suffix — `0.1` is a `double`, `0.1m` is a `decimal`.

Two related habits: set the database column to `decimal(18,2)` and not `float` ([EF Core](../Dotnet/dotnet-ef-core.md) rounds silently if you don't), and never compare two `double`s with `==` — check the difference is small instead:

```csharp
if (Math.Abs(x - y) < 0.0001) { }   // ✅ how to compare doubles
```

---

### Q8. Does integer arithmetic check for overflow?

**Answer.** **No.** By default, an integer that exceeds its maximum silently **wraps around** to the negative end.

```csharp
int max = int.MaxValue;         // 2,147,483,647
int result = max + 1;           // -2,147,483,648 ❗ no error at all
```

That silent wrap is the danger — a total, a counter, or an ID quietly becomes negative and the bug shows up much later, somewhere else.

**`checked` makes it throw instead:**

```csharp
checked
{
    int result = max + 1;       // ✅ throws OverflowException
}
```

You can turn it on for the whole project in the `.csproj`, which is worth doing for financial code:

```xml
<CheckForOverflowUnderflow>true</CheckForOverflowUnderflow>
```

The practical advice: use `long` when values might get large, and use `checked` where a wrong number would be serious. Note that `decimal` **always** throws on overflow, and `double` doesn't overflow at all — it goes to `Infinity`.

---

## L4 — Big-O & Choosing Collections

### Q9. What is Big-O notation?

**Answer.** Big-O describes **how much slower an operation gets as the data grows**. It ignores exact timings and describes the shape of the growth.

| Notation | Meaning | Example |
|---|---|---|
| **O(1)** | Same speed regardless of size | `dict[key]`, `list[5]` |
| **O(log n)** | Barely grows — doubling data adds one step | Binary search |
| **O(n)** | Doubles when the data doubles | `list.Contains(x)`, a `foreach` |
| **O(n log n)** | Good sorting | `list.Sort()` |
| **O(n²)** | 10× data = **100× slower** | A loop inside a loop |

The gap is what matters. With 1,000 items, an O(n²) operation does a **million** steps where O(n) does a thousand.

```csharp
// O(1) — jumps straight to the value however big the dictionary is
var user = usersById[42];

// O(n) — checks every item until it finds one
var user = usersList.First(u => u.Id == 42);
```

Both look similar in code. On 10 items the difference is nothing; on 100,000 it's the difference between instant and noticeably slow.

---

### Q10. How do you choose the right collection?

**Answer.** By what you need to do most often — look things up, or keep an order.

| Collection | Lookup | Add | Use when |
|---|---|---|---|
| `List<T>` | **O(n)** | O(1) | You need order, or you just iterate |
| `Dictionary<K,V>` | **O(1)** | O(1) | You look items up by a key |
| `HashSet<T>` | **O(1)** | O(1) | You need uniqueness, or "does this exist?" |
| `Queue<T>` | — | O(1) | First in, first out |
| `Stack<T>` | — | O(1) | Last in, first out |

**The most common real mistake is using a `List` for lookups:**

```csharp
// ❌ O(n) every time — with 10,000 customers this scans thousands of items per call
var customer = customerList.First(c => c.Id == id);

// ✅ O(1) — goes straight to it
var customer = customersById[id];
```

The rule of thumb: **if you search a collection by the same value repeatedly, it should be a `Dictionary` or `HashSet`.**

---

### Q11. How do you spot accidental O(n²) code?

**Answer.** Look for a **lookup inside a loop**. The loop is O(n), the lookup inside is O(n), so together they're O(n²).

```csharp
// ❌ O(n²) — for every order, scans the whole customer list
foreach (var order in orders)                    // n
{
    var customer = customers.First(c => c.Id == order.CustomerId);   // n
    Console.WriteLine(customer.Name);
}
```

With 1,000 orders and 1,000 customers that's a **million** comparisons.

**The fix is to build a dictionary once, then look up in O(1):**

```csharp
// ✅ O(n) — one pass to build the index, then instant lookups
var byId = customers.ToDictionary(c => c.Id);

foreach (var order in orders)
{
    var customer = byId[order.CustomerId];
    Console.WriteLine(customer.Name);
}
```

The signals to watch for in a review:

- `.First(...)`, `.Any(...)`, `.Contains(...)`, or `.Where(...)` **inside** a `foreach`.
- A nested loop where the inner one walks a whole collection.
- A database query inside a loop — the same shape, but far worse, since each step is a network round-trip (the N+1 problem in [EF Core](../Dotnet/dotnet-ef-core.md)).

This is the single most common performance bug in business code, and it always looks fine on small test data.
