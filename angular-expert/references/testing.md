# Testing with Jasmine

## Table of Contents
- [Test Setup](#test-setup)
- [Component Testing](#component-testing)
- [Service Testing](#service-testing)
- [Pipe Testing](#pipe-testing)
- [Directive Testing](#directive-testing)
- [HTTP Testing](#http-testing)
- [Testing with Signals](#testing-with-signals)
- [Testing in Zoneless Apps](#testing-in-zoneless-apps)
- [Best Practices](#best-practices)

---

## Test Setup

Angular 20 uses Jasmine as the default test framework with the esbuild-based `@angular/build:unit-test` builder (Karma is legacy). Tests are configured via `angular.json`:

```json
{
  "test": {
    "builder": "@angular/build:unit-test",
    "options": {
      "tsConfig": "tsconfig.spec.json"
    }
  }
}
```

For zoneless apps, remove `zone.js` and `zone.js/testing` from polyfills and add `provideZonelessChangeDetection()` in test setup.

Run tests:

```bash
ng test              # Watch mode
ng test --no-watch   # Single run (CI)
ng test --code-coverage  # With coverage report
```

### tsconfig.spec.json

```json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "outDir": "./out-tsc/spec",
    "types": ["jasmine"]
  },
  "include": ["src/**/*.spec.ts", "src/**/*.d.ts"]
}
```

## Component Testing

### Basic component test

```typescript
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { UserCardComponent } from './user-card.component';

describe('UserCardComponent', () => {
  let component: UserCardComponent;
  let fixture: ComponentFixture<UserCardComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [UserCardComponent],
    }).compileComponents();

    fixture = TestBed.createComponent(UserCardComponent);
    component = fixture.componentInstance;
  });

  it('should create', () => {
    expect(component).toBeTruthy();
  });

  it('should display user name', () => {
    fixture.componentRef.setInput('name', 'Alice');
    fixture.detectChanges();

    const el: HTMLElement = fixture.nativeElement;
    expect(el.textContent).toContain('Alice');
  });
});
```

### Setting inputs

Always use `setInput()` for signal-based inputs — it properly triggers change detection:

```typescript
// Correct — works with signal inputs
fixture.componentRef.setInput('name', 'Alice');
fixture.componentRef.setInput('showActions', false);

// Then trigger change detection
fixture.detectChanges();
```

### Testing outputs

```typescript
it('should emit addToCart when button clicked', () => {
  fixture.componentRef.setInput('product', mockProduct);
  fixture.detectChanges();

  spyOn(component.addToCart, 'emit');

  const button = fixture.nativeElement.querySelector('button');
  button.click();

  expect(component.addToCart.emit).toHaveBeenCalledWith(mockProduct);
});
```

### Testing with dependencies

```typescript
describe('DashboardComponent', () => {
  let mockUserService: jasmine.SpyObj<UserService>;

  beforeEach(async () => {
    mockUserService = jasmine.createSpyObj('UserService', ['getUsers']);
    mockUserService.getUsers.and.returnValue(of([{ id: '1', name: 'Alice' }]));

    await TestBed.configureTestingModule({
      imports: [DashboardComponent],
      providers: [
        { provide: UserService, useValue: mockUserService },
      ],
    }).compileComponents();
  });

  it('should load users on init', () => {
    const fixture = TestBed.createComponent(DashboardComponent);
    fixture.detectChanges();

    expect(mockUserService.getUsers).toHaveBeenCalled();
    const el: HTMLElement = fixture.nativeElement;
    expect(el.textContent).toContain('Alice');
  });
});
```

### Testing template control flow

```typescript
it('should show spinner when loading', () => {
  fixture.componentRef.setInput('isLoading', true);
  fixture.detectChanges();

  expect(fixture.nativeElement.querySelector('app-spinner')).toBeTruthy();
  expect(fixture.nativeElement.querySelector('.content')).toBeNull();
});

it('should show content when loaded', () => {
  fixture.componentRef.setInput('isLoading', false);
  fixture.componentRef.setInput('data', mockData);
  fixture.detectChanges();

  expect(fixture.nativeElement.querySelector('app-spinner')).toBeNull();
  expect(fixture.nativeElement.querySelector('.content')).toBeTruthy();
});
```

## Service Testing

### Simple service

```typescript
import { TestBed } from '@angular/core/testing';
import { TodoStore } from './todo.store';

describe('TodoStore', () => {
  let store: TodoStore;

  beforeEach(() => {
    TestBed.configureTestingModule({});
    store = TestBed.inject(TodoStore);
  });

  describe('initial state', () => {
    it('should start with empty todos', () => {
      expect(store.todos().length).toBe(0);
    });
  });

  describe('addTodo', () => {
    it('should add a todo to the list', () => {
      store.addTodo('Test task');

      expect(store.todos().length).toBe(1);
      expect(store.todos()[0].title).toBe('Test task');
      expect(store.todos()[0].completed).toBeFalse();
    });
  });

  describe('toggleTodo', () => {
    it('should toggle completion status', () => {
      store.addTodo('Task 1');
      store.addTodo('Task 2');
      store.toggleTodo(store.todos()[0].id);

      expect(store.pendingCount()).toBe(1);
      expect(store.completedCount()).toBe(1);
    });
  });
});
```

### Service with HttpClient

```typescript
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
    httpTesting.verify();
  });

  describe('getUser', () => {
    it('should fetch a user by id', () => {
      const mockUser = { id: '1', name: 'Alice' };

      service.getUser('1').subscribe(user => {
        expect(user).toEqual(mockUser);
      });

      const req = httpTesting.expectOne('/api/users/1');
      expect(req.request.method).toBe('GET');
      req.flush(mockUser);
    });

    it('should handle 404 errors for unknown id', () => {
      service.getUser('999').subscribe({
        error: (err) => {
          expect(err.status).toBe(404);
        },
      });

      const req = httpTesting.expectOne('/api/users/999');
      req.flush('Not found', { status: 404, statusText: 'Not Found' });
    });
  });
});
```

## Pipe Testing

Pipes are easy to test directly — no TestBed needed for pure pipes:

```typescript
import { TruncatePipe } from './truncate.pipe';

describe('TruncatePipe', () => {
  const pipe = new TruncatePipe();

  describe('transform', () => {
    it('should return original string if shorter than max', () => {
      expect(pipe.transform('Hello', 10)).toBe('Hello');
    });

    it('should truncate long strings', () => {
      expect(pipe.transform('Hello World', 5)).toBe('Hello...');
    });

    it('should use custom suffix', () => {
      expect(pipe.transform('Hello World', 5, '…')).toBe('Hello…');
    });

    it('should handle empty strings', () => {
      expect(pipe.transform('', 10)).toBe('');
    });
  });
});
```

For pipes that use `inject()`, use TestBed:

```typescript
describe('RelativeTimePipe', () => {
  let pipe: RelativeTimePipe;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [{ provide: LOCALE_ID, useValue: 'en-US' }],
    });
    pipe = TestBed.inject(RelativeTimePipe);
  });

  describe('transform', () => {
    it('should format relative time', () => {
      const yesterday = new Date(Date.now() - 86400000);
      expect(pipe.transform(yesterday)).toContain('yesterday');
    });
  });
});
```

## Directive Testing

Use a test host component to test directives:

```typescript
import { Component } from '@angular/core';
import { TestBed } from '@angular/core/testing';
import { HighlightDirective } from './highlight.directive';

@Component({
  imports: [HighlightDirective],
  template: `<p appHighlight="lightblue">Test</p>`,
})
class TestHostComponent {}

describe('HighlightDirective', () => {
  describe('applied to element', () => {
    it('should apply background color', () => {
      @Component({
        imports: [HighlightDirective],
        template: `<p appHighlight="lightblue">Test</p>`,
      })
      class TestHostComponent {}

      const fixture = TestBed.configureTestingModule({
        imports: [TestHostComponent],
      }).createComponent(TestHostComponent);

      fixture.detectChanges();

      const p = fixture.nativeElement.querySelector('p');
      expect(p.style.backgroundColor).toBe('lightblue');
    });
  });
});
```

## HTTP Testing

### Testing interceptors

```typescript
import { HttpClient } from '@angular/common/http';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { provideHttpClientTesting, HttpTestingController } from '@angular/common/http/testing';
import { authInterceptor } from './auth.interceptor';

describe('authInterceptor', () => {
  let http: HttpClient;
  let httpTesting: HttpTestingController;
  let mockAuth: jasmine.SpyObj<AuthService>;

  beforeEach(() => {
    mockAuth = jasmine.createSpyObj('AuthService', ['getToken']);

    TestBed.configureTestingModule({
      providers: [
        provideHttpClient(withInterceptors([authInterceptor])),
        provideHttpClientTesting(),
        { provide: AuthService, useValue: mockAuth },
      ],
    });

    http = TestBed.inject(HttpClient);
    httpTesting = TestBed.inject(HttpTestingController);
  });

  afterEach(() => httpTesting.verify());

  describe('intercept', () => {
    it('should add Authorization header when token exists', () => {
      mockAuth.getToken.and.returnValue('test-token');

      http.get('/api/data').subscribe();

      const req = httpTesting.expectOne('/api/data');
      expect(req.request.headers.get('Authorization')).toBe('Bearer test-token');
      req.flush({});
    });

    it('should not add header when no token', () => {
      mockAuth.getToken.and.returnValue(null);

      http.get('/api/data').subscribe();

      const req = httpTesting.expectOne('/api/data');
      expect(req.request.headers.has('Authorization')).toBeFalse();
      req.flush({});
    });
  });
});
```

## Testing with Signals

### Testing signal-based state

```typescript
describe('CartStore', () => {
  describe('addItem', () => {
    it('should update computed signals when source changes', () => {
      const store = TestBed.inject(CartStore);

      store.addItem({ id: '1', name: 'Widget', price: 10, quantity: 2 });
      store.addItem({ id: '2', name: 'Gadget', price: 20, quantity: 1 });

      expect(store.itemCount()).toBe(3);
      expect(store.total()).toBe(40);
    });
  });
});
```

### Testing components with signal inputs

```typescript
describe('UserCardComponent', () => {
  describe('name input', () => {
    it('should react to input changes', () => {
      const fixture = TestBed.createComponent(UserCardComponent);

      fixture.componentRef.setInput('name', 'Alice');
      fixture.detectChanges();
      expect(fixture.nativeElement.textContent).toContain('Alice');

      fixture.componentRef.setInput('name', 'Bob');
      fixture.detectChanges();
      expect(fixture.nativeElement.textContent).toContain('Bob');
    });
  });
});
```

### Testing linkedSignal

```typescript
describe('ListStore', () => {
  describe('linkedSignal selection', () => {
    it('should reset selection when items change', () => {
      const store = TestBed.inject(ListStore);

      store.setItems(['Apple', 'Banana', 'Cherry']);
      expect(store.selectedItem()).toBe('Apple');

      store.selectItem('Cherry');
      expect(store.selectedItem()).toBe('Cherry');

      // When items change, selection resets to first if previous is gone
      store.setItems(['Banana', 'Date']);
      expect(store.selectedItem()).toBe('Banana');
    });
  });
});
```

## Testing in Zoneless Apps

When using `provideZonelessChangeDetection()`, follow these rules:

### Setup

```typescript
describe('MyComponent', () => {
  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [MyComponent],
      providers: [provideZonelessChangeDetection()],
    }).compileComponents();
  });

  describe('async rendering', () => {
    it('should update after async operation', async () => {
      const fixture = TestBed.createComponent(MyComponent);
      fixture.componentRef.setInput('userId', '1');

      // Wait for all pending async work to complete
      await fixture.whenStable();

      expect(fixture.nativeElement.textContent).toContain('Alice');
    });
  });
});
```

### TestBed.tick()

In Angular 20, `TestBed.tick()` replaces the removed `TestBed.flushEffects()`. Use it to flush pending effects and signal notifications in tests:

```typescript
it('should update after effect runs', () => {
  const fixture = TestBed.createComponent(MyComponent);
  fixture.componentRef.setInput('data', newData);

  TestBed.tick(); // Flush effects and signal notifications

  expect(fixture.nativeElement.textContent).toContain('updated');
});
```

### Key differences from zone-based testing

| Zone-based | Zoneless |
|---|---|
| `fixture.detectChanges()` triggers CD | `fixture.detectChanges()` still works |
| Zone auto-detects async | Must use signals or `markForCheck()` |
| `fakeAsync` / `tick` work | `fakeAsync` / `tick` do NOT work — use `async`/`await` |
| `TestBed.flushEffects()` | Removed — use `TestBed.tick()` |
| | Prefer `await fixture.whenStable()` for realistic testing |

## Best Practices

### Structure

- Place test files next to source: `user.service.ts` → `user.service.spec.ts`
- **MUST create a `describe` block for each method** — all `it` tests for a method go inside its `describe`
- Use `beforeEach` for shared setup, avoid duplicating setup across tests
- One assertion per `it` block when possible — makes failures easier to diagnose

#### Required structure pattern

```typescript
describe('UserService', () => {
  describe('getUser', () => {           // Method under test
    it('should return user by id', () => { /* ... */ });
    it('should throw 404 for unknown id', () => { /* ... */ });
  });

  describe('createUser', () => {        // Method under test
    it('should send POST request', () => { /* ... */ });
    it('should return the created user', () => { /* ... */ });
  });

  describe('deleteUser', () => {         // Method under test
    it('should send DELETE request', () => { /* ... */ });
    it('should handle not found gracefully', () => { /* ... */ });
  });
});
```

This applies to:
- **Services**: `describe` per public method
- **Components**: `describe` per input/output/lifecycle hook/public method
- **Pipes**: `describe` per `transform` method (or other public methods)
- **Directives**: `describe` per hook or event handler

### Naming

```typescript
describe('UserService', () => {
  describe('getUser', () => {
    it('should return user by id', () => { /* ... */ });
    it('should throw 404 for unknown id', () => { /* ... */ });
  });

  describe('createUser', () => {
    it('should send POST request', () => { /* ... */ });
    it('should return the created user', () => { /* ... */ });
  });
});
```

### Mocking

- Use `jasmine.createSpyObj()` for service mocks
- Use `spyOn()` for partial mocking of real objects
- Use `provideHttpClientTesting()` for HTTP mocks — never mock `HttpClient` directly
- Prefer `{ provide: Service, useValue: mock }` over `{ provide: Service, useClass: MockService }`

### Async testing

```typescript
describe('DataService', () => {
  describe('getData', () => {
    it('should handle async data', (done: DoneFn) => {
      service.getData().subscribe(data => {
        expect(data.length).toBe(3);
        done();
      });

      httpTesting.expectOne('/api/data').flush([1, 2, 3]);
    });
  });

  describe('createUser', () => {
    it('should handle promise', async () => {
      const result = await service.createUser({ name: 'Alice' });
      expect(result.id).toBeDefined();
    });
  });
});
```

### Common gotchas

- Always call `httpTesting.verify()` in `afterEach` to catch unexpected requests
- Call `fixture.detectChanges()` after `setInput()` — inputs alone don't trigger rendering
- In zoneless tests, `fakeAsync()` and `tick()` do NOT work. Use real `async`/`await` with `fixture.whenStable()`
- When testing components with `OnPush`, always use `fixture.componentRef.setInput()` instead of directly assigning properties
- Use `fixture.nativeElement.querySelector()` to query rendered DOM — don't test internal component state directly

## Testing SSR-Specific Behaviour

### Testing isPlatformBrowser guards

Provide a fake `PLATFORM_ID` to simulate server vs browser environments in the same test suite:

```typescript
import { PLATFORM_ID } from '@angular/core';

// Use string literals — PLATFORM_BROWSER_ID/PLATFORM_SERVER_ID are not public API
const PLATFORM_BROWSER = 'browser';
const PLATFORM_SERVER = 'server';

describe('ThemeService', () => {
  describe('browser environment', () => {
    beforeEach(() => {
      TestBed.configureTestingModule({
        providers: [{ provide: PLATFORM_ID, useValue: PLATFORM_BROWSER }],
      });
    });

    describe('getTheme', () => {
      it('should read from localStorage', () => {
        localStorage.setItem('theme', 'dark');
        const service = TestBed.inject(ThemeService);
        expect(service.getTheme()).toBe('dark');
      });
    });
  });

  describe('server environment', () => {
    beforeEach(() => {
      TestBed.configureTestingModule({
        providers: [{ provide: PLATFORM_ID, useValue: PLATFORM_SERVER }],
      });
    });

    describe('getTheme', () => {
      it('should return default theme without touching localStorage', () => {
        const service = TestBed.inject(ThemeService);
        expect(service.getTheme()).toBe('light');
      });
    });
  });
});
```

### Testing TransferState

Provide mock transfer state data to test how a service behaves when a server-transferred value is present:

```typescript
import { TransferState, makeStateKey } from '@angular/core';

const CONFIG_KEY = makeStateKey<AppConfig>('appConfig');

describe('ConfigService', () => {
  describe('with TransferState', () => {
    let transferState: TransferState;

    beforeEach(() => {
      TestBed.configureTestingModule({
        providers: [
          { provide: PLATFORM_ID, useValue: PLATFORM_BROWSER },
          provideHttpClient(),
          provideHttpClientTesting(),
        ],
      });
      transferState = TestBed.inject(TransferState);
    });

    describe('getConfig', () => {
      it('should use transferred config instead of fetching', () => {
        const mockConfig: AppConfig = { apiUrl: 'https://api.example.com', production: true };
        transferState.set(CONFIG_KEY, mockConfig);

        const service = TestBed.inject(ConfigService);
        let result!: AppConfig;
        service.getConfig().subscribe(c => (result = c));

        // No HTTP request should be made
        TestBed.inject(HttpTestingController).expectNone('/api/config');
        expect(result).toEqual(mockConfig);
      });

      it('should fetch config when transfer state is empty', () => {
        const service = TestBed.inject(ConfigService);
        const mockConfig: AppConfig = { apiUrl: 'https://api.example.com', production: false };

        service.getConfig().subscribe();

        const req = TestBed.inject(HttpTestingController).expectOne('/api/config');
        req.flush(mockConfig);
      });
    });
  });
});
```

### Testing REQUEST token (server-side services)

Mock the `REQUEST` token to test services that read headers or cookies on the server:

```typescript
import { REQUEST } from '@angular/ssr';
import { PLATFORM_ID } from '@angular/core';

// Use string literal — PLATFORM_SERVER_ID is not a public API export
const PLATFORM_SERVER = 'server';

describe('LocaleService', () => {
  describe('server environment', () => {
    const mockRequest = {
      headers: new Headers({ 'accept-language': 'fr-FR,fr;q=0.9' }),
    } as unknown as Request;

    beforeEach(() => {
      TestBed.configureTestingModule({
        providers: [
          { provide: PLATFORM_ID, useValue: PLATFORM_SERVER },
          { provide: REQUEST, useValue: mockRequest },
        ],
      });
    });

    describe('getPreferredLocale', () => {
      it('should extract locale from Accept-Language header', () => {
        const service = TestBed.inject(LocaleService);
        expect(service.getPreferredLocale()).toBe('fr-FR');
      });
    });
  });
});
```