# Article 8: Arrays of Objects - Real-World Data

## Managing Complex Collections in Svelte 5

---

## 🎣 Hook

You've mastered reactive arrays of primitives and reactive objects. But here's the truth: real applications don't deal with simple lists of numbers or isolated user profiles. They manage *collections* of *complex data*—arrays of users, products, todos, comments, orders, and more. Each item has multiple properties, relationships, and states. This is where most developers hit a wall with Svelte 5's reactivity.

Consider a user management dashboard. You need to display users, edit their roles, toggle their active status, delete them, and show statistics—all while keeping the UI perfectly in sync. Update one user's role and the admin count should change. Delete a user and the pagination should adjust. Toggle a status and the filtered view should update. This is the juggling act of reactive arrays of objects.

The challenge isn't just keeping things reactive—it's doing so *efficiently* and *readably*. Deeply nested updates, maintaining immutability, avoiding unnecessary re-renders, and keeping your code maintainable. In this article, we'll master the patterns that power real-world applications, from simple updates to complex derived statistics, and build a complete user management dashboard that handles everything gracefully.

---

## 📚 Learning Goals

By the end of this article, you will:

- ✅ Create and manage reactive arrays of objects with `$state()`
- ✅ Update specific items using immutable patterns
- ✅ Add and remove items without breaking reactivity
- ✅ Derive statistics and filtered views from collections
- ✅ Build helper functions to reduce boilerplate
- ✅ Understand when to normalize data vs keep it nested
- ✅ Handle complex real-world scenarios efficiently
- ✅ Avoid common performance pitfalls

---

## 📋 Prerequisites

Before diving in, make sure you're comfortable with:

- **Article 2**: `$state()` for reactive values
- **Article 3**: `$derived()` for computed values
- **Article 6**: Reactive arrays and immutable updates
- **Article 7**: Reactive objects and nested updates
- JavaScript array methods: `map()`, `filter()`, `find()`
- Spread operators and object destructuring

---

## 🎓 Core Concept: Arrays of Objects Are Everywhere

### The Real-World Pattern

Almost every application manages collections of structured data:

```javascript
// E-commerce
const products = $state([
  { id: 1, name: 'Laptop', price: 999, inStock: true, category: 'electronics' },
  { id: 2, name: 'Mouse', price: 29, inStock: false, category: 'electronics' }
]);

// Social media
const posts = $state([
  { id: 1, author: 'Alice', content: 'Hello!', likes: 5, comments: [] },
  { id: 2, author: 'Bob', content: 'Great day', likes: 12, comments: [] }
]);

// Project management
const tasks = $state([
  { id: 1, title: 'Design', assignee: 'Alice', status: 'done', priority: 'high' },
  { id: 2, title: 'Code', assignee: 'Bob', status: 'in-progress', priority: 'high' }
]);
```

Each item is an object with multiple properties. The array gives you ordered collection semantics, while each object provides rich, structured data.

### The Reactivity Challenge

Here's what makes this tricky in Svelte 5:

**1. Deep Updates Must Be Immutable**

```javascript
// ❌ WRONG - Mutation doesn't trigger reactivity
users[0].role = 'admin';

// ✅ CORRECT - Create new array with new object
users = users.map((user, i) => 
  i === 0 ? { ...user, role: 'admin' } : user
);
```

**2. You Need to Update the RIGHT Item**

```javascript
// Update user with specific ID
users = users.map(user =>
  user.id === targetId 
    ? { ...user, role: 'admin' }
    : user
);
```

**3. Statistics Must Derive from the Collection**

```javascript
// Count admins whenever users change
const adminCount = $derived(
  users.filter(u => u.role === 'admin').length
);
```

### The Mental Model

Think of your array of objects as a **reactive spreadsheet**:

- Each row is an object (a user, product, etc.)
- Each column is a property (name, price, status)
- When you update a cell, you create a new row
- When you add/remove rows, you create a new spreadsheet
- Statistics (sums, counts) recalculate automatically

