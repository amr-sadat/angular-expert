# Forms

## Table of Contents
- [Reactive Forms](#reactive-forms)
- [Nested Form Groups](#nested-form-groups)
- [Built-in Validators](#built-in-validators)
- [Custom Validators](#custom-validators)
- [Async Validators](#async-validators)
- [Dynamic Forms with FormArray](#dynamic-forms-with-formarray)
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