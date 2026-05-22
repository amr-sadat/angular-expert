---
name: angular-expert
description: Expert Angular 20 reference for writing, debugging, and reviewing Angular code. Use for any Angular task — components, services, routing, forms, HTTP, signals, SSR, hydration, zoneless, testing, and architecture.
---

# Angular 20 Expert Skill

Authoritative guidance for building enterprise-grade Angular 20 applications. Every pattern here is sourced from Angular's official documentation at https://angular.dev and uses only **stable, production-ready** APIs. Experimental features (Signal Forms, resource, httpResource, rxResource) are excluded. Developer Preview APIs (`provideCheckNoChangesConfig`, `pendingUntilEvent`) are marked with warnings where used.

**Angular 20 requirements:** TypeScript 5.8+, Node.js 20.11.1+.

## Core Principles

1. **Standalone by default** — Components, directives, and pipes are standalone. Do NOT set `standalone: true` in decorators — it is the default. NgModules are legacy; avoid creating new ones.
2. **Signals over imperative state** — Use `signal()`, `computed()`, `linkedSignal()`, and `effect()` for reactivity. Prefer `computed()` or `linkedSignal()` over `effect()` for derived state — while signal writes in effects are technically allowed in v20, they create harder-to-trace data flows. Reserve RxJS for event streams and complex async orchestration.
3. **Zoneless change detection** — Angular 20 supports stable zoneless (`provideZonelessChangeDetection()`). Use `OnPush` change detection strategy on all components.
4. **Functional APIs** — Use `inject()` over constructor injection, functional guards/resolvers, and functional interceptors.
5. **Native control flow** — Use `@if`, `@for`, `@switch`, and `@let` in templates. Never use `*ngIf`, `*ngFor`, `*ngSwitch` — these are officially deprecated in Angular 20.
6. **Strong typing** — Strict TypeScript everywhere. Avoid `any`; use `unknown` when the type is uncertain.

## Architecture Overview

```
src/
├── app/
│   ├── app.ts                    # Root component
│   ├── app.routes.ts             # Top-level route config
│   ├── app.config.ts             # bootstrapApplication providers
│   ├── core/                     # Singleton services, interceptors, guards
│   │   ├── auth/
│   │   ├── http/
│   │   └── layout/
│   ├── shared/                   # Reusable components, directives, pipes
│   │   ├── components/
│   │   ├── directives/
│   │   └── pipes/
│   └── features/                 # Feature areas (lazy-loaded routes)
│       ├── dashboard/
│       │   ├── dashboard.routes.ts
│       │   ├── dashboard.component.ts
│       │   └── services/
│       └── users/
├── environments/
└── main.ts                       # bootstrapApplication entry point
```

Use **smart/dumb component separation**: smart (container) components manage state and inject services; dumb (presentational) components receive data via `input()` and emit events via `output()`.

## Decision Guide: Signals vs RxJS

| Use Case | Use |
|---|---|
| Local component state | `signal()` |
| Derived/computed values | `computed()` |
| Derived state that needs write access or prior value | `linkedSignal()` |
| Side effects reacting to signal changes | `effect()` |
| Data fetching | `HttpClient` + `toSignal()` |
| DOM event streams, debounce, merge, switchMap | RxJS |
| Converting Observable → Signal | `toSignal()` |
| Converting Signal → Observable | `toObservable()` |
| WebSocket streams | RxJS + `toSignal()` |

## Component Patterns

Use `input()` and `output()` functions instead of `@Input` / `@Output` decorators. Use `model()` for two-way binding. Set `changeDetection: ChangeDetectionStrategy.OnPush`. Prefer inline templates for small components. Use `class` bindings instead of `ngClass`, and `style` bindings instead of `ngStyle`. Put host bindings in the `host` object of `@Component`/`@Directive` — never use `@HostBinding` or `@HostListener`. For accessibility: prefer semantic HTML over ARIA, wire ARIA attributes via the `host` object (`'[attr.aria-expanded]': 'open()'`), add keyboard handlers in `host` listeners, manage focus with `inject(ElementRef)`, and use `aria-live` regions for dynamic announcements.

