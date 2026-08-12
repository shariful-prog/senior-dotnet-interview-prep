# J. JavaScript Interview Questions
---

> TypeScript questions — `interface` vs `type`, generics, narrowing, utility types — are in [js-typescript.md](js-typescript.md).

## J1 — Variables & Types

### Q1. What is the difference between `var`, `let`, and `const`?

**Answer.** All three declare a variable. They differ in **scope** (where the variable is visible) and whether the value can be changed.
#### `var`
- Function-scoped.
- Can be reassigned and redeclared.

```javascript
var name = "John";
name = "Bob"; // OK

var name = "Alice"; // Also OK
```

#### `let`
- Block-scoped.
- Can be reassigned but cannot be redeclared in the same scope.

```javascript
let age = 25;
age = 26; // OK

// let age = 30; // Error
```

#### `const`
- Block-scoped.
- Must be initialized when declared.
- Cannot be reassigned.

```javascript
const PI = 3.14;

// PI = 3.14159; // Error
```

For objects and arrays, the reference cannot change, but their contents can:

```javascript
const user = { name: "John" };
user.name = "Bob"; // OK
```

---

### Q2. What is hoisting, and what is the temporal dead zone?

**Answer.** Hoisting is JavaScript's behavior where variable and function **declarations** are moved to the top of their scope before execution.

Example:

```javascript
console.log(x); // undefined

var x = 10;
```

JavaScript treats it like:

```javascript
var x;

console.log(x); // undefined

x = 10;
```

Only the declaration is hoisted, not the assignment.

### Temporal Dead Zone (TDZ)
The Temporal Dead Zone is the time between entering a scope and the point where a `let` or `const` variable is declared. During this time, the variable cannot be accessed.

Example:

```javascript
console.log(x); // ReferenceError

let x = 10;
```

`let` and `const` are hoisted, but they are not initialized until their declaration line executes, so accessing them before that causes a `ReferenceError`.

### Summary
- `var` → hoisted and initialized with `undefined`.
- `let` and `const` → hoisted but remain in the Temporal Dead Zone until declared.

---

### Q3. `==` vs `===` — what is the difference?

**Answer.** Both compare two values. The difference is whether they convert types first.

- **`===` (strict equality)** — compares value **and** type. No conversion. If the types differ, the result is `false`.
- **`==` (loose equality)** — converts the operands to a common type first, then compares. This conversion is called **coercion**.

```js
5 === "5"     // false — number vs string
5 ==  "5"     // true  — "5" is converted to 5

null == undefined     // true  — a special rule in the spec
null === undefined    // false — different types

0 == ""       // true  — both convert to 0
0 == false    // true
NaN == NaN    // false — NaN is never equal to anything
```

The conversion rules are irregular, which is why `==` produces results people do not expect. **Always use `===`.** The one accepted use of `==` is `x == null`, which checks for `null` or `undefined` in a single test.

---

### Q4. What are the primitive types, and how do they differ from objects?

**Answer.** JavaScript has **7 primitives**: `string`, `number`, `boolean`, `null`, `undefined`, `symbol`, `bigint`. Everything else — objects, arrays, functions, dates — is an **object**.

The difference is how they are copied and compared.

**Primitives are copied by value.** Each variable holds its own copy:

```js
let a = 1;
let b = a;
b = 2;
console.log(a);   // 1 — unaffected
```

**Objects are copied by reference.** Both variables point at the same thing:

```js
let x = { n: 1 };
let y = x;
y.n = 2;
console.log(x.n);   // 2 — same object

{ n: 1 } === { n: 1 }   // false — two different objects
```

Primitives are also **immutable**. `str.toUpperCase()` returns a new string rather than changing the original.

❌ `typeof null` returns `"object"`. This is a long-standing bug in the language, kept for backwards compatibility. To test for null, use `x === null`.

---

### Q5. What does `typeof` return, and how do you reliably check a type?

**Answer.** `typeof` is a JavaScript operator that returns the type of a value as a string.

Examples:

```javascript
typeof "Hello";   // "string"
typeof 10;        // "number"
typeof true;      // "boolean"
typeof {};        // "object"
typeof undefined; // "undefined"
typeof function(){}; // "function"
```

One common bug:

```javascript
typeof null; // "object"
```

This is a historical JavaScript behavior.

---

## J2 — Functions & Scope

### Q6. What is a closure?

**Answer.** A **closure** is a function that remembers and can access variables from its outer scope, even after the outer function has finished executing.

Suppose you create an order and return a function to send a confirmation email. Even though the order creation function has already finished executing, the email function still remembers the customer's name and order ID. This is possible because of a **closure**.

```javascript
function createOrder(customerName, orderId) {
  return function sendConfirmationEmail() {
    console.log(
      `Email sent to ${customerName}: Your order #${orderId} has been confirmed.`
    );
  };
}

const sendEmail = createOrder("Alice", 1001);

