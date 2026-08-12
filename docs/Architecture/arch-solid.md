# AA. SOLID Principles
---

## AA0 — Framing

### Q1. What does SOLID stand for, and what's the one-sentence intent of each?

**Answer.** SOLID is five object-oriented design principles (coined by Robert C. Martin) that together aim at code that is **easy to change without breaking** — low coupling, high cohesion, and safe extension.

| Letter | Principle | One-line intent |
|--------|-----------|-----------------|
| **S** | Single Responsibility | A class should have one reason to change. |
| **O** | Open/Closed | Open for extension, closed for modification. |
| **L** | Liskov Substitution | Subtypes must be usable anywhere their base type is, without surprises. |
| **I** | Interface Segregation | Many small, focused interfaces beat one fat one. |
| **D** | Dependency Inversion | Depend on abstractions, not concretions. |

**How to talk about it in an interview.** Don't recite definitions — interviewers hear that from everyone. Lead with the *smell* each principle removes and show a *refactor*. The senior signal is knowing **when applying a principle is worth it and when it's premature abstraction**. See Q14.

---

## AA1 — Single Responsibility Principle (SRP)

### Q2. What is SRP, really? "One responsibility" is vague.

**Answer.** The precise formulation is: **a class should have only one reason to change** — meaning it should be answerable to a single *actor* / stakeholder. "Responsibility" is best read as "a source of change," not "does one thing."

A class that formats a report *and* persists it has two reasons to change: the reporting/business team changes the format; the DBA/infra team changes persistence. Those are different actors on different schedules — split them.

```csharp
// Before: three reasons to change bundled together
public class InvoiceService
{
 public decimal CalculateTotal(Invoice inv) { /* business rules */ }
 public string RenderHtml(Invoice inv) { /* presentation */ }
 public void SaveToDatabase(Invoice inv) { /* persistence */ }
}
```

```csharp
// After: each responsibility owned by one class / actor
public class InvoiceCalculator // changes when business rules change
{
 public decimal CalculateTotal(Invoice inv) { /* ... */ }
}

public class InvoiceHtmlRenderer // changes when presentation changes
{
 public string Render(Invoice inv) { /* ... */ }
}

public class InvoiceRepository // changes when persistence changes
{
 public Task SaveAsync(Invoice inv) { /* ... */ }
}
```

