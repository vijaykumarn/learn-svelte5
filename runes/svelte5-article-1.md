# Why Svelte 5 Changed Everything

## The Quiet Revolution

You've probably heard the buzz: Svelte 5 is here, and it's different. Not "new features added" different, but "fundamentally reimagined" different. If you're a Svelte 4 developer, your first reaction might be confusion, even resistance. The `$:` reactive statements you know and love? Gone. The automatic reactivity that made Svelte feel like magic? Now explicit. The framework that prided itself on "writing less code" now asks you to write *more*.

So why did the Svelte team make these changes? Was it just to chase trends, or is there something deeper happening here? After working with Svelte 5 in production for several months, I can tell you: this isn't just a new version. It's a paradigm shift that solves real problems you might not have realized you had—and it makes you a better developer in the process.

In this article, we'll explore why Svelte 5 changed everything, what problems the new Runes system solves, and why explicit reactivity isn't a step backward—it's a leap forward. Whether you're deciding if you should migrate, learning Svelte for the first time, or just curious about the future of web frameworks, this guide will give you the context you need.

---

## 🎯 Learning Goals

By the end of this article, you'll understand:

- Why Svelte 4's "magical" reactivity created hidden scaling problems
- The core philosophy behind Runes and explicit reactivity
- What specific problems Runes solve (predictability, composability, debugging)
- The mental shift required to work effectively with Svelte 5
- When to use Svelte 5 versus staying on Svelte 4
- How Runes make your code more maintainable in the long run

---

## 📚 Prerequisites

This article assumes you:

- Have basic JavaScript knowledge (variables, functions, objects)
- Understand what "reactivity" means in a UI framework context
- Optionally: Have some experience with Svelte 4 (helpful but not required)

If you're completely new to Svelte, that's actually an advantage—you won't have to "unlearn" anything!

---

## 🧠 Core Concept: The Problem with Magic

### Svelte 4: Beautiful, but Brittle

Let's start with what made Svelte 4 special. Here's a simple counter in Svelte 4:

```svelte
<script>
  let count = 0;
  $: doubled = count * 2;
  
  function increment() {
    count += 1;
  }
</script>

<button on:click={increment}>
  Count: {count} (doubled: {doubled})
</button>
```

This feels magical. You just declare a variable, and it's reactive. You use `$:` to create computed values, and they update automatically. No boilerplate, no ceremony, just straightforward code.

But here's what's happening under the hood: the Svelte compiler is analyzing your code at build time, figuring out what's reactive, and injecting update logic. It's incredibly clever—until it isn't.

### The Three Hidden Problems

**Problem 1: Unpredictable Reactivity**

Consider this seemingly innocent code:

```svelte
<script>
  let items = [1, 2, 3];
  
  function addItem() {
    items.push(4); // Doesn't trigger updates!
    items = items;  // This does... but why?
  }
</script>
```

In Svelte 4, mutating arrays and objects doesn't trigger reactivity—you need reassignment. This isn't a bug; it's how the compiler tracks changes. But it's confusing for developers and leads to subtle bugs.

**Problem 2: Non-Composable Reactivity**

Try to extract reactive logic into a function:

```svelte
<script>
  let firstName = 'John';
  let lastName = 'Doe';
  
  // This doesn't work as you'd expect
  function getFullName() {
    $: fullName = `${firstName} ${lastName}`; // ❌ Can't use $: in functions
    return fullName;
  }
</script>
```

Reactive statements (`$:`) only work at the component's top level. You can't extract them into reusable functions, which severely limits code reuse and composition.

**Problem 3: Difficult Debugging**

When reactivity doesn't work as expected, debugging is painful:

```svelte
<script>
  let user = { name: 'Alice', age: 30 };
  $: greeting = `Hello, ${user.name}`;
  
  function updateUser() {
    user.name = 'Bob'; // Doesn't update greeting
    user = user;       // Now it does... but this feels wrong
  }
</script>
```

Why doesn't `user.name = 'Bob'` trigger the update? The compiler tracks assignments to `user`, not properties of `user`. Understanding when and why reactivity triggers requires mental modeling of the compiler's behavior.

### The Scaling Problem

These issues might seem minor in small apps. But as your application grows:

- Teams struggle with "reactive vs non-reactive" patterns
- Code reviews become debates about reactivity edge cases
- Bugs emerge from misunderstanding reactivity rules
- Reusable logic is hard to share across components
- Testing reactive code requires component-level testing

The magic that made Svelte 4 delightful for small projects became a maintenance burden for large ones.

---

## 🚀 Enter Runes: Explicit Reactivity

Svelte 5 introduces **Runes**—explicit primitives for reactivity. Here's that same counter in Svelte 5:

```svelte
<script>
  let count = $state(0);
  let doubled = $derived(count * 2);
  
  function increment() {
    count += 1;
  }
</script>

<button onclick={increment}>
  Count: {count} (doubled: {doubled})
</button>
```

At first glance, this looks like *more* code. You're explicitly marking `count` as reactive state with `$state()`, and `doubled` as a derived value with `$derived()`. But this explicitness solves all three problems:

### Solution 1: Predictable Reactivity

```svelte
<script>
  let items = $state([1, 2, 3]);
  
  function addItem() {
    items.push(4); // ✅ Works! Deep reactivity by default
  }
</script>
```

With `$state()`, you get deep reactivity for arrays and objects. Mutations work as you'd expect. No more `items = items` workarounds.

### Solution 2: Composable Reactivity

```svelte
<script>
  // Extract to a reusable function
  function useFullName(firstName, lastName) {
    return $derived(`${firstName} ${lastName}`);
  }
  
  let firstName = $state('John');
  let lastName = $state('Doe');
  let fullName = useFullName(firstName, lastName);
</script>
```

Runes work anywhere in your code—functions, classes, imported modules. You can build reusable reactive logic (often called "composables" or "custom Runes") and share it across your entire application.

### Solution 3: Debuggable Reactivity

```svelte
<script>
  let user = $state({ name: 'Alice', age: 30 });
  let greeting = $derived(`Hello, ${user.name}`);
  
  function updateUser() {
    user.name = 'Bob'; // ✅ Updates greeting immediately
  }
  
  $inspect(user, greeting); // Debug reactive values
</script>
```

Because reactivity is explicit, it's clear what's reactive and what isn't. The new `$inspect()` Rune lets you debug reactive dependencies in real-time. No more guessing why something didn't update.

---

## 💡 Live Example: Comparing Both Approaches

Let's build a practical example to see the difference. We'll create a todo list with filtering—a common pattern that exposes Svelte 4's limitations.

### Svelte 4 Approach

```svelte
<script>
  let todos = [
    { id: 1, text: 'Learn Svelte 5', done: false },
    { id: 2, text: 'Build an app', done: false },
    { id: 3, text: 'Ship it', done: false }
  ];
  
  let filter = 'all'; // 'all', 'active', 'done'
  
  $: filteredTodos = filter === 'all' 
    ? todos
    : filter === 'active'
    ? todos.filter(t => !t.done)
    : todos.filter(t => t.done);
  
  $: activeCount = todos.filter(t => !t.done).length;
  
  function addTodo(text) {
    todos = [...todos, { 
      id: Date.now(), 
      text, 
      done: false 
    }];
  }
  
  function toggleTodo(id) {
    todos = todos.map(t => 
      t.id === id ? { ...t, done: !t.done } : t
    );
  }
</script>
```

Issues with this code:

- Can't extract filtering logic for reuse
- Array spread operators everywhere (`...todos`)
- Reactive statements are component-scoped only
- Testing requires mounting the entire component

### Svelte 5 Approach

```svelte
<script>
  let todos = $state([
    { id: 1, text: 'Learn Svelte 5', done: false },
    { id: 2, text: 'Build an app', done: false },
    { id: 3, text: 'Ship it', done: false }
  ]);
  
  let filter = $state('all');
  
  let filteredTodos = $derived(
    filter === 'all' 
      ? todos
      : filter === 'active'
      ? todos.filter(t => !t.done)
      : todos.filter(t => t.done)
  );
  
  let activeCount = $derived(
    todos.filter(t => !t.done).length
  );
  
  function addTodo(text) {
    todos.push({ 
      id: Date.now(), 
      text, 
      done: false 
    });
  }
  
  function toggleTodo(id) {
    const todo = todos.find(t => t.id === id);
    if (todo) todo.done = !todo.done;
  }
</script>
```

