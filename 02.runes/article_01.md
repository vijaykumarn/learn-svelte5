# Learning Svelte 5 Runes – Part 1

## Understanding `$state`, `$derived`, and `$effect` with a Simple Counter

Svelte 5 introduces a brand-new way of managing reactivity called **Runes**.
If you’re new to Svelte (or frontend frameworks in general), don’t worry — this article explains everything **from the ground up**.

In this first article, we will:

* Learn what **Runes** are
* Use **basic data types** (numbers only)
* Understand `$state`, `$derived`, and `$effect`
* Build a simple **counter component**

No arrays, no objects, no advanced concepts — just the essentials.

---

## What Are Runes in Svelte 5?

In earlier versions of Svelte, reactivity was “implicit” — variables magically updated the UI when they changed.

In **Svelte 5**, reactivity is now **explicit** using *Runes*.

A **Rune** is a special function that starts with a `$` symbol, such as:

* `$state`
* `$derived`
* `$effect`

Each Rune has a **clear purpose**, making your code:

* Easier to understand
* Easier to debug
* More predictable

---

## Our Example: A Simple Counter

We’ll build a counter that:

* Increases a number
* Shows double the number
* Logs changes to the console
* Can be reset

Here is the full code we’ll explain step by step:

```svelte
<script lang="ts">
	let count = $state(0);

	const increment = () => {
		return count++;
	};

	const doubleCounter = () => {
		return count * 2;
	};

	const reset = () => {
		return (count = 0);
	};

	let doubleCount = $derived(doubleCounter());

	$effect(() => {
		console.log(`The count is now ${count} and doubleCount is ${doubleCount}`);
	});
</script>
```

---

## Step 1: `$state` — Creating Reactive State

```ts
let count = $state(0);
```

### What is `$state`?

`$state` creates **reactive state**.

That means:

* When `count` changes
* The UI updates automatically
* Any dependent logic re-runs automatically

### Why not just `let count = 0`?

In Svelte 5:

* Normal variables are **not reactive**
* Only `$state` variables trigger updates

So `$state` is how we say:

> “This value can change, and Svelte should track it.”

### Key Points

* `count` is a **number**
* It starts at `0`
* It is reactive

---

## Step 2: Updating State with Functions

### Increment Function

```ts
const increment = () => {
	return count++;
};
```

This function:

* Increases `count` by 1
* Automatically updates the UI
* Triggers all dependent logic

Because `count` is `$state`, **Svelte reacts instantly**.

---

### Reset Function

```ts
const reset = () => {
	return (count = 0);
};
```

This function:

* Sets `count` back to `0`
* Updates everything that depends on `count`

---

## Step 3: Derived Values with `$derived`

```ts
const doubleCounter = () => {
	return count * 2;
};

let doubleCount = $derived(doubleCounter());
```

### What is `$derived`?

`$derived` creates a **value that depends on other state**.

In plain English:

> “Whenever `count` changes, recalculate this.”

### How This Works

* `doubleCounter()` calculates `count * 2`
* `$derived()` tracks its dependencies
* `doubleCount` always stays in sync with `count`

You **never manually update** `doubleCount`.

That’s the key benefit.

---

### Why Not Just Calculate Inline?

You *could* do:

```svelte
{count * 2}
```

But `$derived` is better when:

* The logic is reused
* The calculation is complex
* You want clean, readable code

---

## Step 4: Side Effects with `$effect`

```ts
$effect(() => {
	console.log(`The count is now ${count} and doubleCount is ${doubleCount}`);
});
```

### What is `$effect`?

`$effect` runs **whenever its dependencies change**.

In this case:

* `count`
* `doubleCount`

Whenever either value updates, the effect runs again.

---

### What Should `$effect` Be Used For?

Use `$effect` for:

* Logging
* API calls
* DOM interactions
* Analytics
* Debugging

⚠️ **Important Rule:**
`$effect` is for *side effects*, not calculations.

---

## Step 5: Displaying State in the Template

```svelte
<span>{count}</span>
<span>{doubleCount}</span>
```

Because both values are reactive:

* The UI updates automatically
* No manual DOM updates
* No watchers
* No setters

This is the heart of Svelte.

---

## Step 6: Handling User Interaction

```svelte
<button onclick={increment}>Increment</button>
<button onclick={reset}>Reset</button>
```

Svelte uses **native DOM events**.

* `onclick` calls the function
* The function updates `$state`
* Everything else updates automatically

---

## Mental Model (Very Important)

Here’s how to *think* about Svelte 5 Runes:

| Rune       | Purpose                              |
| ---------- | ------------------------------------ |
| `$state`   | Stores reactive data                 |
| `$derived` | Calculates values from state         |
| `$effect`  | Runs side effects when state changes |

If you understand this table, you understand **80% of Svelte 5 reactivity**.

---

## What We Learned in This Article

✅ What Svelte 5 Runes are

✅ How `$state` creates reactive variables

✅ How `$derived` keeps values in sync

✅ How `$effect` reacts to changes

✅ How UI updates automatically

All using **only numbers and functions**.

---

## What’s Next in the Series?

In the next articles, we’ll cover:

* Arrays with `$state`
* Objects with `$state`
* Updating nested data
* Performance best practices
* Common beginner mistakes

This article is your **foundation** — everything else builds on it.
