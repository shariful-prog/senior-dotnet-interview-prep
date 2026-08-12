# I. Modern C# Language Features
---

## I1 — Pattern Matching

### Q1. What is pattern matching, and what problem does it solve?

**Answer.** **Pattern matching** is testing a value against a shape — a type, a set of property values, a range — and, if it matches, pulling data out of it at the same time.

The problem it solves: before it existed you had to test the type, cast it, then read the properties, all as separate steps.

```csharp
// Before
if (shape is Circle)
{
    var c = (Circle)shape;          // test, then cast
    if (c.Radius > 10) { /* ... */ }
}

// After
if (shape is Circle { Radius: > 10 } c) { /* ... */ }
```

That one line does three things: checks the type is `Circle`, checks `Radius > 10`, and gives you a typed variable `c`.

The bigger win is in a `switch`, where a chain of type checks becomes a readable table:

```csharp
static double Area(Shape s) => s switch
{
    Circle c   => Math.PI * c.Radius * c.Radius,
    Rect r     => r.W * r.H,
    Triangle t => 0.5 * t.Base * t.Height,
    null       => throw new ArgumentNullException(nameof(s)),
    _          => throw new NotSupportedException()
};
```

Besides being shorter, the compiler checks your work — it warns when you've missed a case (Q4).

---

### Q2. What kinds of patterns are there?

**Answer.** A **pattern** is a test you write against a value. C# has several kinds, each testing something different, and they all combine freely. These are the ones worth knowing.

```csharp
// Type — is it this type? (and give me a typed variable)
if (o is string s) { }

// Constant — does it equal this value?
if (x is 0) { }
if (o is null) { }

// Property — does it have these property values?
if (p is Person { Age: 18, Name: "Bob" }) { }

// Relational — comparisons
if (n is > 0 and < 100) { }

// Logical — and / or / not
if (c is 'a' or 'e' or 'i' or 'o' or 'u') { }
if (o is not null) { }

// var — always matches, just binds the value
if (Compute() is var result) { }
```

**Relational and logical patterns clean up range checks the most:**

```csharp
// Before — 'age' repeated on every line
if (age >= 0 && age < 13) return "Child";
if (age >= 13 && age < 20) return "Teen";

// After
var label = age switch
{
    < 0  => throw new ArgumentException(),
    < 13 => "Child",
    < 20 => "Teen",
    _    => "Adult"
};
```

Note `is not null` — it reads better than `!= null`, and it can't be broken by a custom `!=` operator.

There are also **list patterns** for matching a sequence's shape, which are handy but less common:

```csharp
if (nums is [1, 2, ..])          { }   // starts with 1, 2
if (nums is [var first, .., var last]) { }   // bind the ends, ignore the middle
```

---

### Q3. How do property patterns work, including nested ones?

**Answer.** A **property pattern** checks values inside an object using `{ }`:

```csharp
if (person is { Age: > 18, Name: "Bob" }) { }
```

You can look **inside nested objects**, which is where they save the most code:

```csharp
// Before — a null check at every level
if (order != null && order.Customer != null && order.Customer.Address != null
    && order.Customer.Address.City == "Oslo") { }

// After
if (order is { Customer.Address.City: "Oslo" }) { }
```

That dotted form is the one to remember: **a property pattern never throws on null.** If any part of the chain is null, it simply doesn't match.

You can bind values at the same time:

```csharp
if (order is { Total: > 100, Customer.Name: var name })
    Console.WriteLine($"{name} spent a lot");
```

An empty `{ }` means "not null", so `o is { }` is another way to write `o is not null`.

---

### Q4. What is exhaustiveness, and how does the compiler help?

**Answer.** **Exhaustiveness** means every possible input value is handled by some arm of the switch.

It matters because a `switch` expression must produce a value for **every** input. If the compiler can't prove you've covered them all it warns you (**CS8509**), and an unmatched value throws `SwitchExpressionException` at runtime.

```csharp
// ⚠️ warning: not all inputs are handled
string Describe(int n) => n switch
{
    > 0 => "positive",
    < 0 => "negative"
    // 0 is missing!
};
```

