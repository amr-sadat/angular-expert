# forms.md custom ControlValueAccessor extension

Status: ready-for-agent
Type: AFK

## Parent

`.scratch/angular-20-missing-features/PRD.md`

## What to build

Extend `references/forms.md` with a custom `ControlValueAccessor` (CVA) section showing how to wrap a third-party or custom input so it plugs cleanly into Reactive Forms.

Topics:

- `ControlValueAccessor` interface (`writeValue`, `registerOnChange`, `registerOnTouched`, `setDisabledState`)
- `NG_VALUE_ACCESSOR` provider registration via `forwardRef`
- Signal-based internal state with `toObservable` / explicit emit on change
- Disabled state handling
- Integration with Reactive Forms `FormControl` (worked example: a wrapped color picker or toggle)
- Zoneless-compatibility: emit changes synchronously; do not rely on zone-driven change detection
- Anti-patterns: reading the value out of the DOM, using `@Input` for the value (use the CVA contract instead)

`SKILL.md` hookup:

- The existing Forms section already deep-links to `references/forms.md`. No new section required.
- Optionally add a Gotchas entry if zoneless CVAs surface a real footgun (e.g. "CVA `registerOnChange` callback must be invoked synchronously on value change in zoneless apps").

Source verification: query angular.dev via context7 for the current CVA interface and provider registration pattern.

## Acceptance criteria

- [ ] `references/forms.md` gains a CVA section with a full worked example
- [ ] Interface signatures exactly match angular.dev
- [ ] All claims verified via context7
- [ ] Any new Gotchas added to `SKILL.md`
- [ ] No content changes to unrelated `forms.md` sections

## Blocked by

- Issue #01 (animations pilot) — for style precedent
