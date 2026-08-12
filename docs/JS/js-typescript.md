# K. TypeScript Interview Questions
---

> Core JavaScript questions — closures, `this`, the event loop, prototypes — are in [js-javascript.md](js-javascript.md). This file covers what TypeScript adds on top.

## K1 — Fundamentals

### Q1. What is TypeScript, and what does it add to JavaScript?

**Answer.** TypeScript is JavaScript with **static types**. You annotate your code with types, a compiler checks them, and then it strips them out — the output is plain JavaScript.

```ts
function add(a: number, b: number): number {
  return a + b;
}

add(1, 2);       // ✅
add(1, "2");     // ❌ compile error — never reaches the browser
```

The key point: **all checking happens at compile time.** There are no types at runtime. The generated JavaScript for `add` above has no type information in it at all.

What you get:

- **Errors before running** — typos, wrong argument types, missing properties.
- **Editor support** — autocomplete and refactoring work because the tool knows the shapes.
- **Documentation that cannot go stale** — the signature is checked.

TypeScript is a **superset**: every valid JavaScript file is valid TypeScript. You can rename `.js` to `.ts` and add types gradually.

---

### Q2. What are the basic types?

**Answer.** A **type annotation** declares what kind of value a variable, parameter, or return value holds. You write it after a colon, and the compiler checks every use against it.

TypeScript's types fall into two groups: those mirroring JavaScript's own types, and a few that exist only in the type system and disappear at compile time.

**The JavaScript ones:**

```ts
let name: string = "John";
let age: number = 30;
let active: boolean = true;
let ids: number[] = [1, 2, 3];
let pair: [string, number] = ["a", 1];      // tuple — fixed length and order
```

The TypeScript-only ones matter more in interviews:

| Type | Meaning |
|---|---|
| `any` | turns checking off for that value |
| `unknown` | a value of unknown type — must be narrowed before use |
| `void` | a function returns nothing |
| `never` | a function never returns, or a value that cannot exist |
| `object` | any non-primitive |

```ts
function fail(msg: string): never {       // always throws
  throw new Error(msg);
}

function log(msg: string): void { }       // returns nothing
```

**Type inference** means you usually do not write annotations. TypeScript works it out:

```ts
let count = 5;          // inferred as number
count = "five";         // ❌ error
```

Annotate function parameters and return types; let inference handle local variables.

---

### Q3. `any` vs `unknown` vs `never` — what is the difference?

**Answer.** Three types that all relate to "I do not know what this is", with very different levels of safety.

- **`any`** — switches type checking **off** for that value. Anything is permitted.
- **`unknown`** — the safe counterpart. Holds any value, but you must **check** its type before using it.
- **`never`** — a value that can **never occur**. The return type of a function that always throws or loops forever.

With **`any`**, mistakes pass silently — the compiler stops looking:

```ts
let a: any = "hello";
a.toFixed(2);          // ✅ compiles, ❌ crashes at runtime
```

With **`unknown`**, the same mistake is caught:

```ts
let u: unknown = "hello";
u.toFixed(2);                    // ❌ compile error — good
if (typeof u === "number") {
  u.toFixed(2);                  // ✅ narrowed to number
}
```

**`never`** also appears when you have narrowed away every possibility, which is how exhaustiveness checks work:

```ts
function assertNever(x: never): never {
  throw new Error("Unexpected: " + x);
}
```

| | Assign anything to it | Use it without checking |
|---|---|---|
| `any` | yes | yes — unsafe |
| `unknown` | yes | no — must narrow first |
| `never` | no | — |

**Use `unknown` instead of `any`** for values from outside your code — API responses, `JSON.parse`, `catch` blocks. It forces you to validate.

---

## K2 — Interfaces & Types

### Q4. `interface` vs `type` — when do you use each?

**Answer.** Both name a type so you can reuse it.

- **`interface`** — declares the shape of an **object**. Can be extended and re-opened.
- **`type`** (a type alias) — gives a name to **any** type: an object, a union, a primitive, a tuple, or a function signature.

For plain object shapes they are almost interchangeable, which is why the question is asked.

```ts
interface User {
  name: string;
  age: number;
}

type User2 = {
  name: string;
  age: number;
};
```

Two real differences:

**1. `interface` supports declaration merging.** Declaring it twice adds to it. `type` throws a duplicate error:

```ts
interface Window { title: string; }
interface Window { version: number; }    // ✅ merged — Window now has both

type A = { x: number };
type A = { y: number };                  // ❌ Duplicate identifier
```

