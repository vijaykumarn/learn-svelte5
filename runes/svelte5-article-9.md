# Article 9: $bindable - Two-Way Binding

## The Power of Bidirectional Data Flow

You've built a beautiful custom input component with perfect styling and validation logic. You pass a value down as a prop, and when the user types, you emit an event back up. Your parent component listens for that event and updates its state. It works, but it's verbose. For every custom input, you write the same boilerplate: pass value down, listen for changes, update state. There has to be a better way.

In Svelte 4, you could use `bind:value` on native elements, but creating custom components that supported two-way binding required complex workarounds with stores or special patterns. Svelte 5's `$bindable()` rune changes everything. It makes two-way binding a first-class citizen for your custom components, allowing data to flow down as props and back up through the same binding seamlessly.

This isn't just about saving a few lines of code—it's about creating components that feel native, intuitive, and delightful to use. When you can `bind:` to your custom components just like you `bind:` to native inputs, your component API becomes cleaner, your code becomes more maintainable, and your developers (including future you) will thank you.

---

## 🎯 Learning Goals

By the end of this article, you will:

- ✅ Understand the difference between one-way and two-way data flow
- ✅ Create bindable props with `$bindable()` in custom components
- ✅ Use the `bind:` directive on custom components
- ✅ Know when to use `$bindable()` vs callback props
- ✅ Implement fallback values and validation for bindable props
- ✅ Build reusable form controls (Input, Checkbox, Select)
- ✅ Design component APIs that feel natural and intuitive

---

## 📚 Prerequisites

Before diving into this article, you should be comfortable with:

- **Article 5**: `$props()` for component communication
- **Article 2**: `$state()` for reactive values
- **Article 3**: `$derived()` for computed values
- Basic understanding of Svelte's `bind:` directive on native elements

---

## 🧠 Core Concept: Two-Way Binding Explained

### One-Way vs Two-Way Data Flow

In traditional one-way data flow, data moves in a single direction:

```svelte
<!-- Parent.svelte -->
<script>
  let name = $state('');
  
  function handleChange(newName) {
    name = newName;
  }
</script>

<CustomInput 
  value={name} 
  onchange={handleChange} 
/>
```

```svelte
<!-- CustomInput.svelte -->
<script>
  let { value, onchange } = $props();
</script>

<input 
  value={value} 
  oninput={(e) => onchange(e.target.value)} 
/>
```

This works, but it's verbose. Every time you use this component, you need to:
1. Pass the value down as a prop
2. Define a change handler function
3. Wire up the event to update your state

### The `$bindable()` Solution

With `$bindable()`, you create a prop that can be both read from and written to:

```svelte
<!-- CustomInput.svelte -->
<script>
  let { value = $bindable('') } = $props();
</script>

<input bind:value />
```

```svelte
<!-- Parent.svelte -->
<script>
  let name = $state('');
</script>

<CustomInput bind:value={name} />
```

Now the parent can bind to your component just like a native input. Data flows both ways automatically.

---

## 💻 Live Example: Building Bindable Form Controls

Let's build a complete set of form controls with proper two-way binding.### Understanding the Demo

The interactive demo above shows four custom form controls:

1. **TextInput**: A text input with validation and error messages
2. **Checkbox**: A custom checkbox with descriptions
3. **Select**: A styled dropdown selector
4. **Slider**: A range input with value display

All use two-way binding through `$bindable()`. Notice how the parent component simply uses `bind:value` or `bind:checked` without any event handlers—the data flows automatically in both directions.

---

## 🎨 Deep Dive: Creating Bindable Props

### Basic Bindable Syntax

```svelte
<!-- MyComponent.svelte -->
<script>
  let { value = $bindable() } = $props();
</script>

<input bind:value />
```

The `$bindable()` function creates a special prop that:
- Can be read like any other prop
- Can be written to, and changes propagate back to the parent
- Supports default values
- Works with TypeScript for full type safety

### Bindable with Default Values

Always provide sensible defaults for bindable props:

```svelte
<script>
  let { 
    value = $bindable(''),           // String with empty default
    checked = $bindable(false),      // Boolean with false default
    items = $bindable([]),           // Array with empty default
    count = $bindable(0)             // Number with zero default
  } = $props();
</script>
```

### Multiple Bindable Props

A component can have multiple bindable props:

```svelte
<!-- RangeInput.svelte -->
<script>
  let { 
    min = $bindable(0), 
    max = $bindable(100),
    step = $bindable(1)
  } = $props();
</script>

<div class="range-group">
  <input type="number" bind:value={min} />
  <input type="number" bind:value={max} />
  <input type="number" bind:value={step} />
</div>
```

```svelte
<!-- Parent.svelte -->
<script>
  let rangeMin = $state(0);
  let rangeMax = $state(100);
  let rangeStep = $state(5);
</script>

<RangeInput 
  bind:min={rangeMin} 
  bind:max={rangeMax} 
  bind:step={rangeStep} 
/>
```

---

## ⚖️ When to Use $bindable() vs Callbacks

### Use `$bindable()` When:

✅ The component represents a **form control** or input  
✅ There's a clear **one-to-one relationship** between parent and child state  
✅ Changes are **frequent** (typing, dragging, selecting)  
✅ You want the component to feel **native** and intuitive  
✅ The component is **reusable** across many contexts

**Examples**: Custom inputs, sliders, color pickers, date pickers

### Use Callback Props When:

✅ The action is **discrete** (button clicks, modal closes)  
✅ You need to **transform** or **validate** data before accepting it  
✅ Multiple children might trigger the **same parent action**  
✅ The change represents an **event** more than a state change  
✅ You need **async validation** before accepting the change

**Examples**: Submit buttons, delete confirmations, complex workflows

### Hybrid Approach

Sometimes you want both:

```svelte
<!-- SearchInput.svelte -->
<script>
  let { 
    value = $bindable(''),
    onsearch = $props()  // Callback for when user hits Enter
  } = $props();
</script>

<input 
  bind:value 
  onkeydown={(e) => {
    if (e.key === 'Enter') onsearch?.(value);
  }}
/>
```

This gives users flexibility: they can bind to the value for live updates OR listen to the search event for discrete actions.

---

## 🔒 Validation and Constraints

### Internal Validation

You can validate and constrain values inside your bindable component:

```svelte
<!-- ConstrainedInput.svelte -->
<script>
  let { 
    value = $bindable(0),
    min = 0,
    max = 100
  } = $props();
  
  // Constrain the value when it changes
  function handleInput(e) {
    let newValue = Number(e.target.value);
    
    // Clamp to min/max
    if (newValue < min) newValue = min;
    if (newValue > max) newValue = max;
    
    value = newValue;
  }
</script>

<input 
  type="number" 
  value={value} 
  oninput={handleInput}
/>
```

### External Validation

For complex validation, let the parent handle it:

```svelte
<!-- Parent.svelte -->
<script>
  import { z } from 'zod';
  
  let email = $state('');
  
  let emailError = $derived(() => {
    try {
      z.string().email().parse(email);
      return '';
    } catch (err) {
      return 'Invalid email address';
    }
  });
</script>

<TextInput 
  bind:value={email} 
  error={emailError}
/>
```

This keeps your component simple while allowing sophisticated validation in the parent.

---

## ⚠️ Common Mistakes

### Mistake 1: Forgetting `$bindable()`

```svelte
<!-- ❌ WRONG -->
<script>
  let { value } = $props();  // Not bindable!
</script>

<!-- ✅ CORRECT -->
<script>
  let { value = $bindable() } = $props();
</script>
```

Without `$bindable()`, the prop is read-only. Changes won't propagate back.

### Mistake 2: Not Providing Defaults

```svelte
<!-- ❌ RISKY -->
<script>
  let { value = $bindable() } = $props();  // undefined by default
</script>

<!-- ✅ BETTER -->
<script>
  let { value = $bindable('') } = $props();  // Empty string default
</script>
```

Undefined values can cause issues with input elements and comparisons.

### Mistake 3: Over-Using Bindable

```svelte
<!-- ❌ OVERKILL -->
<script>
  let { 
    onclick = $bindable(),  // This should be a callback
    onclose = $bindable(),  // This should be a callback
    loading = $bindable()   // This should be a regular prop
  } = $props();
</script>

<!-- ✅ APPROPRIATE -->
<script>
  let { 
    onclick,      // Callback prop
    onclose,      // Callback prop
    loading       // Regular prop
  } = $props();
</script>
```

Not everything should be bindable. Use `$bindable()` for values that naturally flow both ways.

### Mistake 4: Mutating Bindable Objects

```svelte
<!-- ❌ WRONG -->
<script>
  let { user = $bindable({}) } = $props();
  
  function updateName(name) {
    user.name = name;  // Mutation won't trigger reactivity
  }
</script>

<!-- ✅ CORRECT -->
<script>
  let { user = $bindable({}) } = $props();
  
  function updateName(name) {
    user = { ...user, name };  // Create new object
  }
</script>
```

Remember: Svelte 5 reactivity requires reassignment, not mutation.

---

## ✨ Best Practices

### 1. Name Bindable Props Clearly

```svelte
<!-- Good naming -->
<script>
  let { 
    value = $bindable(''),      // For inputs
    checked = $bindable(false), // For checkboxes
    selected = $bindable(null), // For selects
    items = $bindable([])       // For lists
  } = $props();
</script>
```

Use conventional names that match native elements when possible.

### 2. Document Bindable Props

```svelte
<script>
  /**
   * The current input value
   * @bindable
   */
  let { value = $bindable('') } = $props();
  
  /**
   * Label text to display
   */
  let { label } = $props();
</script>
```

Use JSDoc comments to indicate which props are bindable—this helps users of your component.

### 3. Provide Type Safety

```svelte
<script lang="ts">
  interface Props {
    value?: string;
    label: string;
    error?: string;
  }
  
  let { 
    value = $bindable<string>(''), 
    label,
    error 
  }: Props = $props();
</script>
```

TypeScript makes bindable props safer and more discoverable.

### 4. Keep Components Focused