// Later...
sendEmail();
```

**Output:**

```
Email sent to Alice: Your order #1001 has been confirmed.
```

Although `createOrder()` has already returned, `sendConfirmationEmail()` still remembers `customerName` and `orderId`. This is a **closure**.

#### Common Use Cases
- Event handlers
- Callbacks
- Private variables
- Memoization
- Configuration functions

---

### Q7. How does `this` work, and how do arrow functions change it?

**Answer.** **`this`** is a reference to the object a function is currently acting on. In a normal function its value is not fixed when you write the function — it is decided **at the moment of the call**, by how the function is called.

```js
const user = {
  name: "John",
  greet() { return this.name; }
};

user.greet();              // "John" — called on user

const fn = user.greet;
fn();                      // undefined — no owner at the call site
```

**Arrow functions have no `this` of their own.** They use the `this` of the scope where they were written, and it cannot be changed:

```js
const user = {
  name: "Alice",
  greet() {
    setTimeout(() => {
      console.log(this.name);
    }, 1000);
  }
};

user.greet(); // Alice
```

#### Without an Arrow Function

```javascript
const user = {
  name: "Alice",
  greet() {
    setTimeout(function () {
      console.log(this.name);
    }, 1000);
  }
};

user.greet(); // undefined
```

---

### Q8. What do `call`, `apply`, and `bind` do?

**Answer.** Three methods available on every function, all of which let you set `this` explicitly rather than letting the call site decide it (Q7).
### `call()`
Calls a function immediately and lets you specify what `this` should be. Arguments are passed individually.

```javascript
const user = { name: "Alice" };

function greet(city) {
  console.log(`${this.name} from ${city}`);
}

greet.call(user, "London"); // Alice from London
```

### `apply()`
Works like `call()`, but arguments are passed as an array.


```javascript
const user = { name: "Alice" };

function introduce(city, country) {
  console.log(`${this.name} lives in ${city}, ${country}`);
}

introduce.apply(user, ["London", "UK"]);
// Alice lives in London, UK
```

### `bind()`
Does **not** call the function immediately. It returns a new function with `this` permanently set.

```javascript
const user = { name: "Alice" };

function greet() {
  console.log(this.name);
}

const sayHello = greet.bind(user);

sayHello(); // Alice
```


---

### Q9. What are default, rest, and spread parameters?

**Answer.** Three features that all involve function arguments, and two of which share the `...` syntax.

- **Default parameter** — a fallback value used when an argument is not supplied
- **Rest parameter** — collects any number of remaining arguments into an array
- **Spread** — expands an array or object into individual items

Rest and spread are the same three dots doing opposite jobs, told apart by where they appear.

#### Default
a value used when the argument is `undefined`:

```js
function greet(name = "guest") { return `Hi ${name}`; }
greet();            // "Hi guest"
greet(undefined);   // "Hi guest"
greet(null);        // "Hi null" — null is a value, so no default
```

#### Rest Parameters (`...`)
Rest parameters collect multiple arguments into a single array.

```javascript
function sum(...numbers) {
  console.log(numbers);
}

sum(1, 2, 3, 4); // [1, 2, 3, 4]
```

#### Spread Operator (`...`)
The spread operator expands an array (or object) into individual elements.

```javascript
const numbers = [1, 2, 3];

console.log(...numbers); // 1 2 3
```

It is commonly used to pass array elements as function arguments.

```javascript
function add(a, b, c) {
  return a + b + c;
}

const numbers = [1, 2, 3];

console.log(add(...numbers)); // 6
```

---

## J3 — Asynchronous JavaScript

### Q10. What is the event loop?

**Answer.** JavaScript runs on **one thread** — one piece of code at a time. The **event loop** is what lets it handle asynchronous work without blocking.
The **Event Loop** is JavaScript's mechanism for handling asynchronous tasks. It continuously checks whether the **Call Stack** is empty, and if it is, it moves tasks from the **Callback Queue** to the Call Stack for execution.


Three parts:

- **Call stack** — the function currently running.
- **Task queues** — callbacks waiting to run.
- **Event loop** — when the stack is empty, it moves the next callback onto it.

Slow work (timers, network, file access) is handed to the browser or Node, not the thread. When it finishes, its callback joins a queue.

**Microtasks run before macrotasks.** Promise callbacks are microtasks; `setTimeout` is a macrotask:

```js
console.log("1");
setTimeout(() => console.log("2"), 0);      // macrotask
Promise.resolve().then(() => console.log("3"));   // microtask
console.log("4");

// 1, 4, 3, 2
```

`1` and `4` are synchronous. Then the stack empties, **all** microtasks run (`3`), and only then the macrotask (`2`).

❌ `setTimeout(fn, 0)` does not mean "run now". It means "run after the stack clears and all microtasks finish".

---

### Q11. Callbacks, promises, `async`/`await` — what problem does each solve?

**Answer.** All three are ways to handle a result that is not available yet. They arrived in that order, each solving a problem with the one before.

- **Callback** — a function you pass in, to be called when the work finishes
- **Promise** — an object representing a future value, with `.then()` and `.catch()`
- **`async`/`await`** — syntax that lets you write promise-based code as if it were synchronous

**Callbacks** work, but nesting them becomes unreadable and error handling must be repeated at every level:

```js
getUser(id, (err, user) => {
  if (err) return handle(err);
  getOrders(user, (err, orders) => {      // callback hell
    if (err) return handle(err);
  });
});
```

**Promises** flatten the nesting and give one error path:

```js
getUser(id)
  .then(user => getOrders(user))
  .then(orders => render(orders))
  .catch(handle);                          // one handler for the whole chain
