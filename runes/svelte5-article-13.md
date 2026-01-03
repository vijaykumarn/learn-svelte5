# Article 13: $inspect - Debugging Reactivity

## 🔍 The Invisible Problem

You've built a Svelte 5 component. Everything compiles. No errors in the console. But something's wrong—your UI isn't updating when it should, or worse, it's updating *too much*, causing performance issues. You add `console.log()` statements everywhere, but they only fire once. The reactive updates are invisible.

This is the debugging challenge of reactive systems: **you can't see what's happening between state changes**. Traditional debugging tools show you the "before" and "after," but the reactive machinery in between is a black box. When a derived value recalculates, when an effect re-runs, which dependencies triggered the update—all invisible.

Enter `$inspect()`, Svelte 5's built-in debugging superpower. It's like `console.log()` on steroids, designed specifically for reactive systems. It automatically logs whenever reactive values change, shows you exactly which dependencies triggered updates, and helps you understand the invisible web of reactivity in your application. Let's learn how to use it to debug like a pro.

---

## 🎯 Learning Goals

By the end of this article, you'll be able to:

- ✅ Use `$inspect()` to track reactive state changes automatically
- ✅ Understand why `$inspect()` is better than `console.log()` for reactive code
- ✅ Format debug output with `$inspect().with()` for custom logging
- ✅ Trace dependency chains to understand what triggers updates
- ✅ Identify and fix unnecessary re-renders and performance issues
- ✅ Use `$state.snapshot()` to capture non-reactive copies of data
- ✅ Debug complex reactive flows in real-world applications

---

## 📚 Prerequisites

Before diving in, you should understand:

- **$state** - Creating reactive state (Article 2)
- **$derived** - Computed values (Article 3)
- **$effect** - Side effects (Article 4)
- **Basic debugging** - Using browser DevTools console

If you're comfortable with these concepts, you're ready!

---

## 💡 Core Concept: Reactive Debugging

### The Problem with console.log()

Let's start with why regular debugging doesn't work well with reactive systems:

```svelte
<script>
  let count = $state(0);
  
  console.log('count:', count); // Only logs once on init
  
  let doubled = $derived(count * 2);
  
  console.log('doubled:', doubled); // Only logs once on init
</script>

<button onclick={() => count++}>
  {count} (doubled: {doubled})
</button>
```

**What happens?** Both `console.log()` statements run once when the component initializes. When you click the button, `count` and `doubled` update in the UI, but nothing logs to the console. You're blind to the reactive updates.

You could add logging inside an effect:

```svelte
<script>
  let count = $state(0);
  let doubled = $derived(count * 2);
  
  $effect(() => {
    console.log('count:', count, 'doubled:', doubled);
  });
</script>
```

This works, but it's verbose and requires you to manually track dependencies. You also need to remember to remove it before production.

### Enter $inspect()

`$inspect()` is designed specifically for this use case:

```svelte
<script>
  let count = $state(0);
  
  $inspect(count); // Logs automatically on every change!
  
  let doubled = $derived(count * 2);
  
  $inspect(doubled); // Logs automatically on every change!
</script>

<button onclick={() => count++}>
  {count} (doubled: {doubled})
</button>
```

**Output in console (after clicking 3 times):**
```
init: 0
init: 0
update: 1
update: 2
update: 2
update: 4
update: 3
update: 6
```

Notice how `$inspect()`:
1. **Automatically re-runs** whenever the inspected value changes
2. **Labels output** with "init" or "update" to show the lifecycle
3. **Requires no manual effect** - it's a single-line debug statement
4. **Is tree-shakeable** - removed from production builds automatically

### How $inspect() Works

Under the hood, `$inspect()` creates a hidden `$effect()` that tracks the reactive value. But it's smarter:

- It only runs in development mode
- It's automatically removed from production bundles
- It has special formatting in the console
- It shows you when dependencies trigger updates

---

## 🔨 Live Example: Debugging a Search Filter

Let's debug a real-world example—a search filter with derived results:

