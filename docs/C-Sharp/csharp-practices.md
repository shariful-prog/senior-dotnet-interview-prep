# K. Coding Practices & Testing
---

## K1 — Unit Testing & Mocking

### Q1. What is the AAA pattern, and what makes a good unit test?

**Answer.** **AAA** stands for **Arrange, Act, Assert** — the three steps every unit test follows.

- **Arrange** — set up what you need.
- **Act** — do the one thing you're testing.
- **Assert** — check the result.

```csharp
[Fact]
public void Withdraw_AmountExceedsBalance_Throws()
{
    // Arrange
    var account = new Account(balance: 100m);

    // Act
    Action act = () => account.Withdraw(150m);

    // Assert
    Assert.Throws<InsufficientFundsException>(act);
}
```

Separating the three with blank lines makes a test read as: given this, when I do that, then this happens.

**What makes a test good:**

- **Fast** — no database, no network, no `Thread.Sleep`. Milliseconds.
- **Independent** — order doesn't matter, no shared state between tests.
- **Repeatable** — same result every run. That means no `DateTime.Now` and no `Random` (Q4).
- **Tests one behaviour** — not one `Assert` necessarily, but one idea.
- **Named clearly** — `Method_Scenario_ExpectedResult`, so a failure tells you what broke without opening the file.

---

### Q2. `[Fact]` vs `[Theory]` in xUnit?

**Answer.** **`[Fact]`** is a test that takes no input and runs once. **`[Theory]`** is a test that takes parameters and runs once per set of data.

```csharp
[Fact]
public void Add_TwoAndThree_ReturnsFive()
    => Assert.Equal(5, Calculator.Add(2, 3));

[Theory]
[InlineData(2, 3, 5)]
[InlineData(-1, 1, 0)]
[InlineData(0, 0, 0)]
public void Add_ReturnsSum(int a, int b, int expected)
    => Assert.Equal(expected, Calculator.Add(a, b));
```

The `[Theory]` above runs **three separate tests**. If the second one fails, the other two still pass and the report shows exactly which input broke — much better than one test looping over an array, where the first failure hides the rest.

Use `[InlineData]` for simple constants. For anything else — objects, decimals, data you want to share between tests — use `[MemberData]`:

```csharp
public static IEnumerable<object[]> Orders =>
    [ [new Order(100m), 10m], [new Order(500m), 25m] ];

[Theory]
[MemberData(nameof(Orders))]
public void Discount_IsCorrect(Order order, decimal expected)
    => Assert.Equal(expected, order.Discount());
```

---

### Q3. What are the different kinds of test doubles?

**Answer.** A **test double** is any fake object you pass into a test instead of the real thing. The names come up in interviews, but in practice you'll mostly use two of them.

| Name | What it does |
|---|---|
| **Dummy** | Just fills a parameter. Never actually used. |
| **Stub** | Returns canned answers. *"When asked for user 5, return this user."* |
| **Mock** | Records calls so you can **verify** them. *"Check that Send was called once."* |
| **Fake** | A real working implementation, simplified — like an in-memory repository. |
| **Spy** | A real object that also records what happened to it. |

The distinction that actually matters is **stub vs mock**:

```csharp
// STUB — provides input. You assert on the RESULT.
var repo = new Mock<IUserRepository>();
repo.Setup(r => r.GetById(5)).Returns(new User("Bob"));

var name = new UserService(repo.Object).GetDisplayName(5);
Assert.Equal("Bob", name);                    // ✅ asserting on the outcome

// MOCK — verifies an interaction. You assert on the CALL.
var email = new Mock<IEmailSender>();
new OrderService(email.Object).Place(order);

email.Verify(e => e.Send(It.IsAny<string>(), It.IsAny<string>()), Times.Once);
```

**Prefer stubs.** Verifying calls ties your test to *how* the code works, so harmless refactors break tests. Use `Verify` when the call **is** the behaviour — an email that must be sent, a payment that must be charged.

---

### Q4. How do you test code that depends on the current time?

**Answer.** Don't read the clock inside your class. **Pass the time in**, so a test can decide what "now" is.

**The problem:**

```csharp
public class Order
{
    public DateTime DueDate { get; set; }

    public bool IsOverdue() => DateTime.Now > DueDate;   // ❌ reads the real clock
}
```

You can't test this. `DateTime.Now` is whatever time it is when the test runs, so to check "overdue" you'd have to change your computer's clock or wait until the due date passes.

**The fix — make the clock something you hand in:**

