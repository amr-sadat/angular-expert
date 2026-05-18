# SSR, Hydration & Performance

## Table of Contents
- [Zoneless Change Detection](#zoneless-change-detection)
- [Server-Side Rendering (SSR)](#server-side-rendering-ssr)
- [Client Hydration Setup](#client-hydration-setup)
- [HTTP Transfer Cache](#http-transfer-cache)
- [Event Replay](#event-replay)
- [Incremental Hydration](#incremental-hydration)
- [Skipping Hydration](#skipping-hydration)
- [Platform Detection & SSR-Safe Code](#platform-detection--ssr-safe-code)
- [Transfer State](#transfer-state)
- [Hydration Mismatch Debugging](#hydration-mismatch-debugging)
- [Deferred Loading Performance](#deferred-loading-performance)
- [Performance Checklist](#performance-checklist)
- [Image Optimization](#image-optimization)
- [Bundle Size Optimization](#bundle-size-optimization)

---

## Zoneless Change Detection

Zoneless is stable in Angular 20 and default in v21+. Removing ZoneJS improves performance, reduces bundle size, and simplifies debugging.

### Enabling zoneless (Angular 20)

```typescript
// app.config.ts
import { provideZonelessChangeDetection } from '@angular/core';

export const appConfig: ApplicationConfig = {
  providers: [
    provideZonelessChangeDetection(),
    // ... other providers
  ],
};
```

### Removing ZoneJS from the build

1. Remove `zone.js` from polyfills in `angular.json` (both `build` and `test` targets)
2. Remove `zone.js/testing` from test polyfills
3. Uninstall the package: `npm uninstall zone.js`

### Zoneless compatibility requirements

Angular detects state changes through these mechanisms:
- `ChangeDetectorRef.markForCheck()` (called automatically by `AsyncPipe`)
- `ComponentRef.setInput()`
- Updating a signal read in a template
- Bound host or template listener callbacks
- Attaching a view marked dirty

**All components should use `OnPush` change detection.** This is the recommended strategy and will become the default.

### Common zoneless pitfalls

| Problem | Solution |
|---|---|
| Template not updating after `setTimeout`/`setInterval` | Use signals or call `markForCheck()` |
| Reactive Forms not updating UI | Connect `valueChanges` to a signal or call `markForCheck()` |
| Third-party lib modifies state imperatively | Wrap the update in `markForCheck()` or convert to signals |
| `NgZone.onStable` / `onMicrotaskEmpty` used | Replace with `afterNextRender` / `afterEveryRender` |

### Debug mode

> ⚠️ **Developer Preview** — `provideCheckNoChangesConfig` is in developer preview since v20.0. It may change in future releases.

Detect missed change notifications during development:

```typescript
import { provideCheckNoChangesConfig } from '@angular/core';

// Only in dev mode
providers: [
  provideCheckNoChangesConfig({ exhaustive: true, interval: 500 }),
]
```

This periodically checks for bindings that updated without notification and throws `ExpressionChangedAfterItHasBeenCheckedError`.

## Server-Side Rendering (SSR)

### Setup

SSR is built into the Angular CLI. Create a new SSR project:

```bash
ng new my-app --ssr
```

Or add SSR to an existing project:

```bash
ng add @angular/ssr
```

### PendingTasks for SSR stability

In zoneless apps, Angular needs to know when async work is complete before serializing. Use `PendingTasks`:

```typescript
import { inject, PendingTasks } from '@angular/core';

@Component({ /* ... */ })
export class DataComponent {
  private pendingTasks = inject(PendingTasks);

  async loadData(): Promise<void> {
    // Tells Angular to wait for this before serializing
    this.pendingTasks.run(async () => {
      const data = await this.fetchData();
      this.dataSignal.set(data);
    });
  }
}
```

For RxJS-based work, use `pendingUntilEvent()`:

> ⚠️ **Developer Preview** — `pendingUntilEvent` is in developer preview since v20.0. It may change in future releases.

```typescript
import { pendingUntilEvent } from '@angular/core/rxjs-interop';

readonly data$ = this.http.get('/api/data').pipe(pendingUntilEvent());
```

The framework automatically handles `HttpClient` requests and `Router` navigations — you only need `PendingTasks` for custom async work.

### Route-level render mode

Configure per-route rendering strategy in `app.routes.server.ts`:

```typescript
import { RenderMode, ServerRoute } from '@angular/ssr';

export const serverRoutes: ServerRoute[] = [
  { path: '', renderMode: RenderMode.Prerender },
  { path: 'dashboard', renderMode: RenderMode.Server },
  { path: 'admin/**', renderMode: RenderMode.Client },
];
```

See `references/routing.md` for more details on `getPrerenderParams`.

## Client Hydration Setup

`provideClientHydration()` enables Angular's full-app non-destructive hydration. Without it, Angular destroys and re-renders the server-rendered DOM on the client, causing a flash of content and duplicate HTTP requests. With it, Angular reuses server-rendered HTML and attaches event listeners in place.

### Enabling hydration

```typescript
// app.config.ts
import { ApplicationConfig, provideZonelessChangeDetection } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideHttpClient, withFetch } from '@angular/common/http';
import { provideClientHydration, withEventReplay, withHttpTransferCacheOptions, withIncrementalHydration } from '@angular/platform-browser';

export const appConfig: ApplicationConfig = {
  providers: [
    provideZonelessChangeDetection(),
    provideRouter(routes),
    provideHttpClient(withFetch()),
    provideClientHydration(
      withHttpTransferCacheOptions(),  // Avoid duplicate HTTP requests on the client
      withEventReplay(),             // Replay user events captured before hydration
      withIncrementalHydration(),    // Enable @defer (hydrate on ...) triggers
    ),
  ],
};
```

### provideClientHydration feature functions

| Feature | Purpose |
|---|---|
| `withHttpTransferCacheOptions()` | Serializes HTTP responses into the server HTML; client reuses them instead of re-fetching |
| `withEventReplay()` | Captures user events (clicks, inputs) that fire before hydration and replays them after |
| `withIncrementalHydration()` | Enables hydration triggers on `@defer` blocks (`hydrate on viewport`, etc.) |
| `withI18nSupport()` | Enables hydration for apps using Angular i18n (required when using i18n) |

> `provideClientHydration()` must be present for incremental hydration (`@defer (hydrate ...)`) to function. Without it, `@defer` blocks defer loading but do not use server-rendered HTML.

## HTTP Transfer Cache

When Angular SSR renders a page, it may make `HttpClient` requests to populate component state. Without transfer cache, the client would re-fire those same requests immediately after boot — doubling your API load and causing visible flicker.

`withHttpTransferCacheOptions()` serializes all `HttpClient` GET responses into the HTML payload. On the client, Angular intercepts those same requests and returns the cached response instead of hitting the network.

```typescript
// Enabled in provideClientHydration():
provideClientHydration(withHttpTransferCacheOptions())
```

### What gets cached

- All `HttpClient` GET requests made during SSR rendering are cached by default.
- POST/PUT/DELETE requests are never cached.
- Requests made outside the Angular injection context (e.g., raw `fetch()`) are NOT cached.

### Filtering cached requests

Use the `filter` option in `withHttpTransferCacheOptions` to exclude specific requests:

```typescript
provideClientHydration(
  withHttpTransferCacheOptions({
    filter: (request) => !request.url.includes('/api/live-scores'),  // Exclude real-time data
  })
)
```

### Including request bodies in cache key

By default, only the URL is used as the cache key. To differentiate requests by headers or body:

```typescript
provideClientHydration(
  withHttpTransferCacheOptions({
    includeHeaders: ['Authorization', 'Accept-Language'],
  })
)
```

## Event Replay

Between the time the server HTML arrives in the browser and the time Angular fully hydrates the app, users may interact with the page (click buttons, type in inputs). Without event replay, those interactions are lost.

`withEventReplay()` installs a lightweight event listener that captures all user events during the pre-hydration window. Once hydration completes, Angular replays the captured events in order against the now-live component tree.

```typescript
provideClientHydration(withEventReplay())
```

This is especially important for:
- CTAs visible above the fold that users may click immediately on page load
- Navigation links
- Form inputs on fast-loading pages

## Incremental Hydration

Incremental hydration is stable in Angular 20. It renders HTML on the server but defers hydration of specific components until a trigger fires on the client.

> `withIncrementalHydration()` depends on and enables event replay automatically. If you already have `withEventReplay()` in your provider list, you can safely remove it after enabling incremental hydration.

```html
<!-- Renders on server, hydrates when scrolled into view -->
@defer (hydrate on viewport) {
  <app-product-reviews [productId]="product().id" />
} @placeholder {
  <div class="reviews-skeleton"></div>
}

<!-- Renders on server, hydrates on user interaction -->
@defer (hydrate on interaction) {
  <app-comment-form />
} @placeholder {
  <p>Click to load comment form</p>
}

<!-- Renders on server, hydrates when browser is idle -->
@defer (hydrate on idle) {
  <app-recommendations />
}

<!-- Never hydrates — static server-rendered HTML -->
@defer (hydrate never) {
  <app-static-footer />
}
```

### Combining hydrate triggers with regular triggers

Hydrate triggers work alongside regular `@defer` triggers. On initial SSR load, the hydrate trigger applies. On subsequent client-side renders, the regular trigger applies:

```html
<!-- SSR: hydrates on interaction. Client-side: loads on idle -->
@defer (on idle; hydrate on interaction) {
  <app-widget />
} @placeholder {
  <div>Widget placeholder</div>
}
```

### Available hydration triggers

| Trigger | When hydration occurs |
|---|---|
| `hydrate on viewport` | Element enters the viewport |
| `hydrate on idle` | Browser is idle |
| `hydrate on interaction` | User clicks or focuses |
| `hydrate on hover` | User hovers |
| `hydrate on timer(ms)` | After a delay |
| `hydrate on immediate` | Immediately after page load |
| `hydrate when condition` | When a boolean expression is true |
| `hydrate never` | Component stays server-rendered, never hydrates |

## Skipping Hydration

Some third-party components or legacy code produce server HTML that intentionally differs from the client output (e.g., date/time components, randomised IDs, browser-only widgets). Hydrating them causes Angular to emit hydration mismatch warnings and fall back to full re-render for that subtree.

Use `ngSkipHydration` to opt a component out of hydration entirely. Angular will destroy and re-render only that subtree on the client, while the rest of the app hydrates normally.

```html
<!-- Skip hydration for this component and all its descendants -->
<app-map-widget ngSkipHydration />

<!-- Also works on a host element -->
<div ngSkipHydration>
  <app-date-picker />
</div>
```

```typescript
// Or set it programmatically in the host metadata
@Component({
  selector: 'app-clock',
  host: { ngSkipHydration: 'true' },
  template: `{{ now | date:'HH:mm:ss' }}`,
})
export class ClockComponent { /* ... */ }
```

**Use sparingly.** `ngSkipHydration` negates the performance benefits of SSR for that subtree. Prefer fixing the underlying mismatch cause when possible.

## Platform Detection & SSR-Safe Code

Angular SSR runs component code in a Node.js environment — browser APIs like `window`, `document`, `localStorage`, and `navigator` are not available. Accessing them during SSR throws errors.

### Using PLATFORM_ID and isPlatformBrowser

```typescript
import { inject, PLATFORM_ID } from '@angular/core';
import { isPlatformBrowser, isPlatformServer } from '@angular/common';

@Injectable({ providedIn: 'root' })
export class ThemeService {
  private platformId = inject(PLATFORM_ID);

  applyTheme(theme: string): void {
    if (isPlatformBrowser(this.platformId)) {
      document.documentElement.setAttribute('data-theme', theme);
      localStorage.setItem('theme', theme);
    }
  }

  loadSavedTheme(): string | null {
    if (isPlatformBrowser(this.platformId)) {
      return localStorage.getItem('theme');
    }
    return null; // Return a safe default for SSR
  }
}
```

### afterNextRender for DOM work

`afterNextRender` is the idiomatic Angular 20 approach — it only runs in the browser, never during SSR, so you don't need a platform check:

```typescript
import { afterNextRender, ElementRef, viewChild } from '@angular/core';

@Component({
  selector: 'app-canvas-chart',
  template: `<canvas #canvas></canvas>`,
})
export class CanvasChartComponent {
  canvas = viewChild.required<ElementRef<HTMLCanvasElement>>('canvas');

  constructor() {
    // Safe: never runs during SSR
    afterNextRender(() => {
      const ctx = this.canvas().nativeElement.getContext('2d');
      this.renderChart(ctx!);
    });
  }
}
```

### Common SSR-unsafe patterns and fixes

| Unsafe | Safe alternative |
|---|---|
| `document.getElementById(...)` | `viewChild()` + `afterNextRender` |
| `window.innerWidth` in constructor | Read in `afterNextRender`, or use `ResizeObserver` via `afterNextRender` |
| `localStorage.getItem(...)` | `isPlatformBrowser` guard or a server-safe default |
| `navigator.userAgent` | Use Angular's `@angular/ssr` request context or an HTTP header |
| `setTimeout` affecting initial render | Use signals; avoid timing-dependent initial state |
| Third-party browser-only libs in constructors | Lazy-import inside `afterNextRender` |

## Transfer State

`TransferState` lets you pass arbitrary data from the server rendering context to the client, preventing duplicate computation or API calls that can't go through `HttpClient`.

```typescript
import { inject, makeStateKey, TransferState } from '@angular/core';
import { isPlatformServer } from '@angular/common';
import { PLATFORM_ID } from '@angular/core';

const CONFIG_KEY = makeStateKey<AppConfig>('appConfig');

@Injectable({ providedIn: 'root' })
export class ConfigService {
  private transferState = inject(TransferState);
  private platformId = inject(PLATFORM_ID);
  private http = inject(HttpClient);

  getConfig(): Observable<AppConfig> {
    // Client: use transferred data if available
    if (this.transferState.hasKey(CONFIG_KEY)) {
      const config = this.transferState.get(CONFIG_KEY, null)!;
      this.transferState.remove(CONFIG_KEY); // Consume — prevents stale data
      return of(config);
    }

    // Server (or client miss): fetch from API
    return this.http.get<AppConfig>('/api/config').pipe(
      tap(config => {
        if (isPlatformServer(this.platformId)) {
          this.transferState.set(CONFIG_KEY, config);
        }
      }),
    );
  }
}
```

> **Prefer `withHttpTransferCacheOptions()`** for standard `HttpClient` GET requests — it handles caching automatically. Reserve `TransferState` for non-HTTP data (e.g., database queries in a Node.js server, computed values, file reads).

## Hydration Mismatch Debugging

Angular logs hydration mismatch warnings to the browser console in development mode when the server HTML differs from what the client would render. Mismatches cause Angular to fall back to full client re-render for the affected subtree.

### Hydration warnings

Angular automatically emits `NG0500`–`NG0506` warnings to the browser console in development mode when hydration problems occur. Each code links to angular.dev/errors for details.

> **Note:** For debugging application stability issues (app not hydrating), Angular provides `provideStabilityDebugging()` (stable from v21.1). In Angular 20, use browser DevTools console — Angular logs hydration stats and mismatch warnings automatically in dev mode.

### Common causes and fixes

| Warning / Symptom | Likely Cause | Fix |
|---|---|---|
| `NG0500`: Node not found | Component renders differently on server vs client | Use `ngSkipHydration` or fix the mismatch |
| `NG0501`: Incorrect number of nodes | Conditional rendering differs (e.g., `@if` depending on `window`) | Guard browser-only state with `isPlatformBrowser` |
| `NG0502`: Nodes out of order | Async data resolves differently on server | Ensure identical initial state using resolvers or transfer state |
| `NG0503`: Unsupported projection | `<ng-content>` inside `@defer` | Move projection outside the defer block |
| Extra nodes in server HTML | `@for` track generates different keys server vs client | Use stable `track` expressions (e.g., `item.id`, not `Math.random()`) |
| Flash of unstyled content | `ngSkipHydration` causing re-render | Investigate and fix the root mismatch instead |

### Suppressing known-safe mismatches

If a component intentionally renders differently on the server (e.g., a live clock), suppress the mismatch with `ngSkipHydration` rather than disabling hydration globally:

```html
<app-live-ticker ngSkipHydration />
```

## Deferred Loading Performance

`@defer` reduces initial bundle size by code-splitting template sections:

```html
@defer (on idle) {
  <app-analytics-dashboard />
} @placeholder {
  <div class="placeholder">Dashboard will load shortly...</div>
} @loading (minimum 200ms) {
  <app-skeleton-loader />
} @error {
  <p>Failed to load dashboard</p>
}
```

### Best practices for @defer

- Use `on viewport` for below-the-fold content
- Use `on idle` for non-critical UI that should load ASAP after main content
- Use `on interaction` for content that only matters after user engagement
- Use `@placeholder` to prevent layout shift (CLS)
- Set `minimum` on `@loading` to avoid flash-of-loading-state for fast loads
- Combine with `prefetch` to start loading earlier:

```html
@defer (on viewport; prefetch on idle) {
  <app-heavy-widget />
}
```

## Performance Checklist

### Build-time

- [ ] Enable production mode (`ng build --configuration production`)
- [ ] Ensure tree-shaking removes unused code
- [ ] Use `@defer` to code-split heavy components
- [ ] Lazy-load feature routes with `loadComponent` / `loadChildren`
- [ ] Use `NgOptimizedImage` for all static images

### Runtime

- [ ] Use `OnPush` change detection on all components
- [ ] Enable zoneless (`provideZonelessChangeDetection()`)
- [ ] Use signals for all component state
- [ ] Use `computed()` for derived values (memoized automatically)
- [ ] Avoid `effect()` for setting signals — use `computed()` or `linkedSignal()`
- [ ] Use `track` with unique IDs in `@for` (not `$index` unless items never reorder)
- [ ] Keep templates simple — move complex logic to `computed()` signals
- [ ] Use `untracked()` to avoid unnecessary effect re-runs

### Network

- [ ] Use `HttpClient` with `toSignal()` for reactive data fetching
- [ ] Use functional interceptors for auth, retry, and caching
- [ ] Use `firstValueFrom()` for one-shot mutations
- [ ] Implement loading and error states for all async data

### SSR / Core Web Vitals

- [ ] Enable SSR for marketing and content pages
- [ ] Add `provideClientHydration()` to `appConfig` — required for non-destructive hydration
- [ ] Add `withHttpTransferCacheOptions()` to avoid duplicate API calls on client boot
- [ ] Add `withEventReplay()` so user interactions before hydration aren't lost
- [ ] Add `withIncrementalHydration()` when using `@defer (hydrate ...)` blocks
- [ ] Use incremental hydration (`hydrate on viewport`, `hydrate on idle`) for below-fold content
- [ ] Use `@defer (hydrate never)` for static sections
- [ ] Guard all browser-only APIs with `isPlatformBrowser()` or move into `afterNextRender`
- [ ] Use `ngSkipHydration` only for components that intentionally produce server/client mismatches
- [ ] Resolve hydration mismatch warnings (`NG0500`–`NG0506`) in dev before shipping
- [ ] Mark LCP images with `priority` on `NgOptimizedImage`
- [ ] Minimize Cumulative Layout Shift (CLS) with `@placeholder` and image dimensions

### Deprecations to watch

- `ng-reflect-*` debug attributes are deprecated — use `provideNgReflectAttributes()` if tooling still depends on them
- `@angular/platform-browser-dynamic` is deprecated — use `bootstrapApplication()` directly
- `@angular/platform-server/testing` is deprecated — prefer e2e tests for SSR verification

## Image Optimization

Always use `NgOptimizedImage`:

```typescript
import { NgOptimizedImage } from '@angular/common';

@Component({
  imports: [NgOptimizedImage],
  template: `
    <!-- LCP hero image — loads eagerly with priority -->
    <img ngSrc="/assets/hero.webp" width="1200" height="600" priority />

    <!-- Below-the-fold images — lazy loaded automatically -->
    <img ngSrc="/assets/feature.webp" width="600" height="400" />

    <!-- With a CDN image loader -->
    <img ngSrc="product-123.jpg" width="300" height="300" />
  `,
})
```

Rules:
- Always provide `width` and `height` to prevent layout shift
- Add `priority` to the Largest Contentful Paint (LCP) image
- Does NOT work with base64/data URIs
- Configure an image loader for CDN integration (Cloudinary, Imgix, etc.)

## Bundle Size Optimization

### Analyze your bundle

```bash
ng build --stats-json
npx webpack-bundle-analyzer dist/my-app/stats.json
```

### Common wins

- Remove `zone.js` (saves ~40KB gzipped)
- Lazy load all feature routes
- Use `@defer` for heavy third-party components (charts, editors, maps)
- Import only what you need from libraries (avoid barrel imports that pull in everything)
- Use the Angular CLI's built-in optimization (enabled by default in production builds)