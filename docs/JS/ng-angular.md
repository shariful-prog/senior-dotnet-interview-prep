# L. Angular Interview Questions
---

> Covers modern Angular (2+, current versions) — not AngularJS. TypeScript questions are in [../JS/js-typescript.md](../JS/js-typescript.md).

## L1 — Components & Templates

### Q1. What is a component, and what is a module?

**Answer.** A **component** is the basic building block of an Angular application. It controls a part of the UI (a view) and holds the logic for that view.

A component consists of three parts:

- **HTML template** — the UI
- **TypeScript class** — the logic and data
- **CSS/SCSS** — the styling

The `@Component` decorator ties those three together:

```ts
@Component({
  selector: "app-user",           // used as <app-user></app-user>
  templateUrl: "./user.html",     // the template
  styleUrls: ["./user.css"]       // the styles
})
export class UserComponent {      // the logic
  name = "John";
}
```

Every Angular application is a tree of components, starting from one root component.

A **module** (`NgModule`) is a container that groups related Angular building blocks together — components, directives, pipes, and services. It organises the application and controls what is available to other parts of the app.

```ts
@NgModule({
  declarations: [UserComponent],    // components, directives, pipes it owns
  imports: [CommonModule],          // other modules this one needs
  providers: [UserService],         // services
  exports: [UserComponent]          // what other modules may use
})
export class UserModule { }
```

**Standalone components** (Angular 14+) removed the need for modules in most cases. The component lists what it needs itself:

```ts
@Component({
  standalone: true,
  imports: [CommonModule],        // ✅ no NgModule needed
  selector: "app-user",
  template: `<p>{{ name }}</p>`
})
export class UserComponent { }
```

New code uses standalone. Modules are still found in older projects.

---

### Q2. What are the types of data binding?

**Answer.** **Data binding** is the connection between the component class and its template. It keeps the two in sync, so you never write code to update the DOM by hand.

There are four types, grouped by direction:

- **Interpolation** — displays a class value in the template
- **Property binding** — sets an element or component property from the class
- **Event binding** — runs a class method when the user does something
- **Two-way binding** — both directions at once

| Syntax | Direction | Purpose |
|---|---|---|
| `{{ value }}` | component → view | display text |
| `[prop]="value"` | component → view | set a property |
| `(event)="method()"` | view → component | respond to an action |
| `[(ngModel)]="value"` | both ways | form inputs |

```html
<!-- 1. Interpolation — class to template -->
<p>{{ user.name }}</p>

<!-- 2. Property binding — class to element property -->
<img [src]="imageUrl">
<button [disabled]="isLoading">Save</button>

<!-- 3. Event binding — template to class -->
<button (click)="save()">Save</button>
<input (input)="onType($event)">

<!-- 4. Two-way binding — both directions -->
<input [(ngModel)]="user.name">
```

**`[(ngModel)]` is not magic** — the "banana in a box" is shorthand for a property binding plus an event binding:

```html
<input [(ngModel)]="name">

<!-- is exactly -->
<input [ngModel]="name" (ngModelChange)="name = $event">
```

So two-way binding is just the other two combined.

---

### Q3. How do `@Input` and `@Output` work?

**Answer.** They are the two decorators used for communication between a parent and child component.

- **`@Input`** marks a property as data the **parent passes in**. Data flows **down**.
- **`@Output`** marks a property as an event the child **sends out**, using an `EventEmitter`. Data flows **up**.

The rule behind it: a child never reaches into its parent. It receives data through inputs and reports back through events, so the parent stays in control of the state.

```ts
@Component({ selector: "app-child", template: `
  <p>{{ item.name }}</p>
  <button (click)="remove.emit(item.id)">Delete</button>
`})
export class ChildComponent {
  @Input() item!: Item;                              // data in
  @Output() remove = new EventEmitter<number>();     // events out
}
```

```html
<!-- parent -->
<app-child [item]="product" (remove)="onRemove($event)"></app-child>
```

`$event` holds whatever you passed to `emit()`.

**Inputs are not immediately available.** They are unset in the constructor and only populated before `ngOnInit`:

```ts
constructor() { console.log(this.item); }    // ❌ undefined
ngOnInit() { console.log(this.item); }       // ✅ set
```

For anything beyond parent and child — siblings, or components far apart — use a shared service instead (Q14).

---

### Q4. What is the difference between a component and a directive?

**Answer.** A **directive** is a class that adds behaviour to an existing element in the DOM. A **component** is a directive that also has its own template.

A component creates its own UI.

```typescript
@Component({
  selector: 'app-user',
  template: `<h1>User Profile</h1>`
})
export class UserComponent {}
```

Usage:

```html
<app-user></app-user>
```

### Directive Example

A directive modifies an existing element.

```typescript
@Directive({
  selector: '[highlight]'
})
export class HighlightDirective {
  constructor(el: ElementRef) {
    el.nativeElement.style.backgroundColor = 'yellow';
  }
}
```

Usage:

```html
<p highlight>Hello</p>
```

The directive does **not** create the `<p>` element. It only changes its appearance.

### Built-in Directives

- `*ngIf` – Conditionally adds or removes an element.
- `*ngFor` – Repeats an element for each item in a collection.
- `[ngClass]` – Dynamically adds or removes CSS classes.
- `[ngStyle]` – Dynamically applies CSS styles.

Example:

```html
<div *ngIf="isLoggedIn">
  Welcome!
</div>
```

---

### Q5. What is the new control flow syntax (`@if`, `@for`)?

**Answer.** **Built-in control flow** is template syntax for conditionals and loops, added in Angular 17. It replaces the structural directives `*ngIf`, `*ngFor`, and `*ngSwitch` with blocks written directly in the template.

Three blocks, matching the three directives:

- **`@if` / `@else if` / `@else`** — conditional rendering
- **`@for` ... `@empty`** — loops, with a fallback when the list is empty
- **`@switch` / `@case` / `@default`** — multi-way branching

```html
<!-- old -->
<div *ngIf="user; else loading">{{ user.name }}</div>
<ng-template #loading>Loading...</ng-template>

<!-- new -->
@if (user) {
  <div>{{ user.name }}</div>
} @else {
  Loading...
}
```

```html
@for (item of items; track item.id) {
  <li>{{ item.name }}</li>
} @empty {
  <li>No items</li>
}

@switch (status) {
  @case ("active") { <span>Active</span> }
  @default { <span>Unknown</span> }
}
```

Why it is better: no imports needed, `track` is required in `@for` (which prevents a common performance bug — see Q6), and `@empty` handles the empty list case.

The old syntax still works, but new code should use these blocks.

---

### Q6. What is `trackBy` and why does it matter?

**Answer.** 
`trackBy` is a function used with `*ngFor` to tell Angular **how to uniquely identify each item** in a list.

By default, Angular tracks items by **object reference**. If the list changes, Angular may recreate all DOM elements, even if only one item changed.

Using `trackBy` allows Angular to track items by a unique value (such as `id`), so it updates **only the changed items** instead of recreating the entire list.

#### Without `trackBy`

```html
<li *ngFor="let user of users">
  {{ user.name }}
</li>
```

If the `users` array is refreshed from the server, Angular may recreate every `<li>` element, even if only one user changed.

#### With `trackBy`

```html
<li *ngFor="let user of users; trackBy: trackByUserId">
  {{ user.name }}
</li>
```

```typescript
trackByUserId(index: number, user: User): number {
  return user.id;
}
```

Now Angular uses `user.id` to identify each item and updates only the items that actually changed.

---

### Q7. What is the difference between an attribute and a property binding?

**Answer.** 

#### Property Binding

Property binding updates a **DOM property** of an HTML element. A DOM property represents the **current state** of the element after it has been created by the browser. Since Angular works with the DOM, property binding is the most commonly used type of binding.

