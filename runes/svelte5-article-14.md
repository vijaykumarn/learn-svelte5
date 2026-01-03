# $state.raw() - Performance Optimization in Svelte 5

## Why Performance Matters (Even When It Doesn't)

You've built a beautiful data table in Svelte 5. It displays 10,000 rows of product information, each row an object with 20 properties. You wire everything up with `$state()`, proud of your reactive masterpiece. Then you scroll... and it stutters. You type in the search box... and there's a noticeable lag. Your beautifully reactive code has become a performance bottleneck.

Here's the thing: Svelte 5's reactivity is powerful, but it comes at a cost. Every object and array you wrap in `$state()` gets deep reactivity—Svelte tracks every nested property, every array element, every change at any depth. For a shopping cart with 5 items? Perfect. For a data table with 10,000 rows? That's 200,000 properties being tracked. The overhead adds up fast.

But there's a solution: `$state.raw()`. It gives you reactive state without the deep tracking overhead. In this article, we'll explore when deep reactivity helps, when it hurts, and how to use `$state.raw()` to build blazingly fast applications without sacrificing reactivity where you need it.

---

## 🎯 Learning Goals

By the end of this article, you'll be able to:

- Understand how deep reactivity works and its performance implications
- Identify scenarios where `$state.raw()` outperforms `$state()`
- Implement `$state.raw()` for performance-critical data structures
- Mix reactive and raw state strategically
- Convert between reactive and raw state safely
- Optimize real-world applications with large datasets
- Debug performance issues related to reactivity overhead

---

## 📋 Prerequisites

Before diving in, you should understand:

- **Article 2**: Basic `$state()` usage
- **Article 6**: Working with reactive arrays
- **Article 7**: Managing reactive objects
- **Article 8**: Arrays of objects patterns

Familiarity with JavaScript performance concepts (time complexity, memory overhead) is helpful but not required.

---

## 🧠 Core Concept: Deep Reactivity Explained

### How $state() Works Under the Hood

When you write `let items = $state([])`, Svelte doesn't just store your array. It wraps it in a Proxy that intercepts every operation:

```svelte
<script>
let user = $state({
  name: 'Alice',
  address: {
    city: 'Stockholm',
    coordinates: { lat: 59.3293, lng: 18.0686 }
  }
});
</script>
```

Behind the scenes, Svelte creates proxies for:
1. The `user` object
2. The nested `address` object
3. The nested `coordinates` object

Every property access, every mutation triggers the proxy's intercept logic. This is **deep reactivity**—changes anywhere in the tree are detected automatically.

### The Performance Cost

Each proxy adds overhead:

| Operation | Regular Object | Reactive Object |
|-----------|---------------|-----------------|
| Property read | ~1 ns | ~5-10 ns |
| Property write | ~1 ns | ~50-100 ns (+ dependency tracking) |
| Memory per object | Base size | Base size + proxy wrapper |

For small objects, this overhead is negligible. But multiply it by 10,000 items, and suddenly you're paying a significant performance tax.

### When Deep Reactivity Shines

Deep reactivity is **amazing** for:

```svelte
<script>
let cart = $state({
  items: [
    { id: 1, name: 'Book', price: 20, quantity: 2 }
  ],
  discount: 0.1
});

// This "just works" - any nested change triggers updates
cart.items[0].quantity = 3; // ✅ Reactive!
cart.discount = 0.15; // ✅ Reactive!
</script>
```

You don't think about reactivity—it's automatic and delightful.

### When Deep Reactivity Hurts

Deep reactivity becomes a problem when:

1. **Large datasets**: Thousands of objects with many properties
2. **Frequent updates**: High-frequency data changes (real-time data, animations)
3. **Read-heavy operations**: You're not actually mutating the data
4. **Immutable patterns**: You replace entire objects instead of mutating them

---

## 💻 Live Example: The Performance Problem

Let's see the problem in action with a data table:

```svelte
<script>
// ❌ SLOW: Every row, every property is deeply reactive
let products = $state(
  Array.from({ length: 10000 }, (_, i) => ({
    id: i,
    name: `Product ${i}`,
    price: Math.random() * 100,
    stock: Math.floor(Math.random() * 1000),
    category: ['Electronics', 'Clothing', 'Food'][i % 3],
    description: 'Lorem ipsum dolor sit amet...',
    ratings: { average: 4.5, count: 120 },
    metadata: { created: new Date(), updated: new Date() }
  }))
);

let searchTerm = $state('');

// This recalculates on every keystroke and it's SLOW
let filtered = $derived(
  products.filter(p => 
    p.name.toLowerCase().includes(searchTerm.toLowerCase())
  )
);
</script>

<input bind:value={searchTerm} placeholder="Search products..." />

<table>
  {#each filtered as product}
    <tr>
      <td>{product.name}</td>
      <td>${product.price.toFixed(2)}</td>
      <td>{product.stock}</td>
    </tr>
  {/each}
</table>
```

**Problem**: Typing in the search box feels sluggish because:
- 10,000 proxy-wrapped objects
- Each with 8 properties (some nested)
- Filter operation touches every object
- Reactivity overhead on every iteration

### The Solution: $state.raw()

```svelte
<script>
// ✅ FAST: Raw state, no deep reactivity overhead
let products = $state.raw(
  Array.from({ length: 10000 }, (_, i) => ({
    id: i,
    name: `Product ${i}`,
    price: Math.random() * 100,
    stock: Math.floor(Math.random() * 1000),
    category: ['Electronics', 'Clothing', 'Food'][i % 3],
    description: 'Lorem ipsum dolor sit amet...',
    ratings: { average: 4.5, count: 120 },
    metadata: { created: new Date(), updated: new Date() }
  }))
);

let searchTerm = $state('');

// Same derived logic, but now it's fast!
let filtered = $derived(
  products.filter(p => 
    p.name.toLowerCase().includes(searchTerm.toLowerCase())
  )
);
</script>
```

**The catch**: Updates to nested properties aren't reactive:

```svelte
<script>
// ❌ Won't trigger UI updates
products[0].price = 99.99;

// ✅ Will trigger UI updates (replacing the array)
products = products.map(p => 
  p.id === 0 ? { ...p, price: 99.99 } : p
);
</script>
```

---

## 🎭 Real-World Example: Optimized Data Table

Here's a complete, production-ready data table with smart state management:

```svelte
<script>
// Raw state for the large dataset (read-heavy)
let allProducts = $state.raw(
  Array.from({ length: 10000 }, (_, i) => ({
    id: i,
    name: `Product ${i}`,
    price: Math.random() * 100,
    stock: Math.floor(Math.random() * 1000),
    category: ['Electronics', 'Clothing', 'Food'][i % 3],
    sku: `SKU-${i.toString().padStart(5, '0')}`
  }))
);

// Reactive state for UI controls (small, frequently updated)
let filters = $state({
  search: '',
  category: 'all',
  minPrice: 0,
  maxPrice: 100
});

let sorting = $state({
  column: 'name',
  direction: 'asc'
});

// Reactive state for selected items (moderate size)
let selectedIds = $state(new Set());

// Derived values for filtered and sorted results
let filteredProducts = $derived(() => {
  let results = allProducts;
  
  // Filter by search
  if (filters.search) {
    const term = filters.search.toLowerCase();
    results = results.filter(p => 
      p.name.toLowerCase().includes(term) ||
      p.sku.toLowerCase().includes(term)
    );
  }
  
  // Filter by category
  if (filters.category !== 'all') {
    results = results.filter(p => p.category === filters.category);
  }
  
  // Filter by price range
  results = results.filter(p => 
    p.price >= filters.minPrice && p.price <= filters.maxPrice
  );
  
  return results;
});

let sortedProducts = $derived(() => {
  const sorted = [...filteredProducts];
  const { column, direction } = sorting;
  const multiplier = direction === 'asc' ? 1 : -1;
  
  sorted.sort((a, b) => {
    if (a[column] < b[column]) return -1 * multiplier;
    if (a[column] > b[column]) return 1 * multiplier;
    return 0;
  });
  
  return sorted;
});

// Statistics (derived from filtered results)
let stats = $derived({
  total: sortedProducts.length,
  selected: selectedIds.size,
  avgPrice: sortedProducts.reduce((sum, p) => sum + p.price, 0) / 
           (sortedProducts.length || 1)
});

function toggleSelection(id) {
  // Create new Set to trigger reactivity
  const newSet = new Set(selectedIds);
  if (newSet.has(id)) {
    newSet.delete(id);
  } else {
    newSet.add(id);
  }
  selectedIds = newSet;
}

function toggleSort(column) {
  if (sorting.column === column) {
    sorting.direction = sorting.direction === 'asc' ? 'desc' : 'asc';
  } else {
    sorting = { column, direction: 'asc' };
  }
}

function updatePrice(id, newPrice) {
  // Immutable update pattern for raw state
  allProducts = allProducts.map(p =>
    p.id === id ? { ...p, price: newPrice } : p
  );
}
</script>

<div class="controls">
  <input
    bind:value={filters.search}
    placeholder="Search products..."
  />
  
  <select bind:value={filters.category}>
    <option value="all">All Categories</option>
    <option value="Electronics">Electronics</option>
    <option value="Clothing">Clothing</option>
    <option value="Food">Food</option>
  </select>
  
  <div class="price-range">
    <input
      type="range"
      bind:value={filters.minPrice}
      min="0"
      max="100"
    />
    <input
      type="range"
      bind:value={filters.maxPrice}
      min="0"
      max="100"
    />
    <span>${filters.minPrice} - ${filters.maxPrice}</span>
  </div>
</div>

<div class="stats">
  <p>Showing {stats.total} products</p>
  <p>{stats.selected} selected</p>
  <p>Avg Price: ${stats.avgPrice.toFixed(2)}</p>
</div>

<table>
  <thead>
    <tr>
      <th><input type="checkbox" /></th>
      <th onclick={() => toggleSort('name')}>
        Name {sorting.column === 'name' ? (sorting.direction === 'asc' ? '↑' : '↓') : ''}
      </th>
      <th onclick={() => toggleSort('price')}>
        Price {sorting.column === 'price' ? (sorting.direction === 'asc' ? '↑' : '↓') : ''}
      </th>
      <th onclick={() => toggleSort('stock')}>
        Stock {sorting.column === 'stock' ? (sorting.direction === 'asc' ? '↑' : '↓') : ''}
      </th>
      <th>Category</th>
      <th>Actions</th>
    </tr>
  </thead>
  <tbody>
    {#each sortedProducts as product (product.id)}
      <tr class:selected={selectedIds.has(product.id)}>
        <td>
          <input
            type="checkbox"
            checked={selectedIds.has(product.id)}
            onchange={() => toggleSelection(product.id)}
          />
        </td>
        <td>{product.name}</td>
        <td>${product.price.toFixed(2)}</td>
        <td>{product.stock}</td>
        <td>{product.category}</td>
        <td>
          <button onclick={() => updatePrice(product.id, product.price * 1.1)}>
            +10%
          </button>
        </td>
      </tr>
    {/each}
  </tbody>
</table>

<style>
  .controls {
    display: flex;
    gap: 1rem;
    margin-bottom: 1rem;
  }
  
  .stats {
    display: flex;
    gap: 2rem;
    margin-bottom: 1rem;
    padding: 1rem;
    background: #f5f5f5;
    border-radius: 4px;
  }
  
  table {
    width: 100%;
    border-collapse: collapse;
  }
  
  th {
    cursor: pointer;
    user-select: none;
    background: #eee;
    padding: 0.5rem;
    text-align: left;
  }
  
  td {
    padding: 0.5rem;
    border-bottom: 1px solid #ddd;
  }
  
  tr.selected {
    background: #e3f2fd;
  }
</style>
```

**Key Optimization Strategies**:

1. **Raw state for data**: The 10,000 products use `$state.raw()`
2. **Reactive state for UI**: Filters, sorting, selections use regular `$state()`
3. **Immutable updates**: When changing products, we replace the entire array
4. **Keyed each blocks**: `{#each ... (product.id)}` for efficient DOM updates
5. **Derived computations**: Filtering and sorting are memoized

---

## ⚠️ Common Mistakes

### Mistake 1: Using raw state for everything

```svelte
<script>
// ❌ Too aggressive - UI state should be reactive!
let formData = $state.raw({
  name: '',
  email: '',
  age: 0
});

// Now this won't trigger updates when you type:
// <input bind:value={formData.name} />
</script>
```

**Fix**: Use raw state only for large, read-heavy datasets.