Two ways to complete it — handle the case explicitly (`0 => "zero"`), or add a catch-all `_`.

❌ **Don't add `_` automatically.** It silences the warning, but it also means that adding a new enum value later compiles silently instead of warning you. When you want to be told about new cases, list them all and leave `_` out — or make `_` throw:

```csharp
_ => throw new NotSupportedException($"Unhandled: {status}")
```

Also remember `null`. For a nullable input, add an explicit `null` arm, or it falls through to `_`.

---

## I2 — `switch` Expressions

### Q5. What is a switch expression, and how is it different from a switch statement?

**Answer.** A **switch expression** is a form of `switch` that returns a value, written with `=>` arms instead of `case` labels.

The difference in one line: a switch **statement** *does* something, a switch **expression** *produces a value*.

```csharp
// Statement — assign to a variable, break on every case
string label;
switch (status)
{
    case Status.Active: label = "Running"; break;
    case Status.Done:   label = "Finished"; break;
    default:            label = "Unknown"; break;
}

// Expression — returns the value directly
string label = status switch
{
    Status.Active => "Running",
    Status.Done   => "Finished",
    _             => "Unknown"
};
```

| | Statement | Expression |
|---|---|---|
| Produces a value | No | **Yes** |
| Syntax | `case X:` + `break` | `X => value,` |
| Missing cases | Silently does nothing | **Compiler warns** (Q4) |

Use an **expression** when every branch produces a value — most of the time. Use a **statement** when branches perform actions, like logging or calling methods.

**`when` guards** add a condition the pattern itself can't express:

```csharp
var fee = order switch
{
    { Total: > 1000 } when order.Customer.IsVip => 0m,
    { Total: > 1000 }                           => 5m,
    _                                           => 10m
};
```

⚠️ **Order matters.** Arms are tested top to bottom, so the most specific goes first. Put the plain `{ Total: > 1000 }` above the VIP arm and the VIP arm can never run.

---

## I3 — Strings

### Q6. How does string interpolation work, and when do you need `StringBuilder`?

**Answer.** **String interpolation** (`$"..."`) puts values straight into a string:

```csharp
var msg = $"Hello {name}, you are {age} years old";
```

The compiler turns it into efficient string-building code, so for a one-off string it's the right choice.

**The problem is loops.** Strings are immutable, so every `+` creates a **brand-new string**. Do that 10,000 times and you allocate 10,000 strings that immediately become garbage:

```csharp
// ❌ a new string every iteration
var result = "";
foreach (var item in items)
    result += item + ", ";
```

**`StringBuilder`** keeps one buffer and appends into it:

```csharp
// ✅ one buffer, one string at the end
var sb = new StringBuilder();
foreach (var item in items)
    sb.Append(item).Append(", ");
var result = sb.ToString();
```

**The rule:** a fixed number of pieces → interpolation. A loop or an unknown count → `StringBuilder`. Don't use it for three concatenations; it's slower there because of the extra object.

---

### Q7. What are raw string literals?

**Answer.** Raw strings (`"""`) let you write text **exactly as it is**, with no escaping. They're for JSON, SQL, XML, and regex — anything full of quotes or backslashes.

```csharp
// Before — escape every quote
var json = "{ \"name\": \"Bob\", \"age\": 30 }";

// After
var json = """
    { "name": "Bob", "age": 30 }
    """;
```

Two rules worth knowing:

**Indentation is stripped based on the closing quotes.** Whatever column the final `"""` sits at is removed from every line, so you can indent the literal to match your code without that whitespace landing in the string.

**Use more quotes if the text itself contains three quotes** — four `""""` delimit a string containing `"""`.

Interpolation works too, and the number of `$` signs says how many braces start an expression. That's how you keep literal `{` in JSON:

```csharp
var body = $$"""
    { "user": "{{name}}", "literal": { "nested": true } }
    """;
```

Here `{{name}}` is the value and single `{` stays literal.

---

## I4 — Parameters & `ref`/`out`/`in`

