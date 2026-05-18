# Angular Expert Skill

Expert Angular 20 reference for writing, debugging, and reviewing Angular code. Targets **Angular 20** with TypeScript 5.8+ and Node.js 20.11.1+.

## What It Does

Provides production-ready Angular 20 patterns sourced from the official docs at angular.dev. Only stable APIs are used — experimental features (Signal Forms, resource, httpResource, rxResource) are excluded. Developer Preview APIs are clearly marked where referenced.

## Coverage

| Area | Key Topics |
|---|---|
| **Components** | Standalone, `input()`/`output()`/`model()`, `@defer`, `@let`, `NgOptimizedImage`, content projection, view queries |
| **Signals & State** | `signal()`, `computed()`, `linkedSignal()`, `effect()`, `outputFromObservable()`/`outputToObservable()`, signal-based stores |
| **Routing** | Functional guards/resolvers, lazy loading, view transitions, navigation error handling, scroll management, route-level render mode |
| **Forms** | Reactive Forms with strict typing, custom/async validators, `FormArray`, zoneless compatibility |
| **HTTP** | Functional interceptors, `toSignal()` data fetching, HTTP transfer cache, error handling |
| **SSR & Hydration** | Zoneless, `provideClientHydration()`, incremental hydration, event replay, `TransferState`, platform detection |
| **Testing** | Jasmine + esbuild-based `@angular/build:unit-test`, signal inputs, zoneless testing, `TestBed.tick()`, SSR testing |
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
    ├── components.md                 # Components, directives, @let, @defer, lifecycle
    ├── signals-and-state.md          # Signals, RxJS interop, output interop, stores
    ├── routing.md                    # Routes, guards, view transitions, error handling
    ├── forms.md                      # Reactive Forms, validation, zoneless compat
    ├── http.md                       # HttpClient, interceptors, testing
    ├── ssr-and-performance.md        # SSR, hydration, zoneless, performance checklist
    ├── advanced-patterns.md          # DI, pipes, directives, architecture
    ├── testing.md                    # Jasmine, zoneless testing, SSR testing
    └── configuration.md             # Bootstrap, providers, environment, CLI
```

## Usage

Invoke for any Angular task — building components, routing, forms, HTTP, SSR, testing, architecture, performance, or debugging. The skill loads SKILL.md first and reads reference files on demand for deeper detail.
