---
name: angular-component
description: Create modern Angular standalone components following v20+ best practices. Use for building UI components with signal-based inputs/outputs, OnPush change detection, host bindings, content projection, and lifecycle hooks. Triggers on component creation, refactoring class-based inputs to signals, adding host bindings, or implementing accessible interactive components.
---

# Angular Component

Create standalone components for Angular v20+. Components are standalone by default—do NOT set `standalone: true`.

## Component Structure

```typescript
import { Component, ChangeDetectionStrategy, input, output, computed } from '@angular/core';

@Component({
  selector: 'app-user-card',
  changeDetection: ChangeDetectionStrategy.OnPush,
  host: {
    'class': 'user-card',
    '[class.active]': 'isActive()',
    '(click)': 'handleClick()',
  },
  template: `
    <img [src]="avatarUrl()" [alt]="name() + ' avatar'" />
    <h2>{{ name() }}</h2>
    @if (showEmail()) {
      <p>{{ email() }}</p>
    }
  `,
  styles: `
    :host { display: block; }
    :host.active { border: 2px solid blue; }
  `,
})
export class UserCard {
  // Required input
  readonly name = input.required<string>();
  
  // Optional input with default
  readonly email = input<string>('');
  readonly showEmail = input(false);
  
  // Input with transform
  readonly isActive = input(false, { transform: booleanAttribute });
  
  // Computed from inputs
  protected readonly avatarUrl = computed(() => `https://api.example.com/avatar/${this.name()}`);
  
  // Output
  readonly selected = output<string>();
  
  protected handleClick() {
    this.selected.emit(this.name());
  }
}
```

## Signal Inputs

```typescript
// Required - must be provided by parent
readonly name = input.required<string>();

// Optional with default value
readonly count = input(0);

// Optional without default (undefined allowed)
readonly label = input<string>();

// With alias for template binding
readonly size = input('medium', { alias: 'buttonSize' });

// With transform function
readonly disabled = input(false, { transform: booleanAttribute });
readonly value = input(0, { transform: numberAttribute });
```

## Signal Outputs

```typescript
import { output, outputFromObservable } from '@angular/core';

// Basic output
readonly clicked = output<void>();
readonly selected = output<Item>();

// With alias
readonly valueChange = output<number>({ alias: 'change' });

// From Observable (for RxJS interop)
readonly #scroll$ = new Subject<number>();
readonly scrolled = outputFromObservable(this.#scroll$);

// Emit values
this.clicked.emit();
this.selected.emit(item);
```

## Host Bindings

Use the `host` object in `@Component`—do NOT use `@HostBinding` or `@HostListener` decorators.

```typescript
@Component({
  selector: 'app-button',
  host: {
    // Static attributes
    'role': 'button',
    
    // Dynamic class bindings
    '[class.primary]': 'variant() === "primary"',
    '[class.disabled]': 'disabled()',
    
    // Dynamic style bindings
    '[style.--btn-color]': 'color()',
    
    // Attribute bindings
    '[attr.aria-disabled]': 'disabled()',
    '[attr.tabindex]': 'disabled() ? -1 : 0',
    
    // Event listeners
    '(click)': 'onClick($event)',
    '(keydown.enter)': 'onClick($event)',
    '(keydown.space)': 'onClick($event)',
  },
  template: `<ng-content />`,
})
export class Button {
  readonly variant = input<'primary' | 'secondary'>('primary');
  readonly disabled = input(false, { transform: booleanAttribute });
  readonly color = input('#007bff');
  
  readonly clicked = output<void>();
  
  protected onClick(event: Event) {
    if (!this.disabled()) {
      this.clicked.emit();
    }
  }
}
```

## Content Projection

```typescript
@Component({
  selector: 'app-card',
  template: `
    <header>
      <ng-content select="[card-header]" />
    </header>
    <main>
      <ng-content />
    </main>
    <footer>
      <ng-content select="[card-footer]" />
    </footer>
  `,
})
export class Card {}

// Usage:
// <app-card>
//   <h2 card-header>Title</h2>
//   <p>Main content</p>
//   <button card-footer>Action</button>
// </app-card>
```

## Lifecycle Hooks

```typescript
import { OnDestroy, OnInit, afterNextRender, afterRender } from '@angular/core';

export class My implements OnInit, OnDestroy {
  constructor() {
    // For DOM manipulation after render (SSR-safe)
    afterNextRender(() => {
      // Runs once after first render
    });

    afterRender(() => {
      // Runs after every render
    });
  }

  ngOnInit() { /* Component initialized */ }
  ngOnDestroy() { /* Cleanup */ }
}
```

## Accessibility Requirements

Components MUST:
- Pass AXE accessibility checks
- Meet WCAG AA standards
- Include proper ARIA attributes for interactive elements
- Support keyboard navigation
- Maintain visible focus indicators

```typescript
@Component({
  selector: 'app-toggle',
  host: {
    'role': 'switch',
    '[attr.aria-checked]': 'checked()',
    '[attr.aria-label]': 'label()',
    'tabindex': '0',
    '(click)': 'toggle()',
    '(keydown.enter)': 'toggle()',
    '(keydown.space)': 'toggle(); $event.preventDefault()',
  },
  template: `<span class="toggle-track"><span class="toggle-thumb"></span></span>`,
})
export class Toggle {
  readonly label = input.required<string>();
  readonly checked = input(false, { transform: booleanAttribute });
  readonly checkedChange = output<boolean>();
  
  protected toggle() {
    this.checkedChange.emit(!this.checked());
  }
}
```

## Template Syntax

Use native control flow—do NOT use `*ngIf`, `*ngFor`, `*ngSwitch`.

```html
<!-- Conditionals -->
@if (isLoading()) {
  <app-spinner />
} @else if (error()) {
  <app-error [message]="error()" />
} @else {
  <app-content [data]="data()" />
}

<!-- Loops -->
@for (item of items(); track item.id) {
  <app-item [item]="item" />
} @empty {
  <p>No items found</p>
}

<!-- Switch -->
@switch (status()) {
  @case ('pending') { <span>Pending</span> }
  @case ('active') { <span>Active</span> }
  @default { <span>Unknown</span> }
}
```

## Template Variables

Use `@let` to declare local variables in your template. This is especially useful for caching signal values to avoid multiple executions.

```html
<!-- Caching signal value -->
@let user = currentUser();

<div class="profile">
  <img [src]="user.avatarUrl" />
  <h3>{{ user.name }}</h3>
  <p>{{ user.bio }}</p>
</div>

<!-- Basic usage -->
@let status = isActive() ? 'Active' : 'Inactive';
<span [title]="status">{{ status }}</span>
```

## Class and Style Bindings

Do NOT use `ngClass` or `ngStyle`. Use direct bindings:

```html
<!-- Class bindings -->
<div [class.active]="isActive()">Single class</div>
<div [class]="classString()">Class string</div>

<!-- Style bindings -->
<div [style.color]="textColor()">Styled text</div>
<div [style.width.px]="width()">With unit</div>
```

## Images

Use `NgOptimizedImage` for static images:

```typescript
import { NgOptimizedImage } from '@angular/common';

@Component({
  imports: [NgOptimizedImage],
  template: `
    <img ngSrc="/assets/hero.jpg" width="800" height="600" priority />
    <img [ngSrc]="imageUrl()" width="200" height="200" />
  `,
})
export class Hero {
  readonly imageUrl = input.required<string>();
}
```

For detailed patterns, see [references/component-patterns.md](references/component-patterns.md).