**Syntax:**

```html
<input [value]="username">
<button [disabled]="isDisabled">Save</button>
```

Equivalent JavaScript:

```javascript
input.value = username;
button.disabled = isDisabled;
```

#### Attribute Binding

Attribute binding updates an **HTML attribute** of an element. Attributes are defined in the HTML markup and provide **initial configuration or metadata** for the element. It is mainly used for attributes that do not have corresponding DOM properties.

**Syntax:**

```html
<td [attr.colspan]="2">Total</td>
<img [attr.aria-label]="'Profile Image'">
```

Equivalent JavaScript:

```javascript
td.setAttribute('colspan', '2');
img.setAttribute('aria-label', 'Profile Image');
```

---

### Q8. What is `ng-template` and `ngTemplateOutlet`?

**Answer.** 

### `ng-template`

`ng-template` is an Angular element used to **define a block of HTML that is not rendered immediately**. It acts as a reusable template that can be displayed later.

> **Purpose:** Define reusable or conditional UI content.

Example:

```html
<ng-template #loading>
  <p>Loading...</p>
</ng-template>
```

The template is **not rendered** until Angular is instructed to display it.

### `ngTemplateOutlet`

`ngTemplateOutlet` is a directive used to **render an `ng-template` at a specific location**.

Example:

```html
<ng-container *ngTemplateOutlet="loading"></ng-container>

<ng-template #loading>
  <p>Loading...</p>
</ng-template>
```

Output:

```text
Loading...
```


---

### Q9. How does view encapsulation work?

**Answer.** Styles in a component are **scoped to that component** by default. They do not leak out, and outside styles do not leak in.

```ts
@Component({
  styles: [`p { color: red; }`]      // only this component's <p> elements
})
```

Angular does this by adding a generated attribute to the elements and to the CSS selector, so the rule can only match this component's elements.

Three modes:

| Mode | Behaviour |
|---|---|
| `Emulated` (default) | Angular scopes the styles for you |
| `ShadowDom` | uses the browser's real shadow DOM |
| `None` | no scoping — the styles become global |

`:host` styles the component's own element, which is how you set its layout:

```css
:host { display: block; padding: 1rem; }
```

---

## L2 — Lifecycle & Change Detection

### Q10. What are the lifecycle hooks, in order?

**Answer.** **Lifecycle hooks** are methods Angular calls automatically at defined moments in a component's life — when it is created, when its inputs change, when its view is ready, and when it is destroyed. You implement the ones you need to run code at the right time.

1. **`ngOnChanges`** – Called when an `@Input()` property changes (also before `ngOnInit` on the first change).
2. **`ngOnInit`** – Called once after Angular initializes the component. Used for initialization logic.
3. **`ngDoCheck`** – Called during every change detection cycle.
4. **`ngAfterContentInit`** – Called once after external content (`<ng-content>`) is projected.
5. **`ngAfterContentChecked`** – Called after every check of projected content.
6. **`ngAfterViewInit`** – Called once after the component's view and child views are initialized.
7. **`ngAfterViewChecked`** – Called after every check of the component's view and child views.
8. **`ngOnDestroy`** – Called just before the component is destroyed. Used for cleanup (unsubscribe, remove timers, etc.).

---

### Q11. How does change detection work, and what is `OnPush`?

**Answer.** Change detection is the process Angular uses to **check if component data has changed and update the UI**.

By default, Angular runs change detection after events such as:

- Button clicks
- HTTP responses
- Timers (`setTimeout`, `setInterval`)
- User input

Example:

```typescript
count = 0;

increment() {
  this.count++;
}
```

```html
<button (click)="increment()">+</button>
<p>{{ count }}</p>
```

When the button is clicked, Angular detects that `count` changed and automatically updates the UI.

#### `OnPush` Change Detection

With the default strategy, Angular checks the component whenever change detection runs.

With **`OnPush`**, Angular only checks the component when:

- An `@Input()` reference changes.
- An event occurs inside the component (click, input, etc.).
- An observable used with the `async` pipe emits a new value.
- You manually trigger change detection.

Example:

```typescript
@Component({
  selector: 'app-user',
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserComponent {
  @Input() user!: User;
}
```

If you modify the existing object:

```typescript
this.user.name = "Bob";
```

The UI **may not update** because the object reference didn't change.

Instead, create a new object:

```typescript
this.user = {
  ...this.user,
  name: "Bob"
};
```

Now the object reference changes, so Angular updates the UI.

---

### Q12. What are signals?

**Answer.** **Signals** are Angular's reactive state management feature. A signal holds a value, and Angular automatically updates the UI when that value changes.

Example:

```typescript
import { signal } from '@angular/core';

count = signal(0);

increment() {
  this.count.update(value => value + 1);
}
```

```html
<p>{{ count() }}</p>
<button (click)="increment()">+</button>
```

When `increment()` is called, the UI automatically updates because the signal's value changed.

#### Why use Signals?

- Simpler state management.
- Automatic UI updates.
- Better performance because Angular updates only the parts of the UI that depend on the changed signal.


#### Why use Signals if normal variables already update the UI?

With a normal variable:

```typescript
count = 0;

increment() {
  this.count++;
}
```

Angular updates the UI, but it first **checks the component (and possibly many others)** to see what changed.

With a signal:

```typescript
count = signal(0);

increment() {
  this.count.update(v => v + 1);
}
```

The signal **notifies Angular that it changed**, so Angular updates **only the parts of the UI that depend on that signal**, reducing unnecessary checks and improving performance.

The practical split: **signals for state, RxJS for async streams.**

---

### Q13. What is Zone.js, and what does "zoneless" mean?

**Answer.** **Zone.js** is a library that Angular uses to detect asynchronous operations (such as button clicks, HTTP requests, timers, and promises) and automatically trigger **change detection**.

Example:

```typescript
count = 0;

setTimeout(() => {
  this.count++;
}, 1000);
```

When `count` changes, **Zone.js** notices that `setTimeout` finished and tells Angular to run change detection, so the UI updates automatically.

#### What is Zoneless?

In **zoneless** Angular, **Zone.js is not used**.

Instead of automatically checking for changes after every asynchronous operation, Angular relies on **Signals** or manual change detection to know exactly when to update the UI.

---

## L3 — Services & Dependency Injection

### Q14. How does dependency injection work in Angular?

**Answer.** **Dependency injection (DI)** is a pattern where a class does not create the objects it needs — it declares them, and the framework supplies them. Angular has a built-in **injector** that creates these objects and passes them in.

A **service** is a class holding logic or state meant to be shared: HTTP calls, business rules, application state. Keeping that out of components makes it reusable and testable.

Three pieces make it work:

- **`@Injectable()`** — marks a class as available for injection
- **Provider** — the registration telling the injector how to create it
- **Injection** — declaring it in a constructor, or with `inject()`

```ts
@Injectable({ providedIn: "root" })      // one instance for the whole app
export class UserService {
  getUsers() { return this.http.get<User[]>("/api/users"); }
  constructor(private http: HttpClient) { }
}
```

```ts
export class UserComponent {
  constructor(private users: UserService) { }    // ✅ injected
}
```

---

### Q15. What is `HttpClient`, and how do you handle errors?

**Answer.** `HttpClient` performs HTTP requests and returns **observables**. Nothing is sent until you subscribe.

```ts
@Injectable({ providedIn: "root" })
export class UserService {
  constructor(private http: HttpClient) { }

  getUsers(): Observable<User[]> {
    return this.http.get<User[]>("/api/users");
  }
}
```

```ts
this.service.getUsers().subscribe({
  next: users => this.users = users,
  error: err => this.error = err.message
});
```

Errors are handled with `catchError` in the pipe. `pipe()` is used to **transform or handle data** before it reaches the subscriber.

