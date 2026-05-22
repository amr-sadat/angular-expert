# PRD: Angular 20 Missing Features in `angular-expert` Skill

Status: ready-for-agent

## Problem Statement

The `angular-expert` skill claims to be an authoritative Angular 20 reference for writing, debugging, and reviewing Angular code. In practice it has notable gaps: several stable Angular 20 features have no coverage, and a few are referenced only by name in provider configs. A reviewer relying on this skill today would miss legitimate v20 issues — wrong animations syntax, missing i18n hooks, absent a11y attributes, stale build/HMR guidance, and others — because the skill does not teach the relevant patterns.

The skill explicitly excludes experimental APIs (Signal Forms, `resource`, `httpResource`, `rxResource`). That exclusion is correct and stays in place. The problem is strictly about **stable** v20 surface that the skill does not yet cover.

## Solution

Extend the skill to cover the missing stable v20 features the user actually needs for review-quality guidance, without ballooning the skill into a Material/Service Worker/Web Components catch-all. Two new reference files (animations, i18n), five extensions to existing reference files, and a corresponding update to `SKILL.md` so the new patterns surface in the principles, gotchas, and anti-patterns tables — not just as deep links.

## User Stories

1. As a reviewer using the skill, I want guidance on Angular 20's `animate.enter` / `animate.leave` template syntax, so that I can flag code still using the classic imperative `@angular/animations` `trigger()` API for zoneless apps.
2. As a reviewer, I want the skill to teach the View Transitions API integration with the router, so that I can review SPA navigation animations correctly.
3. As an Angular 20 developer, I want to know how `provideAnimationsAsync` and `provideAnimations` differ in bundle impact, so that I pick the right one.
4. As a developer building an internationalised app, I want clear guidance on `@angular/localize` setup, message extraction, runtime translation loading, and `$localize` tagged templates, so that I do not roll my own i18n.
5. As an SSR developer, I want the skill to call out `withI18nSupport()` in the hydration provider list, so that hydration does not break on translated text nodes.
6. As a reviewer, I want basic a11y patterns (semantic landmarks, ARIA labels, keyboard handling, focus management with `inject(ElementRef)` / native APIs), so that I can flag inaccessible components without pulling in CDK a11y.
7. As a developer using signals, I want to know when to reach for `afterRenderEffect` versus `effect`, so that I correctly schedule DOM-reading side effects after a render pass.
8. As a developer working on the dev loop, I want guidance on Angular 20 stable HMR, so that I configure it correctly and know its limitations (component state preserved, route/provider changes not).
9. As a developer setting up a new project, I want the skill to document the esbuild-based application builder (`@angular/build:application`), so that I do not configure the legacy webpack builder by mistake.
10. As a developer debugging signal flow, I want a pointer to Angular DevTools and what it shows in a zoneless app, so that I can diagnose without running blind.
11. As a developer using `@defer`, I want a full table of hydrate triggers (`hydrate on idle`, `hydrate on viewport`, `hydrate on interaction`, `hydrate on hover`, `hydrate on immediate`, `hydrate never`), so that I pick the right one for each deferred block.
12. As a developer wrapping a third-party input, I want a reference implementation of a custom `ControlValueAccessor` that integrates cleanly with Reactive Forms and signals, so that I do not have to reverse-engineer the contract.
13. As a reviewer, I want the SKILL.md `Gotchas` section to call out new v20 traps (e.g. classic animations are zone-dependent; `afterRenderEffect` vs `effect` confusion), so that the most reviewable failures surface at the top level.
14. As a reviewer, I want the `Quick Reference: What NOT to Do` table extended (e.g. classic `trigger()` animation → `animate.enter` class), so that v20 anti-patterns get caught in a single scan.
15. As a maintainer of the skill, I want every new pattern to cite angular.dev (verified via context7), so that future readers can trust the authority claim in the skill's preamble.

## Implementation Decisions

### Module map

**New reference files**

- `references/animations.md` — `animate.enter` / `animate.leave` template syntax, View Transitions API with `withViewTransitions()` router feature, `provideAnimationsAsync` vs `provideAnimations` tradeoffs, classic imperative API explicitly out of scope.
- `references/i18n.md` — `@angular/localize` setup, `$localize` tagged templates, `i18n` attribute, message extraction (`ng extract-i18n`), runtime translation loading, SSR `withI18nSupport()` hookup.

**Extensions to existing files**

