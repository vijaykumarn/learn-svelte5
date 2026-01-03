# Article 5: $props - Component Communication

## Hook

You've built a beautiful button component. It has perfect styling, smooth hover effects, and feels great to click. But then you try to use it in different places across your app—and you realize it only does one thing. You need a primary button here, a danger button there, a disabled state, different sizes, loading states... Suddenly, you're copy-pasting components and making slight variations.

This is where props become your superpower. Props are how components receive data from their parents, transforming rigid, single-purpose components into flexible, reusable building blocks. In Svelte 5, the `$props()` rune makes this clearer and more powerful than ever before.

But here's the thing: good component APIs are hard to design. How do you know which things should be props? How do you handle defaults? What about TypeScript types? By the end of this article, you'll understand not just *how* to use `$props()`, but how to design component interfaces that make your code a joy to work with.

---

## Learning Goals

By the end of this article, you will:

- ✅ Declare component props with `$props()` and understand its syntax
- ✅ Use destructuring with default values for optional props
- ✅ Differentiate between optional and required props
- ✅ Type props correctly with TypeScript
- ✅ Implement prop validation patterns
- ✅ Use rest props (`...rest`) to forward attributes
- ✅ Design intuitive component APIs
- ✅ Build a production-ready reusable Button component

---

## Prerequisites

Before diving in, you should understand:

- **Article 1**: Why Svelte 5 uses explicit runes
- **Article 2**: `$state()` for reactive values
- **Article 3**: `$derived()` for computed values
- **Article 4**: `$effect()` for side effects
- Basic component composition in Svelte

---

## Core Concept: Props as Component Inputs

### What Are Props?

Props (short for "properties") are how parent components pass data to child components. They're the **inputs** to your component's function. Think of a component as a black box:

```
┌─────────────────────────────┐
│  Parent Component           │
│                             │
│  <Button                    │
│    text="Click me"          │  ← Props flow down
│    variant="primary"        │
│    onclick={handleClick}    │
│  />                         │
└─────────────────────────────┘
         │
         ▼
┌─────────────────────────────┐
│  Button Component           │
│  (receives props)           │
│                             │
│  Renders based on props     │
└─────────────────────────────┘
```

Props make components **configurable** and **reusable**. Without props, every button would be identical. With props, one Button component can serve your entire application.

### The `$props()` Rune

In Svelte 5, you declare props using the `$props()` rune:

```svelte
<script>
  let props = $props();
</script>

<button>{props.text}</button>
```

But the real power comes from **destructuring**:

```svelte
<script>
  let { text, variant = 'primary' } = $props();
</script>

<button class={variant}>{text}</button>
```

This immediately tells you:
- What props this component accepts (`text`, `variant`)
- Which props have defaults (`variant` defaults to `'primary'`)
- Which props are required (`text` has no default)

### Props Are Read-Only

**Critical rule**: Props are immutable from the child component's perspective. You **cannot** reassign them:

```svelte
<script>
  let { count } = $props();
  
  // ❌ WRONG: Cannot reassign props
  count = count + 1;
  
  // ✅ CORRECT: Create local state if you need to modify
  let localCount = $state(count);
  localCount = localCount + 1;
</script>
```