```ts
getUsers() {
  return this.http.get<User[]>("/api/users").pipe(
    catchError(err => {
      console.error(err);
      return of([]);              // ✅ return a safe fallback
    })
  );
}
```

Two useful points: nothing is sent until you subscribe, and a 404 or 500 **is** treated as an error (unlike `fetch`), so it reaches `catchError`.

---

### Q16. What is an HTTP interceptor?

**Answer.** An **HTTP Interceptor** is a service that intercepts **every HTTP request and response**. It allows you to modify requests or handle responses globally.

Common uses:

- Add authentication tokens.
- Log requests and responses.
- Handle errors globally.
- Show/hide loading spinners.

Example:

```typescript
@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler) {

    const authReq = req.clone({
      setHeaders: {
        Authorization: 'Bearer my-token'
      }
    });

    return next.handle(authReq);
  }
}
```

❌ **The request object is immutable** — you must `clone()` it. Setting a header directly does nothing.

---

### Q17. What is an `InjectionToken`, and how do you inject a value that is not a class?

**Answer.** An `InjectionToken` is used to inject **values that are not classes**, such as strings, numbers, objects, or configuration settings.

#### Example

Create an injection token:

```typescript
import { InjectionToken } from '@angular/core';

export const API_URL = new InjectionToken<string>('API_URL');
```

Provide a value:

```typescript
providers: [
  { provide: API_URL, useValue: 'https://api.example.com' }
]
```

Inject the value into a component or service:

```typescript
import { Inject } from '@angular/core';

constructor(
  @Inject(API_URL) private apiUrl: string
) {}
```

Now `apiUrl` contains:

```text
https://api.example.com
```

#### Why not inject a string directly?

Angular's DI system uses **tokens** to identify dependencies. Since strings and other primitive values don't have a class type, an `InjectionToken` is used as a unique identifier.


---

## L4 — RxJS

### Q18. What is an observable, and how does it differ from a promise?

**Answer.** An **Observable** is an object that **can emit zero, one, or many values over time**. It starts producing values **when you subscribe to it**.

Example:

```typescript
this.http.get<User[]>('/api/users').subscribe(users => {
  console.log(users);
});
```

The HTTP request is **not sent** until `subscribe()` is called.

#### What is a Promise?

A **Promise** represents the result of an asynchronous operation that will produce **one value (or one error) in the future**.

Unlike an Observable, a Promise **starts executing immediately when it is created**.

Example:

```typescript
const promise = fetch('/api/users');

promise
  .then(response => response.json())
  .then(data => console.log(data));
```

The HTTP request is sent **as soon as `fetch()` is called**, even if you never call `.then()`.

Angular uses observables because most things a UI deals with arrive over time: keystrokes, clicks, route changes, HTTP responses. Being able to cancel matters too — leaving a page should stop its pending requests.

---

### Q19. What are the most common RxJS operators?

**Answer.** An **operator** is a function that takes an observable and returns a new one, transforming the stream as values pass through. You chain them inside `pipe()`.

They fall into groups: **transformation** (`map`), **filtering** (`filter`, `debounceTime`), **flattening** (`switchMap`), and **error handling** (`catchError`).

The ones that appear in nearly every Angular codebase:

```ts
map(u => u.name)                    // transform each value
filter(u => u.active)               // drop values
tap(u => console.log(u))            // side effect, passes through
switchMap(id => this.http.get(...)) // flatten an inner observable
debounceTime(300)                   // wait for a pause
distinctUntilChanged()              // ignore repeats
takeUntilDestroyed()                // stop when the component is destroyed
catchError(err => of([]))           // recover
```

**`switchMap` vs `mergeMap`** is the one interviews ask about:

| Operator | When a new value arrives |
|---|---|
| `switchMap` | **cancels** the previous request — use for search and navigation |
| `mergeMap` | lets all requests run at once — use for independent work |

The classic search box uses three operators together:

```ts
this.search.valueChanges.pipe(
  debounceTime(300),                        // wait for typing to pause
  distinctUntilChanged(),                   // skip if unchanged
  switchMap(term => this.api.search(term))  // ✅ cancel the stale request
).subscribe(r => this.results = r);
```

❌ With `mergeMap` here, a slow earlier response can arrive **after** a faster later one and overwrite the correct results.

> `debounceTime()` is an RxJS operator that **waits for a specified amount of time after the last emitted value before emitting it**. If a new value arrives before the time expires, the timer resets.

---

### Q20. How do you avoid memory leaks with subscriptions?

**Answer.** A memory leak can occur if you subscribe to an Observable but never unsubscribe. The subscription may continue receiving values even after the component is destroyed.

#### 1. Unsubscribe in `ngOnDestroy()`

```typescript
subscription!: Subscription;

ngOnInit() {
  this.subscription = this.userService.getUsers().subscribe();
}

ngOnDestroy() {
  this.subscription.unsubscribe();
}
```

#### Use the `async` pipe (Recommended)

Instead of subscribing manually:

```typescript
users$ = this.userService.getUsers();
```

```html
<ul>
  <li *ngFor="let user of users$ | async">
    {{ user.name }}
  </li>
</ul>
```

The `async` pipe:
- Automatically subscribes to `users$`.
- Updates the UI when data arrives.
- Automatically unsubscribes when the component is destroyed.

#### 3. Use `takeUntil` or `takeUntilDestroyed`

```typescript
this.userService.getUsers()
  .pipe(takeUntilDestroyed())
  .subscribe();
```

Angular automatically unsubscribes when the component is destroyed.

- **What does not leak:** `HttpClient` calls finish on their own after one value. 
- The ones to watch are **never-ending** streams — `interval`, `fromEvent`, `Subject`, and form `valueChanges`.

Prefer the `async` pipe, which removes the problem entirely.

---

### Q21. What are the four kinds of Subject?

**Answer.** 
### The Four Types of `Subject` in RxJS

#### 1. `Subject`
Does **not** store any previous value. Subscribers receive only values emitted **after** they subscribe.

```typescript
const subject = new Subject<number>();

subject.subscribe(v => console.log("A:", v));

subject.next(1); // A: 1
```

#### 2. `BehaviorSubject`
Stores the **latest value** and requires an initial value. New subscribers immediately receive the current value.

```typescript
const subject = new BehaviorSubject<number>(0);

subject.next(1);

subject.subscribe(v => console.log(v)); // 1
```

#### 3. `ReplaySubject`
Stores a specified number of previous values and replays them to new subscribers.

```typescript
const subject = new ReplaySubject<number>(2);

subject.next(1);
subject.next(2);
subject.next(3);

subject.subscribe(v => console.log(v));
// 2
// 3
```

#### 4. `AsyncSubject`
Emits **only the last value**, and only after `complete()` is called.

```typescript
const subject = new AsyncSubject<number>();

subject.next(1);
subject.next(2);
subject.next(3);

subject.complete();

subject.subscribe(v => console.log(v)); // 3
```

---

### Q22. How do you turn a callback or event into an observable?

**Answer.** Use RxJS creation functions such as **`fromEvent()`** for DOM events or **`new Observable()`** for custom callbacks.

#### `fromEvent()`

```typescript
import { fromEvent } from 'rxjs';

const clicks$ = fromEvent(document, 'click');

clicks$.subscribe(() => {
  console.log('Clicked!');
});
```

Every time the user clicks, the Observable emits a value.

#### `new Observable()`

```typescript
import { Observable } from 'rxjs';

const observable = new Observable(observer => {
  observer.next('Hello');
  observer.next('World');
  observer.complete();
});

observable.subscribe(value => console.log(value));
```

---

## L5 — Routing & Forms

### Q23. How does routing work?

