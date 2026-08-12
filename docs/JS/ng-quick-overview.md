# Comprehensive Angular Architecture & Interview Cheat Sheet

This guide provides a detailed breakdown of an Angular project's file structure, execution flow, core concepts, modern Angular paradigms (v16–v19), forms, component communication, performance optimization, and testing to help you ace technical interviews.

---

### 1. Application Execution & Bootstrap Flow

Understanding how Angular starts and renders a page is a classic interview topic.

```
[Browser Request]
       │
       ▼
 📄 index.html  ──────► Contains <app-root></app-root> host element
       │
       ▼
 ⚙️ main.ts     ──────► JavaScript Entry Point. Calls bootstrapApplication(AppComponent, appConfig)
       │
       ▼
 🛠️ app.config.ts ────► Configures global providers (Routing, HttpClient, Animations, State)
       │
       ▼
 🧩 app.ts      ──────► Root Component (AppComponent) initializes and targets <app-root>
       │
       ▼
 🖼️ app.html    ──────► Root Template renders inside <app-root>
```

---

### 2. Key File Descriptions

#### Core Application Files (`src/`)

| File / Location | Description & Purpose | Interview Takeaways |
| :--- | :--- | :--- |
| **`src/index.html`** | The main HTML page of the single-page application (SPA). It includes standard metadata, scripts, and the host element `<app-root></app-root>`. | Browser only loads this HTML once. All views render inside `<app-root>`. |
| **`src/main.ts`** | The main TypeScript entry point executed when the app launches. | Uses `bootstrapApplication(AppComponent, appConfig)` in standalone Angular applications. |
| **`src/app/app.config.ts`** | The central provider configuration file for standalone Angular apps (replaces `AppModule` providers). | Configures `provideRouter(routes)`, `provideHttpClient()`, `provideAnimations()`, etc. |
| **`src/app/app.routes.ts`** | Defines navigation routes mapping URL paths to components (`{ path: 'dashboard', component: DashboardComponent }`). | Supports lazy loading via `loadComponent: () => import(...)`. |
| **`src/app/app.ts`** (or `app.component.ts`) | The root component of the application (`@Component`). | Root component logic; holds top-level layout (e.g., navbar, footer, `<router-outlet>`). |
| **`src/app/app.html`** | HTML template for the root component. | Uses `<router-outlet></router-outlet>` to dynamically render child routes. |
| **`src/styles.css`** | Global stylesheet applied to the entire application. | Component-specific styles do NOT affect global styles due to View Encapsulation. |

---

#### Key Angular Building Blocks & Files

#### 1. Components (`*.component.ts`)
- **Purpose**: Controls a portion of the UI screen (template + logic + styles).
- **Metadata**: Defined using `@Component({ selector, templatePath/template, styleUrl, standalone: true })`.
- **Standalone Mode**: In modern Angular, components are standalone by default, importing required directives/components directly in `imports: [...]`.

#### 2. Services (`*.service.ts`)
- **Purpose**: Encapsulates reusable business logic, API calls, data sharing, and state management.
- **Decorator**: `@Injectable({ providedIn: 'root' })`.
- **Injection**: Can be injected using `inject(MyService)` or via constructor `constructor(private myService: MyService)`.

#### 3. Guards (`*.guard.ts`)
- **Purpose**: Protects routes from unauthorized access or prevents leaving dirty forms (`CanActivateFn`, `CanDeactivateFn`).
- **Modern Syntax**: Functional guards returning `boolean`, `UrlTree`, `Observable`, or `Promise`.

#### 4. Interceptors (`*.interceptor.ts`)
- **Purpose**: Intercepts HTTP requests and responses globally (e.g., adding Authorization Bearer tokens, global error handling, logging).
- **Modern Syntax**: Functional interceptors (`HttpInterceptorFn`).

#### 5. Directives (`*.directive.ts`)
- **Purpose**: Adds custom behavior to DOM elements.
  - **Attribute Directives**: Changes appearance or behavior (e.g., `ngClass`, `ngStyle`, custom tooltip directive).
  - **Structural Directives**: Manipulates DOM structure (e.g., `*ngIf`, `*ngFor` in legacy code, or custom structural directives).