Benefits of this code:

- Direct mutations work (`todos.push()`, `todo.done = !todo.done`)
- Logic can be extracted into custom Runes
- Clear distinction between state and derived values
- Easier to unit test individual functions

Even better, we can extract reusable logic:

```svelte
<script>
  // useTodoList.js - reusable across components
  export function useTodoList(initialTodos = []) {
    let todos = $state(initialTodos);
    let filter = $state('all');
    
    let filteredTodos = $derived(
      filter === 'all' 
        ? todos
        : filter === 'active'
        ? todos.filter(t => !t.done)
        : todos.filter(t => t.done)
    );
    
    let activeCount = $derived(
      todos.filter(t => !t.done).length
    );
    
    return {
      todos,
      filter,
      filteredTodos,
      activeCount,
      addTodo(text) {
        todos.push({ id: Date.now(), text, done: false });
      },
      toggleTodo(id) {
        const todo = todos.find(t => t.id === id);
        if (todo) todo.done = !todo.done;
      }
    };
  }
  
  // In your component
  const todoList = useTodoList([
    { id: 1, text: 'Learn Svelte 5', done: false }
  ]);
</script>

<div>
  <button onclick={() => todoList.filter = 'all'}>All</button>
  <button onclick={() => todoList.filter = 'active'}>Active</button>
  <button onclick={() => todoList.filter = 'done'}>Done</button>
  
  <p>{todoList.activeCount} items left</p>
  
  {#each todoList.filteredTodos as todo}
    <div>
      <input 
        type="checkbox" 
        checked={todo.done}
        onchange={() => todoList.toggleTodo(todo.id)}
      />
      {todo.text}
    </div>
  {/each}
</div>
```

This pattern—creating custom Runes—is impossible in Svelte 4 but natural in Svelte 5. You can now build libraries of reactive logic that work anywhere.

---

## ⚠️ Common Mistakes When Starting with Svelte 5

### Mistake 1: Forgetting `$state()` for Reactivity

```svelte
<script>
  let count = 0; // ❌ Not reactive in Svelte 5
  
  function increment() {
    count += 1; // Won't trigger UI update
  }
</script>
```

**Fix**: Always use `$state()` for reactive values:

```svelte
<script>
  let count = $state(0); // ✅ Reactive
</script>
```

### Mistake 2: Using `$derived()` for Side Effects

```svelte
<script>
  let count = $state(0);
  
  // ❌ Don't use $derived for side effects
  let logger = $derived(() => {
    console.log('Count:', count);
    return count;
  });
</script>
```

**Fix**: Use `$effect()` for side effects (covered in Article 4):

```svelte
<script>
  let count = $state(0);
  
  // ✅ Use $effect for side effects
  $effect(() => {
    console.log('Count:', count);
  });
</script>
```

### Mistake 3: Mixing Svelte 4 and Svelte 5 Syntax

```svelte
<script>
  let count = $state(0);
  $: doubled = count * 2; // ❌ Don't mix $: with Runes
</script>
```

**Fix**: Use Runes consistently:

```svelte
<script>
  let count = $state(0);
  let doubled = $derived(count * 2); // ✅ Use $derived
</script>
```

### Mistake 4: Over-Nesting `$state()`

```svelte
<script>
  // ❌ Don't nest $state calls
  let user = $state({
    name: $state('Alice'),
    age: $state(30)
  });
</script>
```

**Fix**: Only the top level needs `$state()`:

```svelte
<script>
  // ✅ Deep reactivity by default
  let user = $state({
    name: 'Alice',
    age: 30
  });
  
  // This works:
  user.name = 'Bob';
</script>
```

---

## ✅ Best Practices for Svelte 5

### 1. Start with `$state()` for All Reactive Data

Make it a habit: if a value changes and the UI should update, wrap it in `$state()`.

```svelte
<script>
  let username = $state('');
  let isLoading = $state(false);
  let items = $state([]);
  let settings = $state({ theme: 'dark', lang: 'en' });
</script>
```

### 2. Use `$derived()` for Computed Values

If a value is calculated from other values and should update automatically, use `$derived()`.

