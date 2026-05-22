# Pilot: animations.md + SKILL.md hookup

Status: ready-for-agent
Type: HITL (style/depth review gate at end)

## Parent

`.scratch/angular-20-missing-features/PRD.md`

## What to build

Create `references/animations.md` covering Angular 20's stable animations surface, then wire it into `SKILL.md`. This is the pilot for the larger angular-20-missing-features PRD — it establishes the style, depth, and density precedent that issues #02–#07 will match.

Topics for `references/animations.md`:

- `animate.enter` and `animate.leave` template syntax (the v20-native, CSS-driven path)
- View Transitions API integration via the router (`withViewTransitions()` feature)
- `provideAnimationsAsync` vs `provideAnimations` — bundle impact and when to pick each
- Zoneless-compatibility notes and why classic `trigger()` is being skipped
- Anti-patterns inline (legacy imperative API, `@HostBinding('@fade')`, etc.)

`SKILL.md` hookup:

- New short prose section between existing sections, ending with the `> Read [references/animations.md]` pointer (match existing pattern)
- New `Gotchas` entry: classic `@angular/animations` is zone-dependent; prefer `animate.enter` / `animate.leave`
- New `Quick Reference: What NOT to Do` row: classic `trigger()` animations → `animate.enter` class
- New line in the `References` list at the bottom

Source verification: query angular.dev via context7 for every API name, provider, and router feature before writing. The skill's preamble claims angular.dev authority — preserve that.

## Acceptance criteria

- [ ] `references/animations.md` created, ~400–550 lines, density matches existing references
- [ ] Only stable v20 animations APIs covered; classic imperative API not introduced
- [ ] All API names verified against angular.dev via context7
- [ ] `SKILL.md` updated: new prose section, new Gotcha, new Quick Reference row, new References list line
- [ ] No content changes to unrelated `SKILL.md` sections
- [ ] No experimental APIs leaked in (grep for `resource(`, `httpResource`, `rxResource`, `signalForms` returns nothing in new content)
- [ ] User reviews and approves style + depth before issues #02–#07 begin

## Blocked by

None — can start immediately.
