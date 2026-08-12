# Pure JavaScript Code Output Practice Questions (50 Questions)
---

> **Tip:** Try solving each pure JavaScript code snippet yourself first before expanding the `<details>` dropdown to check the output and explanation!

---

### Q1. What will be the output of the following JavaScript code?

```js
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log("var:", i), 100);
}

for (let j = 0; j < 3; j++) {
  setTimeout(() => console.log("let:", j), 100);
}
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
var: 3
var: 3
var: 3
let: 0
let: 1
let: 2
```

**Explanation:**
- `var` is function-scoped. A single shared variable `i` is bound across all iterations. By the time `setTimeout` callbacks run 100ms later, `i` has reached `3`.
- `let` is block-scoped. Each loop iteration creates a new, separate binding for `j`, preserving `0`, `1`, and `2` inside the closure.

</details>

---

### Q2. What will be the output of the following JavaScript code?

```js
const a = {};
const b = { key: "b" };
const c = { key: "c" };

a[b] = 123;
a[c] = 456;

console.log(a[b]);
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
456
```

**Explanation:**
JavaScript object keys must be strings or Symbols. When non-string keys like `{ key: "b" }` are used, JS calls `.toString()`, converting both `b` and `c` into the literal string `"[object Object]"`.
Therefore, `a[b]` is `a["[object Object]"] = 123`, which gets overwritten by `a[c]` as `a["[object Object]"] = 456`.

</details>

---

### Q3. What will be the output of the following JavaScript code?

```js
const obj = {
  name: "Alice",
  getName: function() {
    return this.name;
  },
  getArrowName: () => {
    return this.name;
  }
};

const fn = obj.getName;

console.log(obj.getName());
console.log(fn());
console.log(obj.getArrowName());
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
Alice
undefined
undefined
```

**Explanation:**
1. `obj.getName()`: Method call; `this` points to `obj`. Returns `"Alice"`.
2. `fn()`: Standalone function call; `this` defaults to `window` / `undefined` (in strict mode). Returns `undefined`.
3. `obj.getArrowName()`: Arrow functions do NOT have their own `this`. They capture `this` lexically from the outer scope (`window`), where `name` is undefined.

</details>

---

### Q4. What will be the output of the following JavaScript code?

```js
console.log("1");

setTimeout(() => console.log("2"), 0);

Promise.resolve().then(() => {
  console.log("3");
}).then(() => {
  console.log("4");
});

queueMicrotask(() => console.log("5"));

console.log("6");
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
1
6
3
5
4
2
```

**Explanation:**
1. Synchronous code executes first (`1`, `6`).
2. `setTimeout` goes to the **Macrotask queue**.
3. `Promise.then` and `queueMicrotask` go to the **Microtask queue**.
4. The Event Loop empties the entire Microtask queue (`3`, `5`, `4`) before processing the Macrotask queue (`2`).

</details>

---

### Q5. What will be the output of the following JavaScript code?

```js
console.log(false == '0');
console.log(false === '0');
console.log([] == ![]);
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
true
false
true
```

**Explanation:**
1. `false == '0'`: Both operands coerce to number: `Number(false) -> 0` and `Number('0') -> 0`. `0 == 0` is `true`.
2. `false === '0'`: Strict equality checks type without coercion (boolean vs string) -> `false`.
3. `[] == ![]`: `![]` evaluates to `false`. `[] == false` converts `[]` to primitive `""` (`Number("") -> 0`) and `false` to `0`. `0 == 0` is `true`.

</details>

---

### Q6. What will be the output of the following JavaScript code?

```js
const result = ["1", "2", "10"].map(parseInt);
console.log(result);
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
[1, NaN, 2]
```

**Explanation:**
Array `.map(callback)` passes 3 arguments to the callback: `(element, index, array)`.
`parseInt` takes 2 arguments: `parseInt(string, radix)`.

- Iteration 1: `parseInt("1", 0)` -> Radix 0 defaults to 10 -> `1`.
- Iteration 2: `parseInt("2", 1)` -> Radix 1 is invalid -> `NaN`.
- Iteration 3: `parseInt("10", 2)` -> Radix 2 (binary) parses `"10"` as `2`.

