# A. Type System & Fundamentals
---

## A1 — Value vs Reference Types (Stack vs Heap)

### Q1. What is the difference between a value type and a reference type?

**Answer.** A **value type** variable holds the actual data directly. When you assign it to another variable or pass it into a method, C# makes a **copy** — so the two variables are completely separate, and changing one doesn't affect the other. Common value types: `int`, `double`, `bool`, `char`, `decimal`, `DateTime`, any `struct`, and `enum`. They can't be `null` (unless you write them as `int?`, `bool?`, etc.), and if you don't set them they start at a "zero" value (`0`, `false`, and so on).

A **reference type** variable doesn't hold the object itself — it holds a **link** to the object (which lives somewhere else in memory). When you assign it, you copy the *link*, not the object. So two variables can point at the **same** object, and a change made through one is seen through the other. Common reference types: `class`, `string`, `object`, arrays, delegates, interfaces. They can be `null`, and if you don't set them they start as `null`.

```csharp
int a = 5; int b = a; b = 10; // a == 5, b == 10 (independent copies)

class Box { public int Value; }
Box x = new Box { Value = 5 };
Box y = x; // y points to the SAME object
y.Value = 10; // x.Value == 10 too — both see the change
```

**Analogy:** value type = a *photocopy*; reference type = the *address* of a house — everyone holding the address sees the same house.

---

### Q2. What are the stack and the heap, and how do they relate?

**Answer.** The **stack** and **heap** are two different areas of memory used by a .NET application.

The **stack** stores information related to method execution, such as local variables, method parameters, and call frames. Memory on the stack is allocated and released automatically when methods are called and return, making it very fast.

The **heap** stores dynamically allocated objects, such as instances of classes, arrays, and strings. Memory on the heap is managed by the .NET Garbage Collector (GC), which frees objects that are no longer referenced.

#### How They Relate

When you create a reference type:

```csharp
Person person = new Person();
```

- The `Person` object is allocated on the **heap**.
- The variable `person` (a reference) is stored in the current method's **stack frame**.
- The reference points to the object on the heap.

When the method ends, the reference is removed from the stack. If no other references point to the object, it becomes eligible for garbage collection.


### Q3. Is `string` a value type or a reference type?

**Answer.** `string` is a **reference type**. It just *feels* like a value type because it is **immutable** — you can never change a string's contents in place. Any "change" you make actually creates a brand-new string.

```csharp
string s1 = "hello"; string s2 = s1;
s2 = "world"; // creates a NEW string; s1 stays "hello", s2 is "world"
```

Because a string can never be changed in place, you never run into the "two variables share the same object" surprise that other reference types have. Keep in mind: "immutable" and "value type" are two different ideas — a string is immutable but still a reference type. One practical tip: joining strings together in a loop is slow (each join creates a new string), so use `StringBuilder` for that.

---

### Q4. If I pass a reference type to a method and modify it, does the caller see the change? What about reassigning the parameter?

When a reference type is passed to a method, C# passes **a copy of the reference**, not a copy of the object.

If the method **changes the object's data**, the caller sees the change because both references point to the **same object**.

```csharp
void Update(Person person)
{
    person.Name = "John";
}

Person p = new Person { Name = "Alice" };
Update(p);

Console.WriteLine(p.Name); // John
```

If the method **assigns the parameter to a new object**, the caller does **not** see the change. Only the local reference inside the method now points to the new object.

```csharp
void Update(Person person)
{
    person = new Person { Name = "John" };
}

Person p = new Person { Name = "Alice" };
Update(p);

Console.WriteLine(p.Name); // Alice
```

If you want the method to replace the caller's object, use `ref`.

```csharp
void Update(ref Person person)
{
    person = new Person { Name = "John" };
}
```

---

### Q5. What's the difference between `ref`, `out`, and `in` parameters?

**Answer.**
These keywords control how arguments are passed to methods **by reference** rather than by value.

#### `ref` — read and write

Passes a variable **by reference**. The variable **must already be initialized**, and the method can **read and modify** it.


```csharp
void Add(ref int x)
{
    x++;
}

int number = 5;
Add(ref number);

Console.WriteLine(number); // 6
```

Use it when the method needs to both see the incoming value and modify it.

#### `out` — write only (output)

The parameter is also a reference, but the variable does **not** need to be initialized before the call. The method **must assign** a value before it returns. Typically used to return extra values.

```csharp
void GetValue(out int x)
{
    x = 10;
}

int number;
GetValue(out number);

Console.WriteLine(number); // 10
```

The classic example is the `Try...` pattern (`int.TryParse`, `Dictionary.TryGetValue`). You can declare the variable inline at the call site, as shown above.

#### `in` — read only (input)

Passes a variable **by read-only reference**. The variable **must be initialized**, but the method **cannot modify** it.

```csharp
void Print(in int x)
{
    Console.WriteLine(x);
}

int number = 5;
Print(in number);
```

Use `in` mainly when you're passing large structs and want to avoid the cost of copying them. For small types like `int` it makes no real difference, so don't bother.

---

### Q6. When would you use a `struct` (value type) instead of a `class`?

**Answer.** Choose a `struct` when the data is **small, simple, and represents a single value** — where two instances holding the same data can be treated as equal and independent. Good examples: a `Point`, a `Money` amount, coordinates, a small key.

The main benefit is speed: small value types avoid extra memory-management work, which means less clean-up for the garbage collector to do. Rule of thumb: keep structs small (a handful of fields), and ideally make them unchangeable (`readonly struct`).

Choose a `class` when the type is large, represents something with its own identity, needs inheritance, or gets passed around a lot (copying a big struct over and over is wasteful).

---

### Q7. "Value types always live on the stack." Is that true?

No — this is one of the most common myths in C#. **Value types often live on the stack, but "always" is wrong.** Where a value type is stored depends on **where you declare it**, not on the fact that it's a value type.

#### When Value Types Don't Live on the Stack

##### 1. Fields of a Class Live on the Heap

If a struct or other value type is a field of a class, it is stored **inside the class object**, which lives on the heap.