```svelte
<script>
// ✅ Reactive for small, interactive data
let formData = $state({
  name: '',
  email: '',
  age: 0
});
</script>
```

### Mistake 2: Mutating raw state

```svelte
<script>
let items = $state.raw([{ id: 1, name: 'A' }]);

function update() {
  // ❌ Won't trigger UI updates!
  items[0].name = 'B';
}
</script>
```

**Fix**: Use immutable update patterns.

```svelte
<script>
function update() {
  // ✅ Triggers reactivity by replacing the array
  items = items.map(item =>
    item.id === 1 ? { ...item, name: 'B' } : item
  );
}
</script>
```

### Mistake 3: Mixing reactivity expectations

```svelte
<script>
let config = $state.raw({
  theme: 'dark',
  settings: { fontSize: 16 }
});

// ❌ Expects nested reactivity, but it's raw!
config.settings.fontSize = 18; // No UI update!
</script>
```

**Fix**: Understand your state's reactivity level and update accordingly.

```svelte
<script>
// ✅ Replace the entire object
config = {
  ...config,
  settings: { ...config.settings, fontSize: 18 }
};
</script>
```

### Mistake 4: Converting unnecessarily

```svelte
<script>
let data = $state.raw([...]);

// ❌ Converting back and forth defeats the purpose!
let reactive = $state(data);
let raw = $state.raw(reactive);
</script>
```

**Fix**: Stick with one approach per data structure.

---

## ✅ Best Practices

### 1. Profile Before Optimizing

Don't guess—measure:

```svelte
<script>
let items = $state([...]); // Start with regular state

$effect(() => {
  console.time('filter');
  const filtered = items.filter(/* ... */);
  console.timeEnd('filter');
});
</script>
```

Only switch to `$state.raw()` if you see actual performance problems.

### 2. Use Raw State for Read-Heavy Operations

Perfect candidates for `$state.raw()`:

- Large datasets you're filtering/searching
- Configuration that rarely changes
- Reference data (countries, categories, etc.)
- API response caches

```svelte
<script>
// ✅ Read frequently, updated rarely
let productCatalog = $state.raw(await fetchProducts());
let categories = $state.raw(['Electronics', 'Clothing', 'Food']);
</script>
```

### 3. Keep UI State Reactive

Always use regular `$state()` for:

- Form inputs
- UI toggles and flags
- Animation state
- User selections

```svelte
<script>
// ✅ Interactive UI state
let isOpen = $state(false);
let selectedTab = $state('home');
let searchQuery = $state('');
</script>
```

### 4. Batch Updates

When updating raw state, batch changes:

```svelte
<script>
let items = $state.raw([...]);

// ❌ Multiple array replacements
function updateMultiple() {
  items = items.map(i => i.id === 1 ? { ...i, checked: true } : i);
  items = items.map(i => i.id === 2 ? { ...i, checked: true } : i);
  items = items.map(i => i.id === 3 ? { ...i, checked: true } : i);
}

// ✅ Single update
function updateMultiple() {
  const idsToCheck = new Set([1, 2, 3]);
  items = items.map(i => 
    idsToCheck.has(i.id) ? { ...i, checked: true } : i
  );
}
</script>
```

### 5. Strategic Snapshots

Use `$state.snapshot()` to get non-reactive copies:

```svelte
<script>
let liveData = $state({ temp: 20, humidity: 60 });

// Create a snapshot for comparison
let baseline = $state.snapshot(liveData);

let delta = $derived({
  temp: liveData.temp - baseline.temp,
  humidity: liveData.humidity - baseline.humidity
});
</script>
```

### 6. Document Your Decisions

```svelte
<script>
// Using $state.raw() here because:
// - 50,000+ items
// - Read-heavy (filtering, sorting)
// - Updates happen via full array replacement
// - Profiling showed 300ms -> 50ms improvement
let products = $state.raw(largeDataset);
</script>
```

---

## 🧠 Mental Model: Reactivity as a Spectrum

Think of state reactivity as a spectrum:

```
Deep Reactive    Shallow Reactive    Raw (Non-Reactive)
    $state()       $state.raw()         Plain JS
       |                |                    |
  Automatic          Semi-Automatic       Manual
  Convenient         Performant          Full Control
  (Small data)       (Large data)        (External libs)
```