```svelte
<script>
  let searchTerm = $state('');
  let minPrice = $state(0);
  let maxPrice = $state(1000);
  
  // Sample products
  let products = $state([
    { id: 1, name: 'Laptop', price: 899 },
    { id: 2, name: 'Mouse', price: 25 },
    { id: 3, name: 'Keyboard', price: 75 },
    { id: 4, name: 'Monitor', price: 299 },
    { id: 5, name: 'Headphones', price: 150 }
  ]);
  
  // Derived filtered results
  let filteredProducts = $derived(
    products.filter(p => 
      p.name.toLowerCase().includes(searchTerm.toLowerCase()) &&
      p.price >= minPrice &&
      p.price <= maxPrice
    )
  );
  
  // Debug the reactive flow
  $inspect(searchTerm);
  $inspect(minPrice, maxPrice);
  $inspect(filteredProducts);
</script>

<div>
  <input bind:value={searchTerm} placeholder="Search products...">
  
  <label>
    Min: ${minPrice}
    <input type="range" bind:value={minPrice} min="0" max="1000">
  </label>
  
  <label>
    Max: ${maxPrice}
    <input type="range" bind:value={maxPrice} min="0" max="1000">
  </label>
  
  <h3>Results: {filteredProducts.length}</h3>
  
  {#each filteredProducts as product}
    <div>{product.name} - ${product.price}</div>
  {/each}
</div>
```

**What you'll see in the console:**

When you type "la" in the search:
```
init: ""
init: 0 1000
init: (5) [{…}, {…}, {…}, {…}, {…}]
update: "l"
update: (1) [{id: 1, name: 'Laptop', price: 899}]
update: "la"
```

Notice that:
1. `searchTerm` updates with each keystroke
2. `filteredProducts` recalculates automatically
3. `minPrice` and `maxPrice` don't log because they didn't change

This helps you understand the reactive flow: **search input changes → searchTerm updates → filteredProducts recalculates → UI updates**.

---

## 🎨 Custom Formatting with .with()

Sometimes you want more control over the debug output. Use `$inspect().with()`:

```svelte
<script>
  let cart = $state([
    { id: 1, name: 'Laptop', price: 899, qty: 1 },
    { id: 2, name: 'Mouse', price: 25, qty: 2 }
  ]);
  
  let total = $derived(
    cart.reduce((sum, item) => sum + (item.price * item.qty), 0)
  );
  
  // Custom formatting
  $inspect(cart).with((type, value) => {
    console.log(`[CART ${type.toUpperCase()}]`);
    console.table(value);
  });
  
  $inspect(total).with((type, value) => {
    console.log(`💰 Total (${type}): $${value.toFixed(2)}`);
  });
</script>
```

**Output:**
```
[CART INIT]
┌─────────┬────┬──────────┬───────┬─────┐
│ (index) │ id │   name   │ price │ qty │
├─────────┼────┼──────────┼───────┼─────┤
│    0    │ 1  │ 'Laptop' │  899  │  1  │
│    1    │ 2  │ 'Mouse'  │  25   │  2  │
└─────────┴────┴──────────┴───────┴─────┘
💰 Total (init): $949.00
```

The `.with()` method receives:
- **type**: `'init'` or `'update'`
- **value**: The current value of the inspected state

This lets you format output however you want: tables, colors, emojis, conditional logging, etc.

---

## 🕵️ Tracking Dependency Chains

One of the hardest debugging challenges is understanding *why* something updated. Let's trace a complex dependency chain:

```svelte
<script>
  // Level 1: Base state
  let firstName = $state('John');
  let lastName = $state('Doe');
  
  // Level 2: Derived from base state
  let fullName = $derived(`${firstName} ${lastName}`);
  
  // Level 3: Derived from level 2
  let greeting = $derived(`Hello, ${fullName}!`);
  
  // Level 4: Derived from level 3
  let formalGreeting = $derived(`${greeting} Welcome to our site.`);
  
  // Debug the chain
  $inspect(firstName).with((type) => 
    console.log(`🔵 [${type}] firstName changed`)
  );
  
  $inspect(lastName).with((type) => 
    console.log(`🔵 [${type}] lastName changed`)
  );
  
  $inspect(fullName).with((type, value) => 
    console.log(`🟢 [${type}] fullName recomputed:`, value)
  );
  
  $inspect(greeting).with((type, value) => 
    console.log(`🟡 [${type}] greeting recomputed:`, value)
  );
  
  $inspect(formalGreeting).with((type, value) => 
    console.log(`🔴 [${type}] formalGreeting recomputed:`, value)
  );
</script>

<input bind:value={firstName} placeholder="First name">
<input bind:value={lastName} placeholder="Last name">

<p>{formalGreeting}</p>
```

**When you type in firstName, the console shows:**
```
🔵 [update] firstName changed
🟢 [update] fullName recomputed: John Doe
🟡 [update] greeting recomputed: Hello, John Doe!
🔴 [update] formalGreeting recomputed: Hello, John Doe! Welcome to our site.
```

This cascade visualization shows you exactly how changes propagate through your reactive system. You can see that changing `firstName` triggers a waterfall of recalculations.

---

## 🐛 Debugging Performance Issues

Let's use `$inspect()` to find and fix a performance problem:

```svelte
<script>
  let items = $state(Array.from({ length: 1000 }, (_, i) => ({
    id: i,
    name: `Item ${i}`,
    value: Math.random() * 100
  })));
  
  let searchTerm = $state('');
  let sortBy = $state('name');
  
  // 🚨 PROBLEM: This recalculates on every searchTerm change
  let sortedItems = $derived(
    [...items].sort((a, b) => {
      if (sortBy === 'name') return a.name.localeCompare(b.name);
      return a.value - b.value;
    })
  );
  
  // Then filter the sorted items
  let filteredItems = $derived(
    sortedItems.filter(item => 
      item.name.toLowerCase().includes(searchTerm.toLowerCase())
    )
  );
  
  // Debug to see what's recalculating
  $inspect(searchTerm).with(() => 
    console.log('🔍 searchTerm changed')
  );
  
  $inspect(sortBy).with(() => 
    console.log('📊 sortBy changed')
  );
  
  $inspect(sortedItems).with(() => 
    console.log('⚠️ sortedItems recalculated (EXPENSIVE!)')
  );
  
  $inspect(filteredItems).with((type, value) => 
    console.log(`✅ filteredItems recalculated (${value.length} items)`)
  );
</script>
```

**When you type in the search box:**
```
🔍 searchTerm changed
⚠️ sortedItems recalculated (EXPENSIVE!)
✅ filteredItems recalculated (347 items)
🔍 searchTerm changed
⚠️ sortedItems recalculated (EXPENSIVE!)
✅ filteredItems recalculated (89 items)
```

**Problem identified!** `sortedItems` recalculates on every keystroke, even though `sortBy` didn't change. The expensive sort operation happens unnecessarily.

**Fix: Reverse the order**

```svelte
<script>
  // Filter first (cheap operation)
  let filteredItems = $derived(
    items.filter(item => 
      item.name.toLowerCase().includes(searchTerm.toLowerCase())
    )
  );
  
  // Then sort the filtered results (sort fewer items)
  let sortedAndFiltered = $derived(
    [...filteredItems].sort((a, b) => {
      if (sortBy === 'name') return a.name.localeCompare(b.name);
      return a.value - b.value;
    })
  );
  
  $inspect(filteredItems).with((type, value) => 
    console.log(`✅ Filtered (${value.length} items)`)
  );
  
  $inspect(sortedAndFiltered).with((type, value) => 
    console.log(`📊 Sorted (${value.length} items)`)
  );
</script>
```

**Now when you search:**
```
✅ Filtered (347 items)
📊 Sorted (347 items)
✅ Filtered (89 items)
📊 Sorted (89 items)
```

Much better! We're sorting fewer items, and the flow is clearer.

---

## 📸 $state.snapshot() - Non-Reactive Copies

Sometimes you want to capture state at a specific moment without creating reactive dependencies. Use `$state.snapshot()`:

```svelte
<script>
  let history = $state([]);
  let currentValue = $state(0);
  
  function saveToHistory() {
    // ❌ WRONG: This creates a reactive reference
    // history.push(currentValue);
    
    // ✅ CORRECT: Snapshot creates a non-reactive copy
    history = [...history, $state.snapshot(currentValue)];
  }
  
  // Debug
  $inspect(currentValue);
  $inspect(history).with((type, value) => {
    console.log(`📜 History (${type}):`, value);
  });
</script>

<input type="number" bind:value={currentValue}>
<button onclick={saveToHistory}>Save to History</button>

<h3>History:</h3>
{#each history as value, i}
  <div>Entry {i + 1}: {value}</div>
{/each}
```

### Why snapshot matters

Without `$state.snapshot()`, complex objects maintain reactive connections:

```svelte
<script>
  let user = $state({ name: 'Alice', age: 30 });
  
  let savedUsers = $state([]);
  
  function saveUser() {
    // ❌ This saves a REFERENCE, not a copy
    savedUsers = [...savedUsers, user];
    // Now changing user.name affects ALL saved entries!
  }
  
  function saveUserCorrectly() {
    // ✅ Snapshot breaks reactive connection
    savedUsers = [...savedUsers, $state.snapshot(user)];
  }
  
  $inspect(user);
  $inspect(savedUsers).with((type, value) => {
    console.table(value);
  });
</script>
```

Use `$state.snapshot()` when you need to:
- Save state history for undo/redo
- Log data at a specific point in time
- Compare before/after values
- Break reactive connections intentionally

---

## ⚠️ Common Mistakes

### 1. Forgetting $inspect() is for Development Only