```svelte
<script>
  let firstName = $state('John');
  let lastName = $state('Doe');
  let fullName = $derived(`${firstName} ${lastName}`);
  
  let items = $state([1, 2, 3, 4, 5]);
  let total = $derived(items.reduce((a, b) => a + b, 0));
</script>
```

### 3. Extract Reusable Logic into Functions

This is the superpower of Svelte 5—don't keep everything in components.

```svelte
<script>
  // useCounter.js
  export function useCounter(initial = 0) {
    let count = $state(initial);
    let doubled = $derived(count * 2);
    
    return {
      count,
      doubled,
      increment: () => count++,
      decrement: () => count--,
      reset: () => count = initial
    };
  }
  
  // In your component
  const counter = useCounter(10);
</script>

<button onclick={counter.decrement}>-</button>
<span>{counter.count} (2x = {counter.doubled})</span>
<button onclick={counter.increment}>+</button>
```

### 4. Embrace Direct Mutations

Unlike Svelte 4, you can directly mutate arrays and objects:

```svelte
<script>
  let items = $state([1, 2, 3]);
  let user = $state({ name: 'Alice', age: 30 });
  
  function update() {
    items.push(4);        // ✅ Works
    items[0] = 10;        // ✅ Works
    user.age = 31;        // ✅ Works
    user.name = 'Bob';    // ✅ Works
  }
</script>
```

### 5. Be Explicit About Non-Reactive Values

If something shouldn't be reactive, don't wrap it in `$state()`:

```svelte
<script>
  const API_URL = 'https://api.example.com'; // Not reactive, doesn't need $state
  let currentUser = $state(null); // Reactive
</script>
```

---

## 🎨 Mental Model: Reactive Containers

Here's how to think about Svelte 5 Runes:

**Svelte 4 Mental Model**: "The compiler figures out what's reactive"
- Implicit, magical, sometimes unpredictable
- Works great until it doesn't
- Hard to reason about in complex scenarios

**Svelte 5 Mental Model**: "I explicitly mark reactive containers"
- `$state()` = "This is a reactive container"
- `$derived()` = "This automatically recomputes when dependencies change"
- `$effect()` = "Run this side effect when dependencies change"
- Everything else is just regular JavaScript

Think of `$state()` as creating a special "box" that Svelte watches. When you change what's in the box, Svelte updates the UI. When you read from the box inside `$derived()` or `$effect()`, Svelte creates a dependency.

```
Regular Variable:     let x = 1;           [x = 1]
Reactive State:       let x = $state(1);   [$state(x = 1)]  ← Svelte watches this
```

This mental model makes it easy to understand what's happening and why.

---

## 📋 Quick Reference: Svelte 4 vs Svelte 5

| Concept | Svelte 4 | Svelte 5 |
|---------|----------|----------|
| **Reactive variable** | `let count = 0` | `let count = $state(0)` |
| **Computed value** | `$: doubled = count * 2` | `let doubled = $derived(count * 2)` |
| **Side effect** | `$: { console.log(count) }` | `$effect(() => console.log(count))` |
| **Array push** | `items = [...items, item]` | `items.push(item)` |
| **Object update** | `user = { ...user, age: 31 }` | `user.age = 31` |
| **Props** | `export let name` | `let { name } = $props()` |
| **Two-way binding** | `export let value` | `let { value = $bindable() } = $props()` |
| **Reusable logic** | Not really possible | Custom Runes (functions with Runes) |

---

## 🏋️ Practice Exercise: Your First Runes Component

Build a simple "profile editor" to practice the concepts:

**Requirements**:
1. Create reactive state for `firstName`, `lastName`, and `age`
2. Create a derived `fullName` that combines first and last names
3. Create a derived `isAdult` that checks if age >= 18
4. Add buttons to increment/decrement age
5. Add a reset button that restores original values

**Starter Template**:

```svelte
<script>
  // Your code here
  let firstName = $state('');
  let lastName = $state('');
  let age = $state(0);
  
  // Add derived values
  // Add functions
</script>

<div>
  <input bind:value={firstName} placeholder="First name" />
  <input bind:value={lastName} placeholder="Last name" />
  
  <div>
    <button>-</button>
    Age: {age}
    <button>+</button>
  </div>
  
  <p>Full name: {/* Show fullName */}</p>
  <p>Status: {/* Show isAdult ? 'Adult' : 'Minor' */}</p>
  
  <button>Reset</button>
</div>
```

