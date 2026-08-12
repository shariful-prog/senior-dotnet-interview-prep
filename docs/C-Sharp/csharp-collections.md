# D. Collections & Data Structures
---

## D1 — The Common Collections

### Q1. What are the main collection types in C#, and when do you use each?

**Answer.** These are the ones you'll use every day. Pick based on what you need to do with the data.

| Collection | Use it when you need to... | Order |
|---|---|---|
| `List<T>` | Store items and access them by position (index) | Keeps insertion order |
| `Dictionary<TKey, TValue>` | Look things up fast by a key (like a phone book) | No guaranteed order |
| `HashSet<T>` | Keep only unique items / check "does this exist?" fast | No guaranteed order |
| `Queue<T>` | Process items in the order they arrived (first in, first out) | FIFO |
| `Stack<T>` | Process the most recent item first (last in, first out) | LIFO |

```csharp
var list = new List<string> { "a", "b", "c" };
string first = list[0];                 // access by index → "a"

var ages = new Dictionary<string, int>();
ages["alice"] = 30;                     // look up by key
int age = ages["alice"];                // → 30

var unique = new HashSet<int> { 1, 2, 2, 3 };   // → { 1, 2, 3 } (duplicates dropped)

var queue = new Queue<int>();
queue.Enqueue(1); queue.Enqueue(2);
queue.Dequeue();                        // → 1 (first one added)

var stack = new Stack<int>();
stack.Push(1); stack.Push(2);
stack.Pop();                            // → 2 (last one added)
```

**The simple rule:** need positions → `List`. Need fast lookup by key → `Dictionary`. Need uniqueness → `HashSet`. Need "line up and wait" → `Queue`. Need "undo" style (most recent first) → `Stack`.

---

### Q2. What's the difference between a `List` and a `Dictionary`?

**Answer.**

- A **`List<T>`** is like a numbered row of boxes. You find things by their **position**: `list[0]`, `list[1]`. To find a specific value you have to look through them one by one.
- A **`Dictionary<TKey, TValue>`** is like a phone book. You find things by a **key** you chose (a name, an id), and it jumps straight to the value — no scanning.

```csharp
// List: searching by value means checking each item
var names = new List<string> { "alice", "bob" };
bool hasBob = names.Contains("bob");    // has to scan the list

// Dictionary: jump straight to the value by its key
var scores = new Dictionary<string, int> { ["alice"] = 90, ["bob"] = 85 };
int bobScore = scores["bob"];           // instant lookup
```

**Interview point:** if you find yourself searching a `List` over and over to find items by some id or name, a `Dictionary` is almost always the better choice — it's built for fast lookups.

---

### Q3. How do you safely look something up in a `Dictionary` without crashing?

**Answer.** If you ask a dictionary for a key that doesn't exist using `dict[key]`, it **throws an exception**. Use `TryGetValue` (or `ContainsKey`) instead.

```csharp
var ages = new Dictionary<string, int> { ["alice"] = 30 };

// int missing = ages["bob"];           // ❌ throws KeyNotFoundException

if (ages.TryGetValue("bob", out int age))
    Console.WriteLine(age);             // only runs if "bob" exists
else
    Console.WriteLine("Not found");     // safe

// Or check first:
if (ages.ContainsKey("alice"))
    Console.WriteLine(ages["alice"]);
```

**Interview point:** `TryGetValue` is the preferred way — it checks and fetches in one step, and never throws.

---

### Q4. What is a `HashSet` good for?

**Answer.** A `HashSet<T>` stores **unique** items and lets you check "is this in here?" very fast. It automatically ignores duplicates.

Two main uses:

1. **Removing duplicates** from data.
2. **Fast membership checks** — much faster than `List.Contains` when you have lots of items.

```csharp
// Remove duplicates
var numbers = new List<int> { 1, 2, 2, 3, 3, 3 };
var unique = new HashSet<int>(numbers);     // → { 1, 2, 3 }

// Fast "have I seen this before?" check
var seen = new HashSet<string>();
foreach (var email in emails)
{
    if (!seen.Add(email))                   // Add returns false if already present
        Console.WriteLine($"Duplicate: {email}");
}
```

