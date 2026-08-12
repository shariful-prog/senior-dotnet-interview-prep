# B. OOP & Type Design
---

## B1 — Encapsulation, Inheritance, Polymorphism, Abstraction

### Q1. What are the four pillars of OOP?

**Answer.** OOP has four main ideas. Let's use a real feature every online store needs — **taking a payment at checkout**, where a customer can pay by card, PayPal, or bank transfer.

- **Encapsulation** — means hiding an object's data and controlling how it is accessed or modified. Data is kept private, and access is provided through public methods or properties. Protects data from invalid changes and makes the code more secure and maintainable.

 *In our example:* a `CreditCardPayment` holds the card number `private`. Other code can call `Charge(amount)` but can never read the raw number, so it can't be leaked or corrupted from outside.

- **Abstraction** — Abstraction means hiding the implementation details and showing only the essential features to the user. It focuses on what an object does, not how it does it. It is commonly achieved using abstract classes and interfaces.

 *In our example:* the checkout depends on `IPaymentMethod` with one action, `Charge(amount)`. It never sees the Stripe calls, retries, or currency handling hidden behind that contract.

- **Inheritance** — *creating a new type from an existing one so it automatically gets the base type's members and can add or change its own.* It models an "is-a" relationship and lets you write shared code once.
 *In our example:* `CreditCardPayment` and `PayPalPayment` both inherit from a base `PaymentMethod`, so shared logic like logging and fees is written once instead of copied into each.

- **Polymorphism** — *one call working on many types, where the actual object decides which version of the method runs at runtime.* This lets code work with a general type without knowing the specific one.
 *In our example:* the checkout calls `payment.Charge(amount)` on an `IPaymentMethod`. C# runs the card's or PayPal's version based on the real object — so adding Apple Pay later needs no change to the checkout.

---

### Q2. What is the difference between abstraction and encapsulation?

**Answer.** People confuse them because both "hide" something — but they hide *different things* and solve *different problems*.

- **Encapsulation** — *hides the data and inner workings of a single object, and controls how they're accessed.* It's about **protection**: the object guards its own state so no outside code can put it in an invalid state. Mechanism: `private`/`protected`, properties instead of public fields, methods that enforce the rules.
- **Abstraction** — *hides complexity behind a simpler model, exposing only the essential actions.* It's about **simplification**: callers work with a clean idea of *what* a thing does and ignore *how* it's built. Mechanism: `interface` and `abstract` types that declare the essential operations.

The clearest way to keep them apart:

| | Encapsulation | Abstraction |
|---|---|---|
| **Hides** | The *how* — internal data and implementation | The *how* — but at the level of a whole concept |
| **Goal** | **Protect** state from misuse | **Simplify** a complex thing to its essentials |
| **Level** | One object protecting its own fields | The design/contract a caller programs against |
| **Tools** | `private`, `protected`, properties | `interface`, `abstract` classes |

**Payment example:** a `CreditCardPayment` keeps the card number `private` — that's *encapsulation*, protecting its data. The checkout depends only on `IPaymentMethod.Charge(amount)` and never sees the Stripe calls behind it — that's *abstraction*, simplifying a complex operation to one action.

A one-liner: **encapsulation is protecting the data; abstraction is simplifying the design.** They usually work together — you abstract *what* an object does, and encapsulate *how* it does it.

---

### Q3. What is polymorphism, and what are its two forms in C#?

**Answer.** Polymorphism is the ability of a single interface, method, or object reference to represent multiple behaviors. The actual behavior depends on the object being used or the method signature. C# has two kinds:

- **Compile-time (static) polymorphism** — resolved by the compiler: **method overloading** (same name, different parameter lists) and **generics**. The right method is chosen at compile time from the argument types.
- **Runtime (dynamic) polymorphism** — resolved at runtime via **virtual dispatch**: a `virtual` method called through a base reference runs the **derived override** based on the object's actual type.

