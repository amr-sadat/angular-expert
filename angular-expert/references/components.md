# Components, Directives & Templates

## Table of Contents
- [Component Anatomy](#component-anatomy)
- [Inputs and Outputs](#inputs-and-outputs)
- [Two-Way Binding with model()](#two-way-binding-with-model)
- [View Queries](#view-queries)
- [Lifecycle Hooks](#lifecycle-hooks)
- [Content Projection](#content-projection)
- [Host Bindings](#host-bindings)
- [Accessibility](#accessibility)
- [Deferred Loading with @defer](#deferred-loading-with-defer)
- [NgOptimizedImage](#ngoptimizedimage)
- [Template Control Flow](#template-control-flow)
- [Local Template Variables with @let](#local-template-variables-with-let)
- [Custom Directives](#custom-directives)
- [SSR-Safe Components](#ssr-safe-components)

---

## Component Anatomy

Every component is standalone by default in Angular 20. Do NOT set `standalone: true`.

```typescript
import { Component, ChangeDetectionStrategy } from '@angular/core';

@Component({
  selector: 'app-user-card',
  changeDetection: ChangeDetectionStrategy.OnPush,
  imports: [DatePipe],
  template: `
    <div class="user-card">
      <h3>{{ name() }}</h3>
      <p>Joined: {{ joinDate() | date:'mediumDate' }}</p>
    </div>
  `,
  styles: `
    .user-card {
      padding: 1rem;
      border: 1px solid #e0e0e0;
      border-radius: 8px;
    }
  `,
})
export class UserCardComponent {
  name = input.required<string>();
  joinDate = input<Date>(new Date());
}
```

Key rules:
- Always set `changeDetection: ChangeDetectionStrategy.OnPush`
- Prefer inline templates for small components (< 20 lines of HTML)
- Add only what the component needs to `imports`
- Use `styles` (inline) or `styleUrl` for styles

## Inputs and Outputs

Use the `input()` and `output()` functions — not the `@Input()` / `@Output()` decorators.

```typescript
import { Component, input, output } from '@angular/core';

@Component({
  selector: 'app-product-item',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div>
      <h4>{{ product().name }}</h4>
      <span>{{ product().price | currency }}</span>
      <button (click)="addToCart.emit(product())">Add to Cart</button>
    </div>
  `,
})
export class ProductItemComponent {
  // Required input — must be provided by parent
  product = input.required<Product>();

  // Optional input with default
  showActions = input(true);

  // Input with alias
  highlighted = input(false, { alias: 'isHighlighted' });

  // Input with transform
  label = input('', { transform: (v: string) => v.trim().toUpperCase() });

  // Output
  addToCart = output<Product>();
}
```

### Input types

| Pattern | Code |
|---|---|
| Required | `input.required<Type>()` |
| Optional with default | `input(defaultValue)` |
| Optional with explicit type | `input<Type \| undefined>(undefined)` |
| With alias | `input(default, { alias: 'publicName' })` |
| With transform | `input(default, { transform: fn })` |

## Two-Way Binding with model()

Use `model()` for inputs that the child can write back to:

```typescript
import { Component, model } from '@angular/core';

@Component({
  selector: 'app-toggle',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <button (click)="checked.set(!checked())">
      {{ checked() ? 'ON' : 'OFF' }}
    </button>
  `,
})
export class ToggleComponent {
  checked = model(false);
}
```

Parent usage:

```html
<app-toggle [(checked)]="isActive" />
```

`model()` creates both an input and an output (`checkedChange`) automatically. The parent can use banana-in-a-box syntax `[()]` for two-way binding.

## View Queries

Use `viewChild()` and `viewChildren()` functions (not `@ViewChild` / `@ViewChildren` decorators).

```typescript
import { Component, viewChild, viewChildren, ElementRef, AfterViewInit } from '@angular/core';

@Component({
  selector: 'app-gallery',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div #container>
      @for (img of images(); track img.id) {
        <img #imageEl [src]="img.url" [alt]="img.alt" />
      }
    </div>
  `,
})
export class GalleryComponent {
  images = input.required<Image[]>();

  // Single element query — returns Signal<ElementRef | undefined>
  container = viewChild<ElementRef>('container');

  // Required query — returns Signal<ElementRef>
  requiredContainer = viewChild.required<ElementRef>('container');

  // Multiple elements — returns Signal<readonly ElementRef[]>
  imageElements = viewChildren<ElementRef>('imageEl');
}
```

Content queries use `contentChild()` and `contentChildren()` with the same pattern.

## Lifecycle Hooks

Prefer `afterNextRender` and `afterEveryRender` over `ngAfterViewInit` for DOM-dependent work. These run only in the browser (not during SSR).

```typescript
import { Component, afterNextRender, afterEveryRender, ElementRef, viewChild } from '@angular/core';

@Component({
  selector: 'app-chart',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `<canvas #canvas></canvas>`,
})
export class ChartComponent {
  canvas = viewChild.required<ElementRef<HTMLCanvasElement>>('canvas');

  constructor() {
    // Runs once after the first render
    afterNextRender(() => {
      this.initChart(this.canvas().nativeElement);
    });

    // Runs after every render cycle
    afterEveryRender(() => {
      this.updateChart();
    });
  }

  private initChart(el: HTMLCanvasElement) { /* ... */ }
  private updateChart() { /* ... */ }
}
```

Standard lifecycle hooks still available: `ngOnInit`, `ngOnDestroy`, `ngOnChanges`, `ngAfterViewInit`, etc. Use them when the render-based hooks don't fit.

## Content Projection

### Single slot

```typescript
@Component({
  selector: 'app-card',
  template: `
    <div class="card">
      <ng-content />
    </div>
  `,
})
export class CardComponent {}

// Usage:
// <app-card><p>Projected content</p></app-card>
```

### Named slots

```typescript
@Component({
  selector: 'app-dialog',
  template: `
    <header><ng-content select="[dialog-title]" /></header>
    <section><ng-content /></section>
    <footer><ng-content select="[dialog-actions]" /></footer>
  `,
})
export class DialogComponent {}

// Usage:
// <app-dialog>
//   <h2 dialog-title>Confirm</h2>
//   <p>Are you sure?</p>
//   <div dialog-actions>
//     <button>Yes</button>
//     <button>No</button>
//   </div>
// </app-dialog>
```

## Host Bindings

Use the `host` object — never `@HostBinding` or `@HostListener`.

```typescript
@Component({
  selector: 'app-tooltip',
  changeDetection: ChangeDetectionStrategy.OnPush,
  host: {
    'role': 'tooltip',
    '[class.visible]': 'isVisible()',
    '[attr.aria-hidden]': '!isVisible()',
    '(mouseenter)': 'show()',
    '(mouseleave)': 'hide()',
  },
  template: `<ng-content />`,
})
export class TooltipComponent {
  isVisible = signal(false);

  show() { this.isVisible.set(true); }
  hide() { this.isVisible.set(false); }
}
```

## Accessibility

### Semantic HTML first

Prefer native HTML elements before reaching for ARIA. Native elements carry implicit roles, keyboard behaviour, and focus management at zero cost:

```html
<!-- Prefer <button> — it's focusable, activates on Enter/Space, has role="button" -->
<button (click)="submit()">Submit</button>

<!-- Prefer <nav> / <main> / <header> over generic <div> -->
<nav aria-label="Primary">...</nav>
<main>...</main>
```

Only add `role` and keyboard handlers when you must use a non-semantic element.

### ARIA attributes via the `host` object

Wire static ARIA attributes and dynamic ARIA bindings through the `host` object — never through `@HostBinding`:

```typescript
import { Component, input, signal } from '@angular/core';

@Component({
  selector: 'app-progressbar',
  changeDetection: ChangeDetectionStrategy.OnPush,
  host: {
    // Static role and range bounds
    'role': 'progressbar',
    'aria-valuemin': '0',
    'aria-valuemax': '100',
    // Dynamic binding — updates whenever value() changes
    '[attr.aria-valuenow]': 'value()',
    '[attr.aria-label]': 'label()',
  },
  template: `<div class="bar" [style.width.%]="value()"></div>`,
})
export class ProgressbarComponent {
  value = input.required<number>();
  label = input('Loading');
}
```

Key rules:
- Static ARIA: plain string values (`'role': 'progressbar'`)
- Dynamic ARIA: `[attr.aria-*]` bindings evaluated against the component class
- The `[attr.*]` form removes the attribute when the value is `null` — use this to toggle ARIA attributes conditionally

### Keyboard event handlers via the `host` listeners object

Add keyboard handlers in `host` — never with `@HostListener`:

```typescript
@Component({
  selector: 'app-disclosure',
  changeDetection: ChangeDetectionStrategy.OnPush,
  host: {
    'role': 'button',
    'tabindex': '0',
    '[attr.aria-expanded]': 'open()',
    '(click)': 'toggle()',
    '(keydown.enter)': 'toggle()',
    '(keydown.space)': '$event.preventDefault(); toggle()',
  },
  template: `
    <ng-content />
    @if (open()) {
      <div role="region"><ng-content select="[panel]" /></div>
    }
  `,
})
export class DisclosureComponent {
  open = signal(false);
  toggle() { this.open.update(v => !v); }
}
```

### Focus management with `inject(ElementRef)`

Use `inject(ElementRef)` and the native `.focus()` method for programmatic focus:

```typescript
import { Component, ElementRef, inject, signal } from '@angular/core';

@Component({
  selector: 'app-modal',
  changeDetection: ChangeDetectionStrategy.OnPush,
  host: {
    'role': 'dialog',
    'aria-modal': 'true',
    '[attr.aria-label]': 'title()',
  },
  template: `
    <button #closeBtn (click)="close()">Close</button>
    <ng-content />
  `,
})
export class ModalComponent {
  title = input.required<string>();
  private el = inject(ElementRef);

  open() {
    // Move focus into the modal so screen readers announce it
    this.el.nativeElement.focus();
  }

  close() { /* emit close event */ }
}
```

For focus trapping (keeping Tab inside a modal), use native `querySelectorAll` on `nativeElement` to collect focusable elements and handle `keydown.tab` in the `host` listener.

### `aria-live` regions for dynamic content

Announce dynamic updates to screen readers without moving focus:

```typescript
@Component({
  selector: 'app-status-announcer',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <!-- polite: waits for current speech to finish -->
    <div aria-live="polite" aria-atomic="true" class="sr-only">
      {{ message() }}
    </div>
  `,
  styles: `.sr-only { position: absolute; width: 1px; height: 1px; overflow: hidden; clip: rect(0,0,0,0); }`,
})
export class StatusAnnouncerComponent {
  message = input('');
}
```

| `aria-live` value | When to use |
|---|---|
| `polite` | Status messages, search results, form validation (waits for idle) |
| `assertive` | Time-sensitive errors that must interrupt immediately |
| `off` (default) | No announcements needed |

### Augmenting native elements

When building a custom interactive component, prefer an **attribute selector on a native element** over a custom element. This gives the component all native keyboard, focus, and ARIA semantics for free — no manual `role`, `tabindex`, or keyboard handlers needed:

```typescript
// Selector targets the native <button> — all button behaviour is inherited automatically
@Component({
  selector: 'button[app-upload]',
  changeDetection: ChangeDetectionStrategy.OnPush,
  imports: [SpinnerComponent],
  host: {
    '[attr.aria-busy]': 'uploading()',
  },
  template: `
    @if (uploading()) { <app-spinner /> } @else { <ng-content /> }
  `,
})
export class UploadButtonComponent {
  uploading = input(false);
}
```

Usage — consumers get a real `<button>`:

```html
<button app-upload [uploading]="saving()">Save</button>
```

This pattern applies most naturally to `<button>` and `<a>` but works with any element.

### Active navigation links with `ariaCurrentWhenActive`

Use `RouterLinkActive` with `ariaCurrentWhenActive` to mark the active link for screen readers. This adds an `aria-current` attribute automatically when the route matches — no manual binding needed:

```html
<nav aria-label="Primary">
  <a routerLink="/home"     routerLinkActive="active" ariaCurrentWhenActive="page">Home</a>
  <a routerLink="/about"    routerLinkActive="active" ariaCurrentWhenActive="page">About</a>
  <a routerLink="/settings" routerLinkActive="active" ariaCurrentWhenActive="page">Settings</a>
</nav>
```

`aria-current` values: `"page"` (current page), `"step"` (wizard step), `"location"`, `"date"`, `"time"`.

### Anti-patterns

| Anti-pattern | Why it fails | Fix |
|---|---|---|
| `<div (click)="action()">` with no `role` | Not keyboard-accessible; invisible to screen readers | Use `<button>` or add `role="button"` + `tabindex="0"` + keyboard handler |
| `<button><mat-icon>close</mat-icon></button>` with no label | Screen reader reads "close" only if the icon has text content | Add `aria-label="Close dialog"` to the `<button>` |
| Custom button element (`<app-btn>`) | Loses native focus, keyboard, and role semantics | Use attribute selector `button[app-btn]` to wrap native `<button>` |
| `@HostBinding('attr.aria-expanded')` | Banned — use `host: { '[attr.aria-expanded]': '...' }` | Move to the `host` object |
| `aria-hidden="false"` on visible content | Redundant and sometimes confusing; omit it | Remove the attribute entirely |
| `routerLinkActive` without `ariaCurrentWhenActive` | Active link not announced to screen readers | Add `ariaCurrentWhenActive="page"` alongside `routerLinkActive` |

---

## Deferred Loading with @defer

`@defer` lazily loads parts of a template. It reduces initial bundle size and improves LCP.

```html
<!-- Load when the element enters the viewport -->
@defer (on viewport) {
  <app-comments [postId]="post().id" />
} @placeholder {
  <div class="skeleton">Loading comments...</div>
} @loading (minimum 300ms) {
  <app-spinner />
} @error {
  <p>Failed to load comments.</p>
}
```

### Common triggers

| Trigger | When it loads |
|---|---|
| `on viewport` | Element enters viewport |
| `on idle` | Browser is idle |
| `on interaction` | User clicks/focuses the placeholder |
| `on hover` | User hovers the placeholder |
| `on timer(2s)` | After a delay |
| `on immediate` | Immediately after non-deferred content renders |
| `when condition` | When a boolean expression becomes true |

### Incremental hydration (SSR)

`@defer` blocks also support hydration triggers for SSR:

```html
@defer (hydrate on viewport) {
  <app-heavy-widget />
}
```

This renders the HTML on the server but delays hydration until the trigger fires on the client.

## NgOptimizedImage

Use `NgOptimizedImage` for all static images. It provides automatic lazy loading, priority hints, and srcset generation.

```typescript
import { NgOptimizedImage } from '@angular/common';

@Component({
  imports: [NgOptimizedImage],
  template: `
    <!-- LCP image — mark as priority -->
    <img ngSrc="/hero.jpg" width="1200" height="600" priority />

    <!-- Regular image — lazy loaded by default -->
    <img ngSrc="/thumbnail.jpg" width="300" height="200" />
  `,
})
export class HeroComponent {}
```

`NgOptimizedImage` does NOT work for inline base64 images. Always provide `width` and `height` attributes.

## Template Control Flow

Always use the native control flow syntax:

```html
<!-- Conditional -->
@if (user(); as user) {
  <h1>Welcome, {{ user.name }}</h1>
} @else if (isLoading()) {
  <app-spinner />
} @else {
  <a routerLink="/login">Sign in</a>
}

<!-- Iteration — track is REQUIRED -->
@for (item of items(); track item.id) {
  <app-item [data]="item" />
} @empty {
  <p>No items found.</p>
}

<!-- Switch -->
@switch (status()) {
  @case ('active') { <span class="badge-active">Active</span> }
  @case ('inactive') { <span class="badge-inactive">Inactive</span> }
  @default { <span>Unknown</span> }
}
```

`track` in `@for` is mandatory. Use a unique identifier (like `item.id`) for optimal DOM reconciliation. `$index` is available as a fallback but is less efficient for reordering.

## Local Template Variables with @let

`@let` declares read-only local variables in templates. They are reactive — when any signal or expression they reference changes, the variable updates.

```html
<!-- Simple computed value -->
@let fullName = user().firstName + ' ' + user().lastName;
<h1>{{ fullName }}</h1>

<!-- Unwrap async data -->
@let data = data$ | async;
@if (data) {
  <app-table [rows]="data" />
}

<!-- Complex expressions -->
@let total = cart().items.length;
@let hasDiscount = total > 5;
<p>{{ total }} items @if (hasDiscount) { (discount applied!) }</p>

<!-- Destructure-like patterns -->
@let coordinates = { x: position().x * scale(), y: position().y * scale() };
<app-marker [x]="coordinates.x" [y]="coordinates.y" />
```

Key rules:
- `@let` variables are **read-only** — you cannot reassign them
- Scoped to the current template view and its descendants
- Support pipes: `@let formatted = rawDate | date:'mediumDate';`
- Support the template expression syntax including `**` (exponentiation)
- Cannot be referenced before declaration (no hoisting)

### When to use @let vs computed()

| Scenario | Use |
|---|---|
| Value needed only in the template | `@let` |
| Value needed in both template and TypeScript | `computed()` |
| Expensive derivation that should be memoized | `computed()` |
| Unwrapping an async pipe result | `@let` |
| Simplifying a repeated template expression | `@let` |

## Custom Directives

Directives are also standalone by default.

```typescript
import { Directive, input, effect, ElementRef, inject } from '@angular/core';

@Directive({
  selector: '[appHighlight]',
  host: {
    '[style.backgroundColor]': 'color()',
  },
})
export class HighlightDirective {
  color = input('yellow', { alias: 'appHighlight' });
}
```

Usage:

```html
<p appHighlight="lightblue">Highlighted text</p>
```

For structural behavior, prefer `@if` / `@for` / `@defer` over custom structural directives in most cases.

## SSR-Safe Components

Components render in both a Node.js server environment (SSR) and the browser. Browser-only APIs (`window`, `document`, `localStorage`, `navigator`) are unavailable during SSR and will throw if accessed outside a browser guard.

### afterNextRender — idiomatic browser-only hook

`afterNextRender` is the preferred way to run browser-only setup. It never executes during SSR — no guard needed:

```typescript
import { Component, afterNextRender, viewChild, ElementRef } from '@angular/core';

@Component({
  selector: 'app-map',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `<div #mapContainer style="height:400px"></div>`,
})
export class MapComponent {
  container = viewChild.required<ElementRef<HTMLDivElement>>('mapContainer');

  constructor() {
    afterNextRender(() => {
      // Safe: only runs in the browser, never during SSR
      initThirdPartyMap(this.container().nativeElement);
    });
  }
}
```

### isPlatformBrowser — explicit platform guard

Use when the check must happen outside a render hook (e.g. in a service method, a signal initialiser, or a conditional block):

```typescript
import { Component, inject, PLATFORM_ID, signal } from '@angular/core';
import { isPlatformBrowser } from '@angular/common';

@Component({
  selector: 'app-theme-toggle',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `<button (click)="toggle()">Toggle theme</button>`,
})
export class ThemeToggleComponent {
  private platformId = inject(PLATFORM_ID);

  // Read localStorage only in the browser
  theme = signal(
    isPlatformBrowser(this.platformId)
      ? (localStorage.getItem('theme') ?? 'light')
      : 'light',
  );

  toggle(): void {
    const next = this.theme() === 'light' ? 'dark' : 'light';
    this.theme.set(next);
    if (isPlatformBrowser(this.platformId)) {
      localStorage.setItem('theme', next);
    }
  }
}
```

### ngSkipHydration — escaping hydration mismatches

When a component intentionally renders different HTML on the server vs the client (e.g. a live clock, a randomised component, or a third-party widget that modifies the DOM directly), Angular will emit hydration mismatch warnings (`NG0500`–`NG0506`).

Apply `ngSkipHydration` to tell Angular to skip hydration for that component and its subtree, falling back to full client re-render for that section only:

```html
<!-- Applied as an attribute on the host element -->
<app-live-clock ngSkipHydration />

<!-- Applied on a wrapper when you don't control the component -->
<div ngSkipHydration>
  <third-party-widget />
</div>
```

```typescript
// Applied via host metadata when you do own the component
@Component({
  selector: 'app-random-hero',
  host: { ngSkipHydration: 'true' },
  template: `<img [src]="randomHeroImage()" />`,
})
export class RandomHeroComponent {
  randomHeroImage = signal(pickRandom(HERO_IMAGES));
}
```

**Use sparingly.** `ngSkipHydration` trades the SSR performance benefit (reusing server HTML) for a client re-render of that subtree. Always prefer fixing the root mismatch cause first.

### Common SSR-unsafe patterns

| Unsafe pattern | SSR-safe alternative |
|---|---|
| `document.getElementById(...)` in constructor | `viewChild()` + `afterNextRender` |
| `window.innerWidth` in a signal initialiser | Read in `afterNextRender` or use a `ResizeObserver` |
| `localStorage.getItem(...)` unconditionally | Wrap in `isPlatformBrowser()` guard |
| `navigator.language` unconditionally | `isPlatformBrowser()` guard with server fallback |
| Importing a browser-only lib at module level | Dynamic `import()` inside `afterNextRender` |
| Timers (`setTimeout`) that affect initial render | Use signals; derive initial state without timing |