# HTTP & Data Fetching

## Table of Contents
- [HttpClient Setup](#httpclient-setup)
- [GET Requests with Signals](#get-requests-with-signals)
- [Mutations (POST, PUT, DELETE)](#mutations-post-put-delete)
- [SSR & HTTP Transfer Cache](#ssr--http-transfer-cache)
- [Functional Interceptors](#functional-interceptors)
- [Error Handling](#error-handling)
- [Testing HTTP](#testing-http)

---

## HttpClient Setup

Register `HttpClient` in `app.config.ts`:

```typescript
import { provideHttpClient, withInterceptors, withFetch } from '@angular/common/http';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withInterceptors([authInterceptor, errorInterceptor]),
      withFetch(), // Use fetch API instead of XMLHttpRequest
    ),
  ],
};
```

## GET Requests with Signals

Use `HttpClient` with `toSignal()` to convert Observable responses into signals for template consumption:

```typescript
import { Component, inject, input, computed } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { toSignal } from '@angular/core/rxjs-interop';
import { switchMap } from 'rxjs';
import { toObservable } from '@angular/core/rxjs-interop';

@Component({
  selector: 'app-user-profile',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    @if (user()) {
      <h1>{{ user()!.name }}</h1>
    } @else {
      <app-spinner />
    }
  `,
})
export class UserProfileComponent {
  private http = inject(HttpClient);
  userId = input.required<string>();

  // Reactive: refetches whenever userId changes
  user = toSignal(
    toObservable(this.userId).pipe(
      switchMap(id => this.http.get<User>(`/api/users/${id}`)),
    ),
  );
}
```

### Service-based data fetching pattern

Encapsulate HTTP calls in a service and expose signals:

```typescript
@Injectable({ providedIn: 'root' })
export class UserService {
  private http = inject(HttpClient);

  getUser(id: string) {
    return this.http.get<User>(`/api/users/${id}`);
  }

  getUsers() {
    return this.http.get<User[]>('/api/users');
  }
}
```

Consume in a component:

```typescript
@Component({ /* ... */ })
export class UsersPageComponent {
  private userService = inject(UserService);

  users = toSignal(this.userService.getUsers(), { initialValue: [] });
  isLoaded = computed(() => this.users().length > 0);
}
```

### Reactive search with debounce

```typescript
@Component({ /* ... */ })
export class SearchComponent {
  private http = inject(HttpClient);
  query = signal('');

  results = toSignal(
    toObservable(this.query).pipe(
      debounceTime(300),
      distinctUntilChanged(),
      switchMap(q => q
        ? this.http.get<SearchResult[]>(`/api/search?q=${q}`)
        : of([])
      ),
    ),
    { initialValue: [] },
  );
}
```

## SSR & HTTP Transfer Cache

When Angular SSR renders a page, `HttpClient` GET requests run on the server. Without transfer cache the client would re-fire every one of those requests on boot — doubling API load and causing visible content flicker.

Enable `withHttpTransferCacheOptions()` inside `provideClientHydration()` (in `app.config.ts`) so that GET responses are serialised into the HTML payload and reused on the client:

```typescript
// app.config.ts
provideClientHydration(withHttpTransferCacheOptions())
```

### What is and isn't cached

- All `HttpClient` GET requests made during SSR are cached automatically.
- POST / PUT / PATCH / DELETE requests are **never** cached.
- Raw `fetch()` calls made outside the Angular DI context are **not** cached.

### Filtering cached requests

Use the `filter` option to exclude specific requests from the cache:

```typescript
provideClientHydration(
  withHttpTransferCacheOptions({
    filter: (request) => !request.url.includes('/api/live-scores'),  // Exclude real-time data
  })
)
```

### Differentiating cached requests by header

By default the cache key is the URL only. Include headers in the key when the same URL returns different data per user:

```typescript
provideClientHydration(
  withHttpTransferCacheOptions({ includeHeaders: ['Authorization', 'Accept-Language'] }),
)
```

## Mutations (POST, PUT, DELETE)

Use `HttpClient` directly for mutations. Use `firstValueFrom()` to convert the Observable to a Promise for one-shot operations:

```typescript
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { firstValueFrom } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class UserService {
  private http = inject(HttpClient);

  createUser(data: CreateUserDto): Promise<User> {
    return firstValueFrom(this.http.post<User>('/api/users', data));
  }

  updateUser(id: string, data: Partial<User>): Promise<User> {
    return firstValueFrom(this.http.patch<User>(`/api/users/${id}`, data));
  }

  deleteUser(id: string): Promise<void> {
    return firstValueFrom(this.http.delete<void>(`/api/users/${id}`));
  }
}
```

## Functional Interceptors

Interceptors are functions, registered via `withInterceptors()`.

### Auth token interceptor

```typescript
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { AuthService } from './auth.service';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const auth = inject(AuthService);
  const token = auth.getToken();

  if (token) {
    const cloned = req.clone({
      setHeaders: { Authorization: `Bearer ${token}` },
    });
    return next(cloned);
  }

  return next(req);
};
```

### Logging interceptor

```typescript
import { HttpInterceptorFn } from '@angular/common/http';
import { tap } from 'rxjs';

export const loggingInterceptor: HttpInterceptorFn = (req, next) => {
  const started = performance.now();
  return next(req).pipe(
    tap({
      next: () => {
        const elapsed = performance.now() - started;
        console.log(`${req.method} ${req.url} — ${elapsed.toFixed(0)}ms`);
      },
      error: (err) => {
        console.error(`${req.method} ${req.url} — FAILED`, err);
      },
    }),
  );
};
```

### Retry interceptor

```typescript
import { HttpInterceptorFn } from '@angular/common/http';
import { retry, timer } from 'rxjs';

export const retryInterceptor: HttpInterceptorFn = (req, next) => {
  return next(req).pipe(
    retry({
      count: 3,
      delay: (error, retryCount) => {
        if (error.status >= 500) {
          return timer(retryCount * 1000);
        }
        throw error; // Don't retry client errors
      },
    }),
  );
};
```

### Registration

```typescript
import { provideHttpClient, withInterceptors } from '@angular/common/http';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withInterceptors([authInterceptor, loggingInterceptor, retryInterceptor]),
    ),
  ],
};
```

Interceptors run in the order they are listed. Place auth before logging so the logged request includes the auth header.

## Error Handling

### Centralized error interceptor

```typescript
import { HttpInterceptorFn, HttpErrorResponse } from '@angular/common/http';
import { inject } from '@angular/core';
import { catchError, throwError } from 'rxjs';
import { NotificationService } from './notification.service';

export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  const notifications = inject(NotificationService);

  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      if (error.status === 401) {
        notifications.show('Session expired. Please log in again.');
      } else if (error.status === 403) {
        notifications.show('Access denied.');
      } else if (error.status >= 500) {
        notifications.show('Server error. Please try again later.');
      }
      return throwError(() => error);
    }),
  );
};
```

### Component-level error handling

```typescript
@Component({
  template: `
    @if (error()) {
      <div class="error">
        <p>{{ error() }}</p>
        <button (click)="loadData()">Try Again</button>
      </div>
    } @else if (user()) {
      <app-user-detail [user]="user()!" />
    } @else {
      <app-spinner />
    }
  `,
})
export class UserPageComponent {
  private userService = inject(UserService);
  userId = input.required<string>();