**2. `type` can describe things that are not objects.** Unions, primitives, tuples, and mapped types need `type`:

```ts
type Status = "open" | "closed";         // union — interface cannot do this
type ID = string | number;
type Pair = [number, number];
type Callback = (x: number) => void;
```

The practical rule: **`interface` for object shapes you might extend, `type` for everything else.** Declaration merging is what lets you add to types from a library you do not control.

---

### Q5. What are union and intersection types?

**Answer.** Two ways of combining types.

- **Union (`|`)** — the value may be **any one** of the listed types. Written `A | B`.
- **Intersection (`&`)** — the value must satisfy **all** of the listed types at once. Written `A & B`.

A union widens what is allowed; an intersection narrows it by demanding more.

```ts
type Status = "loading" | "success" | "error";     // union of literals
type ID = string | number;

let s: Status = "loading";     // ✅
let s2: Status = "done";       // ❌ not in the union
```

With a union you can only use members common to **every** option, until you narrow:

```ts
function print(x: string | number) {
  x.toUpperCase();                    // ❌ number has no toUpperCase
  if (typeof x === "string") {
    x.toUpperCase();                  // ✅ narrowed
  }
}
```

**Intersection** combines shapes — the result must satisfy all of them:

```ts
type Person = { name: string };
type Employee = { salary: number };

type Staff = Person & Employee;

const s: Staff = { name: "John", salary: 100 };    // both required
```

Unions are far more common. Intersections are mainly used to compose small shapes, or to add properties to an existing type.

---

### Q6. What are literal types and `as const`?

**Answer.** A **literal type** is a type whose only allowed value is one specific value:

```ts
let status: "active" = "active";
status = "inactive";                 // ❌ only "active" is allowed
```

On their own they are not useful; combined into a union they replace enums:

```ts
type Role = "admin" | "user" | "guest";

function setRole(r: Role) { }
setRole("admin");      // ✅ autocompletes the three options
setRole("root");       // ❌ caught at compile time
```

**`as const`** tells TypeScript to infer the narrowest possible type, and to make everything readonly:

```ts
const a = { role: "admin" };              // role: string  — widened
const b = { role: "admin" } as const;     // role: "admin" — literal, readonly
```

This is the common fix when passing an object to something expecting literal types:

```ts
const config = { method: "GET" };
fetch(url, config);                       // ❌ string is not "GET" | "POST" | ...

const config2 = { method: "GET" } as const;
fetch(url, config2);                      // ✅
```

`as const` on an array gives a readonly tuple, which is how you derive a union from a list:

```ts
const roles = ["admin", "user"] as const;
type Role2 = typeof roles[number];        // "admin" | "user"
```

---

### Q7. What are optional, readonly, and index-signature properties?

**Answer.** Three ways to modify how a property behaves in an object type.

- **Optional (`?`)** — the property may be absent
- **`readonly`** — the property cannot be reassigned after the object is created
- **Index signature** — allows keys that are not known in advance

**Optional (`?`)** — the property may be absent. Its type becomes `T | undefined`:

```ts
interface User {
  name: string;
  email?: string;              // may be missing
}

const u: User = { name: "John" };        // ✅
u.email.toLowerCase();                    // ❌ possibly undefined
u.email?.toLowerCase();                   // ✅
```

**`readonly`** — the property cannot be reassigned after creation. Compile-time only; nothing stops it at runtime:

```ts
interface Config {
  readonly apiUrl: string;
}

config.apiUrl = "x";          // ❌ error
```

**Index signature** — describes objects with keys you do not know in advance:

```ts
interface Scores {
  [name: string]: number;      // any string key, number value
}

const s: Scores = { john: 5, jane: 9 };
s.anyone = 3;                  // ✅
```

❌ An index signature weakens checking — typos are no longer errors, since any key is valid. Prefer `Record<K, V>` with a known key type where possible:

```ts
type Scores2 = Record<"john" | "jane", number>;   // ✅ only these two keys
```

---

## K3 — Functions & Generics

### Q8. What are generics, and why do you need them?

**Answer.** A **generic** is a type parameter — it lets a function or type work with many types while keeping the relationship between them.

Without generics you lose type information:

```ts
function firstAny(arr: any[]): any { return arr[0]; }

const n = firstAny([1, 2]);      // n is any — checking is gone
```

With a generic, the type flows through:

```ts
function first<T>(arr: T[]): T {
  return arr[0];
}

const n = first([1, 2]);         // number ✅
const s = first(["a", "b"]);     // string ✅
```

