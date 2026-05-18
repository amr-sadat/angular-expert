# Advanced Patterns

## Table of Contents
- [Dependency Injection Patterns](#dependency-injection-patterns)
- [Custom Pipes](#custom-pipes)
- [Custom Directives](#custom-directives)
- [RxJS Interop Patterns](#rxjs-interop-patterns)
- [Error Handling Patterns](#error-handling-patterns)
- [Testing Overview](#testing-overview)
- [Architecture Patterns](#architecture-patterns)

---

## Dependency Injection Patterns

### inject() function

Always prefer `inject()` over constructor injection:

```typescript
import { inject, Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';

@Injectable({ providedIn: 'root' })
export class ApiService {
  private http = inject(HttpClient);
  private config = inject(APP_CONFIG);
}
```

### InjectionToken for non-class values

```typescript
import { InjectionToken } from '@angular/core';

export interface AppConfig {
  apiUrl: string;
  production: boolean;
}

export const APP_CONFIG = new InjectionToken<AppConfig>('app.config');

// Provide in app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [
    {
      provide: APP_CONFIG,
      useValue: {
        apiUrl: 'https://api.example.com',
        production: true,
      },
    },
  ],
};

// Inject
const config = inject(APP_CONFIG);
```

### Factory providers

```typescript
export const LOGGER = new InjectionToken<Logger>('logger', {
  providedIn: 'root',
  factory: () => {
    const config = inject(APP_CONFIG);
    return config.production
      ? new ProductionLogger()
      : new ConsoleLogger();
  },
});
```

### Component-scoped providers

Provide a service at the component level for per-instance state:

```typescript
@Component({
  selector: 'app-editor',
  providers: [EditorStateService],
  template: `...`,
})
export class EditorComponent {
  private state = inject(EditorStateService);
}
```

Each `<app-editor>` instance gets its own `EditorStateService`.

### Optional injection

```typescript
const analytics = inject(AnalyticsService, { optional: true });
analytics?.track('page_view');
```

### EnvironmentInjector for dynamic components

```typescript
import { EnvironmentInjector, createComponent, ViewContainerRef, inject } from '@angular/core';

@Component({ /* ... */ })
export class DynamicHost {
  private vcr = inject(ViewContainerRef);
  private injector = inject(EnvironmentInjector);

  loadComponent(component: Type<unknown>): void {
    this.vcr.clear();
    this.vcr.createComponent(component, {
      environmentInjector: this.injector,
    });
  }
}
```

## Custom Pipes

Pipes are standalone by default. Use them for pure data transformations in templates.

### Simple pipe

```typescript
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({ name: 'truncate' })
export class TruncatePipe implements PipeTransform {
  transform(value: string, maxLength = 50, suffix = '...'): string {
    if (!value || value.length <= maxLength) return value;
    return value.substring(0, maxLength).trimEnd() + suffix;
  }
}
```

Usage:

```html
<p>{{ description() | truncate:100:'…' }}</p>
```

### Pipe with inject()

```typescript
import { Pipe, PipeTransform, inject, LOCALE_ID } from '@angular/core';

@Pipe({ name: 'relativeTime' })
export class RelativeTimePipe implements PipeTransform {
  private locale = inject(LOCALE_ID);

  transform(value: Date | string): string {
    const date = new Date(value);
    const formatter = new Intl.RelativeTimeFormat(this.locale, { numeric: 'auto' });
    const diff = date.getTime() - Date.now();
    const days = Math.round(diff / (1000 * 60 * 60 * 24));

    if (Math.abs(days) < 1) {
      const hours = Math.round(diff / (1000 * 60 * 60));
      return formatter.format(hours, 'hour');
    }
    return formatter.format(days, 'day');
  }
}
```

### When to use pipes vs computed

| Scenario | Use |
|---|---|
| Formatting for display (date, currency, truncation) | Pipe |
| Derived business logic | `computed()` signal |
| Transformation shared across many templates | Pipe |
| Transformation specific to one component | `computed()` |

## Custom Directives

### Attribute directive with signals

```typescript
import { Directive, input, effect, inject, ElementRef } from '@angular/core';

@Directive({
  selector: '[appAutofocus]',
})
export class AutofocusDirective {
  private el = inject(ElementRef);
  enabled = input(true, { alias: 'appAutofocus' });

  constructor() {
    effect(() => {
      if (this.enabled()) {
        this.el.nativeElement.focus();
      }
    });
  }
}
```

### Directive with host bindings

```typescript
@Directive({
  selector: '[appClickOutside]',
  host: {
    '(document:click)': 'onDocumentClick($event)',
  },
})
export class ClickOutsideDirective {
  private el = inject(ElementRef);
  clickOutside = output<void>();

  onDocumentClick(event: MouseEvent): void {
    if (!this.el.nativeElement.contains(event.target)) {
      this.clickOutside.emit();
    }
  }
}
```

Usage:

```html
<div appClickOutside (clickOutside)="closeDropdown()">
  <!-- dropdown content -->
</div>
```

### Directive composition

Apply multiple directives to a host component:

```typescript
@Component({
  selector: 'app-button',
  hostDirectives: [
    { directive: TooltipDirective, inputs: ['tooltip'] },
    { directive: RippleDirective },
  ],
  template: `<ng-content />`,
})
export class ButtonComponent {}
```

## RxJS Interop Patterns

### toSignal with initial value

```typescript
import { toSignal } from '@angular/core/rxjs-interop';

// From an Observable — requires initialValue since observables are async
const data = toSignal(this.dataService.getData(), { initialValue: [] });

// From a BehaviorSubject — can use requireSync
const user = toSignal(this.authService.currentUser$, { requireSync: true });
```

### toObservable for RxJS operators

```typescript
import { toObservable } from '@angular/core/rxjs-interop';
import { debounceTime, distinctUntilChanged, switchMap } from 'rxjs';

searchQuery = signal('');

searchResults = toSignal(
  toObservable(this.searchQuery).pipe(
    debounceTime(300),
    distinctUntilChanged(),
    switchMap(query => this.searchService.search(query)),
  ),
  { initialValue: [] },
);
```

### Combining signals and observables

```typescript
// Signal-first: derive from signals, use computed
const fullName = computed(() => `${firstName()} ${lastName()}`);

// Observable-first: convert with toSignal
const route = toSignal(this.route.params);

// Bridge: signal → observable → RxJS operators → signal
const debouncedSearch = toSignal(
  toObservable(this.query).pipe(debounceTime(300)),
  { initialValue: '' }
);
```

### pendingUntilEvent for SSR

> ⚠️ **Developer Preview** — `pendingUntilEvent` is in developer preview since v20.0. It may change in future releases.

```typescript
import { pendingUntilEvent } from '@angular/core/rxjs-interop';

// Keeps the app "unstable" until this observable emits, completes, or errors
readonly config$ = this.http.get('/api/config').pipe(pendingUntilEvent());
```

## Error Handling Patterns

### Global error handler

```typescript
import { ErrorHandler, Injectable, inject } from '@angular/core';

@Injectable()
export class GlobalErrorHandler implements ErrorHandler {
  private logger = inject(LoggingService);

  handleError(error: unknown): void {
    this.logger.error('Unhandled error', error);
    // Optionally show user notification
  }
}

// app.config.ts
providers: [
  { provide: ErrorHandler, useClass: GlobalErrorHandler },
]
```

### Component error handling pattern

```typescript
@Component({
  template: `
    @if (error()) {
      <app-error-state
        [message]="error()!"
        (retry)="loadData()"
      />
    } @else if (data()) {
      <app-data-table [data]="data()!" />
    } @else {
      <app-loading-skeleton />
    }
  `,
})
export class DataPageComponent {
  private http = inject(HttpClient);

  data = signal<Data[] | null>(null);
  error = signal<string | null>(null);

  constructor() {
    this.loadData();
  }

  loadData(): void {
    this.error.set(null);
    this.data.set(null);
    this.http.get<Data[]>('/api/data').subscribe({
      next: (data) => this.data.set(data),
      error: (err: HttpErrorResponse) => {
        if (err.status === 0) this.error.set('Network error. Check your connection.');
        else if (err.status === 404) this.error.set('Data not found.');
        else if (err.status >= 500) this.error.set('Server error. Try again later.');
        else this.error.set('An unexpected error occurred.');
      },
    });
  }
}
```

## Testing Overview

Angular 20 uses **Jasmine** as the default test framework with the esbuild-based `@angular/build:unit-test` builder. For comprehensive testing patterns including component, service, pipe, directive, and HTTP testing, see [references/testing.md](../references/testing.md).

## Architecture Patterns

### Smart/Dumb component separation

```
features/
  products/
    product-list.component.ts      ← Smart: injects ProductStore, manages state
    product-card.component.ts      ← Dumb: input(product), output(addToCart)
    product-filter.component.ts    ← Dumb: input(categories), output(filterChange)
    product.store.ts               ← Signal-based store service
```

**Smart components**: inject services, manage state, coordinate child components. Minimal template logic.

**Dumb components**: receive data via `input()`, emit events via `output()`. No injected services. Highly reusable and testable.

### Facade pattern

Wrap complex service interactions behind a facade:

```typescript
@Injectable({ providedIn: 'root' })
export class CheckoutFacade {
  private cartStore = inject(CartStore);
  private orderService = inject(OrderService);
  private notifications = inject(NotificationService);

  readonly cart = this.cartStore.items;
  readonly total = this.cartStore.total;
  readonly isProcessing = signal(false);

  async placeOrder(): Promise<void> {
    this.isProcessing.set(true);
    try {
      await this.orderService.create(this.cart());
      this.cartStore.clear();
      this.notifications.show('Order placed successfully!');
    } catch (e) {
      this.notifications.show('Failed to place order.');
    } finally {
      this.isProcessing.set(false);
    }
  }
}
```

### Feature module organization

```
features/
  feature-name/
    feature-name.routes.ts         # Route definitions
    feature-name.component.ts      # Entry/smart component
    components/                    # Dumb components
    services/                      # Feature-scoped services
    models/                        # Interfaces and types
    utils/                         # Feature-specific utilities
```

## SSR: REQUEST and RESPONSE Tokens

Angular SSR exposes the incoming Node.js `Request` and `Response` objects via injection tokens from `@angular/ssr`. Use them in services, resolvers, or guards that need to read or write HTTP-level data on the server.

> Always inject these with `{ optional: true }` — they are `null` in the browser, so code that uses them must guard accordingly.

### Reading the incoming request

```typescript
import { inject, Injectable, PLATFORM_ID } from '@angular/core';
import { isPlatformServer } from '@angular/common';
import { REQUEST } from '@angular/ssr';

@Injectable({ providedIn: 'root' })
export class LocaleService {
  private platformId = inject(PLATFORM_ID);
  // null in the browser — always use optional injection
  private request = inject(REQUEST, { optional: true });

  getPreferredLocale(): string {
    if (isPlatformServer(this.platformId) && this.request) {
      // Read Accept-Language header on the server
      return this.request.headers.get('accept-language')?.split(',')[0] ?? 'en';
    }
    // Fall back to browser navigator on the client
    return navigator.language ?? 'en';
  }
}
```

### Reading cookies on the server

```typescript
import { inject, Injectable, PLATFORM_ID } from '@angular/core';
import { isPlatformServer } from '@angular/common';
import { REQUEST } from '@angular/ssr';

@Injectable({ providedIn: 'root' })
export class AuthTokenService {
  private platformId = inject(PLATFORM_ID);
  private request = inject(REQUEST, { optional: true });

  getToken(): string | null {
    if (isPlatformServer(this.platformId) && this.request) {
      const cookie = this.request.headers.get('cookie') ?? '';
      const match = cookie.match(/auth_token=([^;]+)/);
      return match?.[1] ?? null;
    }
    // Browser: read from document.cookie or localStorage
    return localStorage.getItem('auth_token');
  }
}
```

### Setting response headers from a service

```typescript
import { inject, Injectable, PLATFORM_ID } from '@angular/core';
import { isPlatformServer } from '@angular/common';
import { RESPONSE } from '@angular/ssr';

@Injectable({ providedIn: 'root' })
export class CacheService {
  private platformId = inject(PLATFORM_ID);
  private response = inject(RESPONSE, { optional: true });

  setPublicCache(maxAge: number): void {
    if (isPlatformServer(this.platformId) && this.response) {
      this.response.headers.set('Cache-Control', `public, max-age=${maxAge}`);
    }
  }
}
```

### Dynamic redirect in a resolver

Read request data to redirect before Angular renders a component:

```typescript
import { inject, PLATFORM_ID } from '@angular/core';
import { ResolveFn, Router } from '@angular/router';
import { isPlatformServer } from '@angular/common';
import { REQUEST } from '@angular/ssr';

export const legacyRedirectResolver: ResolveFn<void> = () => {
  const router = inject(Router);
  const platformId = inject(PLATFORM_ID);
  const request = inject(REQUEST, { optional: true });

  if (isPlatformServer(platformId) && request) {
    const legacyId = new URL(request.url, 'http://localhost').searchParams.get('id');
    if (legacyId) {
      return router.parseUrl(`/items/${legacyId}`);
    }
  }
  return undefined;
};
```

### Token reference

| Token | Type | Available | Purpose |
|---|---|---|---|
| `REQUEST` | `Request \| null` | Server only | Incoming HTTP request (headers, cookies, URL, method) |
| `RESPONSE` | `Response \| null` | Server only | Outgoing HTTP response (set headers, status) |