# $effect - Side Effects & Reactions

## The Power (and Danger) of Doing Things

You've mastered `$state` for tracking changes and `$derived` for calculating new values. But what about *doing* things when state changes? What about logging to the console, saving to localStorage, sending analytics events, or updating the document title?

These are **side effects** — actions that reach outside your reactive values to interact with the world. And in Svelte 5, there's a dedicated Rune for this: `$effect()`.

Here's the thing: side effects are where most bugs happen. They're where you accidentally create infinite loops, forget to clean up resources, or make your code unpredictable. But when used correctly, `$effect()` is incredibly powerful. It's your bridge between Svelte's reactive world and everything else: the DOM, localStorage, APIs, timers, and more.

---

## 🎯 Learning Goals

By the end of this article, you'll be able to:

- ✅ Identify what counts as a side effect
- ✅ Use `$effect()` to react to state changes
- ✅ Understand when effects run and in what order
- ✅ Write cleanup functions to prevent memory leaks
- ✅ Know when to use `$effect()` vs `$derived()`
- ✅ Avoid common anti-patterns that lead to bugs
- ✅ Implement practical patterns like auto-saving and analytics

---

## 📚 Prerequisites

Before diving in, you should be comfortable with:

- **Article 2**: `$state()` and how reactive state works
- **Article 3**: `$derived()` and computed values
- Basic JavaScript concepts: functions, closures, `setTimeout`

---

## 🧠 Core Concept: What is a Side Effect?

### The Pure vs Impure Divide

In programming, we classify functions into two categories:

**Pure functions** always return the same output for the same input and don't affect anything outside themselves:

```javascript
// Pure: just calculates and returns
function add(a, b) {
  return a + b;
}

// Pure: $derived is always pure
let total = $derived(price * quantity);
```

**Impure functions** (side effects) do something beyond returning a value:

```javascript
// Impure: modifies the DOM
function updateTitle(text) {
  document.title = text;
}

// Impure: sends data over the network
function logAnalytics(event) {
  fetch('/api/analytics', { method: 'POST', body: JSON.stringify(event) });
}

// Impure: modifies external state
function saveToLocalStorage(key, value) {
  localStorage.setItem(key, value);
}
```

### Why Side Effects Need Special Treatment

Side effects are tricky because they:
- Can fail (network errors, permission denials)
- Can have timing issues (race conditions)
- Can create dependencies that need cleanup
- Can accidentally trigger themselves (infinite loops)
- Can affect things outside Svelte's control

That's why Svelte 5 gives them their own Rune: `$effect()`.

---

## 🔧 Using $effect()

### The Basics

An effect runs whenever any reactive state it reads changes:

```svelte
<script>
  let count = $state(0);
  
  // This effect runs whenever count changes
  $effect(() => {
    console.log(`Count is now: ${count}`);
  });
</script>

<button onclick={() => count++}>
  Increment (check the console!)
</button>
```

**Key points:**
- Effects run **after** the component renders
- They run **automatically** when dependencies change
- They track dependencies just like `$derived()`

### When Effects Run

Effects have a specific lifecycle:

1. **First run**: Executes after the component mounts
2. **Subsequent runs**: Execute after any dependency changes
3. **Before re-run**: Cleanup function runs (if provided)
4. **On unmount**: Final cleanup runs

```svelte
<script>
  let name = $state('Alice');
  
  $effect(() => {
    console.log('Effect running for:', name);
    // Runs immediately after mount, then whenever name changes
  });
</script>
```

---

## 🧹 Cleanup Functions: Preventing Memory Leaks

Many side effects create resources that need cleanup: timers, event listeners, subscriptions, etc. Return a function from your effect to clean up:

```svelte
<script>
  let seconds = $state(0);
  
  $effect(() => {
    // Start a timer
    const interval = setInterval(() => {
      seconds++;
    }, 1000);
    
    // Cleanup: this runs before the next effect or on unmount
    return () => {
      clearInterval(interval);
      console.log('Timer cleaned up');
    };
  });
</script>

<p>Seconds: {seconds}</p>
```