**Gotcha.** SRP is *not* "one method per class" or "make everything tiny." Over-applied, it produces a swarm of anemic single-method classes that are harder to follow than the thing you split. The test is **"who asks for this to change?"** — if two chunks of code always change together for the same reason, they belong together (that's cohesion).

---

### Q3. How is SRP related to cohesion and coupling?

**Answer.** They're the same idea from two angles:

- **Cohesion** — how strongly the members of a module belong together. SRP *maximizes cohesion*: everything in the class serves the one responsibility.
- **Coupling** — how much one module depends on another. Splitting responsibilities *reduces coupling* because a change in one area no longer forces recompilation/retesting of unrelated code.

The goal is **high cohesion, low coupling.** A god class is the failure mode of both: it's low-cohesion (unrelated things crammed together) and high-coupling (everything depends on it, so everything breaks when it changes).

---

## AA2 — Open/Closed Principle (OCP)

### Q4. What is OCP and how do you achieve it in C#?

**Answer.** **Open for extension, closed for modification** — you should be able to add new behavior *without editing existing, tested code*. 


##### Bad Example (Violates OCP)

```csharp
public class PaymentService
{
    public void Pay(string paymentType)
    {
        if (paymentType == "CreditCard")
        {
            // Credit card payment
        }
        else if (paymentType == "PayPal")
        {
            // PayPal payment
        }
    }
}
```

##### Problem

When a new payment method such as **Bkash** is introduced, you must modify the existing class.

```csharp
else if (paymentType == "Bkash")
{
    // Bkash payment
}
```

Every new payment method requires changing `PaymentService`, increasing the risk of introducing bugs.

#### Good Example (Follows OCP)

##### Step 1: Create an abstraction

```csharp
public interface IPaymentMethod
{
    void Pay();
}
```

##### Step 2: Implement different payment methods

```csharp
public class CreditCardPayment : IPaymentMethod
{
    public void Pay()
    {
        Console.WriteLine("Paid by Credit Card");
    }
}

public class PayPalPayment : IPaymentMethod
{
    public void Pay()
    {
        Console.WriteLine("Paid by PayPal");
    }
}
```

##### Step 3: Depend on the abstraction

```csharp
public class PaymentService
{
    public void Pay(IPaymentMethod paymentMethod)
    {
        paymentMethod.Pay();
    }
}

// call it 
IPaymentMethod payment = new CreditCardPayment();

PaymentService paymentService = new PaymentService();

paymentService.Pay(payment);
```

##### Add a new payment method

```csharp
public class BkashPayment : IPaymentMethod
{
    public void Pay()
    {
        Console.WriteLine("Paid by Bkash");
    }
}
```

**Gotcha / senior nuance.** OCP is not "never edit code." You *can't* predict every axis of change, and abstracting speculatively is over-engineering (see Q14). The pragmatic rule: **the first time** you write the switch, leave it; **the second or third time** you're editing it for a new case, that's the signal to invert it to OCP. Abstract on demonstrated change, not imagined change.

---

## AA3 — Liskov Substitution Principle (LSP)

### Q5. State LSP precisely and give a real violation.

**Answer.** A child class should be able to be used anywhere the parent class is used, without causing errors or changing the expected behavior.

Wherever a parent object is expected, a child object should work correctly without changing the correctness of the program.

#### ❌ A real violation: payment refunds

An online shop takes payments. Every payment method can charge and refund:

```csharp
public interface IPaymentMethod
{
    void Charge(decimal amount);
    void Refund(decimal amount);
}
```

Card payments work fine:

```csharp
public class CreditCardPayment : IPaymentMethod
{
    public void Charge(decimal amount) { /* calls Stripe */ }
    public void Refund(decimal amount) { /* calls Stripe */ }
}
```

Then the business adds gift cards. Gift cards **cannot be refunded** — so the developer does this:

```csharp
public class GiftCardPayment : IPaymentMethod
{
    public void Charge(decimal amount) { /* deduct from balance */ }

    public void Refund(decimal amount)
    {
        throw new NotSupportedException("Gift cards cannot be refunded");  // ❌
    }
}
```

Now look at the code that processes returns. It was written months ago and is perfectly correct:

```csharp
public class ReturnsService
{
    private readonly IPaymentMethod _payment;

    public ReturnsService(IPaymentMethod payment)
    {
        _payment = payment;
    }

    public void ProcessReturn(Order order)
    {
        _payment.Refund(order.Total);      // any IPaymentMethod can refund... right?
        order.Status = "Refunded";
    }
}
```

It works with a card and crashes with a gift card:

```csharp
new ReturnsService(new CreditCardPayment()).ProcessReturn(order);   // ✅ fine
new ReturnsService(new GiftCardPayment()).ProcessReturn(order);     // ❌ crash
```

#### Why this breaks LSP

`ReturnsService` depends on `IPaymentMethod`, so it is entitled to assume **every** payment method can refund. `GiftCardPayment` claims to be an `IPaymentMethod` but is not usable as one — it throws an exception the interface never mentioned, which breaks rule four.

The real damage: **the compiler cannot help you.** `GiftCardPayment` compiles, `ReturnsService` compiles, the DI registration compiles. You discover the problem when a customer returns an item bought with a gift card, in production.

#### ✅ The fix

The mistake was putting `Refund` on an interface not every payment method can honour. Split it so the type says what it can actually do:

```csharp
public interface IPaymentMethod           // everything can charge
{
    void Charge(decimal amount);
}

public interface IRefundable               // only some can refund
{
    void Refund(decimal amount);
}
```

```csharp
public class CreditCardPayment : IPaymentMethod, IRefundable
{
    public void Charge(decimal amount) { /* ... */ }
    public void Refund(decimal amount) { /* ... */ }
}

public class GiftCardPayment : IPaymentMethod     // no Refund — and that's honest
{
    public void Charge(decimal amount) { /* ... */ }
}
```

`ReturnsService` now asks for what it needs:

```csharp
public class ReturnsService
{
    private readonly IRefundable _payment;

    public ReturnsService(IRefundable payment)     // gift cards can't be passed here
    {
        _payment = payment;
    }
}
```

Passing a `GiftCardPayment` is now a **compile error**, not a production crash. The capability lives in the type system instead of a runtime check.

#### How to spot LSP violations in review

An implementation that:

- throws `NotSupportedException`
- silently does nothing
- returns `null` where the base never does
- rejects input the base accepted

A violation does not have to throw, either. A `Square : Rectangle` that couples its setters, so `Width = 10; Height = 4` gives an area of 16, raises no exception — it just returns a wrong answer, which is harder to find than a crash.

**Interview line:** "An override that throws `NotSupportedException`, returns `null` where the base never does, weakens a promise, or silently no-ops is an LSP red flag — the type is lying about being substitutable."

---

## AA4 — Interface Segregation Principle (ISP)

### Q7. What is ISP and what smell does it fix?

**Answer.** **No client should be forced to depend on methods it does not use.** Prefer several small, role-focused interfaces over one large "fat" interface. The smell is an interface where implementers are forced to stub out or throw on members they can't support.

```csharp
// Before: fat interface — a SimplePrinter can't fax or scan
public interface IMachine
{
 void Print(Document d);
 void Scan(Document d);
 void Fax(Document d);
}

public class SimplePrinter : IMachine
{
 public void Print(Document d) { /* ok */ }
 public void Scan(Document d) => throw new NotSupportedException(); // forced stub
 public void Fax(Document d) => throw new NotSupportedException(); // forced stub
}
```

```csharp
// After: segregated roles — implement only what you support
public interface IPrinter { void Print(Document d); }
public interface IScanner { void Scan(Document d); }
public interface IFax { void Fax(Document d); }

public class SimplePrinter : IPrinter
{
 public void Print(Document d) { /* ok */ }
}

// A multifunction device composes the roles it actually provides
public class AllInOne : IPrinter, IScanner, IFax { /* ... */ }
```

**Gotcha.** ISP is client-driven: segregate by **how consumers use the type**, not arbitrarily. Splitting a cohesive interface into single-method fragments nobody uses separately is just noise. The BCL's `IReadOnlyCollection<T>` / `ICollection<T>` / `IList<T>` split is ISP done right — read-only consumers depend only on the read-only slice.

---

### Q7b. Are LSP and ISP the same thing? They both seem to be about `NotSupportedException`.

**Answer.** They are different principles that happen to share a symptom. The `NotSupportedException` stub is the visible smell; LSP and ISP are two different explanations of what went wrong.

|  | LSP | ISP |
|---|---|---|
| Concerns | the **implementer** honouring a contract | the **interface** being too big |
| Question it asks | "can callers substitute this safely?" | "is this client forced to depend on things it never calls?" |
| Violated by | a type that claims a contract it can't fulfil | an interface that bundles unrelated roles |
| Typical fix | don't declare the capability | split the interface by role |
| Broken without the other? | yes — one method, still substitutable badly | yes — fat interface everyone fully implements |

**Why they get confused:** a fat interface *causes* LSP violations. Force `SimplePrinter` to implement `Scan` and it has no option but to throw, so an ISP failure produces an LSP failure. That is why the fix so often satisfies both at once.

**Each can be broken alone.** That is the clearest way to see they are separate.

**ISP violated, LSP intact** — the interface is too fat, but every implementer honours all of it:

An interface with three ways to notify a user:

```csharp
public interface INotificationService
{
    void SendEmail(string to, string message);
    void SendSms(string number, string message);
    void SendPush(string deviceId, string message);
}
```

The class that implements it supports all three. Nothing throws, so **LSP is fine** — you can substitute it anywhere:

```csharp
public class NotificationService : INotificationService
{
    public void SendEmail(string to, string message) { /* works */ }
    public void SendSms(string number, string message) { /* works */ }
    public void SendPush(string deviceId, string message) { /* works */ }
}
```

But look at a class that only sends email:

```csharp
public class PasswordResetService
{
    private readonly INotificationService _notifications;

    public PasswordResetService(INotificationService notifications)
    {
        _notifications = notifications;      // it gets all three methods...
    }

    public void Reset(User user)
    {
        _notifications.SendEmail(user.Email, "Reset your password");   // ...uses one
    }
}
```

**That is the ISP violation:** `PasswordResetService` depends on SMS and push even though it only sends email. Two problems follow:

- **Testing it means faking methods it never calls** — you must mock `SendSms` and `SendPush` to test a password reset.
- **Unrelated changes reach it.** Add `SendSlack` to the interface and this class is affected, even though it has nothing to do with Slack.

**The fix** — ask for only what you use:

```csharp
public interface IEmailSender
{
    void SendEmail(string to, string message);
}

public class PasswordResetService
{
    private readonly IEmailSender _email;

    public PasswordResetService(IEmailSender email)    // only email now
    {
        _email = email;
    }
}
```

Note that `PasswordResetService` never implements the interface — it **receives** it in the constructor, because it does not send notifications itself, it asks something else to. ISP is about what a class *depends on*.

**LSP violated, ISP intact** — a single-method interface, so it cannot be "too fat", yet substitution still breaks:

```csharp
public interface IDiscountPolicy
{
    decimal Apply(decimal price);   // contract: returns a price >= 0
}

public class SeasonalDiscount : IDiscountPolicy
{
    public decimal Apply(decimal price) => price * 0.9m;      // ✅ honours it
}

public class AggressiveDiscount : IDiscountPolicy
{
    public decimal Apply(decimal price) => price - 100m;      // ❌ can go negative
}
```

Nothing throws and nothing is stubbed. `AggressiveDiscount` just weakens the post-condition, so a caller that trusted the result to be non-negative now writes a negative total to the database. One method, perfectly segregated, still not substitutable.

**How to tell which one you are looking at:**

- Implementers are forced to stub members → **ISP**, split the interface.
- One implementer misbehaves while its siblings are fine → **LSP**, that type is wrong.
- Both → the fat interface caused it. Segregate first, and the LSP problem usually disappears.

## AA5 — Dependency Inversion Principle (DIP)

### Q8. State DIP and show the inversion.

**Answer.** Two clauses:

1. **High-level modules must not depend on low-level modules. Both must depend on abstractions.**
2. **Abstractions must not depend on details. Details must depend on abstractions.**

The terms have specific meanings:

- **High-level module** — code that holds business policy. What the system does: validate an order, apply a discount, place it.
- **Low-level module** — code that performs a mechanism. How it is carried out: a SQL query, an HTTP call, a file write.
- **Abstraction** — an interface or abstract class declaring *what* is needed, with no implementation.
- **Detail** — a concrete implementation of an abstraction.

Policy is stable and specific to your business. Mechanisms are replaceable. A dependency from policy to mechanism means the stable code must change whenever the replaceable code does.

**Before — the high-level module depends directly on a detail:**

```csharp
public class OrderService
{
    private readonly SqlOrderRepository _repo = new SqlOrderRepository();   // hard-wired

    public void Place(Order order)
    {
        // business rules here
        _repo.Save(order);
    }
}
```

This violates both clauses. `OrderService` (policy) references `SqlOrderRepository` (detail) directly, and constructs it with `new`, so the dependency is fixed at compile time. Three consequences: the class cannot be tested without a database, the storage technology cannot be replaced without editing it, and a change to the repository's constructor forces a change to business code.

**After — both depend on an abstraction:**

```csharp
public interface IOrderRepository        // declared by the high-level module
{
    void Save(Order order);
}
```

```csharp
public class OrderService
{
    private readonly IOrderRepository _repo;

    public OrderService(IOrderRepository repo)     // supplied by the caller
    {
        _repo = repo;
    }

    public void Place(Order order)
    {
        _repo.Save(order);
    }
}
```

```csharp
public class SqlOrderRepository : IOrderRepository      // the detail implements it
{
    public void Save(Order order) { /* SQL here */ }
}
```

#### What is inverted

The direction of the compile-time dependency. Before, policy referenced the detail:

```text
OrderService  ───────────────►  SqlOrderRepository
(high-level)                    (low-level)
```

After, the detail references an abstraction declared by the policy:

```text
OrderService  ──►  IOrderRepository  ◄──  SqlOrderRepository
(high-level)       (abstraction)          (low-level)
```

`OrderService` contains no reference to `SqlOrderRepository`. The low-level module is now the one constrained by the abstraction. That reversal of dependency direction is the inversion.

**Ownership of the abstraction is part of the principle.** `IOrderRepository` must be declared with the high-level module, and its members must be expressed in terms the high-level module needs — `Save(Order)`, not `ExecuteNonQuery(string)`. If the interface is declared alongside the low-level module and mirrors its API, the dependency direction is unchanged and no inversion has occurred.

**Consequences:**

- **Testability** — a test supplies a stub implementation; no database is required.
- **Substitutability** — a second implementation such as `MongoOrderRepository` requires only a change to the DI registration.
- **Isolation** — changes to low-level code cannot propagate into high-level code, since the reference does not exist.

This is the mechanism that keeps the domain free of infrastructure in Clean and Onion architecture — see [arch-clean-onion.md](arch-clean-onion.md).

---

### Q9. DIP vs Dependency Injection vs IoC — aren't they the same thing?

**Answer.** No — this is a favorite senior-level clarifier. They operate at different levels:

### IoC (Inversion of Control)

A **design concept** where the responsibility for creating and managing objects is transferred from your code to a framework or container. Instead of creating dependencies yourself, something else provides them.

### Dependency Injection (DI)

A **technique for implementing IoC**. Dependencies are provided (injected) into a class—typically through its constructor—instead of the class creating them itself.

---

## AA6 — Judgment & trade-offs

### Q10. When does applying SOLID become over-engineering?

**Answer.** SOLID is a set of *forces*, not laws — each has a cost (indirection, more types, harder navigation). Over-applied, you get "abstraction astronaut" code: an interface with a single implementation that'll never have a second, a factory for a factory, five files to follow one call.

Senior heuristics:
- **YAGNI first.** Don't invert/abstract for a variation that doesn't exist yet. Add the seam when the *second* case appears (rule of three).
- **An interface with exactly one implementation, forever, adds cost with no payoff** — unless you need it for a *test seam* or a *plugin boundary*. "I might swap the database someday" rarely materializes.
- **Cohesion beats dogmatic SRP.** If two things always change together, keeping them in one class is *more* maintainable than splitting them.
- **Optimize for change you can see coming**, not every conceivable axis. The value of SOLID is realized at the *second* change, not the first write.

The mature framing: *"SOLID reduces the cost of change — but abstraction itself is a cost. Apply it where change is likely, not everywhere."*

---

### Q11. Give a single example that violates several SOLID principles at once, then fix it.

**Answer.** A god "manager" class:

```csharp
// Violates SRP (3 responsibilities), OCP (switch on type), DIP (new's concretes)
public class NotificationManager
{
 public void Send(string type, string to, string msg)
 {
 if (type == "email")
 {
 var smtp = new SmtpClient("smtp.corp.com"); // DIP: hard-wired detail
 smtp.Send(to, msg);
 }
 else if (type == "sms") // OCP: edit to add a channel
 {
 var twilio = new TwilioClient("key");
 twilio.SendSms(to, msg);
 }
 // logging, formatting, retry all crammed in here too → SRP
 }
}
```

```csharp
// Fixed: abstraction + strategy selection, each channel isolated
public interface INotificationChannel
{
 bool CanHandle(NotificationType type);
 Task SendAsync(string to, string message);
}

public class EmailChannel : INotificationChannel { /* only email */ }
public class SmsChannel : INotificationChannel { /* only sms */ }

public class NotificationService
{
 private readonly IEnumerable<INotificationChannel> _channels;
 public NotificationService(IEnumerable<INotificationChannel> channels) => _channels = channels;

 public Task SendAsync(NotificationType type, string to, string msg)
 {
 var channel = _channels.First(c => c.CanHandle(type)); // OCP: new channel = new class
 return channel.SendAsync(to, msg); // DIP: depends on abstraction
 }
}
```

Adding push notifications = add `PushChannel : INotificationChannel` and register it. Nothing else changes. This is SRP + OCP + DIP together, and the channel selection is the Strategy pattern.

---

## AA — Cheat Sheet

| Principle | One-liner | Smell it fixes | Fix |
|-----------|-----------|----------------|-----|
| **SRP** | One reason to change | God class; unrelated methods | Split by *actor*/source of change |
| **OCP** | Extend without modifying | Growing `switch`/`if` on type | Abstraction + new implementations (Strategy) |
| **LSP** | Subtypes substitutable | `NotSupportedException` override; behavior surprise | Model by contract; segregate interface |
| **ISP** | No fat interfaces | Implementers stub/throw unused members | Small role-based interfaces |
| **DIP** | Depend on abstractions | High-level `new`s concrete low-level | Inject an abstraction the high level owns |

**DIP vs DI vs IoC:** DIP = principle (depend on abstractions) · DI = technique (inject dependencies) · IoC = umbrella pattern (framework calls you) · container = tool that automates DI.

**Real LSP violations:** subtype weakens a post-condition (CDN store where content isn't available right after `Save`); a read-only subclass throwing `NotSupportedException` on `Add`.

**ISP done right in the BCL:** `IReadOnlyList<T>` / `ICollection<T>` / `IList<T>`.

**When to stop (anti-over-engineering):**
- Rule of three — abstract on the *second/third* variation, not the first.
- One implementation forever = probably YAGNI (unless test seam / plugin point).
- High cohesion > dogmatic SRP: things that change together stay together.
- SOLID's payoff is at the *second change*; abstraction is a real cost.

**Interview move:** never recite definitions — name the *smell*, show *before/after*, then state *when it'd be overkill*.