```csharp
class Container
{
    public int Count;      // Stored on the heap
    public Point Origin;   // This struct is also stored on the heap
}

Container container = new Container();
```

`Count` and `Origin` are part of the `Container` object, so they are stored on the heap.


##### 2. Boxed Value Types Live on the Heap

When a value type is assigned to an `object` or an interface, **boxing** occurs. Boxing creates a copy of the value inside a heap-allocated object.

```csharp
int x = 5;
object o = x;   // Boxing
```

Memory layout:

```
Stack                  Heap
-----                  -----------------
x = 5        ---->     Boxed object
                         Value = 5
```

`x` remains unchanged on the stack, while a copy is created on the heap.


##### 3. Array Elements Live on the Heap

Arrays are reference types, so the array itself is allocated on the heap. Therefore, all value-type elements are stored inside that heap-allocated array.

```csharp
int[] numbers = new int[1000];
```

Memory layout:

```
Heap
-------------------------------
numbers
+----+----+----+----+----+
| 0  | 0  | 0  |... | 0  |
+----+----+----+----+----+
```

Although `int` is a value type, the 1000 integers are stored on the heap because the array is on the heap.


##### 4. Variables Captured by a Lambda Move to the Heap

If a local variable gets used inside a lambda (or an `async` method or a `yield` iterator), C# has to keep that variable alive even after the method moves on. To do that, it quietly moves the variable into an object on the heap.

```csharp
int total = 0;

Action add = () => total++;

add();
Console.WriteLine(total); // 1
```

Normally `total` would just be a stack variable. But because the lambda still needs it later, C# stores it on the heap instead.


##### What Is Actually True?

A more accurate statement is:

> **Value types are stored inline wherever they are declared.**

For example:

- A **local variable** usually lives on the stack.
- A **field of a class** lives inside the class object, which is on the heap.
- An **array element** lives inside the array, which is on the heap.
- A **boxed value** lives in a heap object.
- A **variable captured by a lambda** gets moved to the heap.

So the **stack is just one of several places** a value type can live — it is **not** always where value types go.

---

### Q8. What is the default value of a value type vs a reference type? And what is `int?`?

**Answer.** `default(T)` for a **value type** gives a zeroed instance — `0`, `false`, `'\0'`, and for a struct an instance with **all fields zeroed** (never `null`). For a **reference type**, `default` is `null`.

```csharp
int x = default; // 0
Point p = default; // struct, all fields zeroed (NOT null)
string s = default; // null
int? n = default; // null
```

`int?` is just a shorthand for `Nullable<int>`, which is **itself a `struct`** that holds two things: the value, plus a flag saying whether a value is actually there. So a nullable int is **still a value type** — it simply has an extra "no value" state on top of the normal ones.

---

## A2 — Boxing & Unboxing

### Q1. What is boxing and unboxing?

**Answer.** **Boxing** is when C# takes a **value type** (like an `int`) and wraps it inside an object so it can be treated like a reference type. **Unboxing** is pulling that value back out again. Boxing happens automatically when you assign a value type to an `object`, `dynamic`, or an interface. Unboxing needs an explicit cast back to the exact original type.

```csharp
int i = 42;          // a plain int holding 42
object o = i;        // BOXING: C# creates a new object to hold a copy of 42, and 'o' points to it.
                     // The original 'i' is untouched.
int j = (int)o;      // UNBOXING: copies the 42 back out of that object into a new int 'j'.
```

Because boxing creates a new object each time, it costs a bit of CPU time and gives the garbage collector more to clean up later.

---

### Q2. Why does boxing matter for performance?

**Answer.** Every time you box a value, C# creates a new object. If this happens in a loop or other code that runs a lot, it adds up:
1. Creating each object takes time.
2. All those objects give the garbage collector more work, which can cause more frequent pauses.
3. There's extra copying — the value is copied in when boxed and out when unboxed.

```csharp
object sum = 0;
for (int i = 0; i < 1_000_000; i++)
 sum = (int)sum + i; // unbox, add, box — a million objects created!
```

Avoiding this kind of waste is a big reason generics exist (see Q5).

---

### Q3. Where does boxing happen accidentally?

**Answer.** Boxing often happens quietly, without you writing a cast:

- **Assigning a value type to `object`:** `object o = 5;`
- **Assigning it to an interface:** `IComparable c = 5;`
- **Old-style collections that hold `object`:** `ArrayList list = new(); list.Add(5);` — every `int` gets boxed.
- **String formatting:** `string.Format("{0}", 5)`, `Console.WriteLine("{0}", 5)`.
- **Putting an `enum` into an `object`.**

The fix in almost every case: use **generics** (like `List<int>`) so the value is stored directly, with no boxing.

---

### Q4. Does calling `ToString()` on an `int` cause boxing?

**Answer.** **No.** The `int` type provides its own `ToString()`, so C# can call it directly on the value **without boxing**. Boxing only kicks in when you actually store the value in an `object` or an interface.

```csharp
int i = 42;
string s = i.ToString(); // NO boxing — int has its own ToString()
object o = i; // boxing happens here
```

Calling a method through an **interface variable** boxes; calling it directly on the value usually does not.

---

### Q5. How do generics help avoid boxing?

**Answer.** A generic type like `List<int>` knows it's holding `int`s specifically, so it stores them directly instead of wrapping each one in an object. No boxing needed.

```csharp
ArrayList a = new ArrayList(); a.Add(5); // boxing — ArrayList holds 'object'
List<int> b = new List<int>(); b.Add(5); // no boxing — List<int> holds int directly
```

This was one of the main reasons generics were added in .NET 2.0: `List<T>` and `Dictionary<TKey,TValue>` removed the boxing overhead that the older `ArrayList` and `Hashtable` had.

---

### Q6. Does using a value type through an interface always box?

**Answer.** It depends on **how** you use it:

- **Storing it in an interface variable** → **boxes**: `IComparable c = 5;`
- **Using it through a generic constraint** → **does not box**:
 ```csharp
 T Max<T>(T a, T b) where T : IComparable<T>
 => a.CompareTo(b) >= 0 ? a : b; // no boxing when T is a value type
 ```