**Why cleanup matters:**
- Without it, you'd create a new interval every time the effect re-runs
- Old intervals would keep running, creating memory leaks
- The cleanup function ensures old resources are disposed

### Cleanup Timing

```svelte
<script>
  let count = $state(0);
  
  $effect(() => {
    console.log('1. Effect running');
    
    return () => {
      console.log('2. Cleanup running');
    };
  });
  
  // When count changes:
  // 1. "2. Cleanup running" (cleanup previous effect)
  // 2. "1. Effect running" (run new effect)
</script>

<button onclick={() => count++}>Change state</button>
```

---

## 🎯 Live Example: Auto-Saving Form

Let's build a realistic example: a form that auto-saves to localStorage as the user types, with debouncing to avoid excessive saves:

```svelte
<script>
  let formData = $state({
    title: '',
    content: '',
    lastSaved: null
  });
  
  let saveStatus = $state('No changes');
  
  // Load from localStorage on mount
  $effect(() => {
    const saved = localStorage.getItem('draft');
    if (saved) {
      const parsed = JSON.parse(saved);
      formData.title = parsed.title;
      formData.content = parsed.content;
      formData.lastSaved = parsed.lastSaved;
    }
  });
  
  // Auto-save with debouncing
  $effect(() => {
    // Read the values we want to track
    const { title, content } = formData;
    
    // Don't save if both are empty (initial state)
    if (!title && !content) return;
    
    saveStatus = 'Saving...';
    
    // Debounce: wait 1 second after typing stops
    const timeout = setTimeout(() => {
      const dataToSave = {
        title,
        content,
        lastSaved: new Date().toISOString()
      };
      
      localStorage.setItem('draft', JSON.stringify(dataToSave));
      formData.lastSaved = dataToSave.lastSaved;
      saveStatus = `Saved at ${new Date(dataToSave.lastSaved).toLocaleTimeString()}`;
    }, 1000);
    
    // Cleanup: cancel the timeout if user keeps typing
    return () => {
      clearTimeout(timeout);
    };
  });
</script>

<div class="form">
  <input 
    bind:value={formData.title} 
    placeholder="Title"
  />
  
  <textarea 
    bind:value={formData.content} 
    placeholder="Start writing..."
    rows="10"
  />
  
  <div class="status">{saveStatus}</div>
</div>

<style>
  .form {
    max-width: 600px;
    margin: 2rem auto;
  }
  
  input, textarea {
    width: 100%;
    padding: 0.5rem;
    margin-bottom: 1rem;
    font-size: 1rem;
  }
  
  .status {
    color: #666;
    font-size: 0.875rem;
  }
</style>
```

**What's happening here:**

1. **First effect** loads saved data from localStorage once on mount
2. **Second effect** watches `title` and `content` for changes
3. When they change, it sets a timeout to save after 1 second
4. If the user types again, cleanup cancels the old timeout
5. Only the final timeout actually saves to localStorage

This is a real production pattern for autosave functionality!

---

## ⚠️ $effect() vs $derived(): When to Use Which

This is **critical** to understand:

### Use `$derived()` when you want to COMPUTE a value:

```svelte
<script>
  let price = $state(10);
  let quantity = $state(2);
  
  // ✅ Correct: computing a value
  let total = $derived(price * quantity);
</script>
```

### Use `$effect()` when you want to DO something:

```svelte
<script>
  let price = $state(10);
  let quantity = $state(2);
  
  // ✅ Correct: performing a side effect
  $effect(() => {
    console.log('Values changed:', price, quantity);
    document.title = `Total: $${price * quantity}`;
  });
</script>
```

### ❌ ANTI-PATTERN: Computing values in effects

```svelte
<script>
  let price = $state(10);
  let quantity = $state(2);
  let total = $state(0);
  
  // ❌ WRONG: Don't compute values in effects
  $effect(() => {
    total = price * quantity;
  });
  
  // ✅ RIGHT: Use $derived
  let total = $derived(price * quantity);
</script>
```