**Solution** (try it yourself first!):

```svelte
<script>
  const INITIAL_FIRST = 'John';
  const INITIAL_LAST = 'Doe';
  const INITIAL_AGE = 25;
  
  let firstName = $state(INITIAL_FIRST);
  let lastName = $state(INITIAL_LAST);
  let age = $state(INITIAL_AGE);
  
  let fullName = $derived(`${firstName} ${lastName}`);
  let isAdult = $derived(age >= 18);
  
  function incrementAge() {
    age++;
  }
  
  function decrementAge() {
    if (age > 0) age--;
  }
  
  function reset() {
    firstName = INITIAL_FIRST;
    lastName = INITIAL_LAST;
    age = INITIAL_AGE;
  }
</script>

<div>
  <input bind:value={firstName} placeholder="First name" />
  <input bind:value={lastName} placeholder="Last name" />
  
  <div>
    <button onclick={decrementAge}>-</button>
    Age: {age}
    <button onclick={incrementAge}>+</button>
  </div>
  
  <p>Full name: {fullName}</p>
  <p>Status: {isAdult ? 'Adult' : 'Minor'}</p>
  
  <button onclick={reset}>Reset</button>
</div>
```

---

## 🤔 When to Use Svelte 5 vs Staying on Svelte 4

### Choose Svelte 5 if:

✅ **Starting a new project** - No migration needed, learn modern patterns from day one

✅ **You need reusable reactive logic** - Custom Runes unlock composition patterns impossible in Svelte 4

✅ **Your app is growing complex** - Explicit reactivity scales better for large teams and codebases

✅ **You're hitting Svelte 4's limitations** - Array/object mutation workarounds, can't extract reactive logic

✅ **You want better TypeScript support** - Runes have clearer types and better IDE integration

### Stay on Svelte 4 if:

⚠️ **Existing app with no pain points** - If it works and isn't growing significantly, migration might not be worth it

⚠️ **Very simple projects** - Svelte 4's terseness might still be advantageous for tiny apps

⚠️ **Team not ready for the shift** - The mental model change requires buy-in from everyone

⚠️ **Third-party dependencies** - Some Svelte 4 component libraries haven't updated yet

That said, **Svelte 5 is the future**. Even if you stay on Svelte 4 now, learning Svelte 5 prepares you for inevitable migration.

---

## 🎯 Key Takeaways

1. **Svelte 5 isn't just a new API**—it's a new philosophy. Explicit reactivity trades some terseness for predictability, composability, and scalability.

2. **The "magic" of Svelte 4 had real costs**: unpredictable behavior, limited composition, difficult debugging. Runes solve these problems.

3. **Runes are JavaScript functions**: `$state()`, `$derived()`, `$effect()` work anywhere—components, functions, classes, modules. This enables true code reuse.

4. **Direct mutations now work**: Arrays and objects have deep reactivity by default. No more spreading operators everywhere.

5. **The learning curve is worth it**: The initial "this feels like more code" reaction fades quickly, replaced by confidence and productivity.

6. **Start simple, scale up**: Begin with `$state()` and `$derived()`. As you grow comfortable, explore custom Runes and advanced patterns.

---

## 🚀 Next Steps

Now that you understand *why* Svelte 5 changed and *what* Runes solve, it's time to get hands-on.

**In Article 2: "$state - Your First Rune"**, we'll dive deep into reactive state:
- Primitives vs objects vs arrays
- `$state()` vs `$state.raw()` - when to use each
- Common beginner mistakes and how to avoid them
- Building a practical counter with multiple state variables
- Mental models for thinking about reactive state

The best way to learn Runes is to start using them. See you in the next article!

---

## 💬 Discussion Questions

Before moving on, consider:

1. What aspects of Svelte 4's reactivity have confused you in the past?
2. Can you think of reactive logic in your current projects that would benefit from extraction into custom Runes?
3. Are you more excited or concerned about the explicit nature of Runes?

Share your thoughts in the comments, and let's learn together!

---

**Series Progress**: Article 1 of 35 complete ✅

**Next**: [Article 2: $state - Your First Rune →](#)