> Read [references/components.md](references/components.md) for component lifecycle, content projection, view queries, `@defer`, `NgOptimizedImage`, and the [Accessibility section](references/components.md#accessibility) for ARIA patterns, keyboard handling, and focus management.

## Signals & State Management

`signal()` is the foundation. `computed()` derives read-only state. `linkedSignal()` provides writable derived state with access to the previous value. `effect()` runs side effects when signals change — use sparingly and only for syncing with external systems (DOM, localStorage, analytics). While signal writes inside `effect()` are technically allowed in v20, prefer `computed()` or `linkedSignal()` for derived state — they produce clearer, more predictable data flows.

For async data, use `HttpClient` with `toSignal()` to bridge Observables into the signal world.

> Read [references/signals-and-state.md](references/signals-and-state.md) for signal patterns, store patterns, and RxJS interop.

## Routing

Define routes as `Routes` arrays. Use `loadComponent` for lazy loading. Guards and resolvers are plain functions that call `inject()`. Use `provideRouter()` with feature functions (`withComponentInputBinding()`, `withPreloading()`, `withViewTransitions()`, etc.). Use route-level render mode for SSR/SSG granularity.

> Read [references/routing.md](references/routing.md) for guards, resolvers, lazy loading, and route-level render mode.

## Animations

Use `animate.enter` and `animate.leave` for element-level entry and exit animations — these are CSS-driven, require no imports, and are fully zoneless-compatible. The classic `@angular/animations` package (`trigger`, `state`, `transition`) is deprecated as of v20.2. For route-level transitions, add `withViewTransitions()` to the router config to leverage the browser's native View Transitions API. No animation provider is needed for either approach.

> Read [references/animations.md](references/animations.md) for `animate.enter` / `animate.leave` syntax, View Transitions setup, provider selection, zoneless notes, and anti-patterns.

## Internationalization (i18n)

Add `@angular/localize` with `ng add @angular/localize`. Mark template strings with the `i18n` attribute and TypeScript strings with `$localize` tagged template literals. Extract messages with `ng extract-i18n`. Choose between build-time locale baking (`ng build --localize`, one bundle per locale, zero runtime overhead) and runtime loading (`loadTranslations()` before `bootstrapApplication()`, one bundle, network round-trip at startup). In SSR apps, always include `withI18nSupport()` in `provideClientHydration()` — omitting it causes hydration to silently skip translated components.

> Read [references/i18n.md](references/i18n.md) for `@angular/localize` setup, `$localize` tagged templates, `i18n` attribute syntax, `ng extract-i18n` output formats, runtime vs build-time tradeoffs, and SSR hydration wiring.

## Forms

Use **Reactive Forms** (`FormGroup`, `FormControl`, `FormBuilder`) — the stable, production-ready forms API. Use `fb.nonNullable.group()` for strict typing. In zoneless apps, reactive form updates (`setValue`, `patchValue`) do NOT auto-trigger change detection — connect `valueChanges` to signals via `toSignal()` or call `markForCheck()`.

> Read [references/forms.md](references/forms.md) for Reactive Forms, validation, custom validators, and zoneless compatibility.

## HTTP & Data Fetching

Use `HttpClient` for all HTTP operations. Write interceptors as functions with `withInterceptors()`. Use `toSignal()` to convert Observable responses into signals for template consumption. Use `firstValueFrom()` for one-shot mutations (POST, PUT, DELETE).

> Read [references/http.md](references/http.md) for HttpClient patterns, interceptors, error handling, and testing.

## SSR, Hydration & Performance

Enable zoneless with `provideZonelessChangeDetection()`. Add `provideClientHydration()` with `withHttpTransferCacheOptions()`, `withEventReplay()`, and `withIncrementalHydration()` for full SSR hydration. Use `@defer (hydrate on ...)` triggers for incremental hydration. Guard browser-only APIs with `isPlatformBrowser()` or `afterNextRender`. Use `ngSkipHydration` to escape hydration mismatches. Use `TransferState` to pass server-computed data to the client. Use `NgOptimizedImage` for all static images. Avoid `ExpressionChangedAfterItHasBeenCheckedError` by ensuring all state mutations go through signals or `markForCheck()`.

> Read [references/ssr-and-performance.md](references/ssr-and-performance.md) for SSR setup, client hydration configuration, HTTP transfer cache, event replay, platform detection, transfer state, hydration mismatch debugging, and performance checklist.

## Dependency Injection

Use `inject()` in field initializers or constructors. Use `providedIn: 'root'` for singleton services. Use `InjectionToken` for non-class dependencies. Use `EnvironmentInjector` for dynamic component creation.

> Read [references/advanced-patterns.md](references/advanced-patterns.md) for DI patterns, custom directives, pipes, RxJS interop, and architecture patterns.

## Application Configuration

Bootstrap with `bootstrapApplication()` in `main.ts`. Compose providers via `provideRouter()`, `provideHttpClient()`, `provideZonelessChangeDetection()`, etc. Use `app.config.ts` to centralize providers.

> Read [references/configuration.md](references/configuration.md) for the full provider setup, environment configs, and build configuration.

## Gotchas

- `standalone: true` in `@Component` / `@Directive` / `@Pipe` decorators is the default — setting it explicitly causes a build warning in Angular 20.
- `signal.mutate()` does not exist. Always use `signal.update()` or `signal.set()` with a new object/array reference.
- While signal writes inside `effect()` are allowed in v20 (`allowSignalWrites` is deprecated — writes are permitted by default), prefer `computed()` or `linkedSignal()` for derived state. Effects that write to signals create implicit data flows that are harder to trace and debug.
- Reactive Forms `setValue` / `patchValue` do NOT trigger change detection in zoneless apps. Bridge `valueChanges` to a signal via `toSignal()` or manually call `ChangeDetectorRef.markForCheck()`.
- `NgOptimizedImage` does NOT work with inline base64/data URIs. Only use it with URL-based image sources.
- `@for` requires a `track` expression. Omitting it is a compile error. Use a unique identifier (like `item.id`); avoid `$index` unless items never reorder.
- `NgZone.onStable`, `NgZone.onMicrotaskEmpty`, and `NgZone.isStable` do nothing in zoneless apps. Replace with `afterNextRender` or `afterEveryRender`.
- Functional guards/resolvers must call `inject()` synchronously at the top of the function — not inside a callback or `setTimeout`.
- Classic `@angular/animations` (`trigger`, `state`, `transition`, `animate`) is zone-dependent and deprecated since v20.2. In zoneless apps it may silently fail to advance multi-step animation sequences. Use `animate.enter` / `animate.leave` for element animations instead.
- SSR apps that use `i18n` attributes or `$localize` **must** include `withI18nSupport()` in `provideClientHydration()`. Without it, Angular silently skips hydration for every component with translated text nodes, causing a client-side re-render and a visible content flash.

## Quick Reference: What NOT to Do

| Anti-pattern | Modern Alternative |
|---|---|
| `@NgModule` for new code | Standalone components + `bootstrapApplication` |
| `*ngIf`, `*ngFor`, `*ngSwitch` (deprecated) | `@if`, `@for`, `@switch` |
| `@Input()` / `@Output()` decorators | `input()` / `output()` functions |
| `@HostBinding` / `@HostListener` | `host` object in decorator |
| `ngClass` / `ngStyle` | `[class.x]` / `[style.x]` bindings |
| Constructor injection | `inject()` function |
| `standalone: true` in decorator | Omit it — standalone is default |
| `signal.mutate()` | `signal.update()` or `signal.set()` |
| `effect()` for derived state | `computed()` or `linkedSignal()` |
| Zone.js-dependent patterns | Signals + `OnPush` + zoneless |
| Classic `trigger()` animations / `@HostBinding('@fade')` | `@animate.enter="'css-class'"` / `host: { '@animate.enter': ... }` |

## References

All reference files are in the `references/` directory. Read them when you need deeper detail on a specific topic:

- [references/components.md](references/components.md) — Components, directives, content projection, lifecycle, `@defer`
- [references/signals-and-state.md](references/signals-and-state.md) — Signals, computed, linkedSignal, effect, RxJS interop, state patterns
- [references/routing.md](references/routing.md) — Routes, lazy loading, guards, resolvers, render mode, view transitions
- [references/forms.md](references/forms.md) — Reactive Forms, validation, zoneless compatibility
- [references/http.md](references/http.md) — HttpClient, interceptors, error handling, testing
- [references/ssr-and-performance.md](references/ssr-and-performance.md) — SSR, hydration, zoneless, performance
- [references/advanced-patterns.md](references/advanced-patterns.md) — DI, directives, pipes, RxJS interop, architecture
- [references/testing.md](references/testing.md) — Jasmine best practices, component/service/pipe/directive testing, zoneless testing
- [references/configuration.md](references/configuration.md) — Bootstrap, providers, environment config
- [references/animations.md](references/animations.md) — animate.enter / animate.leave, View Transitions, provider selection, zoneless notes
- [references/i18n.md](references/i18n.md) — @angular/localize setup, $localize tagged templates, i18n attribute, message extraction, runtime vs build-time translation, SSR hydration