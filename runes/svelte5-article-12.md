# Article 12: untrack() & $effect.tracking() - Controlling Dependencies

## Why This Matters

You've built a search feature. Every keystroke triggers an API call. You add analytics logging to track searches. Now you have a problem: the analytics code is causing extra re-renders because it reads reactive state. You need to read that state *without* creating a dependency. Or consider this: you're building a form autosave feature that triggers on any field change, but you don't want the "last saved" timestamp to trigger *another* save. These are the moments when you need surgical control over Svelte's reactivity.

Most of the time, Svelte 5's automatic dependency tracking is brilliant. But sometimes you need to tell Svelte: "I need to read this value, but don't watch it." That's where `untrack()` comes in. It's the escape hatch that lets you opt out of reactivity exactly when and where you need to. And `$effect.tracking()` is your diagnostic tool to understand when you're in a reactive context.

Think of automatic dependency tracking like a surveillance system that watches everything. Usually that's great—you want your UI to update when data changes. But sometimes you need to sneak past the cameras. `untrack()` is your invisibility cloak, and `$effect.tracking()` tells you whether the cameras are watching.

## Learning Goals

By the end of this article, you'll be able to:

- ✅ Use `untrack()` to read state without creating dependencies
- ✅ Understand when and why to break reactive dependencies
- ✅ Use `$effect.tracking()` to detect reactive contexts
- ✅ Prevent infinite loops caused by circular dependencies
- ✅ Optimize performance by limiting unnecessary reactivity
- ✅ Build advanced patterns like debounced search with proper dependency control
- ✅ Debug complex reactivity issues using tracking tools

## Prerequisites

Before diving in, you should be comfortable with:

- `$state()` for reactive variables (Article 2)
- `$derived()` for computed values (Article 3)
- `$effect()` for side effects (Article 4)
- Understanding when effects run and what triggers them
- Basic async/await patterns in JavaScript

## Core Concept: The Dependency Problem

### How Automatic Tracking Works

When you read reactive state inside an effect or derived value, Svelte automatically adds that state as a dependency:

```svelte
<script>
let count = $state(0);
let doubled = $state(0);

// This effect depends on 'count'
$effect(() => {
  console.log(count); // Reading count creates a dependency
  doubled = count * 2;
});
</script>
```

Every time `count` changes, the effect runs. This is usually exactly what you want. But what if you need more control?

### The Problem: Unwanted Dependencies

Consider this autosave feature:

```svelte
<script>
let formData = $state({ name: '', email: '' });
let lastSaved = $state(null);

$effect(() => {
  // Save the form
  saveToServer(formData);
  
  // Update the timestamp
  lastSaved = new Date(); // ⚠️ PROBLEM!
});
</script>
```

**This creates an infinite loop!** Here's why:

1. `formData` changes
2. Effect runs, saves data
3. Effect updates `lastSaved`
4. But wait—the effect is reading `lastSaved` (implicitly, by being in the same scope)
5. Actually, it's worse: writing to state inside an effect can cause issues
6. You need to read `lastSaved` elsewhere, which creates a dependency

Let's fix the actual problematic pattern:

```svelte
<script>
let formData = $state({ name: '', email: '' });
let lastSaved = $state(null);
let saveCount = $state(0);

$effect(() => {
  // We want this to run only when formData changes,
  // not when saveCount changes
  if (saveCount > 0) {
    console.log(`Saved ${saveCount} times`);
  }
  
  saveToServer(formData);
  saveCount++; // This creates an unwanted dependency!
});
</script>
```

### Enter untrack()

`untrack()` lets you read reactive state without creating a dependency:

```svelte
<script>
import { untrack } from 'svelte';

let formData = $state({ name: '', email: '' });
let saveCount = $state(0);

$effect(() => {
  // Only depends on formData, not saveCount
  const currentCount = untrack(() => saveCount);
  
  if (currentCount > 0) {
    console.log(`Saved ${currentCount} times`);
  }
  
  saveToServer(formData);
  saveCount++;
});
</script>
```

Now the effect **only** re-runs when `formData` changes, even though it reads `saveCount`.

## Live Example: Debounced Search

Let's build a real-world example: a search input with debounced API calls and analytics.

