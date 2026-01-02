# $state - Your First Rune

## Why This Matters

Remember the last time you stared at your UI, wondering why it didn't update when you changed a variable? Or when you mutated an array and nothing happened? In Svelte 4, reactivity often felt magical—until it didn't. You'd add a `$:` here, wonder why it didn't trigger there, and eventually resort to workarounds that made your code harder to understand.

`$state()` is Svelte 5's answer to this confusion. It's explicit, predictable, and once you understand it, you'll never wonder "will this trigger a re-render?" again. Instead of magic behind the scenes, you declare exactly what should be reactive, and Svelte handles the rest with perfect consistency.

Think of `$state()` as wrapping your data in a container that says "watch this." When the container's contents change, Svelte knows to update your UI. It's simple, but this simplicity is what makes complex applications manageable. Let's master this fundamental building block.

---

## Learning Goals

By the end of this article, you'll be able to:

- Create reactive state for primitives (numbers, strings, booleans)
- Understand why reassignment works but mutation doesn't
- Choose between `$state()` and `$state.raw()` confidently
- Manage multiple pieces of state in a single component
- Avoid the 5 most common beginner mistakes
- Develop a mental model for thinking about reactive state

---

## Prerequisites

You should be comfortable with:
- Basic JavaScript variables and data types
- Svelte component structure (.svelte files)
- Event handlers in Svelte (`on:click`)

If you haven't read Article 1 ("Why Svelte 5 Changed Everything"), start there to understand the context behind this new approach.

---

## Core Concept: Reactive Containers

### What is `$state()`?

`$state()` is a **rune**—a compiler signal that tells Svelte "this value should be reactive." When you wrap a value with `$state()`, you're creating a reactive container. Any time you reassign the value inside that container, Svelte tracks the change and updates your UI.

Here's the simplest example:

```svelte
<script>
  let count = $state(0);
</script>

<button onclick={() => count++}>
  Clicks: {count}
</button>
```

Every time you click the button, `count` is reassigned (`count++` is shorthand for `count = count + 1`), and the UI updates instantly.

### The Golden Rule: Reassignment, Not Mutation

This is the most important concept to grasp:

**✅ Reassignment triggers reactivity:**
```javascript
count = count + 1;  // Creates new value
name = "Alice";     // Creates new value
isActive = !isActive;  // Creates new value
```

**❌ Mutation does NOT trigger reactivity (for objects/arrays):**
```javascript
// These DON'T work as you might expect:
user.name = "Bob";     // Mutates existing object
items.push("new");     // Mutates existing array
settings.theme = "dark";  // Mutates existing object
```

Why? Because Svelte tracks when you assign a new value to a reactive variable. Mutation changes the internals of an existing value without reassigning the variable itself.

**The fix? Reassign with a new value:**
```javascript
user = { ...user, name: "Bob" };  // ✅ New object
items = [...items, "new"];        // ✅ New array
settings = { ...settings, theme: "dark" };  // ✅ New object
```

Don't worry—we'll cover arrays and objects in detail in Articles 6-8. For now, focus on primitives.

### Primitives: The Easy Case

Primitives (numbers, strings, booleans, null, undefined) are **always** reassigned, never mutated:

```svelte
<script>
  let count = $state(0);
  let name = $state("World");
  let isVisible = $state(true);
  let selected = $state(null);
</script>

<button onclick={() => count++}>Count: {count}</button>
<button onclick={() => name = name + "!"}>Name: {name}</button>
<button onclick={() => isVisible = !isVisible}>Toggle</button>
<button onclick={() => selected = count}>Select: {selected}</button>
```

Every button click reassigns a variable, so everything works perfectly.

### `$state()` vs `$state.raw()`

Svelte 5 provides two flavors of state:

#### `$state()` - Deep Reactivity (Default)

```javascript
let user = $state({ name: "Alice", age: 25 });
```

With `$state()`, Svelte makes the entire object deeply reactive. This means:
- Changes to `user` itself are tracked
- Changes to `user.name` are tracked
- Changes to nested properties are tracked
- It "just works" for complex structures

**When to use:** Most of the time, especially for objects and arrays.

#### `$state.raw()` - Shallow Reactivity