This is why writing `where T : IComparable<T>` is better than a method that takes a plain `IComparable` — you get the same capability without the boxing.

---

## A3 — Struct vs Class

### Q1. What is the core difference between a `struct` and a `class`?

**Answer.** A `struct` is a **value type** and a `class` is a **reference type** — and that one difference explains everything else. A `struct` holds its data **directly** and gets **copied** whenever you assign it, pass it, or return it, so two struct variables are always separate. A `class` variable holds a **link** to an object; assigning it copies only the link, so two variables can point at — and change — the same object.

```csharp
struct PointS { public int X, Y; }
class PointC { public int X, Y; }

var s1 = new PointS { X = 1 }; var s2 = s1; s2.X = 99; // s1.X still 1 (full copy)
var c1 = new PointC { X = 1 }; var c2 = c1; c2.X = 99; // c1.X is now 99 (shared object)
```

A `struct` also can't be `null` (unless you write it as `T?`), it starts out with all its fields zeroed, and it can't use inheritance — though it can still implement interfaces.

---

### Q2. What are the practical limitations of a `struct`?

**Answer.**

- **No user-defined inheritance.** A struct can't derive from anything and nothing derives from it (implicitly `sealed`); it *can* implement interfaces.
- **All fields must be definitely assigned** by the end of every constructor.
- **Cannot be `null`** unless `Nullable<T>` (`T?`).
- **A struct field cannot be of its own type** (infinitely sized), whereas a class can reference its own type (e.g. a linked-list node).

```csharp
struct Node { public Node Next; } // ERROR: struct cannot contain itself
class CNode { public CNode Next; } // fine — it's a reference
```

Lack of inheritance is often the deciding factor: if you need polymorphism through a base type, use a `class`.

---

### Q3. What are value semantics vs reference semantics, and how do they affect equality?

**Answer.** **Value semantics** means two things count as "equal" when their contents match. **Reference semantics** means they only count as equal when they're literally the *same object*.

By default: a `struct` compares itself **field by field** when you call `Equals`; but `==` doesn't work on a plain struct unless you write it yourself. A `class`, by default, treats two variables as equal only when they point at the same object — for both `==` and `Equals`.

```csharp
var a = new PointS { X = 1, Y = 2 };
var b = new PointS { X = 1, Y = 2 };
a.Equals(b); // True — field-by-field
// a == b; // won't compile — no operator== on a plain struct

var c = new PointC { X = 1, Y = 2 };
var d = new PointC { X = 1, Y = 2 };
c.Equals(d); // False — reference equality
```

Worth knowing: the built-in field-by-field comparison for structs is **slow** under the hood. So if you use a struct as a dictionary key or compare it a lot, it's worth writing your own `Equals` and `GetHashCode` (and `==`/`!=`) — or just use a `record struct` (see A4/A9), which does all of that for you.

---

### Q4. When should you choose a `struct` over a `class`?

**Answer.** Choose a `struct` only when **all** of these are true:
1. It represents a **single value** (like `Point`, `Money`, `Color`, or a timestamp).
2. It is **small** (just a few fields — if it gets big, all the copying costs more than it saves).
3. It **doesn't change** after it's created (prefer `readonly struct`).
4. It **won't get boxed a lot** (boxing cancels out the benefit).

The upside is speed: less memory-management overhead, and arrays of structs are laid out efficiently. Choose a `class` when the type is large, has its own **identity**, needs **inheritance**, is **changed and shared** across your code, or gets passed around a lot. **When in doubt, use a `class`** — a struct is a deliberate optimization you reach for on purpose.

---

## A4 — Records

### Q1. What is a `record` and why was it added?

**Answer.** A **record** is a special reference type introduced in **C# 9** for storing **immutable data**. It compares objects by **value** instead of by reference.

```csharp
public record Person(string FirstName, string LastName);
var a = new Person("Ada", "Lovelace");
var b = new Person("Ada", "Lovelace");
a == b; // True — compared by contents, not by identity
a; // Person { FirstName = Ada, LastName = Lovelace }
```

Records were added to make it easier to create **immutable data models** without writing lots of boilerplate code.

Benefits:
- Less code
- Built-in value equality
- Supports immutability
- Useful for DTOs, API request/response models, and configuration objects

- **Class** → Reference equality by default.
- **Record** → Value equality by default.
- Use **record** when you want immutable objects that represent data rather than behavior.

---

### Q2. How does value equality on a record work?

**Answer.** A **record** uses **value equality**, which means two records are equal if:

- They are the **same record type**.
- **All corresponding properties have the same values**.

```csharp
public record Person(string Name, int Age);

var p1 = new Person("John", 30);
var p2 = new Person("John", 30);

Console.WriteLine(p1 == p2); // True
```
C# compares:

```text
Name: "John" == "John"   ✔
Age : 30 == 30           ✔
----------------------------
Result: True
```
#### If one property is different

```csharp
var p1 = new Person("John", 30);
var p2 = new Person("John", 31);

Console.WriteLine(p1 == p2); // False
```

Comparison:

```text
Name: "John" == "John"   ✔
Age : 30 == 31           ✘
----------------------------
Result: False
```
#### Different record types

```csharp
public record Person(string Name, int Age);
public record Employee(string Name, int Age, decimal Salary);

var p = new Person("John", 30);
var e = new Employee("John", 30, 5000);

// p == e // Compile-time error
```
Even though some values are the same, they are **different record types**, so C# does not compare them.

---

### Q3. What is a `with` expression?

**Answer.** A **`with` expression** makes a **copy** of a record with a few properties changed, and leaves the original alone.

```csharp
var p1 = new Person("Ada", "Lovelace");
var p2 = p1 with { LastName = "Byron" }; // p1 stays the same; p2 is a new copy
```

Two things worth knowing. First, the copy is **shallow**: if the record contains something like a `List<T>`, both the original and the copy point at the *same* list — the list itself isn't duplicated. Second, `with` works on `record struct` too, not just `record class`.

---

### Q8. Records feel like value types — are they?

**Answer.** No — a plain `record` is a **reference type**. Assigning it copies the **link**, not the object, and it can be **`null`**. It only *acts* value-like in two specific ways: comparing by contents, and the `with` copy. This is similar to how `string` is a reference type that feels value-like because it's immutable.