This immutable update pattern ensures Svelte can detect changes and update the DOM efficiently.

---

## 💻 Live Example: User Management Dashboard

Let's build a real user management system that handles everything:

```svelte
<script>
  // Initial user data
  let users = $state([
    { id: 1, name: 'Alice Johnson', email: 'alice@example.com', role: 'admin', active: true },
    { id: 2, name: 'Bob Smith', email: 'bob@example.com', role: 'user', active: true },
    { id: 3, name: 'Carol White', email: 'carol@example.com', role: 'user', active: false },
    { id: 4, name: 'David Brown', email: 'david@example.com', role: 'moderator', active: true }
  ]);

  // Filter state
  let roleFilter = $state('all');
  let statusFilter = $state('all');
  let searchQuery = $state('');

  // Derived statistics
  const totalUsers = $derived(users.length);
  const activeUsers = $derived(users.filter(u => u.active).length);
  const adminCount = $derived(users.filter(u => u.role === 'admin').length);
  const moderatorCount = $derived(users.filter(u => u.role === 'moderator').length);

  // Derived filtered list
  const filteredUsers = $derived(() => {
    let filtered = users;

    // Filter by role
    if (roleFilter !== 'all') {
      filtered = filtered.filter(u => u.role === roleFilter);
    }

    // Filter by status
    if (statusFilter !== 'all') {
      const isActive = statusFilter === 'active';
      filtered = filtered.filter(u => u.active === isActive);
    }

    // Filter by search query
    if (searchQuery.trim()) {
      const query = searchQuery.toLowerCase();
      filtered = filtered.filter(u =>
        u.name.toLowerCase().includes(query) ||
        u.email.toLowerCase().includes(query)
      );
    }

    return filtered;
  });

  // Add new user
  function addUser() {
    const newUser = {
      id: Math.max(...users.map(u => u.id), 0) + 1,
      name: 'New User',
      email: 'newuser@example.com',
      role: 'user',
      active: true
    };
    users = [...users, newUser];
  }

  // Update user property
  function updateUser(id, updates) {
    users = users.map(user =>
      user.id === id
        ? { ...user, ...updates }
        : user
    );
  }

  // Toggle active status
  function toggleActive(id) {
    users = users.map(user =>
      user.id === id
        ? { ...user, active: !user.active }
        : user
    );
  }

  // Change role
  function changeRole(id, newRole) {
    users = users.map(user =>
      user.id === id
        ? { ...user, role: newRole }
        : user
    );
  }

  // Delete user
  function deleteUser(id) {
    users = users.filter(user => user.id !== id);
  }

  // Bulk operations
  function activateAll() {
    users = users.map(user => ({ ...user, active: true }));
  }

  function deactivateAll() {
    users = users.map(user => ({ ...user, active: false }));
  }
</script>

<!-- Statistics Dashboard -->
<div class="stats">
  <div class="stat">
    <h3>Total Users</h3>
    <p>{totalUsers}</p>
  </div>
  <div class="stat">
    <h3>Active</h3>
    <p>{activeUsers}</p>
  </div>
  <div class="stat">
    <h3>Admins</h3>
    <p>{adminCount}</p>
  </div>
  <div class="stat">
    <h3>Moderators</h3>
    <p>{moderatorCount}</p>
  </div>
</div>

<!-- Filters -->
<div class="filters">
  <input
    type="text"
    placeholder="Search users..."
    bind:value={searchQuery}
  />

  <select bind:value={roleFilter}>
    <option value="all">All Roles</option>
    <option value="admin">Admin</option>
    <option value="moderator">Moderator</option>
    <option value="user">User</option>
  </select>

  <select bind:value={statusFilter}>
    <option value="all">All Status</option>
    <option value="active">Active</option>
    <option value="inactive">Inactive</option>
  </select>
</div>

<!-- Actions -->
<div class="actions">
  <button onclick={addUser}>Add User</button>
  <button onclick={activateAll}>Activate All</button>
  <button onclick={deactivateAll}>Deactivate All</button>
</div>

<!-- User List -->
<div class="user-list">
  {#each filteredUsers as user (user.id)}
    <div class="user-card">
      <div class="user-info">
        <input
          type="text"
          value={user.name}
          onchange={(e) => updateUser(user.id, { name: e.target.value })}
        />
        <input
          type="email"
          value={user.email}
          onchange={(e) => updateUser(user.id, { email: e.target.value })}
        />
      </div>

      <div class="user-controls">
        <select
          value={user.role}
          onchange={(e) => changeRole(user.id, e.target.value)}
        >
          <option value="user">User</option>
          <option value="moderator">Moderator</option>
          <option value="admin">Admin</option>
        </select>

        <label>
          <input
            type="checkbox"
            checked={user.active}
            onchange={() => toggleActive(user.id)}
          />
          Active
        </label>

        <button onclick={() => deleteUser(user.id)}>Delete</button>
      </div>
    </div>
  {/each}
</div>

<style>
  .stats {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 1rem;
    margin-bottom: 2rem;
  }

  .stat {
    background: #f5f5f5;
    padding: 1rem;
    border-radius: 4px;
    text-align: center;
  }

  .stat h3 {
    margin: 0 0 0.5rem 0;
    font-size: 0.875rem;
    color: #666;
  }

  .stat p {
    margin: 0;
    font-size: 2rem;
    font-weight: bold;
  }

  .filters, .actions {
    display: flex;
    gap: 1rem;
    margin-bottom: 1rem;
  }

  .filters input, .filters select {
    padding: 0.5rem;
    border: 1px solid #ddd;
    border-radius: 4px;
  }

  .user-list {
    display: flex;
    flex-direction: column;
    gap: 1rem;
  }

  .user-card {
    border: 1px solid #ddd;
    border-radius: 4px;
    padding: 1rem;
    display: flex;
    justify-content: space-between;
    align-items: center;
  }

  .user-info {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    flex: 1;
  }

  .user-info input {
    padding: 0.25rem 0.5rem;
    border: 1px solid #ddd;
    border-radius: 4px;
  }

  .user-controls {
    display: flex;
    gap: 1rem;
    align-items: center;
  }

  button {
    padding: 0.5rem 1rem;
    background: #007bff;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
  }

  button:hover {
    background: #0056b3;
  }
</style>
```