```javascript
let user = $state.raw({ name: "Alice", age: 25 });
```

With `$state.raw()`, only the variable itself is tracked:
- Changes to `user` (reassignment) are tracked
- Changes to `user.name` are NOT tracked
- You must reassign the entire object: `user = { ...user, name: "Bob" }`

**When to use:** 
- Large data structures where deep tracking is expensive
- When you know you'll always reassign the whole value
- Performance-critical code (covered in Article 14)

**For primitives, both are identical:**
```javascript
let count = $state(0);
let count = $state.raw(0);
// These behave exactly the same for numbers/strings/booleans
```

---

## Live Example: Multi-Counter Component

Let's build something practical—a component with multiple pieces of state working together:

```svelte
<script>
  // Individual state variables
  let clicks = $state(0);
  let name = $state("");
  let isActive = $state(false);
  let message = $state("Hello!");
  
  // Functions that update state
  function increment() {
    clicks = clicks + 1;
  }
  
  function reset() {
    clicks = 0;
    name = "";
    isActive = false;
    message = "Reset!";
  }
  
  function toggle() {
    isActive = !isActive;
    message = isActive ? "Activated!" : "Deactivated!";
  }
</script>

<div class="counter-app">
  <h1>Multi-Counter Demo</h1>
  
  <!-- Click counter -->
  <div class="section">
    <h2>Clicks: {clicks}</h2>
    <button onclick={increment}>Click Me</button>
    <button onclick={() => clicks = clicks + 10}>+10</button>
    <button onclick={() => clicks = Math.max(0, clicks - 1)}>-1</button>
  </div>
  
  <!-- Text input -->
  <div class="section">
    <h2>Name: {name || "(empty)"}</h2>
    <input 
      type="text" 
      value={name}
      oninput={(e) => name = e.target.value}
      placeholder="Enter your name"
    />
    <p>Length: {name.length} characters</p>
  </div>
  
  <!-- Toggle state -->
  <div class="section">
    <h2>Status: {isActive ? "Active" : "Inactive"}</h2>
    <button onclick={toggle}>Toggle</button>
    <p class:active={isActive}>{message}</p>
  </div>
  
  <!-- Reset all -->
  <div class="section">
    <button onclick={reset} class="reset">Reset Everything</button>
  </div>
</div>

<style>
  .counter-app {
    padding: 2rem;
    font-family: system-ui, sans-serif;
  }
  
  .section {
    margin: 2rem 0;
    padding: 1rem;
    border: 2px solid #e0e0e0;
    border-radius: 8px;
  }
  
  button {
    margin: 0.5rem;
    padding: 0.5rem 1rem;
    font-size: 1rem;
    cursor: pointer;
  }
  
  input {
    padding: 0.5rem;
    font-size: 1rem;
    width: 200px;
  }
  
  .active {
    color: green;
    font-weight: bold;
  }
  
  .reset {
    background: #ff4444;
    color: white;
    border: none;
  }
</style>
```

### What's Happening Here?

1. **Four independent state variables:** Each tracks a different piece of data
2. **Direct reassignment:** `clicks = clicks + 1` updates the UI immediately
3. **Event handlers:** Both inline (`() => clicks++`) and function references work
4. **String concatenation:** `name = e.target.value` updates as you type
5. **Boolean logic:** `isActive = !isActive` toggles between states
6. **Computed display:** `{name || "(empty)"}` and `{name.length}` update reactively

Notice how we never call any special "update" or "setState" function. Reassignment is all you need.

---

## Common Mistakes

### Mistake 1: Forgetting to Initialize

```javascript
// ❌ Wrong - not reactive
let count;

// ✅ Correct
let count = $state(0);
```

**Why it matters:** Without `$state()`, `count` is just a regular variable. Changes won't trigger UI updates.

### Mistake 2: Trying to Mutate Objects

```javascript
let user = $state({ name: "Alice" });

// ❌ Wrong - mutation doesn't work
user.name = "Bob";

// ✅ Correct - reassign with new object
user = { ...user, name: "Bob" };
```

**Note:** Actually, with `$state()` (not `.raw()`), the mutation DOES work due to deep reactivity. But it's best to build the habit of reassignment for consistency, especially when you move to arrays. We'll clarify this in Article 7.

