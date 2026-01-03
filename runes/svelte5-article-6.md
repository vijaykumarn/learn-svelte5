# Arrays - Lists That React

## Why Arrays Are Different in Svelte 5

If you've worked through the first five articles in this series, you've mastered single values with `$state()`, computed values with `$derived()`, and side effects with `$effect()`. But here's where things get interesting: **arrays don't behave the way you might expect.**

In traditional JavaScript, we're used to mutating arrays directly. We `push()` items, `splice()` them out, and `sort()` them in place. It feels natural. But in Svelte 5's reactive system, this approach breaks down. Why? Because Svelte tracks *assignments*, not *mutations*. When you call `array.push(item)`, you're modifying the array's contents, but you're not reassigning the variable itself. Svelte doesn't see this change, so your UI doesn't update.

This isn't a limitation—it's actually a feature that makes your code more predictable and easier to debug. Once you understand the patterns for working with reactive arrays, you'll find they're not just powerful, but elegant. In this article, we'll explore how to manage collections of data in Svelte 5, from simple lists to complex filtered and sorted datasets. By the end, you'll be building dynamic, reactive lists with confidence.

---

## 🎯 Learning Goals

By the end of this article, you will:

- ✅ Understand why array mutations don't trigger reactivity
- ✅ Master immutable update patterns (spread, map, filter)
- ✅ Use the `.with()` method for cleaner array updates
- ✅ Know when to use `$state()` vs `$state.raw()` for arrays
- ✅ Create derived values from array data (counts, totals, filters)
- ✅ Build a complete todo list with filtering and statistics
- ✅ Avoid common pitfalls that break reactivity

---

## 📚 Prerequisites

Before diving into this article, you should be comfortable with:

- **Article 2**: `$state()` for managing reactive values
- **Article 3**: `$derived()` for computed values
- **Article 4**: `$effect()` for side effects (helpful but not required)
- **JavaScript arrays**: Basic array methods (map, filter, reduce)

---

## 🧠 Core Concept: Assignments vs Mutations

### The Fundamental Rule

Svelte 5's reactivity system tracks **assignments**, not **mutations**. Let's see what this means:

```svelte
<script>
  let count = $state(0);
  
  // ✅ This works - assignment
  count = count + 1;
  
  // ❌ This doesn't exist for primitives
  // count.increment(); // No such method
</script>
```

For primitives (numbers, strings, booleans), this is straightforward. You always use assignment. But arrays are objects, and objects can be mutated:

```svelte
<script>
  let items = $state([1, 2, 3]);
  
  // ❌ This DOESN'T trigger reactivity
  items.push(4); // Mutation - no assignment
  
  // ✅ This DOES trigger reactivity
  items = [...items, 4]; // Assignment - creates new array
</script>
```

**Why does this matter?** When you mutate an array, you're changing its internal contents, but the variable `items` still points to the same array object. Svelte sees no change. When you assign a new array, Svelte detects that `items` now points to a different object and updates your UI.

### Deep Reactivity Explained

When you use `$state([])`, Svelte wraps your array in a Proxy that tracks access and assignments. This is called **deep reactivity**:

```svelte
<script>
  let items = $state([
    { id: 1, name: 'Apple', price: 1.50 },
    { id: 2, name: 'Banana', price: 0.75 }
  ]);
  
  // ✅ This works - reassigning the array
  items = items.map(item => 
    item.id === 1 ? { ...item, price: 2.00 } : item
  );
  
  // ❌ This doesn't work - mutating without reassignment
  items[0].price = 2.00;
</script>
```

Even though we're dealing with nested objects, we still need to reassign the parent array for reactivity to trigger.

---

## 💻 Live Example: Todo List with Statistics

Let's build a complete todo list application that demonstrates all the key concepts:

```svelte
<script>
  // State: Our list of todos
  let todos = $state([
    { id: 1, text: 'Learn Svelte 5 Runes', completed: false },
    { id: 2, text: 'Build a todo app', completed: false },
    { id: 3, text: 'Master reactive arrays', completed: false }
  ]);
  
  let newTodoText = $state('');
  let filter = $state('all'); // 'all', 'active', 'completed'
  
  // Derived: Statistics
  let totalCount = $derived(todos.length);
  let completedCount = $derived(
    todos.filter(t => t.completed).length
  );
  let activeCount = $derived(totalCount - completedCount);
  
  // Derived: Filtered list
  let filteredTodos = $derived(() => {
    if (filter === 'active') {
      return todos.filter(t => !t.completed);
    }
    if (filter === 'completed') {
      return todos.filter(t => t.completed);
    }
    return todos;
  });
  
  // Actions
  function addTodo() {
    if (newTodoText.trim() === '') return;
    
    const newTodo = {
      id: Date.now(),
      text: newTodoText.trim(),
      completed: false
    };
    
    // ✅ Immutable update pattern
    todos = [...todos, newTodo];
    newTodoText = '';
  }
  
  function toggleTodo(id) {
    // ✅ Using map to create a new array
    todos = todos.map(todo =>
      todo.id === id 
        ? { ...todo, completed: !todo.completed }
        : todo
    );
  }
  
  function deleteTodo(id) {
    // ✅ Using filter to create a new array
    todos = todos.filter(todo => todo.id !== id);
  }
  
  function clearCompleted() {
    todos = todos.filter(todo => !todo.completed);
  }
</script>

<div class="todo-app">
  <h1>My Todos</h1>
  
  <!-- Add new todo -->
  <div class="add-todo">
    <input
      type="text"
      bind:value={newTodoText}
      placeholder="What needs to be done?"
      onkeydown={(e) => e.key === 'Enter' && addTodo()}
    />
    <button onclick={addTodo}>Add</button>
  </div>
  
  <!-- Filter buttons -->
  <div class="filters">
    <button 
      class:active={filter === 'all'}
      onclick={() => filter = 'all'}
    >
      All ({totalCount})
    </button>
    <button 
      class:active={filter === 'active'}
      onclick={() => filter = 'active'}
    >
      Active ({activeCount})
    </button>
    <button 
      class:active={filter === 'completed'}
      onclick={() => filter = 'completed'}
    >
      Completed ({completedCount})
    </button>
  </div>
  
  <!-- Todo list -->
  <ul class="todo-list">
    {#each filteredTodos as todo (todo.id)}
      <li class:completed={todo.completed}>
        <input
          type="checkbox"
          checked={todo.completed}
          onchange={() => toggleTodo(todo.id)}
        />
        <span>{todo.text}</span>
        <button onclick={() => deleteTodo(todo.id)}>Delete</button>
      </li>
    {/each}
  </ul>
  
  <!-- Stats and actions -->
  <div class="footer">
    <span>{activeCount} item{activeCount !== 1 ? 's' : ''} left</span>
    {#if completedCount > 0}
      <button onclick={clearCompleted}>Clear completed</button>
    {/if}
  </div>
</div>

<style>
  .todo-app {
    max-width: 600px;
    margin: 0 auto;
    padding: 20px;
  }
  
  .add-todo {
    display: flex;
    gap: 10px;
    margin-bottom: 20px;
  }
  
  .add-todo input {
    flex: 1;
    padding: 10px;
    font-size: 16px;
  }
  
  .filters {
    display: flex;
    gap: 10px;
    margin-bottom: 20px;
  }
  
  .filters button {
    padding: 8px 16px;
    background: #f0f0f0;
    border: 2px solid transparent;
  }
  
  .filters button.active {
    border-color: #4CAF50;
    background: #e8f5e9;
  }
  
  .todo-list {
    list-style: none;
    padding: 0;
  }
  
  .todo-list li {
    display: flex;
    align-items: center;
    gap: 10px;
    padding: 12px;
    border-bottom: 1px solid #eee;
  }
  
  .todo-list li.completed span {
    text-decoration: line-through;
    color: #999;
  }
  
  .todo-list span {
    flex: 1;
  }
  
  .footer {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-top: 20px;
    padding-top: 20px;
    border-top: 1px solid #eee;
  }
</style>
```

### Breaking Down the Example

**State Management:**
- `todos`: Our reactive array of todo objects
- `newTodoText`: Input value for new todos
- `filter`: Current filter selection