**Interview point:** if you only care whether something *exists* (not its position or a value attached to it), and you want that check to be fast, use a `HashSet`.

---

### Q5. Why can't you change a collection while looping over it with `foreach`?

**Answer.** If you add or remove items from a list *while* you're looping through it with `foreach`, C# throws:

> `InvalidOperationException: Collection was modified; enumeration operation may not execute.`

This is a **safety feature**. Changing the collection mid-loop could make the loop skip items or read the wrong ones, so C# stops you instead of giving wrong results.

```csharp
// ❌ This throws
foreach (var x in list)
    if (x < 0)
        list.Remove(x);

// ✅ Fix 1: use the built-in bulk remover (cleanest)
list.RemoveAll(x => x < 0);

// ✅ Fix 2: loop over a copy, change the original
foreach (var x in list.ToList())
    if (x < 0)
        list.Remove(x);

// ✅ Fix 3: use a regular for-loop going backwards
for (int i = list.Count - 1; i >= 0; i--)
    if (list[i] < 0)
        list.RemoveAt(i);
```

**Interview point:** the safest habit is to never modify the exact collection you're looping over. Use `RemoveAll`, or loop over a copy.

---

## D2 — Collection Interfaces

### Q6. What's the difference between `IEnumerable<T>`, `ICollection<T>`, and `IList<T>`?

**Answer.** These are interfaces that describe **what you can do** with a collection. Each one adds more abilities than the last:

| Interface | Purpose | Key Features |
|-----------|---------|--------------|
| `IEnumerable<T>` | Read-only iteration | `foreach`, LINQ, no add/remove, no indexing |
| `ICollection<T>` | Basic collection operations | Everything in `IEnumerable<T>` + `Count`, `Add()`, `Remove()`, `Clear()` |
| `IList<T>` | Ordered collection | Everything in `ICollection<T>` + index access (`list[index]`), `Insert()`, `RemoveAt()`, `IndexOf()` |

## Relationship

```text
IEnumerable<T>
      ↑
ICollection<T>
      ↑
IList<T>
```

Each interface inherits from the previous one.

## When to Use

### `IEnumerable<T>`
Use when you only need to **read or iterate** over a collection.

- Supports `foreach`
- Supports LINQ (`Where`, `Select`, `Any`, etc.)
- Does **not** support `Add()`, `Remove()`, or indexing

Example:

```csharp
IEnumerable<int> numbers = new List<int> { 1, 2, 3 };

foreach (var n in numbers)
{
    Console.WriteLine(n);
}
```


### `ICollection<T>`
Use when you need to **add/remove items** and know the **count**, but **indexing is not required**.

Features:
- `Count`
- `Add()`
- `Remove()`
- `Clear()`
- `Contains()`

Example:

```csharp
ICollection<int> numbers = new List<int>();

numbers.Add(1);
numbers.Add(2);

Console.WriteLine(numbers.Count);
```


### `IList<T>`
Use when you need an **ordered collection** with **index-based access**.

Additional features:
- `list[index]`
- `Insert()`
- `RemoveAt()`
- `IndexOf()`

Example:

```csharp
IList<int> numbers = new List<int>();

numbers.Add(10);
numbers.Add(20);

Console.WriteLine(numbers[0]); // 10
```

## Which One Should You Return?

Return the **least restrictive interface** that satisfies the caller's needs.

- Read-only → `IEnumerable<T>`
- Read + Add/Remove → `ICollection<T>`
- Read + Add/Remove + Indexing → `IList<T>`

This follows the **Interface Segregation Principle (ISP)** and reduces coupling.

---

### Q7. Why is it better to return `IEnumerable<T>` or `IReadOnlyList<T>` instead of `List<T>` from a method?

**Answer.** If a public method returns a `List<T>`, **anyone who calls it can change your list** — they can `Add`, `Remove`, or `Clear` it, messing up your object's internal state from the outside.

