# Angular Expert Skill

Expert Angular 20 reference for writing, debugging, and reviewing Angular code.

## What It Does

Provides production-ready Angular 20 patterns sourced from the official docs at angular.dev. Only stable APIs are used — experimental features (Signal Forms, resource, httpResource, rxResource) are excluded. Developer Preview APIs are clearly marked where referenced.

## Coverage

| Area | Key Topics |
|---|---|
| **Components** | Standalone, `input()`/`output()`/`model()`, `@defer`, `@let`, `NgOptimizedImage`, content projection, view queries, accessibility (ARIA via `host`, keyboard handling, focus management, `aria-live`, `ariaCurrentWhenActive`) |
| **Signals & State** | `signal()`, `computed()`, `linkedSignal()`, `effect()`, `afterRenderEffect()` phases (`earlyRead`, `write`, `mixedReadWrite`, `read`), `outputFromObservable()`/`outputToObservable()`, signal-based stores |
| **Routing** | Functional guards/resolvers, lazy loading, view transitions (`withViewTransitions()`, `onViewTransitionCreated`), navigation error handling, scroll management, route-level render mode |
| **Forms** | Reactive Forms with strict typing, custom/async validators, `FormArray`, custom `ControlValueAccessor` (`NG_VALUE_ACCESSOR`), zoneless compatibility |
| **HTTP** | Functional interceptors, `toSignal()` data fetching, HTTP transfer cache, error handling |
| **SSR & Hydration** | Zoneless, `provideClientHydration()`, incremental hydration with all `@defer (hydrate on ...)` triggers (`idle`, `viewport`, `interaction`, `hover`, `timer`, `immediate`, `when`, `never`), event replay, `TransferState`, platform detection |
| **Testing** | Jasmine + esbuild-based `@angular/build:unit-test`, signal inputs, zoneless testing, `TestBed.tick()`, SSR testing |
| **Animations** | `animate.enter`/`animate.leave` (CSS-driven, zoneless), View Transitions API, `provideAnimationsAsync` vs `provideAnimations`, deprecation of classic `@angular/animations` |
| **i18n** | `@angular/localize` setup, `$localize` tagged templates, `i18n` attribute, `ng extract-i18n` (xlf, xlf2, xmb, json, arb), build-time vs runtime loading, ICU formats, SSR via `withI18nSupport()` |
| **Configuration** | `bootstrapApplication()`, provider composition, `application` builder (esbuild + Vite), HMR (default on `ng serve`), Angular DevTools, environment files, `tsconfig` baseline |
| **Architecture** | Smart/dumb components, facade pattern, DI patterns, feature organization |

## Core Principles

1. **Standalone by default** — no NgModules for new code
2. **Signals over imperative state** — prefer `computed()`/`linkedSignal()` over `effect()` for derived state
3. **Zoneless change detection** — stable in v20 via `provideZonelessChangeDetection()`
4. **Functional APIs** — `inject()`, functional guards, resolvers, interceptors
5. **Native control flow** — `@if`, `@for`, `@switch`, `@let` (`*ngIf`/`*ngFor`/`*ngSwitch` are deprecated)
6. **Strong typing** — strict TypeScript, no `any`

## Structure

```
angular-expert/
├── SKILL.md                          # Main skill entry point
└── references/
    ├── components.md                 # Components, directives, @let, @defer, lifecycle, accessibility
    ├── signals-and-state.md          # Signals, RxJS interop, output interop, stores
    ├── routing.md                    # Routes, guards, view transitions, error handling
    ├── forms.md                      # Reactive Forms, validation, zoneless compat
    ├── http.md                       # HttpClient, interceptors, testing
    ├── ssr-and-performance.md        # SSR, hydration, zoneless, performance checklist
    ├── advanced-patterns.md          # DI, pipes, directives, architecture
    ├── testing.md                    # Jasmine, zoneless testing, SSR testing
    ├── configuration.md             # Bootstrap, providers, environment, CLI
    ├── animations.md                 # animate.enter/leave, View Transitions, zoneless notes
    └── i18n.md                       # @angular/localize, $localize, runtime vs build-time, SSR hydration
```

## Usage

Invoke for any Angular task — building components, routing, forms, HTTP, SSR, testing, architecture, performance, or debugging. The skill loads SKILL.md first and reads reference files on demand for deeper detail.
