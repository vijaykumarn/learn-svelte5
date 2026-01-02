# $derived - Computed Values Done Right

## Why Computed Values Are Everywhere (And Why They're Hard)

You've built a shopping cart. You have items, quantities, and prices stored in `$state()`. Now you need to show the subtotal, apply a discount, calculate tax, and display the final total. Your first instinct might be to create more state variables and manually update them whenever something changes. But there's a problem: you now have **derived state**—values that depend on other values—and keeping them in sync becomes a nightmare.

This is one of the most common sources of bugs in UI applications. You update the quantity, but forget to recalculate the tax. You change a price, but the discount percentage is stale. In Svelte 4, you'd use reactive statements (`$: total = items.reduce(...)`), which worked but had mysterious rules about when they'd run. In Svelte 5, `$derived()` makes computed values explicit, predictable, and—most importantly—automatic.

Think of `$derived()` as a formula in a spreadsheet. When you write `=A1 + B1` in cell C1, the spreadsheet automatically recalculates C1 whenever A1 or B1 changes. You never have to manually update C1—it just stays synchronized. That's exactly what `$derived()` does for your reactive state.

---

## 🎯 Learning Goals

By the end of this article, you'll be able to:

- ✅ Understand what makes a value "derived" vs "independent state"
- ✅ Use `$derived()` for simple computed values
- ✅ Use `$derived.by()` for complex logic with multiple statements
- ✅ Chain derived values that depend on other derived values
- ✅ Know when to use derived vs inline expressions in templates
- ✅ Leverage automatic memoization for performance
- ✅ Avoid common pitfalls like side effects in derived values

---

## 📋 Prerequisites

Before diving in, you should:

- ✅ Be comfortable with `$state()` (Article 2)
- ✅ Understand basic JavaScript expressions and functions
- ✅ Know the difference between values and functions

---

## 🧠 Core Concept: What Is Derived State?

### The Two Types of State

In any application, you have two categories of values:

**1. Independent State (Source of Truth)**
Values that exist on their own and change through user actions or external events:
```javascript
let count = $state(5);
let price = $state(29.99);
let isEnabled = $state(true);
```

**2. Derived State (Computed from Other State)**
Values that are calculated from independent state and should update automatically:
```javascript
let doubled = $derived(count * 2);  // Always count × 2
let displayPrice = $derived(`$${price.toFixed(2)}`);  // Always formatted price
let status = $derived(isEnabled ? 'Active' : 'Disabled');  // Always reflects isEnabled
```

The golden rule: **If you can compute it from other state, it should be derived, not stored.**

### Why Not Just Use More $state()?

You might be tempted to do this:

```javascript
// ❌ BAD: Storing derived values as state
let count = $state(5);
let doubled = $state(count * 2);  // Starts at 10

function increment() {
    count++;
    doubled = count * 2;  // Must manually update!
}
```

This has three problems:
1. **Fragile**: You must remember to update `doubled` everywhere `count` changes
2. **Bug-prone**: It's easy to forget, causing stale data
3. **Redundant**: You're storing information that already exists

With `$derived()`, the synchronization is automatic:

```javascript
// ✅ GOOD: Derived values stay in sync automatically
let count = $state(5);
let doubled = $derived(count * 2);  // Always correct

function increment() {
    count++;  // doubled updates automatically!
}
```

---

## 💡 $derived() - Simple Expressions

Use `$derived()` for straightforward computations that fit in a single expression:

```svelte
<script>
    let quantity = $state(3);
    let price = $state(19.99);
    
    // Simple derived values
    let subtotal = $derived(quantity * price);
    let total = $derived(subtotal * 1.1);  // Add 10% tax
    let formattedTotal = $derived(`$${total.toFixed(2)}`);
</script>

<div>
    <input type="number" bind:value={quantity} min="1" />
    <p>Price: ${price}</p>
    <p>Subtotal: ${subtotal.toFixed(2)}</p>
    <p>Total (with tax): {formattedTotal}</p>
</div>
```

**Key insight**: You write the formula once, and Svelte tracks all dependencies automatically. Change `quantity` or `price`, and everything recalculates.

### What Gets Tracked?

Svelte automatically tracks any `$state()` or `$derived()` value you **read** inside the `$derived()` expression:

```javascript
let a = $state(1);
let b = $state(2);
let c = $state(3);

let sum = $derived(a + b);  // Tracks 'a' and 'b', not 'c'
```

If `a` or `b` changes, `sum` recalculates. If `c` changes, `sum` doesn't care—it's not a dependency.

---

## 🔧 $derived.by() - Complex Logic

When your computation needs multiple statements, control flow, or local variables, use `$derived.by()`:

```svelte
<script>
    let scores = $state([85, 92, 78, 95, 88]);
    
    // Complex computation with multiple steps
    let stats = $derived.by(() => {
        const total = scores.reduce((sum, score) => sum + score, 0);
        const average = total / scores.length;
        const highest = Math.max(...scores);
        const lowest = Math.min(...scores);
        
        return {
            average: average.toFixed(1),
            highest,
            lowest,
            total
        };
    });
</script>

<div>
    <p>Average: {stats.average}</p>
    <p>Highest: {stats.highest}</p>
    <p>Lowest: {stats.lowest}</p>
    <p>Total: {stats.total}</p>
</div>
```

**When to use `$derived.by()`:**
- You need multiple intermediate variables
- You need if/else logic or loops
- You want to return an object or array with calculated properties
- The computation is too complex for a single expression

---

## 🔗 Chaining Derived Values

Derived values can depend on other derived values, creating a dependency chain:

```svelte
<script>
    let items = $state([
        { name: 'Laptop', price: 999, quantity: 1 },
        { name: 'Mouse', price: 29, quantity: 2 },
        { name: 'Keyboard', price: 79, quantity: 1 }
    ]);
    
    let discountCode = $state('SAVE10');
    
    // Level 1: Calculate subtotal from items
    let subtotal = $derived(
        items.reduce((sum, item) => sum + (item.price * item.quantity), 0)
    );
    
    // Level 2: Calculate discount from subtotal and code
    let discountAmount = $derived.by(() => {
        if (discountCode === 'SAVE10') return subtotal * 0.10;
        if (discountCode === 'SAVE20') return subtotal * 0.20;
        return 0;
    });
    
    // Level 3: Calculate total after discount
    let discountedTotal = $derived(subtotal - discountAmount);
    
    // Level 4: Add tax to get final total
    let tax = $derived(discountedTotal * 0.08);
    let finalTotal = $derived(discountedTotal + tax);
</script>

<div>
    <p>Subtotal: ${subtotal.toFixed(2)}</p>
    <p>Discount: -${discountAmount.toFixed(2)}</p>
    <p>Tax: ${tax.toFixed(2)}</p>
    <p><strong>Total: ${finalTotal.toFixed(2)}</strong></p>
</div>
```

**How it works:**
1. Change `items` → `subtotal` updates
2. `subtotal` changes → `discountAmount` updates
3. `discountAmount` changes → `discountedTotal` updates
4. `discountedTotal` changes → `tax` and `finalTotal` update

Each derived value recalculates only when its direct dependencies change. This creates an efficient update cascade.

---

## 🎨 Live Example: Shopping Cart with Discounts

Let's build a complete shopping cart that demonstrates all the concepts:

```svelte
<script>
    let items = $state([
        { id: 1, name: 'Mechanical Keyboard', price: 129.99, quantity: 1 },
        { id: 2, name: 'Wireless Mouse', price: 49.99, quantity: 2 },
        { id: 3, name: 'USB-C Cable', price: 12.99, quantity: 3 }
    ]);
    
    let discountCode = $state('');
    let taxRate = $state(0.08);  // 8% tax
    
    // Derived: Item count
    let itemCount = $derived(
        items.reduce((sum, item) => sum + item.quantity, 0)
    );
    
    // Derived: Subtotal
    let subtotal = $derived(
        items.reduce((sum, item) => sum + (item.price * item.quantity), 0)
    );
    
    // Derived: Discount validation and amount
    let discount = $derived.by(() => {
        const code = discountCode.toUpperCase().trim();
        
        if (code === 'SAVE10') {
            return { valid: true, rate: 0.10, amount: subtotal * 0.10 };
        }
        if (code === 'SAVE20' && subtotal > 100) {
            return { valid: true, rate: 0.20, amount: subtotal * 0.20 };
        }
        if (code === 'FREESHIP') {
            return { valid: true, rate: 0, amount: 0, freeShipping: true };
        }
        
        return { valid: false, rate: 0, amount: 0 };
    });
    
    // Derived: Shipping cost
    let shippingCost = $derived(
        discount.freeShipping ? 0 : (subtotal > 50 ? 0 : 5.99)
    );
    
    // Derived: Tax amount
    let taxAmount = $derived((subtotal - discount.amount + shippingCost) * taxRate);
    
    // Derived: Final total
    let total = $derived(subtotal - discount.amount + shippingCost + taxAmount);
    
    // Actions
    function updateQuantity(id, delta) {
        items = items.map(item => 
            item.id === id 
                ? { ...item, quantity: Math.max(0, item.quantity + delta) }
                : item
        );
    }
    
    function removeItem(id) {
        items = items.filter(item => item.id !== id);
    }
</script>

<div class="cart">
    <h2>Shopping Cart</h2>
    
    {#each items as item (item.id)}
        <div class="cart-item">
            <div class="item-info">
                <strong>{item.name}</strong>
                <span>${item.price.toFixed(2)}</span>
            </div>
            <div class="item-controls">
                <button onclick={() => updateQuantity(item.id, -1)}>-</button>
                <span>{item.quantity}</span>
                <button onclick={() => updateQuantity(item.id, 1)}>+</button>
                <button onclick={() => removeItem(item.id)}>Remove</button>
            </div>
            <div class="item-total">
                ${(item.price * item.quantity).toFixed(2)}
            </div>
        </div>
    {/each}
    
    <div class="discount-section">
        <input 
            type="text" 
            bind:value={discountCode} 
            placeholder="Discount code"
        />
        {#if discountCode && discount.valid}
            <span class="valid">✓ {discount.rate * 100}% discount applied!</span>
        {:else if discountCode}
            <span class="invalid">✗ Invalid code</span>
        {/if}
    </div>
    
    <div class="summary">
        <div class="summary-row">
            <span>Items ({itemCount}):</span>
            <span>${subtotal.toFixed(2)}</span>
        </div>
        
        {#if discount.amount > 0}
            <div class="summary-row discount">
                <span>Discount:</span>
                <span>-${discount.amount.toFixed(2)}</span>
            </div>
        {/if}
        
        <div class="summary-row">
            <span>Shipping:</span>
            <span>
                {#if shippingCost === 0}
                    FREE
                {:else}
                    ${shippingCost.toFixed(2)}
                {/if}
            </span>
        </div>
        
        <div class="summary-row">
            <span>Tax ({(taxRate * 100).toFixed(0)}%):</span>
            <span>${taxAmount.toFixed(2)}</span>
        </div>
        
        <div class="summary-row total">
            <strong>Total:</strong>
            <strong>${total.toFixed(2)}</strong>
        </div>
    </div>
</div>

<style>
    .cart { max-width: 600px; margin: 0 auto; padding: 20px; }
    .cart-item { 
        display: flex; 
        justify-content: space-between; 
        align-items: center;
        padding: 15px;
        border-bottom: 1px solid #eee;
    }
    .item-info { flex: 1; }
    .item-controls { display: flex; gap: 10px; align-items: center; }
    .discount-section { margin: 20px 0; }
    .discount-section input { width: 200px; padding: 8px; }
    .valid { color: green; margin-left: 10px; }
    .invalid { color: red; margin-left: 10px; }
    .summary { margin-top: 20px; border-top: 2px solid #333; padding-top: 15px; }
    .summary-row { 
        display: flex; 
        justify-content: space-between; 
        padding: 8px 0;
    }
    .summary-row.discount { color: green; }
    .summary-row.total { 
        font-size: 1.3em; 
        border-top: 1px solid #333; 
        margin-top: 10px;
        padding-top: 10px;
    }
    button { padding: 5px 10px; cursor: pointer; }
</style>
```

**What's happening:**
1. All pricing logic is derived—no manual updates needed
2. Change quantity → everything recalculates automatically
3. Enter discount code → discount, tax, and total update
4. Complex discount rules live in `$derived.by()`
5. The entire pricing cascade is managed declaratively

---

## ⚠️ Common Mistakes

### 1. Side Effects in Derived Values

