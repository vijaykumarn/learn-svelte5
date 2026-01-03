# Article 11: $effect.pre & $effect.root - Advanced Effects

## 🎯 Hook

You've mastered `$effect()` for handling side effects, syncing with localStorage, and tracking analytics. But what happens when you need surgical precision over *when* your effects run? What if you need to measure DOM elements *before* Svelte updates them? Or create effects that live outside the normal component lifecycle?

Most developers hit a wall when building animations, implementing scroll restoration, or managing complex third-party integrations. The basic `$effect()` runs *after* the DOM updates—which is perfect 90% of the time. But that other 10%? That's where `$effect.pre()` and `$effect.root()` become indispensable.

In this article, we'll explore Svelte 5's advanced effect APIs that give you complete control over effect timing and lifecycle. You'll learn when the DOM actually updates, how to intercept it beforehand, and how to create effects that outlive components. By the end, you'll have the tools to build anything from smooth FLIP animations to sophisticated effect managers.

---

## 📚 Learning Goals

By the end of this article, you will:

- Understand the exact timing of `$effect()` vs `$effect.pre()` execution
- Know when to use pre-effects for DOM measurements and animations
- Master `$effect.root()` for creating manual effect scopes
- Learn how to properly dispose of effect roots to prevent memory leaks
- Build a practical animation system using pre-effects
- Recognize patterns where fine-grained timing control is essential
- Avoid common pitfalls with effect ordering and cleanup

---

## 📋 Prerequisites

Before diving into this article, you should be comfortable with:

- **Article 4**: Basic `$effect()` usage and cleanup functions
- **Article 2**: `$state()` and reactive primitives
- **Article 3**: `$derived()` for computed values
- Basic understanding of the browser's rendering pipeline (optional but helpful)
- Familiarity with JavaScript async behavior

If effects are still new to you, revisit Article 4 first. This article builds directly on those foundations.

---

## 🧠 Core Concept: The Effect Timeline

### Understanding Effect Execution Order

When Svelte detects a state change, it doesn't immediately update the DOM and run effects. Instead, it follows a precise sequence:

```
1. State changes (e.g., count = 5)
2. Derived values recalculate
3. $effect.pre() runs ← Before DOM updates
4. DOM updates (Svelte patches the real DOM)
5. $effect() runs ← After DOM updates
```

This timeline is crucial for understanding when to use each effect type.

### Why $effect.pre() Exists

The standard `$effect()` runs *after* the DOM has been updated. This is ideal for:
- Focusing elements
- Triggering animations on new content
- Syncing with external APIs
- Logging analytics

But sometimes you need to act *before* the DOM changes:
- Measuring element dimensions before they change (for FLIP animations)
- Reading scroll positions before content shifts
- Capturing "before" snapshots for transitions
- Coordinating with libraries that need pre-update hooks

That's exactly what `$effect.pre()` provides.

### The $effect.root() Pattern

Normal effects are automatically tied to the component lifecycle—they're created when the component mounts and cleaned up when it unmounts. But what if you need effects that:

- Outlive a single component
- Are created dynamically outside components
- Need manual lifecycle control
- Manage global state or singletons

This is where `$effect.root()` shines. It creates a manual "effect scope" that you control explicitly.

---

## 💻 Live Examples

### Example 1: Basic Pre-Effect Timing

Let's start by observing the timing difference:

```svelte
<script>
  let count = $state(0);
  
  $effect.pre(() => {
    console.log('PRE: count is', count);
    console.log('PRE: DOM says:', document.querySelector('.count')?.textContent);
  });
  
  $effect(() => {
    console.log('POST: count is', count);
    console.log('POST: DOM says:', document.querySelector('.count')?.textContent);
  });
</script>

<div class="count">{count}</div>
<button onclick={() => count++}>Increment</button>
```