Returning a simpler interface stops that:

- Return **`IReadOnlyList<T>`** if callers only need to read and index — they can't modify it.
- Return **`IEnumerable<T>`** if callers only need to loop.

```csharp
public class OrderService
{
    private readonly List<Order> _orders = new();

    // ❌ Leaky: caller can do GetOrders().Clear() and wreck your state
    // public List<Order> GetOrders() => _orders;

    // ✅ Better: caller can read but not modify
    public IReadOnlyList<Order> GetOrders() => _orders;
}
```

**Interview point:** expose the least powerful type that still does the job. It protects your data and keeps you free to change how things work internally later.

---

### Q8. Does `IReadOnlyList<T>` mean the collection can never change?

**Answer.** **No** — this is a common trick question. `IReadOnlyList<T>` means the **caller** can't change it *through that reference*. But the object that owns the real list can still change it, and the caller will see those changes.

```csharp
List<int> list = new() { 1, 2, 3 };

IReadOnlyList<int> readOnly = list;

Console.WriteLine(readOnly[0]); // 1

list.Add(4); // Modify the original list

Console.WriteLine(readOnly.Count); // 4
```

Although `readOnly` cannot modify the collection, it still sees changes made to the original `List<int>` because both references point to the same object.

---

### What `IReadOnlyList<T>` Prevents


```csharp
IReadOnlyList<int> numbers = new List<int> { 1, 2, 3 };

// numbers.Add(4);      // ❌ Compile-time error
// numbers.Remove(1);   // ❌ Compile-time error
// numbers[0] = 10;     // ❌ Compile-time error
```

You can only:

- Read items by index (`numbers[0]`)
- Get the `Count`
- Iterate using `foreach`
---

## D3 — Iterators (`yield return`)

### Q9. What does `yield return` do?

**Answer.** `yield return` is used to **return one item at a time** instead of creating and returning the entire collection at once.

It creates an **iterator** (`IEnumerable<T>`) that generates values **lazily** (on demand). The method **pauses** after each `yield return` and **resumes** from the same point when the next item is requested.

```csharp
public static IEnumerable<int> GetNumbers()
{
    yield return 1;
    yield return 2;
    yield return 3;
}

foreach (var number in GetNumbers())
{
    Console.WriteLine(number);
}
```

**Output:**

```text
1
2
3
```


#### Real-World Example 1: Reading a Large File

Instead of loading the entire file into memory, read and process one line at a time.

```csharp
public IEnumerable<string> ReadLines(string path)
{
    using var reader = new StreamReader(path);

    string? line;
    while ((line = reader.ReadLine()) != null)
    {
        yield return line;
    }
}
```

Usage:

```csharp
foreach (var line in ReadLines("users.csv"))
{
    Console.WriteLine(line);
}
```

**Benefit:** Only one line is kept in memory at a time, making it efficient for very large files.

#### Real-World Example 2: Returning Active Employees

Instead of creating a new list of active employees, return each one as it's found.

```csharp
public IEnumerable<Employee> GetActiveEmployees(List<Employee> employees)
{
    foreach (var employee in employees)
    {
        if (employee.IsActive)
        {
            yield return employee;
        }
    }
}
```

Usage:

```csharp
foreach (var employee in GetActiveEmployees(employees))
{
    Console.WriteLine(employee.Name);
}
```

---

### Q10. What is "deferred execution", and why does it matter?

**Answer.** **Deferred execution** means a query is **not executed immediately** when it is created. Instead, it executes **only when you iterate over it** (e.g., using `foreach`, `ToList()`, `First()`, `Count()`, etc.).

This is the default behavior for most LINQ queries that return `IEnumerable<T>`.


```csharp
List<int> numbers = new() { 1, 2, 3 };

var query = numbers.Where(n => n > 1); // Query is NOT executed here

numbers.Add(4);

foreach (var number in query) // Query executes here
{
    Console.WriteLine(number);
}
```

**Output:**