`T` is filled in at each call site, usually inferred so you never write it.

**Constraints** limit what `T` can be, so you can use its members:

```ts
function longest<T extends { length: number }>(a: T, b: T): T {
  return a.length >= b.length ? a : b;      // ✅ length is guaranteed
}

longest("ab", "c");          // ✅ strings have length
longest(1, 2);               // ❌ numbers do not
```

Generics also apply to interfaces and classes:

```ts
interface ApiResponse<T> {
  data: T;
  status: number;
}

const r: ApiResponse<User> = { data: user, status: 200 };
```

The rule: **use a generic when two things must be the same type**, and `unknown` when you genuinely do not care.

---

### Q9. What are function overloads?

**Answer.** **Function overloads** let one function have several type signatures. You declare each accepted call form, then write a single implementation that handles them all.

They exist for the case where the **return type depends on the argument type** — something a single signature cannot express.

```ts
function parse(x: string): string[];
function parse(x: number): number[];
function parse(x: any): any[] {          // implementation — not callable
  return typeof x === "string" ? x.split("") : [x];
}

const a = parse("ab");     // string[] ✅
const b = parse(2);        // number[] ✅
```

Callers see only the first two lines. The implementation signature must be broad enough to cover them all, but is not itself an option.

A union return type would be worse, because every caller would then have to narrow:

```ts
function parse2(x: string | number): string[] | number[] { }
const c = parse2("ab");    // string[] | number[] ❌ caller must check
```

Prefer a **union parameter** when the return type does not change, and overloads only when it does.

---

### Q10. What are the built-in utility types?

**Answer.** **Utility types** are generic types built into TypeScript that take an existing type and produce a modified version of it. They let you derive related types from one definition instead of writing each by hand.

They are purely compile-time: `Partial<User>` creates no code, it just describes a shape.

```ts
interface User {
  id: number;
  name: string;
  email: string;
}
```

| Utility | Result |
|---|---|
| `Partial<User>` | all properties optional |
| `Required<User>` | all properties required |
| `Readonly<User>` | all properties readonly |
| `Pick<User, "id" \| "name">` | only those properties |
| `Omit<User, "email">` | everything except those |
| `Record<K, V>` | an object type with keys `K`, values `V` |
| `ReturnType<typeof fn>` | what a function returns |
| `Awaited<T>` | the type inside a promise |

The common ones in practice:

```ts
function update(id: number, changes: Partial<User>) { }    // ✅ any subset
update(1, { name: "Jane" });

type UserPreview = Pick<User, "id" | "name">;              // ✅ list responses
type NewUser = Omit<User, "id">;                           // ✅ before insert
type UsersById = Record<number, User>;                     // ✅ lookup map
```

These keep one source of truth. Add a field to `User` and every derived type updates with it.

---

## K4 — Narrowing & Safety

### Q11. What is type narrowing?

**Answer.** **Narrowing** is TypeScript reducing a broad type to a specific one based on checks you have written. It happens automatically.

Four common forms:

```ts
// typeof — for primitives
if (typeof x === "string") { x.toUpperCase(); }

// instanceof — for classes
if (err instanceof Error) { err.message; }

// in — for optional properties
if ("email" in user) { user.email; }

// truthiness — removes null and undefined
if (user) { user.name; }
```

**Discriminated unions** are the important pattern. Give each option a shared literal field, and TypeScript narrows on it:

```ts
type Result =
  | { status: "success"; data: string }
  | { status: "error"; message: string };

function handle(r: Result) {
  if (r.status === "success") {
    r.data;         // ✅ available
    r.message;      // ❌ does not exist here
  } else {
    r.message;      // ✅
  }
}
```

This is how you model API results, form states, and Redux-style actions. A `switch` on the discriminant gives you exhaustiveness checking — add a new case to the union and any unhandled `switch` becomes an error.

---

### Q12. What is a type guard, and what does `is` do?

**Answer.** A **type guard** is a function returning a special boolean that tells TypeScript what a value is. The return type uses `x is T`:

```ts
function isString(x: unknown): x is string {
  return typeof x === "string";
}

function run(v: unknown) {
  if (isString(v)) {
    v.toUpperCase();         // ✅ narrowed to string
  }
}
```

Without `is` the function returns a plain `boolean`, and no narrowing happens:

```ts
function isString2(x: unknown): boolean {
  return typeof x === "string";
}

if (isString2(v)) { v.toUpperCase(); }     // ❌ v is still unknown
```