**Derived Values:**
- `totalCount`, `completedCount`, `activeCount`: Computed statistics
- `filteredTodos`: Filtered list based on current filter

**Update Patterns:**
- `addTodo()`: Uses spread operator `[...todos, newTodo]`
- `toggleTodo()`: Uses `map()` to create new array with updated item
- `deleteTodo()`: Uses `filter()` to create new array without item
- `clearCompleted()`: Uses `filter()` to remove completed items

---

## ⚠️ Common Mistakes

### Mistake #1: Direct Mutations

```svelte
<script>
  let items = $state([1, 2, 3]);
  
  function addItem(value) {
    // ❌ WRONG - mutation doesn't trigger reactivity
    items.push(value);
  }
  
  function addItemCorrect(value) {
    // ✅ CORRECT - assignment triggers reactivity
    items = [...items, value];
  }
</script>
```

### Mistake #2: Mutating Nested Objects

```svelte
<script>
  let users = $state([
    { id: 1, name: 'Alice', age: 25 }
  ]);
  
  function updateAge(id, newAge) {
    // ❌ WRONG - mutating the object directly
    const user = users.find(u => u.id === id);
    user.age = newAge;
  }
  
  function updateAgeCorrect(id, newAge) {
    // ✅ CORRECT - creating new array with new object
    users = users.map(user =>
      user.id === id ? { ...user, age: newAge } : user
    );
  }
</script>
```

### Mistake #3: Sorting and Reversing

```svelte
<script>
  let numbers = $state([3, 1, 4, 1, 5]);
  
  function sortNumbers() {
    // ❌ WRONG - sort() mutates in place
    numbers.sort((a, b) => a - b);
  }
  
  function sortNumbersCorrect() {
    // ✅ CORRECT - create a copy first
    numbers = [...numbers].sort((a, b) => a - b);
    // Or: numbers = numbers.toSorted((a, b) => a - b); // New JS method
  }
</script>
```

### Mistake #4: Forgetting the Assignment

```svelte
<script>
  let items = $state([1, 2, 3]);
  
  function updateItems() {
    // ❌ WRONG - map creates new array but we don't assign it
    items.map(x => x * 2);
  }
  
  function updateItemsCorrect() {
    // ✅ CORRECT - assign the result
    items = items.map(x => x * 2);
  }
</script>
```

---

## ✅ Best Practices

### 1. Use the Spread Operator

The spread operator (`...`) is your best friend for immutable updates:

```svelte
<script>
  let items = $state([1, 2, 3]);
  
  // Adding items
  items = [...items, 4]; // Add to end
  items = [0, ...items]; // Add to beginning
  items = [...items.slice(0, 2), 99, ...items.slice(2)]; // Insert at index
  
  // Combining arrays
  let more = [4, 5, 6];
  items = [...items, ...more];
</script>
```

### 2. Use Array Methods That Return New Arrays

These methods create new arrays and are perfect for reactive updates:

```svelte
<script>
  let numbers = $state([1, 2, 3, 4, 5]);
  
  // ✅ These all return new arrays
  numbers = numbers.filter(n => n > 2);    // [3, 4, 5]
  numbers = numbers.map(n => n * 2);       // [6, 8, 10]
  numbers = numbers.slice(0, 2);           // [6, 8]
  numbers = numbers.concat([12, 14]);      // [6, 8, 12, 14]
  numbers = [...numbers].reverse();        // [14, 12, 8, 6]
</script>
```

### 3. Use the `.with()` Method (ES2023)

Modern JavaScript has a `.with()` method for updating array items:

```svelte
<script>
  let items = $state(['a', 'b', 'c']);
  
  // Update item at index 1
  items = items.with(1, 'B'); // ['a', 'B', 'c']
  
  // This is cleaner than:
  items = [...items.slice(0, 1), 'B', ...items.slice(2)];
</script>
```

### 4. Create Helper Functions

For complex updates, create reusable helper functions:

```svelte
<script>
  let todos = $state([]);
  
  // Helper function for updating items
  function updateItem(array, id, updates) {
    return array.map(item =>
      item.id === id ? { ...item, ...updates } : item
    );
  }
  
  // Helper function for removing items
  function removeItem(array, id) {
    return array.filter(item => item.id !== id);
  }
  
  // Usage
  function completeTodo(id) {
    todos = updateItem(todos, id, { completed: true });
  }
  
  function deleteTodo(id) {
    todos = removeItem(todos, id);
  }
</script>
```

### 5. Combine Derived Values for Complex Filtering

```svelte
<script>
  let products = $state([
    { id: 1, name: 'Laptop', price: 999, category: 'electronics' },
    { id: 2, name: 'Mouse', price: 29, category: 'electronics' },
    { id: 3, name: 'Desk', price: 299, category: 'furniture' }
  ]);
  
  let searchTerm = $state('');
  let selectedCategory = $state('all');
  let maxPrice = $state(1000);
  
  // Chain filters with derived
  let filteredProducts = $derived(() => {
    let result = products;
    
    // Filter by search term
    if (searchTerm) {
      result = result.filter(p => 
        p.name.toLowerCase().includes(searchTerm.toLowerCase())
      );
    }
    
    // Filter by category
    if (selectedCategory !== 'all') {
      result = result.filter(p => p.category === selectedCategory);
    }
    
    // Filter by price
    result = result.filter(p => p.price <= maxPrice);
    
    return result;
  });
  
  // Derived statistics
  let averagePrice = $derived(
    filteredProducts.length > 0
      ? filteredProducts.reduce((sum, p) => sum + p.price, 0) / filteredProducts.length
      : 0
  );
</script>
```

---

## 🎨 When to Use `$state.raw()`

For arrays of primitives where you don't need deep reactivity, `$state.raw()` can be more efficient:

```svelte
<script>
  // For simple arrays of numbers/strings
  let tags = $state.raw(['javascript', 'svelte', 'web']);
  
  // You still need immutable updates
  tags = [...tags, 'runes'];
  
  // When to use $state.raw():
  // ✅ Array of primitives (numbers, strings, booleans)
  // ✅ You don't need nested reactivity
  // ✅ Performance matters (large lists)
  
  // When to use $state():
  // ✅ Array of objects
  // ✅ Nested data structures
  // ✅ Default choice for most cases
</script>
```

**Performance comparison:**

```svelte
<script>
  // $state() - Deep reactivity (slight overhead)
  let items1 = $state([1, 2, 3, 4, 5]); // Wrapped in Proxy
  
  // $state.raw() - No deep reactivity (faster)
  let items2 = $state.raw([1, 2, 3, 4, 5]); // Plain array
  
  // Both still require reassignment for updates
  items1 = [...items1, 6];
  items2 = [...items2, 6];
</script>
```

For most applications, the performance difference is negligible. Use `$state()` by default and optimize with `$state.raw()` only if you measure a performance issue.

---

## 🧠 Mental Model: Arrays as Snapshots

Think of reactive arrays as **snapshots in time**. Each assignment creates a new snapshot:

```svelte
<script>
  let items = $state([1, 2, 3]);
  
  // Snapshot 1: [1, 2, 3]
  items = [...items, 4];
  
  // Snapshot 2: [1, 2, 3, 4]
  items = items.filter(x => x > 2);
  
  // Snapshot 3: [3, 4]
</script>
```

When you assign a new array, you're creating a new snapshot. Svelte compares the old snapshot to the new one and updates the UI accordingly. This is why mutations don't work—they modify the existing snapshot without creating a new one.

**The immutable mindset:**
- Don't modify arrays in place
- Create new arrays with your changes
- Trust that Svelte will efficiently update the DOM
- The old arrays are garbage collected automatically

---

## 📖 Quick Reference

### Adding Items

```javascript
// Add to end
items = [...items, newItem];

// Add to beginning
items = [newItem, ...items];

// Add at specific index
items = [...items.slice(0, index), newItem, ...items.slice(index)];

// Add multiple items
items = [...items, ...moreItems];
```

### Removing Items