### Mistake 3: Using `const` Instead of `let`

```javascript
// ❌ Wrong - can't reassign const
const count = $state(0);
count = 1;  // Error!

// ✅ Correct
let count = $state(0);
count = 1;  // Works!
```

**Why it matters:** Reactivity requires reassignment, and you can't reassign `const` variables.

### Mistake 4: Not Understanding Scope

```javascript
<script>
  let count = $state(0);
  
  function increment() {
    // ❌ Wrong - creates new local variable
    let count = count + 1;
    
    // ✅ Correct - updates the outer variable
    count = count + 1;
  }
</script>
```

### Mistake 5: Overthinking It

```javascript
// ❌ Overthinking - you don't need this
let count = $state(0);
function setCount(newValue) {
  count = newValue;
}
setCount(5);

// ✅ Keep it simple
let count = $state(0);
count = 5;
```

**Why it matters:** Unlike React, you don't need setter functions. Direct assignment is the Svelte way.

---

## Best Practices

### 1. Initialize with Meaningful Defaults

```javascript
// ✅ Good - clear intent
let count = $state(0);
let userName = $state("");
let isLoggedIn = $state(false);

// ❌ Avoid - confusing undefined states
let count = $state();
let userName = $state(null);
```

### 2. Use Descriptive Variable Names

```javascript
// ✅ Good - clear purpose
let userClickCount = $state(0);
let isModalOpen = $state(false);
let selectedProductId = $state(null);

// ❌ Avoid - unclear meaning
let x = $state(0);
let flag = $state(false);
let temp = $state(null);
```

### 3. Group Related State

```javascript
// ❌ Avoid - too granular
let firstName = $state("");
let lastName = $state("");
let email = $state("");
let age = $state(0);

// ✅ Better - group as object (covered in Article 7)
let user = $state({
  firstName: "",
  lastName: "",
  email: "",
  age: 0
});
```

### 4. Keep State at the Right Level

```javascript
// ❌ Avoid - global when it should be local
<script context="module">
  let clickCount = $state(0);  // Shared across all instances!
</script>

// ✅ Correct - local to component
<script>
  let clickCount = $state(0);  // Each instance has its own
</script>
```

### 5. Use Functions for Complex Updates

```javascript
let count = $state(0);

// ✅ Good - clear and reusable
function incrementBy(amount) {
  count = count + amount;
}

function resetCount() {
  count = 0;
}

// Usage is clean
<button onclick={() => incrementBy(5)}>+5</button>
<button onclick={resetCount}>Reset</button>
```

---

## Mental Model: The Reactive Container

Think of `$state()` as a **glass box** around your data:

```
Regular variable:     count = 5
                      (Svelte can't see changes)

Reactive state:       ┌──────────┐
                      │ count: 5 │  ← Glass box
                      └──────────┘
                      (Svelte watches this box)
```

When you reassign:
```javascript
count = 6;  // Svelte sees you put new value in the box
```

The glass box analogy helps because:
1. **Transparent:** You can always read the current value
2. **Observable:** Svelte can see when the contents change
3. **Simple:** You just put new things in; Svelte does the rest

For objects/arrays (covered later):
```javascript
// With $state() - the box is magic, tracks everything inside
let user = $state({ name: "Alice" });
user.name = "Bob";  // Magic box notices

// With $state.raw() - regular glass box
let user = $state.raw({ name: "Alice" });
user.name = "Bob";  // Box doesn't notice (still has same object)
user = { name: "Bob" };  // Box notices (new object)
```

**Key insight:** Svelte doesn't care about the value itself—it watches for reassignment of the variable.

---

## Quick Reference

### Basic Syntax

```javascript
// Primitives
let count = $state(0);
let name = $state("Alice");
let isActive = $state(true);
let data = $state(null);

// Update via reassignment
count = count + 1;
count++;
count += 5;

name = name.toUpperCase();
name = `Hello, ${name}`;

isActive = !isActive;
isActive = true;
```

### `$state()` vs `$state.raw()`

