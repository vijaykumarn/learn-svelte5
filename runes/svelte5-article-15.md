# Article 15: Class Fields & Runes - Object-Oriented Reactive Patterns

## Why Object-Oriented Reactivity Matters

If you've been working through this series, you've mastered using Runes with functions and plain objects. But what about classes? Many developers coming from frameworks like React (with class components) or working with established patterns like stores, services, or domain models might wonder: "Can I use Runes with my object-oriented code?"

The answer is a resounding yes. Svelte 5 Runes work beautifully with ES6 class fields, opening up powerful patterns for encapsulating reactive state and behavior together. This isn't just about supporting legacy code—there are legitimate architectural reasons to use classes: they provide natural boundaries for related state and logic, enable inheritance and polymorphism when needed, and offer familiar patterns for developers from OOP backgrounds.

In this article, you'll learn how to combine the best of object-oriented programming with Svelte 5's reactivity system. We'll build reactive classes that feel natural, perform well, and integrate seamlessly with your Svelte components. By the end, you'll know exactly when classes add value versus when simpler approaches suffice.

---

## Learning Goals

By the end of this article, you'll be able to:

- ✅ Use `$state` in class field declarations
- ✅ Create computed properties using getters with `$derived`
- ✅ Write methods that safely update reactive state
- ✅ Build reusable reactive store classes
- ✅ Understand when classes provide architectural benefits
- ✅ Avoid common pitfalls with class-based reactivity
- ✅ Decide between classes, plain objects, and functions

---

## Prerequisites

Before diving in, you should be comfortable with:

- **Article 2**: Understanding `$state()` for reactive primitives
- **Article 3**: Using `$derived()` for computed values
- **Article 7**: Working with reactive objects
- **JavaScript ES6 Classes**: Fields, methods, getters, constructors
- Basic TypeScript (helpful but not required)

---

## Core Concept: Runes as Class Fields

### The Fundamentals

In Svelte 5, you can declare Runes directly as class fields. Here's the basic syntax:

```javascript
class Counter {
  count = $state(0);
  
  double = $derived(this.count * 2);
  
  increment() {
    this.count++;
  }
}
```

When you create an instance of this class, each property backed by a Rune becomes reactive:

```svelte
<script>
  let counter = new Counter();
</script>

<p>Count: {counter.count}</p>
<p>Double: {counter.double}</p>
<button onclick={() => counter.increment()}>+1</button>
```

### How It Works Under the Hood

Each class field initialized with `$state()` or `$derived()` becomes a reactive property on that instance. The reactivity is tied to the specific instance, not the class prototype. This means:

1. **Instance-level reactivity**: Each instance has its own reactive state
2. **Automatic tracking**: Components using these properties automatically subscribe
3. **No manual subscriptions**: Unlike Svelte 4 stores, no `$:` or `.subscribe()` needed

### Mental Model: Classes as Reactive Containers

Think of a class with Runes as a **reactive capsule**:

```
┌─────────────────────────────────┐
│     Reactive Class Instance     │
├─────────────────────────────────┤
│  State ($state fields)          │
│    ↓                            │
│  Computed ($derived getters)    │
│    ↓                            │
│  Actions (methods)              │
│    ↓                            │
│  Effects (if needed)            │
└─────────────────────────────────┘
```

All the reactivity is contained within the instance, making it portable and testable.

---

## Live Example: Building a Reactive Store Class

Let's build a practical example: a generic `Store` class that manages any type of data with built-in history and validation.

### Basic Store Implementation

```javascript
class Store {
  #value = $state();
  #history = $state([]);
  #maxHistory = 10;
  
  constructor(initialValue, options = {}) {
    this.#value = initialValue;
    this.#history = [initialValue];
    this.#maxHistory = options.maxHistory ?? 10;
  }
  
  // Getter for current value
  get value() {
    return this.#value;
  }
  
  // Setter with history tracking
  set value(newValue) {
    this.#value = newValue;
    this.#history = [...this.#history, newValue].slice(-this.#maxHistory);
  }
  
  // Computed property
  get hasChanged() {
    return this.#history.length > 1;
  }
  
  // Method to update
  update(updater) {
    this.value = updater(this.#value);
  }
  
  // Method to reset
  reset() {
    if (this.#history.length > 0) {
      this.value = this.#history[0];
    }
  }
  
  // Method to undo
  undo() {
    if (this.#history.length > 1) {
      const previousValue = this.#history[this.#history.length - 2];
      this.#value = previousValue;
      this.#history = this.#history.slice(0, -1);
    }
  }
}
```