**Output when clicking the button:**
```
PRE: count is 1
PRE: DOM says: 0           ← DOM hasn't updated yet!
POST: count is 1
POST: DOM says: 1          ← DOM is now updated
```

**Key Insight**: `$effect.pre()` sees the new state value but the old DOM content. This is perfect for "before" measurements.

### Example 2: FLIP Animation with Pre-Effects

FLIP (First, Last, Invert, Play) is a technique for smooth animations. You need to:
1. Measure the element's position BEFORE the change
2. Let the change happen
3. Measure the AFTER position
4. Animate from before to after

```svelte
<script>
  let items = $state(['Alice', 'Bob', 'Charlie']);
  let measurements = new Map();
  
  // FIRST: Capture positions before DOM update
  $effect.pre(() => {
    items; // Track dependency
    
    measurements.clear();
    document.querySelectorAll('.item').forEach((el) => {
      measurements.set(el.textContent, el.getBoundingClientRect());
    });
  });
  
  // LAST, INVERT, PLAY: Animate to new positions after DOM update
  $effect(() => {
    items; // Track same dependency
    
    document.querySelectorAll('.item').forEach((el) => {
      const first = measurements.get(el.textContent);
      const last = el.getBoundingClientRect();
      
      if (first) {
        const deltaY = first.top - last.top;
        
        if (deltaY !== 0) {
          // Invert: move element back to old position
          el.style.transform = `translateY(${deltaY}px)`;
          el.style.transition = 'none';
          
          // Play: animate to new position
          requestAnimationFrame(() => {
            el.style.transform = '';
            el.style.transition = 'transform 0.3s ease-out';
          });
        }
      }
    });
  });
  
  function shuffle() {
    items = items.slice().sort(() => Math.random() - 0.5);
  }
</script>

<style>
  .item {
    padding: 12px;
    background: #f0f0f0;
    margin: 4px 0;
    border-radius: 4px;
  }
</style>

<button onclick={shuffle}>Shuffle</button>

<div>
  {#each items as item (item)}
    <div class="item">{item}</div>
  {/each}
</div>
```

**What's Happening:**
1. `$effect.pre()` runs first, capturing positions in the old DOM
2. Svelte updates the DOM (reorders the items)
3. `$effect()` runs, calculates the movement, and animates

Without `$effect.pre()`, you couldn't capture the "before" positions!

### Example 3: Scroll Position Restoration

When content changes size, you often want to maintain scroll position:

```svelte
<script>
  let expanded = $state(false);
  let scrollBefore = 0;
  let container;
  
  $effect.pre(() => {
    expanded; // Track dependency
    
    if (container) {
      // Capture scroll position BEFORE content expands/collapses
      scrollBefore = container.scrollTop;
    }
  });
  
  $effect(() => {
    expanded; // Track same dependency
    
    if (container) {
      // Restore scroll position AFTER DOM updates
      container.scrollTop = scrollBefore;
    }
  });
</script>

<div bind:this={container} style="height: 200px; overflow-y: auto; border: 1px solid #ccc;">
  <div style="height: 100px; background: #e0e0e0;">Header content</div>
  
  {#if expanded}
    <div style="height: 400px; background: #f5f5f5;">
      Expanded content that would normally shift scroll position...
    </div>
  {/if}
  
  <div style="height: 500px; background: #e0e0e0;">
    More content below...
  </div>
</div>

<button onclick={() => expanded = !expanded}>
  Toggle Content
</button>
```

Without pre-effects, the scroll would jump when content is added or removed.

### Example 4: Manual Effect Scopes with $effect.root()

Sometimes you need effects that aren't tied to a component lifecycle:

```svelte
<script>
  let globalCount = $state(0);
  let effectRoot = null;
  
  function startGlobalEffect() {
    // Create a manual effect scope
    effectRoot = $effect.root(() => {
      $effect(() => {
        console.log('Global effect running, count:', globalCount);
        
        // This effect runs even after the component that created it unmounts!
      });
    });
  }
  
  function stopGlobalEffect() {
    if (effectRoot) {
      effectRoot(); // Calling the cleanup function disposes the effect
      effectRoot = null;
      console.log('Global effect stopped');
    }
  }
  
  // Cleanup when component unmounts
  $effect(() => {
    return () => {
      stopGlobalEffect();
    };
  });
</script>

<div>
  <p>Global Count: {globalCount}</p>
  <button onclick={() => globalCount++}>Increment</button>
  
  <hr>
  
  <button onclick={startGlobalEffect} disabled={effectRoot !== null}>
    Start Global Effect
  </button>
  <button onclick={stopGlobalEffect} disabled={effectRoot === null}>
    Stop Global Effect
  </button>
</div>
```

**Key Points:**
- `$effect.root()` returns a cleanup function
- Effects inside the root run independently of component lifecycle
- You **must** call the cleanup function manually to prevent memory leaks
- Perfect for singletons, global event listeners, or dynamic effect creation

### Example 5: Dynamic Effect Management

Here's a practical pattern for managing multiple effect roots:

```svelte
<script>
  class EffectManager {
    roots = new Map();
    
    create(id, fn) {
      // Dispose existing effect with this ID
      this.dispose(id);
      
      // Create new effect root
      const cleanup = $effect.root(() => {
        fn();
      });
      
      this.roots.set(id, cleanup);
    }
    
    dispose(id) {
      const cleanup = this.roots.get(id);
      if (cleanup) {
        cleanup();
        this.roots.delete(id);
      }
    }
    
    disposeAll() {
      this.roots.forEach(cleanup => cleanup());
      this.roots.clear();
    }
  }
  
  const manager = new EffectManager();
  let trackedValue = $state(0);
  
  function addTracker(id) {
    manager.create(id, () => {
      $effect(() => {
        console.log(`Tracker ${id}: value is ${trackedValue}`);
      });
    });
  }
  
  // Cleanup all on unmount
  $effect(() => {
    return () => manager.disposeAll();
  });
</script>

<p>Value: {trackedValue}</p>
<button onclick={() => trackedValue++}>Increment</button>

<hr>

<button onclick={() => addTracker('A')}>Add Tracker A</button>
<button onclick={() => addTracker('B')}>Add Tracker B</button>
<button onclick={() => manager.dispose('A')}>Remove Tracker A</button>
<button onclick={() => manager.dispose('B')}>Remove Tracker B</button>
<button onclick={() => manager.disposeAll()}>Remove All</button>
```

This pattern is useful for:
- Dynamically adding/removing listeners
- Managing plugin systems
- Creating effect-based middleware
- Building reactive service layers

---

## ⚠️ Common Mistakes

### Mistake 1: Using Pre-Effects for Regular Side Effects

```svelte
<!-- ❌ BAD: Using pre-effect when regular effect would work -->
<script>
  let username = $state('');
  
  $effect.pre(() => {
    // This doesn't need to run before DOM updates
    localStorage.setItem('username', username);
  });
</script>
```

```svelte
<!-- ✅ GOOD: Use regular effect for simple side effects -->
<script>
  let username = $state('');
  
  $effect(() => {
    localStorage.setItem('username', username);
  });
</script>
```

**Why**: Pre-effects add complexity. Only use them when timing relative to DOM updates actually matters.

### Mistake 2: Forgetting to Dispose Effect Roots

```svelte
<!-- ❌ BAD: Memory leak! -->
<script>
  function createWatcher() {
    $effect.root(() => {
      $effect(() => {
        console.log('Watching...');
      });
    });
    // The cleanup function is lost—this effect never gets disposed!
  }
</script>
```

```svelte
<!-- ✅ GOOD: Store and call cleanup -->
<script>
  let cleanup = null;
  
  function createWatcher() {
    cleanup = $effect.root(() => {
      $effect(() => {
        console.log('Watching...');
      });
    });
  }
  
  function stopWatcher() {
    if (cleanup) {
      cleanup();
      cleanup = null;
    }
  }
  
  // Always cleanup on unmount
  $effect(() => () => stopWatcher());
</script>
```

