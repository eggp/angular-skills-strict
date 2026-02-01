---
name: angular-signals
description: Implement signal-based reactive state management in Angular v20+. Use for creating reactive state with signal(), derived state with computed(), dependent state with linkedSignal(), and side effects with effect(). Triggers on state management questions, converting from BehaviorSubject/Observable patterns to signals, or implementing reactive data flows.
---

# Angular Signals

Signals are Angular's reactive primitive for state management. They provide synchronous, fine-grained reactivity.

## Core Signal APIs

### signal() - Writable State

```typescript
import { signal } from '@angular/core';

// Create writable signal
const count = signal(0);

// Read value
console.log(count()); // 0

// Set new value
count.set(5);

// Update based on current value
count.update(c => c + 1);

// With explicit type
const user = signal<User | null>(null);
user.set({ id: 1, name: 'Alice' });
```

### computed() - Derived State

```typescript
import { signal, computed } from '@angular/core';

const firstName = signal('John');
const lastName = signal('Doe');

// Derived signal - automatically updates when dependencies change
const fullName = computed(() => `${firstName()} ${lastName()}`);

console.log(fullName()); // "John Doe"
firstName.set('Jane');
console.log(fullName()); // "Jane Doe"

// Computed with complex logic
const items = signal<Item[]>([]);
const filter = signal('');

const filteredItems = computed(() => {
  const query = filter().toLowerCase();
  return items().filter(item => 
    item.name.toLowerCase().includes(query)
  );
});

const totalPrice = computed(() => 
  filteredItems().reduce((sum, item) => sum + item.price, 0)
);
```

### linkedSignal() - Dependent State with Reset

```typescript
import { signal, linkedSignal } from '@angular/core';

const options = signal(['A', 'B', 'C']);

// Resets to first option when options change
const selected = linkedSignal(() => options()[0]);

console.log(selected()); // "A"
selected.set('B');       // User selects B
console.log(selected()); // "B"
options.set(['X', 'Y']); // Options change
console.log(selected()); // "X" - auto-reset to first

// With previous value access
const items = signal<Item[]>([]);

const selectedItem = linkedSignal<Item[], Item | null>({
  source: () => items(),
  computation: (newItems, previous) => {
    // Try to preserve selection if item still exists
    const prevItem = previous?.value;
    if (prevItem && newItems.some(i => i.id === prevItem.id)) {
      return prevItem;
    }
    return newItems[0] ?? null;
  },
});
```

### effect() - Side Effects

```typescript
import { signal, effect, inject, DestroyRef } from '@angular/core';

@Component({...})
export class Search {
  protected readonly query = signal('');
  
  constructor() {
    // Effect runs when query changes
    effect(() => {
      console.log('Search query:', this.query());
    });
    
    // Effect with cleanup
    effect((onCleanup) => {
      const timer = setInterval(() => {
        console.log('Current query:', this.query());
      }, 1000);
      
      onCleanup(() => clearInterval(timer));
    });

    // Self-destroying effect (Performance optimization)
    const effectRef = effect(() => {
      if (this.check()) {
        // ...
        effectRef.destroy();
      }
    });
  }

  protected check(): boolean {
    return true;
  }
}
```

**Effect rules:**
- Run in injection context (constructor or with `runInInjectionContext`)
- Automatically cleaned up when component destroys

## Component State Pattern