```csharp
var a = new Person("Ada", "Lovelace");
var b = a; // copies the LINK — both point at the same object
Person? c = null; // records can be null
```

A simple way to remember it: a `record` is a reference type with **content-based equality added on top**; a `record struct` is a value type with the same equality added on. "Compares by content" and "is a value type" are two separate things — a record only gives you the first one.

---

## A5 — Nullability

### Q1. What does `int?` actually compile to?

**Answer.** Normally a value type **can't be `null`**. `int?` is just shorthand for **`Nullable<int>`**, which is **itself a `struct`** — so an `int?` is *still a value type*. It simply wraps the number with one extra flag that says whether a value is present:

```csharp
public struct Nullable<T> where T : struct
{
 private readonly bool hasValue; // is there a value?
 private readonly T value;       // the actual value
 public bool HasValue => hasValue;
 public T Value => hasValue ? value : throw new InvalidOperationException();
}
```

When you don't set it, an `int?` starts out as "no value" (i.e. `null`), *not* `0`. And because this only works for value types, you can write `int?` or `DateTime?` — but reference types like `string?` work in a totally different way (see Q2).

---

### Q2. What's the difference between nullable *value* types and nullable *reference* types?

**Answer.** They look alike (`int?` vs `string?`) but work in **completely different ways**:

| | `int?` (nullable value type) | `string?` (nullable reference type) |
|---|---|---|
| Since | C# 2.0 | C# 8.0 |
| What it is | A real type — the `Nullable<T>` struct | **Just a hint to the compiler** |
| Effect while running | Yes — carries the extra "has value?" flag | **None** — it's the same old `string` at runtime |
| How it's enforced | By the type system | Only through compiler **warnings** |

`int?` is a genuinely different type with real behavior. `string?` doesn't change the type at all — it's just a note that helps the compiler warn you about possible `null` mistakes, and it disappears once the program runs. Nothing actually stops a `string` from being `null` while the program runs; it's purely a **help-you-catch-bugs-early** feature. That's the main thing to remember: `int?` is a real type, `string?` is just a warning aid.

---

### Q3. How do you read a nullable value type safely?


#### Safe ways to read a nullable value type

Suppose:

```csharp
int? age = GetAge();
```

##### 1. Check `HasValue` (Most Explicit)

```csharp
if (age.HasValue)
{
    Console.WriteLine(age.Value);
}
else
{
    Console.WriteLine("No value");
}
```

- `HasValue` tells you whether the nullable contains a value.
- Only access `.Value` after checking `HasValue`; otherwise, an `InvalidOperationException` is thrown.


##### 2. Use the Null-Coalescing Operator (`??`) (Most Common)

```csharp
int value = age ?? 0;
```

If `age` is `null`, `value` becomes `0`.

Example:

```csharp
int? age = null;

Console.WriteLine(age ?? -1); // -1
```

##### 3. Use `GetValueOrDefault()`

```csharp
int value = age.GetValueOrDefault();
```

Returns:

- The actual value if present.
- The default value of the underlying type (`0` for `int`, `false` for `bool`, etc.) if `null`.

You can also specify a custom default:

```csharp
int value = age.GetValueOrDefault(18);
```

##### 4. Pattern Matching (Modern C#)

```csharp
if (age is int actualAge)
{
    Console.WriteLine(actualAge);
}
```

This checks for a value and extracts it in one step.

#### Unsafe Approach

```csharp
int? age = null;

Console.WriteLine(age.Value); // Throws InvalidOperationException
```

`.Value` should only be used when you're certain the nullable contains a value.


---

### Q4. What are nullable reference types (NRT) and how do you enable them?

