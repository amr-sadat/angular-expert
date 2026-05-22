# configuration.md HMR + esbuild builder + DevTools extension

Status: ready-for-agent
Type: AFK

## Parent

`.scratch/angular-20-missing-features/PRD.md`

## What to build

Extend `references/configuration.md` with three new subsections.

Topics:

- **Stable HMR** — Angular 20 enables HMR by default with `ng serve`. Cover what survives reload (component state for `signal`-backed state), what does not (provider tree changes, route config changes), and how to disable if needed.
- **esbuild application builder** — `@angular/build:application` is the default in v20 projects. Cover what changed from `@angular-devkit/build-angular:browser` (the legacy webpack builder), how to identify which builder a project uses, and when to opt back to webpack (almost never).
- **Angular DevTools** — short pointer with what it shows in a zoneless app (signal graph, component tree, profiler) and what's degraded (zone-based change detection visualisation does not apply).

`SKILL.md` hookup:

- No new top-level section. Configuration topic already exists.
- Optionally add an HMR-related gotcha if a real footgun surfaces during writing (e.g. "HMR does not re-run `bootstrapApplication` providers — full reload required for provider changes").

Source verification: query angular.dev via context7 for the builder package name, current CLI flags, and HMR documentation.

## Acceptance criteria

- [ ] `references/configuration.md` gains three new subsections with code/CLI examples
- [ ] Builder name and CLI flags exactly match angular.dev
- [ ] All claims verified via context7
- [ ] Any new gotchas added to `SKILL.md`
- [ ] No content changes to unrelated `configuration.md` sections

## Blocked by

- Issue #01 (animations pilot) — for style precedent