**Answer.** The **Angular Router** maps URL paths to components. When the URL changes it loads the matching component into a `<router-outlet>` in your template, without a full page reload — which is what makes it a single-page application.

Four pieces:

- **Routes** — the array mapping each path to a component
- **`<router-outlet>`** — the placeholder where the matched component is rendered
- **`routerLink`** — navigation in templates, instead of `href`
- **`Router` service** — navigation from code

```ts
export const routes: Routes = [
  { path: "", component: HomeComponent },
  { path: "users/:id", component: UserComponent },        // route parameter
  { path: "admin", loadChildren: () => import("./admin/routes") },  // lazy
  { path: "**", component: NotFoundComponent }            // wildcard — must be last
];
```

```html
<a routerLink="/users/1" routerLinkActive="active">User</a>
<router-outlet></router-outlet>
```

Reading parameters — use the observable form, since the component is **reused** when only the id changes:

```ts
ngOnInit() {
  this.route.paramMap.subscribe(p => this.load(p.get("id")));   // ✅ reacts
}

// ❌ snapshot only reads once — misses /users/1 → /users/2
const id = this.route.snapshot.paramMap.get("id");
```

#### Route Guards

**Route Guards** control whether a user is allowed to navigate to a route. They are commonly used for authentication, authorization, or preventing navigation away from a page.

Example:

```typescript
export const authGuard: CanActivateFn = () =>
  inject(AuthService).isLoggedIn();
```

Use in routing:

```typescript
const routes: Routes = [
  {
    path: 'admin',
    component: AdminComponent,
    canActivate: [authGuard]
  }
];
```

If `isLoggedIn()` returns `true`, navigation is allowed; otherwise, it is blocked or redirected.


#### Lazy Loading

**Lazy Loading** loads a feature module **only when it is first visited**, instead of downloading it when the application starts.

Without lazy loading, when a user opens your application, Angular downloads **all modules** (Home, Products, Admin, Reports, etc.), even if the user only visits the Home page. This increases the initial bundle size and slows down the application's startup.

With lazy loading, Angular downloads only the modules required for the initial page. Other feature modules are downloaded **on demand** when the user first navigates to their routes.

Example:

```typescript
const routes: Routes = [
  {
    path: 'admin',
    loadChildren: () =>
      import('./admin/admin.routes').then(m => m.ADMIN_ROUTES)
  }
];
```

When the user visits `/admin`:

1. Angular downloads the admin module.
2. The routes are registered.
3. The requested component is displayed.

---

### Q24. Template-driven vs reactive forms?

**Answer.** Angular provides two approaches for building forms:

- **Template-Driven Forms** – Form logic is defined mainly in the HTML template.
- **Reactive Forms** – Form logic is defined in the TypeScript component.

### Template-Driven Forms

In **Template-Driven Forms**, Angular automatically creates and manages the form based on directives in the template.

Example:

```html
<form #userForm="ngForm">
  <input
    name="name"
    [(ngModel)]="user.name"
    required
  />
</form>
```

#### Characteristics

- Form logic is in the HTML.
- Uses `ngModel`.
- Simple to write.
- Best for small and simple forms.


### Reactive Forms

In **Reactive Forms**, the form is created and managed in the component using `FormGroup` and `FormControl`.

Component:

```typescript
form = new FormGroup({
  name: new FormControl('')
});
```

Template:

```html
<form [formGroup]="form">
  <input formControlName="name" />
</form>
```

#### Characteristics

- Form logic is in TypeScript.
- Uses `FormGroup` and `FormControl`.
- Easier to test.
- Better for large and complex forms.
- Supports dynamic forms and custom validation.

### Which one should you use?

- **Template-Driven Forms** → Small, simple forms.
- **Reactive Forms** → Large, complex, dynamic forms (recommended for most enterprise applications).

---

### Q25. What is a `FormArray`, and how do you build a dynamic form?

**Answer.** A **FormArray** is a Reactive Forms class that stores an **ordered list of form controls, form groups, or other form arrays**. It is used when the **number of form fields is dynamic** and not known in advance.

> **Purpose:** Build dynamic forms where users can add or remove fields at runtime.

## How to build a dynamic form

### Step 1: Create a `FormArray`

```typescript
form = new FormGroup({
  phones: new FormArray([])
});
```

### Step 2: Add controls dynamically

```typescript
get phones() {
  return this.form.get('phones') as FormArray;
}

addPhone() {
  this.phones.push(new FormControl(''));
}
```

### Step 3: Remove controls

```typescript
removePhone(index: number) {
  this.phones.removeAt(index);
}
```

### Step 4: Display the controls

```html
<form [formGroup]="form">
  <div formArrayName="phones">
    <div *ngFor="let phone of phones.controls; let i = index">
      <input [formControlName]="i">
      <button (click)="removePhone(i)">Remove</button>
    </div>
  </div>

  <button (click)="addPhone()">Add Phone</button>
</form>

```

---

### Q26. What is a resolver, and how else do you pass data to a route?

**Answer.** A **resolver** fetches data *before* the route activates, so the component renders with data already present instead of showing an empty state.

```ts
export const userResolver: ResolveFn<User> = (route) =>
  inject(UserService).getUser(route.paramMap.get("id")!);
```

```ts
{ path: "users/:id", component: UserComponent, resolve: { user: userResolver } }
```

```ts
ngOnInit() {
  this.user = this.route.snapshot.data["user"];    // ✅ already loaded
}
```

The trade-off: navigation **waits** for the resolver. A slow call means the old page stays visible with nothing happening, which can feel broken. Load in the component with a loading indicator when the request is slow; use a resolver when it is fast and the component genuinely cannot render without the data.

**The other ways to pass data to a route:**

| Way | Example | Read with |
|---|---|---|
| Route parameter | `users/:id` | `route.paramMap` |
| Query parameter | `/users?page=2` | `route.queryParamMap` |
| Static data | `data: { title: "Admin" }` | `route.data` |

Use a route parameter for **which record** it is, and a query parameter for **state you want in the URL** — page number, filters, sorting.

---

### Q27. What is content projection (`ng-content`)?

**Answer.** **Content Projection** is an Angular feature that allows a parent component to **pass HTML content into a child component**. The child component displays the passed content using the `<ng-content>` placeholder.

> **Purpose:** Create reusable components whose content can be customized by the parent.

### Example

#### Child Component

```html
<div class="card">
  <h2>Product</h2>
  <ng-content></ng-content>
</div>
```

#### Parent Component

```html
<app-card>
  <p>Apple iPhone 16</p>
  <button>Buy Now</button>
</app-card>
```

#### Rendered Output

```html
<div class="card">
  <h2>Product</h2>
  <p>Apple iPhone 16</p>
  <button>Buy Now</button>
</div>
```

The content inside `<app-card>` is projected into the `<ng-content>` placeholder.

---

## L6 — Practical & Performance

### Q28. What is a pipe, and when should you make one impure?

**Answer.** A **Pipe** is an Angular feature that **transforms data for display in a template** without modifying the original data.

> **Purpose:** Format or transform values before displaying them in the UI.

### Example

Built-in pipes:

```html
{{ price | currency }}
{{ name | uppercase }}
{{ today | date:'dd/MM/yyyy' }}
```

Custom pipe:

```typescript
@Pipe({
  name: 'greet'
})
export class GreetPipe implements PipeTransform {
  transform(value: string): string {
    return `Hello ${value}`;
  }
}
```

Usage:

```html
{{ 'John' | greet }}
```

Output:

```text
Hello John
```

### Pure Pipe (Default)

A **pure pipe** runs **only when the input value or object reference changes**.

```typescript
@Pipe({
  name: 'greet',
  pure: true
})
```

- Better performance.
- Recommended for most scenarios.

### Impure Pipe

An **impure pipe** runs **on every Angular change detection cycle**, even if the object's reference hasn't changed.