This is essential for validating data from outside your program, where the compiler cannot help you:

```ts
function isUser(x: unknown): x is User {
  return typeof x === "object" && x !== null && "id" in x && "name" in x;
}

const data: unknown = await res.json();
if (isUser(data)) { data.name; }           // ✅ safe
```

❌ TypeScript **trusts** your guard. If the logic is wrong, it narrows to the wrong type and you get a runtime error. A guard is a promise you must keep.

---

### Q13. What does `strictNullChecks` do?

**Answer.** With it on, `null` and `undefined` are **not** assignable to other types. You must handle them explicitly.

```ts
// strictNullChecks: false — the old, unsafe behaviour
let name: string = null;        // ✅ allowed

// strictNullChecks: true
let name: string = null;        // ❌ error
let name2: string | null = null;   // ✅ must be explicit
```

The value is that possible absence becomes visible in the type, and the compiler forces a check:

```ts
function find(id: number): User | undefined { }

const u = find(1);
u.name;            // ❌ possibly undefined
u?.name;           // ✅
if (u) u.name;     // ✅
```

This is part of the `strict` flag, which you should always enable. It is the single setting that catches the most bugs — the whole class of "cannot read property of undefined".

Related strict options: `noImplicitAny` (a parameter with no type is an error) and `strictFunctionTypes`.

---

### Q14. What is the non-null assertion (`!`), and when is it dangerous?

**Answer.** `!` tells the compiler "this is not null or undefined, trust me". It removes `null` and `undefined` from the type without any check:

```ts
const el = document.getElementById("app");    // HTMLElement | null
el.innerHTML = "hi";                          // ❌ possibly null
el!.innerHTML = "hi";                         // ✅ compiles
```

It is **not** a runtime check. Nothing is verified — if the value really is null, you get the same crash you would have had in plain JavaScript.

```ts
const u = users.find(x => x.id === 99)!;      // ❌ crashes if not found
u.name;
```

Prefer a real check, which costs one line and cannot lie:

```ts
const u = users.find(x => x.id === 99);
if (!u) throw new Error("not found");
u.name;                                        // ✅ narrowed properly
```

Use `!` only where you can see the invariant holds and the compiler cannot — for example, an element you created moments earlier. Treat it the same way as `any`: a deliberate escape hatch, not a convenience.

---

## K5 — Classes & Practical

### Q15. What do the access modifiers do, and what is a parameter property?

**Answer.** **Access modifiers** control where a class member can be read or written from. TypeScript has three, and they are checked by the compiler only.

A **parameter property** is a shorthand: putting a modifier on a constructor parameter declares the field and assigns it in one step.

| Modifier | Accessible from |
|---|---|
| `public` (default) | anywhere |
| `protected` | the class and its subclasses |
| `private` | the class only |

```ts
class Account {
  public id: number;
  protected balance: number;
  private pin: string;
}
```

These are **compile-time only**. Nothing prevents access at runtime, and the emitted JavaScript has no protection. For real privacy, use JavaScript's `#` fields:

```ts
class Account2 {
  #pin = "1234";       // ✅ genuinely private at runtime
}
```

**Parameter properties** are shorthand: put a modifier on a constructor parameter and TypeScript declares and assigns the field for you.

```ts
// verbose
class Service {
  private repo: Repository;
  constructor(repo: Repository) {
    this.repo = repo;
  }
}

// same thing
class Service2 {
  constructor(private repo: Repository) { }
}
```

This is used constantly with dependency injection, since it removes the repeated field declaration and assignment.

---

### Q16. `enum` vs union of literals — which should you use?

**Answer.** Both give a fixed set of allowed values. They differ in what reaches the runtime.

```ts
enum Status { Open, Closed }          // generates real JavaScript
type Status2 = "open" | "closed";     // erased completely
```

An `enum` emits an object into your output:

```js
// compiled from enum Status
var Status;
(function (Status) {
  Status[Status["Open"] = 0] = "Open";
  Status[Status["Closed"] = 1] = "Closed";
})(Status || (Status = {}));
```

Numeric enums also accept values outside the set, which defeats the purpose:

```ts
let s: Status = 99;        // ✅ compiles — no error
```

**Prefer a union of string literals.** It has zero runtime cost, cannot hold an invalid value, and the strings are readable when logged or stored:

```ts
type Status3 = "open" | "closed";
const s: Status3 = "open";
```

Where you need the list at runtime, use `as const` (Q6):