### Using the Store in a Component

```svelte
<script>
  let userStore = new Store({ name: '', email: '' });
  
  function updateName(event) {
    userStore.update(user => ({
      ...user,
      name: event.target.value
    }));
  }
  
  function updateEmail(event) {
    userStore.update(user => ({
      ...user,
      email: event.target.value
    }));
  }
</script>

<div class="form">
  <label>
    Name:
    <input value={userStore.value.name} oninput={updateName} />
  </label>
  
  <label>
    Email:
    <input value={userStore.value.email} oninput={updateEmail} />
  </label>
  
  {#if userStore.hasChanged}
    <button onclick={() => userStore.undo()}>Undo</button>
    <button onclick={() => userStore.reset()}>Reset</button>
  {/if}
</div>

<pre>{JSON.stringify(userStore.value, null, 2)}</pre>
```

### Advanced Store with Validation

Let's add validation and derived state:

```javascript
class ValidatedStore {
  #value = $state();
  #errors = $state([]);
  #validators = [];
  
  constructor(initialValue, validators = []) {
    this.#value = initialValue;
    this.#validators = validators;
    this.#validate();
  }
  
  get value() {
    return this.#value;
  }
  
  set value(newValue) {
    this.#value = newValue;
    this.#validate();
  }
  
  get errors() {
    return this.#errors;
  }
  
  get isValid() {
    return this.#errors.length === 0;
  }
  
  #validate() {
    this.#errors = this.#validators
      .map(validator => validator(this.#value))
      .filter(error => error !== null);
  }
  
  update(updater) {
    this.value = updater(this.#value);
  }
}

// Validator functions
const required = (value) => 
  !value || value.trim() === '' ? 'This field is required' : null;

const email = (value) => 
  !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value) ? 'Invalid email' : null;

const minLength = (min) => (value) =>
  value.length < min ? `Must be at least ${min} characters` : null;
```

### Using Validated Store

```svelte
<script>
  let emailStore = new ValidatedStore('', [required, email]);
  let passwordStore = new ValidatedStore('', [required, minLength(8)]);
  
  let canSubmit = $derived(emailStore.isValid && passwordStore.isValid);
</script>

<form>
  <div>
    <input 
      type="email"
      bind:value={emailStore.value}
      placeholder="Email"
    />
    {#each emailStore.errors as error}
      <span class="error">{error}</span>
    {/each}
  </div>
  
  <div>
    <input 
      type="password"
      bind:value={passwordStore.value}
      placeholder="Password"
    />
    {#each passwordStore.errors as error}
      <span class="error">{error}</span>
    {/each}
  </div>
  
  <button disabled={!canSubmit}>
    Submit
  </button>
</form>
```

---

## Common Mistakes

### ❌ Mistake 1: Forgetting `this` in Class Methods

```javascript
class Counter {
  count = $state(0);
  
  // WRONG: Doesn't update the instance property
  increment() {
    count++; // ReferenceError: count is not defined
  }
  
  // CORRECT: Use this to reference instance property
  increment() {
    this.count++;
  }
}
```

### ❌ Mistake 2: Using Arrow Functions That Lose `this` Context

```javascript
class Counter {
  count = $state(0);
  
  // WRONG: Arrow function in class field
  increment = () => {
    this.count++; // `this` might not be what you expect
  }
}

// When used in event handlers, this is fine:
// <button onclick={() => counter.increment()}>
// But if you pass the method directly:
// <button onclick={counter.increment}> // Can lose context
```

**Solution**: Use regular methods or bind explicitly:

```javascript
class Counter {
  count = $state(0);
  
  increment() {
    this.count++;
  }
  
  // Or if you need an arrow function for binding
  constructor() {
    this.boundIncrement = this.increment.bind(this);
  }
}
```

### ❌ Mistake 3: Mutating Reactive Objects Directly

```javascript
class TodoList {
  items = $state([]);
  
  // WRONG: Direct mutation
  addItem(todo) {
    this.items.push(todo); // Won't trigger reactivity properly
  }
  
  // CORRECT: Create new array
  addItem(todo) {
    this.items = [...this.items, todo];
  }
}
```

### ❌ Mistake 4: Using `$derived` Instead of Getters