  user = signal<User | null>(null);
  error = signal<string | null>(null);

  constructor() {
    effect(() => this.loadData());
  }

  loadData(): void {
    this.error.set(null);
    this.userService.getUser(this.userId()).subscribe({
      next: (user) => this.user.set(user),
      error: (err: HttpErrorResponse) => {
        if (err.status === 0) this.error.set('Network error. Check your connection.');
        else if (err.status === 404) this.error.set('User not found.');
        else if (err.status >= 500) this.error.set('Server error. Try again later.');
        else this.error.set('An unexpected error occurred.');
      },
    });
  }
}
```

## Testing HTTP

Use `HttpTestingController` to mock HTTP requests:

```typescript
import { TestBed } from '@angular/core/testing';
import { provideHttpClient } from '@angular/common/http';
import { provideHttpClientTesting, HttpTestingController } from '@angular/common/http/testing';

describe('UserService', () => {
  let service: UserService;
  let httpTesting: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [provideHttpClient(), provideHttpClientTesting()],
    });
    service = TestBed.inject(UserService);
    httpTesting = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    httpTesting.verify(); // Ensure no outstanding requests
  });

  it('should fetch a user by id', () => {
    const mockUser: User = { id: '1', name: 'Alice', email: 'alice@example.com' };

    service.getUser('1').subscribe(user => {
      expect(user).toEqual(mockUser);
    });

    const req = httpTesting.expectOne('/api/users/1');
    expect(req.request.method).toBe('GET');
    req.flush(mockUser);
  });

  it('should handle errors', () => {
    service.getUser('999').subscribe({
      error: (err) => {
        expect(err.status).toBe(404);
      },
    });

    const req = httpTesting.expectOne('/api/users/999');
    req.flush('Not found', { status: 404, statusText: 'Not Found' });
  });
});
```

### Testing interceptors

```typescript
it('should add auth header', () => {
  const authService = TestBed.inject(AuthService);
  spyOn(authService, 'getToken').and.returnValue('test-token');

  TestBed.inject(HttpClient).get('/api/data').subscribe();

  const req = httpTesting.expectOne('/api/data');
  expect(req.request.headers.get('Authorization')).toBe('Bearer test-token');
  req.flush({});
});
```

For more testing patterns, see the Angular HttpClient testing guide at https://angular.dev/guide/http/testing.