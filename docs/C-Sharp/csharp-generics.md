# C. Generics
---

## C1 — Generic Classes, Methods & Type Inference

### Q1. What are generics, and what problem do they solve?

**Answer.** **Generics** allow you to write classes, methods, and interfaces that work with **any data type** while maintaining **type safety**.

Instead of writing separate code for different types, you write it once using a **type parameter** (e.g., `T`), and specify the actual type when using it.

#### What problem do generics solve?

Before generics, collections stored objects as `object`, which had two major problems:

- **No type safety** – You could accidentally store the wrong type, causing runtime errors.
- **Boxing and unboxing** – Value types (like `int`) were converted to `object` and back, reducing performance.

Generics solve these problems by:

- Providing **compile-time type safety**.
- Eliminating unnecessary **boxing and unboxing**.
- Allowing **code reuse** for different data types.
- Improving **performance** by avoiding conversions.

#### Common generic types in C#

- `List<T>`
- `Dictionary<TKey, TValue>`
- `HashSet<T>`
- `Queue<T>`
- `Stack<T>`

```csharp
// Before generics — object-based, unsafe, boxes value types
ArrayList list = new ArrayList();
list.Add(1); // boxing
int x = (int)list[0]; // cast; runtime error if the type is wrong

// Generic — type-safe, no boxing, no cast
List<int> nums = new List<int>();
nums.Add(1); // no boxing
int y = nums[0]; // no cast
```

---

### Q3. How do generics work at runtime? What is the difference between value-type and reference-type instantiations?

**Answer.** Generics in .NET are implemented using **reified generics**, which means the actual type information (`int`, `string`, etc.) is preserved at runtime.

The CLR (Common Language Runtime) generates code differently for **value types** and **reference types**.

#### Value-type instantiations (`int`, `double`, `struct`, etc.)

For **each value type**, the CLR creates a **separate version** of the generic type or method.

Example:

```csharp
List<int>
List<double>
```

The CLR generates different implementations for `List<int>` and `List<double>`.

**Why?**
- Value types have different sizes and memory layouts.
- This avoids **boxing/unboxing** and improves performance.

---

#### Reference-type instantiations (`string`, `Employee`, etc.)

For **reference types**, the CLR creates **one shared implementation**.

Example:

```csharp
List<string>
List<Employee>
List<Customer>
```

All of these share the same generated code because every reference type is represented internally as a reference (pointer), which has the same layout.

---

### Q4. Can you overload a generic type or method? What is `default(T)`?

**Answer.** 
#### Can you overload a generic method?
**Yes.** Generic methods can be overloaded, just like regular methods, as long as their signatures are different.

Examples of valid overloads:

```csharp
void Print(int value)

void Print(string value)

void Print<T>(T value)

void Print<T>(T value, int count)
```

However, you **cannot** overload methods based **only on generic type parameter names**.

Invalid:

```csharp
void Print<T>(T value)

void Print<U>(U value) // Error - same signature
```


#### Can you overload a generic type (class)?

**No.** A class cannot be overloaded.

Invalid:

```csharp
class Repository<T> { }

class Repository<U> { } // Error
```

However, you **can** have classes with the same name if they have a **different number of generic type parameters** (different arity).

Valid:

```csharp
class Repository<T> { }

class Repository<TKey, TValue> { }
```


#### What is `default(T)`?

`default(T)` returns the **default value** for the type `T`.

- **Reference types** → `null`
- **Numeric types** → `0`
- **bool** → `false`
- **char** → `'\0'`
- **structs** → A struct with all fields initialized to their default values.

Examples:

```csharp
default(int)      // 0
default(bool)     // false
default(string)   // null
default(DateTime) // 01/01/0001 00:00:00
```
It is commonly used in generic code when the actual type is not known.
---

## C2 — Constraints (`where`)

### Q1. What are generic constraints, and why do you need them?