```javascript
class User {
  firstName = $state('');
  lastName = $state('');
  
  // WRONG: Trying to use $derived as a field
  fullName = $derived(this.firstName + ' ' + this.lastName);
  // This might work but it's not idiomatic
  
  // CORRECT: Use a getter
  get fullName() {
    return this.firstName + ' ' + this.lastName;
  }
}
```

### ❌ Mistake 5: Sharing Class Instances Incorrectly

```javascript
// WRONG: Singleton pattern that breaks reactivity
class AppState {
  static instance = new AppState();
  user = $state(null);
}

// Each component gets the same instance, but reactivity
// might not propagate as expected across component boundaries
```

**Solution**: Use proper context or prop passing, or create a module-level reactive object instead.

---

## Best Practices

### ✅ Practice 1: Use Private Fields for Internal State

```javascript
class Counter {
  #count = $state(0);
  #step = 1;
  
  get count() {
    return this.#count;
  }
  
  get step() {
    return this.#step;
  }
  
  setStep(newStep) {
    this.#step = newStep;
  }
  
  increment() {
    this.#count += this.#step;
  }
  
  decrement() {
    this.#count -= this.#step;
  }
}
```

Benefits:
- Encapsulation: Control how state is accessed and modified
- Clear API: Only expose what consumers need
- Easier refactoring: Internal implementation can change

### ✅ Practice 2: Combine Reactive and Non-Reactive Properties

Not everything needs to be reactive:

```javascript
class DataTable {
  // Reactive: Changes trigger UI updates
  data = $state([]);
  sortColumn = $state('name');
  sortDirection = $state('asc');
  
  // Non-reactive: Configuration that doesn't change
  columns = ['name', 'email', 'age'];
  pageSize = 10;
  
  // Derived: Computed from reactive state
  get sortedData() {
    return [...this.data].sort((a, b) => {
      const aVal = a[this.sortColumn];
      const bVal = b[this.sortColumn];
      const direction = this.sortDirection === 'asc' ? 1 : -1;
      return aVal > bVal ? direction : -direction;
    });
  }
  
  setSort(column) {
    if (this.sortColumn === column) {
      this.sortDirection = this.sortDirection === 'asc' ? 'desc' : 'asc';
    } else {
      this.sortColumn = column;
      this.sortDirection = 'asc';
    }
  }
}
```

### ✅ Practice 3: Use TypeScript for Better DX

```typescript
interface User {
  id: string;
  name: string;
  email: string;
}

class UserStore {
  #user = $state<User | null>(null);
  #loading = $state(false);
  #error = $state<string | null>(null);
  
  get user(): User | null {
    return this.#user;
  }
  
  get loading(): boolean {
    return this.#loading;
  }
  
  get error(): string | null {
    return this.#error;
  }
  
  async fetchUser(id: string): Promise<void> {
    this.#loading = true;
    this.#error = null;
    
    try {
      const response = await fetch(`/api/users/${id}`);
      this.#user = await response.json();
    } catch (err) {
      this.#error = err instanceof Error ? err.message : 'Unknown error';
    } finally {
      this.#loading = false;
    }
  }
  
  clearUser(): void {
    this.#user = null;
  }
}
```

### ✅ Practice 4: Create Factory Functions for Complex Setup

```javascript
function createPaginatedList(initialItems = [], itemsPerPage = 10) {
  return new class {
    items = $state(initialItems);
    currentPage = $state(1);
    itemsPerPage = itemsPerPage;
    
    get totalPages() {
      return Math.ceil(this.items.length / this.itemsPerPage);
    }
    
    get paginatedItems() {
      const start = (this.currentPage - 1) * this.itemsPerPage;
      return this.items.slice(start, start + this.itemsPerPage);
    }
    
    nextPage() {
      if (this.currentPage < this.totalPages) {
        this.currentPage++;
      }
    }
    
    previousPage() {
      if (this.currentPage > 1) {
        this.currentPage--;
      }
    }
    
    goToPage(page) {
      if (page >= 1 && page <= this.totalPages) {
        this.currentPage = page;
      }
    }
    
    addItem(item) {
      this.items = [...this.items, item];
    }
  };
}

// Usage
let productList = createPaginatedList(products, 20);
```

---

## Mental Model: When to Use Classes

### Use Classes When You Have:

1. **Complex State + Behavior Together**
   - Multiple related pieces of state
   - Many methods that operate on that state
   - Clear encapsulation boundaries

2. **Reusable Logic Across Components**
   - Need multiple instances with isolated state
   - Want to pass instances as props
   - Testing benefits from instantiation

