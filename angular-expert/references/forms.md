# Forms

## Table of Contents
- [Reactive Forms](#reactive-forms)
- [Nested Form Groups](#nested-form-groups)
- [Built-in Validators](#built-in-validators)
- [Custom Validators](#custom-validators)
- [Async Validators](#async-validators)
- [Dynamic Forms with FormArray](#dynamic-forms-with-formarray)
- [Custom ControlValueAccessor](#custom-controlvalueaccessor)
- [Reactive Forms in Zoneless Apps](#reactive-forms-in-zoneless-apps)
- [Form Patterns](#form-patterns)

---

## Reactive Forms

Reactive Forms are Angular 20's stable, production-ready forms API. Use `FormBuilder` with `nonNullable` for strict typing.

```typescript
import { Component, ChangeDetectionStrategy, inject } from '@angular/core';
import { FormBuilder, ReactiveFormsModule, Validators } from '@angular/forms';

@Component({
  selector: 'app-profile',
  changeDetection: ChangeDetectionStrategy.OnPush,
  imports: [ReactiveFormsModule],
  template: `
    <form [formGroup]="profileForm" (ngSubmit)="onSubmit()">
      <label>
        Name
        <input formControlName="name" />
      </label>
      @if (profileForm.controls.name.touched && profileForm.controls.name.invalid) {
        <span class="error">Name is required (min 2 chars)</span>
      }

      <label>
        Email
        <input type="email" formControlName="email" />
      </label>

      <button type="submit" [disabled]="profileForm.invalid">Save</button>
    </form>
  `,
})
export class ProfileComponent {
  private fb = inject(FormBuilder);

  profileForm = this.fb.nonNullable.group({
    name: ['', [Validators.required, Validators.minLength(2)]],
    email: ['', [Validators.required, Validators.email]],
  });

  onSubmit(): void {
    if (this.profileForm.valid) {
      const value = this.profileForm.getRawValue();
      // value is fully typed: { name: string; email: string }
    }
  }
}
```

Use `fb.nonNullable.group()` for strict typing — all controls become `FormControl<string>` instead of `FormControl<string | null>`.

## Nested Form Groups

Group related fields with nested `FormGroup`:

```typescript
profileForm = this.fb.nonNullable.group({
  name: ['', [Validators.required, Validators.minLength(2)]],
  email: ['', [Validators.required, Validators.email]],
  address: this.fb.nonNullable.group({
    street: [''],
    city: ['', Validators.required],
    state: [''],
    zip: ['', [Validators.required, Validators.pattern(/^\d{5}$/)]],
  }),
});
```

Template:

```html
<form [formGroup]="profileForm" (ngSubmit)="onSubmit()">
  <input formControlName="name" />
  <input formControlName="email" />

  <div formGroupName="address">
    <input formControlName="street" />
    <input formControlName="city" />
    <input formControlName="state" />
    <input formControlName="zip" />
  </div>

  <button type="submit" [disabled]="profileForm.invalid">Save</button>
</form>
```

## Built-in Validators

| Validator | Usage |
|---|---|
| `Validators.required` | Field must not be empty |
| `Validators.minLength(n)` | Minimum character length |
| `Validators.maxLength(n)` | Maximum character length |
| `Validators.email` | Valid email format |
| `Validators.pattern(regex)` | Must match regex pattern |
| `Validators.min(n)` | Minimum numeric value |
| `Validators.max(n)` | Maximum numeric value |

## Custom Validators

### Synchronous validator

```typescript
import { AbstractControl, ValidationErrors, ValidatorFn } from '@angular/forms';

export function forbiddenNameValidator(forbiddenName: RegExp): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const forbidden = forbiddenName.test(control.value);
    return forbidden ? { forbiddenName: { value: control.value } } : null;
  };
}

// Usage
name: ['', [Validators.required, forbiddenNameValidator(/admin/i)]],
```

### Cross-field validator

```typescript
export const passwordMatchValidator: ValidatorFn = (
  group: AbstractControl
): ValidationErrors | null => {
  const password = group.get('password')?.value;
  const confirm = group.get('confirmPassword')?.value;
  return password === confirm ? null : { passwordMismatch: true };
};

// Usage on the group
passwordGroup = this.fb.nonNullable.group(
  {
    password: ['', [Validators.required, Validators.minLength(8)]],
    confirmPassword: ['', Validators.required],
  },
  { validators: passwordMatchValidator }
);
```

### Displaying errors

```html
@if (profileForm.controls.name.touched && profileForm.controls.name.errors; as errors) {
  @if (errors['required']) {
    <span class="error">Name is required</span>
  }
  @if (errors['minlength']) {
    <span class="error">Name must be at least {{ errors['minlength'].requiredLength }} characters</span>
  }
  @if (errors['forbiddenName']) {
    <span class="error">This name is not allowed</span>
  }
}
```

## Async Validators

```typescript
import { AsyncValidatorFn, AbstractControl, ValidationErrors } from '@angular/forms';
import { Observable, map, catchError, of, debounceTime, switchMap, first } from 'rxjs';

export function uniqueEmailValidator(userService: UserService): AsyncValidatorFn {
  return (control: AbstractControl): Observable<ValidationErrors | null> => {
    return control.valueChanges.pipe(
      debounceTime(300),
      switchMap(email => userService.checkEmailAvailable(email)),
      map(available => (available ? null : { emailTaken: true })),
      catchError(() => of(null)),
      first(),
    );
  };
}

// Usage
email: ['', {
  validators: [Validators.required, Validators.email],
  asyncValidators: [uniqueEmailValidator(this.userService)],
  updateOn: 'blur',
}],
```

```html
@if (profileForm.controls.email.pending) {
  <span>Checking availability...</span>
}
@if (profileForm.controls.email.errors?.['emailTaken']) {
  <span class="error">This email is already registered</span>
}
```

## Dynamic Forms with FormArray

```typescript
import { FormArray, FormBuilder, ReactiveFormsModule, Validators } from '@angular/forms';

@Component({
  selector: 'app-skills-form',
  changeDetection: ChangeDetectionStrategy.OnPush,
  imports: [ReactiveFormsModule],
  template: `
    <form [formGroup]="skillsForm">
      <div formArrayName="skills">
        @for (skill of skillsArray.controls; track $index) {
          <div>
            <input [formControlName]="$index" />
            <button type="button" (click)="removeSkill($index)">Remove</button>
          </div>
        }
      </div>
      <button type="button" (click)="addSkill()">Add Skill</button>
    </form>
  `,
})
export class SkillsFormComponent {
  private fb = inject(FormBuilder);

  skillsForm = this.fb.group({
    skills: this.fb.array([this.fb.nonNullable.control('', Validators.required)]),
  });

  get skillsArray(): FormArray {
    return this.skillsForm.get('skills') as FormArray;
  }

  addSkill(): void {
    this.skillsArray.push(this.fb.nonNullable.control('', Validators.required));
  }

  removeSkill(index: number): void {
    this.skillsArray.removeAt(index);
  }
}
```

## Custom ControlValueAccessor

A `ControlValueAccessor` (CVA) is the bridge between a custom input component and Angular's forms API. Implement it whenever you wrap a third-party widget (color picker, rich-text editor, date picker, toggle) so it plugs into `FormControl` / `formControlName` like a native `<input>`.

### The interface

```typescript
interface ControlValueAccessor {
  writeValue(obj: any): void;
  registerOnChange(fn: any): void;
  registerOnTouched(fn: any): void;
  setDisabledState?(isDisabled: boolean): void;
}
```

| Method | Called by Angular when… | What you do |
|---|---|---|
| `writeValue(value)` | Parent form pushes a new value into the control (`setValue`, `patchValue`, initial value) | Update internal state to reflect `value` |
| `registerOnChange(fn)` | Once, on init | Store `fn`; call it whenever the user changes the value |
| `registerOnTouched(fn)` | Once, on init | Store `fn`; call it when the control is "touched" (blur) |
| `setDisabledState(isDisabled)` | Parent calls `control.disable()` / `enable()` | Reflect disabled state in the UI |

### Worked example: signal-based color picker

```typescript
import {
  Component,
  ChangeDetectionStrategy,
  forwardRef,
  signal,
} from '@angular/core';
import {
  ControlValueAccessor,
  FormBuilder,
  NG_VALUE_ACCESSOR,
  ReactiveFormsModule,
} from '@angular/forms';

@Component({
  selector: 'app-color-picker',
  changeDetection: ChangeDetectionStrategy.OnPush,
  providers: [
    {
      provide: NG_VALUE_ACCESSOR,
      useExisting: forwardRef(() => ColorPickerComponent),
      multi: true,
    },
  ],
  template: `
    <input
      type="color"
      [value]="value()"
      [disabled]="disabled()"
      (input)="onInput($event)"
      (blur)="onBlur()"
    />
  `,
})
export class ColorPickerComponent implements ControlValueAccessor {
  protected readonly value = signal('#000000');
  protected readonly disabled = signal(false);

  private onChange: (value: string) => void = () => {};
  private onTouched: () => void = () => {};

  writeValue(value: string | null): void {
    this.value.set(value ?? '#000000');
  }

  registerOnChange(fn: (value: string) => void): void {
    this.onChange = fn;
  }

  registerOnTouched(fn: () => void): void {
    this.onTouched = fn;
  }

  setDisabledState(isDisabled: boolean): void {
    this.disabled.set(isDisabled);
  }

  protected onInput(event: Event): void {
    const next = (event.target as HTMLInputElement).value;
    this.value.set(next);
    // Call onChange synchronously inside the DOM event handler.
    // Deferring it (setTimeout, debounced subject without explicit emit) leaves
    // the parent FormControl out of sync until the next CD trigger in zoneless apps.
    this.onChange(next);
  }

  protected onBlur(): void {
    this.onTouched();
  }
}
```

### Using it in a Reactive Form

The custom component is now indistinguishable from a native input in the parent form:

```typescript
@Component({
  selector: 'app-theme-form',
  imports: [ReactiveFormsModule, ColorPickerComponent],
  template: `
    <form [formGroup]="themeForm">
      <label>
        Primary color
        <app-color-picker formControlName="primary" />
      </label>
    </form>
  `,
})
export class ThemeFormComponent {
  private fb = inject(FormBuilder);

  themeForm = this.fb.nonNullable.group({
    primary: ['#3366ff'],
  });
}
```

`themeForm.controls.primary.setValue('#ff0000')` calls `writeValue`. User input calls `onChange`, which updates the `FormControl`. `themeForm.controls.primary.disable()` calls `setDisabledState(true)`.

### Provider registration notes

- `NG_VALUE_ACCESSOR` is a **multi-provider** — always set `multi: true`.
- `useExisting: forwardRef(() => MyComponent)` is required because the provider must reference the class that is still being declared.
- Register on the **component itself**, not the consumer.

### Zoneless compatibility

- Call the `onChange` callback inside the user-input handler. In zoneless apps, deferring the call (e.g. via `setTimeout`, `queueMicrotask`, or a debounced observable that has not yet emitted) leaves the parent `FormControl` out of sync until the next change-detection trigger. Calling `onChange` directly in the event handler keeps the form value, validators, and dependent computed signals in step with user input.
- The component itself uses `OnPush` and signal-driven state — no `markForCheck` needed.
- `writeValue` is called by the forms engine; updating a signal inside it is safe and synchronous.

### Anti-patterns

| Don't | Do |
|---|---|
| Expose the value via `@Input()` / `input()` and read it back via `@Output()` | Implement `ControlValueAccessor` so the value flows through Angular's forms API |
| Read the current value from `event.target.value` outside the change handler | Track it in a signal (single source of truth) |
| Forget `multi: true` on the provider — silently overwrites other accessors | Always include `multi: true` |
| Omit `setDisabledState` and instead expose a `disabled` input | Implement `setDisabledState` so `control.disable()` works |
| Call `onChange` from an `effect()` that watches the value signal | Call `onChange` directly in the DOM event handler — keeps the data flow explicit and avoids feedback loops with `writeValue` |

## Reactive Forms in Zoneless Apps

In zoneless applications, Reactive Forms' `setValue`, `patchValue`, and similar APIs do NOT automatically trigger change detection. You must either:

1. **Connect to signals** — convert form observables to signals with `toSignal()`
2. **Call markForCheck()** — manually notify Angular

```typescript
import { ChangeDetectorRef, inject } from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

@Component({ /* ... */ })
export class ZonelessFormComponent {
  private cdr = inject(ChangeDetectorRef);

  constructor() {
    this.profileForm.valueChanges
      .pipe(takeUntilDestroyed())
      .subscribe(() => this.cdr.markForCheck());
  }
}
```

Or the signal-based approach (preferred):

```typescript
import { toSignal } from '@angular/core/rxjs-interop';

formValue = toSignal(this.profileForm.valueChanges, {
  initialValue: this.profileForm.value,
});
```

## Form Patterns

### Loading form data from an API

```typescript
@Component({ /* ... */ })
export class EditUserComponent {
  private fb = inject(FormBuilder);
  private userService = inject(UserService);

  userId = input.required<string>();

  userForm = this.fb.nonNullable.group({
    name: ['', Validators.required],
    email: ['', [Validators.required, Validators.email]],
  });

  private user = toSignal(
    toObservable(this.userId).pipe(
      switchMap(id => this.userService.getUser(id)),
    ),
  );

  constructor() {
    effect(() => {
      const user = this.user();
      if (user) {
        this.userForm.patchValue(user);
      }
    });
  }
}
```

### Form submission with loading state

```typescript
isSubmitting = signal(false);

async onSubmit(): Promise<void> {
  if (this.profileForm.invalid) return;

  this.isSubmitting.set(true);
  try {
    await firstValueFrom(
      this.apiService.updateProfile(this.profileForm.getRawValue())
    );
  } finally {
    this.isSubmitting.set(false);
  }
}
```

```html
<button type="submit" [disabled]="isSubmitting() || profileForm.invalid">
  @if (isSubmitting()) {
    Saving...
  } @else {
    Submit
  }
</button>
```