</details>

---

### Q7. What will be the output of the following JavaScript code?

```js
let x = 10;

function test() {
  console.log(x);
  let x = 20;
}

test();
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
ReferenceError: Cannot access 'x' before initialization
```

**Explanation:**
Inside `test()`, the inner `let x = 20` hoists its variable declaration to the top of the function scope, creating a Temporal Dead Zone (TDZ) for `x` from the start of the function until line `let x = 20`. Attempting to read `console.log(x)` while inside the TDZ throws a `ReferenceError`.

</details>

---

### Q8. What will be the output of the following JavaScript code?

```js
function mutate(obj) {
  obj.age = 30;
  obj = { name: "Bob", age: 40 };
  return obj;
}

const user = { name: "Alice", age: 20 };
const result = mutate(user);

console.log(user.age);
console.log(result.name);
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
30
Bob
```

**Explanation:**
In JS, objects are passed by value-of-reference.
1. `obj.age = 30` mutates the underlying `user` object in place.
2. `obj = { name: "Bob", age: 40 }` reassigns the local parameter reference `obj` to a brand-new object in memory, without changing the caller's `user` variable.

</details>

---

### Q9. What will be the output of the following JavaScript code?

```js
console.log(typeof null);
console.log(typeof NaN);
console.log(typeof ([] + {}));
console.log(typeof (function() {}));
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
object
number
string
function
```

**Explanation:**
1. `typeof null`: Legacy JS bug preserved for backwards compatibility (`"object"`).
2. `typeof NaN`: `"number"` (Not-a-Number is a numeric type).
3. `[] + {}`: Binary `+` converts `[]` to `""` and `{}` to `"[object Object]"`, producing string `"[object Object]"`. `typeof` returns `"string"`.
4. `typeof function`: Returns `"function"` (callable object).

</details>

---

### Q10. What will be the output of the following JavaScript code?

```js
const [a = 1, b = 2, ...c] = [0, undefined, 3, 4, 5];
console.log(a, b, c);
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
0 2 [3, 4, 5]
```

**Explanation:**
1. `a`: Takes `0` from array index 0 (`0` is not `undefined`, so default `1` is ignored).
2. `b`: Array index 1 is `undefined`, so default value `2` is assigned.
3. `c`: Rest operator `...c` collects all remaining array elements `[3, 4, 5]`.

</details>

---

### Q11. What will be the output of the following JavaScript code?

```js
function createCounters() {
  let count = 0;
  return {
    increment() { count++; return count; },
    decrement() { count--; return count; }
  };
}

const c1 = createCounters();
const c2 = createCounters();

console.log(c1.increment());
console.log(c1.increment());
console.log(c2.increment());
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
1
2
1
```

**Explanation:**
Each call to `createCounters()` creates a new lexical environment with its own independent `count` variable. `c1` and `c2` do not share state.

</details>

---

### Q12. What will be the output of the following JavaScript code?

```js
console.log(foo());
// console.log(bar()); // Throws TypeError if un-commented

function foo() { return "Foo"; }
var bar = function() { return "Bar"; };
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
Foo
```

**Explanation:**
Function declarations (`function foo()`) are fully hoisted along with their function body. Function expressions assigned to `var` (`var bar = ...`) hoist only the variable declaration initialized to `undefined`. Calling `bar()` before initialization throws `TypeError: bar is not a function`.

</details>

---

### Q13. What will be the output of the following JavaScript code?

```js
const parent = { count: 1 };
const child = Object.create(parent);

child.count++;
console.log(child.count);
console.log(parent.count);
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
2
1
```

**Explanation:**
`child.count++` evaluates to `child.count = child.count + 1`.
1. It reads `child.count` (found on `parent` prototype as `1`).
2. It assigns `1 + 1 = 2` onto `child` directly (creating an own property on `child` that shadows `parent.count`). `parent.count` remains `1`.

</details>

---

### Q14. What will be the output of the following JavaScript code?

```js
function regular() {
  return () => arguments[0];
}

const arrowFn = regular(10, 20);
console.log(arrowFn(30));
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
10
```

