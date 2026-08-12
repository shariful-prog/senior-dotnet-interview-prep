# JavaScript Top 20 Interview Questions (Quick Answers)

> Goal: Fast revision. Each question opens with a one-line definition, then the detail — plus a link to the exact section in the detailed JS notes.
> Scope: **core JavaScript**. TypeScript and Angular are separate subjects — see [Sister Topics](#sister-topics) at the bottom.

## Variables, Types & Coercion

### 1. What is the difference between `var`, `let`, and `const`?

**Definition:** `var` is **function-scoped**. `let` and `const` are **block-scoped** — they only exist inside the `{ }` they're declared in.

**Detail:** `var` is hoisted and initialized to `undefined`, so reading it early gives you `undefined` instead of an error. `let` and `const` are hoisted but stay uninitialized, so reading them early throws. `const` prevents **reassignment**, not mutation — you can still push to a `const` array. Default to `const`, use `let` when you must reassign.

:material-file-document-outline: **Deep dive:** [J1 — Variables & Types](../JS/js-javascript.md#j1--variables--types)

---

### 2. What is hoisting, and what is the temporal dead zone?

**Definition:** **Hoisting** is JavaScript registering declarations before any code runs. The **temporal dead zone (TDZ)** is the gap where a `let`/`const` variable exists but can't be read yet.

**Detail:** Function declarations are hoisted completely, so you can call them before they appear. `var` is hoisted with the value `undefined`. `let` and `const` are hoisted too, but touching them before the declaration line throws `ReferenceError` — that window is the TDZ, and it exists to catch exactly this mistake.

:material-file-document-outline: **Deep dive:** [J1 — Variables & Types](../JS/js-javascript.md#j1--variables--types)

---

### 3. What is the difference between `==` and `===`?

**Definition:** `===` compares **type and value** with no conversion. `==` converts the operands to a common type first, then compares.

**Detail:** That conversion produces the famous surprises: `'' == 0`, `'1' == 1`, and `null == undefined` are all `true`. Always use `===`. The one accepted exception is `x == null`, which is a neat way to check for both `null` and `undefined` at once.

:material-file-document-outline: **Deep dive:** [J1 — Variables & Types](../JS/js-javascript.md#j1--variables--types)

---

### 4. What is the difference between `null` and `undefined`?

**Definition:** **`undefined`** means a value was never assigned. **`null`** means "deliberately empty" — you assigned it on purpose.

**Detail:** JavaScript gives you `undefined` automatically for missing variables, missing parameters, and missing object properties. You only ever get `null` because someone wrote it. Quirk worth knowing: `typeof null` returns `'object'`, which is a long-standing bug in the language kept for backwards compatibility.

:material-file-document-outline: **Deep dive:** [J5 — Common Interview Traps](../JS/js-javascript.md#j5--common-interview-traps)

## Functions & Scope

### 5. What is a closure?

**Definition:** A **closure** is a function that keeps access to the variables from the scope where it was created, even after that outer function has finished running.

**Detail:** This is how you get private state, function factories, and the module pattern. It's also behind the classic loop bug: a closure captures the **variable**, not a copy of its value, so `var` in a loop gives every callback the same final value while `let` creates a fresh binding each iteration.

:material-file-document-outline: **Deep dive:** [J2 — Functions & Scope](../JS/js-javascript.md#j2--functions--scope)

---

### 6. How does `this` work, and how do arrow functions change it?

**Definition:** In a normal function, **`this` is decided when the function is called**, not where it's written. An **arrow function** has no `this` of its own — it inherits it from the surrounding code.

**Detail:** For a normal function, `this` is the object before the dot, or the new instance when called with `new`, or whatever you pass to `call`/`apply`/`bind`, or `undefined`/global otherwise. That's why passing a method as a callback loses `this` — and why arrow functions fixed a whole category of bugs.

:material-file-document-outline: **Deep dive:** [J2 — Functions & Scope](../JS/js-javascript.md#j2--functions--scope)

---

### 7. What do `call`, `apply`, and `bind` do?

**Definition:** All three set `this` explicitly. **`call`** invokes immediately with comma-separated arguments. **`apply`** invokes immediately with an array. **`bind`** invokes nothing — it returns a new function with `this` permanently attached.

**Detail:** The mnemonic: **C**all = **C**ommas, **A**pply = **A**rray, **B**ind = **B**ound for later. `bind` is the one you reach for when passing a method as a callback.

:material-file-document-outline: **Deep dive:** [J2 — Functions & Scope](../JS/js-javascript.md#j2--functions--scope)

---

### 8. What is currying?

**Definition:** **Currying** turns a function that takes several arguments into a chain of functions that each take one — `f(a, b, c)` becomes `f(a)(b)(c)`.

**Detail:** A generic `curry()` collects arguments until it has as many as the original function declares (`fn.length`), then calls through. It's useful for building specialized functions out of general ones — `const double = multiply(2)`.

:material-file-document-outline: **Deep dive:** [J7 — Browser & Practical](../JS/js-javascript.md#j7--browser--practical)

## Asynchronous JavaScript

### 9. What is the event loop?

**Definition:** The **event loop** is what lets single-threaded JavaScript handle asynchronous work. It moves queued callbacks onto the call stack, but only when the stack is empty.

**Detail:** Your code runs on one thread with one call stack. Async operations are handed to the browser or Node, and when they finish their callbacks join a queue. The loop waits for the stack to clear before running the next one — which is why a long synchronous loop freezes everything, including rendering.

:material-file-document-outline: **Deep dive:** [J3 — Asynchronous JavaScript](../JS/js-javascript.md#j3--asynchronous-javascript)

---

### 10. What is the difference between microtasks and macrotasks?

**Definition:** **Microtasks** are promise callbacks and `queueMicrotask`. **Macrotasks** are `setTimeout`, `setInterval`, and I/O events. Microtasks have priority.

**Detail:** After each macrotask the event loop **drains the entire microtask queue** before picking up the next macrotask. So a `Promise.then` queued at the same moment as a `setTimeout(..., 0)` always runs first. This is the single most common "what does this print?" question.

:material-file-document-outline: **Deep dive:** [J7 — Browser & Practical](../JS/js-javascript.md#j7--browser--practical)

---

### 11. How do promises and `async`/`await` relate?

**Definition:** A **promise** represents a value that will exist later. **`async`/`await`** is syntax that lets you write promise-based code as if it were sequential.

**Detail:** `await` pauses the function until the promise settles, and `try`/`catch` handles rejection. Watch out for accidental serialization: awaiting inside a loop runs the calls one after another. To run them at the same time, start them all first and then `await Promise.all([...])`. Use `Promise.allSettled` when you want every result even if some fail.

:material-file-document-outline: **Deep dive:** [J3 — Asynchronous JavaScript](../JS/js-javascript.md#j3--asynchronous-javascript)

---

### 12. What is the difference between debouncing and throttling?

**Definition:** **Debounce** waits until the activity stops, then fires once. **Throttle** fires at most once per interval while activity continues.

**Detail:** Debounce resets its timer on every call, so nothing happens until things go quiet — perfect for a search-as-you-type box, where you only want to call the API after the user pauses. Throttle guarantees a steady rate regardless of how fast events arrive — right for scroll and resize handlers.

:material-file-document-outline: **Deep dive:** [J7 — Browser & Practical](../JS/js-javascript.md#j7--browser--practical)

## Objects & Prototypes

### 13. What is the prototype chain?

**Definition:** Every object has a hidden link to another object called its **prototype**. When you read a property, JavaScript walks up that chain until it finds the property or reaches `null`.

**Detail:** This is JavaScript's inheritance model. It's why every array can call `.map` without owning it — `.map` lives on `Array.prototype`. The `class` keyword is syntactic sugar over exactly this mechanism, not a separate system.

:material-file-document-outline: **Deep dive:** [J4 — Objects & Prototypes](../JS/js-javascript.md#j4--objects--prototypes)

---

### 14. What is the difference between a shallow copy and a deep copy?

**Definition:** A **shallow copy** duplicates only the top level — nested objects are still shared. A **deep copy** duplicates everything, all the way down.

**Detail:** Spread (`{...obj}`) and `Object.assign` are shallow, so mutating a nested object affects both copies. For a real deep copy use `structuredClone()`, which also handles `Date`, `Map`, `Set`, and circular references — all things `JSON.parse(JSON.stringify(x))` either loses or throws on.

:material-file-document-outline: **Deep dive:** [J4 — Objects & Prototypes](../JS/js-javascript.md#j4--objects--prototypes)

---

### 15. When should you use `Map` over an object, and `Set` over an array?

**Definition:** A **`Map`** is a key-value collection that accepts any type as a key. A **`Set`** is a collection of unique values.

**Detail:** Use `Map` for real dictionaries with dynamic keys — it keeps insertion order, has a genuine `.size`, allows object keys, and has no inherited prototype keys to collide with. Use `Set` when you need fast membership checks or automatic de-duplication; both are O(1) where an array scan is O(n).

:material-file-document-outline: **Deep dive:** [J6 — ES6+ Features](../JS/js-javascript.md#j6--es6-features)

## Common Traps

### 16. When do you use `map`, `forEach`, `filter`, and `reduce`?

**Definition:** **`map`** transforms each item into a new array of the same length. **`filter`** selects a subset. **`reduce`** folds everything into a single value. **`forEach`** returns nothing and exists only for side effects.

**Detail:** All four leave the original array untouched. `reduce`'s result doesn't have to be a number — it's often an object or a grouped array. A good tell: if you're pushing into an array inside a `forEach`, you actually wanted `map` or `filter`.

:material-file-document-outline: **Deep dive:** [J5 — Common Interview Traps](../JS/js-javascript.md#j5--common-interview-traps)

---

### 17. Why does `[10, 9, 1].sort()` give the wrong order?

**Definition:** The default `sort()` converts every element to a **string** and compares them alphabetically.

**Detail:** So `10` comes before `9`, because `"10"` sorts before `"9"`. Pass a comparator to sort numerically: `.sort((a, b) => a - b)`. Also worth mentioning: `sort` **mutates** the original array — use `toSorted()` or copy first if that matters.

:material-file-document-outline: **Deep dive:** [J5 — Common Interview Traps](../JS/js-javascript.md#j5--common-interview-traps)

---

### 18. What is the difference between `slice` and `splice`?

**Definition:** **`slice`** returns a copy of part of an array and leaves the original alone. **`splice`** changes the original array in place.

**Detail:** `slice(start, end)` is safe and non-destructive. `splice(start, deleteCount, ...items)` removes and/or inserts items, and returns whatever it removed. One letter apart, opposite behaviour — a genuinely common source of bugs.

:material-file-document-outline: **Deep dive:** [J5 — Common Interview Traps](../JS/js-javascript.md#j5--common-interview-traps)

---

### 19. What is the difference between `for...in` and `for...of`?

**Definition:** **`for...in`** iterates over **keys**, including inherited ones. **`for...of`** iterates over **values** of anything iterable.

**Detail:** Avoid `for...in` on arrays — you get index strings rather than numbers, plus anything added to the prototype. Use `for...of` for arrays, strings, `Map`, and `Set`; use `Object.entries()` when you need an object's keys and values together.

:material-file-document-outline: **Deep dive:** [J5 — Common Interview Traps](../JS/js-javascript.md#j5--common-interview-traps)

---

### 20. What causes memory leaks in JavaScript?

**Definition:** A **memory leak** is memory that's no longer needed but is still reachable, so the garbage collector won't free it.

**Detail:** The GC uses mark-and-sweep: anything reachable from the roots survives. Leaks come from things that stay reachable by accident — timers you never clear, event listeners you never remove, detached DOM nodes still held in a variable, and caches or globals that only ever grow. `WeakMap` and `WeakSet` hold their keys weakly, so entries disappear once nothing else refers to the key.

:material-file-document-outline: **Deep dive:** [J7 — Browser & Practical](../JS/js-javascript.md#j7--browser--practical)

---

## Runners-Up (ask-me-next round)

- **`typeof` and reliable type checking** — [J1](../JS/js-javascript.md#j1--variables--types)
- **Default, rest & spread parameters** — [J2](../JS/js-javascript.md#j2--functions--scope)
- **Destructuring; `Object.keys`/`values`/`entries`** — [J4](../JS/js-javascript.md#j4--objects--prototypes)
- **Truthy & falsy values; event delegation** — [J5](../JS/js-javascript.md#j5--common-interview-traps)
- **Optional chaining `?.` and nullish coalescing `??`** — [J6](../JS/js-javascript.md#j6--es6-features)
- **Template literals; named vs default exports** — [J6](../JS/js-javascript.md#j6--es6-features)
- **`localStorage` vs `sessionStorage` vs cookies** — [J7](../JS/js-javascript.md#j7--browser--practical)
- **Generators & the iterator protocol** — [J7](../JS/js-javascript.md#j7--browser--practical)
- **`Symbol`; `Object.freeze` vs `seal` vs `preventExtensions`** — [J7](../JS/js-javascript.md#j7--browser--practical)
- **`MutationObserver` / `ResizeObserver` / `IntersectionObserver`** — [J7](../JS/js-javascript.md#j7--browser--practical)

## Code-Output Practice

Interviewers love "what does this print?" — especially hoisting, closures in loops, `this` binding, and microtask vs macrotask ordering.
**50 worked examples:** [Code Output Practice Questions](../JS/js-code-output-questions.md)

## Sister Topics

- **TypeScript** — [K1 Fundamentals](../JS/js-typescript.md#k1--fundamentals) · [K2 Interfaces & Types](../JS/js-typescript.md#k2--interfaces--types) · [K3 Functions & Generics](../JS/js-typescript.md#k3--functions--generics) · [K4 Narrowing & Safety](../JS/js-typescript.md#k4--narrowing--safety) · [K5 Classes](../JS/js-typescript.md#k5--classes--practical) · [K6 Advanced Types](../JS/js-typescript.md#k6--advanced-type-system-features)
- **Angular** — [L1 Components](../JS/ng-angular.md#l1--components--templates) · [L2 Change Detection](../JS/ng-angular.md#l2--lifecycle--change-detection) · [L3 Services & DI](../JS/ng-angular.md#l3--services--dependency-injection) · [L4 RxJS](../JS/ng-angular.md#l4--rxjs) · [L5 Routing & Forms](../JS/ng-angular.md#l5--routing--forms) · [L6 Performance](../JS/ng-angular.md#l6--practical--performance)
- **Angular cheat sheet** — [Quick Overview](../JS/ng-quick-overview.md)
