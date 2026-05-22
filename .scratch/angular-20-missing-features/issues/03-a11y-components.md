# components.md a11y extension + SKILL.md hookup

Status: done
Type: AFK

## Parent

`.scratch/angular-20-missing-features/PRD.md`

## What to build

Extend `references/components.md` with a basic accessibility section using plain Angular only — no CDK a11y module.

Topics:

- Semantic HTML defaults (use `<button>`, `<nav>`, `<main>`, etc. before reaching for ARIA)
- ARIA attributes wired through the `host` object of `@Component` / `@Directive`
- Keyboard event handlers via the `host` listeners object (no `@HostListener`)
- Focus management with `inject(ElementRef)` and native `.focus()`
- `aria-live` regions for dynamic content announcements
- Anti-patterns: `div` with `(click)` and no role, missing `aria-label` on icon-only buttons

`SKILL.md` hookup:

- Add an a11y bullet to the existing components principle, or a one-line note pointing to the new section in `components.md`
- No new top-level section required — fits inside the existing components topic

Source verification: query angular.dev via context7 for the `host` object syntax and any a11y-specific guidance.

## Acceptance criteria

- [x] `references/components.md` gains an a11y section with code examples
- [x] No CDK a11y APIs introduced
- [x] `host` object usage matches the skill's existing anti-`@HostBinding` / `@HostListener` stance
- [x] All API patterns verified via context7
- [x] `SKILL.md` references the new section
- [x] No content changes to unrelated sections in `components.md` or `SKILL.md`

## Blocked by

- Issue #01 (animations pilot) — for style precedent