```svelte
<script>
  let data = $state([]);
  
  // ❌ DON'T: Leave inspect in production
  $inspect(data);
  
  // ✅ DO: It's automatically removed, but be mindful
  // Only use for debugging, not for production logging
</script>
```

**Why it matters:** While `$inspect()` is removed in production, relying on it for actual application logic is a mistake. Use `$effect()` for production side effects.

### 2. Inspecting Non-Reactive Values

```svelte
<script>
  let staticValue = 42; // Not reactive!
  
  // ❌ This only logs once
  $inspect(staticValue);
  
  // ✅ Make it reactive if you want to track it
  let reactiveValue = $state(42);
  $inspect(reactiveValue);
</script>
```

### 3. Over-Inspecting

```svelte
<script>
  let items = $state([/* 1000 items */]);
  
  // ❌ Logging huge arrays on every keystroke
  $inspect(items); // Console flooded!
  
  // ✅ Inspect derived summary instead
  $inspect(items.length);
  $inspect(items[0]); // Just first item
</script>
```

### 4. Not Using .with() for Complex Data

```svelte
<script>
  let complexObject = $state({
    user: { name: 'Alice', metadata: {/* huge object */} },
    settings: {/* another huge object */}
  });
  
  // ❌ Hard to read in console
  $inspect(complexObject);
  
  // ✅ Format for readability
  $inspect(complexObject).with((type, value) => {
    console.log(`${type}:`, {
      userName: value.user.name,
      settingsCount: Object.keys(value.settings).length
    });
  });
</script>
```

---

## ✅ Best Practices

### 1. Use Meaningful Labels

```svelte
<script>
  let count = $state(0);
  
  // ❌ Which count is this?
  $inspect(count);
  
  // ✅ Clear context
  $inspect(count).with((type, value) => {
    console.log(`[CartCount ${type}]:`, value);
  });
</script>
```

### 2. Inspect at Different Levels

```svelte
<script>
  // Inspect inputs
  let input1 = $state('');
  $inspect(input1).with(() => console.log('🔵 Input changed'));
  
  // Inspect computed values
  let processed = $derived(input1.toUpperCase());
  $inspect(processed).with(() => console.log('🟢 Processed'));
  
  // Inspect effects
  $effect(() => {
    console.log('🟡 Effect ran with:', processed);
  });
</script>
```

### 3. Conditional Inspection

```svelte
<script>
  let debugMode = $state(true);
  let value = $state(0);
  
  // Only inspect when debugging
  $inspect(value).with((type, val) => {
    if (debugMode) {
      console.log('Debug:', type, val);
    }
  });
</script>
```

### 4. Compare Before and After

```svelte
<script>
  let items = $state([1, 2, 3]);
  
  $inspect(items).with((type, value) => {
    if (type === 'update') {
      console.log('Items changed to:', value);
      console.log('Length:', value.length);
    }
  });
  
  function addItem() {
    items = [...items, items.length + 1];
  }
</script>
```

### 5. Group Related Inspections

```svelte
<script>
  let form = $state({
    email: '',
    password: ''
  });
  
  let isValid = $derived(
    form.email.includes('@') && form.password.length >= 8
  );
  
  // Group related logs
  $inspect(form).with((type, value) => {
    console.group('Form State');
    console.log('Email:', value.email);
    console.log('Password length:', value.password.length);
    console.groupEnd();
  });
  
  $inspect(isValid).with((type, value) => {
    console.log(`✅ Valid: ${value}`);
  });
</script>
```

---

## 🧠 Mental Model: The Debug Observer

Think of `$inspect()` as placing invisible observers throughout your reactive system:

```
User Action
    ↓
State Changes ← 👁️ $inspect() watching
    ↓
Derived Updates ← 👁️ $inspect() watching
    ↓
Effects Run ← 👁️ $inspect() watching
    ↓
UI Updates
```

Each `$inspect()` is like a surveillance camera that:
- **Activates automatically** when its target changes
- **Reports to the console** with context (init/update)
- **Doesn't interfere** with the reactive flow
- **Disappears in production** (security cameras turned off)

When debugging, sprinkle `$inspect()` calls at key points in your reactive chain. They'll illuminate the invisible flow of reactivity, showing you exactly what's happening and when.

---

## 📋 Quick Reference

### Basic Inspection

```svelte
$inspect(value);                    // Auto-log on every change
$inspect(value1, value2);           // Inspect multiple values
```

### Custom Formatting

```svelte
$inspect(value).with((type, val) => {
  // type: 'init' | 'update'
  // val: current value
  console.log(`Custom: ${type}`, val);
});
```

### State Snapshot