### Breaking Down the Pattern

**1. Statistics with $derived()**

```javascript
const adminCount = $derived(
  users.filter(u => u.role === 'admin').length
);
```

Every time `users` changes, the count recalculates automatically. No manual tracking needed.

**2. Complex Filtering with $derived.by()**

```javascript
const filteredUsers = $derived(() => {
  let filtered = users;
  
  if (roleFilter !== 'all') {
    filtered = filtered.filter(u => u.role === roleFilter);
  }
  
  // ... more filters
  
  return filtered;
});
```

Multiple dependencies (`users`, `roleFilter`, `statusFilter`, `searchQuery`) are tracked automatically. The filtered list updates when any dependency changes.

**3. Update Pattern: Map with Conditional Spread**

```javascript
function updateUser(id, updates) {
  users = users.map(user =>
    user.id === id
      ? { ...user, ...updates }  // Create new object with updates
      : user                     // Keep existing object
  );
}
```

This pattern:
- Finds the right item by ID
- Creates a new object with the updates
- Leaves other items unchanged
- Creates a new array (triggers reactivity)

**4. Toggle Pattern**

```javascript
function toggleActive(id) {
  users = users.map(user =>
    user.id === id
      ? { ...user, active: !user.active }
      : user
  );
}
```

Read the current value, flip it, and create a new object—all in one pass.

**5. Delete Pattern: Filter**