```csharp
// Overloading — compile-time
int Add(int a, int b) => a + b;
double Add(double a, double b) => a + b;

// Overriding — runtime
class Animal { public virtual string Speak() => "..."; }
class Dog : Animal { public override string Speak() => "Woof"; }

Animal a = new Dog();
a.Speak(); // "Woof" — resolved at runtime by the actual type
```

The distinction interviewers want: **overloading** is *ad hoc* (compile-time, by signature); **overriding** is *subtype* polymorphism (runtime, by the virtual method table).

---

### Q4. What is composition, and why is "favor composition over inheritance" common advice?

**Answer.** **Composition** — means building a class by **using other classes** instead of inheriting from them. It represents a **has-a** relationship.It models a **"has-a"** relationship rather than the **"is-a"** relationship of inheritance.

Take the payment example. A checkout `PaymentProcessor` needs to charge a card, then send a receipt email. Two ways to give it the email ability:

- **Inheritance ("is-a"):** `class PaymentProcessor : EmailSender`. But a payment processor *isn't* a kind of email sender — that relationship is wrong, and you're now locked to `EmailSender` forever.
- **Composition ("has-a"):** the processor *holds* an `IEmailSender` and calls it. That's the correct relationship, and it stays flexible.

```csharp
interface IEmailSender { void Send(string to, string body); }

class PaymentProcessor
{
 private readonly IEmailSender _email; // composition — HAS an email sender
 public PaymentProcessor(IEmailSender email) => _email = email; // injected in

 public void Charge(IPaymentMethod payment, decimal amount, string customerEmail)
 {
 payment.Charge(amount);
 _email.Send(customerEmail, $"Paid {amount:C}"); // delegate the work
 }
}
```

**Why "favor composition over inheritance"?** Inheritance is powerful but rigid:

- **Tight coupling** — a subclass depends on its base's internal details, so changing the base can silently break every subclass (the "fragile base class" problem).
- **Fixed at compile time** — a type's base is set in code and can't be swapped at runtime.
- **Only one base allowed** — C# permits a single base class, so that one inheritance "slot" is spent quickly.

Composition avoids all three: you inject whatever collaborator you need, swap it at runtime, and — crucially — **test easily by passing a fake**. In the code above, a unit test hands in a `FakeEmailSender` and asserts an email was sent, with no real email leaving the machine.

> **Rule of thumb:** use **inheritance** only for a genuine, stable *"is-a"* relationship (a `CreditCardPayment` *is a* `PaymentMethod`). Use **composition** for *"has-a" / "uses-a"* (a `PaymentProcessor` *has an* email sender) and whenever you need flexibility, testability, or runtime swapping. This is also the "D" in SOLID — depend on abstractions — in action.

---

## B2 — Abstract vs Interface (+ Default Interface Methods)

### Q1. What is the difference between an abstract class and an interface?

**Answer.** The simplest way to put it:

- An **interface** defines a **contract** — it specifies *what* a class must do, but not *how*. It has no code and no data of its own.
- An **abstract class** provides a **base with shared implementation** that derived classes inherit, while still forcing certain members to be implemented.

So an interface is a promise; an abstract class is a partly-finished parent that hands down real code and leaves a few blanks to fill in.

In the payment example, card, PayPal, and bank transfer all share real code — logging, fees, receipts. That shared code belongs in an **abstract class**: `PaymentMethod` writes the common parts once, and each type only fills in `Charge`.

```csharp
abstract class PaymentMethod
{
 protected decimal ApplyFee(decimal amount) => amount * 1.03m; // shared code, written once
 public abstract PaymentResult Charge(decimal amount); // blank — each type must implement it
}
```

An **interface** is for a capability, with no shared code. "Can be refunded" is just an ability — an `Order` or a `Subscription` could have it too, even though they aren't payments:

```csharp
interface IRefundable // just the contract — no code
{
 RefundResult Refund(decimal amount);
}

class CreditCardPayment : PaymentMethod, IRefundable // one base class + any interfaces
{
 public override PaymentResult Charge(decimal amount) { /* ... */ return PaymentResult.Success; }
 public RefundResult Refund(decimal amount) { /* ... */ return RefundResult.Ok; }
}
```