**Explanation:**
Arrow functions do not have their own `arguments` object. The arrow function inside `regular()` captures `arguments` lexically from `regular(10, 20)`, returning `10`.

</details>

---

### Q15. What will be the output of the following JavaScript code?

```js
let a = 0;
let b = 0;

a ||= 10;
b ??= 10;

console.log(a, b);
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
10 0
```

**Explanation:**
- `a ||= 10` executes if `a` is falsy. Since `0` is falsy, `a` becomes `10`.
- `b ??= 10` executes ONLY if `b` is `null` or `undefined`. Since `0` is a valid number (not nullish), `b` remains `0`.

</details>

---

### Q16. What will be the output of the following JavaScript code?

```js
const arr = [1, , 3]; // Sparse array with empty slot at index 1

console.log(arr.length);
console.log(arr.map(x => x * 2));
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
3
[ 2, <1 empty item>, 6 ]
```

**Explanation:**
Sparse array holes are skipped by `.map()`, `.forEach()`, and `.filter()`. The output array preserves the empty slot without invoking the callback for index 1.

</details>

---

### Q17. What will be the output of the following JavaScript code?

```js
console.log(NaN === NaN);
console.log(Object.is(NaN, NaN));
console.log([NaN].indexOf(NaN));
console.log([NaN].includes(NaN));
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
false
true
-1
true
```

**Explanation:**
1. `NaN === NaN` is `false` according to IEEE 754 standards.
2. `Object.is(NaN, NaN)` compares exact values (`true`).
3. `indexOf()` uses strict equality `===`, so it cannot find `NaN` (`-1`).
4. `includes()` uses the `SameValueZero` algorithm, which correctly matches `NaN` (`true`).

</details>

---

### Q18. What will be the output of the following JavaScript code?

```js
function show() {
  console.log(this.name);
}

const obj1 = { name: "First" };
const obj2 = { name: "Second" };

const boundFn = show.bind(obj1).bind(obj2);
boundFn();
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
First
```

**Explanation:**
`bind()` creates a new function with its `this` permanently bound to `obj1`. Subsequent `.bind(obj2)` calls cannot override the initial `this` binding.

</details>

---

### Q19. What will be the output of the following JavaScript code?

```js
function test() {
  return
  {
    status: 200
  };
}

console.log(test());
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
undefined
```

**Explanation:**
JS inserts an automatic semicolon (ASI) after `return` because the newline separates `return` from `{`. The function returns `undefined`, and the object block below is ignored.

</details>

---

### Q20. What will be the output of the following JavaScript code?

```js
const numbers = [10, 5, 40, 25];
numbers.sort();

console.log(numbers);
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
[ 10, 25, 40, 5 ]
```

**Explanation:**
Array `.sort()` without a comparison function converts elements to strings and sorts them lexicographically (alphabetically). `"10"` comes before `"25"`, which comes before `"5"`.

</details>

---

### Q21. What will be the output of the following JavaScript code?

```js
const proto = { inherited: true };
const obj = Object.create(proto);
obj.own = true;

for (let key in obj) console.log(key);
console.log(Object.keys(obj));
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
own
inherited
[ 'own' ]
```

**Explanation:**
- `for...in` iterates over all enumerable properties, including prototype properties (`own`, `inherited`).
- `Object.keys()` returns an array of only **own** enumerable properties (`['own']`).

</details>

---

### Q22. What will be the output of the following JavaScript code?

```js
class Base {
  name = "Base";
  constructor() {
    this.print();
  }
  print() {
    console.log(this.name);
  }
}

class Derived extends Base {
  name = "Derived";
}

new Derived();
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
Base
```

**Explanation:**
1. `new Derived()` invokes `Base` constructor via `super()`.
2. Inside `Base` constructor, `this.print()` runs.
3. At this moment, `Derived` field initializers (`name = "Derived"`) have NOT run yet, so `this.name` outputs `"Base"`.

</details>

---

### Q23. What will be the output of the following JavaScript code?