### Mistake 3: Mixing Pre and Post Effect Logic

```svelte
<!-- ❌ BAD: Trying to do too much in one effect -->
<script>
  let items = $state([1, 2, 3]);
  
  $effect.pre(() => {
    // Measure before...
    const before = measureElements();
    
    // Wait, can't measure after here—DOM hasn't updated yet!
    // This won't work as intended
  });
</script>
```

```svelte
<!-- ✅ GOOD: Split into pre and post effects -->
<script>
  let items = $state([1, 2, 3]);
  let beforePositions = new Map();
  
  $effect.pre(() => {
    items; // Track dependency
    beforePositions = measureElements();
  });
  
  $effect(() => {
    items; // Track same dependency
    const afterPositions = measureElements();
    animateDifference(beforePositions, afterPositions);
  });
</script>
```

### Mistake 4: Unnecessary Nesting of Effect Roots

```svelte
<!-- ❌ BAD: Over-engineering -->
<script>
  const root1 = $effect.root(() => {
    const root2 = $effect.root(() => {
      $effect(() => {
        console.log('Too much nesting!');
      });
    });
  });
</script>
```

```svelte
<!-- ✅ GOOD: Keep it simple -->
<script>
  const root = $effect.root(() => {
    $effect(() => {
      console.log('Simple and clear');
    });
  });
</script>
```

**Why**: Effect roots are for manual lifecycle control. Nesting them creates unnecessary complexity without benefit.

---

## ✅ Best Practices

### 1. Use Pre-Effects Only When Timing Matters

**Good use cases:**
- FLIP animations (measuring before/after)
- Scroll position preservation
- Focus management during transitions
- Canvas/WebGL pre-render setup
- Third-party library coordination

**Bad use cases:**
- API calls
- localStorage sync
- Analytics tracking
- Simple DOM updates

### 2. Always Store Effect Root Cleanup Functions

```svelte
<script>
  // Create a proper cleanup system
  const effectRoots = new Set();
  
  function addEffectRoot(fn) {
    const cleanup = $effect.root(fn);
    effectRoots.add(cleanup);
    return () => {
      cleanup();
      effectRoots.delete(cleanup);
    };
  }
  
  // Cleanup all on unmount
  $effect(() => {
    return () => {
      effectRoots.forEach(cleanup => cleanup());
      effectRoots.clear();
    };
  });
</script>
```

### 3. Document Why You're Using Pre-Effects

```svelte
<script>
  // FLIP animation: need element positions BEFORE DOM update
  $effect.pre(() => {
    items; // Track
    capturePositions();
  });
  
  // Apply transforms AFTER DOM update
  $effect(() => {
    items; // Track
    animateToNewPositions();
  });
</script>
```

Comments help future maintainers understand the timing requirements.

### 4. Batch Pre and Post Effects with Shared State

```svelte
<script>
  // Shared state between pre and post effects
  let animationState = {
    before: new Map(),
    after: new Map()
  };
  
  $effect.pre(() => {
    items;
    animationState.before = measureElements();
  });
  
  $effect(() => {
    items;
    animationState.after = measureElements();
    performAnimation(animationState);
  });
</script>
```

### 5. Test Effect Cleanup Thoroughly

```svelte
<script>
  // Always test that your cleanup works
  function createTestEffect() {
    const cleanup = $effect.root(() => {
      $effect(() => {
        const interval = setInterval(() => {
          console.log('Interval running');
        }, 1000);
        
        return () => clearInterval(interval);
      });
    });
    
    // Verify cleanup works
    setTimeout(() => {
      cleanup();
      console.log('Effect should be stopped');
    }, 5000);
  }
</script>
```

---