```typescript
@Pipe({
  name: 'filterItems',
  pure: false
})
```

Use an impure pipe when the data is **mutated without changing its reference**.

Example:

```typescript
items.push(newItem);
```

The array reference remains the same, so a **pure pipe won't run**. An **impure pipe** detects the change and updates the UI.

> **Note:** Since impure pipes execute on every change detection cycle, they can negatively impact performance and should be used only when necessary.


---

### Q29. How do you improve performance in an Angular app?

**Answer.** Angular performance work targets two things: the **initial bundle size** (how long before the app is usable) and the **cost of change detection** (how much work each user interaction triggers).

The measures that matter, in rough order of impact:

**1. Lazy load routes** — the biggest win on initial load:

```ts
{ path: "admin", loadChildren: () => import("./admin/routes") }
```

**2. `OnPush` change detection** (Q11) — stops most of the tree being checked on every event.

**3. `trackBy` / `track`** (Q6) — prevents full list re-rendering.

**4. Signals** (Q12) — updates only the views that read the changed value.

**5. Move work out of templates.** A method call in a binding runs on every cycle:

```html
{{ getTotal() }}      <!-- ❌ every change detection -->
{{ total }}           <!-- ✅ computed once when data changes -->
```

**6. `@defer`** (Angular 17+) — load a section only when it is needed:

```html
@defer (on viewport) {
  <app-comments />
}
```

**7. Virtual scrolling** for long lists, so only the visible rows exist in the DOM.

Measure first with Angular DevTools, so you fix what is actually slow.

---

### Q30. What is `ViewChild`, and when can you use it?

**Answer.** **`@ViewChild`** is a decorator that gives your class a reference to an element, directive, or child component from its own template. It is how you reach the DOM or call a child's methods when a binding is not enough.

```html
<input #search>
<app-child></app-child>
```

```ts
@ViewChild("search") searchInput!: ElementRef<HTMLInputElement>;
@ViewChild(ChildComponent) child!: ChildComponent;

ngAfterViewInit() {
  this.searchInput.nativeElement.focus();    // ✅ view exists now
  this.child.reload();
}
```

❌ **It is `undefined` in `ngOnInit`** — the view has not been created yet. This is one of the most common Angular errors.

Also note: if the element is inside an `@if` or `*ngIf` that is currently false, it does not exist, so check before using it.

Use `@ViewChildren` when several elements match. Prefer normal bindings where you can — reach for `ViewChild` only when you really need the DOM, such as setting focus.

---

### Q31. How do you share state between unrelated components?

**Answer.** `@Input` and `@Output` only connect a direct parent and child. For components that are siblings, or far apart in the tree, you use a **shared service**: a singleton that holds the state, which every component injects.

Because all of them receive the same instance, one component writing to the service is immediately visible to the others.

With signals (current approach):

```ts
@Injectable({ providedIn: "root" })
export class CartService {
  private items = signal<Item[]>([]);

  readonly all = this.items.asReadonly();          // ✅ expose read-only
  readonly total = computed(() => this.items().reduce((s, i) => s + i.price, 0));

  add(item: Item) {
    this.items.update(list => [...list, item]);    // ✅ new array reference
  }
}
```

```ts
export class HeaderComponent {
  private cart = inject(CartService);
  count = computed(() => this.cart.all().length);   // updates automatically
}
```

The older way used a `BehaviorSubject` instead of a signal, which works the same way:

```ts
private items$ = new BehaviorSubject<Item[]>([]);
readonly all$ = this.items$.asObservable();
```

❌ Expose the state as readonly (`asReadonly()` or `asObservable()`) so components have to go through your methods instead of changing it directly.

---

### Q32. What is NgRx, and when is it worth the complexity?

**Answer.** **NgRx** is a Redux-style store for Angular: one immutable state object for the app, changed only by dispatching actions.

Five pieces:

| Piece | Role |
|---|---|
| **Store** | the single state object |
| **Action** | a description of an event ("Load Users") |
| **Reducer** | a pure function: `(state, action) => newState` |
| **Selector** | reads a slice of state, memoised |
| **Effect** | handles side effects — HTTP, then dispatches a result action |

```ts
// 1. an action describes what happened
export const loadUsers = createAction("[Users] Load");

// 2. a reducer produces the new state
export const reducer = createReducer(initialState,
  on(loadUsers, s => ({ ...s, loading: true }))     // ✅ always a new object
);

// 3. a component dispatches the action and reads state through a selector
users = this.store.selectSignal(selectAllUsers);
load() { this.store.dispatch(loadUsers()); }
```

**The flow goes one way only:** component dispatches an action → effect calls the API → effect dispatches a success action → reducer makes the new state → every component reading that state updates.

**What you gain:** every change is traceable, state is predictable, and DevTools lets you step back through changes.

**What it costs:** many files per feature and a lot of boilerplate.

❌ **NgRx is often used when it is not needed.** For most apps a shared service with signals (Q31) does the same job with far less code. Use NgRx when several unrelated features share the same state, or when a large team needs one enforced pattern.

---

### Q33. How do you test a component and a service?

**Answer.** Angular tests use **`TestBed`**, which builds a small testing module so components and services can be created with their dependencies replaced by stubs.

The two are tested differently:

- **A service** is a plain class, so you can often construct it directly. `HttpTestingController` replaces the real HTTP backend so no request leaves the test.
- **A component** needs `TestBed` to create it and its template. You get a **fixture** — a handle on the component instance and its rendered DOM.

**Testing a service:**

```ts
it("fetches users", () => {
  TestBed.configureTestingModule({
    providers: [UserService, provideHttpClient(), provideHttpClientTesting()]
  });
  const service = TestBed.inject(UserService);
  const http = TestBed.inject(HttpTestingController);

  service.getUsers().subscribe(u => expect(u.length).toBe(1));

  http.expectOne("/api/users").flush([{ id: 1, name: "John" }]);  // ✅ fake response
  http.verify();                                    // ✅ no unexpected requests
});
```

**Components** need `TestBed` to build them, and a `fixture` to interact with:

```ts
beforeEach(() => {
  TestBed.configureTestingModule({
    imports: [UserComponent],                   // standalone
    providers: [{ provide: UserService, useValue: mockService }]   // ✅ stub the dep
  });
  fixture = TestBed.createComponent(UserComponent);
  component = fixture.componentInstance;
});

it("shows the name", () => {
  component.user = { id: 1, name: "John" };
  fixture.detectChanges();                       // ✅ required — runs CD manually
  const el = fixture.nativeElement.querySelector("p");
  expect(el.textContent).toContain("John");
});
```

❌ **`detectChanges()` is the most common cause of a failing test.** Change detection does not run automatically in tests, so the DOM still holds the old value until you call it.

The general principle: **replace the dependencies with stubs, then test the behaviour.** Check what the user would see, not which internal methods ran.

---

### Q34. What is SSR, and what problem does hydration solve?

**Answer.** By default Angular renders in the browser: the server sends a nearly empty HTML page, and JavaScript builds the DOM. That means a blank screen until the bundle downloads and runs, and crawlers may see nothing.

**SSR (Server-Side Rendering)** renders the HTML on the server, so the first response already contains the content:

```bash
ng add @angular/ssr
```

What it improves: **first contentful paint**, since content is visible before the JavaScript arrives, and **SEO**, since crawlers receive real markup.

**The problem it creates.** The server sends working HTML, then Angular boots in the browser and — without hydration — **throws it all away and rebuilds from scratch.** The page visibly flickers and all that server work is wasted.

**Hydration** fixes this: Angular reuses the existing DOM instead of re-creating it, attaching event listeners and state to the nodes already there.

```ts
provideClientHydration()          // ✅ Angular 16+, on by default in 17+
```