```javascript
// ❌ BAD: Side effects don't belong in $derived()
let log = $state([]);
let doubled = $derived.by(() => {
    console.log('Calculating...');  // Side effect!
    log.push('calculated');  // Modifying state!
    return count * 2;
});
```

**Why it's bad:** Derived values should be pure calculations. Side effects belong in `$effect()` (covered in Article 4).

```javascript
// ✅ GOOD: Pure calculation only
let doubled = $derived(count * 2);

// Handle logging separately
$effect(() => {
    console.log('Doubled value:', doubled);
});
```

### 2. Storing Derived Values as State

```javascript
// ❌ BAD: Redundant state
let price = $state(10);
let tax = $state(price * 0.1);  // This should be derived!

function updatePrice(newPrice) {
    price = newPrice;
    tax = newPrice * 0.1;  // Manual sync = bug risk
}
```

```javascript
// ✅ GOOD: Derive it
let price = $state(10);
let tax = $derived(price * 0.1);  // Always in sync

function updatePrice(newPrice) {
    price = newPrice;  // tax updates automatically
}
```

### 3. Over-Deriving Simple Values

```javascript
// ❌ Unnecessary: Too simple for $derived()
let name = $state('Alice');
let greeting = $derived(`Hello, ${name}!`);
```

```svelte
<!-- ✅ GOOD: Just use inline expressions for simple cases -->
<script>
    let name = $state('Alice');
</script>

<p>Hello, {name}!</p>
```

**Rule of thumb:** Use `$derived()` when the value is:
- Used in multiple places
- Part of a calculation chain
- Complex enough to warrant a variable

### 4. Forgetting $derived.by() for Complex Logic

```javascript
// ❌ This won't work—you need $derived.by() for function bodies
let result = $derived({
    const a = Math.random();  // Syntax error!
    return a * 2;
});
```

```javascript
// ✅ GOOD: Use $derived.by() for multi-statement logic
let result = $derived.by(() => {
    const a = Math.random();
    return a * 2;
});
```

---

## ✨ Best Practices

### 1. Name Derived Values Descriptively

```javascript
// ❌ Vague
let d = $derived(price * quantity);
let x = $derived(d * 1.1);

// ✅ Clear
let subtotal = $derived(price * quantity);
let totalWithTax = $derived(subtotal * 1.1);
```

### 2. Extract Complex Derivations

```javascript
// ❌ Hard to read
let result = $derived(items.filter(i => i.active).map(i => i.price * i.qty).reduce((a, b) => a + b, 0) * 1.08);

// ✅ Break it down
let activeItems = $derived(items.filter(item => item.active));
let subtotal = $derived(
    activeItems.reduce((sum, item) => sum + (item.price * item.qty), 0)
);
let totalWithTax = $derived(subtotal * 1.08);
```

### 3. Use $derived.by() for Object Construction

```javascript
// ✅ Clean object assembly
let userDisplay = $derived.by(() => {
    const fullName = `${user.firstName} ${user.lastName}`;
    const age = new Date().getFullYear() - user.birthYear;
    const isAdult = age >= 18;
    
    return { fullName, age, isAdult };
});
```

### 4. Keep Derivations Close to Usage

```javascript
// ✅ Group related derived values
let items = $state([...]);

// Cart calculations
let itemCount = $derived(items.length);
let subtotal = $derived(items.reduce(...));
let tax = $derived(subtotal * 0.08);
let total = $derived(subtotal + tax);

// Item statistics  
let averagePrice = $derived(subtotal / itemCount);
let mostExpensive = $derived(Math.max(...items.map(i => i.price)));
```

---

## 🧠 Mental Model: The Reactive Spreadsheet

Think of your component as a spreadsheet:

| Cell | Formula | Value |
|------|---------|-------|
| A1 (price) | `$state(29.99)` | 29.99 |
| A2 (quantity) | `$state(3)` | 3 |
| A3 (subtotal) | `$derived(A1 * A2)` | 89.97 |
| A4 (tax) | `$derived(A3 * 0.08)` | 7.20 |
| A5 (total) | `$derived(A3 + A4)` | 97.17 |

When you change A1 (price) or A2 (quantity), cells A3-A5 automatically recalculate in order. You never manually update them—they're formulas, not values.

**Key principles:**
1. **Derived values are always fresh** - They recalculate when dependencies change
2. **No manual synchronization** - The framework handles it
3. **Efficient updates** - Only recalculates what's needed
4. **Dependency tracking is automatic** - Just read the state you need