```js
async function fail() {
  throw new Error("Failed");
}

async function test() {
  try {
    fail();
  } catch (err) {
    console.log("Caught");
  }
}

test();
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
(UnhandledPromiseRejectionWarning: Error: Failed)
```

**Explanation:**
Calling `fail()` returns a rejected Promise. Because `fail()` is NOT preceded by `await`, the `try/catch` block finishes synchronously before the Promise rejects, missing the exception completely.

</details>

---

### Q24. What will be the output of the following JavaScript code?

```js
var a = 1;
b = 2;
const obj = { c: 3 };

delete a;
delete b;
delete obj.c;

console.log(typeof a, typeof b, obj.c);
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
number undefined undefined
```

**Explanation:**
- `delete a`: Fails on declared variables (`var`, `let`, `const`); `a` remains `1` (`typeof a` is `"number"`).
- `delete b`: Succeeds on undeclared global variables (`b = 2`); `b` is deleted (`typeof b` is `"undefined"`).
- `delete obj.c`: Succeeds on configurable object properties; `obj.c` becomes `undefined`.

</details>

---

### Q25. What will be the output of the following JavaScript code?

```js
function* gen() {
  const x = yield 1;
  const y = yield (x + 2);
  return x + y;
}

const g = gen();
console.log(g.next().value);
console.log(g.next(10).value);
console.log(g.next(20).value);
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
1
12
30
```

**Explanation:**
1. `g.next()`: Starts execution, yields `1`.
2. `g.next(10)`: Passes `10` as the evaluation of `yield 1` into `x`. Yields `10 + 2 = 12`.
3. `g.next(20)`: Passes `20` into `y`. Returns `x + y = 10 + 20 = 30`.

</details>

---

### Q26. What will be the output of the following JavaScript code?

```js
function test() {
  x = 2;
  console.log(x);
}

test();
console.log(x);
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
2
2
```

**Explanation:**
In non-strict mode, assigning to an undeclared variable `x = 2` automatically creates a property on the global `window` / `globalThis` object, making `x` globally accessible even outside `test()`.

</details>

---

### Q27. What will be the output of the following JavaScript code?

```js
console.log([1, [2, 3]] + [4, 5]);
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
1,2,34,5
```

**Explanation:**
When the binary `+` operator is used with arrays, JS converts both arrays to strings via `.toString()`:
- `[1, [2, 3]].toString()` becomes `"1,2,3"`
- `[4, 5].toString()` becomes `"4,5"`
String concatenation gives `"1,2,3" + "4,5"` = `"1,2,34,5"`.

</details>

---

### Q28. What will be the output of the following JavaScript code?

```js
let name = "Global";

function greet(name) {
  console.log(name);
}

greet("Local");
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
Local
```

**Explanation:**
Function parameters shadow variables declared in outer scope with the same name. `greet("Local")` binds parameter `name` to `"Local"`.

</details>

---

### Q29. What will be the output of the following JavaScript code?

```js
const obj = {
  valueOf() { return 10; },
  toString() { return "20"; }
};

console.log(obj + 5);
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
15
```

**Explanation:**
When performing numeric addition (`+`), JS first invokes `valueOf()`. Since `valueOf()` returns primitive `10`, JS computes `10 + 5 = 15`. (`toString()` is only called if `valueOf()` returns an object).

</details>

---

### Q30. What will be the output of the following JavaScript code?

```js
const arr = [1, 2, 3];
const res1 = arr.push(4);
const res2 = arr.concat(5);

console.log(res1, res2);
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
4 [ 1, 2, 3, 4, 5 ]
```

**Explanation:**
- `.push(4)` mutates `arr` in place and returns the **new length** of the array (`4`).
- `.concat(5)` does NOT mutate `arr`; it returns a **new array** `[1, 2, 3, 4, 5]`.

</details>

---

### Q31. What will be the output of the following JavaScript code?

```js
const obj = {};
Object.defineProperty(obj, "role", {
  value: "Admin"
});

obj.role = "User";
console.log(obj.role);
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
Admin
```

**Explanation:**
When defining properties via `Object.defineProperty()`, descriptor flags default to `false`. `writable: false` prevents property modification, so `obj.role = "User"` fails silently in non-strict mode.