```

A promise is an object with a state: **pending**, then either **fulfilled** or **rejected**. Once settled it never changes.

**`async`/`await`** is the same promise machinery with synchronous-looking syntax, and it uses ordinary `try`/`catch`:

```js
async function load(id) {
  try {
    const user = await getUser(id);
    const orders = await getOrders(user);
    return orders;
  } catch (err) {
    handle(err);
  }
}
```

`await` pauses that function only, not the thread.

❌ **Sequential awaits waste time when the calls are independent:**

```js
const a = await getA();     // 1s
const b = await getB();     // 1s → 2s total

const [a, b] = await Promise.all([getA(), getB()]);   // ✅ 1s
```

---

### Q12. What does `Promise.all` do, and how does it differ from `allSettled` and `race`?

**Answer.** All three take an array of promises and return one promise. They differ in when they settle.

#### `Promise.all()`
Runs multiple promises in parallel and waits for **all** of them to succeed. If **any one** promise fails, it immediately rejects.

```javascript
const p1 = Promise.resolve("A");
const p2 = Promise.resolve("B");

Promise.all([p1, p2])
  .then(result => console.log(result)); // ["A", "B"]
```

#### `Promise.allSettled()`
Waits for **all** promises to finish, whether they succeed or fail.

```javascript
const p1 = Promise.resolve("A");
const p2 = Promise.reject("Error");

Promise.allSettled([p1, p2])
  .then(result => console.log(result));
```

Output:

```text
[
  { status: "fulfilled", value: "A" },
  { status: "rejected", reason: "Error" }
]
```

#### `Promise.race()`
Returns the result of the **first promise to settle** (fulfilled or rejected).

```javascript
const p1 = new Promise(resolve => setTimeout(() => resolve("A"), 2000));
const p2 = new Promise(resolve => setTimeout(() => resolve("B"), 1000));

Promise.race([p1, p2])
  .then(result => console.log(result)); // B
```

---

### Q13. How do you handle errors in async code?

**Answer.** An async operation can fail after the surrounding function has already returned, so a plain `try`/`catch` does not always cover it. The mechanism you use has to match the syntax you wrote.

**With `async`/`await`** — ordinary `try`/`catch`:

```js
try {
  const user = await getUser(id);
} catch (err) {
  handle(err);            // catches both thrown errors and rejections
}
```

**With promises** — `.catch()`, placed last so it covers the whole chain:

```js
getUser(id)
  .then(u => getOrders(u))
  .catch(handle);         // catches a failure in either step
```

❌ **An unawaited promise escapes `try`/`catch`:**

```js
try {
  getUser(id);            // ❌ no await — rejection is unhandled
} catch (err) { }         // never runs
```

❌ **`fetch` does not reject on HTTP errors.** A 404 or 500 is a successful response as far as `fetch` is concerned. Only a network failure rejects:

```js
const res = await fetch(url);
if (!res.ok) throw new Error(res.status);   // ✅ you must check this yourself
const data = await res.json();
```

Use `finally` for cleanup that must run either way, such as hiding a loading spinner.

---

## J4 — Objects & Prototypes

### Q14. What is the prototype chain?

**Answer.** Every JavaScript object has a hidden internal link to another object, called its **prototype**. That prototype has its own prototype, and so on until one has `null`. This series of links is the **prototype chain**.

It exists to make **inheritance** work: when you read a property the object does not have, JavaScript follows the chain upward until it finds the property or reaches the end.

```js
const arr = [1, 2];
arr.push(3);        // not on arr — found on Array.prototype
arr.toString();     // not there either — found on Object.prototype
```

That chain is how inheritance works in JavaScript. There are no classes underneath — `class` is syntax over prototypes:

```js
class Animal {
  speak() { return "sound"; }
}

class Dog extends Animal {
  speak() { return "bark"; }        // found first, so it wins
}

new Dog().speak();     // "bark"
```

**Shadowing** is the consequence: a property on the object hides one further up the chain. Nothing is overwritten — the lookup simply stops earlier.

---

### Q15. Shallow copy vs deep copy?

**Answer.** Both create a copy of an object. The difference is how deep the copying goes.

- **Shallow copy** — copies the top-level properties only. Nested objects are **shared** between the copy and the original.
- **Deep copy** — copies every level, so nothing is shared and the two are fully independent.

The distinction only matters when the object contains other objects. With a flat object of primitives, both are the same.

**Shallow:**

```js
const original = { name: "John", address: { city: "Dhaka" } };

const shallow = { ...original };
shallow.address.city = "Oslo";
console.log(original.address.city);    // "Oslo" ❌ — shared
```

Spread and `Object.assign` are both shallow.

A **deep** copy duplicates every level, so nothing is shared:

```js
const deep = structuredClone(original);   // ✅ built in, handles Date, Map, Set
deep.address.city = "Oslo";
console.log(original.address.city);       // "Dhaka"
```

`JSON.parse(JSON.stringify(obj))` also deep-copies, but it silently loses `undefined`, functions, `Date` objects (they become strings), `Map`, `Set`, and throws on circular references. Prefer `structuredClone`.

---

### Q16. What is destructuring?

**Answer.** **Destructuring** is syntax for extracting values from an object or array into individual variables in a single statement, instead of accessing each property by hand.

```js
// without destructuring
const name = user.name;
const age = user.age;