---

## 📖 Quick Reference

```javascript
// Simple expression - single calculation
let doubled = $derived(count * 2);

// Complex logic - multiple statements
let stats = $derived.by(() => {
    const total = scores.reduce((sum, s) => sum + s, 0);
    const average = total / scores.length;
    return { total, average };
});

// Chaining - derived from derived
let subtotal = $derived(price * quantity);
let total = $derived(subtotal + (subtotal * taxRate));

// Conditional logic
let status = $derived(score >= 90 ? 'A' : score >= 80 ? 'B' : 'C');

// Object/array derivation
let summary = $derived.by(() => ({
    count: items.length,
    total: items.reduce((sum, item) => sum + item.price, 0)
}));
```

### $derived() vs $derived.by()

| Use $derived() when... | Use $derived.by() when... |
|------------------------|---------------------------|
| Single expression | Multiple statements needed |
| `count * 2` | Local variables needed |
| Simple ternary | Complex if/else logic |
| One line of code | Loops or iterations |
| No intermediate values | Building objects/arrays |

---

## 🎯 Performance: Memoization Explained

**Key insight:** `$derived()` is automatically memoized. The calculation only runs when dependencies change.

```javascript
let items = $state([/* 1000 items */]);

// This ONLY recalculates when 'items' changes
let expensiveCalculation = $derived.by(() => {
    console.log('Calculating...');  // Watch this log
    return items.reduce((sum, item) => sum + item.value, 0);
});
```

If you read `expensiveCalculation` 100 times in your template, the calculation runs **once** per change to `items`. The result is cached until `items` changes again.

**Compare to a function:**

```javascript
// ❌ Runs every time you call it
function calculateTotal() {
    console.log('Calculating...');  // Logs constantly!
    return items.reduce((sum, item) => sum + item.value, 0);
}

// Every render = new calculation
<p>Total: {calculateTotal()}</p>
<p>Again: {calculateTotal()}</p>  // Runs again!
```

**Why derived is better:**
- ✅ Automatic caching
- ✅ Only recalculates on dependency changes
- ✅ Safe to use in multiple places
- ✅ No manual optimization needed

---

## 💪 Practice Exercise

Build a **grade calculator** with these requirements:

1. **Input:** Array of assignments, each with name, score (0-100), and weight (percentage)
2. **Derived values to calculate:**
   - Total weight (should equal 100%)
   - Weighted average score
   - Letter grade (A: 90+, B: 80-89, C: 70-79, D: 60-69, F: <60)
   - Number of assignments completed (score > 0)
   - Highest and lowest scores
3. **Validation:** Show warning if total weight ≠ 100%

**Starter code:**

```svelte
<script>
    let assignments = $state([
        { name: 'Homework 1', score: 95, weight: 20 },
        { name: 'Midterm', score: 88, weight: 30 },
        { name: 'Final Project', score: 0, weight: 50 }
    ]);
    
    // TODO: Add your derived values here
    // - totalWeight
    // - weightedAverage
    // - letterGrade
    // - completedCount
    // - highestScore
    // - lowestScore (excluding 0s)
</script>
```

**Solution in next article's intro!**

---

## 🔜 Next Steps

You now understand how to create values that automatically stay in sync with your state. But what about **side effects**—things like API calls, logging, or updating localStorage? That's where `$effect()` comes in.

**Coming up in Article 4: "$effect - Side Effects & Reactions"**
- When to use `$effect()` vs `$derived()`
- Handling side effects correctly
- Cleanup patterns
- Common pitfalls and anti-patterns

**Quick teaser:** If `$derived()` is for "what is the value?", then `$effect()` is for "what should happen?". See you there! 🚀

---

## 📚 Further Reading

- [Svelte 5 Runes Documentation](https://svelte.dev/docs/svelte/$derived)
- Previous: [Article 2: $state - Your First Rune](#)
- Next: [Article 4: $effect - Side Effects & Reactions](#)
- [Complete Code Examples](https://github.com/your-repo/svelte-5-runes-examples)

---

*Have questions or found this helpful? Leave a comment below! Want to dive deeper? The next article on `$effect()` will blow your mind with how elegantly Svelte 5 handles side effects.* 🎉