```javascript
function deleteUser(id) {
  users = users.filter(user => user.id !== id);
}
```

`filter()` already creates a new array, so this is naturally reactive.

---

## ⚠️ Common Mistakes

### Mistake 1: Direct Mutation

```javascript
// ❌ WRONG - Doesn't trigger reactivity
function updateUser(id, updates) {
  const user = users.find(u => u.id === id);
  user.name = updates.name;
  user.role = updates.role;
}
```

**Why it fails**: You're mutating the existing object. Svelte doesn't detect this change.

**Fix**: Use the map pattern to create a new array with a new object.

### Mistake 2: Mutating Before Assigning

```javascript
// ❌ WRONG - Mutates first
function addUser(newUser) {
  users.push(newUser);
  users = users;  // Too late!
}
```

**Why it fails**: The mutation happens before the assignment. Svelte sees the same array reference.

**Fix**: Create a new array first: `users = [...users, newUser]`

### Mistake 3: Forgetting Keys in {#each}

```svelte
<!-- ❌ WRONG - No key -->
{#each users as user}
  <UserCard {user} />
{/each}

<!-- ✅ CORRECT - Use unique ID -->
{#each users as user (user.id)}
  <UserCard {user} />
{/each}
```

**Why it matters**: Without keys, Svelte can't track which items changed, leading to incorrect updates and lost state.

### Mistake 4: Over-Deriving

```javascript
// ❌ WRONG - Derives from derived
const activeUsers = $derived(users.filter(u => u.active));
const activeUserCount = $derived(activeUsers.length);
const hasActiveUsers = $derived(activeUserCount > 0);
```

**Why it's wrong**: Three derived values when you only need one or two. Each adds overhead.

**Fix**: Combine when possible:

```javascript
const activeUserCount = $derived(users.filter(u => u.active).length);
const hasActiveUsers = $derived(activeUserCount > 0);
```

### Mistake 5: Inefficient Filtering in Templates

```svelte
<!-- ❌ WRONG - Filters on every render -->
{#each users.filter(u => u.active) as user}
  ...
{/each}

<!-- ✅ CORRECT - Filter once with $derived -->
<script>
  const activeUsers = $derived(users.filter(u => u.active));
</script>

{#each activeUsers as user}
  ...
{/each}
```

### Mistake 6: Searching Without Normalization

```javascript
// ❌ WRONG - Case-sensitive search
const filtered = $derived(
  users.filter(u => u.name.includes(searchQuery))
);

// ✅ CORRECT - Normalize case
const filtered = $derived(
  users.filter(u =>
    u.name.toLowerCase().includes(searchQuery.toLowerCase())
  )
);
```

---

## ✨ Best Practices

### 1. Use Helper Functions for Common Operations

```javascript
// Define reusable update patterns
function updateItem(array, id, updates) {
  return array.map(item =>
    item.id === id ? { ...item, ...updates } : item
  );
}

function removeItem(array, id) {
  return array.filter(item => item.id !== id);
}

function addItem(array, newItem) {
  return [...array, newItem];
}

// Use them
users = updateItem(users, userId, { role: 'admin' });
users = removeItem(users, userId);
users = addItem(users, newUser);
```

### 2. Batch Multiple Updates

```javascript
// ❌ LESS EFFICIENT - Multiple updates
users = updateUser(users, id1, { active: true });
users = updateUser(users, id2, { active: true });
users = updateUser(users, id3, { active: true });

// ✅ BETTER - Single pass
users = users.map(user =>
  [id1, id2, id3].includes(user.id)
    ? { ...user, active: true }
    : user
);
```

### 3. Use $derived.by() for Complex Logic

```javascript
const filteredAndSorted = $derived.by(() => {
  // Step 1: Filter
  let result = users.filter(u => u.active);
  
  // Step 2: Sort
  result.sort((a, b) => a.name.localeCompare(b.name));
  
  // Step 3: Group
  const grouped = result.reduce((acc, user) => {
    acc[user.role] = acc[user.role] || [];
    acc[user.role].push(user);
    return acc;
  }, {});
  
  return grouped;
});
```