```javascript
// Deep reactivity (default)
let user = $state({ name: "Alice" });
user.name = "Bob";  // Works (tracks nested changes)

// Shallow reactivity (performance)
let user = $state.raw({ name: "Alice" });
user.name = "Bob";  // Doesn't trigger update
user = { ...user, name: "Bob" };  // Does trigger update

// For primitives, both are identical
let count = $state(0);       // Same as...
let count = $state.raw(0);   // ...this
```

### Common Patterns

```javascript
// Toggle
let isOpen = $state(false);
isOpen = !isOpen;

// Increment/decrement
let count = $state(0);
count++;
count--;
count += 10;
count = Math.max(0, count - 1);  // With minimum

// String manipulation
let text = $state("");
text = text.toUpperCase();
text = text + "!";
text = text.trim();

// Conditional updates
let status = $state("idle");
status = isError ? "error" : "success";

// Reset to default
let count = $state(0);
count = 0;  // Reset
```

### Event Handler Patterns

```javascript
// Inline arrow function
<button onclick={() => count++}>Click</button>

// Function reference
<button onclick={increment}>Click</button>

// With event object
<input oninput={(e) => name = e.target.value} />

// Multiple updates
<button onclick={() => {
  count++;
  message = "Clicked!";
}}>
  Click
</button>
```

---

## Practice Exercise

Build a **temperature converter** that demonstrates multiple state variables:

### Requirements:

1. Two input fields: Celsius and Fahrenheit
2. Update Fahrenheit when Celsius changes (and vice versa)
3. Display a message if temperature is below freezing or above boiling
4. Add a "Reset" button that clears both values
5. Show the count of how many conversions have been done

### Starter Template:

```svelte
<script>
  let celsius = $state(0);
  let fahrenheit = $state(32);
  let conversionCount = $state(0);
  let message = $state("");
  
  function updateFromCelsius(value) {
    // Your code here
  }
  
  function updateFromFahrenheit(value) {
    // Your code here
  }
  
  function reset() {
    // Your code here
  }
  
  function updateMessage() {
    // Your code here
  }
</script>

<div>
  <h1>Temperature Converter</h1>
  
  <!-- Add your inputs and UI here -->
  
  <p>Conversions done: {conversionCount}</p>
  {#if message}
    <p class="message">{message}</p>
  {/if}
</div>
```

### Bonus Challenges:

1. Add Kelvin conversion
2. Add number validation (prevent non-numeric input)
3. Style the message differently based on temperature range
4. Add keyboard shortcuts (R for reset, C to focus Celsius, etc.)

### Solution Hints:

- Celsius to Fahrenheit: `F = (C × 9/5) + 32`
- Fahrenheit to Celsius: `C = (F - 32) × 5/9`
- Remember to increment `conversionCount` on each conversion
- Use `parseFloat()` to convert string input to numbers

---

## Next Steps

You now understand the foundation of Svelte 5 reactivity! You can create reactive state, update it through reassignment, and manage multiple pieces of state in a component.

But what about values that are calculated from state? What if you want to show a total, a filtered list, or a formatted string that updates automatically when the underlying state changes?

That's where **`$derived()`** comes in. In Article 3, we'll explore computed values—how to create values that automatically update based on state without manual recalculation. You'll learn the difference between state and derived values, when to use each, and how they work together to create reactive UIs.

**Coming up in Article 3: "$derived - Computed Values Done Right"**
- Automatic calculations that update with state
- `$derived()` vs `$derived.by()` for complex logic
- Performance benefits of memoization
- Building a shopping cart with automatic totals

---

## Key Takeaways

✅ `$state()` creates reactive containers for your data
✅ Reassignment triggers reactivity; mutation doesn't (for objects/arrays)
✅ Primitives always work with simple reassignment
✅ Use `let`, never `const`, for state variables
✅ `$state.raw()` is for performance optimization (covered more in Article 14)
✅ Direct assignment is the Svelte way—no setter functions needed
✅ Think of state as glass boxes that Svelte watches for changes

**Remember:** If it doesn't update, ask yourself: "Did I reassign the variable?" That's the key to debugging 90% of reactivity issues in Svelte 5.

---

Ready to level up? Head to Article 3 and learn how `$derived()` makes computed values effortless!