// with destructuring
const { name, age } = user;
```

```js
const user = { name: "John", age: 30, city: "Dhaka" };

const { name, age } = user;              // by property name
const { city: location } = user;         // rename while extracting
const { country = "BD" } = user;         // default when missing

const [first, second] = [1, 2, 3];       // by position
const [, , third] = [1, 2, 3];           // skip with commas
```

It works in parameter lists too, which is common for options objects:

```js
function createUser({ name, role = "user" }) {
  return `${name} (${role})`;
}
createUser({ name: "John" });     // "John (user)"
```

**Nested destructuring** reads deeper values in one statement:

```js
const { address: { city } } = user;      // city, not address
```

❌ Destructuring `null` or `undefined` throws. Guard with a default:

```js
const { name } = user ?? {};             // ✅ safe when user is missing
```

---

### Q17. What do `Object.keys`, `values`, and `entries` return?

**Answer.** All three take an object and return an array of its **own enumerable** properties — inherited ones are skipped.

```js
const user = { name: "John", age: 30 };

Object.keys(user);      // ["name", "age"]
Object.values(user);    // ["John", 30]
Object.entries(user);   // [["name", "John"], ["age", 30]]
```

`entries` is what lets you loop an object with destructuring:

```js
for (const [key, value] of Object.entries(user)) {
  console.log(`${key}: ${value}`);
}
```

`Object.fromEntries` reverses it, which makes filtering or transforming an object straightforward:

```js
const filtered = Object.fromEntries(
  Object.entries(user).filter(([k, v]) => v !== null)
);
```

Note that objects have no `.map` or `.filter` of their own — converting to entries and back is the standard way to do it.

---

## J5 — Common Interview Traps

### Q18. What is the difference between `null` and `undefined`?

**Answer.** `undefined` means **no value has been assigned**. JavaScript produces it for you. `null` means **deliberately empty** — you assign it yourself.

```js
let a;                    // undefined — declared, never set
const obj = {};
obj.missing;              // undefined — property does not exist
function f() {}
f();                      // undefined — no return

let b = null;             // null — an intentional "nothing"
```

```js
typeof undefined    // "undefined"
typeof null         // "object"  ← the historical bug

null == undefined   // true
null === undefined  // false
```

Use `??` to supply a default for either, without catching valid falsy values:

```js
count ?? 10      // 10 only when count is null or undefined
count || 10      // 10 also when count is 0 or "" ❌
```

---

### Q19. What are truthy and falsy values?

**Answer.** Any value used in a condition is converted to a boolean. There are exactly **8 falsy values**; everything else is truthy.

```js
false, 0, -0, 0n, "", null, undefined, NaN
```

The surprises are what is *truthy*:

```js
if ([])  { }      // ✅ runs — an empty array is truthy
if ({})  { }      // ✅ runs — an empty object is truthy
if ("0") { }      // ✅ runs — a non-empty string is truthy
```

To test whether an array has items, check its length: `if (arr.length)`.

❌ `||` treats `0` and `""` as missing, which is a common source of bugs. Use `??` when only `null` and `undefined` should trigger the default (Q18).

---

### Q20. `map`, `forEach`, `filter`, `reduce` — when do you use each?

**Answer.** All four iterate over an array. The difference is **what each returns**, which decides what it is for.

The last three are **pure** — they return a new value and leave the original array untouched. `forEach` returns nothing, so it exists purely for side effects.

| Method | Returns | Use for |
|---|---|---|
| `forEach` | `undefined` | side effects — logging, saving |
| `map` | new array, same length | transforming every item |
| `filter` | new array, fewer items | selecting items |
| `reduce` | a single value | totals, grouping, flattening |

```js
const nums = [1, 2, 3, 4];

nums.forEach(n => console.log(n));       // undefined
nums.map(n => n * 2);                    // [2, 4, 6, 8]
nums.filter(n => n % 2 === 0);           // [2, 4]
nums.reduce((sum, n) => sum + n, 0);     // 10
```

They can be chained, since `map` and `filter` return arrays:

```js
orders
  .filter(o => o.paid)
  .map(o => o.total)
  .reduce((a, b) => a + b, 0);
```

❌ **`forEach` ignores `await`.** It does not wait, so the loop finishes before the work does. Use `for...of` for async work:

```js
items.forEach(async i => await save(i));    // ❌ does not wait
for (const i of items) await save(i);       // ✅ waits
```

---

### Q21. What is event delegation?

**Answer.** Instead of attaching a listener to every element, you attach **one** to a shared ancestor and check what was clicked. It works because events **bubble** up from the target through its parents.

```js
// ❌ one listener per row — and new rows get none
document.querySelectorAll(".delete").forEach(btn =>
  btn.addEventListener("click", handleDelete));