❌ **Two things break under SSR** and come up in interviews:

```ts
// 1. Browser globals do not exist on the server
if (typeof window !== "undefined") { }                // ✅ guard
if (isPlatformBrowser(inject(PLATFORM_ID))) { }       // ✅ Angular's way

// 2. Direct DOM manipulation
document.getElementById("x");                          // ❌ no document on the server
```

Use SSR for public pages where SEO and fast first load matter. For an internal dashboard behind a login it adds complexity for little benefit.


## How does Server-Side Rendering (SSR) actually work?

Normally, an Angular application runs **in the browser**.

```text
Browser
    │
Downloads HTML
    │
Downloads JavaScript
    │
Angular starts
    │
Builds the page (DOM)
```

With **SSR**, Angular is also able to run **on a Node.js server**.

Instead of the browser building the page, the **server builds it first**.

### Step-by-step

#### 1. User requests a page

```text
GET /products/5
```

#### 2. Request reaches the Node.js server

The server starts an Angular application (just like the browser normally would).

```text
Client
   │
   ▼
Node.js Server
```

#### 3. Angular runs on the server

Angular:

- Creates the components
- Executes lifecycle hooks (`ngOnInit`, etc.)
- Calls APIs (if needed)
- Generates the HTML

For example, if your component is:

```html
<h1>{{ product.name }}</h1>
```

and the API returns:

```text
iPhone 16
```

Angular renders:

```html
<h1>iPhone 16</h1>
```

**This rendering happens on the server, not in the browser.**

#### 4. Server sends the rendered HTML

Instead of sending an empty page:

```html
<body>
  <app-root></app-root>
</body>
```

the server sends:

```html
<body>
  <app-root>
      <h1>iPhone 16</h1>
      <p>$999</p>
  </app-root>
</body>
```

The browser can display the page immediately.

#### 5. Browser downloads Angular

After displaying the HTML, the browser downloads the Angular JavaScript bundle.

#### 6. Hydration

Angular starts in the browser and **attaches event handlers** (clicks, forms, etc.) to the existing HTML instead of rebuilding it.

```text
User Request
      │
      ▼
Node.js Server
      │
Angular renders HTML
      │
      ▼
Rendered HTML sent to browser
      │
      ▼
Browser displays page immediately
      │
      ▼
Angular JavaScript downloads
      │
      ▼
Hydration makes the page interactive
```

## Why does Angular run on the server?

Angular normally manipulates the DOM, which doesn't exist on a server. For SSR, Angular uses **Angular Universal**, which runs the application in a **Node.js** environment. It renders the components into an HTML string instead of updating a real browser DOM.

---

## L2 — Advanced Modern Angular, Architecture & Security (Q35 to Q50)

### Q35. What are Standalone Components, Directives, and Pipes (`standalone: true`), and how do they eliminate `NgModule`?

**Answer.** Introduced in Angular 14 and made the default in Angular 17, **Standalone Components** allow building Angular applications without `NgModule` declarations.

Instead of declaring components inside an `@NgModule`, a standalone component directly specifies its own dependencies via the `imports` array inside `@Component`:

```ts
@Component({
  selector: "app-user-profile",
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule, UserAvatarComponent], // Direct imports!
  template: `
    <h2>{{ user.name }}</h2>
    <app-user-avatar [url]="user.avatarUrl"></app-user-avatar>
  `
})
export class UserProfileComponent {
  @Input() user!: { name: string; avatarUrl: string };
}
```

#### Bootstrapping Standalone Applications:
In standalone applications, `AppModule` is completely removed. Bootstrapping is configured via `bootstrapApplication()` in `main.ts`:

```ts
import { bootstrapApplication } from "@angular/platform-browser";
import { provideHttpClient } from "@angular/common/http";
import { provideRouter } from "@angular/router";
import { AppComponent } from "./app/app.component";
import { routes } from "./app/app.routes";

bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes),
    provideHttpClient()
  ]
}).catch(err => console.error(err));
```

---

### Q36. What are Signal Inputs (`input()`, `input.required()`) and Model Inputs (`model()`) in Angular 17.1+?

**Answer.** In Angular 17.1+, Signal Inputs replace traditional `@Input()` decorators with read-only Signal values, seamlessly integrating with `computed()` and `effect()`.

```ts
import { Component, input, model, computed } from "@angular/core";

@Component({
  selector: "app-counter",
  standalone: true,
  template: `
    <p>Count: {{ count() }} (Double: {{ doubleCount() }})</p>
    <button (click)="increment()">Increment Model</button>
  `
})
export class CounterComponent {
  // 1. Optional Signal Input with default value
  label = input<string>("Default Label");

  // 2. Required Signal Input (compile error if parent omits binding)
  maxLimit = input.required<number>();

  // 3. Computed signal derived from signal input
  doubleLimit = computed(() => this.maxLimit() * 2);

  // 4. Model Input (Two-way binding signal: creates two-way binding with parent!)
  count = model<number>(0);

  increment() {
    this.count.update(c => c + 1); // Updates value and notifies parent!
  }
}
```

#### Summary of Input Signals:
- **`input()`**: Returns a **Read-only Signal** (`Signal<T>`). Parent updates flow in automatically.
- **`model()`**: Returns a **Writable Signal** (`WritableSignal<T>`). Enables two-way binding without needing `@Output() countChange = new EventEmitter()`.

---

### Q37. What are Deferrable Views (`@defer`, `@placeholder`, `@loading`, `@error`) in Angular 17+?

**Answer.** **Deferrable Views** (`@defer`) allow declarative lazy-loading of template sections and their associated component JS bundles directly in the HTML template.

```html
@defer (on viewport) {
  <!-- Large heavy component JS bundle is fetched ONLY when scrolled into viewport! -->
  <app-heavy-chart [data]="chartData"></app-heavy-chart>
} @placeholder (minimum 500ms) {
  <!-- Rendered immediately before defer trigger activates -->
  <div class="skeleton-loader">Loading Chart Placeholder...</div>
} @loading (after 100ms; minimum 1s) {
  <!-- Rendered while the JS bundle is downloading -->
  <app-spinner></app-spinner>
} @error {
  <!-- Rendered if fetching the JS bundle fails -->
  <p>Failed to load chart component.</p>
}
```

#### Available `@defer` Triggers:
- **`on viewport`**: Triggers when content enters the browser viewport.
- **`on hover`**: Triggers when the user hovers over the placeholder.
- **`on interaction`**: Triggers on user click or keypress on placeholder.
- **`on idle`**: Triggers when browser reaches idle state.
- **`when condition`**: Triggers based on a custom boolean expression (e.g. `when isDataReady()`).

---

### Q38. How do `toSignal()` and `toObservable()` work in `@angular/core/rxjs-interop`?

**Answer.** The `@angular/core/rxjs-interop` package provides helper functions to convert between RxJS Observables and Angular Signals.

#### 1. `toSignal(observable$)`: Converts Observable $\rightarrow$ Signal
```ts
import { Component, inject } from "@angular/core";
import { toSignal } from "@angular/core/rxjs-interop";
import { HttpClient } from "@angular/common/http";

@Component({
  selector: "app-user-list",
  standalone: true,
  template: `
    @for (user of users(); track user.id) {
      <li>{{ user.name }}</li>
    }
  `
})
export class UserListComponent {
  private http = inject(HttpClient);

  // Automatically subscribes to HTTP observable and updates signal!
  // Unsubscribes automatically when component is destroyed.
  users = toSignal(this.http.get<User[]>("/api/users"), { initialValue: [] });
}
```