**Answer.** By default a type parameter `T` is treated as `object` — inside the generic you can only use members that exist on `object` (`ToString`, `Equals`, etc.). A **constraint** (`where T : ...`) restricts what types can be substituted for `T`, and in return **lets you use the capabilities you constrained to**.

```csharp
// Without a constraint you can't call CompareTo — object has no such method.
T Max<T>(T a, T b) where T : IComparable<T> // now T is guaranteed to have CompareTo
 => a.CompareTo(b) >= 0 ? a : b;
```

So constraints serve two purposes: they **enforce** that only appropriate types are used (compile-time safety), and they **unlock** members/operations on `T` inside the method body. They're the mechanism that makes generics both flexible and useful.

---

### Q2. What kinds of constraints exist?

**Answer.** The main constraint kinds:

| Constraint | Meaning |
|---|---|
| `where T : struct` | `T` is a **non-nullable value type** |
| `where T : class` | `T` is a **reference type** |
| `where T : notnull` | `T` is non-nullable (value type or non-nullable reference) |
| `where T : unmanaged` | `T` is an unmanaged value type (no reference fields) |
| `where T : new()` | `T` has a **public parameterless constructor** |
| `where T : BaseClass` | `T` derives from `BaseClass` |
| `where T : IInterface` | `T` implements `IInterface` |
| `where T : U` | `T` derives from / is another type parameter `U` |
| `where T : default` | disambiguates in override/explicit-impl scenarios |

```csharp
class Repository<T> where T : class, IEntity, new()
{
 public T Create() => new T(); // enabled by new()
 public int GetId(T item) => item.Id; // enabled by IEntity
}
```

You can apply **multiple constraints** to one parameter and constrain **multiple parameters**. Ordering rules: `class`/`struct`/`notnull` must come **first**, `new()` must come **last**, and a base class (if any) precedes interfaces.

---

### Q3. Why can't you use operators like `+` or `==` on a generic `T`, and how do constraints (or newer features) help?

**Answer.** Operators such as `+`, `-`, `<`, `==` are **static** and **not part of any interface** that the classic constraints could require, so on an unconstrained `T` the compiler doesn't know they exist. `==` is a special case: on an unconstrained `T` it may compile but bind to **reference** equality (via `object`), not the type's value `==` — a subtle bug.

Historical workarounds:
- For **comparison/equality**, constrain to `IComparable<T>` / `IEquatable<T>` and call `CompareTo`/`Equals` (which *are* interface methods), or use `Comparer<T>.Default` / `EqualityComparer<T>.Default`.

```csharp
bool AreEqual<T>(T a, T b) => EqualityComparer<T>.Default.Equals(a, b); // works for any T
```

The modern answer (**C# 11 / .NET 7**) is **static abstract interface members** and the generic-math interfaces (`INumber<T>`, `IAdditionOperators<...>`), which *do* let you write `a + b` on a constrained `T`:

```csharp
T Sum<T>(T a, T b) where T : INumber<T> => a + b; // C# 11+
```

*(Generic math is trimmed from deep coverage per the map — knowing it exists and enables operators on `T` is enough.)*

---

### Q4. What does the `new()` constraint enable, and what are its limitations?

**Answer.** `where T : new()` guarantees `T` has an accessible **parameterless constructor**, so you can write `new T()` inside the generic — useful for factories, object pools, and deserialization helpers.

```csharp
T CreateInstance<T>() where T : new() => new T();
```

Limitations worth mentioning:
- It only allows a **parameterless** constructor — you can't require a constructor with parameters via a constraint. For that, pass a **factory delegate** (`Func<T>`) instead.
- `new T()` on a reference type actually compiles to an `Activator.CreateInstance`-style call, which historically had a small performance cost (largely optimized in modern runtimes).
- `struct` constraint implies a parameterless constructor already (value types always have one), so `where T : struct` alone lets you write `new T()`.

```csharp
// When you need constructor arguments, use a factory instead of new():
T Build<T>(Func<T> factory) => factory();
```

---