```svelte
let copy = $state.snapshot(reactiveValue);  // Non-reactive copy
```

### Common Patterns

```svelte
// Count updates
let updateCount = 0;
$inspect(value).with(() => {
  console.log(`Updates: ${++updateCount}`);
});

// Conditional logging
$inspect(value).with((type, val) => {
  if (val > 100) console.warn('High value!');
});

// Formatted tables
$inspect(arrayOfObjects).with((type, val) => {
  console.table(val);
});

// Timing
$inspect(value).with(() => {
  console.log('Updated at:', new Date().toLocaleTimeString());
});
```

---

## 🏋️ Practice Exercise: Debug a Real Issue

Here's a component with hidden bugs. Use `$inspect()` to find and fix them:

```svelte
<script>
  let todos = $state([
    { id: 1, text: 'Learn Svelte', done: false },
    { id: 2, text: 'Build app', done: false },
    { id: 3, text: 'Deploy', done: false }
  ]);
  
  let filter = $state('all'); // 'all' | 'active' | 'done'
  
  let filteredTodos = $derived(() => {
    if (filter === 'all') return todos;
    if (filter === 'active') return todos.filter(t => !t.done);
    return todos.filter(t => t.done);
  });
  
  let activeCount = $derived(todos.filter(t => !t.done).length);
  let doneCount = $derived(todos.filter(t => t.done).length);
  
  function toggleTodo(id) {
    const todo = todos.find(t => t.id === id);
    if (todo) todo.done = !todo.done;
  }
</script>

<!-- UI here -->
```

**Your tasks:**

1. **Add `$inspect()` calls** to track state, derived values, and the toggle function
2. **Find the bug** in `filteredTodos` (hint: it's a function, not a value!)
3. **Identify the performance issue** (hint: filtering happens multiple times)
4. **Fix both issues** and verify with `$inspect()`

<details>
<summary>💡 Solution</summary>

```svelte
<script>
  let todos = $state([
    { id: 1, text: 'Learn Svelte', done: false },
    { id: 2, text: 'Build app', done: false },
    { id: 3, text: 'Deploy', done: false }
  ]);
  
  let filter = $state('all');
  
  // 🐛 BUG #1: Extra () makes this a function!
  // ❌ let filteredTodos = $derived(() => {
  // ✅ Fixed:
  let filteredTodos = $derived.by(() => {
    $inspect(filter).with((type) => console.log(`🔍 Filter ${type}`));
    
    if (filter === 'all') return todos;
    if (filter === 'active') return todos.filter(t => !t.done);
    return todos.filter(t => t.done);
  });
  
  // 🐛 BUG #2: Filtering todos twice (once for active, once for done)
  // ✅ Fixed: Calculate once
  let todoCounts = $derived({
    active: todos.filter(t => !t.done).length,
    done: todos.filter(t => t.done).length
  });
  
  $inspect(todoCounts).with((type, val) => {
    console.log(`📊 Counts: ${val.active} active, ${val.done} done`);
  });
  
  function toggleTodo(id) {
    $inspect(id).with((type, val) => console.log(`Toggle: ${val}`));
    
    // ❌ PROBLEM: Mutating doesn't trigger reactivity!
    // const todo = todos.find(t => t.id === id);
    // if (todo) todo.done = !todo.done;
    
    // ✅ Fixed: Immutable update
    todos = todos.map(t => 
      t.id === id ? { ...t, done: !t.done } : t
    );
  }
</script>
```

</details>

---

## 🚀 Next Steps

Congratulations! You now have powerful debugging tools for Svelte 5's reactive system. You can track state changes, identify performance issues, and understand complex dependency chains.

**In Article 14**, we'll dive into **$state.raw - Performance Optimization**. You'll learn when and why to skip deep reactivity, how to benchmark performance, and when to use raw state for large datasets. We'll optimize a data table handling thousands of rows!

**Preview:**
```svelte
// Coming next: When to use raw state
let largeDataset = $state.raw([/* 10,000 items */]);
// vs
let largeDataset = $state([/* 10,000 items */]); // Slower!
```

---

## 💬 Discussion Prompts

- What's the most complex dependency chain you've debugged?
- Have you found bugs using `console.log()` that `$inspect()` would have caught faster?
- What custom `.with()` formatters would be useful for your projects?

---

## 🔗 Resources

- [Svelte 5 Documentation - $inspect](https://svelte.dev/docs/svelte/$inspect)
- [Browser DevTools Console Guide](https://developer.chrome.com/docs/devtools/console/)
- [GitHub Repo: All Examples](https://github.com/your-repo/svelte-5-runes)

Happy debugging! 🐛🔍