```typescript
@Component({
  selector: 'app-todo-list',
  template: `
    <input [value]="newTodo()" (input)="newTodo.set($any($event.target).value)" />
    <button (click)="addTodo()" [disabled]="!canAdd()">Add</button>
    
    <ul>
      @let visibleTodos = filteredTodos();
      @for (todo of visibleTodos; track todo.id) {
        <li [class.done]="todo.done">
          {{ todo.text }}
          <button (click)="toggleTodo(todo.id)">Toggle</button>
        </li>
      }
    </ul>
    
    <p>{{ remaining() }} remaining</p>
  `,
})
export class TodoList {
  // State
  protected readonly todos = signal<Todo[]>([]);
  protected readonly newTodo = signal('');
  protected readonly filter = signal<'all' | 'active' | 'done'>('all');
  
  // Derived state
  protected readonly canAdd = computed(() => this.newTodo().trim().length > 0);
  
  protected readonly filteredTodos = computed(() => {
    const todos = this.todos();
    switch (this.filter()) {
      case 'active': return todos.filter(t => !t.done);
      case 'done': return todos.filter(t => t.done);
      default: return todos;
    }
  });
  
  protected readonly remaining = computed(() =>
    this.todos().filter(t => !t.done).length
  );
  
  // Actions
  protected addTodo() {
    const text = this.newTodo().trim();
    if (text) {
      this.todos.update(todos => [
        ...todos,
        { id: crypto.randomUUID(), text, done: false }
      ]);
      this.newTodo.set('');
    }
  }
  
  protected toggleTodo(id: string) {
    this.todos.update(todos =>
      todos.map(t => t.id === id ? { ...t, done: !t.done } : t)
    );
  }
}
```

## RxJS Interop

### toSignal() - Observable to Signal

```typescript
import { toSignal } from '@angular/core/rxjs-interop';
import { interval } from 'rxjs';

@Component({...})
export class Timer {
  readonly #http = inject(HttpClient);
  
  // From observable - requires initial value or allowUndefined
  protected readonly counter = toSignal(interval(1000), { initialValue: 0 });
  
  // From HTTP - undefined until loaded
  protected readonly users = toSignal(this.#http.get<User[]>('/api/users'));
  
  // With requireSync for synchronous observables (BehaviorSubject)
  readonly #user$ = new BehaviorSubject<User | null>(null);
  protected readonly currentUser = toSignal(this.#user$, { requireSync: true });
}
```

### toObservable() - Signal to Observable

```typescript
import { toObservable } from '@angular/core/rxjs-interop';
import { switchMap, debounceTime } from 'rxjs';

@Component({...})
export class Search {
  protected readonly query = signal('');
  
  readonly #http = inject(HttpClient);
  
  // Convert signal to observable for RxJS operators
  protected readonly results = toSignal(
    toObservable(this.query).pipe(
      debounceTime(300),
      switchMap(q => this.#http.get<Result[]>(`/api/search?q=${q}`))
    ),
    { initialValue: [] }
  );
}
```

## Signal Equality

```typescript
// Custom equality function
const user = signal<User>(
  { id: 1, name: 'Alice' },
  { equal: (a, b) => a.id === b.id }
);

// Only triggers updates when ID changes
user.set({ id: 1, name: 'Alice Updated' }); // No update
user.set({ id: 2, name: 'Bob' }); // Triggers update
```

## Untracked Reads

```typescript
import { untracked } from '@angular/core';

const a = signal(1);
const b = signal(2);

// Only depends on 'a', not 'b'
const result = computed(() => {
  const aVal = a();
  const bVal = untracked(() => b());
  return aVal + bVal;
});
```

## Service State Pattern

```typescript
@Injectable({ providedIn: 'root' })
export class Auth {
  // Private writable state
  readonly #user = signal<User | null>(null);
  readonly #loading = signal(false);
  
  // Public read-only signals
  readonly user = this.#user.asReadonly();
  readonly loading = this.#loading.asReadonly();
  readonly isAuthenticated = computed(() => this.#user() !== null);
  
  readonly #http = inject(HttpClient);
  
  async login(credentials: Credentials): Promise<void> {
    this.#loading.set(true);
    try {
      const user = await firstValueFrom(
        this.#http.post<User>('/api/login', credentials)
      );
      this.#user.set(user);
    } finally {
      this.#loading.set(false);
    }
  }
  
  logout(): void {
    this.#user.set(null);
  }
}
```

For advanced patterns including resource(), see [references/signal-patterns.md](references/signal-patterns.md).