Two more practical differences:

- A class can inherit **only one** abstract class, but implement **many** interfaces (as `CreditCardPayment` does above).
- An abstract class can have **constructors and fields (data)**; an interface cannot hold data.

Rule of thumb: use an **abstract class** when types share real **code or data**; use an **interface** to define a **capability** that even unrelated types can offer, or when a type needs to fulfill several contracts at once.

---

### Q2. When would you choose an interface over an abstract class, and vice versa?

**Answer.** Decision guidance:

Choose an **interface** when you want to define a **contract** that multiple unrelated classes can implement. Interfaces are ideal when different classes need to expose the same behavior, and they allow a class to implement **multiple interfaces**.

Choose an **abstract class** when you have closely related classes that share common state or behavior. An abstract class lets you provide **partial implementation**, define **fields**, **constructors**, and both **abstract** and **concrete methods**, reducing code duplication.

#### Rule of Thumb

- **Interface** → **"Can do"** (capability/contract)
- **Abstract Class** → **"Is a"** (shared base type and implementation)

#### Example

- **`ILogger`** → **Interface**, because many different classes can provide logging.
- **`Employee`** → **Abstract class**, because **FullTimeEmployee**, **PartTimeEmployee**, and **ContractEmployee** all share common properties (such as `Name`, `EmployeeId`, and `Salary`) and common behavior, while each calculates benefits or pay differently.
---

### Q3. Can a class implement multiple interfaces with the same method signature? How do you resolve conflicts?

**Answer.** Yes — a class can implement any number of interfaces. If two interfaces declare the **same member**, a single implementation satisfies both. When you need **different** behavior per interface, use **explicit interface implementation**: qualify the method with the interface name, and it becomes callable only through a reference of that interface type.

```csharp
interface IEnglish { string Greet(); }
interface IFrench { string Greet(); }

class Greeter : IEnglish, IFrench
{
 string IEnglish.Greet() => "Hello"; // explicit — no access modifier
 string IFrench.Greet() => "Bonjour";
}

var g = new Greeter();
// g.Greet(); // won't compile — no public Greet
((IEnglish)g).Greet(); // "Hello"
((IFrench)g).Greet(); // "Bonjour"
```

Explicit implementation is also used to **hide** a member from the class's public surface (e.g. implement `IEnumerable.GetEnumerator()` explicitly while exposing a strongly-typed one) and to resolve ambiguity when two interfaces collide.

---

### Q4. What are default interface methods (C# 8), and why were they added?

**Answer.** A **default interface method** lets an interface provide a **method body**, so implementers can inherit that behavior without writing it themselves:

```csharp
interface ILogger
{
 void Log(string message);
 // default implementation — implementers get this for free
 void LogError(string message) => Log($"ERROR: {message}");
}

class ConsoleLogger : ILogger
{
 public void Log(string message) => Console.WriteLine(message);
 // LogError is inherited from the interface's default
}
```

**Default Interface Methods** were introduced primarily to support **API evolution (interface versioning)**. Before **C# 8**, adding a new method to a public interface was a breaking change because every class implementing that interface had to implement the new method, causing existing code to fail to compile.

With **Default Interface Methods**, the interface can provide a **default implementation** for the new method. This allows existing implementations to continue working without modification, while still giving developers the option to **override** the default behavior if needed.

**Example:**

**Before C# 8:**

```csharp
public interface ILogger
{
    void Log(string message);
}
```

Later, adding:

```csharp
void LogError(string message);
```

would break every existing `ILogger` implementation.

**With C# 8:**

```csharp
public interface ILogger
{
    void Log(string message);

    void LogError(string message)
    {
        Log($"ERROR: {message}");
    }
}
```

Existing classes continue to work without implementing `LogError()`, making it much easier for library authors to evolve their APIs without breaking existing applications.