**Why is this wrong?**
- It's more code for no benefit
- It's less clear what `total` depends on
- It breaks the mental model (effects are for side effects)
- It can cause timing issues

**Simple rule:** If you're just assigning a value, use `$derived()`. If you're interacting with the world outside your component, use `$effect()`.

---

## 🎨 Common Effect Patterns

### Pattern 1: Logging and Analytics

```svelte
<script>
  let page = $state('home');
  
  $effect(() => {
    // Track page views
    analytics.track('page_view', { page });
    console.log('User viewed:', page);
  });
</script>
```

### Pattern 2: Document Title Updates

```svelte
<script>
  let unreadCount = $state(0);
  
  $effect(() => {
    document.title = unreadCount > 0 
      ? `(${unreadCount}) My App` 
      : 'My App';
  });
</script>
```

### Pattern 3: Syncing with External Libraries

```svelte
<script>
  import { onMount } from 'svelte';
  
  let chartData = $state([]);
  let chartInstance = $state(null);
  
  onMount(() => {
    // Initialize chart once
    chartInstance = new Chart('#myChart', { /* ... */ });
  });
  
  $effect(() => {
    // Update chart when data changes
    if (chartInstance) {
      chartInstance.setData(chartData);
    }
  });
</script>
```

### Pattern 4: Window Event Listeners

```svelte
<script>
  let windowWidth = $state(0);
  
  $effect(() => {
    function handleResize() {
      windowWidth = window.innerWidth;
    }
    
    // Set initial value
    handleResize();
    
    // Listen for changes
    window.addEventListener('resize', handleResize);
    
    // Cleanup
    return () => {
      window.removeEventListener('resize', handleResize);
    };
  });
</script>

<p>Window width: {windowWidth}px</p>
```

### Pattern 5: Focus Management

```svelte
<script>
  let showModal = $state(false);
  let modalElement;
  
  $effect(() => {
    if (showModal && modalElement) {
      modalElement.focus();
    }
  });
</script>

<button onclick={() => showModal = true}>Open Modal</button>

{#if showModal}
  <dialog bind:this={modalElement}>
    <p>Modal content</p>
    <button onclick={() => showModal = false}>Close</button>
  </dialog>
{/if}
```

---

## 🚫 Common Mistakes to Avoid

### Mistake 1: Infinite Loops

```svelte
<script>
  let count = $state(0);
  
  // ❌ INFINITE LOOP!
  $effect(() => {
    count++; // This changes count, which triggers the effect again!
  });
</script>
```

**Fix:** Only read state in effects, don't modify the same state you're reading:

```svelte
<script>
  let count = $state(0);
  let doubled = $state(0);
  
  // ✅ This is safe
  $effect(() => {
    doubled = count * 2; // Reading count, writing doubled
  });
  
  // ✅ Even better: use $derived
  let doubled = $derived(count * 2);
</script>
```

### Mistake 2: Forgetting Cleanup

```svelte
<script>
  let active = $state(true);
  
  // ❌ Memory leak!
  $effect(() => {
    const interval = setInterval(() => {
      console.log('tick');
    }, 1000);
    // Missing cleanup!
  });
  
  // ✅ With cleanup
  $effect(() => {
    const interval = setInterval(() => {
      console.log('tick');
    }, 1000);
    
    return () => clearInterval(interval);
  });
</script>
```

### Mistake 3: Using Effects for Computed Values

```svelte
<script>
  let firstName = $state('John');
  let lastName = $state('Doe');
  let fullName = $state('');
  
  // ❌ Don't do this
  $effect(() => {
    fullName = `${firstName} ${lastName}`;
  });
  
  // ✅ Use $derived
  let fullName = $derived(`${firstName} ${lastName}`);
</script>
```

### Mistake 4: Complex Logic in Effects