// ✅ one listener on the container
document.querySelector("#list").addEventListener("click", e => {
  if (e.target.matches(".delete")) {
    handleDelete(e);
  }
});
```

Two benefits: fewer listeners, and it keeps working for elements added **after** the listener was attached — which the first version does not.

`e.target` is the element actually clicked. `e.currentTarget` is the element holding the listener.

---

### Q22. What is the difference between `slice` and `splice`?

**Answer.** Two array methods with similar names and very different behaviour.

- **`slice`** — returns a **copy** of part of the array. The original is untouched.
- **`splice`** — **removes or inserts** items, changing the original array in place, and returns whatever it removed.

The distinction is *mutation*: `slice` is safe, `splice` modifies your data.

```js
const arr = [1, 2, 3, 4, 5];

arr.slice(1, 3);     // [2, 3] — new array, arr unchanged
arr;                 // [1, 2, 3, 4, 5]

arr.splice(1, 2);    // [2, 3] — returns what it REMOVED
arr;                 // [1, 4, 5] ❌ arr was changed
```

| | Mutates original | Returns | Arguments |
|---|---|---|---|
| `slice` | no | the copied part | start, end (exclusive) |
| `splice` | **yes** | the removed part | start, count, items to insert |

`splice` can also insert:

```js
const a = [1, 4];
a.splice(1, 0, 2, 3);    // remove 0 items at index 1, insert 2 and 3
a;                        // [1, 2, 3, 4]
```

Other mutating methods worth remembering: `push`, `pop`, `shift`, `unshift`, `sort`, `reverse`. `sort` mutating in place surprises people:

```js
const sorted = [...arr].sort();     // ✅ copy first
```

---

### Q23. Why does `[10, 9, 1].sort()` give the wrong order?

**Answer.** Because `sort()` with no arguments does not compare numbers. It converts every item to a **string** and sorts those alphabetically, so `"10"` sorts before `"9"` — the same way `"apple"` sorts before `"banana"`.

```js
[10, 9, 1].sort();          // [1, 10, 9] ❌ — "10" < "9" as text
```

Pass a comparator to sort numerically:

```js
[10, 9, 1].sort((a, b) => a - b);    // [1, 9, 10] ✅ ascending
[10, 9, 1].sort((a, b) => b - a);    // [10, 9, 1] ✅ descending
```

The comparator must return a **number**: negative if `a` comes first, positive if `b` does, `0` to leave them as they are.

For strings, `localeCompare` handles accents and case correctly:

```js
names.sort((a, b) => a.localeCompare(b));
```

❌ `sort` mutates the array (Q22). Use `[...arr].sort(...)` or `arr.toSorted(...)` in newer environments.

---

### Q24. What is the difference between `for...in` and `for...of`?

**Answer.** `for...in` loops over **keys**. `for...of` loops over **values**.

```js
const arr = ["a", "b"];

for (const i of arr) console.log(i);    // "a", "b"  — values
for (const i in arr) console.log(i);    // "0", "1"  — indexes, as strings
```

`for...of` works on iterables — arrays, strings, `Map`, `Set` — but **not** plain objects. `for...in` works on objects, and also walks up the prototype chain, which is why it can pick up properties you did not define.

```js
const user = { name: "John" };
for (const key in user) console.log(key);           // "name"
for (const [k, v] of Object.entries(user)) { }      // ✅ preferred for objects
```

The rule: **`for...of` for arrays, `Object.entries` for objects.** Avoid `for...in` on arrays — you get string indexes and no guaranteed order.

---

## J6 — ES6+ Features

### Q25. What are template literals?

**Answer.** A **template literal** is a string written with backticks instead of quotes. It adds two things regular strings lack: **interpolation** (embedding expressions with `${}`) and **multi-line** support without escape characters.

```js
const name = "John";

`Hi ${name}`                 // "Hi John"
`Total: ${price * qty}`      // any expression works
`line one
line two`                    // multi-line, no \n needed
```

Before this you had to concatenate with `+`, which was noisy and easy to get wrong. Expressions inside `${}` are evaluated and converted to strings.

❌ Interpolating user input into HTML is an XSS risk — template literals do no escaping:

```js
el.innerHTML = `<div>${userInput}</div>`;   // ❌ unsafe
el.textContent = userInput;                 // ✅ safe
```

---

### Q26. What do `?.` and `??` do?

**Answer.** Two ES2020 operators for working with values that may be missing.

- **Optional chaining (`?.`)** — reads a property safely, returning `undefined` instead of throwing when part of the path does not exist
- **Nullish coalescing (`??`)** — supplies a fallback, but only when the value is `null` or `undefined`

**Optional chaining `?.`** stops and returns `undefined` instead of throwing when something in the path is `null` or `undefined`:

```js
user.address.city       // ❌ TypeError if address is undefined
user?.address?.city     // ✅ undefined

user.getName?.()        // only calls if the method exists
arr?.[0]                // works for indexes too
```

**Nullish coalescing `??`** supplies a default only for `null` and `undefined`:

```js
count ?? 10       // 10 only when count is null/undefined
count || 10       // 10 also when count is 0 or "" ❌
```

They combine well for reading configuration:

```js
const city = user?.address?.city ?? "unknown";
```

The difference from `||` matters whenever `0`, `""`, or `false` are valid values (Q19).

---

### Q27. `Map` vs object, and `Set` vs array?

**Answer.** Two collection types added in ES6, each an improvement on using a plain object or array for a specific job.

- **`Map`** — key/value pairs where the key can be **any type**, not just a string
- **`Set`** — a collection of **unique** values, with fast membership checks

**`Map` vs object** — a `Map` allows any key type and keeps insertion order:

```js
const m = new Map();
m.set("a", 1);
m.set(42, "num");           // ✅ number key
m.set(objKey, "value");     // ✅ object key