#### 6. Pipes (`*.pipe.ts`)
- **Purpose**: Transforms data format within templates without mutating original values (e.g., `{{ price | currency }}`, `{{ date | date:'short' }}`).
- **Pure vs Impure Pipes**: Pure pipes (default) execute only when input value reference changes. Impure pipes (`pure: false`) execute on every change detection cycle (can affect performance).

---

## 3. Root & Configuration Files

| Configuration File | Primary Responsibilities |
| :--- | :--- |
| **`angular.json`** | Main CLI configuration file. Defines build targets (`build`, `serve`, `test`), asset directories, global CSS/JS file includes, compiler options, and environment setups. |
| **`package.json`** | Lists npm package dependencies, dev dependencies, and CLI scripts (`npm start`, `npm run build`, `npm test`). |
| **`tsconfig.json`** | Root TypeScript compiler configuration (strict mode, module resolution, target JS version). |
| **`tsconfig.app.json`** | App-specific TypeScript config extending `tsconfig.json` for compilation of `src/`. |
| **`tsconfig.spec.json`** | TypeScript config specifically for running unit tests (`*.spec.ts`). |

---

## 4. Component Communication & Content Projection

### A. Data Passing Techniques

| Scenario | Modern Signal Syntax | Traditional Decorator Syntax |
| :--- | :--- | :--- |
| **Parent to Child** | `userId = input<string>();` | `@Input() userId!: string;` |
| **Child to Parent** | `userSelected = output<User>();` | `@Output() userSelected = new EventEmitter<User>();` |
| **Parent Querying Child DOM/Component** | `childComp = viewChild(ChildComponent);` | `@ViewChild(ChildComponent) childComp!: ChildComponent;` |

### B. `<ng-container>`, `<ng-template>`, and `<ng-content>` (Frequent Interview Topic)

- **`<ng-container>`**: A logical grouping element that Angular does NOT render into the DOM (no wrapper `div` pollution).
- **`<ng-template>`**: A template block that is NOT rendered by default. Used for deferred or conditional rendering (e.g., with `ngTemplateOutlet`).
- **`<ng-content>`**: Used for **Content Projection** (similar to slots in web components/Vue).
  ```html
  <!-- Single Slot -->
  <ng-content></ng-content>
  
  <!-- Multi-Slot Projection -->
  <ng-content select="[card-header]"></ng-content>
  <ng-content select="[card-body]"></ng-content>
  ```

---

## 5. Forms in Angular: Reactive vs. Template-Driven

### Comparison

| Feature | Reactive Forms (`ReactiveFormsModule`) | Template-Driven Forms (`FormsModule`) |
| :--- | :--- | :--- |
| **Setup** | Programmatic (`FormGroup`, `FormControl`) | Template directive (`[(ngModel)]`) |
| **Data Flow** | Explicit, immutable reactive data flow | Two-way data binding (mutable) |
| **Testing** | Easy to unit test without DOM | Requires DOM rendering to test |
| **Complex Validation** | Synchronous & Asynchronous custom functions | Custom directives |

### Advanced Concept: `ControlValueAccessor` (CVA)
- **Question**: "How do you build a custom form component that works with `formControlName` or `ngModel`?"
- **Answer**: Implement the `ControlValueAccessor` interface:
  1. `writeValue(obj: any)`: Angular passes model value to the component.
  2. `registerOnChange(fn: any)`: Registers callback when value changes in UI.
  3. `registerOnTouched(fn: any)`: Registers callback when control loses focus.
  4. `setDisabledState?(isDisabled: boolean)`: Handles control disabled state.
  5. Provide `NG_VALUE_ACCESSOR` in component providers array.

---

## 6. Modern Angular Paradigms (v16 – v19)

### A. Standalone Components vs. NgModules
- **Legacy (NgModules)**: Everything had to belong to an `@NgModule({ declarations: [...], imports: [...], providers: [...], exports: [...] })`.
- **Modern (Standalone)**: Components are standalone (`standalone: true`). They declare their imports directly in `@Component({ imports: [CommonModule, ReactiveFormsModule] })`. `AppModule` is completely eliminated.

### B. Signals vs. RxJS Observables

