# ssr-and-performance.md @defer hydrate triggers extension + SKILL.md Quick Reference entry

Status: ready-for-agent
Type: AFK

## Parent

`.scratch/angular-20-missing-features/PRD.md`

## What to build

Extend `references/ssr-and-performance.md` with a full reference for `@defer (hydrate on ...)` triggers, and add a corresponding row to `SKILL.md`'s `Quick Reference: What NOT to Do` table.

Topics:

- Trigger table with semantics and recommended use cases:
  - `hydrate on idle`
  - `hydrate on viewport`
  - `hydrate on interaction`
  - `hydrate on hover`
  - `hydrate on immediate`
  - `hydrate never`
- How hydrate triggers interact with placeholder/loading/error blocks under incremental hydration
- Worked example: a below-the-fold widget with `hydrate on viewport`
- Anti-pattern: manual `IntersectionObserver` for hydration timing

`SKILL.md` hookup:

- New `Quick Reference: What NOT to Do` row: manual `IntersectionObserver` for hydration → `@defer (hydrate on viewport)`

Source verification: query angular.dev via context7 for the exact trigger syntax and supported predicates — `hydrate never` and `hydrate on immediate` are new enough to easily miscall.

## Acceptance criteria

- [ ] `references/ssr-and-performance.md` gains the trigger table and a worked example
- [ ] Trigger names exactly match angular.dev
- [ ] All claims verified via context7
- [ ] `SKILL.md` Quick Reference table gains the new row
- [ ] No content changes to unrelated sections

## Blocked by

- Issue #01 (animations pilot) — for style precedent