m.get("a");     // 1
m.size;         // 3 — objects need Object.keys(o).length
m.has("a");     // true
```

Object keys are always strings or symbols, so `obj[42]` becomes `obj["42"]`. Use a `Map` when keys are not strings, when you add and remove often, or when order matters. Use an object for fixed, known-shape data.

**`Set` vs array** — a `Set` holds unique values only:

```js
const s = new Set([1, 2, 2, 3]);
s.size;          // 3 — duplicates dropped
s.has(2);        // true — fast, no scanning

[...new Set(arr)]    // ✅ the standard way to dedupe an array
```

`Set.has` is much faster than `array.includes` on large collections, because it does not scan.

---

### Q28. What is the difference between named and default exports?

**Answer.** Two forms of ES module export, differing in how many a file may have and how they are imported.

- **Named export** — any number per file. The importer must use the **exact name**.
- **Default export** — at most **one** per file. The importer chooses any name.

```js
// named — any number per file, imported by exact name
export const PI = 3.14;
export function add(a, b) { return a + b; }

import { PI, add } from "./math.js";
import { add as sum } from "./math.js";     // rename on import
```

```js
// default — one per file, named by the importer
export default class User { }

import User from "./user.js";
import Whatever from "./user.js";           // any name works
```

Named exports are usually preferred: the name is fixed, so search and refactoring tools can follow it, and a typo is an error rather than a silently `undefined` import.

`import * as utils from "./utils.js"` brings in everything under one namespace object.

---

## J7 — Browser & Practical

### Q29. What is the difference between `localStorage`, `sessionStorage`, and cookies?

**Answer.** Three browser storage mechanisms. All persist data on the client, but they differ in **size**, **lifetime**, and whether the data is sent to the server.

- **`localStorage`** — key/value strings, persists until explicitly cleared
- **`sessionStorage`** — the same API, but cleared when the tab closes
- **Cookies** — small values sent automatically with every HTTP request

| | Size | Lifetime | Sent to server |
|---|---|---|---|
| `localStorage` | ~5–10 MB | until cleared | no |
| `sessionStorage` | ~5–10 MB | until the tab closes | no |
| Cookie | ~4 KB | a set expiry | **yes, every request** |

```js
localStorage.setItem("theme", "dark");
localStorage.getItem("theme");       // "dark"
localStorage.removeItem("theme");
```

Both storage APIs hold **strings only**, so objects must be serialised:

```js
localStorage.setItem("user", JSON.stringify(user));
const user = JSON.parse(localStorage.getItem("user"));
```

❌ **Never store tokens or personal data in `localStorage`.** Any JavaScript on the page can read it, so a single XSS flaw exposes everything. Auth tokens belong in an `httpOnly` cookie, which JavaScript cannot read.

---

### Q30. What is debouncing, and how does it differ from throttling?

**Answer.** Both are techniques for limiting how often a function runs when its trigger fires rapidly — typing, scrolling, resizing. They differ in *when* they let the call through.

- **Debounce** — waits until the activity **stops**, then runs once. Every new event resets the timer.
- **Throttle** — runs at most once per interval **while** activity continues, ignoring the rest.

```js
function debounce(fn, delay) {
  let timer;
  return (...args) => {
    clearTimeout(timer);                       // cancel the previous one
    timer = setTimeout(() => fn(...args), delay);
  };
}
```

With a 300 ms debounce, typing 10 characters quickly fires **once**, 300 ms after the last keystroke. With a 300 ms throttle, it fires roughly every 300 ms while typing continues.

| | Fires | Use for |
|---|---|---|
| Debounce | after activity stops | search-as-you-type, autosave, resize |
| Throttle | at a fixed rate during activity | scroll position, mousemove, drag |

The rule: **debounce when only the final state matters, throttle when you need regular updates.**

---

### Q31. What is the difference between synchronous and asynchronous code?

**Answer.** **Synchronous** code runs line by line, each finishing before the next starts. **Asynchronous** code starts an operation and continues, handling the result later.

```js
console.log("1");
console.log("2");        // waits for line 1 — synchronous

console.log("A");
setTimeout(() => console.log("B"), 0);
console.log("C");        // A, C, B — asynchronous
```

This matters because JavaScript has **one thread**. Synchronous work blocks everything — the page cannot respond to clicks or repaint while it runs:

```js
// ❌ blocks the browser for a second
const start = Date.now();
while (Date.now() - start < 1000) { }
```

Anything slow — network calls, file reads, timers — must be asynchronous, which is why they return promises. The event loop (Q10) is what makes this work on a single thread.

For genuinely CPU-heavy work, a **Web Worker** runs it on a separate thread so the page stays responsive.

---

### Q32. What is Currying in JavaScript, and how do you implement a generic `curry()` function?

**Answer.** **Currying** is a functional programming technique where a function that takes multiple arguments is transformed into a sequence of functions, each taking a **single argument**.

```js
// Standard multi-argument function
function add(a, b, c) {
  return a + b + c;
}

