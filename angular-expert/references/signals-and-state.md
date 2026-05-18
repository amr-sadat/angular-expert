# Signals & State Management

## Table of Contents
- [Signal Primitives](#signal-primitives)
- [Computed Signals](#computed-signals)
- [Linked Signals](#linked-signals)
- [Effects](#effects)
- [RxJS Interop](#rxjs-interop)
- [State Management Patterns](#state-management-patterns)

---

## Signal Primitives

A `signal()` is a reactive value container. Reading it in a reactive context (template, `computed`, `effect`) creates a dependency.

```typescript
import { signal } from '@angular/core';

// Create
const count = signal(0);

// Read
console.log(count()); // 0

// Write
count.set(5);

// Update based on current value
count.update(c => c + 1);
```

Key rules:
- Use `set()` to replace the value entirely
- Use `update()` when the new value depends on the current value
- Do NOT use `mutate()` — it does not exist in modern Angular. For objects/arrays, create new references with `update()`

```typescript
// Updating objects — always create a new reference
const user = signal<User>({ name: 'Alice', age: 30 });
user.update(u => ({ ...u, age: u.age + 1 }));

// Updating arrays — always create a new reference
const items = signal<string[]>(['a', 'b']);
items.update(list => [...list, 'c']);
```

### Equality

Signals use `Object.is()` for equality by default. Provide a custom equality function to control when subscribers are notified:

```typescript
const position = signal(
  { x: 0, y: 0 },
  { equal: (a, b) => a.x === b.x && a.y === b.y }
);
```

## Computed Signals

`computed()` derives a read-only signal from other signals. It recalculates lazily — only when read and only when a dependency has changed.

```typescript
import { signal, computed } from '@angular/core';

const price = signal(100);
const quantity = signal(3);
const taxRate = signal(0.1);

const subtotal = computed(() => price() * quantity());
const tax = computed(() => subtotal() * taxRate());
const total = computed(() => subtotal() + tax());
```

`computed()` is pure — it should not have side effects. Angular tracks dependencies automatically; there is no need to declare them.

### When to use computed vs linkedSignal

| Need | Use |
|---|---|
| Read-only derived value | `computed()` |
| Derived value that can also be written to | `linkedSignal()` |
| Derived value that needs the previous value | `linkedSignal()` |

## Linked Signals

`linkedSignal()` creates a writable signal whose value is derived from a source signal. When the source changes, the linked signal recomputes. But unlike `computed()`, you can also `set()` or `update()` it directly.

```typescript
import { signal, linkedSignal } from '@angular/core';

const items = signal(['Apple', 'Banana', 'Cherry']);

// Simple form — resets to first item when items change
const selectedItem = linkedSignal(() => items()[0]);

// Advanced form — access previous value
const selectedItem = linkedSignal<string[], string>({
  source: items,
  computation: (newItems, previous) => {
    // Keep previous selection if it still exists in the new list
    if (previous && newItems.includes(previous.value)) {
      return previous.value;
    }
    return newItems[0];
  },
});
```

Common use cases:
- **Form defaults that reset** when a parent value changes
- **Selected item** in a list that persists through list updates
- **Chat history accumulation** from a streaming data source

## Effects

`effect()` runs a callback whenever any signal it reads changes. Use it for side effects that synchronize with external systems.

```typescript
import { effect, signal } from '@angular/core';

@Component({ /* ... */ })
export class SettingsComponent {
  theme = signal<'light' | 'dark'>('light');

  constructor() {
    // Sync theme to localStorage
    effect(() => {
      localStorage.setItem('theme', this.theme());
    });

    // Sync theme to DOM
    effect(() => {
      document.documentElement.setAttribute('data-theme', this.theme());
    });
  }
}
```

### Effect rules

1. **Prefer `computed()` or `linkedSignal()` for derived state** — while signal writes inside `effect()` are technically allowed in v20 (`allowSignalWrites` is deprecated — writes are permitted by default), writing to signals from effects creates implicit data flows that are harder to trace and debug.
2. Effects run at least once after creation.
3. Effects are cleaned up automatically when the owning context (component, service) is destroyed.
4. Effects run asynchronously, after change detection.

### Cleanup in effects

Return a cleanup function or use `onCleanup`:

```typescript
effect((onCleanup) => {
  const id = setInterval(() => console.log(this.count()), 1000);
  onCleanup(() => clearInterval(id));
});
```

### untracked()

Read a signal inside an effect without creating a dependency:

```typescript
import { untracked } from '@angular/core';

effect(() => {
  const name = this.userName();
  // Read analyticsId without triggering re-run when it changes
  const id = untracked(() => this.analyticsId());
  this.analytics.track(name, id);
});
```

## RxJS Interop

### toSignal — Observable to Signal

```typescript
import { toSignal } from '@angular/core/rxjs-interop';
import { inject } from '@angular/core';
import { ActivatedRoute } from '@angular/router';

@Component({ /* ... */ })
export class DetailComponent {
  private route = inject(ActivatedRoute);

  // Convert route params Observable to Signal
  params = toSignal(this.route.params, { initialValue: {} });

  // With requireSync for synchronous sources (like BehaviorSubject)
  // authState = toSignal(this.auth.state$, { requireSync: true });
}
```

### toObservable — Signal to Observable

```typescript
import { toObservable } from '@angular/core/rxjs-interop';
import { switchMap, debounceTime } from 'rxjs';

@Component({ /* ... */ })
export class SearchComponent {
  query = signal('');

  results = toSignal(
    toObservable(this.query).pipe(
      debounceTime(300),
      switchMap(q => this.searchService.search(q)),
    ),
    { initialValue: [] },
  );
}
```

### takeUntilDestroyed

Automatically unsubscribes when the component/service is destroyed:

```typescript
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

constructor() {
  this.router.events
    .pipe(takeUntilDestroyed())
    .subscribe(event => { /* ... */ });
}
```

Must be called in an injection context (constructor or field initializer). If used elsewhere, pass a `DestroyRef`:

```typescript
private destroyRef = inject(DestroyRef);

ngOnInit() {
  this.source$.pipe(
    takeUntilDestroyed(this.destroyRef)
  ).subscribe();
}
```

## Output Interop

### outputFromObservable — Observable to Output

Declare a component output that emits values from an RxJS Observable. Angular manages the subscription lifecycle automatically.

```typescript
import { outputFromObservable } from '@angular/core/rxjs-interop';
import { interval, map } from 'rxjs';

@Component({
  selector: 'app-timer',
  template: `<p>Timer running...</p>`,
})
export class TimerComponent {
  tick = outputFromObservable(
    interval(1000).pipe(map(i => i + 1))
  );
}
```

Parent usage:

```html
<app-timer (tick)="onTick($event)" />
```

### outputToObservable — Output to Observable

Convert an Angular `OutputRef` into an RxJS Observable for use with operators:

```typescript
import { outputToObservable } from '@angular/core/rxjs-interop';
import { debounceTime } from 'rxjs';

@Component({ /* ... */ })
export class ParentComponent {
  @ViewChild(SearchComponent) search!: SearchComponent;

  constructor() {
    afterNextRender(() => {
      outputToObservable(this.search.queryChange).pipe(
        debounceTime(300),
        takeUntilDestroyed(this.destroyRef),
      ).subscribe(query => this.performSearch(query));
    });
  }
}
```

## State Management Patterns

### Signal-based store service

For shared state across components, create a service with signals:

```typescript
import { Injectable, signal, computed } from '@angular/core';

export interface Todo {
  id: number;
  title: string;
  completed: boolean;
}

@Injectable({ providedIn: 'root' })
export class TodoStore {
  // Private writable state
  private readonly _todos = signal<Todo[]>([]);

  // Public read-only projections
  readonly todos = this._todos.asReadonly();
  readonly completedCount = computed(() =>
    this._todos().filter(t => t.completed).length
  );
  readonly pendingCount = computed(() =>
    this._todos().filter(t => !t.completed).length
  );

  addTodo(title: string): void {
    this._todos.update(todos => [
      ...todos,
      { id: Date.now(), title, completed: false },
    ]);
  }

  toggleTodo(id: number): void {
    this._todos.update(todos =>
      todos.map(t => (t.id === id ? { ...t, completed: !t.completed } : t))
    );
  }

  removeTodo(id: number): void {
    this._todos.update(todos => todos.filter(t => t.id !== id));
  }
}
```

### Pattern: Feature state with HttpClient + signals

Combine `HttpClient` with `toSignal()` for reactive data fetching in a store:

```typescript
import { toSignal, toObservable } from '@angular/core/rxjs-interop';
import { switchMap } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class ProductStore {
  private http = inject(HttpClient);
  private readonly _selectedCategory = signal<string | null>(null);

  readonly selectedCategory = this._selectedCategory.asReadonly();

  readonly products = toSignal(
    toObservable(this._selectedCategory).pipe(
      switchMap(cat => {
        const url = cat ? `/api/products?category=${cat}` : '/api/products';
        return this.http.get<Product[]>(url);
      }),
    ),
    { initialValue: [] },
  );

  selectCategory(category: string | null): void {
    this._selectedCategory.set(category);
  }
}
```

### Pattern: Optimistic updates

```typescript
@Injectable({ providedIn: 'root' })
export class TaskStore {
  private http = inject(HttpClient);
  private _tasks = signal<Task[]>([]);
  readonly tasks = this._tasks.asReadonly();

  async completeTask(id: string): Promise<void> {
    // Optimistic update
    const previous = this._tasks();
    this._tasks.update(tasks =>
      tasks.map(t => (t.id === id ? { ...t, completed: true } : t))
    );

    try {
      await firstValueFrom(
        this.http.patch(`/api/tasks/${id}`, { completed: true })
      );
    } catch {
      // Rollback on failure
      this._tasks.set(previous);
    }
  }
}
```