```javascript
// Remove by index
items = items.filter((_, i) => i !== index);

// Remove by value
items = items.filter(item => item !== value);

// Remove by condition
items = items.filter(item => item.id !== idToRemove);

// Remove first item
items = items.slice(1);

// Remove last item
items = items.slice(0, -1);
```

### Updating Items

```javascript
// Update by index
items = items.with(index, newValue); // ES2023

// Update by condition
items = items.map(item =>
  item.id === targetId ? { ...item, ...updates } : item
);

// Update multiple properties
items = items.map(item =>
  item.id === targetId 
    ? { ...item, name: newName, price: newPrice }
    : item
);
```

### Sorting and Reversing

```javascript
// Sort (create copy first!)
items = [...items].sort((a, b) => a - b);
items = items.toSorted((a, b) => a - b); // ES2023, creates copy automatically

// Reverse (create copy first!)
items = [...items].reverse();
items = items.toReversed(); // ES2023, creates copy automatically
```

### Derived Values from Arrays

```javascript
// Count
let count = $derived(items.length);

// Sum
let total = $derived(items.reduce((sum, item) => sum + item.price, 0));

// Filter
let completed = $derived(items.filter(item => item.completed));

// Find
let firstActive = $derived(items.find(item => !item.completed));

// Check if any/all
let hasActive = $derived(items.some(item => !item.completed));
let allCompleted = $derived(items.every(item => item.completed));
```

---

## 🏋️ Practice Exercise: Shopping Cart

Build a shopping cart with these features:

1. **Add products** with name, price, and quantity
2. **Update quantity** of items in cart (increase/decrease)
3. **Remove items** from cart
4. **Calculate totals**:
   - Subtotal (sum of all items)
   - Tax (8% of subtotal)
   - Total (subtotal + tax)
5. **Apply discount code**:
   - "SAVE10" for 10% off
   - "SAVE20" for 20% off
6. **Show statistics**:
   - Total items in cart
   - Number of unique products
7. **Sort options**:
   - Sort by name (A-Z)
   - Sort by price (low to high)

**Starter template:**

```svelte
<script>
  let cartItems = $state([
    { id: 1, name: 'Laptop', price: 999, quantity: 1 },
    { id: 2, name: 'Mouse', price: 29, quantity: 2 }
  ]);
  
  let discountCode = $state('');
  let sortBy = $state('name'); // 'name' or 'price'
  
  // TODO: Implement derived values
  let subtotal = $derived(/* calculate subtotal */);
  let tax = $derived(/* calculate 8% tax */);
  let discount = $derived(/* calculate discount based on code */);
  let total = $derived(/* calculate final total */);
  
  // TODO: Implement functions
  function updateQuantity(id, change) {
    // Update quantity by change amount (+1 or -1)
  }
  
  function removeItem(id) {
    // Remove item from cart
  }
  
  function sortedItems() {
    // Return sorted array based on sortBy
  }
</script>
```

**Challenge additions:**
- Save cart to localStorage (using `$effect()`)
- Add "clear cart" button
- Minimum quantity of 1
- Show original price with strikethrough when discount is applied

---

## 🚀 Next Steps

Congratulations! You now understand how to work with reactive arrays in Svelte 5. You've learned:

- Why mutations don't trigger reactivity
- Immutable update patterns for all common operations
- How to create derived values from array data
- Best practices for clean, maintainable code

**In Article 7: "Objects - Structured State"**, we'll tackle reactive objects and learn:
- How to update object properties safely
- Dealing with nested objects
- Partial updates and spread operators
- When to use `$state.raw()` for objects
- Building a user profile editor

The patterns you learned here with arrays will serve as the foundation for working with complex object structures. See you in the next article!

---

## 📚 Additional Resources

- [MDN: Array Methods](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array)
- [Svelte 5 Docs: $state](https://svelte-5-preview.vercel.app/docs/runes#$state)
- [JavaScript Array Methods That Don't Mutate](https://doesitmutate.xyz/)
- [Immutability in JavaScript](https://developer.mozilla.org/en-US/docs/Glossary/Immutable)

---

*This article is part of the "Mastering Svelte 5 Runes" series. Found this helpful? Check out the complete series roadmap for more in-depth guides.*