3. **Domain Models**
   - Representing business entities (User, Product, Order)
   - Validation and business rules
   - Rich behavior beyond just data

4. **State Machines**
   - Finite state management
   - Transition logic
   - Guard conditions

### Use Plain Objects/Functions When You Have:

1. **Simple State**
   - Just a few related values
   - Minimal logic

2. **One-Off Logic**
   - Used in only one component
   - No need for multiple instances

3. **Simple Computed Values**
   - Just deriving values from state
   - No complex methods needed

### Decision Tree

```
Need to group state + behavior?
├─ Yes → Do you need multiple instances?
│  ├─ Yes → Use a Class
│  └─ No → Use a plain object or function
└─ No → Use separate $state variables
```

---

## Quick Reference

### Class with Runes Cheat Sheet

```javascript
class ReactiveClass {
  // Reactive state (private)
  #count = $state(0);
  
  // Non-reactive configuration
  max = 100;
  
  // Constructor for initialization
  constructor(initialCount = 0) {
    this.#count = initialCount;
  }
  
  // Getter for reactive access
  get count() {
    return this.#count;
  }
  
  // Setter for controlled updates
  set count(value) {
    if (value <= this.max) {
      this.#count = value;
    }
  }
  
  // Computed property (getter)
  get doubled() {
    return this.#count * 2;
  }
  
  // Method that updates state
  increment() {
    this.count++;
  }
  
  // Method with parameters
  add(amount) {
    this.count += amount;
  }
  
  // Method that returns derived value
  isAbove(threshold) {
    return this.#count > threshold;
  }
}
```

### Common Patterns

| Pattern | When to Use | Example |
|---------|-------------|---------|
| Private fields + public getters | Controlled access | `#value = $state()` with `get value()` |
| Getters for computed | Derived values | `get fullName() { return ... }` |
| Methods for actions | State updates | `increment() { this.count++ }` |
| Constructor for setup | Initial state | `constructor(initial) { this.#x = $state(initial) }` |
| Factory functions | Complex instantiation | `function createStore(config) { return new Store(config) }` |

---

## Practice Exercise: Build a Shopping Cart Class

Create a `ShoppingCart` class that:

1. Stores items with quantity
2. Calculates total price
3. Applies discount codes
4. Tracks shipping cost
5. Provides order summary

### Requirements

```javascript
class ShoppingCart {
  // TODO: Add items array
  // TODO: Add discount code
  // TODO: Add shipping method
  
  // TODO: get subtotal
  // TODO: get discount
  // TODO: get shipping
  // TODO: get total
  
  // TODO: addItem(product, quantity)
  // TODO: removeItem(productId)
  // TODO: updateQuantity(productId, quantity)
  // TODO: applyDiscount(code)
  // TODO: setShipping(method)
  // TODO: clear()
}
```

### Starter Code

```svelte
<script>
  const products = [
    { id: 1, name: 'Widget', price: 29.99 },
    { id: 2, name: 'Gadget', price: 49.99 },
    { id: 3, name: 'Gizmo', price: 19.99 }
  ];
  
  const discountCodes = {
    'SAVE10': 0.10,
    'SAVE20': 0.20
  };
  
  let cart = new ShoppingCart();
</script>

<!-- Your component here -->
```

### Bonus Challenges

1. Add persistence to `localStorage`
2. Implement quantity limits per item
3. Add a `history` feature for undo/redo
4. Create a `CartItem` class for individual items
5. Add tax calculation based on location

---

## Next Steps

Congratulations! You now understand how to combine object-oriented programming with Svelte 5's reactivity system. You've seen that classes aren't just legacy patterns—they're powerful tools for organizing complex reactive logic.

**In Article 16: "State Machines with Runes"**, we'll take these concepts further and learn how to model complex application states using finite state machines. You'll discover how to prevent impossible states, manage complex transitions, and build rock-solid user experiences. We'll implement a multi-step wizard, form flows, and even integrate with libraries like XState.

### Additional Resources

- [Svelte 5 Docs: Runes](https://svelte.dev/docs/svelte/$state)
- [MDN: JavaScript Classes](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes)
- [Private Class Fields](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/Private_class_fields)

### Challenge Yourself

Before moving to the next article:
1. Refactor one of your existing components to use a reactive class
2. Build a `Timer` class with start/stop/reset functionality
3. Create a `Form` class that handles validation and submission
4. Implement a `DataCache` class with expiration logic

Happy coding! 🚀