```svelte
<script>
  let data = $state([]);
  
  // ❌ Too much logic in one effect
  $effect(() => {
    const sorted = data.sort();
    const filtered = sorted.filter(x => x > 10);
    const mapped = filtered.map(x => x * 2);
    localStorage.setItem('processed', JSON.stringify(mapped));
  });
  
  // ✅ Separate concerns
  let sorted = $derived([...data].sort());
  let filtered = $derived(sorted.filter(x => x > 10));
  let mapped = $derived(filtered.map(x => x * 2));
  
  $effect(() => {
    localStorage.setItem('processed', JSON.stringify(mapped));
  });
</script>
```

---

## ✅ Best Practices

### 1. Keep Effects Simple

Each effect should do **one thing**. If you're doing multiple unrelated side effects, split them:

```svelte
<script>
  let user = $state(null);
  
  // ❌ Multiple unrelated side effects
  $effect(() => {
    document.title = user?.name ?? 'Guest';
    localStorage.setItem('user', JSON.stringify(user));
    analytics.identify(user?.id);
  });
  
  // ✅ Separate effects for separate concerns
  $effect(() => {
    document.title = user?.name ?? 'Guest';
  });
  
  $effect(() => {
    localStorage.setItem('user', JSON.stringify(user));
  });
  
  $effect(() => {
    if (user?.id) {
      analytics.identify(user.id);
    }
  });
</script>
```

### 2. Always Return Cleanup Functions When Needed

Ask yourself: "Does this effect create something that needs to be cleaned up?"

- Timers: `setTimeout`, `setInterval`
- Event listeners: `addEventListener`
- Subscriptions: WebSocket, EventSource, etc.
- External library instances
- Animation frames: `requestAnimationFrame`

### 3. Guard Against Uninitialized State

```svelte
<script>
  let element;
  
  $effect(() => {
    // ✅ Check element exists
    if (element) {
      element.focus();
    }
  });
</script>

<input bind:this={element} />
```

### 4. Use Early Returns for Conditional Effects

```svelte
<script>
  let enabled = $state(false);
  let count = $state(0);
  
  $effect(() => {
    // ✅ Early return if not enabled
    if (!enabled) return;
    
    console.log('Logging count:', count);
  });
</script>
```

### 5. Document Why You're Using an Effect

Future you (or your teammates) will thank you:

```svelte
<script>
  let darkMode = $state(false);
  
  // Update body class for global dark mode styles
  $effect(() => {
    document.body.classList.toggle('dark', darkMode);
    
    return () => {
      document.body.classList.remove('dark');
    };
  });
</script>
```

---

## 🧠 Mental Model: Effects as Reactions

Think of `$effect()` like a person watching your state:

```
State changes → Effect notices → Effect does something → Cleanup (if needed)
```

**The effect says:** "Whenever these values change, I need to react by doing this side effect."

Compare this to `$derived()`:

```
State changes → Derived notices → Derived recalculates → Returns new value
```

**The derived says:** "Whenever these values change, here's the new computed result."

### The Reactivity Contract

Both `$derived()` and `$effect()` follow the same reactivity rules:
- They automatically track dependencies
- They re-run when dependencies change
- They run synchronously (in order)

The difference is **what they do**:
- `$derived()` → returns a value (pure)
- `$effect()` → does something (impure)

---

## 📖 Quick Reference

```svelte
<script>
  // Basic effect
  $effect(() => {
    console.log('Effect running');
  });
  
  // Effect with dependencies
  let count = $state(0);
  $effect(() => {
    console.log('Count changed:', count);
  });
  
  // Effect with cleanup
  $effect(() => {
    const timer = setInterval(() => {}, 1000);
    
    return () => {
      clearInterval(timer);
    };
  });
  
  // Effect with guard
  $effect(() => {
    if (!someCondition) return;
    doSomething();
  });
  
  // Multiple independent effects
  $effect(() => {
    // Side effect 1
  });
  
  $effect(() => {
    // Side effect 2
  });
</script>
```

### Effect Lifecycle Checklist

- [ ] Does it modify the DOM? → Use `$effect()`
- [ ] Does it call external APIs? → Use `$effect()`
- [ ] Does it create timers/listeners? → Use `$effect()` + cleanup
- [ ] Does it just compute a value? → Use `$derived()` instead
- [ ] Does it modify state it reads? → ⚠️ Risk of infinite loop
- [ ] Does it need to run only once? → Consider `onMount()` instead