```ts
const STATUSES = ["open", "closed"] as const;
type Status4 = typeof STATUSES[number];      // "open" | "closed"
```

Use `const enum` or a real `enum` only when you specifically need numeric values, such as matching a database column.

---

### Q17. What is the difference between type assertion and type conversion?

**Answer.** An **assertion** (`as`) changes what the compiler believes. It performs **no conversion** and generates no code:

```ts
const x = "5" as unknown as number;    // ❌ still the string "5" at runtime
x.toFixed(2);                          // ❌ crashes
```

Compare that with a real conversion, which runs at runtime:

```ts
const y = Number("5");                 // ✅ actually a number
```

Assertions are for when you know more than the compiler — typically DOM queries and parsed JSON:

```ts
const input = document.querySelector("#name") as HTMLInputElement;
input.value;          // ✅ Element has no .value, HTMLInputElement does
```

❌ Two things to know:

- **`as` can lie.** `data as User` on an arbitrary API response asserts a shape nobody verified. Use a type guard (Q12) when the data comes from outside.
- **Double assertion (`as unknown as T`)** bypasses the compiler's own safety check on assertions. If you need it, the types are usually wrong.

`satisfies` is the safer modern alternative when you want checking *and* narrow inference:

```ts
const config = { port: 3000 } satisfies Config;    // ✅ checked, stays literal
```

---

### Q18. How do you type an API response safely?

**Answer.** The problem: `res.json()` returns `any` (or `unknown`), so nothing about the shape is verified.

```ts
const data = await res.json();      // any — no safety at all
data.usr.name;                      // ✅ compiles, ❌ crashes
```

❌ **Asserting is the common mistake.** It silences the compiler without checking anything:

```ts
const user = await res.json() as User;    // ❌ a claim, not a check
```

Three levels of safety:

**1. Assert and accept the risk** — fine for an internal API you control.

**2. Validate with a type guard** (Q12) — no dependencies:

```ts
function isUser(x: unknown): x is User {
  return typeof x === "object" && x !== null
    && typeof (x as User).id === "number";
}

const data: unknown = await res.json();
if (!isUser(data)) throw new Error("bad response");
data.id;                                   // ✅ verified
```

**3. Validate with a schema library** — the production answer. One schema gives you both the runtime check and the type:

```ts
import { z } from "zod";

const UserSchema = z.object({ id: z.number(), name: z.string() });
type User = z.infer<typeof UserSchema>;    // ✅ type derived from the schema

const user = UserSchema.parse(await res.json());   // throws if wrong
```

The principle: **types describe what you expect; only runtime validation proves it.** At any boundary — HTTP, `localStorage`, user input, environment variables — you need both.

---

## K6 — Advanced Type System Features

### Q19. What are Mapped Types and Indexed Access Types (`in keyof`, `T[K]`)?

**Answer.** **Indexed Access Types** (`T[K]`) look up the type of a specific property on another type. **Mapped Types** iterate over keys (using `in keyof`) to construct a new type dynamically.

```ts
type User = {
  id: number;
  name: string;
  age: number;
};

// 1. Indexed Access Type: Get the type of 'name' property
type NameType = User["name"]; // string

// 2. Mapped Type: Create a type where all properties of T become boolean flags
type OptionsFlags<T> = {
  [K in keyof T]: boolean;
};

type UserFlags = OptionsFlags<User>;
// Result: { id: boolean; name: boolean; age: boolean; }
```

**Modifiers (`+` / `-`, `readonly`, `?`):**
You can add or remove `readonly` or optional modifiers during mapping:

```ts
// Custom Readonly implementation
type CustomReadonly<T> = {
  +readonly [K in keyof T]: T[K];
};

// Remove optional modifier (-?) to make all properties required
type Concrete<T> = {
  [K in keyof T]-?: T[K];
};
```

---

### Q20. What are Conditional Types and the `infer` keyword (`T extends U ? X : Y`)?

**Answer.** **Conditional Types** introduce branching logic (`condition ? trueType : falseType`) at the type level.

```ts
type IsString<T> = T extends string ? true : false;

type A = IsString<string>; // true
type B = IsString<number>; // false
```

#### The `infer` Keyword (Type Extraction)
The `infer` keyword allows you to **declare a temporary type variable** inside the `extends` clause of a conditional type to extract inner types (such as Promise values, array elements, or function return types).