- `references/components.md` — basic a11y patterns: semantic HTML defaults, `host` ARIA bindings, keyboard event handlers via `host` listeners, focus management via `ElementRef` + native `.focus()`. Does **not** introduce CDK a11y.
- `references/signals-and-state.md` — `afterRenderEffect` section with decision rules vs `effect` and `afterRenderEffect` phases (`earlyRead`, `write`, `mixedReadWrite`, `read`).
- `references/configuration.md` — Angular 20 stable HMR (enabled by default with `ng serve`, what survives reload, what does not), esbuild application builder (`@angular/build:application` as default, legacy webpack opt-out), Angular DevTools pointer.
- `references/ssr-and-performance.md` — full `@defer (hydrate on ...)` trigger table with semantics and recommended use cases.
- `references/forms.md` — custom `ControlValueAccessor` implementation with `NG_VALUE_ACCESSOR` provider, signal-based internal state, integration with Reactive Forms.

**SKILL.md updates**

- New short prose sections for animations and i18n, each ending with a `> Read [references/...](...)` pointer matching the existing pattern.
- `Gotchas` section: add entries for classic animations being zone-dependent, `afterRenderEffect` vs `effect` confusion, missing `withI18nSupport()` on hydrated i18n apps.
- `Quick Reference: What NOT to Do` table: add classic `trigger()` animations → `animate.enter`, `@HostBinding('@fade')` → `animate.enter` class, manual `intersectionObserver` for hydration → `@defer (hydrate on viewport)`.
- `References` list at the bottom: add `animations.md` and `i18n.md` entries.

### Out-of-scope exclusions (locked decisions from grilling)

- **Experimental APIs** stay excluded: Signal Forms, `resource()`, `httpResource()`, `rxResource()`.
- **Classic imperative animations API** stays excluded (no `trigger`, `state`, `animate`, `keyframes`, `query`, `stagger`).
- **Adjacent libraries** stay excluded: Angular Material, CDK overlay/dragdrop/portal/a11y module, Service Worker / PWA, Web Components / `createCustomElement`, CSP / Trusted Types, schematics authoring.
- The a11y extension covers patterns achievable with plain Angular only — CDK a11y is not introduced.

### Source authority

Every new pattern is verified against current angular.dev docs via the context7 MCP server before being written. The skill's preamble already claims angular.dev as the source of truth; this PRD preserves that claim.

### Execution sequencing

Pilot pass: write `references/animations.md` end-to-end, plus its SKILL.md hookup (prose section + Gotchas + Quick Reference entries + References list line). Pause for the user's review of style, depth, and density. Iterate if needed.

Full pass after pilot approval: remaining six work items (`i18n.md`, five file extensions, remaining SKILL.md updates).

### Style invariants (from the existing skill)

- Match existing reference file density (~400–550 lines), code-example heavy, anti-patterns called out inline.
- Use stable APIs only; mark Developer Preview APIs with explicit warnings if any are unavoidable.
- Keep TypeScript examples strictly typed (no `any`), standalone-by-default, `OnPush`, zoneless-compatible, native control flow.

## Testing Decisions

This is a documentation skill, not application code. There are no unit tests.

Verification instead happens as:

1. **Source verification** — every API signature and provider name is grepped against the response from a context7 `query-docs` call before being written into a reference file.
2. **Cross-reference check** — `SKILL.md`'s new sections, Gotchas entries, Quick Reference rows, and References list links resolve to existing files and existing anchors.
3. **No regression in existing files** — extensions to existing reference files add sections at the end (or in the topically-correct place) without rewording adjacent content. Diff review covers this.
4. **Stable-only audit** — final pass greps the new content for `resource(`, `httpResource`, `rxResource`, `signalForms` to confirm no experimental APIs leaked in.

## Out of Scope

- Experimental Angular APIs (Signal Forms, `resource`, `httpResource`, `rxResource`).
- Classic imperative animations API (`trigger`, `state`, `animate`, `keyframes`, `query`, `stagger`).
- Angular Material, CDK overlay/dragdrop/portal/a11y, Service Worker / PWA, Web Components, CSP / Trusted Types, schematic authoring.
- Migration tooling and one-off `ng update` recipes — angular.dev's upgrade guide owns that surface.
- Rewriting any existing reference content. Edits are additive only.

## Further Notes

- The skill's `Core Principles` block already commits to zoneless, signals, native control flow, and functional APIs. New content must not contradict those commitments. In particular, the animations file argues for `animate.enter` / `animate.leave` partly *because* the classic API is zone-coupled — that argument stays explicit in the file.
- After the pilot, expect to revisit depth conventions once. Locking depth on the wrong default and finding out at the seventh file is the failure mode this sequencing prevents.
- The skill's existing exclusions list ("Signal Forms, resource, httpResource, rxResource") in `SKILL.md`'s opening paragraph stays accurate after this work — no change required there.
