# Animations

## Table of Contents
- [Overview](#overview)
- [animate.enter — Entry Animations](#animateenter--entry-animations)
- [animate.leave — Exit Animations](#animateleave--exit-animations)
- [Enter and Leave Together](#enter-and-leave-together)
- [Dynamic and Conditional Animations](#dynamic-and-conditional-animations)
- [Multiple Classes](#multiple-classes)
- [Host Bindings for Animations](#host-bindings-for-animations)
- [View Transitions API](#view-transitions-api)
- [provideAnimationsAsync vs provideAnimations](#provideanimationsasync-vs-provideanimations)
- [Zoneless Compatibility](#zoneless-compatibility)
- [Accessibility and Reduced Motion](#accessibility-and-reduced-motion)
- [Anti-patterns](#anti-patterns)

---

## Overview

Angular 20 provides two complementary animation systems:

1. **`animate.enter` / `animate.leave`** — element-level CSS-driven animations. These are compiler-supported bindings (not directives), require no imports, and are fully zoneless-compatible.
2. **View Transitions API** via `withViewTransitions()` — page-level animated route transitions using the browser's native `document.startViewTransition()`.

The classic `@angular/animations` package (`trigger`, `state`, `transition`, `animate`, `keyframes`, `query`, `stagger`) is **deprecated** in v20 and slated for future removal. New code should never introduce it.

| Goal | API |
|---|---|
| Element enters the DOM | `@animate.enter="'css-class'"` |
| Element leaves the DOM | `@animate.leave="'css-class'"` |
| Route-to-route page transition | `withViewTransitions()` + `::view-transition-*` CSS |
| Named element persistence across routes | `view-transition-name` + `::view-transition-*` CSS |
| Disable animations in tests | `provideAnimationsAsync('noop')` or omit providers |
| Legacy code still on classic API | `provideAnimations()` or `provideAnimationsAsync()` (deprecated — migrate away) |

---

## animate.enter — Entry Animations

`@animate.enter` applies one or more CSS classes to an element the moment it is inserted into the DOM. Angular removes those classes once the longest CSS animation or transition on the element completes.

```typescript
import { Component, ChangeDetectionStrategy, input, computed } from '@angular/core';

@Component({
  selector: 'app-notification',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    @if (visible()) {
      <div @animate.enter="'fade-slide-in'" class="notification">
        {{ message() }}
      </div>
    }
  `,
  styles: `
    .notification {
      padding: 1rem;
      background: #4caf50;
      color: white;
      border-radius: 4px;
    }

    .fade-slide-in {
      animation: fadeSlideIn 300ms ease-out;
    }

    @keyframes fadeSlideIn {
      from {
        opacity: 0;
        transform: translateY(-8px);
      }
      to {
        opacity: 1;
        transform: translateY(0);
      }
    }
  `,
})
export class NotificationComponent {
  visible = input(false);
  message = input('');
}
```

Key rules:
- The value is a CSS class name string (or an array — see [Multiple Classes](#multiple-classes)).
- Angular applies the class immediately on DOM insertion, then removes it after the animation finishes.
- The element's resting styles must be defined on a persistent class, **not** on the animation class, because the animation class is removed when done.
- No import is needed — `@animate.enter` is understood by the Angular template compiler.

CSS `transition` properties work too — not just `@keyframes`. Define the start state in the animation class and the end state in the element's base class; Angular's removal of the animation class triggers the transition.

---

## animate.leave — Exit Animations

`@animate.leave` applies CSS classes when an element is about to be removed from the DOM. Angular holds the element in the DOM until all animations complete, then removes it.

```html
@if (visible()) {
  <div @animate.leave="'fade-slide-out'" class="toast">
    {{ message() }}
  </div>
}
```

```css
.toast { padding: 0.75rem 1.25rem; background: #1976d2; color: white; border-radius: 4px; }

.fade-slide-out {
  animation: fadeSlideOut 200ms ease-in forwards;
}

@keyframes fadeSlideOut {
  from { opacity: 1; transform: translateY(0); }
  to   { opacity: 0; transform: translateY(8px); }
}
```

Always use `forwards` fill mode on leave keyframe animations. Without it, the element flashes back to its original style in the brief gap between `animationend` firing and Angular's DOM removal.

---

## Enter and Leave Together

Apply both bindings on the same element:

```html
<div @animate.enter="'slide-in'" @animate.leave="'slide-out'" class="panel">
  {{ content() }}
</div>
```

```css
.panel { /* resting styles */ }

.slide-in {
  animation: slideIn 250ms ease-out;
}

.slide-out {
  animation: slideOut 200ms ease-in forwards;
}

@keyframes slideIn {
  from { transform: translateX(-100%); opacity: 0; }
  to   { transform: translateX(0);    opacity: 1; }
}

@keyframes slideOut {
  from { transform: translateX(0);    opacity: 1; }
  to   { transform: translateX(100%); opacity: 0; }
}
```

For stagger effects in a list, use an inline `[style.animation-delay]` binding — do not reach for the classic `stagger()` API:

```html
@for (item of items(); track item.id; let i = $index) {
  <li
    @animate.enter="'list-item-enter'"
    @animate.leave="'list-item-leave'"
    class="list-item"
    [style.animation-delay]="i * 40 + 'ms'"
  >
    {{ item.label }}
  </li>
}
```

---

## Dynamic and Conditional Animations

The binding value is any template expression — a signal, a `computed()`, or a ternary:

```typescript
@Component({
  selector: 'app-alert',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    @if (visible()) {
      <div
        @animate.enter="enterClass()"
        @animate.leave="'fade-out'"
        class="alert"
        [class]="severity()"
      >
        {{ message() }}
      </div>
    }
  `,
})
export class AlertComponent {
  visible  = input(false);
  message  = input('');
  severity = input<'info' | 'warning' | 'error'>('info');

  enterClass = computed(() =>
    this.severity() === 'error' ? 'shake-in' : 'fade-in'
  );
}
```

```css
.fade-in  { animation: fadeIn  300ms ease-out; }
.fade-out { animation: fadeOut 200ms ease-in forwards; }
.shake-in { animation: shakeIn 400ms ease-out; }

@keyframes fadeIn  { from { opacity: 0; } to { opacity: 1; } }
@keyframes fadeOut { from { opacity: 1; } to { opacity: 0; } }
@keyframes shakeIn {
  0%   { transform: translateX(-12px); opacity: 0; }
  30%  { transform: translateX(6px);  opacity: 1; }
  60%  { transform: translateX(-4px); }
  80%  { transform: translateX(2px);  }
  100% { transform: translateX(0);    }
}
```

---

## Multiple Classes

Pass an array to apply several classes at once:

```html
<!-- Applies 'enter-base' and 'enter-primary' simultaneously -->
<div @animate.enter="['enter-base', 'enter-primary']" class="card">
  ...
</div>
```

This is useful for separating the animation definition from theme or duration overrides:

```css
/* Shared animation definition */
.enter-base {
  animation-name: fadeUp;
  animation-timing-function: ease-out;
  animation-duration: 300ms;
}

/* Theme-specific variant — overrides duration only */
.enter-primary {
  animation-duration: 200ms;
}

@keyframes fadeUp {
  from { opacity: 0; transform: translateY(8px); }
  to   { opacity: 1; transform: translateY(0); }
}
```

---

## Host Bindings for Animations

Animate a component's own host element using the `host` object in the decorator. Do **not** use `@HostBinding('@fade')`.

```typescript
@Component({
  selector: 'app-modal',
  changeDetection: ChangeDetectionStrategy.OnPush,
  host: {
    '@animate.enter': "'modal-enter'",
    '@animate.leave': "'modal-leave'",
  },
  template: `
    <div class="modal-content">
      <ng-content />
    </div>
  `,
  styles: `
    :host {
      display: block;
      position: fixed;
      inset: 0;
      background: rgba(0, 0, 0, 0.5);
      display: flex;
      align-items: center;
      justify-content: center;
    }

    .modal-enter {
      animation: modalIn 250ms cubic-bezier(0.34, 1.56, 0.64, 1);
    }

    .modal-leave {
      animation: modalOut 200ms ease-in forwards;
    }

    @keyframes modalIn {
      from { opacity: 0; transform: scale(0.92); }
      to   { opacity: 1; transform: scale(1); }
    }

    @keyframes modalOut {
      from { opacity: 1; transform: scale(1); }
      to   { opacity: 0; transform: scale(0.92); }
    }
  `,
})
export class ModalComponent {}
```

---

## View Transitions API

The browser's View Transitions API animates the transition between two rendered states. Angular's router integrates with it via `withViewTransitions()`, wrapping each navigation in `document.startViewTransition()`. The API degrades gracefully in unsupported browsers.

> For the router configuration details, see [routing.md](routing.md#view-transitions).

### Setup

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideRouter, withViewTransitions } from '@angular/router';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(
      routes,
      withViewTransitions(),
    ),
  ],
};
```

`provideAnimations()` or `provideAnimationsAsync()` are **not** required for View Transitions — this feature uses the browser's native API, not `@angular/animations`.

### Default cross-fade

Without any CSS, the browser cross-fades old and new content. Customise the duration:

```css
/* styles.css */
::view-transition-old(root),
::view-transition-new(root) {
  animation-duration: 200ms;
  animation-timing-function: ease-out;
}
```

### Named element transitions (hero animations)

Assign a unique `view-transition-name` to an element so the browser treats it as the "same" element on both sides of the navigation and morphs it independently.

```typescript
// product-card.component.ts
@Component({
  selector: 'app-product-card',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <a [routerLink]="['/products', product().id]">
      <img
        [src]="product().imageUrl"
        [alt]="product().name"
        [style.view-transition-name]="'product-image-' + product().id"
      />
      <h3>{{ product().name }}</h3>
    </a>
  `,
})
export class ProductCardComponent {
  product = input.required<Product>();
}
```

```typescript
// product-detail.component.ts
@Component({
  selector: 'app-product-detail',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <img
      [src]="product().imageUrl"
      [alt]="product().name"
      [style.view-transition-name]="'product-image-' + product().id"
    />
    <h1>{{ product().name }}</h1>
  `,
})
export class ProductDetailComponent {
  product = input.required<Product>();
}
```

The `view-transition-name` value must be unique across the entire page at any given moment. Including the entity ID prevents collisions when a list displays multiple cards.

```css
/* Customise the hero element's transition */
::view-transition-old(product-image-),
::view-transition-new(product-image-) {
  animation-duration: 400ms;
  animation-timing-function: ease-in-out;
}
```

### Skip transition on specific navigations

```typescript
withViewTransitions({
  onViewTransitionCreated: ({ transition, from }) => {
    // Skip transition for navigations that are not meaningful to animate
    if (!from) {
      transition.skipTransition(); // initial load
    }
  },
})
```

---

## provideAnimationsAsync vs provideAnimations

> **Deprecation notice**: Both providers are deprecated in v20 and slated for future removal. They power the classic `@angular/animations` trigger-based engine. **For new code, use `animate.enter` / `animate.leave` — no animation provider is needed.** Use these providers only when maintaining code that depends on the classic API.

### `provideAnimations()` — eager

Synchronously loads the `BrowserAnimationsModule` renderer. The classic animations engine is included in the initial bundle.

```typescript
// app.config.ts
import { provideAnimations } from '@angular/platform-browser/animations';

export const appConfig: ApplicationConfig = {
  providers: [
    provideAnimations(),
  ],
};
```

### `provideAnimationsAsync()` — lazy

Loads the animations renderer as a separate async chunk, reducing the initial bundle size. Animations are unavailable until the renderer chunk downloads.

```typescript
// app.config.ts
import { provideAnimationsAsync } from '@angular/platform-browser/animations/async';

export const appConfig: ApplicationConfig = {
  providers: [
    provideAnimationsAsync(),          // 'animations' (default) — lazy enable
    // provideAnimationsAsync('noop'), // disables animations — useful in tests
  ],
};
```

### Decision guide

| Scenario | Choice |
|---|---|
| New code, zoneless app | No provider — use `animate.enter` / `animate.leave` |
| Large app, classic API in legacy code | `provideAnimationsAsync()` — smaller initial bundle |
| Small app, classic API, simplicity preferred | `provideAnimations()` |
| Disabling animations in tests | `provideAnimationsAsync('noop')` |
| No `@angular/animations` usage at all | Omit both providers entirely |

Never import `BrowserAnimationsModule` or `NoopAnimationsModule` in standalone apps — these are the NgModule equivalents and belong to legacy codebases.

---

## Zoneless Compatibility

### Why classic `@angular/animations` is zone-dependent

The trigger-based API relies on Zone.js in two ways:

1. **Event patching** — Zone.js patches `animationend` and `transitionend` DOM events. When these fire inside a zone-patched context, Angular's change detection is scheduled automatically to update animation state.
2. **State tracking** — animation state transitions are driven by template bindings that Zone.js marks dirty, triggering a CD cycle.

In a zoneless app (`provideZonelessChangeDetection()`), Zone.js is absent. The animation engine's internal scheduler may fail to advance multi-step sequences (`query`, `stagger`, `group`) because the CD cycles it depended on are never triggered.

### Why `animate.enter` / `animate.leave` are zoneless-safe

These APIs delegate animation execution to the browser's CSS engine:

- Angular applies the CSS class on DOM insertion.
- The browser owns all animation timing — no JS scheduler is involved.
- Angular listens for native `animationend` / `transitionend` events to know when to remove the class or finalize DOM removal — no trigger-based state machine is involved.

The result: `animate.enter` / `animate.leave` behave identically in zoneless and zone-based apps.

### View Transitions and zoneless

`withViewTransitions()` calls `document.startViewTransition()` inside the Angular router's navigation pipeline. The transition's `ready` and `finished` promises are awaited using native `async/await`, not Zone.js-patched Promise chains. The router schedules its own CD via signals; no Zone.js dependency is introduced by enabling view transitions.

---

## Accessibility and Reduced Motion

The `prefers-reduced-motion: reduce` media query must be respected. Users with vestibular disorders can be harmed by large motion effects.

Add a global reduced-motion reset in `styles.css`:

```css
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 1ms !important;
    transition-duration: 1ms !important;
    scroll-behavior: auto !important;
  }

  /* View Transitions */
  ::view-transition-old(root),
  ::view-transition-new(root) {
    animation-duration: 1ms;
  }
}
```

Use `1ms` rather than `0ms` — setting `animation-duration: 0ms` causes some browsers to skip `animationend` entirely, leaving Angular unable to remove the animation class.

---

## Anti-patterns

### Classic trigger-based API (deprecated)

```typescript
// WRONG — deprecated since v20.2, zone-dependent, adds ~50 KB gzipped
import { trigger, state, style, animate, transition } from '@angular/animations';

@Component({
  animations: [
    trigger('fade', [
      state('in',  style({ opacity: 1 })),
      state('out', style({ opacity: 0 })),
      transition('in <=> out', animate('200ms ease-in-out')),
    ]),
  ],
  template: `<div [@fade]="faded() ? 'out' : 'in'">...</div>`,
})
export class FadeComponent {}
```

Replace with:

```html
<!-- Use @if for enter/leave, or animate.enter/leave on a persistent element -->
@if (!faded()) {
  <div @animate.enter="'fade-in'" @animate.leave="'fade-out'" class="content">
    ...
  </div>
}
```

### `@HostBinding('@fade')` (legacy decorator)

```typescript
// WRONG — @HostBinding is banned; also references the legacy trigger API
@HostBinding('@fade') animationState = 'in';
```

Use `host: { '@animate.enter': "'class-name'" }` in the decorator instead.

### `BrowserAnimationsModule` in standalone apps

```typescript
// WRONG — NgModule import
@NgModule({ imports: [BrowserAnimationsModule] })
export class AppModule {}
```

Use `provideAnimationsAsync()` for legacy code, or omit any provider when using `animate.enter` / `animate.leave`.

### Missing `forwards` fill mode on leave animations

```css
/* WRONG — element flashes back before removal */
.slide-out { animation: slideOut 200ms ease-in; }

/* CORRECT */
.slide-out { animation: slideOut 200ms ease-in forwards; }
```

### Duplicate `view-transition-name` values

```html
<!-- WRONG — all list items share the same name; causes undefined browser behavior -->
@for (item of items(); track item.id) {
  <li [style.view-transition-name]="'list-item'">{{ item.label }}</li>
}

<!-- CORRECT — include the entity ID -->
@for (item of items(); track item.id) {
  <li [style.view-transition-name]="'list-item-' + item.id">{{ item.label }}</li>
}
```

### Omitting `prefers-reduced-motion`

Any animation that moves content more than a few pixels must have a reduced-motion override. Omitting it is an accessibility failure.
