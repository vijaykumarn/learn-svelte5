# Article 10: Forms & Validation - Handling User Input Like a Pro

## Why Forms Are the Make-or-Break Moment

Forms are where theory meets reality in web development. You can have the most elegant state management in the world, but if your registration form loses data, shows confusing errors, or feels sluggish, users will leave. Forms are also where reactivity truly shines—every keystroke, every validation check, every error message is a reactive update waiting to happen.

In Svelte 5, forms become a natural extension of the Runes you've already learned. The `$state()` Rune holds your form data, `$derived()` calculates validation errors in real-time, `bind:value` keeps your UI in sync, and `$effect()` handles side effects like autosaving. No libraries required for basic forms—though we'll also explore how to integrate validation schemas when you need more power.

This article will transform you from writing brittle forms with scattered state into crafting robust, user-friendly forms that validate in real-time, provide clear feedback, and handle edge cases gracefully. By the end, you'll have patterns you can apply to any form, from simple login screens to complex multi-step wizards.

---

## 🎯 Learning Goals

By the end of this article, you'll be able to:

- ✅ Manage form state reactively with `$state()`
- ✅ Implement two-way binding with `bind:value`, `bind:checked`
- ✅ Create real-time validation with `$derived()`
- ✅ Handle form submission and async validation
- ✅ Display error messages contextually
- ✅ Track dirty and touched state for better UX
- ✅ Integrate with schema validation libraries (Zod)
- ✅ Build a production-ready registration form

---

## 📚 Prerequisites

Before diving in, you should be comfortable with:

- **Article 2**: `$state()` for reactive values
- **Article 3**: `$derived()` for computed values
- **Article 4**: `$effect()` for side effects
- **Article 7**: Working with reactive objects
- Basic HTML forms and event handling

---

## 💡 Core Concept: Reactive Form Architecture

### The Anatomy of a Form in Svelte 5

A well-designed form has four key pieces of reactive state:

```
┌─────────────────────────────────────┐
│         FORM VALUES                 │
│  $state({ email: '', password: ''}) │
│  ↓                                   │
│         VALIDATION                  │
│  $derived(errors from values)       │
│  ↓                                   │
│         UI STATE                    │
│  $state({ touched, submitting })    │
│  ↓                                   │
│         SIDE EFFECTS                │
│  $effect(() => autosave...)         │
└─────────────────────────────────────┘
```

**1. Form Values** - The data itself

```svelte
<script>
let formData = $state({
  email: '',
  password: '',
  confirmPassword: ''
});
</script>
```

**2. Validation State** - Derived from values

```svelte
<script>
let errors = $derived.by(() => {
  const errs = {};
  if (formData.email && !formData.email.includes('@')) {
    errs.email = 'Invalid email';
  }
  if (formData.password && formData.password.length < 8) {
    errs.password = 'Password too short';
  }
  if (formData.confirmPassword && 
      formData.password !== formData.confirmPassword) {
    errs.confirmPassword = 'Passwords do not match';
  }
  return errs;
});

let isValid = $derived(
  Object.keys(errors).length === 0 && 
  formData.email && 
  formData.password &&
  formData.confirmPassword
);
</script>
```

**3. UI State** - User interaction tracking

```svelte
<script>
let touched = $state({
  email: false,
  password: false,
  confirmPassword: false
});

let isSubmitting = $state(false);

// Only show errors for touched fields
let visibleErrors = $derived.by(() => {
  const visible = {};
  Object.keys(errors).forEach(field => {
    if (touched[field]) {
      visible[field] = errors[field];
    }
  });
  return visible;
});
</script>
```

**4. Side Effects** - Things that happen as state changes

```svelte
<script>
// Autosave draft to localStorage
$effect(() => {
  if (formData.email || formData.password) {
    localStorage.setItem('registrationDraft', 
      JSON.stringify(formData));
  }
});

// Analytics tracking
$effect(() => {
  if (Object.keys(visibleErrors).length > 0) {
    console.log('User encountered validation errors', visibleErrors);
  }
});
</script>
```

---

## 🔨 Complete Example: Production-Ready Form

Here's a full registration form with all patterns combined:

```svelte
<script>
  // 1. FORM DATA
  let formData = $state({
    email: '',
    password: '',
    confirmPassword: '',
    newsletter: false
  });

  // 2. VALIDATION (always computed)
  let errors = $derived.by(() => {
    const errs = {};
    
    // Email validation
    if (!formData.email) {
      errs.email = 'Email is required';
    } else if (!formData.email.includes('@')) {
      errs.email = 'Please enter a valid email';
    }
    
    // Password validation
    if (!formData.password) {
      errs.password = 'Password is required';
    } else if (formData.password.length < 8) {
      errs.password = 'Password must be at least 8 characters';
    }
    
    // Confirm password
    if (!formData.confirmPassword) {
      errs.confirmPassword = 'Please confirm your password';
    } else if (formData.password !== formData.confirmPassword) {
      errs.confirmPassword = 'Passwords do not match';
    }
    
    return errs;
  });

  // 3. UI STATE
  let touched = $state({
    email: false,
    password: false,
    confirmPassword: false
  });

  let isSubmitting = $state(false);

  // Only show errors for touched fields
  let visibleErrors = $derived.by(() => {
    const visible = {};
    Object.keys(errors).forEach(field => {
      if (touched[field]) {
        visible[field] = errors[field];
      }
    });
    return visible;
  });

  let isValid = $derived(Object.keys(errors).length === 0);

  // 4. SIDE EFFECTS
  // Autosave draft
  $effect(() => {
    const hasContent = formData.email || formData.password;
    if (hasContent) {
      localStorage.setItem('draft', JSON.stringify(formData));
    }
  });

  // 5. EVENT HANDLERS
  function markTouched(field) {
    touched[field] = true;
  }

  async function handleSubmit() {
    // Mark all fields as touched
    touched = {
      email: true,
      password: true,
      confirmPassword: true
    };

    if (!isValid) return;

    isSubmitting = true;

    try {
      // Simulate API call
      await new Promise(r => setTimeout(r, 1500));
      
      console.log('Registration successful!', formData);
      alert('Account created successfully!');
      
      // Clear form and localStorage
      formData = {
        email: '',
        password: '',
        confirmPassword: '',
        newsletter: false
      };
      touched = {
        email: false,
        password: false,
        confirmPassword: false
      };
      localStorage.removeItem('draft');
      
    } catch (error) {
      console.error('Registration failed:', error);
      alert('Registration failed. Please try again.');
    } finally {
      isSubmitting = false;
    }
  }
</script>

<div class="form-container">
  <h1>Create Account</h1>
  
  <!-- Email Field -->
  <div class="field">
    <label for="email">Email Address *</label>
    <input
      id="email"
      type="email"
      bind:value={formData.email}
      on:blur={() => markTouched('email')}
      class:error={visibleErrors.email}
      placeholder="you@example.com"
    />
    {#if visibleErrors.email}
      <span class="error-message">{visibleErrors.email}</span>
    {/if}
  </div>

  <!-- Password Field -->
  <div class="field">
    <label for="password">Password *</label>
    <input
      id="password"
      type="password"
      bind:value={formData.password}
      on:blur={() => markTouched('password')}
      class:error={visibleErrors.password}
      placeholder="••••••••"
    />
    {#if visibleErrors.password}
      <span class="error-message">{visibleErrors.password}</span>
    {:else}
      <span class="hint">At least 8 characters</span>
    {/if}
  </div>

  <!-- Confirm Password Field -->
  <div class="field">
    <label for="confirmPassword">Confirm Password *</label>
    <input
      id="confirmPassword"
      type="password"
      bind:value={formData.confirmPassword}
      on:blur={() => markTouched('confirmPassword')}
      class:error={visibleErrors.confirmPassword}
      placeholder="••••••••"
    />
    {#if visibleErrors.confirmPassword}
      <span class="error-message">{visibleErrors.confirmPassword}</span>
    {:else if formData.confirmPassword && !errors.confirmPassword}
      <span class="success-message">✓ Passwords match</span>
    {/if}
  </div>

  <!-- Newsletter Checkbox -->
  <label class="checkbox-field">
    <input
      type="checkbox"
      bind:checked={formData.newsletter}
    />
    Send me product updates and news
  </label>

  <!-- Submit Button -->
  <button
    on:click={handleSubmit}
    disabled={isSubmitting}
    class="submit-button"
  >
    {isSubmitting ? 'Creating account...' : 'Create Account'}
  </button>
</div>

<style>
  .field {
    margin-bottom: 1rem;
  }

  label {
    display: block;
    margin-bottom: 0.25rem;
    font-weight: 500;
  }

  input[type="email"],
  input[type="password"] {
    width: 100%;
    padding: 0.5rem;
    border: 1px solid #ccc;
    border-radius: 4px;
  }

  input.error {
    border-color: #ef4444;
    background-color: #fef2f2;
  }

  .error-message {
    display: block;
    color: #ef4444;
    font-size: 0.875rem;
    margin-top: 0.25rem;
  }

  .success-message {
    display: block;
    color: #10b981;
    font-size: 0.875rem;
    margin-top: 0.25rem;
  }

  .hint {
    display: block;
    color: #6b7280;
    font-size: 0.875rem;
    margin-top: 0.25rem;
  }

  .submit-button {
    width: 100%;
    padding: 0.75rem;
    background: #3b82f6;
    color: white;
    border: none;
    border-radius: 4px;
    font-weight: 500;
    cursor: pointer;
  }

  .submit-button:disabled {
    background: #9ca3af;
    cursor: not-allowed;
  }
</style>
```