```typescript
// Signal Declaration
count = signal(0);
doubleCount = computed(() => this.count() * 2);

// Updates
this.count.set(5);
this.count.update(c => c + 1);

// Reaction
effect(() => console.log('Count updated:', this.count()));
```

- **Signals**: Synchronous state primitives with fine-grained DOM tracking.
- **RxJS Observables**: Asynchronous stream handling (HTTP, WebSockets, timed events).
- **Interoperability**:
  - `toSignal(observable$)`: Converts Observable to Signal.
  - `toObservable(signal)`: Converts Signal to Observable.

### C. Built-in Control Flow (Angular 17+)

```html
<!-- Native Control Flow -->
@if (isLoggedIn()) {
  <user-profile />
} @else {
  <login-form />
}

@for (user of users(); track user.id) {
  <p>{{ user.name }}</p>
} @empty {
  <p>No users found</p>
}
```

### D. Deferrable Views (`@defer`)
Lazy-load UI blocks on demand:
```html
@defer (on viewport) {
  <heavy-chart-component />
} @placeholder {
  <p>Scroll down to load chart...</p>
} @loading (minimum 500ms) {
  <spinner />
}
```

---

## 7. Performance Optimization Techniques

1. **Lazy Loading Routes**: Use `loadComponent` or `loadChildren` to split JavaScript bundles per route.
2. **`OnPush` Change Detection Strategy**: Set `changeDetection: ChangeDetectionStrategy.OnPush` on components to avoid unnecessary change detection runs.
3. **Loop Tracking (`track`)**: Always provide a unique tracking key in `@for (item of items; track item.id)` to avoid re-rendering entire lists.
4. **`NgOptimizedImage` (`ngSrc`)**: Handles automatic responsive sizing, lazy loading, preloading key hero images, and CDN optimizations.
5. **Deferrable Views (`@defer`)**: Postpones downloading and rendering non-critical component bundles until visible or triggered.
6. **Pure Pipes**: Ensure pipes remain pure so Angular caches calculation results per reference.

---

## 8. Angular Unit Testing

- **Testing Frameworks**: Karma/Jasmine (traditional) or Vitest (modern CLI default).
- **Key Concepts**:
  - **`TestBed`**: Configures and initializes the Angular testing module.
  - **`ComponentFixture`**: Provides access to component instance and DOM debug elements.
  - **`HttpTestingController`**: Mocks HTTP requests in service unit tests.
  - **`fakeAsync()` + `tick()` / `flush()`**: Simulates passing of time for async code in unit tests.

---

## 9. Top Interview Questions & Quick Answers

1. **How does Change Detection work in Angular?**
   - Angular checks if model data changed and updates the DOM.
   - **Default**: Checks the entire component tree from top to bottom (historically powered by `zone.js`).
   - **OnPush**: Component is checked ONLY when inputs change by reference, an event fires from the component, or an Observable/Signal emits.
   - **Zoneless / Signals**: Fine-grained change detection updating only affected DOM nodes without checking full component trees.

2. **What is Dependency Injection (DI) & Injector Hierarchy?**
   - A design pattern where a class receives dependencies from Angular's DI system.
   - **Injector Hierarchy**:
     1. `ElementInjector` (Component level)
     2. `EnvironmentInjector` / `RootInjector` (`providedIn: 'root'`)
     3. `NullInjector` (Throws "No provider for..." error if not found).

3. **Difference between `switchMap`, `mergeMap`, `concatMap`, `exhaustMap`?**
   - **`switchMap`**: Cancels previous pending HTTP request when a new emission arrives (ideal for search inputs).
   - **`mergeMap`**: Runs all requests concurrently (ideal for parallel calls where order does not matter).
   - **`concatMap`**: Runs requests sequentially in order (ideal for queue operations).
   - **`exhaustMap`**: Ignores new emissions while a request is pending (ideal for submit buttons).

4. **What is the difference between `Subject`, `BehaviorSubject`, and `ReplaySubject`?**
   - **`Subject`**: No initial value. Subscribers only receive notifications emitted *after* subscription.
   - **`BehaviorSubject`**: Requires initial value. Holds latest value and immediately emits it to new subscribers (`getValue()`).
   - **`ReplaySubject`**: Replays specified number of past buffer values to new subscribers regardless of when they subscribe.