## 🧩 Mental Model

### The Effect Timeline Visualization

Think of Svelte's reactive cycle as a three-stage pipeline:

```
┌─────────────────────────────────────────────┐
│ State Changes                               │
│ count = 5                                   │
└────────────────┬────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────┐
│ Pre-Update Phase                            │
│ • Derived values recalculate                │
│ • $effect.pre() runs                        │
│ • DOM is STILL showing old values           │
└────────────────┬────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────┐
│ DOM Update Phase                            │
│ • Svelte patches the real DOM               │
│ • Browser reflows/repaints                  │
└────────────────┬────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────┐
│ Post-Update Phase                           │
│ • $effect() runs                            │
│ • DOM now shows new values                  │
│ • Side effects execute                      │
└─────────────────────────────────────────────┘
```

### Effect Roots as Containers

Think of `$effect.root()` as creating an independent "effect container":

```
Normal Component Lifecycle:
┌──────────────────────────┐
│ Component Mounts         │
│  ↓                       │
│ Effects Created          │
│  ↓                       │
│ Component Updates        │
│  ↓                       │
│ Component Unmounts       │
│  ↓                       │
│ Effects Auto-Cleanup ✓   │
└──────────────────────────┘

Manual Effect Root:
┌──────────────────────────┐
│ Effect Root Created      │
│  ↓                       │
│ Effects Run              │
│  ↓                       │
│ Component Unmounts       │
│  ↓                       │
│ Effects STILL Running ⚠️  │
│  ↓                       │
│ Manual cleanup() called  │
│  ↓                       │
│ Effects Disposed ✓       │
└──────────────────────────┘
```

### When to Use Each Effect Type

```
Use $effect() when:
├─ You need the DOM to be updated first
├─ You're doing regular side effects
├─ You're syncing with external systems
└─ Timing doesn't matter much

Use $effect.pre() when:
├─ You need DOM measurements BEFORE updates
├─ You're implementing FLIP animations
├─ You're preserving scroll positions
└─ Timing is critical to your logic

Use $effect.root() when:
├─ Effects need to outlive components
├─ You're managing global state/singletons
├─ You're creating dynamic effect systems
└─ You need manual lifecycle control
```

---

## 📖 Quick Reference

### $effect.pre() Syntax

```svelte
<script>
  $effect.pre(() => {
    // Runs BEFORE DOM updates
    // Access state: ✓ (new values)
    // Access DOM: ✓ (old values)
  });
</script>
```

### $effect.root() Syntax

```svelte
<script>
  const cleanup = $effect.root(() => {
    // Create effects here
    $effect(() => {
      // This effect runs independently
    });
  });
  
  // Later: cleanup() to dispose
</script>
```

### Timing Comparison

| Feature | $effect.pre() | $effect() |
|---------|--------------|-----------|
| When it runs | Before DOM updates | After DOM updates |
| State values | New values ✓ | New values ✓ |
| DOM values | Old values | New values ✓ |
| Use for | Measurements, preparation | Side effects, sync |
| Common in | Animations, scroll handling | API calls, logging |

### Effect Root Lifecycle

```svelte
<script>
  // 1. Create
  const cleanup = $effect.root(() => { ... });
  
  // 2. Effects run independently
  
  // 3. Dispose manually
  cleanup(); // Must be called!
</script>
```

---

## 🏋️ Practice Exercise

**Challenge**: Build a "Smooth Reorder" component

**Requirements:**
1. Display a list of items with drag-and-drop reordering
2. When items are reordered, animate them smoothly to their new positions using FLIP
3. Track the number of times each item has been moved
4. Allow adding/removing items with animations
5. Use `$effect.pre()` to capture before-positions
6. Use `$effect()` to animate to after-positions

**Starter Code:**