Key limitations and nuances:
- An interface still **cannot hold instance fields/state** — only static fields (since C# 8/11), constants, and method bodies.
- A default method is **only callable through the interface**, not through the concrete class reference, unless the class re-declares it: `((ILogger)obj).LogError(...)` works, `obj.LogError(...)` may not.
- Overuse blurs the interface/abstract-class distinction. Treat default methods as an **evolution/convenience tool**, not the default design; don't smuggle real behavior into interfaces that belongs in a base class.

---

## B3 — virtual / override / new / sealed

### Q1. What do `virtual` and `override` do?

**Answer.** `virtual` marks a base-class member as **overridable**; `override` provides a new implementation in a derived class that **replaces** the base one under **virtual dispatch**. A call through a base-typed reference runs the *most-derived* override based on the object's **runtime type**.

```csharp
class Base { public virtual void Show() => Console.WriteLine("Base"); }
class Derived : Base { public override void Show() => Console.WriteLine("Derived"); }

Base b = new Derived();
b.Show(); // "Derived" — virtual dispatch picks the runtime type's override
```

Without `virtual`, a method is non-virtual and calls bind to the **static** type. `override` requires the base member to be `virtual`, `abstract`, or itself an `override`. An `override` method is implicitly virtual and can be overridden further down the chain (unless `sealed`).

---

### Q2. What is the difference between `override` and `new` (method hiding)?

**Answer.** 
-  `override` **replaces** a virtual or abstract method from the base class. Calls are resolved based on the **actual object type** (runtime polymorphism).
-  `new` **hides** the base class method instead of overriding it. The method called depends on the **reference type**.

```csharp
class Base { public virtual void Show() => Console.WriteLine("Base"); }
class Over : Base { public override void Show() => Console.WriteLine("Over"); }
class Hide : Base { public new void Show() => Console.WriteLine("Hide"); }

Base o = new Over(); o.Show(); // "Over" — override → runtime type wins
Base h = new Hide(); h.Show(); // "Base" — new → static type (Base) wins!
((Hide)h).Show(); // "Hide" — call through the static Hide type
```

#### When does `new` (method hiding) work?

```csharp
public class Animal
{
    public void Speak() => Console.WriteLine("Animal");
}

public class Dog : Animal
{
    public new void Speak() => Console.WriteLine("Dog");
}

Dog dog = new Dog();
dog.Speak(); // ✅ Dog (reference type is Dog)

Animal animal = new Dog();
animal.Speak(); // ❌ Animal (reference type is Animal)
```

---

### Q3. What does `sealed` do, and why use it?

**Answer.** `sealed` prevents a class from being inherited or a method from being overridden.

#### Sealed Class

A sealed class cannot be used as a base class.

```csharp
public sealed class Logger
{
    public void Log()
    {
        Console.WriteLine("Logging...");
    }
}

// Not allowed
public class FileLogger : Logger
{
}
```

#### Sealed Method

A sealed method stops further overriding in derived classes.

```csharp
public class Animal
{
    public virtual void Speak()
    {
        Console.WriteLine("Animal");
    }
}

public class Dog : Animal
{
    public sealed override void Speak()
    {
        Console.WriteLine("Dog");
    }
}

public class Puppy : Dog
{
    // Not allowed
    public override void Speak()
    {
    }
}
```

---

### Q4. What is an abstract method, and how does it differ from a virtual one?

**Answer.** 

-  An abstract method has **no implementation** in the base class. A derived class **must override** it.
-  A virtual method has a **default implementation** in the base class. A derived class **can override it**, but it is not required.

---

### Q5. Can you call a base-class implementation from an override? How does constructor call order work?

**Answer.** Yes. Use `base` to call the parent class implementation.

Imagine a payment system. The base class handles common payment steps:

```csharp
public class PaymentProcessor
{
    public virtual void ProcessPayment()
    {
        Console.WriteLine("Validate payment details");
        Console.WriteLine("Save transaction");
    }
}

public class CreditCardPayment : PaymentProcessor
{
    public override void ProcessPayment()
    {
        base.ProcessPayment(); // reuse common payment steps

        Console.WriteLine("Charge credit card");
        Console.WriteLine("Send receipt");
    }
}

var payment = new CreditCardPayment();
payment.ProcessPayment();
```

Output:

```
Validate payment details
Save transaction
Charge credit card
Send receipt
```

`base.ProcessPayment()` allows `CreditCardPayment` to reuse the common payment logic and add its own specific behavior.

---

## B4 — Access Modifiers

### Q1. What are the access modifiers in C# and what does each mean?

C# has six accessibility levels that control where a type or member can be accessed:

- **`public`** → Accessible from anywhere, including other assemblies. Used for members that should be exposed externally.

- **`private`** → Accessible only within the containing class or struct. This is the default access modifier for class members and is used to hide internal implementation details.

- **`protected`** → Accessible within the containing class and its derived classes. Used when child classes need access to inherited members.

- **`internal`** → Accessible anywhere within the same assembly (project). Commonly used for classes or members that should be shared inside a project but hidden from external projects.

- **`protected internal`** → Accessible if **either** condition is true:
  - Code is in the same assembly, **OR**
  - Code is in a derived class.
  
  Think of it as **OR (union)**, so it provides broader access.

- **`private protected`** *(C# 7.2)* → Accessible only when **both** conditions are true:
  - Code is in a derived class, **AND**
  - The derived class is in the same assembly.
  
  Think of it as **AND (intersection)**, so it provides more restricted access.

```csharp
public class Widget
{
 public int Id; // anywhere
 private int _secret; // this class only
 protected int _forSubclasses; // this class + subclasses
 internal int _sameAssembly; // same assembly
 protected internal int _orEither; // subclass OR same assembly
 private protected int _andBoth; // subclass AND same assembly
}
```

The two combined modifiers trip people up: **`protected internal` = OR (union, broader); `private protected` = AND (intersection, narrower).**

---

### Q2. What are the default access levels if you don't specify one?

**Answer.** Defaults differ by context — a common interview probe:

- **Top-level types** (class, struct, interface, enum, record directly in a namespace) → **`internal`**.
- **Class/struct members** (fields, methods, properties, nested types) → **`private`**.
- **Interface members** → **`public`** (and historically couldn't be anything else; C# 8 allows access modifiers).
- **Enum members** → **`public`** (implicitly, matching the enum's accessibility).
- **Nested types** → **`private`** (like other members), even though top-level types default to `internal`.

```csharp
class Foo // internal by default
{
 int _x; // private by default
 void M() { } // private by default
}
```

The safe habit: **always state accessibility explicitly** rather than relying on defaults, so intent is unambiguous.

---

### Q3. What is `InternalsVisibleTo` and when is it used?

**Answer.** `InternalsVisibleTo` allows another assembly to access **internal** types and members.

Normally:

```csharp
internal class PaymentService
{
    internal void Process()
    {
    }
}
```

Only the same assembly can access it.

Using `InternalsVisibleTo`:

```csharp
[assembly: InternalsVisibleTo("MyApp.Tests")]
```

Now the `MyApp.Tests` assembly can access internal classes and methods.

#### Common usage

Mostly used for **unit testing**.

Example:

- Main project → `MyApp.dll`
- Test project → `MyApp.Tests.dll`

You keep classes/methods internal for normal code, but allow the test project to access them.

---

## B5 — Properties, Indexers, init-only, required members

### Q1. What is a property, and why prefer it over a public field?

**Answer.** A **property** in C# provides controlled access to a class's data using `get` and `set` accessors. It usually wraps a private field and allows validation, access control, and future changes without breaking code.

Example:

```csharp
public class Employee
{
    private decimal salary;

    public decimal Salary
    {
        get => salary;
        set
        {
            if (value >= 0)
                salary = value;
        }
    }
}
```

A public field exposes data directly:

```csharp
public class Employee
{
    public decimal Salary;
}
```

#### Why prefer properties over public fields?

**1. Encapsulation**  
Controls how data is accessed or modified.

```csharp
public int Age
{
    get => age;
    set
    {
        if (value >= 18)
            age = value;
    }
}
```

**2. Validation**  
Prevents invalid values from being stored.

**3. Access control**  
You can make properties read-only or restrict modification.

```csharp
public DateTime CreatedDate { get; private set; }
```

**4. Flexibility**  
You can change internal implementation later without affecting code that uses the property.

**5. Framework support**  
.NET frameworks like Entity Framework and serializers work better with properties.

#### Auto-implemented property

When no extra logic is needed:

```csharp
public string Name { get; set; }
```

The compiler creates the backing field automatically.

**In short:**  
Use properties instead of public fields because they provide encapsulation, validation, access control, and flexibility.
---

### Q2. What is the difference between `get`-only, `set`, `init`, and computed properties?

**Answer.**
#### 1. Get-only property

A **get-only property** can only be read. It cannot be changed after initialization.

```csharp
public class Employee
{
    public string Name { get; }

    public Employee(string name)
    {
        Name = name;
    }
}
```

Usage:

```csharp
var emp = new Employee("John");

Console.WriteLine(emp.Name); // Allowed
emp.Name = "Mike";           // Error
```

Used when a value should remain immutable after creation.

#### 2. Get/Set property

A **get/set property** allows both reading and modifying the value.

```csharp
public class Employee
{
    public string Name { get; set; }
}
```

Usage:

```csharp
var emp = new Employee();

emp.Name = "John";  // Allowed
Console.WriteLine(emp.Name);
```

Used when the value can change during the object's lifetime.

#### 3. Init-only property

An **init property** can be set only during object creation. After initialization, it becomes read-only.

```csharp
public class Employee
{
    public string Name { get; init; }
}
```

Usage:

```csharp
var emp = new Employee
{
    Name = "John"
};

emp.Name = "Mike"; // Error
```

Introduced in C# 9 to support immutable objects while allowing object initializer syntax.

#### 4. Computed property

A **computed property** does not store a value. It calculates and returns a value from other properties.

```csharp
public class Employee
{
    public decimal Salary { get; set; }
    public decimal Tax => Salary * 0.1m;
}
```

Usage:

```csharp
var emp = new Employee
{
    Salary = 5000
};

Console.WriteLine(emp.Tax); // 500
```

The value is calculated every time the property is accessed.

---

### Summary

- **get-only** → Set it in constructor only.
- **get/set** → Can read and modify anytime.
- **init** → Set it when creating the object, then lock it.
- **computed** → Calculates a value instead of storing it.
---

### Q3. What are `required` members (C# 11)?

**Answer.** The **`required`** modifier forces callers to set a property/field during initialization — the compiler **errors** if they don't. It solves "this property must be provided, but I want to keep it non-nullable and avoid a big constructor."

```csharp
public class Customer
{
 public required string Name { get; init; } // must be set by every caller
 public string? Nickname { get; init; } // optional
}

var ok = new Customer { Name = "Ada" }; // fine
// var bad = new Customer { }; // ERROR: 'Name' must be set
```

Why it matters:
- It lets you keep a property **non-nullable** without an `= null!` hack or a constructor parameter, cooperating cleanly with **nullable reference types** (see A5).
- It combines naturally with `init` for immutable-but-mandatory data.
- It expresses intent that a class-based DTO can be built with object-initializer syntax while still guaranteeing key fields are present.

Note `required` is about **compile-time enforcement of being set**, not runtime — and `SetsRequiredMembers` on a constructor can waive it when the constructor guarantees assignment.

---

### Q4. What is an indexer?

**Answer.** An **indexer** in C# allows an object to be accessed like an array using the `[]` syntax. It provides a way to expose a collection or internal data structure through an index.

Example:

```csharp
public class Team
{
    private string[] members = new string[3];

    public string this[int index]
    {
        get => members[index];
        set => members[index] = value;
    }
}
```

Usage:

```csharp
var team = new Team();

team[0] = "John";
team[1] = "Mary";

Console.WriteLine(team[0]); // John
```

#### Why use indexers?

- Provide array-like access to objects.
- Hide the internal data structure.
- Allow validation when reading or writing values.

Example with validation:

```csharp
public string this[int index]
{
    get => members[index];
    set
    {
        if (index >= 0 && index < members.Length)
            members[index] = value;
    }
}
```

#### Indexer vs Property

A **property** accesses a single named value:

```csharp
employee.Name
```

An **indexer** accesses multiple values using an index or key:

```csharp
team[0]
```

In short:  
An **indexer is a special property that allows objects to be accessed using `[]` notation, similar to arrays.**

---

## B6 — Static Members & Static Constructors

### Q1. What is a static member, and what does a static class mean?

**Answer.** A **static member** is a member (field, property, method, etc.) that belongs to the **class itself**, not to an object instance. Only **one copy** of a static member exists, and all objects of that class share it.

Example:

```csharp
public class Employee
{
    public static int Count = 0;

    public Employee()
    {
        Count++;
    }
}
```

```csharp
var e1 = new Employee();
var e2 = new Employee();

Console.WriteLine(Employee.Count); // 2
```

Here, `Count` is shared by all `Employee` objects. It is accessed using the class name:

```csharp
Employee.Count;
```

A **static class** is a class that cannot be instantiated. It can contain only static members.

Example:

```csharp
public static class Calculator
{
    public static int Add(int a, int b)
    {
        return a + b;
    }
}
```

Usage:

```csharp
int result = Calculator.Add(10, 20);
```

You cannot create an object of a static class:

```csharp
var calc = new Calculator(); // Error
```

Static classes are mainly used for utility/helper methods where no object state is required.

**In short:**

- **Static member** → Belongs to the class and is shared by all objects.
- **Static class** → A class that cannot create objects and contains only static members.
---

### Q2. What is a static constructor, and when does it run?

**Answer.** A **static constructor** is a special constructor used to initialize **static members** of a class. It runs **only once** during the application's lifetime.

Example:

```csharp
public class Employee
{
    public static int Count;

    static Employee()
    {
        Count = 100;
        Console.WriteLine("Static constructor called.");
    }
}
```

Usage:

```csharp
Console.WriteLine(Employee.Count);
```

**Output:**

```
Static constructor called.
100
```

#### When does it run?

A static constructor is called automatically by the CLR:

- **Before the first object is created**, or
- **Before the first static member is accessed**,

whichever happens first.

Example 1:

```csharp
var emp = new Employee();
```

The static constructor runs before creating `emp`.

Example 2:

```csharp
Console.WriteLine(Employee.Count);
```

The static constructor runs before accessing `Count`.

#### Why use a static constructor?

Use it to initialize static data that should be set only once, such as:

- Loading configuration.
- Initializing static fields.
- Opening shared resources.
---

### Q3. What is the difference between a `static` field, a `const`, and a `static readonly` field?

**Answer.**
### `const`

- Value is fixed at **compile time**.
- Cannot be changed.
- Automatically `static`.
- Must be initialized when declared.

```csharp
public const int MaxUsers = 100;
```
### `static field`

- One shared value for the entire class.
- Can be changed at runtime.

```csharp
public static int UserCount = 0;

UserCount++;
```
### `static readonly`

- One shared value for the entire class.
- Can only be assigned during declaration or inside a static constructor.
- Value is fixed after initialization.
- Initialized at **runtime**.

```csharp
public static readonly string ConnectionString;

static Config()
{
    ConnectionString = "Server=SQL";
}
```

---

## B7 — Extension Methods

### Q1. What is an extension method, and how does it work?

**Answer.** An **extension method** allows you to add a new method to an existing type without modifying the original class or creating a derived class.

It is a **static method** inside a **static class**, but it can be called like an instance method.

```csharp
public static class StringExtensions
{
    public static bool IsValidEmail(this string value)
    {
        return value.Contains("@");
    }
}

string email = "test@gmail.com";

bool result = email.IsValidEmail();
```

---