```csharp
public interface IClock { DateTime Now { get; } }

public class SystemClock : IClock { public DateTime Now => DateTime.Now; }   // real app

public class Order(IClock clock)
{
    public DateTime DueDate { get; set; }

    public bool IsOverdue() => clock.Now > DueDate;      // ✅ asks the clock it was given
}
```

Now the class doesn't know where the time comes from. In your app you give it the real clock:

```csharp
var order = new Order(new SystemClock());
```

**In a test you give it a fake clock** set to any date you want:

```csharp
public class FakeClock(DateTime now) : IClock { public DateTime Now => now; }
```

```csharp
[Fact]
public void IsOverdue_WhenPastDueDate_ReturnsTrue()
{
    var clock = new FakeClock(new DateTime(2026, 1, 10));       // pretend it's Jan 10
    var order = new Order(clock) { DueDate = new DateTime(2026, 1, 5) };

    Assert.True(order.IsOverdue());     // runs instantly, same result every time
}

[Fact]
public void IsOverdue_BeforeDueDate_ReturnsFalse()
{
    var clock = new FakeClock(new DateTime(2026, 1, 1));        // pretend it's Jan 1
    var order = new Order(clock) { DueDate = new DateTime(2026, 1, 5) };

    Assert.False(order.IsOverdue());
}
```

Both tests run in milliseconds and give the same answer forever. With `DateTime.Now` the second test would start failing after January 5th.

Since **.NET 8** there's a built-in version of this, so you don't have to write `IClock` yourself — use `TimeProvider`, pass `TimeProvider.System` in your app, and `FakeTimeProvider` in tests. Same idea, standard type.

**The general rule:** anything that changes on its own — the time, `Random`, `Guid.NewGuid()`, the file system — should be **passed in** rather than called directly, so tests can control it.

---

### Q5. What should you not mock?

**Answer.** Mock things you **own** and things that are **slow or external**. Don't mock anything else.

**Don't mock types you don't own:**

```csharp
var http = new Mock<HttpClient>();      // ❌ fragile and often impossible
```

You're guessing at another library's behaviour, and your test passes even when the real library disagrees. Wrap it in your own interface and mock that instead:

```csharp
public interface IPaymentGateway { Task<bool> ChargeAsync(decimal amount); }
var gateway = new Mock<IPaymentGateway>();     // ✅ your interface, your rules
```

**Don't mock things with no behaviour** — DTOs, records, value objects, `List<T>`. Just create a real one:

```csharp
var mock = new Mock<Address>();                          // ❌ pointless
var address = new Address("Main St", "Oslo");            // ✅ simpler and real
```

**Don't mock the thing you're testing.** If you find yourself doing that, the class is doing too much — split it.

Also note **you can only mock what's overridable**: an interface, or a `virtual` member. That's why mocking libraries push you toward interfaces.

---

### Q6. How do you test async code?

**Answer.** Make the test method `async Task` and `await` the call. That's it.

```csharp
[Fact]
public async Task GetUser_ReturnsUser()          // async Task, not async void
{
    var user = await _service.GetUserAsync(5);
    Assert.Equal("Bob", user.Name);
}
```

❌ **Never use `async void`** for a test. The test framework can't wait for it, so it reports a pass before the test has finished — and failures vanish silently.

❌ **Never block with `.Result` or `.Wait()`** — it can deadlock, and it wraps exceptions in `AggregateException`, so `Assert.Throws<MyException>` fails even though your code threw the right thing.

For async exceptions use `ThrowsAsync`:

```csharp
await Assert.ThrowsAsync<InvalidOperationException>(() => _service.FailAsync());
```

Stub async methods with `ReturnsAsync`:

```csharp
repo.Setup(r => r.GetByIdAsync(5)).ReturnsAsync(new User("Bob"));
```

---

### Q7. What's the difference between a unit test and an integration test?

**Answer.** A **unit test** checks one piece of code in isolation, with everything external faked. An **integration test** checks that several real pieces work together.

| | Unit test | Integration test |
|---|---|---|
| Scope | One class | Several parts together |
| Database, files, network | Faked | **Real** |
| Speed | Milliseconds | Seconds |
| When it fails | You know exactly where | Somewhere in the chain |

```csharp
// UNIT — fake repository, tests the discount rule only
var repo = new Mock<IOrderRepository>();
var total = new OrderService(repo.Object).CalculateTotal(order);

// INTEGRATION — real app, real HTTP, real database
var client = _factory.CreateClient();
var response = await client.PostAsJsonAsync("/orders", newOrder);
Assert.Equal(HttpStatusCode.Created, response.StatusCode);
```