#### 2. `toObservable(signal)`: Converts Signal $\rightarrow$ Observable
```ts
import { Component, signal } from "@angular/core";
import { toObservable } from "@angular/core/rxjs-interop";
import { debounceTime, switchMap } from "rxjs";

export class SearchComponent {
  query = signal("");
  
  // Convert signal to RxJS observable to apply operators like debounceTime & switchMap
  query$ = toObservable(this.query);

  searchResults$ = this.query$.pipe(
    debounceTime(300),
    switchMap(q => this.http.get(`/api/search?q=${q}`))
  );
}
```

---

### Q39. How does Angular's Hierarchical Injector work (Element Injector vs Environment Injector), and what do `@Self()`, `@SkipSelf()`, `@Optional()`, and `@Host()` do?

**Answer.** Angular has two injector hierarchies:
1. **Environment Injector Hierarchy**: Configured in `bootstrapApplication({ providers: [...] })` or `providedIn: 'root'`. Serves application-wide singletons.
2. **Element Injector Hierarchy**: Created per DOM element/component via `@Component({ providers: [...] })`. Searches upward from child component to parent component.

#### Resolution Modifiers (DI Parameter Decorators or `inject()` options):

```ts
@Component({
  selector: "app-child",
  standalone: true
})
export class ChildComponent {
  // Using modern inject() function with flags:
  private localService = inject(LoggerService, {
    optional: true,  // Doesn't throw error if missing (returns null)
    self: true,      // Searches ONLY the local component's Element Injector
    skipSelf: false  // Skips local injector and starts searching parent
  });

  // Equivalent legacy parameter decorator syntax:
  constructor(
    @Self() @Optional() private logger: LoggerService,
    @SkipSelf() private parentLogger: LoggerService,
    @Host() private hostService: LoggerService
  ) {}
}
```

| Modifier Flag | Resolution Behavior |
| :--- | :--- |
| **`@Self()` / `self: true`** | Looks **only** in the immediate component's Element Injector. Fails if not provided locally. |
| **`@SkipSelf()` / `skipSelf: true`** | Skips immediate component and begins search in the **parent** component's injector. |
| **`@Host()` / `host: true`** | Stops searching upward when reaching the host component that rendered the view. |
| **`@Optional()` / `optional: true`** | Returns `null` instead of throwing a `NullInjectorError` if dependency is not found. |

---

### Q40. How do you build a Custom Structural Directive (like `*appHasRole`) using `TemplateRef` and `ViewContainerRef`?

**Answer.** Structural directives modify DOM layout by adding, removing, or manipulating elements. They inject `TemplateRef` (the template block) and `ViewContainerRef` (the container DOM location).

```ts
import { Directive, Input, TemplateRef, ViewContainerRef, inject } from "@angular/core";
import { AuthService } from "./auth.service";

@Directive({
  selector: "[appHasRole]",
  standalone: true
})
export class HasRoleDirective {
  private templateRef = inject(TemplateRef<unknown>);
  private vcr = inject(ViewContainerRef);
  private authService = inject(AuthService);

  @Input() set appHasRole(role: string) {
    if (this.authService.hasRole(role)) {
      // Render the embedded template in the DOM container
      this.vcr.createEmbeddedView(this.templateRef);
    } else {
      // Remove template from DOM
      this.vcr.clear();
    }
  }
}
```

#### Usage in Template:
```html
<button *appHasRole="'ADMIN'">Delete System Logs</button>
```

---

### Q41. What is the difference between `@HostBinding` / `@HostListener` and the modern `host` metadata property?

**Answer.**
- **Legacy Decorators (`@HostBinding`, `@HostListener`)**: Applied to individual class properties and methods inside component/directive classes.
- **Modern `host` Metadata (`host: {}`)**: Declared directly inside `@Component({...})` or `@Directive({...})` metadata. Recommended in modern Angular for better performance, refactoring safety, and tree-shaking.

```ts
// ❌ Legacy Decorator Syntax:
export class ButtonComponent {
  @HostBinding("class.active") isActive = true;
  @HostListener("click", ["$event"])
  onClick(event: MouseEvent) { console.log(event); }
}

// ✅ Modern Host Object Metadata Syntax (Angular 15+ Standard):
@Component({
  selector: "app-button",
  standalone: true,
  host: {
    "[class.active]": "isActive()",
    "[attr.aria-disabled]": "isDisabled()",
    "(click)": "handleClick($event)"
  },
  template: `<ng-content></ng-content>`
})
export class ButtonComponent {
  isActive = signal(true);
  isDisabled = signal(false);

  handleClick(event: MouseEvent) {
    console.log("Button clicked", event);
  }
}
```

---

### Q42. What is `ControlValueAccessor` (CVA), and how do you build a custom form control for Reactive Forms?

**Answer.** **`ControlValueAccessor` (CVA)** acts as an interface/bridge between Angular Reactive Forms (`FormControl`) and a custom native/DOM UI component (e.g. custom Star Rating or Toggle Switch).

```ts
import { Component, forwardRef, signal } from "@angular/core";
import { ControlValueAccessor, NG_VALUE_ACCESSOR } from "@angular/forms";

@Component({
  selector: "app-toggle-switch",
  standalone: true,
  providers: [{
    provide: NG_VALUE_ACCESSOR,
    useExisting: forwardRef(() => ToggleSwitchComponent),
    multi: true
  }],
  template: `
    <button [class.on]="value()" (click)="toggle()" [disabled]="disabled()">
      {{ value() ? 'ON' : 'OFF' }}
    </button>
  `
})
export class ToggleSwitchComponent implements ControlValueAccessor {
  value = signal(false);
  disabled = signal(false);

  onChange: (val: boolean) => void = () => {};
  onTouched: () => void = () => {};

  // 1. Written by Reactive Form -> Custom UI Component
  writeValue(val: boolean): void {
    this.value.set(!!val);
  }

  // 2. Registers callback to notify Reactive Form when UI state changes
  registerOnChange(fn: any): void {
    this.onChange = fn;
  }

  // 3. Registers callback to notify Reactive Form when UI element is touched/blurred
  registerOnTouched(fn: any): void {
    this.onTouched = fn;
  }

  // 4. Sets disabled state from form control
  setDisabledState(isDisabled: boolean): void {
    this.disabled.set(isDisabled);
  }

  toggle() {
    if (this.disabled()) return;
    this.value.update(v => !v);
    this.onChange(this.value());   // Notify Form
    this.onTouched();              // Notify Form touched
  }
}
```

---

### Q43. How do custom synchronous validators and asynchronous validators (`AsyncValidatorFn`) work in Reactive Forms?

**Answer.**
- **Synchronous Validator (`ValidatorFn`)**: Returns `ValidationErrors | null` immediately.
- **Asynchronous Validator (`AsyncValidatorFn`)**: Returns `Observable<ValidationErrors | null>` or `Promise<ValidationErrors | null>`. Used for server HTTP verification (e.g. checking username availability).

```ts
import { AbstractControl, ValidationErrors, AsyncValidatorFn } from "@angular/forms";
import { Observable, timer, of } from "rxjs";
import { switchMap, map, catchError } from "rxjs/operators";

// 1. Synchronous Custom Validator
export function forbiddenNameValidator(forbiddenName: string) {
  return (control: AbstractControl): ValidationErrors | null => {
    const forbidden = control.value?.includes(forbiddenName);
    return forbidden ? { forbiddenName: { value: control.value } } : null;
  };
}

// 2. Asynchronous Custom Validator (e.g. HTTP Username Availability)
export function usernameAsyncValidator(userService: UserService): AsyncValidatorFn {
  return (control: AbstractControl): Observable<ValidationErrors | null> => {
    if (!control.value) return of(null);

    // Debounce HTTP requests using timer(300)
    return timer(300).pipe(
      switchMap(() => userService.checkUsername(control.value)),
      map(isTaken => (isTaken ? { usernameTaken: true } : null)),
      catchError(() => of(null))
    );
  };
}
```