---

## ⚠️ Common Mistakes

### 1. **Showing All Errors Immediately**

❌ **Wrong**: Display errors as soon as the page loads

```svelte
<script>
let errors = $derived(validateForm(formData));
</script>

{#if errors.email}
  <span class="error">{errors.email}</span>
{/if}
```

✅ **Right**: Only show errors after user interaction

```svelte
<script>
let errors = $derived(validateForm(formData));
let touched = $state({ email: false });

let visibleErrors = $derived.by(() => {
  const visible = {};
  Object.keys(errors).forEach(field => {
    if (touched[field]) visible[field] = errors[field];
  });
  return visible;
});
</script>

{#if visibleErrors.email}
  <span class="error">{visibleErrors.email}</span>
{/if}
```

### 2. **Validating in Effects Instead of Derived**

❌ **Wrong**: Computing validation in an effect

```svelte
<script>
let formData = $state({ email: '' });
let errors = $state({});

$effect(() => {
  const errs = {};
  if (!formData.email.includes('@')) {
    errs.email = 'Invalid email';
  }
  errors = errs; // Anti-pattern!
});
</script>
```

✅ **Right**: Use `$derived()` for pure computations

```svelte
<script>
let formData = $state({ email: '' });
let errors = $derived.by(() => {
  const errs = {};
  if (!formData.email.includes('@')) {
    errs.email = 'Invalid email';
  }
  return errs;
});
</script>
```

### 3. **Not Disabling Submit During Submission**

❌ **Wrong**: Allow multiple submissions

```svelte
<button on:click={handleSubmit}>Submit</button>
```

✅ **Right**: Disable button and show loading state

```svelte
<script>
let isSubmitting = $state(false);

async function handleSubmit() {
  isSubmitting = true;
  try {
    await api.register(formData);
  } finally {
    isSubmitting = false;
  }
}
</script>

<button 
  on:click={handleSubmit} 
  disabled={isSubmitting}
>
  {isSubmitting ? 'Submitting...' : 'Submit'}
</button>
```

### 4. **Forgetting to Clear Form After Success**

❌ **Wrong**: Leave data in form after submission

```svelte
<script>
async function handleSubmit() {
  await api.register(formData);
  alert('Success!');
  // Forgot to clear the form!
}
</script>
```

✅ **Right**: Reset form and touched state

```svelte
<script>
async function handleSubmit() {
  await api.register(formData);
  alert('Success!');
  
  // Reset everything
  formData = { email: '', password: '', confirmPassword: '' };
  touched = { email: false, password: false, confirmPassword: false };
  localStorage.removeItem('draft');
}
</script>
```

### 5. **Not Handling Async Validation Errors**

❌ **Wrong**: Ignore server-side validation

```svelte
<script>
async function handleSubmit() {
  await api.register(formData); // Might throw!
}
</script>
```

✅ **Right**: Handle and display server errors

```svelte
<script>
let serverErrors = $state({});

async function handleSubmit() {
  try {
    await api.register(formData);
  } catch (error) {
    if (error.validationErrors) {
      serverErrors = error.validationErrors;
    } else {
      alert('Registration failed. Please try again.');
    }
  }
}

let allErrors = $derived.by(() => {
  return { ...errors, ...serverErrors };
});
</script>
```

---

## ✅ Best Practices

### 1. **Separate Validation Logic**

Extract validation into reusable functions:

```svelte
<script>
// validation.svelte.js
export function validateEmail(email) {
  if (!email) return 'Email is required';
  if (!email.includes('@')) return 'Invalid email format';
  if (email.length < 3) return 'Email too short';
  return null;
}

export function validatePassword(password) {
  if (!password) return 'Password is required';
  if (password.length < 8) return 'At least 8 characters';
  if (!/[A-Z]/.test(password)) return 'Need uppercase letter';
  if (!/[0-9]/.test(password)) return 'Need a number';
  return null;
}

// In your component
import { validateEmail, validatePassword } from './validation.svelte';

let errors = $derived.by(() => {
  return {
    email: validateEmail(formData.email),
    password: validatePassword(formData.password)
  };
});
</script>
```

### 2. **Use Zod for Complex Validation**

For production apps, schema validation is cleaner:

```svelte
<script>
import { z } from 'zod';

const registrationSchema = z.object({
  email: z.string().email('Invalid email address'),
  password: z.string()
    .min(8, 'Password must be at least 8 characters')
    .regex(/[A-Z]/, 'Need at least one uppercase letter')
    .regex(/[0-9]/, 'Need at least one number'),
  confirmPassword: z.string()
}).refine(data => data.password === data.confirmPassword, {
  message: 'Passwords do not match',
  path: ['confirmPassword']
});

let errors = $derived.by(() => {
  const result = registrationSchema.safeParse(formData);
  if (result.success) return {};
  
  const errs = {};
  result.error.errors.forEach(err => {
    errs[err.path[0]] = err.message;
  });
  return errs;
});
</script>
```

### 3. **Debounce Expensive Validations**

For API calls or heavy computations:

```svelte
<script>
import { debounce } from './utils';

let usernameAvailable = $state(null);
let checkingUsername = $state(false);

const checkUsernameAvailability = debounce(async (username) => {
  if (username.length < 3) return;
  
  checkingUsername = true;
  try {
    const result = await api.checkUsername(username);
    usernameAvailable = result.available;
  } finally {
    checkingUsername = false;
  }
}, 500);

$effect(() => {
  if (formData.username) {
    checkUsernameAvailability(formData.username);
  }
});
</script>

<input bind:value={formData.username} />
{#if checkingUsername}
  <span>Checking availability...</span>
{:else if usernameAvailable === false}
  <span class="error">Username taken</span>
{:else if usernameAvailable === true}
  <span class="success">Username available</span>
{/if}
```

### 4. **Track Form Dirty State**

Know if the form has unsaved changes:

```svelte
<script>
let initialData = { email: '', password: '' };
let formData = $state({ ...initialData });

let isDirty = $derived(
  JSON.stringify(formData) !== JSON.stringify(initialData)
);

// Warn before leaving
$effect(() => {
  if (isDirty) {
    const handler = (e) => {
      e.preventDefault();
      e.returnValue = '';
    };
    window.addEventListener('beforeunload', handler);
    return () => window.removeEventListener('beforeunload', handler);
  }
});
</script>

{#if isDirty}
  <div class="warning">You have unsaved changes</div>
{/if}
```

### 5. **Build Reusable Field Components**

Create composable form fields:

```svelte
<!-- TextField.svelte -->
<script>
  let { 
    label,
    value = $bindable(''),
    error = null,
    hint = null,
    ...rest 
  } = $props();
</script>

<div class="field">
  <label>{label}</label>
  <input 
    bind:value 
    class:error={error}
    {...rest}
  />
  {#if error}
    <span class="error-message">{error}</span>
  {:else if hint}
    <span class="hint">{hint}</span>
  {/if}
</div>

<!-- Usage -->
<TextField 
  label="Email"
  bind:value={formData.email}
  error={visibleErrors.email}
  type="email"
  placeholder="you@example.com"
/>
```

---

## 🧠 Mental Model: Forms as State Machines

Think of your form as having different states:

```
EMPTY → FILLING → VALIDATING → VALID/INVALID → SUBMITTING → SUCCESS/ERROR
```

Each state determines what the user sees:

- **EMPTY**: Show hints, no errors
- **FILLING**: Real-time feedback as user types
- **VALIDATING**: Show loading spinners for async checks
- **VALID**: Enable submit button, show success indicators
- **INVALID**: Show errors, disable submit
- **SUBMITTING**: Disable all inputs, show progress
- **SUCCESS**: Clear form, show confirmation
- **ERROR**: Show error message, allow retry

Your reactive state naturally represents these transitions:

```svelte
<script>
let formData = $state({ email: '' });
let touched = $state({ email: false });
let isSubmitting = $state(false);
let submitError = $state(null);

let errors = $derived(validate(formData));
let isValid = $derived(Object.keys(errors).length === 0);

// The form's current state
let formState = $derived.by(() => {
  if (submitError) return 'ERROR';
  if (isSubmitting) return 'SUBMITTING';
  if (touched.email && !isValid) return 'INVALID';
  if (touched.email && isValid) return 'VALID';
  if (formData.email) return 'FILLING';
  return 'EMPTY';
});
</script>

{#if formState === 'ERROR'}
  <div class="error-banner">{submitError}</div>
{:else if formState === 'SUBMITTING'}
  <div class="loading">Creating account...</div>
{/if}
```

---

## 📋 Quick Reference

### Form State Pattern

```svelte
<script>
// 1. Form data
let formData = $state({ /* fields */ });

// 2. Validation
let errors = $derived.by(() => {
  // return errors object
});

// 3. UI state
let touched = $state({ /* fields */ });
let isSubmitting = $state(false);

// 4. Visible errors (only touched)
let visibleErrors = $derived.by(() => {
  // filter by touched
});

// 5. Form validity
let isValid = $derived(/* check errors */);

// 6. Handlers
function markTouched(field) { }
async function handleSubmit() { }
</script>
```

### Two-Way Binding

```svelte
<!-- Text input -->
<input bind:value={formData.email} />

<!-- Checkbox -->
<input type="checkbox" bind:checked={formData.newsletter} />

<!-- Radio buttons -->
<input type="radio" bind:group={formData.plan} value="free" />
<input type="radio" bind:group={formData.plan} value="pro" />

<!-- Select -->
<select bind:value={formData.country}>
  <option value="us">United States</option>
  <option value="uk">United Kingdom</option>
</select>

<!-- Textarea -->
<textarea bind:value={formData.message} />
```

### Validation Patterns

```svelte
<script>
// Simple inline validation
let errors = $derived.by(() => ({
  email: !formData.email.includes('@') ? 'Invalid' : null
}));

// Multi-rule validation
function validatePassword(pwd) {
  if (!pwd) return 'Required';
  if (pwd.length < 8) return 'Too short';
  if (!/[A-Z]/.test(pwd)) return 'Need uppercase';
  return null;
}

// Async validation
let usernameError = $state(null);
$effect(() => {
  if (formData.username) {
    api.checkUsername(formData.username)
      .then(result => {
        usernameError = result.available ? null : 'Taken';
      });
  }
});
</script>
```

---

## 🏋️ Practice Exercise

**Build a Multi-Step Registration Form**

Create a 3-step registration wizard:

**Step 1: Account Info**
- Email (validate format)
- Username (check availability via fake API)
- Password (with strength meter)

**Step 2: Personal Info**
- First Name (required)
- Last Name (required)
- Phone (optional, validate format)

**Step 3: Preferences**
- Account type (free/pro/enterprise)
- Newsletter subscription
- Terms acceptance (required)

**Requirements:**
- Navigate between steps with Next/Back buttons
- Show progress indicator
- Validate each step before allowing next
- Remember data when going back
- Submit all data at final step
- Show success confirmation

**Bonus Challenges:**
- Save draft to localStorage
- Add password strength indicator with color coding
- Implement "Skip" option for optional fields
- Add keyboard navigation (Enter to next, Escape to cancel)
- Create smooth animations between steps

---

## 🎯 Next Steps

Congratulations! You now know how to build production-ready forms with Svelte 5 Runes. You've learned:

- Form state management with `$state()`
- Real-time validation with `$derived()`
- Touched state for better UX
- Async validation patterns
- Error handling and display
- Integration with schema validation

**In Article 11**, we'll dive deep into **`$effect.pre` & `$effect.root`** - advanced effect timing and control. You'll learn how to create effects that run before DOM updates, manually control effect lifecycles, and build complex animations with precise timing.

**Coming up**: Understanding when effects run, breaking out of reactive contexts, and gaining fine-grained control over reactivity.

---

## 📚 Further Reading

- [Svelte 5 Forms Documentation](https://svelte.dev/docs/forms)
- [Zod Validation Library](https://zod.dev/)
- [MDN: Client-side Form Validation](https://developer.mozilla.org/en-US/docs/Learn/Forms/Form_validation)
- [Web Accessibility: Forms](https://www.w3.org/WAI/tutorials/forms/)

---

**Ready to master advanced effects?** Continue to Article 11: `$effect.pre` & `$effect.root` →