### Q8. Explain optional and named arguments, and the versioning gotcha.

**Answer.** **Optional arguments** give a parameter a default so callers can skip it. **Named arguments** let you pass by name instead of position:

```csharp
void Send(string to, string subject = "(none)", bool urgent = false) { }

Send("bob@x.com");                     // both defaults
Send("bob@x.com", urgent: true);       // skip 'subject', name the one you want
```

Named arguments also make calls readable — `Send(to, urgent: true)` beats `Send(to, null, true)`.

❌ **The versioning gotcha:** default values are **copied into the caller at compile time**, not read at runtime.

So if you ship a library, change a default from `false` to `true`, and the calling app **doesn't recompile**, it keeps using the old default. Nothing warns you.

Within one solution everything rebuilds together, so it rarely bites. For a **public library**, prefer an overload:

```csharp
public void Send(string to) => Send(to, false);   // ✅ the default lives in your code
public void Send(string to, bool urgent) { }
```

---

### Q9. Explain `ref`, `out`, and `in`.

**Answer.** All three are parameter modifiers that pass a **reference to the caller's variable** instead of a copy:

- **`ref`** — the method can read and change the variable.
- **`out`** — the method must assign the variable before it returns.
- **`in`** — the method can read the variable but not change it.

They differ mainly in who is required to assign it, and when.

```csharp
void RefExample(ref int x) => x += 1;   // set before the call; method may change it
void OutExample(out int x) => x = 42;   // not set before; method MUST set it
void InExample(in int x)   { }          // set before; method CANNOT change it
```

| | Set before the call? | Can the method change it? | Used for |
|---|---|---|---|
| `ref` | **Yes** | Yes | Modifying the caller's variable |
| `out` | No | **Must** | Returning a second value |
| `in` | **Yes** | No | Passing a big struct without copying |

**`out` is the common one**, from the `TryParse` pattern — a bool for success plus the value:

```csharp
if (int.TryParse(input, out var number))
    Console.WriteLine(number * 2);
```

**`in` is a performance tool** for large structs. A struct is normally copied when passed; `in` passes a reference but keeps it read-only. Not worth it for small types like `int`, where the reference costs as much as the copy.

❌ Don't use `ref`/`out` to return several values. A **tuple** is clearer:

```csharp
(bool ok, int value) TryGet() => (true, 42);
```

---

## I5 — Primary Constructors (C# 12)

### Q10. What are Primary Constructors in C# 12 for classes and structs, how do they differ from `record` primary constructors, and what are the hidden pitfalls?

**Answer.** C# 12 added **Primary Constructors** to normal `class` and `struct` declarations, allowing parameters to be placed directly in the type header.

```csharp
// C# 12 Primary Constructor on a class
public class CustomerService(ICustomerRepository repo, ILogger logger)
{
    public async Task<Customer?> GetAsync(int id)
    {
        logger.LogInformation("Fetching customer {Id}", id);
        return await repo.FindByIdAsync(id);
    }
}
```

#### Critical Difference: `class` vs `record` Primary Constructors

| Feature | `record` Primary Constructor | `class` / `struct` Primary Constructor |
| :--- | :--- | :--- |
| **Auto-Generated Properties** | ✅ Generates `public T Param { get; init; }` | ❌ **No properties generated!** |
| **Mutability** | Immutable positional fields | Parameters are in scope throughout class body |
| **Field Creation** | Public init properties | Compiler captures parameters as private fields *only if referenced* |

#### ⚠️ Hidden Pitfalls in Class Primary Constructors:
1. **Implicit Mutability**: Primary constructor parameters are mutable variables within the class body! Modifying them inside a method mutates the captured state.
2. **Double Storage Memory Bug**: If you declare a public property and assign the primary parameter to it (`public string Id { get; } = id;`), and also reference `id` directly inside a method, the compiler generates **two hidden storage locations** (one for the property, one for the captured parameter).

**Rule**: Refer to the property (`Id`) instead of the captured parameter (`id`) in your class methods to avoid double storage allocation.