// Curried equivalent
function curriedAdd(a) {
  return function(b) {
    return function(c) {
      return a + b + c;
    };
  };
}

curriedAdd(1)(2)(3); // 6
```

#### Generic `curry()` Implementation:
A generic currying utility inspects `fn.length` (the number of declared parameters) and accumulates arguments until enough are supplied:

```js
function curry(fn) {
  return function curried(...args) {
    if (args.length >= fn.length) {
      return fn.apply(this, args);
    }
    return function(...nextArgs) {
      return curried.apply(this, args.concat(nextArgs));
    };
  };
}

const sum = (a, b, c) => a + b + c;
const curriedSum = curry(sum);

console.log(curriedSum(1)(2)(3)); // 6
console.log(curriedSum(1, 2)(3));  // 6
```

**Why use currying?** It enables **partial application** and function reusability (e.g. creating specialized helper functions like `const logError = log("ERROR")`).

---

### Q33. What are `WeakMap` and `WeakSet`, and how do they prevent memory leaks compared to `Map` and `Set`?

**Answer.** **`WeakMap`** and **`WeakSet`** are garbage-collection-friendly variants of `Map` and `Set`.

#### Key Differences:
1. **Weak References**: Keys in a `WeakMap` (and values in a `WeakSet`) must be **objects or non-registered symbols**. They are held as **weak references**. If there are no other strong references to an object in memory, the Garbage Collector automatically frees it and removes the key from the `WeakMap`.
2. **Non-Iterable**: You cannot iterate (`forEach`, `for...of`), nor access `.size`, `.keys()`, or `.values()` on a `WeakMap` or `WeakSet`, because the garbage collector could reclaim entries at any moment.

```js
let user = { name: "Bob" };

const weakMap = new WeakMap();
weakMap.set(user, { metadata: "DOM node ref" });

user = null; // Object is now eligible for Garbage Collection!
// The entry inside weakMap will be automatically cleaned up without leaking memory.
```

**Common Use Case:** Storing private data or DOM element metadata. When the DOM element is removed from the document, its associated metadata in `WeakMap` is garbage collected automatically.

---

### Q34. How do Generator functions (`function*`) work, and what is the Iterator Protocol (`Symbol.iterator`)?

**Answer.** **Generator functions** (`function*`) are special functions that can be paused and resumed using the `yield` keyword. Calling a generator function does not run its body immediately—instead, it returns a **Generator Object** that implements both the **Iterable** and **Iterator** protocols.

```js
function* numberSequence() {
  yield 1;
  yield 2;
  yield 3;
}

const gen = numberSequence();
console.log(gen.next()); // { value: 1, done: false }
console.log(gen.next()); // { value: 2, done: false }
console.log(gen.next()); // { value: 3, done: false }
console.log(gen.next()); // { value: undefined, done: true }
```

#### Custom Iterable Object using `Symbol.iterator`:
Any object can be made iterable with `for...of` by implementing `[Symbol.iterator]`:

```js
const customRange = {
  from: 1,
  to: 3,
  *[Symbol.iterator]() {
    for (let value = this.from; value <= this.to; value++) {
      yield value;
    }
  }
};

for (const num of customRange) {
  console.log(num); // 1, 2, 3
}
```

---

### Q35. What is the `Symbol` primitive type, and how are `Symbol.for()` and well-known Symbols used?

**Answer.** **`Symbol`** is a primitive data type (introduced in ES6) that represents a **guaranteed unique, immutable identifier**.

```js
const id1 = Symbol("id");
const id2 = Symbol("id");

console.log(id1 === id2); // false (Every Symbol() creates a distinct reference)
```

#### 1. Hidden / Non-Enumerable Properties
Symbols do not appear in `Object.keys()` or `JSON.stringify()`, preventing property name collisions when adding metadata to third-party objects. Access them via `Object.getOwnPropertySymbols(obj)`.

#### 2. Global Symbol Registry (`Symbol.for`)
`Symbol.for(key)` searches the global symbol registry. If found, it returns the existing Symbol; if not, it creates a global one:

```js
const globalSym1 = Symbol.for("app.user");
const globalSym2 = Symbol.for("app.user");
console.log(globalSym1 === globalSym2); // true
```

#### 3. Well-Known Symbols
Built-in JavaScript symbols customize object language behaviors:
- `Symbol.iterator` — Customizes `for...of` iteration (Q34).
- `Symbol.hasInstance` — Customizes `instanceof` behavior.
- `Symbol.toStringTag` — Customizes `Object.prototype.toString.call(obj)`.

---

### Q36. What are `MutationObserver`, `ResizeObserver`, and `IntersectionObserver` Web APIs?

**Answer.** These three modern Browser APIs replace expensive scroll/resize event listeners and polling loops with **asynchronous, browser-optimized observer callbacks**.

| Observer API | Triggers When | Common Use Cases |
| :--- | :--- | :--- |
| **`IntersectionObserver`** | An element becomes visible inside the viewport | Lazy-loading images, infinite scrolling, tracking ad visibility |
| **`ResizeObserver`** | An element's content rectangle changes size | Container queries, dynamic chart redrawing |
| **`MutationObserver`** | DOM nodes or attributes are added/removed/changed | Third-party script DOM monitoring, extension content scripts |

```js
// IntersectionObserver Example (Lazy Loading Images):
const observer = new IntersectionObserver((entries, obs) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      const img = entry.target;
      img.src = img.dataset.src; // Load real image URL
      obs.unobserve(img);        // Stop observing once loaded
    }
  });
});

