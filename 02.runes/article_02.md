# Learning Svelte 5 Runes – Part 2  
## Working with Arrays using `$state`, `$derived`, and `$effect`

In **Part 1**, we learned how Svelte 5 Runes work using a single number and a counter.

In this article, we move to the next essential concept: **arrays**.

Arrays are where reactivity starts to feel *real*, and they are also where many beginners get confused.  
So we’ll take this **slowly and clearly**.

---

## What You Will Learn

By the end of this article, you will understand:

- How to store arrays using `$state`
- How to add, remove, and reset array values
- Why reassignment is required for reactivity
- How `$derived` works with array data
- How to compute **count, sum, and average**
- How `$effect` reacts to array changes

Still no objects.  
Still beginner-friendly.

---

## Our Example: A Reactive Numbers List

We will build a component that:

- Stores a list of numbers
- Adds numbers automatically
- Removes the last number
- Resets the array
- Displays:
  - total items
  - sum
  - average
- Logs array changes to the console

This example shows how **arrays, derived values, and effects work together**.

---

## Creating a Reactive Array with `$state`

```ts
let numbers: number[] = $state<number[]>([]);
````

### What Is Happening Here?

* `numbers` is an array of numbers
* It starts empty: `[]`
* `$state` makes it **reactive**

Whenever `numbers` changes, Svelte automatically:

* Updates the UI
* Re-runs derived values
* Triggers effects

---

## Adding Items to the Array

```ts
const addNumber = () => {
	numbers = [...numbers, numbers.length + 1];
};
```

### Why This Works

This line does **three important things**:

1. `...numbers` copies the existing array
2. `numbers.length + 1` creates the next number
3. The result is assigned back to `numbers`

👉 **Assignment is the key to reactivity.**

---

### ❌ Common Beginner Mistake

```ts
numbers.push(1); // ❌ UI will NOT update
```

This mutates the array **without reassigning it**.
Svelte 5 does not react to mutations alone.

---

## Removing the Last Item Safely

```ts
const removeLast = () => {
	if (numbers.length > 0) {
		numbers = numbers.slice(0, -1);
	}
};
```

### Why This Is Done This Way

* `slice()` returns a **new array**
* We avoid errors when the array is empty
* Reassignment triggers reactivity

---

## Resetting the Array

```ts
const reset = () => {
	numbers = [];
};
```

This is the simplest (and cleanest) way to reset state:

* Assign a brand new empty array
* Everything updates automatically

---

## Using `$derived` with Arrays

Derived values are where arrays really shine.

---

### Total Number of Items

```ts
const totalNumbers = $derived(numbers.length);
```

* Automatically updates when the array changes
* No manual tracking required

---

### Calculating the Sum

```ts
const sum = $derived(
	numbers.reduce((acc, curr) => acc + curr, 0)
);
```

This:

* Loops through the array
* Adds all values together
* Recalculates whenever `numbers` changes

---

### Calculating the Average

```ts
const average = $derived(
	numbers.length > 0 ? sum / numbers.length : 0
);
```

Important points:

* Avoids division by zero
* Depends on **other derived values**
* Always stays correct

Yes — `$derived` can depend on **another `$derived`**.

---

## Reacting to Changes with `$effect`

```ts
$effect(() => {
	if (numbers.length > 0) {
		console.log(`Numbers updated: ${numbers.length} items, sum: ${sum}, average: ${average}`);
	} else {
		console.log('Numbers array is empty');
	}
});
```

### What Triggers This Effect?

This effect runs whenever:

* The array changes
* The length changes
* The sum changes

Use `$effect` for:

* Logging
* Debugging
* Analytics
* Side effects

⚠️ **Rule:**
Never compute values in `$effect`.
Use `$derived` instead.

---

## Displaying Arrays in the Template

```svelte
{#each numbers as number, index (index)}
	<li>
		Index {index}: {number}
	</li>
{/each}
```

This does the following:

* Loops over the array
* Tracks items by index
* Updates automatically when the array changes

---

## Conditional Rendering with Arrays

```svelte
{#if numbers.length > 0}
	<!-- show list -->
{:else}
	<!-- show empty state -->
{/if}
```

This is a common and recommended pattern:

* Show UI when data exists
* Show a fallback when it doesn’t

---

## Full Mental Model for Arrays in Svelte 5

When working with arrays and Runes, remember:

1. `$state` makes the array reactive
2. Mutating is **not enough**
3. Always assign a new array
4. `$derived` recalculates automatically
5. `$effect` reacts to changes

If you understand these five rules, arrays will never feel confusing again.

---

## Summary

In this article, you learned:

* How to create reactive arrays with `$state`
* Why reassignment is required
* How to add, remove, and reset items
* How to compute derived values like sum and average
* How `$effect` reacts to array updates

This is the **bridge** between simple state and complex state.

---

## What’s Next?

In **Part 3**, we will cover:

* Objects with `$state`
* Updating object properties
* Nested reactivity
* Common object-related mistakes
* Best practices for structured state

Once you understand objects, you can build real applications.

See you in Part 3 🚀