### 4. Normalize Data When Needed

**Nested (Good for Simple Cases)**

```javascript
const users = $state([
  { id: 1, name: 'Alice', posts: [
    { id: 1, title: 'Hello' },
    { id: 2, title: 'World' }
  ]}
]);
```

**Normalized (Better for Complex Cases)**

```javascript
const users = $state([
  { id: 1, name: 'Alice', postIds: [1, 2] }
]);

const posts = $state([
  { id: 1, authorId: 1, title: 'Hello' },
  { id: 2, authorId: 1, title: 'World' }
]);

// Derive relationships
const userWithPosts = $derived(id => {
  const user = users.find(u => u.id === id);
  const userPosts = posts.filter(p => p.authorId === id);
  return { ...user, posts: userPosts };
});
```

**When to normalize:**
- Data is deeply nested (3+ levels)
- Many-to-many relationships
- Same data referenced in multiple places
- Need to update one entity without touching others

### 5. Memoize Expensive Computations

```javascript
// If computing stats is expensive, derive only what you need
const stats = $derived(() => {
  const active = users.filter(u => u.active);
  const admins = users.filter(u => u.role === 'admin');
  
  return {
    total: users.length,
    active: active.length,
    admins: admins.length,
    inactiveAdmins: admins.filter(u => !u.active).length
  };
});

// Then access: stats.total, stats.active, etc.
```

### 6. Type Your Data (TypeScript)

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  role: 'admin' | 'moderator' | 'user';
  active: boolean;
}

let users = $state<User[]>([]);

function updateUser(id: number, updates: Partial<User>) {
  users = users.map(user =>
    user.id === id ? { ...user, ...updates } : user
  );
}
```

---

## 🧠 Mental Model

### The Immutable Update Dance

Think of updating arrays of objects as a **three-step dance**:

1. **Find**: Locate the item(s) you want to change
2. **Clone**: Create new objects with the changes
3. **Assign**: Create a new array and assign it

```javascript
users = users.map(user =>        // Create new array
  user.id === targetId            // Find the item
    ? { ...user, role: 'admin' }  // Clone with changes
    : user                        // Keep unchanged
);
```

### The Derived Cascade

Your derived values form a **dependency tree**:

```
users (state)
  ├─> totalUsers (derived)
  ├─> activeUsers (derived)
  │     └─> hasActiveUsers (derived)
  └─> filteredUsers (derived)
        ├─> filteredCount (derived)
        └─> [Template renders]
```

Changes flow down the tree automatically. Change `users` at the root, and everything updates.

### The Reactivity Checklist

Before any update, ask:

1. **Am I creating a NEW array?** (Yes: `= [...]`, `= map()`, `= filter()`)
2. **Am I creating NEW objects?** (Yes: `{ ...user, ... }`)
3. **Am I assigning to the variable?** (Yes: `users = ...`)

All three must be true for reactivity to work.

---

## 📖 Quick Reference

### Update Patterns

```javascript
// Update one item
items = items.map(item =>
  item.id === id ? { ...item, ...updates } : item
);

// Update multiple items
items = items.map(item =>
  ids.includes(item.id) ? { ...item, ...updates } : item
);

// Update with condition
items = items.map(item =>
  condition(item) ? { ...item, ...updates } : item
);

// Toggle boolean
items = items.map(item =>
  item.id === id ? { ...item, active: !item.active } : item
);

// Update nested property
items = items.map(item =>
  item.id === id 
    ? { ...item, settings: { ...item.settings, theme: 'dark' } }
    : item
);
```

### Add/Remove Patterns

```javascript
// Add to end
items = [...items, newItem];

// Add to beginning
items = [newItem, ...items];