---

## 💪 Practice Exercise

Build a "Reading Timer" component that:

1. Shows how long the user has been on the page
2. Saves the total reading time to localStorage
3. Continues counting from where they left off
4. Pauses when the browser tab is hidden
5. Shows "Currently reading" or "Away" status

**Bonus challenges:**
- Format the time nicely (e.g., "2m 34s")
- Add a "Reset" button
- Track reading time per article (use a prop for article ID)

<details>
<summary>Solution</summary>

```svelte
<script>
  let { articleId = 'default' } = $props();
  
  let secondsRead = $state(0);
  let isReading = $state(true);
  
  // Load saved time on mount
  $effect(() => {
    const saved = localStorage.getItem(`reading-time-${articleId}`);
    if (saved) {
      secondsRead = parseInt(saved, 10);
    }
  });
  
  // Increment timer when reading
  $effect(() => {
    if (!isReading) return;
    
    const interval = setInterval(() => {
      secondsRead++;
    }, 1000);
    
    return () => clearInterval(interval);
  });
  
  // Save to localStorage periodically
  $effect(() => {
    const value = secondsRead; // Read the value
    
    const timeout = setTimeout(() => {
      localStorage.setItem(`reading-time-${articleId}`, value.toString());
    }, 2000);
    
    return () => clearTimeout(timeout);
  });
  
  // Detect tab visibility
  $effect(() => {
    function handleVisibilityChange() {
      isReading = !document.hidden;
    }
    
    document.addEventListener('visibilitychange', handleVisibilityChange);
    
    return () => {
      document.removeEventListener('visibilitychange', handleVisibilityChange);
    };
  });
  
  function reset() {
    secondsRead = 0;
    localStorage.removeItem(`reading-time-${articleId}`);
  }
  
  // Format time nicely
  let formattedTime = $derived(() => {
    const mins = Math.floor(secondsRead / 60);
    const secs = secondsRead % 60;
    return `${mins}m ${secs}s`;
  });
</script>

<div class="reading-timer">
  <div class="status" class:away={!isReading}>
    {isReading ? '📖 Currently reading' : '💤 Away'}
  </div>
  <div class="time">{formattedTime}</div>
  <button onclick={reset}>Reset</button>
</div>

<style>
  .reading-timer {
    padding: 1rem;
    border: 2px solid #ddd;
    border-radius: 8px;
    text-align: center;
  }
  
  .status {
    font-size: 0.875rem;
    color: #22c55e;
    margin-bottom: 0.5rem;
  }
  
  .status.away {
    color: #94a3b8;
  }
  
  .time {
    font-size: 2rem;
    font-weight: bold;
    margin-bottom: 1rem;
  }
  
  button {
    padding: 0.5rem 1rem;
    background: #f1f5f9;
    border: none;
    border-radius: 4px;
    cursor: pointer;
  }
  
  button:hover {
    background: #e2e8f0;
  }
</style>
```

</details>

---

## 🔜 Next Steps

You now understand how to perform side effects reactively with `$effect()`! You know when to use it (vs `$derived()`), how to write cleanup functions, and what patterns work in production.

In **Article 5: "$props - Component Communication"**, we'll learn how to build reusable components that accept data from parent components, handle default values, and work seamlessly with TypeScript.

**Coming up:**
- Declaring props with `$props()`
- Destructuring with defaults
- Type safety with TypeScript
- Building a flexible Button component
- The power of rest props

---

## 📚 Further Reading

- [Svelte 5 Effects Documentation](https://svelte-5-preview.vercel.app/docs/runes#$effect)
- [Understanding Side Effects in Programming](https://en.wikipedia.org/wiki/Side_effect_(computer_science))
- [When to Use useEffect (React, but concepts apply)](https://overreacted.io/a-complete-guide-to-useeffect/)

---

**Ready to level up?** Try building an auto-save text editor, implement real-time analytics tracking, or create a component that syncs with localStorage. The possibilities with `$effect()` are endless!