</details>

---

### Q32. What will be the output of the following JavaScript code?

```js
const set = new Set([1, "1", { a: 1 }, { a: 1 }]);
console.log(set.size);
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
4
```

**Explanation:**
- `1` (number) and `"1"` (string) are distinct primitive types -> 2 entries.
- `{ a: 1 }` and `{ a: 1 }` are two separate object references in memory -> 2 entries.
Total size is `4`.

</details>

---

### Q33. What will be the output of the following JavaScript code?

```js
let a = 1;
let b = a++ + ++a;

console.log(a, b);
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
3 4
```

**Explanation:**
1. `a++` evaluates to `1` (post-increment, then `a` becomes `2`).
2. `++a` increments `a` from `2` to `3` first, then evaluates to `3`.
3. `b = 1 + 3 = 4`. Final `a` is `3`.

</details>

---

### Q34. What will be the output of the following JavaScript code?

```js
console.log(null || undefined || "default");
console.log("hello" && 0 && "world");
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
default
0
```

**Explanation:**
- `||` returns the first truthy operand, or the last operand if all are falsy (`"default"`).
- `&&` returns the first falsy operand, or the last operand if all are truthy (`0`).

</details>

---

### Q35. What will be the output of the following JavaScript code?

```js
const user = { name: "Alice", address: { city: "Paris" } };
const { address: { city: town, country = "France" } } = user;

console.log(town, country);
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
Paris France
```

**Explanation:**
Nested destructuring renames `city` to local variable `town` (`"Paris"`). `country` is missing from `address`, triggering the default value `"France"`.

</details>

---

### Q36. What will be the output of the following JavaScript code?

```js
const f1 = () => { a: 1 };
const f2 = () => ({ a: 1 });

console.log(f1());
console.log(f2());
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
undefined
{ a: 1 }
```

**Explanation:**
- `f1`: The `{ a: 1 }` braces are parsed as a function body block with a statement label `a: 1`. The function returns `undefined`.
- `f2`: Wrapping `({ a: 1 })` in parentheses forces JS to evaluate it as an object literal expression.

</details>

---

### Q37. What will be the output of the following JavaScript code?

```js
(function() {
  var a = b = 3;
})();

console.log(typeof a);
console.log(typeof b);
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
undefined
number
```

**Explanation:**
`var a = b = 3` evaluates right-to-left as `b = 3; var a = b;`.
`b` is assigned without `var`/`let`, creating an implicit global variable (`typeof b` is `"number"`). `a` is function-scoped to the IIFE (`typeof a` is `"undefined"` outside).

</details>

---

### Q38. What will be the output of the following JavaScript code?

```js
const p1 = new Promise((_, reject) => setTimeout(() => reject("Err 1"), 50));
const p2 = new Promise((resolve) => setTimeout(() => resolve("Win 2"), 100));

Promise.race([p1, p2])
  .then(res => console.log("Race:", res))
  .catch(err => console.log("Race Catch:", err));
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
Race Catch: Err 1
```

**Explanation:**
`Promise.race()` settles as soon as the **first** promise settles (whether resolved or rejected). `p1` rejects in 50ms before `p2` resolves in 100ms, triggering `.catch()`.

</details>

---

### Q39. What will be the output of the following JavaScript code?

```js
console.log(-0 === +0);
console.log(Object.is(-0, +0));
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
true
false
```

**Explanation:**
Strict equality `===` treats `-0` and `+0` as equal (`true`). `Object.is(-0, +0)` distinguishes negative zero from positive zero (`false`).

</details>

---

### Q40. What will be the output of the following JavaScript code?

```js
const items = [1, 2];
items.push(3);

console.log(items);

// items = [1, 2, 3]; // Line 1
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
[ 1, 2, 3 ]
```

**Explanation:**
`const` prevents variable re-assignment (`items = [...]` fails). However, `const` does NOT make array contents immutable; mutating methods like `.push()` work completely fine.

</details>

---

### Q41. What will be the output of the following JavaScript code?