```ts
// Extract the resolved type of a Promise (UnwrapPromise)
type UnwrapPromise<T> = T extends Promise<infer U> ? U : T;

type R1 = UnwrapPromise<Promise<string>>; // string
type R2 = UnwrapPromise<number>;          // number

// Extract the return type of a function (custom ReturnType<T>)
type MyReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

type Fn = (a: number) => boolean;
type FnReturn = MyReturnType<Fn>; // boolean
```

---

### Q21. What are Template Literal Types in TypeScript, and how are they used?

**Answer.** **Template Literal Types** (introduced in TS 4.1) build on string literal types and allow constructing types using template literal string syntax.

```ts
type World = "world";
type Greeting = `hello ${World}`; // "hello world"

// Generating all combinations of Event Names
type EventType = "click" | "hover";
type ElementType = "button" | "link";

type EventHandlerName = `on${Capitalize<EventType>}${Capitalize<ElementType>}`;
// Result: "onClickButton" | "onClickLink" | "onHoverButton" | "onHoverLink"
```

**Practical Use Case (Strongly-Typed Event Emitter):**

```ts
type PropEventSource<T> = {
  on(eventName: `${string & keyof T}Changed`, callback: (newValue: any) => void): void;
};

declare function makeWatchedObject<T>(obj: T): T & PropEventSource<T>;

const person = makeWatchedObject({ name: "Bob", age: 30 });

person.on("nameChanged", (newName) => {}); // ✅ Autocompletes "nameChanged" | "ageChanged"!
// person.on("invalidChanged", ...);        // ❌ Compile Error!
```

---

### Q22. How does the `satisfies` operator in TS 4.9 differ from `as` type assertions or explicit type annotations?

**Answer.** Introduced in TypeScript 4.9, the **`satisfies` operator** validates that an expression matches a given interface/type **without widening or modifying the inferred type** of the expression.

```ts
type Colors = "red" | "green" | "blue";
type RGB = [red: number, green: number, blue: number];

type Palette = Record<Colors, string | RGB>;
```

#### Comparison:

```csharp
// 1. Explicit Type Annotation: Type becomes Record<Colors, string | RGB>
// Loose inference: palette1.red could be string OR RGB tuple!
const palette1: Palette = {
  red: [255, 0, 0],
  green: "#00ff00",
  blue: [0, 0, 255]
};
palette1.red.map(x => x); // ❌ Error! Property 'map' does not exist on type 'string | RGB'

// 2. satisfies Operator: Validates structure against Palette, BUT preserves exact tuple inference!
const palette2 = {
  red: [255, 0, 0],
  green: "#00ff00",
  blue: [0, 0, 255]
} satisfies Palette;

palette2.red.map(x => x);       // ✅ Allowed! TS knows palette2.red is exactly an RGB tuple!
palette2.green.toUpperCase();   // ✅ Allowed! TS knows palette2.green is string!
```

| Syntax | Validates Type Structure? | Preserves Exact Literal Inference? | Safe? |
| :--- | :--- | :--- | :--- |
| `const x: T = ...` | ✅ Yes | ❌ No (Widens to target `T`) | ✅ Safe |
| `const x = ... as T` | ❌ No (Bypasses checks) | ❌ No | ⚠️ Unsafe (bypasses missing property checks) |
| `const x = ... satisfies T` | ✅ Yes | ✅ Yes (Keeps exact inferred type) | ✅ Safe & Strict |

---

### Q23. What is Type Distribution over union types in conditional types, and how do you disable it?

**Answer.** When a conditional type acts on a generic parameter `T` that is a **naked type parameter** (e.g. `T extends U`), and `T` is instantiated with a **Union Type**, the conditional type **automatically distributes** across each constituent of the union.

```ts
type ToArray<T> = T extends any ? T[] : never;

type StringOrNumberArray = ToArray<string | number>;
// Distributes to: ToArray<string> | ToArray<number>
// Result: string[] | number[]
```

#### How to Disable Distribution (Tuple Wrapping):
If you want the conditional type to evaluate the union as a **single unit** instead of distributing over each member, wrap both `T` and the target type in square brackets `[T]`:

```ts
// Non-distributive conditional type (wrapped in tuple)
type ToArrayNonDistributive<T> = [T] extends [any] ? T[] : never;

type CombinedArray = ToArrayNonDistributive<string | number>;
// Result: (string | number)[]
```

> **Interview Summary**: Use naked types (`T extends ...`) when mapping or filtering unions. Use tuple wrapping (`[T] extends [...]`) when checking total union properties or preventing automatic union distribution.