// Add at index
items = [...items.slice(0, index), newItem, ...items.slice(index)];

// Remove by ID
items = items.filter(item => item.id !== id);

// Remove multiple
items = items.filter(item => !idsToRemove.includes(item.id));

// Replace entire array
items = newItems;

// Clear array
items = [];
```

### Derived Patterns

```javascript
// Count matching items
const count = $derived(items.filter(condition).length);

// Find one item
const item = $derived(items.find(i => i.id === targetId));

// Check existence
const exists = $derived(items.some(i => i.id === targetId));

// All match condition
const allMatch = $derived(items.every(condition));

// Sum property
const total = $derived(items.reduce((sum, i) => sum + i.value, 0));

// Group by property
const grouped = $derived(
  items.reduce((acc, item) => {
    acc[item.category] = acc[item.category] || [];
    acc[item.category].push(item);
    return acc;
  }, {})
);
```

---

## 🏋️ Practice Exercise

Build a **Product Catalog** with these features:

### Requirements

1. **State**: Array of products with:
   - `id`, `name`, `price`, `category`, `inStock`, `rating`, `tags`

2. **Statistics** (all derived):
   - Total products
   - Products in stock
   - Average price
   - Products by category count

3. **Filtering**:
   - Search by name
   - Filter by category
   - Filter by stock status
   - Filter by price range
   - Filter by tags

4. **Operations**:
   - Add product
   - Edit product (name, price, etc.)
   - Toggle stock status
   - Delete product
   - Update rating

5. **Sorting**:
   - By name (A-Z, Z-A)
   - By price (low to high, high to low)
   - By rating

### Starter Code

```svelte
<script>
  let products = $state([
    {
      id: 1,
      name: 'Laptop',
      price: 999,
      category: 'electronics',
      inStock: true,
      rating: 4.5,
      tags: ['computer', 'portable']
    },
    // Add more products...
  ]);

  // Your filters
  let searchQuery = $state('');
  let categoryFilter = $state('all');
  let priceRange = $state({ min: 0, max: 10000 });

  // Your derived values
  const totalProducts = $derived(products.length);
  // ... more stats

  const filteredProducts = $derived.by(() => {
    // Your filtering logic
  });

  // Your functions
  function addProduct() { /* ... */ }
  function updateProduct(id, updates) { /* ... */ }
  function deleteProduct(id) { /* ... */ }
</script>

<!-- Your UI here -->
```

### Bonus Challenges

- Add "Recently Viewed" section (track last 5 viewed products)
- Implement shopping cart (separate array with product IDs and quantities)
- Add "Featured Products" toggle (filter to featured: true)
- Calculate total inventory value (price × quantity summed)

---

## 🚀 Next Steps

You've mastered arrays of objects—the foundation of real-world applications! You can now:

- ✅ Manage complex collections reactively
- ✅ Update nested data immutably
- ✅ Derive statistics and filtered views
- ✅ Build data-driven interfaces

In **Article 9: $bindable - Two-Way Binding**, we'll learn how to make component props reactive, enabling parent-child components to share and synchronize state seamlessly. You'll build reusable form controls like inputs, checkboxes, and selects that work like native HTML elements but with custom styling and behavior.

### Key Takeaway

**Arrays of objects require immutable updates at both levels**: create new arrays AND new objects. Use `map()` to update, `filter()` to remove, and spread operators to add. Derive everything you can—counts, filters, and statistics—so your UI stays perfectly in sync with your data.

---

## 📚 Further Reading

- [Svelte 5 Docs: Runes](https://svelte-5-preview.vercel.app/docs/runes)
- [MDN: Array Methods](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array)
- [Immutability Patterns in JavaScript](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax)

---

**Previous**: [Article 7 - Objects: Structured State](#)  
**Next**: [Article 9 - $bindable: Two-Way Binding](#)  
**Series**: [Mastering Svelte 5 Runes](#)