document.querySelectorAll("img.lazy").forEach(img => observer.observe(img));
```

---

### Q37. What is the difference between `Object.freeze()`, `Object.seal()`, and `Object.preventExtensions()`?

**Answer.** JavaScript provides three levels of object immutability controls:

| Feature | `Object.preventExtensions()` | `Object.seal()` | `Object.freeze()` |
| :--- | :--- | :--- | :--- |
| **Add New Properties** | ❌ Prevented | ❌ Prevented | ❌ Prevented |
| **Delete Existing Properties** | ✅ Allowed | ❌ Prevented | ❌ Prevented |
| **Modify Property Values** | ✅ Allowed | ✅ Allowed | ❌ Prevented |
| **Reconfigure Property Descriptors** | ✅ Allowed | ❌ Prevented | ❌ Prevented |

```js
const obj = { name: "Alice" };
Object.freeze(obj);

obj.name = "Bob"; // Silently fails in sloppy mode; throws TypeError in strict mode!
```

⚠️ **Shallow Freeze Warning:** `Object.freeze()` is **shallow**. Nested objects inside a frozen object can still be mutated unless recursively deep-frozen!

---

### Q38. Microtasks vs Macrotasks — how does the Event Loop prioritize `queueMicrotask` / `Promise.then` vs `setTimeout`?

**Answer.** The Event Loop handles two distinct execution queues: **Microtask Queue** and **Macrotask Queue (Task Queue)**.

- **Microtasks**: `Promise.then/catch/finally`, `queueMicrotask()`, `process.nextTick` (Node.js), `MutationObserver`.
- **Macrotasks**: `setTimeout`, `setInterval`, `setImmediate`, I/O tasks, UI rendering events.

#### Priority Rule:
After the current synchronous stack finishes, **the Event Loop empties the ENTIRE Microtask queue before picking the next SINGLE Macrotask**.

```js
console.log("1: Sync");

setTimeout(() => console.log("2: Macrotask"), 0);

Promise.resolve().then(() => {
  console.log("3: Microtask 1");
  queueMicrotask(() => console.log("4: Microtask 2"));
});

console.log("5: Sync End");

// Output:
// 1: Sync
// 5: Sync End
// 3: Microtask 1
// 4: Microtask 2
// 2: Macrotask
```

---

### Q39. What are the common causes of JavaScript memory leaks, and how does Garbage Collection (Mark-and-Sweep) work?

**Answer.** Modern JS engines use the **Mark-and-Sweep** Garbage Collection algorithm. The engine starts at "roots" (the `window` or `globalThis` object) and recursively marks all reachable objects. Any object that is **unreachable** from roots is swept (freed from memory).

#### 4 Classic Causes of Memory Leaks:
1. **Accidental Global Variables**: Assigning to `foo = "data"` without `let`/`const`/`var` attaches it to `window`, keeping it alive forever.
2. **Uncleared Timers & Event Listeners**: `setInterval()` or `window.addEventListener()` holding closures over large objects without calling `clearInterval()` or `removeEventListener()`.
3. **Detached DOM Nodes**: Keeping a reference to a DOM element in a JS variable after `element.remove()` was called on the DOM tree.
4. **Closures**: Outer function scopes retained by long-lived inner functions.

---

### Q40. How does native `structuredClone()` work, and why is it better than `JSON.parse(JSON.stringify())`?

**Answer.** **`structuredClone()`** is a native Web API function (supported in all modern browsers and Node.js 17+) for deep cloning JavaScript values.

#### Why `structuredClone()` beats `JSON.parse(JSON.stringify())`:

| Feature | `structuredClone()` | `JSON.parse(JSON.stringify())` |
| :--- | :--- | :--- |
| **Circular References** | ✅ Supported | ❌ Throws `TypeError: Converting circular structure to JSON` |
| **Dates** | ✅ Preserves `Date` objects | ❌ Converts `Date` to ISO string |
| **Map & Set** | ✅ Preserves `Map` and `Set` | ❌ Converts to empty `{}` |
| **RegExp & Error** | ✅ Preserves `RegExp` | ❌ Converts `RegExp` to `{}` |
| **ArrayBuffer / TypedArrays** | ✅ Preserves binary buffers | ❌ Converts to plain object |
| **Functions / DOM Nodes** | ❌ Throws `DataCloneError` | ❌ Strips functions (`undefined`) |

```js
const original = {
  date: new Date(),
  map: new Map([["key", "value"]]),
};
original.self = original; // Circular reference!

const clone = structuredClone(original);

console.log(clone.date instanceof Date); // true
console.log(clone.map.get("key"));        // "value"
console.log(clone.self === clone);        // true
```