```svelte
<script>
import { untrack } from 'svelte';

let searchQuery = $state('');
let searchResults = $state([]);
let isSearching = $state(false);
let searchCount = $state(0);
let lastSearchTime = $state(null);

// Debounce helper
function debounce(fn, delay) {
  let timeoutId;
  return (...args) => {
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => fn(...args), delay);
  };
}

// The actual search function
async function performSearch(query) {
  if (!query.trim()) {
    searchResults = [];
    return;
  }
  
  isSearching = true;
  
  try {
    // Simulate API call
    await new Promise(resolve => setTimeout(resolve, 500));
    searchResults = [
      `Result 1 for "${query}"`,
      `Result 2 for "${query}"`,
      `Result 3 for "${query}"`
    ];
    
    // Update metadata WITHOUT creating dependencies
    untrack(() => {
      searchCount++;
      lastSearchTime = new Date();
    });
    
  } finally {
    isSearching = false;
  }
}

const debouncedSearch = debounce(performSearch, 300);

// Effect that triggers search
$effect(() => {
  // This depends ONLY on searchQuery
  debouncedSearch(searchQuery);
});

// Analytics effect - tracks searches without affecting search logic
$effect(() => {
  // Read searchQuery to create dependency
  const query = searchQuery;
  
  // But read metadata WITHOUT creating dependencies
  const count = untrack(() => searchCount);
  const lastTime = untrack(() => lastSearchTime);
  
  if (query && count > 0) {
    console.log('Analytics:', {
      query,
      searchNumber: count,
      timestamp: lastTime
    });
  }
});
</script>

<div class="search-container">
  <input
    type="text"
    bind:value={searchQuery}
    placeholder="Search..."
  />
  
  {#if isSearching}
    <p class="searching">Searching...</p>
  {/if}
  
  <div class="results">
    {#each searchResults as result}
      <div class="result">{result}</div>
    {/each}
  </div>
  
  <div class="metadata">
    <p>Total searches: {searchCount}</p>
    {#if lastSearchTime}
      <p>Last search: {lastSearchTime.toLocaleTimeString()}</p>
    {/if}
  </div>
</div>

<style>
.search-container {
  max-width: 500px;
  margin: 2rem auto;
}

input {
  width: 100%;
  padding: 0.75rem;
  font-size: 1rem;
  border: 2px solid #ddd;
  border-radius: 4px;
}

.searching {
  color: #666;
  font-style: italic;
  margin: 0.5rem 0;
}

.results {
  margin-top: 1rem;
}

.result {
  padding: 0.75rem;
  background: #f5f5f5;
  margin-bottom: 0.5rem;
  border-radius: 4px;
}

.metadata {
  margin-top: 2rem;
  padding: 1rem;
  background: #e3f2fd;
  border-radius: 4px;
  font-size: 0.9rem;
  color: #1976d2;
}
</style>
```

### What's Happening Here?

1. **Search Effect**: Depends only on `searchQuery`, triggering debounced searches
2. **Metadata Updates**: Inside `performSearch`, we update `searchCount` and `lastSearchTime` using `untrack()` so they don't trigger re-renders
3. **Analytics Effect**: Depends on `searchQuery`, but reads metadata using `untrack()` to avoid circular dependencies

Without `untrack()`, updating `searchCount` inside the search would create a dependency loop, causing the effect to run continuously.

## $effect.tracking(): Your Diagnostic Tool

`$effect.tracking()` returns `true` if you're currently inside a reactive context (effect or derived), `false` otherwise:

```svelte
<script>
import { untrack } from 'svelte';

let count = $state(0);

function logCount() {
  console.log('Am I being tracked?', $effect.tracking());
  console.log('Count:', count);
}

$effect(() => {
  console.log('Inside effect, tracking?', $effect.tracking()); // true
  logCount(); // Will show tracking = true
});

// Called outside reactive context
logCount(); // Will show tracking = false

untrack(() => {
  console.log('Inside untrack, tracking?', $effect.tracking()); // false
});
</script>
```

### When to Use $effect.tracking()

This is mostly useful for:

1. **Library code** that needs to behave differently in reactive vs non-reactive contexts
2. **Debugging** to understand where dependencies are being created
3. **Conditional reactivity** in advanced patterns

Example: A logging utility that adapts its behavior:

```svelte
<script>
function smartLog(message, value) {
  if ($effect.tracking()) {
    // In reactive context: just log without creating dependency
    console.log(message, untrack(() => value));
  } else {
    // Outside reactive context: normal log
    console.log(message, value);
  }
}

let count = $state(0);

$effect(() => {
  // This won't create a dependency on count
  smartLog('Count changed:', count);
});
</script>
```

## Common Mistakes

### ❌ Mistake 1: Untracking Too Much

```svelte
<script>
let name = $state('Alice');
let age = $state(30);

$effect(() => {
  // BAD: This effect will never run!
  untrack(() => {
    console.log(`${name} is ${age}`);
  });
});
</script>
```

**Fix**: Only untrack what you don't want to depend on:

```svelte
<script>
$effect(() => {
  // Depends on name, not age
  const currentAge = untrack(() => age);
  console.log(`${name} is ${currentAge}`);
});
</script>
```

### ❌ Mistake 2: Untracking the Wrong Thing

```svelte
<script>
let items = $state([]);
let filter = $state('');

let filteredItems = $derived(
  // BAD: Untracking filter defeats the purpose!
  items.filter(item => 
    item.includes(untrack(() => filter))
  )
);
</script>
```

**Fix**: Don't untrack unless you have a specific reason:

```svelte
<script>
let filteredItems = $derived(
  items.filter(item => item.includes(filter))
);
</script>
```

### ❌ Mistake 3: Using untrack() Where $derived.by() Is Better

```svelte
<script>
let count = $state(0);
let doubled = $state(0);

$effect(() => {
  // BAD: This should be a derived value!
  doubled = untrack(() => count * 2);
});
</script>
```

**Fix**: Use the right tool:

```svelte
<script>
let count = $state(0);
let doubled = $derived(count * 2);
</script>
```

### ❌ Mistake 4: Forgetting untrack() Returns a Value

```svelte
<script>
let count = $state(5);

$effect(() => {
  // BAD: untrack returns the function's result!
  untrack(() => {
    console.log(count);
  });
  // Nothing is done with the return value
});
</script>
```

**Fix**: Remember `untrack()` returns what the function returns:

```svelte
<script>
$effect(() => {
  const value = untrack(() => count);
  console.log(value);
});
</script>
```

## Best Practices

### ✅ 1. Use untrack() for Metadata and Analytics

```svelte
<script>
let formData = $state({ username: '', email: '' });
let editCount = $state(0);
let lastEdited = $state(null);

$effect(() => {
  // Depends on formData
  validateAndSave(formData);
  
  // Update tracking info without creating dependencies
  untrack(() => {
    editCount++;
    lastEdited = new Date();
  });
});
</script>
```

### ✅ 2. Break Circular Dependencies Explicitly

```svelte
<script>
let value = $state(10);
let history = $state([]);

$effect(() => {
  // Record value in history
  const currentHistory = untrack(() => history);
  history = [...currentHistory, value];
  
  // Limit history size
  if (history.length > 10) {
    history = history.slice(-10);
  }
});
</script>
```

### ✅ 3. Use untrack() for Performance Optimization

```svelte
<script>
let data = $state([/* large array */]);
let processedData = $state([]);
let processingTime = $state(0);

$effect(() => {
  const start = performance.now();
  
  // Process data (creates dependency)
  processedData = expensiveOperation(data);
  
  // Record timing (no dependency needed)
  untrack(() => {
    processingTime = performance.now() - start;
  });
});
</script>
```

### ✅ 4. Document Why You're Using untrack()

```svelte
<script>
$effect(() => {
  // Track changes to user preferences
  savePreferences(userPrefs);
  
  // Update last sync time WITHOUT triggering another save
  // (untrack prevents circular dependency)
  untrack(() => {
    lastSyncTime = new Date();
  });
});
</script>
```

## Mental Model: The Surveillance Metaphor

Think of Svelte's reactivity as a surveillance system:

- **Default behavior**: Every camera (effect/derived) watches everything you touch (read)
- **untrack()**: A special zone where cameras can't see you
- **$effect.tracking()**: A sign telling you "You are being watched"

```
Normal code:
┌─────────────────┐
│  Read state     │──┐
│  (Tracked!)     │  │ Creates
└─────────────────┘  │ dependency
                     ▼
              ┌──────────┐
              │  Effect  │
              │  Re-runs │
              └──────────┘

With untrack():
┌─────────────────┐
│  untrack(() =>  │
│    Read state   │  ✗ No dependency created
│  )              │
└─────────────────┘
              ┌──────────┐
              │  Effect  │
              │  Doesn't │
              │  Re-run  │
              └──────────┘
```