```js
function tag(strings, ...values) {
  return strings[0] + values[0].toUpperCase() + strings[1];
}

const name = "alice";
console.log(tag`Hello ${name}!`);
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
Hello ALICE!
```

**Explanation:**
Tagged template literals pass template string fragments to `strings` (`["Hello ", "!"]`) and interpolated expressions to `values` (`["alice"]`).

</details>

---

### Q42. What will be the output of the following JavaScript code?

```js
class Person {
  constructor(name) {
    this.name = name;
  }
}

try {
  Person("Alice");
} catch (err) {
  console.log(err.name);
}
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
TypeError
```

**Explanation:**
ES6 classes cannot be invoked without the `new` operator. Calling `Person("Alice")` throws a `TypeError: Class constructor Person cannot be invoked without 'new'`.

</details>

---

### Q43. What will be the output of the following JavaScript code?

```js
class Parent {
  static greet() { return "Parent Greet"; }
}

class Child extends Parent {}

console.log(Child.greet());
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
Parent Greet
```

**Explanation:**
In JavaScript ES6 classes, static methods are inherited via the class prototype chain (`Child.__proto__ === Parent`).

</details>

---

### Q44. What will be the output of the following JavaScript code?

```js
try {
  const wm = new WeakMap();
  wm.set("key", 123);
} catch (err) {
  console.log(err.name);
}
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
TypeError
```

**Explanation:**
`WeakMap` keys MUST be objects (or non-registered Symbols). Attempting to use primitive string `"key"` as a key throws `TypeError: Invalid value used as weak map key`.

</details>

---

### Q45. What will be the output of the following JavaScript code?

```js
const user = {
  _age: 20,
  get age() {
    return this._age;
  },
  set age(val) {
    if (val < 0) return;
    this._age = val;
  }
};

user.age = -10;
console.log(user.age);
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
20
```

**Explanation:**
The setter `set age(val)` validates input. Since `-10 < 0`, the setter returns early without modifying `_age`.

</details>

---

### Q46. What will be the output of the following JavaScript code?

```js
function sum(...nums) {
  return nums.reduce((acc, curr) => acc + curr, 0);
}

const numbers = [1, 2, 3];
console.log(sum(...numbers));
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
6
```

**Explanation:**
`...numbers` spreads `[1, 2, 3]` into separate arguments passed to `sum(1, 2, 3)`. The rest parameter `...nums` gathers them into array `[1, 2, 3]`, which reduces to `6`.

</details>

---

### Q47. What will be the output of the following JavaScript code?

```js
function test() {
  try {
    return 1;
  } finally {
    return 2;
  }
}

console.log(test());
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
2
```

**Explanation:**
The `finally` block ALWAYS executes before a function returns. A `return` statement inside `finally` overrides any previous `return` statement in `try` or `catch`.

</details>

---

### Q48. What will be the output of the following JavaScript code?

```js
const arr = [10, 20, 30];
const total = arr.reduce((acc, val) => acc + val);

console.log(total);
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
60
```

**Explanation:**
When no initial value is passed to `.reduce()`, the first element (`10`) is used as the initial accumulator `acc`, and iteration starts from index 1 (`20`). `10 + 20 + 30 = 60`.

</details>

---

### Q49. What will be the output of the following JavaScript code?

```js
const s1 = Symbol("id");
const s2 = Symbol("id");
const s3 = Symbol.for("id");
const s4 = Symbol.for("id");

console.log(s1 === s2);
console.log(s3 === s4);
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
false
true
```

**Explanation:**
- `Symbol("id")` creates a guaranteed unique symbol every call (`s1 === s2` is `false`).
- `Symbol.for("id")` searches the global symbol registry and returns the shared symbol (`s3 === s4` is `true`).

</details>

---

### Q50. What will be the output of the following JavaScript code?

```js
const target = { a: 1 };
const source = { b: 2 };
const result = Object.assign(target, source);

console.log(result === target);
console.log(target);
```

<details>
<summary>Click to view Output & Explanation</summary>

**Output:**
```text
true
{ a: 1, b: 2 }
```

**Explanation:**
`Object.assign(target, source)` mutates the `target` object directly and returns the same `target` reference (`result === target`).
