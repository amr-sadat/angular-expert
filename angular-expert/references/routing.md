# Routing

## Table of Contents
- [Route Configuration](#route-configuration)
- [Lazy Loading](#lazy-loading)
- [Functional Guards](#functional-guards)
- [Functional Resolvers](#functional-resolvers)
- [Route Parameters & Input Binding](#route-parameters--input-binding)
- [Nested Routes & Layouts](#nested-routes--layouts)
- [Route-Level Render Mode](#route-level-render-mode)
  - Custom HTTP headers per route
  - Custom HTTP status codes
  - Server-side redirects
- [Router Events](#router-events)
- [View Transitions](#view-transitions)
- [Navigation Error Handling](#navigation-error-handling)
- [Scroll Management](#scroll-management)
- [Preloading Strategies](#preloading-strategies)

---

## Route Configuration

Define routes in a `Routes` array. Use `provideRouter()` in the application config.

```typescript
// app.routes.ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  { path: '', redirectTo: 'dashboard', pathMatch: 'full' },
  {
    path: 'dashboard',
    loadComponent: () =>
      import('./features/dashboard/dashboard.component')
        .then(m => m.DashboardComponent),
  },
  {
    path: 'users',
    loadChildren: () =>
      import('./features/users/users.routes')
        .then(m => m.USER_ROUTES),
  },
  { path: '**', loadComponent: () => import('./not-found.component').then(m => m.NotFoundComponent) },
];
```

```typescript
// app.config.ts
import { ApplicationConfig, provideZonelessChangeDetection } from '@angular/core';
import { provideRouter, withComponentInputBinding } from '@angular/router';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideZonelessChangeDetection(),
    provideRouter(routes, withComponentInputBinding()),
  ],
};
```

## Lazy Loading

### Lazy-load a single component

```typescript
{
  path: 'settings',
  loadComponent: () =>
    import('./features/settings/settings.component')
      .then(m => m.SettingsComponent),
}
```

### Lazy-load a set of child routes

```typescript
{
  path: 'admin',
  loadChildren: () =>
    import('./features/admin/admin.routes')
      .then(m => m.ADMIN_ROUTES),
}
```

```typescript
// admin.routes.ts
import { Routes } from '@angular/router';

export const ADMIN_ROUTES: Routes = [
  {
    path: '',
    loadComponent: () =>
      import('./admin-layout.component').then(m => m.AdminLayoutComponent),
    children: [
      { path: '', redirectTo: 'overview', pathMatch: 'full' },
      {
        path: 'overview',
        loadComponent: () =>
          import('./overview/overview.component').then(m => m.OverviewComponent),
      },
      {
        path: 'users',
        loadComponent: () =>
          import('./users/admin-users.component').then(m => m.AdminUsersComponent),
      },
    ],
  },
];
```

## Functional Guards

Guards are plain functions. Use `inject()` inside them to access services.

### canActivate

```typescript
// auth.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { AuthService } from '../services/auth.service';

export const authGuard: CanActivateFn = (route, state) => {
  const auth = inject(AuthService);
  const router = inject(Router);

  if (auth.isAuthenticated()) {
    return true;
  }

  return router.createUrlTree(['/login'], {
    queryParams: { returnUrl: state.url },
  });
};
```

### canDeactivate

```typescript
import { CanDeactivateFn } from '@angular/router';

export interface HasUnsavedChanges {
  hasUnsavedChanges(): boolean;
}

export const unsavedChangesGuard: CanDeactivateFn<HasUnsavedChanges> = (component) => {
  if (component.hasUnsavedChanges()) {
    return confirm('You have unsaved changes. Leave anyway?');
  }
  return true;
};
```

### canMatch (conditionally match a route)

```typescript
import { inject } from '@angular/core';
import { CanMatchFn } from '@angular/router';
import { AuthService } from '../services/auth.service';

export const adminGuard: CanMatchFn = () => {
  return inject(AuthService).hasRole('admin');
};
```

Route usage:

```typescript
{
  path: 'dashboard',
  canActivate: [authGuard],
  canDeactivate: [unsavedChangesGuard],
  loadComponent: () => import('./dashboard.component').then(m => m.DashboardComponent),
}
```

## Functional Resolvers

Resolvers are also plain functions:

```typescript
// user.resolver.ts
import { inject } from '@angular/core';
import { ResolveFn } from '@angular/router';
import { UserService } from '../services/user.service';

export const userResolver: ResolveFn<User> = (route) => {
  const userService = inject(UserService);
  return userService.getUser(route.paramMap.get('id')!);
};
```

Route:

```typescript
{
  path: 'users/:id',
  loadComponent: () => import('./user-detail.component').then(m => m.UserDetailComponent),
  resolve: { user: userResolver },
}
```

## Route Parameters & Input Binding

With `withComponentInputBinding()` enabled, route params, query params, data, and resolve results are automatically bound to component inputs:

```typescript
@Component({
  selector: 'app-user-detail',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <h1>{{ user().name }}</h1>
  `,
})
export class UserDetailComponent {
  // Automatically bound from resolve data
  user = input.required<User>();

  // Automatically bound from route param :id
  id = input.required<string>();

  // Automatically bound from query param ?tab=
  tab = input<string>('info');
}
```

This eliminates the need to manually subscribe to `ActivatedRoute.params` or `ActivatedRoute.data` in most cases.

## Nested Routes & Layouts

Use a parent route with a `<router-outlet>` component for shared layouts:

```typescript
// Layout component
@Component({
  selector: 'app-main-layout',
  imports: [RouterOutlet, NavbarComponent, SidebarComponent],
  template: `
    <app-navbar />
    <div class="layout">
      <app-sidebar />
      <main>
        <router-outlet />
      </main>
    </div>
  `,
})
export class MainLayoutComponent {}
```

Routes:

```typescript
{
  path: '',
  component: MainLayoutComponent,
  children: [
    { path: 'dashboard', loadComponent: () => /* ... */ },
    { path: 'profile', loadComponent: () => /* ... */ },
  ],
}
```

## Route-Level Render Mode

Angular 20 supports granular rendering strategies per route. Define them in `app.routes.server.ts`:

```typescript
import { RenderMode, ServerRoute } from '@angular/ssr';

export const serverRoutes: ServerRoute[] = [
  // Pre-render at build time (SSG)
  { path: '', renderMode: RenderMode.Prerender },
  { path: 'about', renderMode: RenderMode.Prerender },

  // Server-side render on each request
  { path: 'dashboard', renderMode: RenderMode.Server },

  // Client-side render only (no SSR)
  { path: 'admin/**', renderMode: RenderMode.Client },

  // Pre-render with dynamic params
  {
    path: 'products/:id',
    renderMode: RenderMode.Prerender,
    async getPrerenderParams() {
      const ids = await fetchProductIds();
      return ids.map(id => ({ id }));
    },
  },
];
```

### RenderMode reference

| Mode | When to use |
|---|---|
| `RenderMode.Prerender` | Static pages known at build time (home, about, blog posts) |
| `RenderMode.Server` | Dynamic pages that depend on request context (dashboards, auth-gated pages) |
| `RenderMode.Client` | SPA-only sections that must not be server-rendered (admin tools, browser-only widgets) |

### Custom HTTP headers per route

> ⚠️ **Version Note** — These `ServerRoute` properties (`headers`, `status`, `redirectTo`) may only be available in Angular 21+. Verify against your Angular version before using.

Set response headers on any `ServerRoute` for cache control, security policies, or CORS:

```typescript
export const serverRoutes: ServerRoute[] = [
  {
    path: 'blog/:slug',
    renderMode: RenderMode.Server,
    headers: {
      'Cache-Control': 'public, max-age=3600, stale-while-revalidate=86400',
      'X-Content-Type-Options': 'nosniff',
    },
  },
  {
    path: 'dashboard',
    renderMode: RenderMode.Server,
    headers: { 'Cache-Control': 'private, no-store' },
  },
];
```

### Custom HTTP status codes

Return non-200 status codes for specific routes — useful for soft 404s or maintenance pages:

```typescript
export const serverRoutes: ServerRoute[] = [
  {
    path: 'not-found',
    renderMode: RenderMode.Server,
    status: 404,
    headers: { 'Cache-Control': 'no-store' },
  },
  {
    path: 'maintenance',
    renderMode: RenderMode.Server,
    status: 503,
    headers: { 'Retry-After': '3600' },
  },
];
```

### Server-side redirects

Declare HTTP-level redirects on `ServerRoute` — they resolve on the server before Angular renders, so crawlers and CDNs receive the correct status immediately:

```typescript
export const serverRoutes: ServerRoute[] = [
  {
    path: 'old-path',
    renderMode: RenderMode.Server,
    redirectTo: '/new-path',
    status: 301,  // Permanent redirect
  },
  {
    path: 'promo',
    renderMode: RenderMode.Server,
    redirectTo: '/shop/sale',
    status: 302,  // Temporary redirect
  },
];
```

> **Server redirects vs client redirects:** Use `ServerRoute.redirectTo` for HTTP-level redirects (crawlers, CDNs, initial page loads). Use `{ path: 'old', redirectTo: 'new' }` in `app.routes.ts` for client-side SPA navigation. They serve different purposes and can coexist.

For dynamic server redirects that read the incoming request (cookies, headers, query params), use the `REQUEST` token inside a resolver — see [advanced-patterns.md](advanced-patterns.md#request-and-response-tokens).

## Router Events

Subscribe to router events for analytics, loading indicators, or error handling:

```typescript
import { inject, computed } from '@angular/core';
import { Router, NavigationEnd } from '@angular/router';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { filter } from 'rxjs';

@Component({
  template: `
    @if (isNavigating()) {
      <div class="loading-bar"></div>
    }
    <router-outlet />
  `,
})
export class AppComponent {
  private router = inject(Router);
  isNavigating = computed(() => !!this.router.currentNavigation());

  constructor() {
    this.router.events.pipe(
      filter(e => e instanceof NavigationEnd),
      takeUntilDestroyed(),
    ).subscribe(event => {
      // Analytics, scroll to top, etc.
    });
  }
}
```

## View Transitions

`withViewTransitions()` enables the browser's View Transitions API for route navigations. Angular wraps route activation/deactivation inside `document.startViewTransition()`, enabling smooth animated transitions between pages. Gracefully degrades in browsers that don't support it.

```typescript
import { provideRouter, withViewTransitions } from '@angular/router';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(
      routes,
      withComponentInputBinding(),
      withViewTransitions(),
    ),
  ],
};
```

### Styling transitions with CSS

```css
/* Fade transition (default) */
::view-transition-old(root),
::view-transition-new(root) {
  animation-duration: 200ms;
}

/* Named transitions for specific elements */
::view-transition-old(hero-image),
::view-transition-new(hero-image) {
  animation-duration: 400ms;
}
```

```typescript
@Component({
  template: `<img [src]="product().image" />`,
  host: {
    '[style.view-transition-name]': '"hero-image"',
  },
})
export class ProductDetailComponent {}
```

### Conditional transitions

```typescript
withViewTransitions({
  onViewTransitionCreated: (transitionInfo) => {
    // Skip transition for back navigation
    if (transitionInfo.from?.url === transitionInfo.to.url) {
      transitionInfo.transition.skipTransition();
    }
  },
})
```

## Navigation Error Handling

`withNavigationErrorHandler()` registers a callback that runs whenever a navigation error occurs. The handler executes in an injection context, so `inject()` works.

```typescript
import { provideRouter, withNavigationErrorHandler, NavigationError, RedirectCommand } from '@angular/router';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(
      routes,
      withNavigationErrorHandler((error: NavigationError) => {
        const router = inject(Router);
        const errorTracker = inject(ErrorTrackingService);

        errorTracker.trackNavigationError(error);

        // Return a RedirectCommand to convert the error into a redirect
        return new RedirectCommand(router.parseUrl('/error'));
      }),
    ),
  ],
};
```

When the handler returns a `RedirectCommand`, Angular emits `NavigationCancel` instead of `NavigationError`, and redirects to the specified URL.

## Scroll Management

`withInMemoryScrolling()` configures scroll position management for route navigations:

```typescript
import { provideRouter, withInMemoryScrolling } from '@angular/router';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(
      routes,
      withInMemoryScrolling({
        scrollPositionRestoration: 'top',     // Scroll to top on forward navigation
        anchorScrolling: 'enabled',           // Enable #fragment scrolling
      }),
    ),
  ],
};
```

| Option | Values | Behavior |
|---|---|---|
| `scrollPositionRestoration` | `'disabled'` (default), `'top'`, `'enabled'` | `'top'` scrolls to top on every navigation; `'enabled'` restores previous position on back/forward |
| `anchorScrolling` | `'disabled'` (default), `'enabled'` | When enabled, scrolls to the element matching the URL fragment |

## Preloading Strategies

Configure how lazy-loaded routes are preloaded:

```typescript
import { provideRouter, withPreloading, PreloadAllModules } from '@angular/router';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(
      routes,
      withComponentInputBinding(),
      withPreloading(PreloadAllModules),
    ),
  ],
};
```

For selective preloading, implement a custom `PreloadingStrategy` or use route data flags:

```typescript
{ path: 'reports', loadChildren: () => ..., data: { preload: true } }
```