```svelte
<script>
  let items = $state([
    { id: 1, name: 'Item A', moveCount: 0 },
    { id: 2, name: 'Item B', moveCount: 0 },
    { id: 3, name: 'Item C', moveCount: 0 }
  ]);
  
  // Your FLIP animation logic here
  let beforePositions = new Map();
  
  $effect.pre(() => {
    items; // Track dependency
    // TODO: Capture element positions
  });
  
  $effect(() => {
    items; // Track dependency
    // TODO: Animate to new positions
  });
  
  function addItem() {
    const newId = Math.max(...items.map(i => i.id)) + 1;
    items = [...items, { id: newId, name: `Item ${String.fromCharCode(64 + newId)}`, moveCount: 0 }];
  }
  
  function removeItem(id) {
    items = items.filter(item => item.id !== id);
  }
  
  function moveUp(index) {
    if (index > 0) {
      const newItems = [...items];
      [newItems[index], newItems[index - 1]] = [newItems[index - 1], newItems[index]];
      
      // Increment move count
      newItems[index].moveCount++;
      
      items = newItems;
    }
  }
  
  function moveDown(index) {
    if (index < items.length - 1) {
      const newItems = [...items];
      [newItems[index], newItems[index + 1]] = [newItems[index + 1], newItems[index]];
      
      // Increment move count
      newItems[index].moveCount++;
      
      items = newItems;
    }
  }
</script>

<style>
  .item {
    padding: 16px;
    margin: 8px 0;
    background: white;
    border: 2px solid #e0e0e0;
    border-radius: 8px;
    display: flex;
    justify-content: space-between;
    align-items: center;
  }
  
  .controls button {
    margin-left: 8px;
    padding: 4px 12px;
  }
</style>

<button onclick={addItem}>Add Item</button>

<div>
  {#each items as item, index (item.id)}
    <div class="item">
      <span>
        {item.name} 
        <small>(moved {item.moveCount} times)</small>
      </span>
      <div class="controls">
        <button onclick={() => moveUp(index)}>↑</button>
        <button onclick={() => moveDown(index)}>↓</button>
        <button onclick={() => removeItem(item.id)}>×</button>
      </div>
    </div>
  {/each}
</div>
```

**Bonus Challenges:**
- Add stagger delay for multiple simultaneous moves
- Implement elastic easing for the animations
- Create an effect root that manages animation state globally
- Add a "shuffle" button that animates all items simultaneously

**Solution Tips:**
1. In `$effect.pre()`, loop through `.item` elements and store their `getBoundingClientRect()`
2. In `$effect()`, compare before/after positions and apply transforms
3. Use `requestAnimationFrame()` for smooth animation timing
4. Don't forget to reset transforms and transitions after animation completes
5. Consider using a WeakMap to track element-to-data relationships

---

## 🎓 Next Steps

Congratulations! You now understand Svelte's advanced effect timing and lifecycle management. You can:

- Control exactly when effects run relative to DOM updates
- Build smooth FLIP animations with pre-effects
- Manage effect lifecycles independently of components
- Avoid common timing pitfalls

**Coming up in Article 12**: We'll explore `untrack()` and `$effect.tracking()` to gain even more control over effect dependencies. You'll learn how to:
- Read state without creating reactive dependencies
- Break infinite effect loops
- Optimize performance with selective reactivity
- Build sophisticated debouncing patterns

These tools will complete your mastery of Svelte's reactivity system!

---

## 📚 Additional Resources

- [Svelte 5 Docs: $effect.pre](https://svelte.dev/docs/svelte/$effect#$effect.pre)
- [Svelte 5 Docs: $effect.root](https://svelte.dev/docs/svelte/$effect#$effect.root)
- [FLIP Animation Technique](https://aerotwist.com/blog/flip-your-animations/)
- [Browser Rendering Pipeline](https://developer.mozilla.org/en-US/docs/Web/Performance/How_browsers_work)

---

**Discussion Question**: When have you needed fine-grained control over effect timing in your projects? Share your use cases in the comments below! 💬