### When to Use Each Tool

**Use `untrack()` when:**
- Reading state for logging/analytics without affecting reactivity
- Updating metadata that shouldn't trigger effects
- Breaking circular dependencies intentionally
- Optimizing performance by limiting dependencies
- Implementing undo/redo or history tracking

**Use `$effect.tracking()` when:**
- Building libraries that work in both reactive and non-reactive contexts
- Debugging unexpected dependencies
- Conditionally creating dependencies
- You need to know if you're inside an effect

**Don't use either when:**
- Normal reactivity works fine (most of the time!)
- You actually want the dependency
- You should be using `$derived()` instead

## Quick Reference

### untrack() Signature

```typescript
function untrack<T>(fn: () => T): T
```

Runs the function and returns its result, but state reads inside don't create dependencies.

### $effect.tracking() Signature

```typescript
function tracking(): boolean
```

Returns `true` if currently inside a reactive context, `false` otherwise.

### Common Patterns

```svelte
<script>
// Pattern 1: Read without dependency
const value = untrack(() => someState);

// Pattern 2: Multiple reads
const [a, b, c] = untrack(() => [state1, state2, state3]);

// Pattern 3: Conditional tracking
if ($effect.tracking()) {
  // Different behavior in reactive context
}

// Pattern 4: Nested untrack (rare)
untrack(() => {
  const x = state1; // Not tracked
  const y = state2; // Not tracked
  return x + y;
});

// Pattern 5: Update state without dependencies
untrack(() => {
  metadata.lastModified = new Date();
  metadata.editCount++;
});
</script>
```

## Practice Exercise: Build a Smart Form

Create a form with these requirements:

1. **Form fields**: name, email, message
2. **Auto-save**: Save to localStorage after 1 second of inactivity
3. **Character counter**: Show characters remaining (max 500 for message)
4. **Save indicator**: Show "Saved" or "Saving..." status
5. **Edit tracking**: Count number of edits, but don't let this trigger saves

**Constraints:**
- Use `untrack()` to prevent edit count from triggering auto-save
- Use `untrack()` for updating "last saved" timestamp
- Debounce the auto-save
- No infinite loops!

### Starter Code

```svelte
<script>
import { untrack } from 'svelte';

let formData = $state({
  name: '',
  email: '',
  message: ''
});

let saveStatus = $state('idle'); // 'idle' | 'saving' | 'saved'
let editCount = $state(0);
let lastSaved = $state(null);

// TODO: Implement auto-save with proper untrack usage
// TODO: Track edits without causing saves
// TODO: Update lastSaved without creating dependencies

</script>

<form>
  <input bind:value={formData.name} placeholder="Name" />
  <input bind:value={formData.email} placeholder="Email" />
  <textarea bind:value={formData.message} placeholder="Message (max 500)" />
  
  <div class="meta">
    <span>Status: {saveStatus}</span>
    <span>Edits: {editCount}</span>
    {#if lastSaved}
      <span>Last saved: {lastSaved.toLocaleTimeString()}</span>
    {/if}
  </div>
</form>
```

### Solution Hints

<details>
<summary>Click to see hints</summary>

1. Create an effect that depends on `formData`
2. Inside that effect, increment `editCount` using `untrack()`
3. Debounce the actual save operation
4. After saving, update `lastSaved` using `untrack()`
5. Use a separate effect for character count (or use `$derived()`)

</details>

## Next Steps

You now have surgical control over Svelte 5's reactivity! You can read state without creating dependencies and understand when you're in a reactive context. This is powerful, but use it sparingly—automatic tracking is usually what you want.

**In Article 13**, we'll explore `$inspect()` and learn how to debug reactive code effectively. You'll discover how to track down why components re-render, visualize dependency chains, and use `$state.snapshot()` to create non-reactive copies of state. If you've ever wondered "why did this effect run?", the next article is for you.

---

**Key Takeaway**: `untrack()` is your escape hatch from automatic dependency tracking. Use it when you need to read state without affecting reactivity—for analytics, metadata, performance optimization, or breaking circular dependencies. But remember: most of the time, automatic tracking is exactly what you want. Use `untrack()` deliberately and document why you're using it.