**Choose based on**:
- **Size**: How much data?
- **Frequency**: How often does it change?
- **Depth**: Do you need nested reactivity?
- **Pattern**: Mutation vs replacement?

---

## 📖 Quick Reference

### $state() vs $state.raw() Comparison

```svelte
<script>
// DEEP REACTIVITY ($state)
let user = $state({ 
  profile: { name: 'Alice' } 
});
user.profile.name = 'Bob'; // ✅ Reactive

// SHALLOW REACTIVITY ($state.raw)
let user = $state.raw({ 
  profile: { name: 'Alice' } 
});
user.profile.name = 'Bob'; // ❌ Not reactive
user = { ...user, profile: { name: 'Bob' }}; // ✅ Reactive
</script>
```

### When to Use What

| Scenario | Use |
|----------|-----|
| Small objects (< 100 items) | `$state()` |
| Large datasets (1000+ items) | `$state.raw()` |
| Form inputs | `$state()` |
| API response caching | `$state.raw()` |
| Nested mutations needed | `$state()` |
| Immutable update pattern | `$state.raw()` |
| UI interaction state | `$state()` |
| Read-heavy operations | `$state.raw()` |

### Update Patterns Cheat Sheet

```svelte
<script>
let items = $state.raw([...]);

// ✅ CORRECT: Replace array
items = [...items, newItem];
items = items.filter(i => i.id !== deleteId);
items = items.map(i => i.id === id ? updated : i);

// ❌ WRONG: Mutate array
items.push(newItem);
items[0].name = 'Updated';
delete items[2];
</script>
```

---

## 🏋️ Practice Exercise

Build a **Virtual Scrolling List** with 100,000 items:

**Requirements**:
1. Display a list of 100,000 user records
2. Only render visible items (virtual scrolling)
3. Search functionality
4. Click to select items
5. Batch operations (select all, delete selected)

**Optimization Challenges**:
- Use `$state.raw()` for the full dataset
- Use `$state()` for visible items and selections
- Implement efficient filtering
- Track scroll position reactively
- Handle selections without scanning the full array

**Starter Template**:

```svelte
<script>
// Generate 100,000 users
let allUsers = $state.raw(
  Array.from({ length: 100000 }, (_, i) => ({
    id: i,
    name: `User ${i}`,
    email: `user${i}@example.com`,
    role: ['Admin', 'User', 'Guest'][i % 3]
  }))
);

let searchTerm = $state('');
let selectedIds = $state(new Set());
let scrollTop = $state(0);

const ITEM_HEIGHT = 50;
const VISIBLE_COUNT = 20;

// TODO: Implement filtered list
// TODO: Calculate visible slice based on scroll
// TODO: Implement selection logic
// TODO: Implement batch operations
</script>

<div class="container">
  <input bind:value={searchTerm} placeholder="Search..." />
  
  <div 
    class="list-container"
    style="height: {VISIBLE_COUNT * ITEM_HEIGHT}px"
  >
    <!-- Render only visible items here -->
  </div>
  
  <div class="actions">
    <button>Select All</button>
    <button>Delete Selected</button>
    <span>{selectedIds.size} selected</span>
  </div>
</div>
```

**Bonus Challenges**:
- Add sorting
- Implement keyboard navigation
- Add infinite scroll loading more data
- Measure and display performance metrics

---

## 🔜 Next Steps

You now understand when and how to use `$state.raw()` for performance optimization. In the next article, **Article 15: "Class Fields & Runes"**, we'll explore object-oriented patterns with Svelte 5, including:

- Using Runes in class properties
- Building reactive class instances
- When OOP patterns make sense in Svelte
- Creating reusable reactive stores with classes

Until then, experiment with `$state.raw()` in your own projects. Look for opportunities where you have large datasets and immutable update patterns—those are prime candidates for optimization!

---

## 📚 Further Reading

- [Svelte 5 Runes Documentation](https://svelte.dev/docs/runes)
- [JavaScript Proxy Performance](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy)
- [Immutable Update Patterns](https://redux.js.org/usage/structuring-reducers/immutable-update-patterns)
- [Virtual Scrolling Techniques](https://developer.chrome.com/blog/virtualize-long-lists-react-window/)

Happy optimizing! 🚀