```text
2
3
4
```

Notice that `4` is included because the query was executed only when the `foreach` started.


#### Immediate Execution

Methods like `ToList()`, `ToArray()`, `First()`, `Count()`, and `Single()` execute the query immediately.

```csharp
List<int> numbers = new() { 1, 2, 3 };

var result = numbers.Where(n => n > 1).ToList(); // Executes immediately

numbers.Add(4);

Console.WriteLine(string.Join(", ", result));
```

**Output:**

```text
2, 3
```

`4` is **not** included because the query had already executed.

#### Why Does It Matter?

- **Better Performance** - Only executes when the data is actually needed.
- **Lower Memory Usage** - No intermediate collection is created until execution.

---

### Q11. How does `List<T>` work internally when elements are added beyond its current `Capacity`?

**Answer.** `List<T>` is **not a linked list**. Internally, it is backed by a contiguous **array (`T[] _items`)**.

- **`Count`**: The number of elements currently stored in the list.
- **`Capacity`**: The total length of the internal array (`_items.Length`).

#### Internal Growth Strategy (Capacity Doubling):
When you call `list.Add(item)` and `Count == Capacity`:
1. `List<T>` allocates a **new array** with **double the capacity** (`newCapacity = Capacity == 0 ? 4 : Capacity * 2`).
2. It calls `Array.Copy()` to copy all existing elements from the old array into the new array ($O(N)$ allocation & copy operation).
3. It assigns the new array as the internal `_items` array and adds the new item.

```csharp
var list = new List<int>(); // Initial Capacity = 0, Count = 0
list.Add(1); // Capacity grows to 4
list.Add(2); // Capacity = 4, Count = 2
list.Add(3); // Capacity = 4, Count = 3
list.Add(4); // Capacity = 4, Count = 4
list.Add(5); // Capacity DOUBLES to 8! Array re-allocation & copy occurs here.
```

**Optimization Tip**: If you know how many items a list will hold (e.g. 10,000 items), always pass the initial capacity to the constructor `new List<int>(10_000)` to eliminate intermediate array re-allocations.

---

### Q12. What is "Multiple Enumeration" of `IEnumerable<T>`, why can it lead to bugs or performance issues, and how do you prevent it?

**Answer.** **Multiple Enumeration** happens when code iterates over an `IEnumerable<T>` parameter or LINQ expression query **more than once**.

Because `IEnumerable<T>` represents an **execution sequence/iterator** rather than in-memory data, iterating it multiple times re-evaluates the generator method or LINQ query every single time.

```csharp
public async Task ProcessOrdersAsync(IEnumerable<Order> orders)
{
    // ❌ Multiple Enumeration Bug / Performance Penalty:
    
    // 1st Enumeration: Iterates sequence to check count
    if (!orders.Any()) return; 

    // 2nd Enumeration: Re-iterates sequence to process items
    foreach (var order in orders) 
    {
        await ProcessAsync(order);
    }
}
```

#### Dangers of Multiple Enumeration:
1. **Performance Overhead**: If `orders` is an `IQueryable` or LINQ pipeline, `.Any()` and `foreach` will issue **two separate database SQL queries**!
2. **Side Effects & Bugs**: If `IEnumerable<T>` reads from an incoming network stream, file handle, or `yield return` logic with side effects, the second iteration will read duplicate or partial/exhausted data!
3. **Different Results**: If the underlying datasource changes between iterations, the two loops see different data.

#### How to Prevent It:
Materialize the sequence into an in-memory collection (using `.ToList()`, `.ToArray()`, or `.ToReadOnlyList()`) before operating on it if it will be read multiple times:

```csharp
public async Task ProcessOrdersAsync(IEnumerable<Order> orders)
{
    // ✅ Safe: Materialize once into IReadOnlyList
    var orderList = orders as IReadOnlyList<Order> ?? orders.ToList();

    if (orderList.Count == 0) return; 

    foreach (var order in orderList) 
    {
        await ProcessAsync(order);
    }
}
```