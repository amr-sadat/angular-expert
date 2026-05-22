# signals-and-state.md afterRenderEffect extension + SKILL.md gotcha

Status: ready-for-agent
Type: AFK

## Parent

`.scratch/angular-20-missing-features/PRD.md`

## What to build

Extend `references/signals-and-state.md` with an `afterRenderEffect` section, and add a corresponding `Gotchas` entry to `SKILL.md`.

Topics:

- What `afterRenderEffect` is and how it differs from `effect`
- The render phases (`earlyRead`, `write`, `mixedReadWrite`, `read`) and which one to pick for each side-effect category
- Decision rules: when to reach for `afterRenderEffect` vs `effect` vs `afterNextRender` / `afterEveryRender`
- Worked example: measuring DOM dimensions after layout
- Anti-pattern: reading DOM in `effect()`

`SKILL.md` hookup:

- Add `afterRenderEffect` mention to the existing `Signals & State Management` section if natural, or rely on the reference file deep link
- New `Gotchas` entry: `effect()` runs synchronously on signal change before render; use `afterRenderEffect` for DOM-reading side effects

Source verification: query angular.dev via context7 for the exact phase names and signatures — these are recent additions and easy to miscall.

## Acceptance criteria

- [ ] `references/signals-and-state.md` gains an `afterRenderEffect` section with examples for at least two phases
- [ ] Phase names exactly match angular.dev
- [ ] `SKILL.md` Gotchas section gains the new entry
- [ ] No content changes to unrelated sections
- [ ] No experimental APIs introduced

## Blocked by

- Issue #01 (animations pilot) — for style precedent