**Answer.** Nullable reference types (NRT) is an **opt-in feature** (C# 8+) that lets you say whether a reference is *allowed* to be null, so the compiler can warn you about likely null-related bugs before you even run the code. Once turned on: `string` means **"should never be null"** (you get a warning if you try to assign `null`); `string?` means **"might be null"** (you get a warning if you use it without checking first).

Turn it on with `#nullable enable` in a single file, or for the whole project (it's on by default in new projects):

```xml
<PropertyGroup>
 <Nullable>enable</Nullable>
</PropertyGroup>
```

```csharp
string name = null; // CS8600
string? maybe = null; // fine
int len = maybe.Length; // CS8602: possible dereference of null
```

Recommendation for new projects: turn it on for the whole project and treat these warnings as real bugs to fix.

---

### Q5. What do `?` and `!` mean under NRT, and what does `!` do at runtime?

**Answer.**

- **`?`** after a type (`string?`) means **"this might be null"** — it turns on null-checking for that value.
- **`!`** is the **null-forgiving operator**. Writing `x!` tells the compiler "trust me, this isn't null here" and **turns off the warning**. Important: it does **nothing when the program runs** — it doesn't check anything or throw anything. It only silences the compiler.

```csharp
string s2 = Find(1)!; // no warning — but if it really IS null, you'll crash later, not here
```

Because `!` just silences the compiler without actually making anything safer, using it too much defeats the whole point of NRT. It's fine in a few cases: test setup, working with older libraries that don't support NRT, or right after a check the compiler can't follow. Otherwise, prefer a real null check so the safety is actually there — not just claimed.

---

## A6 — Enums

### Q1. What is an `enum`, and what is its underlying type?

**Answer.** 

-  An **enum** (enumeration) is a value type that represents a fixed set of named constants, making code more readable and maintainable.
-  Behind the scenes each name is just a number. That number is an `int` by default, but you can pick a smaller type like `byte` if you want to save space.

```csharp
enum Status { Inactive, Active, Suspended } // 0, 1, 2 — numbered automatically
enum SmallFlag : byte { Off = 0, On = 1 } // stored as a byte
```

If you don't assign numbers, they start at **0** and go up by 1. One important gotcha: C# doesn't check that a number is a valid enum member. So `(Status)999` will happily give you a `Status` that isn't any of the three names. If a value comes from somewhere you don't trust (user input, a file), check it with `Enum.IsDefined` first. To turn text into an enum, use `Enum.TryParse`.

---

### Q2. What does `[Flags]` do, and how do you use it?

**Answer.** 
`[Flags]` allows an `enum` to store **multiple values** in a single variable using **bitwise operations**. It's commonly used for **permissions**.

```csharp
[Flags]
public enum Permission
{
    None = 0,
    Read = 1,
    Write = 2,
    Delete = 4
}
```
Assign multiple permissions:

```csharp
Permission permission = Permission.Read | Permission.Write;
```

Check if a permission exists:

```csharp
bool canWrite = permission.HasFlag(Permission.Write);

Console.WriteLine(canWrite); // True
```

#### Why use `1`, `2`, `4`?

Each value is a **power of 2**, so each permission uses a unique bit.

```text
Read   = 001
Write  = 010
Delete = 100
```

This lets multiple values be combined without overlapping.

---

### Q3. What is the default value of an enum, and what's the pitfall?

**Answer.** An enum's default is the number **`0`** — not "whichever member you listed first." So an enum you never set ends up as `0`, which maps to whatever member happens to equal 0, or to a nameless value if none does.

```csharp
enum Priority { Low = 1, Medium = 2, High = 3 }
Priority p = default; // 0 — but NO member is 0! So p is a value with no name.
```

The rule to follow: **always make `0` a real, safe member** (like `None`, `Unknown`, or `Unspecified`), so an unset enum still means something valid:

```csharp
enum Priority { Unspecified = 0, Low = 1, Medium = 2, High = 3 }
```

A couple more practical tips: if you save enums to a database or file, give each member an explicit number and never reuse an old number for something else. And when you save them as JSON, they're written as numbers by default — add `JsonStringEnumConverter` if you'd rather see the names.

---

## A7 — const vs readonly vs static readonly

### Q1. What is the difference between `const` and `readonly`?

#### `const`

-  A **compile-time constant** with a value that **must be set at declaration** using a literal or constant expression. 
-  It is **implicitly `static`**, cannot be changed, and is limited to simple types (`int`, `double`, `string`, etc.). Since it is evaluated at compile time, it is efficient but less flexible.
-  **implicitly `static`**  means you don't need to create an object to access a const field. The compiler automatically treats it as static.
```csharp
const double Pi = 3.14159; // Fixed at compile time

Pi = 3.14; // ❌ Error: cannot modify
```

**Use for:** Unchanging values such as mathematical constants or fixed application settings.


#### `readonly`

-  A field that can be **initialized at declaration or inside a constructor** (instance or static), but cannot be changed afterward. 
-  It works with **any type**, including objects, and can be instance-specific or static. Since it is initialized at runtime, it provides more flexibility than `const`.

```csharp
class Person
{
    readonly int Id;

    public Person(int id)
    {
        Id = id; // Set in constructor
    }

    // Id = 10; // ❌ Error: cannot modify outside the constructor
}
```

**Use for:** Values that are fixed after object creation, such as IDs or configuration values loaded at runtime.

---

### Q3. What types can a `const` be?

**Answer.** Only simple types whose value the compiler can write directly into the code: numbers, `bool`, `char`, `string`, an `enum`, or a reference type that's set to `null`.

```csharp
const int Max = 100;
const string Name = "abc";
const object Nothing = null; // the only reference-type const value allowed
// const DateTime Epoch = new DateTime(1970, 1, 1); // ERROR — needs a runtime ctor
// const int[] Nums = { 1, 2, 3 }; // ERROR — arrays can't be const
```

Objects created with `new`, arrays, `DateTime`, `TimeSpan` require a runtime constructor → use `static readonly` for those.

---

### Q4. What is the cross-assembly versioning problem with `const`?

**Answer.** A `const` value is **embedded into the calling assembly at compile time**.

If a library changes the `const` value later, applications using that library **must be recompiled** to get the updated value.

**Library.dll**

```csharp
public class Config
{
    public const int MaxUsers = 100;
}
```

**App.exe**

```csharp
Console.WriteLine(Config.MaxUsers); // 100
```

Later, the library changes:

```csharp
public const int MaxUsers = 200;
```

If only **Library.dll** is updated and **App.exe** is **not recompiled**, the app still prints:

```text
100
```

because the value `100` was embedded into `App.exe` when it was compiled.

#### Solution

Use `static readonly` instead.

```csharp
public static readonly int MaxUsers = 200;
```

`static readonly` is read **at runtime**, so updating the library is enough—the application does not need to be recompiled.

---

## A8 — Tuples & Deconstruction

### Q1. What is a tuple, and what's the difference between `System.Tuple` and `System.ValueTuple`?

**Answer.** A **tuple** is a lightweight way to group multiple values into a single object **without creating a custom class or struct**.

Instead of creating a class:

```csharp
public class Person
{
    public int Id { get; set; }
    public string Name { get; set; }
}
```

You can simply return multiple values:

```csharp
(int Id, string Name) person = (1, "John");

Console.WriteLine(person.Id);   // 1
Console.WriteLine(person.Name); // John
```

#### `System.Tuple` (Older)

- Introduced in **.NET Framework 4.0**
- **Reference type** (`class`)
- Values are accessed using `Item1`, `Item2`, etc.
- Less readable
- Stored on the **heap**

```csharp
Tuple<int, string> person = new Tuple<int, string>(1, "John");

Console.WriteLine(person.Item1);
Console.WriteLine(person.Item2);
```

#### `System.ValueTuple` (Modern & Recommended)

- Introduced in **C# 7**
- **Value type** (`struct`)
- Supports **named fields**
- More readable
- Better performance in many scenarios

```csharp
(int Id, string Name) person = (1, "John");

Console.WriteLine(person.Id);
Console.WriteLine(person.Name);
```

### Q2. How do you deconstruct a tuple (and any type)?

**Answer.** **Deconstruction** splits a tuple into separate variables in one statement. Use a **discard `_`** for values you ignore.

```csharp
var (count, average) = Analyze(data);
var (_, average2) = Analyze(data); // ignore the count
foreach (var (key, value) in dict) { } // idiomatic
```

You can make *any* type deconstructible by giving it a **`Deconstruct`** method that hands back its parts via `out` parameters:

```csharp
class Point
{
 public int X { get; } public int Y { get; }
 public void Deconstruct(out int x, out int y) => (x, y) = (X, Y);
}
var (x, y) = new Point(3, 4); // calls Deconstruct
```

Records get a `Deconstruct` method for free, and deconstruction also works nicely in pattern matching (`case Point(0, 0):`).

---

## A9 — Equality & Identity

### Q1. What's the difference between reference equality and value equality?

**Answer.** **Reference equality** asks *"are these two variables pointing at the exact same object?"* **Value equality** asks *"do these two have the same contents?"* Two different objects can have equal contents while still being separate objects.

By default, classes and structs behave oppositely. A `class` compares by **identity** — equal only if it's the same object. A `struct` compares **field by field** — equal if all the fields match (though this is a bit slow). `string` and `record` compare by contents too, even though they're reference types, because they were specifically built to.

---

### Q2. What's the difference between `==`, `object.Equals`?

**Answer.** Both compare two things, but they are chosen at different moments, and that is the whole difference.

**`==` is an operator.** The compiler picks which `==` to use while compiling, by looking at how the variables are **declared**.

**`Equals` is a method.** The method is picked while the program runs, by looking at what the object **actually is**.

Most of the time they agree, so the difference never shows. It shows when the declared type and the real type are different:

```csharp
object x = "hello";
object y = new string("hello".ToCharArray()); // separate object, same letters

x == y;      // False
x.Equals(y); // True
```

Same two variables, two different answers. Here is why:

- For `==`, the compiler only knows `x` and `y` are declared as `object`. `object` has no special `==`, so it compares **identity** — are these the same object in memory? They are not, so `False`.
- For `Equals`, the runtime looks at what `x` really is, finds a `string`, and calls **string's** `Equals`, which compares the letters. Same letters, so `True`.

Declare them as `string` instead and `==` returns `True`, because now the compiler can see they are strings and picks string's `==`.

**Why does C# have both?**

`Equals` came first — it is a virtual method on `object`, so every type has it and any type can override it. That is what collections use: `Dictionary` and `HashSet` call `Equals` internally, and they must, because they store your items as `object`-like references and cannot know the declared type at your call site.

`==` exists because `a == b` reads better than `a.Equals(b)`, and because it has one thing `Equals` cannot do: **it works on `null`.** Calling `null.Equals(x)` throws; `null == x` is simply `False`.

```csharp
string? s = null;
s == "hi";        // False — fine
s.Equals("hi");   // NullReferenceException
```

`==` is also faster for value types, since there is no virtual call and no boxing.

**What each one does by default:**

| | `==` | `Equals` |
|---|---|---|
| Chosen by | declared type (compile time) | real type (runtime) |
| Plain class | identity | identity |
| `struct` | does not compile unless defined | compares fields |
| `string`, `record` | compares contents | compares contents |
| Works on `null` | yes | no — throws |

**Which should you use?**

- Comparing **strings, numbers, or records** in normal code → `==`. It is clearer and does what you expect.
- Writing **generic or library code** where the type could be anything → `Equals`, because you cannot rely on the declared type being the real one.
- Need to know if two variables are literally the **same object**, ignoring any overrides → `ReferenceEquals(a, b)`. It always checks identity and no type can change that.

The trap to remember: **`==` follows the declared type.** If a variable is declared as `object`, `==` quietly falls back to identity even when the real object knows how to compare contents.

---

### Q3. How does `==` behave for `string`, and why does it surprise people?

**Answer.** For two variables declared as `string`, `==` compares the **letters**, so `"abc" == "abc"` is `true`. The surprise comes from `string.Equals` having something `==` does not: an option for **how** to compare.

```csharp
string a = "hello", b = "hello";
a == b;                   // True — compares the letters
```

**`==` is always case-sensitive, with no way to change it.** `string.Equals` takes a `StringComparison`:

```csharp
"HELLO" == "hello";                                                   // False
string.Equals("HELLO", "hello", StringComparison.OrdinalIgnoreCase);  // True ✅
```

That overload is the main reason to reach for `string.Equals` — comparing user input, file paths, or header names where case should not matter. The options are covered in [csharp-strings-dates.md](csharp-strings-dates.md).

**The second surprise is null.** The static method is null-safe; the instance method is not:

```csharp
string s = null;

s == "x";                  // False — safe
string.Equals(s, "x");     // False — safe (static)
s.Equals("x");             // ❌ NullReferenceException (instance)
```

**And the trap from Q2 applies here too:** if the variables are declared as `object` rather than `string`, `==` compares references instead of letters. With string literals it often still returns `True` — because C# reuses identical literals — which hides the bug until a string is built at runtime.

**Rule of thumb:** `==` for two `string` variables when case matters. `string.Equals(a, b, StringComparison.Ordinal)` when the variables might not be typed as `string`, or when you need case-insensitivity.

---

### Q4. What is the contract between `Equals` and `GetHashCode`?

**Answer.** The one rule to remember: **if two objects are equal, they must produce the same hash code.** In more detail:

1. If `a.Equals(b)` is true, then `a.GetHashCode()` and `b.GetHashCode()` must match.
2. The reverse isn't required — two *unequal* objects are allowed to share a hash code (that's normal and fine).
3. An object's hash code must stay the same while it's being used as a dictionary key (see Q8 for the gotcha here).
4. `Equals` should behave sensibly: an object equals itself, order doesn't matter, and nothing equals `null`.

Why this matters: a `Dictionary` or `HashSet` first uses the hash code to figure out roughly *where* to look, then uses `Equals` to confirm. If two equal keys had different hash codes, they'd get filed in different spots and the collection would never realize they're the same. So **whenever you write your own `Equals`, always write `GetHashCode` too** (the compiler even warns you if you forget).

```csharp
public override bool Equals(object? obj) => obj is Point p && p.X == X && p.Y == Y;
public override int GetHashCode() => HashCode.Combine(X, Y); // same fields
```

---

## A10 — Object Methods, Cloning & Comparison

### Q1. What methods does every type inherit from `System.Object`?

**Answer.** Every type in C# is built on top of `System.Object`, so every type gets these:

- **`ToString()`** — a text version of the object (by default, just the type name). You can override it.
- **`Equals(object)`** — checks equality. You can override it.
- **`GetHashCode()`** — a number used by dictionaries and sets; must agree with `Equals`. You can override it.
- **`GetType()`** — tells you the object's real type. You *can't* override this — it always tells the truth.
- **`MemberwiseClone()`** — makes a **shallow** copy (available only from inside the class).
- **`Finalize()`** — the finalizer, written as `~ClassName()` (rarely used — see Q6).

Plus two shared helpers: **`ReferenceEquals(a, b)`** (always checks identity) and a null-safe **`Equals(a, b)`**.

---

### Q2. How and why do you override `ToString()` well?

**Answer.** By default `ToString()` just returns the type name, which is useless in logs and the debugger. Override it to return a **short, readable, clear** summary — it's what shows up in `Console.WriteLine`, string interpolation, logs, and when you hover over a variable while debugging.

```csharp
public override string ToString() => $"Order #{Id} (${Total:0.00})";
```

A few tips: keep it **fast and harmless** (the debugger calls it a lot), never throw an error from it, and don't try to print huge lists. Records write a good `ToString()` for you automatically, printing something like `TypeName { Prop = ... }`.

---

### Q3. What's the difference between `GetType()`, `typeof`, and the `is` operator?

**Answer.** All three are about types, but they work differently:

- **`obj.GetType()`** — asks an actual object "what type are you *really*?" You need an object to call it.
- **`typeof(SomeType)`** — gets the type info for a type you name directly in code. No object needed.
- **`obj is SomeType`** — asks "can I treat this object as `SomeType`?" and returns true/false. It also returns true for subtypes and interfaces, whereas `GetType() == typeof(...)` only matches the *exact* type.

```csharp
object o = "hello";
o.GetType(); // System.String — its real type
o is string; // True
o is object; // True — a string is also an object
o.GetType() == typeof(object); // False — its exact type is String, not object
```

Use `GetType() == typeof(Cat)` when you mean "exactly a Cat"; use `is Animal` when you mean "can I use this as an Animal (including subclasses)?"

---

## Cheat sheet (last-minute — whole section)

**Value vs reference**

- **Value type** = holds data, copied on assign, can't be null (unless `T?`), default = zeroed. (`int`, `struct`, `enum`, `DateTime`, `decimal`)
- **Reference type** = holds a pointer, copies the pointer, can be null, default = null. (`class`, `string`, arrays, delegates, interfaces)
- `string` = reference type **but immutable** → behaves value-like. (immutable ≠ value type)
- Storage: value types stored **inline where declared** — *not* "always stack." A value field of a class lives on the heap.
- Everything passed **by value** by default; the *reference* is passed by value → modify object = visible, reassign param = not.
- `ref` = by-ref (init first) · `out` = by-ref output (assign inside) · `in` = read-only by-ref (avoid big-struct copy).
- `int?` = `Nullable<int>` = still a struct.

**Boxing**

- **Boxing** = value type → heap object (implicit); **unboxing** = explicit cast back. Heap alloc → GC pressure; deadly in hot loops.
- Sneaky: `object` assignment, **interface-typed** variable, **non-generic** collections (`ArrayList`, `Hashtable`), `params object[]`, string formatting.
- `int.ToString()` does **not** box. **Generics** avoid boxing (the whole reason for `List<T>`). Interface **variable** boxes; interface **constraint** does not.

**Struct vs class**

- `struct` = value type, **no inheritance** (sealed), can implement interfaces, can't contain a field of its own type.
- Choose a struct only when: one value **and** small (≤ ~16–24 B) **and** immutable **and** rarely boxed. Else default to class.
- Mutable structs → mutate-a-copy bugs (`list[0].X =` won't compile; array element mutation *does* work; `foreach` var is a copy). Prefer `readonly struct`.
- `readonly struct` = all fields readonly → prevents mutation bugs **and** lets compiler skip **defensive copies** (`in` on a plain struct can be *slower*).
- `ref struct` = stack-only (`Span<T>`): no boxing, no class field, no capture, no crossing `await`/`yield`.

**Records**

- `record` = `record class` = **reference type** with generated **value equality**, `ToString`, `with`, `Deconstruct`, `IEquatable<T>`.
- Value equality = same runtime type (`EqualityContract`) + all fields equal. `with` = shallow non-destructive copy.
- Positional record → primary ctor + `init` props + `Deconstruct`. `init` = settable only at construction; immutability is **shallow**.
- `record struct` = value type with record features; **mutable by default** → prefer **`readonly record struct`** (beats plain struct: typed `IEquatable<T>`, no reflection, no boxing).
- record for **data + value equality** (DTOs, value objects); class for **identity + behavior** (services, **EF entities**).

**Nullability**

- `int?` = `Nullable<int>` = struct (runtime state, default `null` not `0`). `.Value` **throws `InvalidOperationException`** if empty → use `??` or `GetValueOrDefault`.
- **Value-type nullability ≠ reference-type nullability.** `int?` is a real type; `string?` is a **compile-time annotation only** (same type at runtime).
- NRT (`<Nullable>enable</Nullable>`): `string` = non-null, `string?` = nullable → compiler **warnings**, no runtime enforcement.
- `!` = null-forgiving; **suppresses the warning only**, zero IL. Still use **`ArgumentNullException.ThrowIfNull`** on trust boundaries. Use `required` to keep props non-nullable without `= null!`.

**Enums**

- Enum = named integral constants, a **value type**, default underlying `int`. Members from **0**, +1; blanks count from last explicit.
- Casting an arbitrary int is **not validated** (`(Status)999` succeeds) → validate with `Enum.IsDefined`. `Enum.TryParse<T>` for strings.
- **Default value = 0**, not the first member → always make `0` a safe member (`None`/`Unknown`).
- `[Flags]`: powers of two (`1 << n`), explicit `None = 0`; test `(x & F) != 0`, add `|=`, remove `&= ~F`. `HasFlag` boxes; `HasFlag(None)` always true; `IsDefined` wrong for flags.
- Enums box like any value type; persisted enums → pin explicit values; `System.Text.Json` writes numbers unless `JsonStringEnumConverter`.

**const / readonly**

- `const` = compile-time, **inlined** at every use site, implicitly `static`, limited types (numerics/`bool`/`char`/`string`/`enum`/`null`).
- `readonly` = runtime, assigned in **declaration or constructor**, real field. `static readonly` = one per type, set by static ctor, any computed value.
- **Cross-assembly gotcha**: `const` values baked into consumers → changing the DLL doesn't update them until recompiled. Use `static readonly` for public changeable values.
- `const` required for `case` labels, attribute args, default params. `readonly`/`init` are **shallow** (can't reassign, can mutate the pointed-to object). `static const` is illegal.

**Tuples**

- `System.Tuple` = immutable **class** (legacy, no `==`). `ValueTuple` = **mutable struct**, no allocation, named elements, `==`, backs `(a, b)`.
- Named elements are **compile-time only**; don't affect type identity/equality.
- Deconstruct: `var (a, b) = x;` (also `foreach (var (k,v) in dict)`); discard `_`. Add `Deconstruct(out ...)` to make any type deconstructible; records auto-generate it.
- tuple = transient/internal · record = named/public/value-equality · class/struct = identity/behavior. Tuples make convenient composite dict keys. **Keep tuples out of public APIs.**

**Equality**

- Reference equality = same object; value equality = same contents.
- `ReferenceEquals` = always identity, unoverridable. `Equals` = **virtual**, binds at **runtime**. `==` = operator, binds at **compile time** on the **static type**.
- **THE CONTRACT:** equal objects **must** have equal hash codes. Override `Equals` ⇒ override `GetHashCode` (`CS0659`). Hash must be **stable while used as a key**. Use `HashCode.Combine(...)`, never `x ^ y`.
- Full class override = `IEquatable<T>` + `object.Equals` + `GetHashCode` + `==`/`!=`, all using the **same fields**; often `sealed`.
- `IEquatable<T>` → no boxing/reflection in generic collections (essential on struct keys).
- Records: auto value equality, compare by **exact runtime type** (base ≠ derived); collection members compared by reference. Default struct `Equals`/`GetHashCode` = boxing + reflection + poor distribution → override or use `readonly record struct`.
- **MUTABLE KEY GOTCHA:** mutating a hashed field after insert → wrong bucket → `ContainsKey` false, entry leaked, key can appear twice. Fix: immutable keys.
- Customize externally with **`IEqualityComparer<T>`** (`StringComparer.OrdinalIgnoreCase`); LINQ accepts one. Null-check inside equality with `is null`/`is not null`.

**Object methods**

- Inherited: `ToString`/`Equals`/`GetHashCode`/`GetType` (instance) + static `ReferenceEquals`/`Equals`, protected `MemberwiseClone`/`Finalize`.
- `ToString`/`Equals`/`GetHashCode` are **virtual**; **`GetType` is not**. Override `ToString` for logs (short, cheap, no throw).
- `GetType()` = runtime actual type (needs instance) · `typeof(T)` = compile-time · `is` = runtime *compatibility* (true for subtypes/interfaces).
- `MemberwiseClone` = shallow (value fields copied, **reference fields shared**), runs no ctor. Deep copy → manual copy-ctor (best) or JSON (slow); avoid `ICloneable`.
- `IComparable<T>.CompareTo` = natural order · `IComparer<T>.Compare` = external order; sign = `-`/`0`/`+`. Never compare via `a - b` (overflow); keep `CompareTo == 0` consistent with `Equals`.
- Write a **finalizer** only for directly-owned unmanaged resources; prefer `IDisposable` + `SafeHandle`, call `GC.SuppressFinalize(this)` in `Dispose`.

---

## A11 — `var`, `object`, and `dynamic`

### Q16. What is the `var` keyword in C#? Is it dynamically typed, and does using it impact runtime performance?

**Answer.** **`var` is implicitly typed, NOT dynamically typed.**

When you use `var`, the C# compiler infers the exact static type of the variable at **compile time** based on the right-hand side initialization expression.

```csharp
var name = "Alice"; // Compiler replaces 'var' with 'string' at compile time
var age = 30;       // Compiler replaces 'var' with 'int' at compile time
```

- **Runtime Performance**: Zero impact. The emitted CIL bytecode is 100% identical to writing `string name = "Alice";`.
- **Type Safety**: Fully statically type-checked at compile time. You cannot assign an `int` to `name` later in code.
- **Rule**: `var` requires an immediate initializer (`var x;` is a compile error) so the compiler can infer the type.

---

### Q17. What are the differences between `var`, `object`, and `dynamic` in C#?

**Answer.** All three keywords allow flexibility when declaring variables, but they operate at fundamentally different stages of compilation and execution.

| Feature | `var` | `object` | `dynamic` |
| :--- | :--- | :--- | :--- |
| **Type Checking** | **Compile time** (statically typed) | **Compile time** (base reference type) | **Runtime** (Dynamic Language Runtime) |
| **Type Binding** | Fixed at compile time | Fixed at compile time | Deferred until runtime (DLR) |
| **Performance** | Zero overhead (identical to explicit type) | Slight overhead if boxing value types | Reflection/DLR callsite caching overhead |
| **IntelliSense** | ✅ Full IDE autocomplete | ✅ Only `object` methods (`ToString`, `Equals`) | ❌ No IntelliSense autocomplete |
| **Errors** | Caught at compile time | Caught at compile time | Runtime `RuntimeBinderException` if missing |

```csharp
// 1. var — statically typed string at compile time
var v = "Hello";
int vLen = v.Length; // ✅ Checked at compile time

// 2. object — base type; requires casting to access string members
object o = "Hello";
int oLen = ((string)o).Length; // Requires explicit cast

// 3. dynamic — bypasses compile-time checks; resolves .Length at runtime
dynamic d = "Hello";
int dLen = d.Length; // Checked at runtime by DLR
// d.NonExistentMethod(); // Compiles fine! Throws RuntimeBinderException at runtime!
```