You need both. Unit tests catch logic errors fast and pinpoint them. Integration tests catch what unit tests can't see — wrong SQL, bad configuration, broken DI registration, a route that doesn't match.

The usual balance: **many unit tests, fewer integration tests, very few end-to-end tests.** Integration tests are more valuable per test but slow, so you can't have thousands of them.

---

## K2 — Immutability & Thread-Safe Design

### Q8. What is immutability, what are its benefits, and how do you get it in C#?

**Answer.** An **immutable** object is one whose state can't change after it's created. Instead of modifying it, you create a new one.

**Why it helps:**

- **Thread-safe for free.** Nothing can change, so there's nothing to race over and no locks needed.
- **No surprise changes.** Pass it to a method and you know it comes back the same.
- **Safe as a dictionary key or in a cache** — the value can't shift under you.

**How to do it in C#** — the cleanest way is a `record`:

```csharp
public record Money(decimal Amount, string Currency);

var price = new Money(100m, "USD");
var higher = price with { Amount = 150m };    // new object; 'price' is unchanged
```

`with` copies the object and changes only what you name.

For a normal class, use `readonly` fields or get-only properties and set everything in the constructor:

```csharp
public class Money
{
    public decimal Amount { get; }            // no setter
    public string Currency { get; }

    public Money(decimal amount, string currency)
        => (Amount, Currency) = (amount, currency);
}
```

`init` properties are the middle ground — settable when creating the object, fixed afterwards:

```csharp
public class Config { public string Host { get; init; } = ""; }

var c = new Config { Host = "localhost" };    // ✅ allowed here
c.Host = "other";                             // ❌ compile error
```

⚠️ **The catch is that this is only shallow** — see Q9.

---

### Q9. What's the difference between shallow and deep immutability?

**Answer.** **Shallow** means the object's own fields can't be reassigned. **Deep** means nothing reachable through it can change either.

C# gives you shallow by default, and that's the trap:

```csharp
public record Team(string Name, List<string> Members);

var team = new Team("Alpha", new List<string> { "Bob" });

team.Members = new List<string>();   // ❌ compile error — the field is locked
team.Members.Add("Alice");           // ✅ ALLOWED — the list itself is still mutable!
```

The record protects the *reference*, not what it points to. Anyone holding that `Team` can still change its members.

**The fix: use a collection that can't be modified.**

```csharp
public record Team(string Name, IReadOnlyList<string> Members);
// or, genuinely immutable:
public record Team(string Name, ImmutableList<string> Members);
```

⚠️ `IReadOnlyList<T>` only means *the caller can't change it through this interface* — if you pass in a `List<T>`, whoever still holds that list can modify it. `ImmutableList<T>` is the real guarantee. A cheap middle ground is copying on the way in:

```csharp
public Team(string name, IEnumerable<string> members)
    => (Name, Members) = (name, members.ToArray());   // ✅ our own copy
```

**The rule:** a record or `readonly` field only protects one level. If it holds a collection or a mutable object, you must handle that yourself.

---

## K3 — Clean Code

### Q10. What are guard clauses, and why prefer early returns?

**Answer.** A **guard clause** checks for invalid input at the top of a method and exits immediately, instead of wrapping the real work in nested `if` blocks.

```csharp
// ❌ nested — the actual work is buried four levels deep
public void Process(Order order)
{
    if (order != null)
    {
        if (order.Items.Any())
        {
            if (order.Customer != null)
            {
                // the real work, finally
            }
        }
    }
}

// ✅ guard clauses — problems handled first, work stays at the top level
public void Process(Order order)
{
    ArgumentNullException.ThrowIfNull(order);
    if (!order.Items.Any()) throw new ArgumentException("Order has no items");
    ArgumentNullException.ThrowIfNull(order.Customer);

    // the real work — no indentation, easy to read
}
```

Two benefits. **Readability** — the happy path stays flat rather than drifting right, and you read the requirements as a list at the top. **Fail fast** — a bad argument is rejected at the door, not ten lines into the logic where the error will be confusing.

.NET has built-in helpers for the common checks:

```csharp
ArgumentNullException.ThrowIfNull(order);
ArgumentException.ThrowIfNullOrWhiteSpace(name);
ArgumentOutOfRangeException.ThrowIfNegative(amount);
```

The same idea works for returns, not just throws — check the simple cases first and get out:

```csharp
public decimal Discount(Order o)
{
    if (o.Total < 100m) return 0m;          // simple cases first
    if (o.Customer.IsVip) return o.Total * 0.2m;
    return o.Total * 0.1m;
}
```