---

### Q44. How do Functional Route Guards (`canActivateFn`, `CanMatchFn`) work in Angular 15+?

**Answer.** Class-based route guards (`implements CanActivate`) are deprecated. Modern Angular uses **Functional Route Guards** defined as lightweight functions using `inject()`.

```ts
import { inject } from "@angular/core";
import { CanActivateFn, Router } from "@angular/router";
import { AuthService } from "./auth.service";

export const authGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (authService.isLoggedIn()) {
    return true; // Allow route access
  }

  // Redirect to login page
  return router.createUrlTree(["/login"], { queryParams: { returnUrl: state.url } });
};
```

#### Route Configuration:
```ts
export const routes: Routes = [
  {
    path: "admin",
    component: AdminComponent,
    canActivate: [authGuard] // Pass functional guard function!
  }
];
```

---

### Q45. What is the difference between `@ViewChild` / `@ViewChildren` and `@ContentChild` / `@ContentChildren`? What is `ngProjectAs`?

**Answer.**
- **`@ViewChild` / `@ViewChildren`**: Queries elements or child components rendered **inside the template** of the current component (`templateUrl`). Available during `ngAfterViewInit()`.
- **`@ContentChild` / `@ContentChildren`**: Queries projected elements passed into the component via `<ng-content>` from a parent template. Available during `ngAfterContentInit()`.

```ts
@Component({
  selector: "app-card",
  standalone: true,
  template: `
    <div class="header"><ng-content select="[card-header]"></ng-content></div>
    <div class="body"><ng-content></ng-content></div>
  `
})
export class CardComponent implements AfterContentInit {
  // Query projected content element passed from parent
  @ContentChild("headerElement") header!: ElementRef;

  ngAfterContentInit() {
    console.log("Projected Header:", this.header.nativeElement);
  }
}
```

#### `ngProjectAs` Attribute:
Allows an element to be projected into a targeted `<ng-content select="...">` slot even if it is wrapped inside a non-matching component or container tag.

```html
<app-card>
  <!-- Overrides matching selector so it projects into [card-header] slot! -->
  <ng-container ngProjectAs="[card-header]">
    <h3>Custom Card Title</h3>
  </ng-container>
</app-card>
```

---

### Q46. How does Angular prevent XSS attacks, what is `DomSanitizer`, and when should `bypassSecurityTrustHtml` be used?

**Answer.** Angular treats all values as untrusted by default. When binding values via `[innerHTML]`, `[src]`, or `[href]`, Angular's built-in **Contextual Sanitizer** automatically sanitizes unsafe HTML tags (`<script>`, `onload=...`) and JavaScript URLs.

```ts
// Angular automatically strips <script> tags here:
htmlSnippet = `<p>Hello</p><script>alert('xss')</script>`;
```
```html
<div [innerHTML]="htmlSnippet"></div> <!-- Rendered safely as: <p>Hello</p> -->
```

#### Disabling Sanitization via `DomSanitizer`:
If you explicitly trust HTML/URLs coming from a secure backend, inject `DomSanitizer` to bypass security checks:

```ts
import { Component, inject } from "@angular/core";
import { DomSanitizer, SafeHtml } from "@angular/platform-browser";

export class TrustHtmlComponent {
  private sanitizer = inject(DomSanitizer);
  trustedHtml!: SafeHtml;

  loadTrustedSnippet(rawSvgOrHtml: string) {
    // ⚠️ WARNING: Bypasses XSS protection! Ensure input is 100% sanitized server-side!
    this.trustedHtml = this.sanitizer.bypassSecurityTrustHtml(rawSvgOrHtml);
  }
}
```

---

### Q47. What is Non-Destructive Hydration in Angular 16+, and how does Event Replaying (`withEventReplay()`) work?

**Answer.**
- **Destructive Hydration (Legacy Universal)**: On client app startup, Angular erased all server-rendered HTML nodes in the browser DOM and re-created the entire component DOM tree from scratch, causing noticeable screen flickering.
- **Non-Destructive Hydration (Angular 16+)**: Client-side Angular reuses existing server-rendered HTML nodes, attaching component logic and event listeners directly without destroying DOM nodes.

#### Event Replaying (`withEventReplay()` in Angular 18):
When a user clicks a button on an SSR page *before* the Angular JS client bundle has finished downloading, standard apps lose that click. **Event Replaying** captures user interactions (clicks, inputs) during initial page load and automatically replays them once Angular hydration completes.

```ts
// main.ts
bootstrapApplication(AppComponent, {
  providers: [
    provideClientHydration(
      withEventReplay() // Captures and replays pre-hydration user events!
    )
  ]
});
```

---

### Q48. How does Angular's Animation API work (`trigger()`, `transition()`, `state()`, `animate()`)?

**Answer.** Angular provides a declarative animation system based on Web Animations API inside `@Component({ animations: [...] })`.

```ts
import { Component, signal } from "@angular/core";
import { trigger, state, style, transition, animate } from "@angular/animations";

@Component({
  selector: "app-fade-box",
  standalone: true,
  animations: [
    trigger("fadeSlide", [
      state("void", style({ opacity: 0, transform: "translateY(-20px)" })),
      state("*", style({ opacity: 1, transform: "translateY(0)" })),
      
      // Enter animation (:enter alias for void => *)
      transition(":enter", [
        animate("300ms ease-out")
      ]),
      
      // Leave animation (:leave alias for * => void)
      transition(":leave", [
        animate("200ms ease-in", style({ opacity: 0, transform: "translateY(20px)" }))
      ])
    ])
  ],
  template: `
    <button (click)="toggle()">Toggle Box</button>
    @if (isVisible()) {
      <div @fadeSlide class="box">Animated Content</div>
    }
  `
})
export class FadeBoxComponent {
  isVisible = signal(true);
  toggle() { this.isVisible.update(v => !v); }
}
```

---

### Q49. How do Angular Workspaces and Nx Monorepos organize micro-frontends and shared library architectures?

**Answer.**
- **Angular Workspaces (`angular.json`)**: Allows managing multiple applications (`projects/app-shell`, `projects/admin-portal`) and shared libraries (`projects/shared-ui`) inside a single repository.
- **Nx Monorepo**: Extends Angular workspaces with advanced dependency graphs, distributed computation caching, affected build commands (`nx affected:test`), and boundary linting rules (`@nx/enforce-module-boundaries`).

#### Architectural Layers in Monorepos:
1. **`apps/`**: Executable Angular applications (shell, features).
2. **`libs/feature/`**: Smart container components and route definitions.
3. **`libs/ui/`**: Presentational (dumb) components.
4. **`libs/data-access/`**: Services, NgRx state, HTTP API callers.
5. **`libs/utils/`**: Pure functions, helpers, models.

---

### Q50. How does Module Federation work in Angular Micro-Frontend architectures?

**Answer.** **Webpack / Rspack Module Federation** allows an Angular application (Shell) to dynamically load independently compiled and deployed Angular applications (Remotes) at runtime over HTTP without bundling them together at compile time.

```ts
// app.routes.ts in Shell Application
import { loadRemoteModule } from "@angular-architects/module-federation";
import { Routes } from "@angular/router";

export const routes: Routes = [
  {
    path: "dashboard",
    loadChildren: () =>
      loadRemoteModule({
        type: "module",
        remoteEntry: "http://localhost:4201/remoteEntry.js", // Remote app deployed independently!
        exposedModule: "./Module"
      }).then(m => m.DashboardModule)
  }
];
```

#### Key Benefits of Micro-Frontends:
- **Independent Deployment**: Remote apps can be updated and deployed to production without re-building or re-deploying the main Shell app.
- **Team Autonomy**: Multiple squads can own separate domain repositories independently while sharing core platform shell layouts.
