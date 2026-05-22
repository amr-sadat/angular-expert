# Application Configuration

## Table of Contents
- [Bootstrap Configuration](#bootstrap-configuration)
- [Provider Composition](#provider-composition)
- [Environment Configuration](#environment-configuration)
- [Angular CLI Configuration](#angular-cli-configuration)
- [esbuild Application Builder](#esbuild-application-builder)
- [Hot Module Replacement (HMR)](#hot-module-replacement-hmr)
- [Angular DevTools](#angular-devtools)
- [TypeScript Configuration](#typescript-configuration)

---

## Bootstrap Configuration

Angular 20 applications bootstrap with `bootstrapApplication()` — no `AppModule` needed.

### main.ts

```typescript
import { bootstrapApplication } from '@angular/platform-browser';
import { AppComponent } from './app/app.component';
import { appConfig } from './app/app.config';

bootstrapApplication(AppComponent, appConfig)
  .catch((err) => console.error(err));
```

### app.config.ts

```typescript
import { ApplicationConfig, provideZonelessChangeDetection } from '@angular/core';
import { provideRouter, withComponentInputBinding, withPreloading, PreloadAllModules } from '@angular/router';
import { provideHttpClient, withInterceptors, withFetch } from '@angular/common/http';
import { provideClientHydration, withEventReplay, withHttpTransferCacheOptions, withIncrementalHydration } from '@angular/platform-browser';
import { provideAnimationsAsync } from '@angular/platform-browser/animations/async';
import { routes } from './app.routes';
import { authInterceptor } from './core/http/auth.interceptor';
import { errorInterceptor } from './core/http/error.interceptor';

export const appConfig: ApplicationConfig = {
  providers: [
    // Zoneless change detection (stable in v20, default in v21+)
    provideZonelessChangeDetection(),

    // Router with feature functions
    provideRouter(
      routes,
      withComponentInputBinding(),
      withPreloading(PreloadAllModules),
    ),

    // HTTP client with interceptors and fetch backend
    provideHttpClient(
      withInterceptors([authInterceptor, errorInterceptor]),
      withFetch(),
    ),

    // SSR hydration — required for non-destructive hydration
    // Remove provideClientHydration() entirely for CSR-only apps
    provideClientHydration(
      withHttpTransferCacheOptions(),  // Reuse server HTTP responses on client — avoids duplicate requests
      withEventReplay(),            // Replay user events that fire before hydration completes
      withIncrementalHydration(),   // Enable @defer (hydrate on ...) triggers
    ),

    // Async animations (lazy-loaded, reduces initial bundle)
    provideAnimationsAsync(),
  ],
};
```

### Root component

```typescript
import { Component } from '@angular/core';
import { RouterOutlet } from '@angular/router';

@Component({
  selector: 'app-root',
  imports: [RouterOutlet],
  template: `<router-outlet />`,
})
export class AppComponent {}
```

## Provider Composition

### Available provider functions

| Provider | Purpose |
|---|---|
| `provideZonelessChangeDetection()` | Enable zoneless mode |
| `provideRouter(routes, ...features)` | Configure the router |
| `provideHttpClient(...features)` | Configure HttpClient |
| `provideClientHydration(...features)` | Enable SSR hydration (required for SSR apps) |
| `provideAnimationsAsync()` | Lazy-load animation engine |
| `provideAnimations()` | Eagerly load animation engine |

### provideClientHydration feature functions

| Feature | Purpose |
|---|---|
| `withHttpTransferCacheOptions()` | Reuse server-rendered HTTP GET responses on the client |
| `withEventReplay()` | Capture and replay user events that fire before hydration |
| `withIncrementalHydration()` | Enable `@defer (hydrate on ...)` triggers |
| `withI18nSupport()` | Required when using Angular i18n |

### Router feature functions

| Feature | Purpose |
|---|---|
| `withComponentInputBinding()` | Bind route params/data to component inputs |
| `withPreloading(strategy)` | Configure preloading (e.g. `PreloadAllModules`) |
| `withNavigationErrorHandler(fn)` | Handle navigation errors, optionally redirect |
| `withDebugTracing()` | Log all router events to console |
| `withHashLocation()` | Use hash-based URLs |
| `withViewTransitions()` | Enable View Transitions API for animated route changes |
| `withInMemoryScrolling(options)` | Configure scroll position restoration and anchor scrolling |

### HttpClient feature functions

| Feature | Purpose |
|---|---|
| `withInterceptors([...fns])` | Register functional interceptors |
| `withFetch()` | Use `fetch` API instead of `XMLHttpRequest` |
| `withJsonpSupport()` | Enable JSONP requests |
| `withRequestsMadeViaParent()` | Forward requests to parent injector's HttpClient |

### SSR providers

```typescript
// app.config.server.ts
import { mergeApplicationConfig, ApplicationConfig } from '@angular/core';
import { provideServerRendering } from '@angular/platform-server';
import { provideServerRoutesConfig } from '@angular/ssr';
import { appConfig } from './app.config';
import { serverRoutes } from './app.routes.server';

const serverConfig: ApplicationConfig = {
  providers: [
    provideServerRendering(),
    provideServerRoutesConfig(serverRoutes),
  ],
};

export const config = mergeApplicationConfig(appConfig, serverConfig);
```

## Environment Configuration

### Using environment files

```typescript
// environments/environment.ts (development)
export const environment = {
  production: false,
  apiUrl: 'http://localhost:3000/api',
  featureFlags: {
    newDashboard: true,
  },
};
```

```typescript
// environments/environment.prod.ts (production)
export const environment = {
  production: true,
  apiUrl: 'https://api.example.com',
  featureFlags: {
    newDashboard: false,
  },
};
```

### File replacement in angular.json

```json
{
  "configurations": {
    "production": {
      "fileReplacements": [
        {
          "replace": "src/environments/environment.ts",
          "with": "src/environments/environment.prod.ts"
        }
      ]
    }
  }
}
```

### Providing environment as a token

```typescript
import { InjectionToken } from '@angular/core';
import { environment } from '../environments/environment';

export interface Environment {
  production: boolean;
  apiUrl: string;
  featureFlags: Record<string, boolean>;
}

export const ENVIRONMENT = new InjectionToken<Environment>('environment');

// In app.config.ts
providers: [
  { provide: ENVIRONMENT, useValue: environment },
]

// In services
@Injectable({ providedIn: 'root' })
export class ApiService {
  private env = inject(ENVIRONMENT);
  private baseUrl = this.env.apiUrl;
}
```

## Angular CLI Configuration

### Key angular.json settings

```json
{
  "projects": {
    "my-app": {
      "architect": {
        "build": {
          "builder": "@angular/build:application",
          "options": {
            "outputPath": "dist/my-app",
            "index": "src/index.html",
            "browser": "src/main.ts",
            "server": "src/main.server.ts",
            "tsConfig": "tsconfig.app.json",
            "assets": [
              { "glob": "**/*", "input": "public" }
            ],
            "styles": ["src/styles.css"],
            "scripts": []
          },
          "configurations": {
            "production": {
              "budgets": [
                {
                  "type": "initial",
                  "maximumWarning": "500kB",
                  "maximumError": "1MB"
                }
              ],
              "outputHashing": "all"
            },
            "development": {
              "optimization": false,
              "extractLicenses": false,
              "sourceMap": true
            }
          }
        },
        "test": {
          "builder": "@angular/build:unit-test",
          "options": {
            "tsConfig": "tsconfig.spec.json"
          }
        }
      }
    }
  }
}
```

### CLI commands reference

| Command | Purpose |
|---|---|
| `ng new my-app --ssr` | Create new project with SSR |
| `ng generate component features/users/user-card` | Generate component |
| `ng generate service core/auth/auth` | Generate service |
| `ng generate pipe shared/pipes/truncate` | Generate pipe |
| `ng generate directive shared/directives/highlight` | Generate directive |
| `ng generate guard core/auth/auth` | Generate guard |
| `ng build` | Production build |
| `ng serve` | Development server |
| `ng test` | Run tests |

## esbuild Application Builder

Angular 20 projects use **`@angular/build:application`** by default — an esbuild + Vite-based builder that handles the browser bundle, server bundle, prerendering, and dev server in a single target. The legacy webpack builder (`@angular-devkit/build-angular:browser`) is still supported but is not generated by `ng new`.

### Identify which builder a project uses

Open `angular.json` and inspect the `architect.build.builder` value:

```json
{
  "projects": {
    "my-app": {
      "architect": {
        "build": {
          "builder": "@angular/build:application"
        }
      }
    }
  }
}
```

| Builder value | Build system | Status |
|---|---|---|
| `@angular/build:application` | esbuild + Vite | Default in v20, recommended |
| `@angular-devkit/build-angular:application` | esbuild + Vite (older alias) | Migrate to `@angular/build:application` |
| `@angular-devkit/build-angular:browser-esbuild` | esbuild (no server target) | Legacy interim, migrate to `application` |
| `@angular-devkit/build-angular:browser` | webpack | Legacy, opt-in only |

### What changed from the webpack builder

- **Single target for browser + server** — One `application` target replaces the separate `browser` and `server` targets. Use `server` and `ssr.entry` options inside `application` instead of a second builder.
- **`main` → `browser`** — The entry point option is named `browser` (and `server` for SSR), not `main`.
- **`assets` shape** — Assets must be objects (`{ "glob": "**/*", "input": "public" }`); plain string paths from the webpack builder are not accepted.
- **No `@types` global include by default** — Add type packages explicitly to `tsconfig.app.json`'s `types` array.
- **Built-in prerendering and SSG** — Configure via `prerender` and `outputMode` on the same target; no separate `prerender` builder is needed.

### When to opt back to webpack

Almost never. Reach for `@angular-devkit/build-angular:browser` only if you depend on:

- A custom webpack config via `@angular-builders/custom-webpack` that cannot be expressed with esbuild plugins.
- A third-party loader chain (e.g. legacy Module Federation setups) that has no esbuild equivalent yet.

New projects, libraries, and migrations should stay on `@angular/build:application`.

## Hot Module Replacement (HMR)

`ng serve` enables HMR **by default** in Angular 20. Saving a file patches the running app in place instead of doing a full page reload, preserving most in-memory state.

### What survives an HMR reload

- **Component signal state** — `signal()`, `computed()`, and `linkedSignal()` values inside live component instances persist across template/style edits.
- **Router URL and query params** — The current route stays put; no navigation is triggered.
- **Open `@defer` blocks** — Loaded blocks stay loaded; Angular eagerly resolves all `@defer` dependencies in HMR mode so swaps are instant. Trigger conditions (`on viewport`, `on hover`, etc.) still gate the visible render.
- **Injected singleton services** — Services registered with `providedIn: 'root'` retain their internal state as long as their source file is not edited.

### What forces a full reload

- Changes to `main.ts`, `app.config.ts`, or any provider tree (`provideRouter`, `provideHttpClient`, etc.) — provider graphs are built once at bootstrap, so HMR cannot patch them.
- Changes to `app.routes.ts` and other route config arrays — route definitions are captured by `provideRouter()` at startup.
- Edits to a service file whose state should not silently mutate — Angular reloads to avoid stale references.
- TypeScript compilation errors — the dev server holds the reload until the next clean build.

### Disabling HMR

```bash
ng serve --no-hmr
# or
ng serve --hmr=false
```

Persist the choice in `angular.json` under the `serve` target:

```json
{
  "architect": {
    "serve": {
      "builder": "@angular/build:dev-server",
      "options": {
        "hmr": false
      }
    }
  }
}
```

## Angular DevTools

Angular DevTools is a browser extension (Chrome Web Store and Firefox Add-ons) that attaches to any Angular app running in development mode. It adds an **Angular** panel to the browser devtools.

### What it shows

- **Component tree** — Live hierarchy of standalone components and directives, with their inputs, outputs, and host bindings inspectable on selection.
- **Signal graph** — For zoneless and signal-based apps, visualises producer/consumer relationships between `signal()`, `computed()`, and `effect()` nodes, and highlights which signals drove the latest update.
- **Profiler** — Records change-detection cycles, attributing time to specific components and template bindings. Useful for spotting unexpectedly frequent renders.
- **Router tree** — The current route configuration and active route snapshot.

### What is degraded in zoneless apps

- The classic **zone-based change-detection visualisation** (highlighting components that ran inside an `NgZone` task) does not apply — there is no zone to instrument. The profiler still records change-detection runs, but they are attributed to signal writes and explicit `markForCheck()` calls rather than to zone events.
- Patterns that depended on `NgZone.onStable` for instrumentation no longer appear in the timeline; use the signal graph and profiler instead.

DevTools requires a **development build** (`ng serve` or `ng build --configuration development`). Production builds strip the runtime hooks DevTools relies on.

## TypeScript Configuration

### Recommended tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ES2022",
    "moduleResolution": "bundler",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "strict": true,
    "noImplicitOverride": true,
    "noPropertyAccessFromIndexSignature": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "sourceMap": true,
    "declaration": false,
    "experimentalDecorators": true,
    "isolatedModules": true,
    "baseUrl": "./",
    "paths": {
      "@core/*": ["src/app/core/*"],
      "@shared/*": ["src/app/shared/*"],
      "@features/*": ["src/app/features/*"],
      "@env": ["src/environments/environment"]
    }
  },
  "angularCompilerOptions": {
    "strictInjectionParameters": true,
    "strictInputAccessModifiers": true,
    "strictTemplates": true
  }
}
```

### Path aliases

Use TypeScript path aliases for clean imports:

```typescript
// Instead of: import { AuthService } from '../../../core/auth/auth.service';
import { AuthService } from '@core/auth/auth.service';

// Instead of: import { TruncatePipe } from '../../shared/pipes/truncate.pipe';
import { TruncatePipe } from '@shared/pipes/truncate.pipe';
```