```svelte
<!-- ✅ GOOD: Focused component -->
<!-- TextInput.svelte -->
<script>
  let { value = $bindable(''), label } = $props();
</script>

<label>
  {label}
  <input bind:value />
</label>

<!-- ❌ TOO MUCH: Kitchen sink component -->
<!-- MegaInput.svelte -->
<script>
  let { 
    value = $bindable(''),
    loading = $bindable(false),
    errors = $bindable([]),
    warnings = $bindable([]),
    suggestions = $bindable([]),
    // ... 20 more bindable props
  } = $props();
</script>
```

A component with too many bindable props is probably doing too much. Consider breaking it down.

### 5. Compose Smaller Components

```svelte
<!-- FormField.svelte - Wrapper component -->
<script>
  let { 
    value = $bindable(''),
    label,
    error,
    helperText
  } = $props();
</script>

<div class="field">
  <label>{label}</label>
  <input bind:value />
  {#if error}
    <span class="error">{error}</span>
  {/if}
  {#if helperText}
    <span class="helper">{helperText}</span>
  {/if}
</div>
```

Build larger components from smaller, focused bindable components.

---

## 🧠 Mental Model: Bindable as a Two-Way Pipe

Think of `$bindable()` as creating a **two-way pipe** between parent and child:

```
Parent Component                Child Component
┌─────────────────┐            ┌─────────────────┐
│                 │            │                 │
│  let value =    │◄──────────►│ value =         │
│    $state('')   │   bind:    │   $bindable()   │
│                 │            │                 │
└─────────────────┘            └─────────────────┘
```

- **Data flows down** when the parent changes
- **Data flows up** when the child changes
- Both sides see the same value always
- No event handlers needed
- It "just works" like native elements

Compare this to callbacks, which are one-way:

```
Parent Component                Child Component
┌─────────────────┐            ┌─────────────────┐
│                 │            │                 │
│  let value =    │◄───────────│ onchange(val)   │
│    $state('')   │  callback  │                 │
│                 │            │                 │
└─────────────────┘            └─────────────────┘
```

Callbacks are great for events, but bindable is perfect for state.

---

## 📋 Quick Reference: $bindable() Cheat Sheet

```svelte
<!-- Basic bindable prop -->
let { value = $bindable('') } = $props();

<!-- Bindable with TypeScript -->
let { value = $bindable<string>('') } = $props();

<!-- Multiple bindable props -->
let { 
  min = $bindable(0),
  max = $bindable(100) 
} = $props();

<!-- Using in parent -->
<MyComponent bind:value={myValue} />

<!-- Bindable object -->
let { user = $bindable({}) } = $props();
user = { ...user, name: 'New Name' };  // Reassign, don't mutate

<!-- Bindable array -->
let { items = $bindable([]) } = $props();
items = [...items, newItem];  // Reassign, don't mutate

<!-- Mixing bindable and regular props -->
let { 
  value = $bindable(''),  // Two-way
  label,                   // One-way
  onclick                  // Callback
} = $props();
```

---

## 🏋️ Practice Exercise: Build a Rating Component

Create a `StarRating.svelte` component with these requirements:

**Features:**
- Bindable `rating` prop (0-5)
- Hoverable stars that preview the rating
- Clicking sets the rating
- Optional `readonly` prop
- Optional `size` prop for different sizes

**Starter Code:**

```svelte
<!-- StarRating.svelte -->
<script>
  let { 
    rating = $bindable(0),
    readonly = false,
    size = 'medium'
  } = $props();
  
  let hoveredRating = $state(0);
  
  // TODO: Implement click handler
  // TODO: Implement hover handlers
  // TODO: Style stars based on rating/hoveredRating
</script>

<div class="star-rating {size}">
  {#each Array(5) as _, i}
    <button 
      class:filled={/* TODO */}
      disabled={readonly}
    >
      ★
    </button>
  {/each}
</div>
```

**Usage:**

```svelte
<script>
  let userRating = $state(0);
</script>

<StarRating bind:rating={userRating} />
<p>You rated: {userRating} stars</p>
```

**Bonus Challenges:**
- Add half-star support (0.5, 1.5, etc.)
- Add keyboard navigation (arrow keys)
- Add an `onchange` callback that fires when rating changes
- Add animations when stars are selected

---

## 🔜 Next Steps

You now understand how to create components with two-way data binding using `$bindable()`. This unlocks a whole new level of component reusability and API elegance.

In **Article 10: "Forms & Validation"**, we'll put everything together. You'll learn how to:
- Build complete form workflows with bindable components
- Implement real-time validation with `$derived()`
- Handle submit logic and async validation
- Integrate with validation libraries like Zod
- Track form state (dirty, touched, valid)

Get ready to build production-ready forms that feel amazing to use!

---

## 💡 Key Takeaways

- `$bindable()` creates props that can be both read and written
- Use `bind:propName` in the parent to enable two-way binding
- Bindable props should always have sensible defaults
- Use `$bindable()` for form controls; callbacks for events
- Bindable values must be reassigned, not mutated
- Multiple bindable props can coexist in the same component
- Keep bindable components focused on a single responsibility

**Remember**: The best component APIs feel natural and intuitive. When users can `bind:` to your components just like native elements, you've created something truly reusable.