Props flow **one way**: from parent to child. If you need two-way communication, use callbacks or `$bindable()` (we'll cover this in Article 9).

---

## Live Example: Building a Button Component

Let's build a real, production-ready Button component step by step.

### Step 1: Basic Props

```svelte
<!-- Button.svelte -->
<script>
  let { text } = $props();
</script>

<button>{text}</button>

<style>
  button {
    padding: 0.5rem 1rem;
    border-radius: 0.25rem;
    border: none;
    cursor: pointer;
  }
</style>
```

Usage:
```svelte
<Button text="Click me" />
```

### Step 2: Adding Variants with Defaults

```svelte
<script>
  let { 
    text,
    variant = 'primary' 
  } = $props();
</script>

<button class={variant}>
  {text}
</button>

<style>
  button {
    padding: 0.5rem 1rem;
    border-radius: 0.25rem;
    border: none;
    cursor: pointer;
    font-weight: 500;
  }
  
  .primary {
    background: #3b82f6;
    color: white;
  }
  
  .secondary {
    background: #6b7280;
    color: white;
  }
  
  .danger {
    background: #ef4444;
    color: white;
  }
</style>
```

Usage:
```svelte
<Button text="Save" />  <!-- primary by default -->
<Button text="Cancel" variant="secondary" />
<Button text="Delete" variant="danger" />
```

### Step 3: Multiple Props with Complex Defaults

```svelte
<script>
  let { 
    text,
    variant = 'primary',
    size = 'medium',
    disabled = false,
    loading = false
  } = $props();
  
  let classes = $derived(() => {
    let result = [variant, size];
    if (disabled) result.push('disabled');
    if (loading) result.push('loading');
    return result.join(' ');
  });
</script>

<button 
  class={classes}
  disabled={disabled || loading}
>
  {#if loading}
    <span class="spinner"></span>
  {/if}
  {text}
</button>

<style>
  button {
    padding: 0.5rem 1rem;
    border-radius: 0.25rem;
    border: none;
    cursor: pointer;
    font-weight: 500;
    display: inline-flex;
    align-items: center;
    gap: 0.5rem;
    transition: all 0.2s;
  }
  
  .primary { background: #3b82f6; color: white; }
  .secondary { background: #6b7280; color: white; }
  .danger { background: #ef4444; color: white; }
  
  .small { padding: 0.25rem 0.5rem; font-size: 0.875rem; }
  .medium { padding: 0.5rem 1rem; font-size: 1rem; }
  .large { padding: 0.75rem 1.5rem; font-size: 1.125rem; }
  
  .disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }
  
  .loading {
    cursor: wait;
  }
  
  .spinner {
    width: 1em;
    height: 1em;
    border: 2px solid currentColor;
    border-right-color: transparent;
    border-radius: 50%;
    animation: spin 0.6s linear infinite;
  }
  
  @keyframes spin {
    to { transform: rotate(360deg); }
  }
</style>
```

Usage:
```svelte
<Button text="Save" loading={isSaving} />
<Button text="Delete" variant="danger" size="small" />
<Button text="Cancel" disabled={true} />
```

### Step 4: Event Handlers as Props

```svelte
<script>
  let { 
    text,
    variant = 'primary',
    size = 'medium',
    disabled = false,
    loading = false,
    onclick = () => {} // Event handler prop
  } = $props();
  
  let classes = $derived(() => {
    let result = [variant, size];
    if (disabled) result.push('disabled');
    if (loading) result.push('loading');
    return result.join(' ');
  });
</script>

<button 
  class={classes}
  disabled={disabled || loading}
  onclick={onclick}
>
  {#if loading}
    <span class="spinner"></span>
  {/if}
  {text}
</button>
```

Usage:
```svelte
<script>
  let saving = $state(false);
  
  async function handleSave() {
    saving = true;
    await saveData();
    saving = false;
  }
</script>

<Button 
  text="Save" 
  loading={saving}
  onclick={handleSave}
/>
```

### Step 5: Rest Props and Forwarding Attributes

Often you want to forward HTML attributes to the underlying element:

```svelte
<script>
  let { 
    text,
    variant = 'primary',
    size = 'medium',
    disabled = false,
    loading = false,
    onclick = () => {},
    ...rest // Capture all other props
  } = $props();
  
  let classes = $derived(() => {
    let result = [variant, size];
    if (disabled) result.push('disabled');
    if (loading) result.push('loading');
    return result.join(' ');
  });
</script>

<button 
  class={classes}
  disabled={disabled || loading}
  onclick={onclick}
  {...rest}  <!-- Forward remaining attributes -->
>
  {#if loading}
    <span class="spinner"></span>
  {/if}
  {text}
</button>
```

Now users can pass any HTML attribute:

```svelte
<Button 
  text="Submit" 
  type="submit"
  form="my-form"
  aria-label="Submit form"
  data-testid="submit-btn"
/>
```

---

## TypeScript: Typing Props

For TypeScript users, props should be properly typed:

```svelte
<script lang="ts">
  interface Props {
    text: string;
    variant?: 'primary' | 'secondary' | 'danger';
    size?: 'small' | 'medium' | 'large';
    disabled?: boolean;
    loading?: boolean;
    onclick?: (event: MouseEvent) => void;
    [key: string]: any; // For rest props
  }
  
  let { 
    text,
    variant = 'primary',
    size = 'medium',
    disabled = false,
    loading = false,
    onclick = () => {},
    ...rest
  }: Props = $props();
</script>
```

This gives you:
- ✅ Autocomplete for prop names
- ✅ Type checking when using the component
- ✅ Documentation in your IDE
- ✅ Compile-time error detection

---

## Common Mistakes

### ❌ Mistake 1: Mutating Props

```svelte
<script>
  let { count } = $props();
  
  // WRONG: Props are read-only
  count++;
</script>
```

**Fix**: Create local state:
```svelte
<script>
  let { count } = $props();
  let localCount = $state(count);
  
  localCount++; // ✅ Modify local state
</script>
```

### ❌ Mistake 2: Not Providing Defaults for Optional Props

```svelte
<script>
  let { variant } = $props(); // What if undefined?
</script>

<button class={variant}>Click</button> <!-- class="undefined" -->
```

**Fix**: Always provide defaults:
```svelte
<script>
  let { variant = 'primary' } = $props();
</script>
```

### ❌ Mistake 3: Too Many Props

```svelte
<script>
  let { 
    color, 
    backgroundColor, 
    fontSize, 
    fontWeight, 
    padding, 
    margin,
    borderRadius,
    // ... 20 more style props
  } = $props();
</script>
```

**Fix**: Use `class` prop and CSS:
```svelte
<script>
  let { variant = 'primary', class: className = '' } = $props();
</script>

<button class="{variant} {className}">
  <slot />
</button>
```

### ❌ Mistake 4: Inconsistent Naming

```svelte
<!-- Some components -->
<Button isDisabled={true} />
<Input disabled={true} />
<Select isEnabled={false} />
```

**Fix**: Be consistent across your component library:
```svelte
<!-- All components use the same names -->
<Button disabled={true} />
<Input disabled={true} />
<Select disabled={true} />
```

---

## Best Practices

### 1. **Required vs Optional Props**

Make your API obvious:

```svelte
<script>
  // Required props: no default
  let { 
    id,    // Required
    title  // Required
  } = $props();
  
  // Optional props: have defaults
  let {
    description = '',
    tags = [],
    published = false
  } = $props();
</script>
```

### 2. **Prop Validation**

For critical props, validate them:

```svelte
<script>
  let { variant = 'primary' } = $props();
  
  const validVariants = ['primary', 'secondary', 'danger'];
  
  $effect(() => {
    if (!validVariants.includes(variant)) {
      console.warn(
        `Invalid variant "${variant}". Must be one of: ${validVariants.join(', ')}`
      );
    }
  });
</script>
```

### 3. **Computed Classes**

Use `$derived()` for complex class logic:

```svelte
<script>
  let { variant, size, disabled, loading } = $props();
  
  let classes = $derived(() => {
    const base = 'btn';
    const parts = [base, `${base}--${variant}`, `${base}--${size}`];
    
    if (disabled) parts.push(`${base}--disabled`);
    if (loading) parts.push(`${base}--loading`);
    
    return parts.join(' ');
  });
</script>

<button class={classes}>
  <slot />
</button>
```

### 4. **Rest Props for Flexibility**

Always include rest props for extensibility:

```svelte
<script>
  let { 
    // Your custom props
    variant = 'primary',
    // Everything else
    ...rest 
  } = $props();
</script>

<button {...rest} class={variant}>
  <slot />
</button>
```

### 5. **Use Slots for Content**

For flexible content, prefer slots over `text` props:

```svelte
<!-- Instead of this: -->
<Button text="Save" />

<!-- Do this: -->
<Button>
  <Icon name="save" />
  Save
</Button>
```

Component:
```svelte
<script>
  let { variant = 'primary', ...rest } = $props();
</script>

<button class={variant} {...rest}>
  <slot />  <!-- Flexible content -->
</button>
```

---

## Mental Model: Props as a Contract

Think of props as a **contract** between your component and its users:

```
┌─────────────────────────────────────────┐
│         Component Contract              │
├─────────────────────────────────────────┤
│                                         │
│  INPUTS (Props):                        │
│  - text: string (required)              │
│  - variant: 'primary' | ... (optional)  │
│  - disabled: boolean (optional)         │
│                                         │
│  OUTPUTS (Events/Callbacks):            │
│  - onclick: (e) => void                 │
│                                         │
│  GUARANTEES:                            │
│  - Will render a <button> element       │
│  - Will apply variant styling           │
│  - Will disable when disabled=true      │
│                                         │
└─────────────────────────────────────────┘
```

A good contract is:
- **Clear**: Obvious what props do
- **Predictable**: Same inputs → same output
- **Documented**: Types and defaults are explicit
- **Flexible**: Rest props allow extension
- **Consistent**: Similar props across components

---

## Quick Reference

### Declaring Props

```svelte
<script>
  // Basic
  let props = $props();
  
  // Destructured
  let { name, age } = $props();
  
  // With defaults
  let { name, age = 18 } = $props();
  
  // With rest
  let { name, ...rest } = $props();
  
  // TypeScript
  let { name, age = 18 }: { name: string; age?: number } = $props();
</script>
```

### Common Patterns

```svelte
<!-- Boolean props -->
let { disabled = false } = $props();
<button {disabled}>...</button>

<!-- Variant props -->
let { variant = 'primary' } = $props();
<div class={variant}>...</div>

<!-- Event handlers -->
let { onclick = () => {} } = $props();
<button {onclick}>...</button>

<!-- Forwarding attributes -->
let { class: className, ...rest } = $props();
<div class={className} {...rest}>...</div>
```

### Props Checklist

- [ ] Required props have no default
- [ ] Optional props have sensible defaults
- [ ] Props are typed (TypeScript)
- [ ] Event handlers have empty function defaults
- [ ] Rest props are forwarded
- [ ] Class names can be extended
- [ ] Validation for critical props (dev mode)

---

## Practice Exercise: Build a Card Component

Create a reusable `Card` component with these requirements:

**Required Props:**
- None (all optional)

**Optional Props:**
- `title`: string - Card title
- `subtitle`: string - Card subtitle  
- `variant`: 'default' | 'outlined' | 'elevated' - Card style
- `padding`: 'none' | 'small' | 'medium' | 'large' - Internal padding
- `clickable`: boolean - Makes card hoverable/clickable
- `onclick`: function - Click handler (if clickable)

**Features:**
- Should use slots for card content
- Should forward rest props
- Should have proper hover states when clickable
- Should support TypeScript

**Starter Code:**

```svelte
<!-- Card.svelte -->
<script>
  // Your implementation here
</script>

<div>
  <!-- Your markup here -->
</div>

<style>
  /* Your styles here */
</style>
```

**Usage Example:**

```svelte
<Card 
  title="Welcome"
  subtitle="Get started with Svelte 5"
  variant="elevated"
  padding="medium"
  clickable={true}
  onclick={() => console.log('Card clicked')}
>
  <p>This is the card content.</p>
  <button>Learn More</button>
</Card>
```

**Solution:** Try it yourself first! Check your approach against the patterns in this article.

---

## Next Steps

Congratulations! You've completed Phase 1: Foundation. You now understand:
- ✅ Why Svelte 5 uses runes (Article 1)
- ✅ `$state()` for reactive values (Article 2)
- ✅ `$derived()` for computed values (Article 3)
- ✅ `$effect()` for side effects (Article 4)
- ✅ `$props()` for component communication (Article 5)

**Coming up in Phase 2: Practical Patterns**

**Article 6: "Arrays - Lists That React"**  
You'll learn how to manage collections of data reactively. We'll explore why array mutations don't work in Svelte 5, immutable update patterns, and build a fully functional todo list with filtering and statistics.

**Preview:**
```svelte
<script>
  // Why doesn't this work?
  let todos = $state([]);
  todos.push({ text: 'Learn Runes' }); // ❌ No reactivity
  
  // How to do it right:
  todos = [...todos, { text: 'Learn Runes' }]; // ✅ Reactive
</script>
```

Arrays are everywhere in real applications—user lists, shopping carts, data tables. Mastering reactive arrays is essential for building real-world Svelte apps.

**See you in Article 6!** 🚀

---

## Additional Resources

- [Svelte 5 Props Documentation](https://svelte.dev/docs/svelte/$props)
- [Component API Design Guidelines](https://component.gallery)
- [TypeScript in Svelte](https://svelte.dev/docs/typescript)
- [GitHub: Article 5 Examples](https://github.com/yourusername/svelte-5-runes-series/tree/main/article-5)

---

**📚 Series Navigation:**
← [Article 4: $effect](/article-4) | [Article 6: Arrays →](/article-6)
