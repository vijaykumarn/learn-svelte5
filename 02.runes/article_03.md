# Learning Svelte 5 Runes – Part 3  
## Working with Objects using `$state`, `$derived`, and `$effect`

In **Part 1**, we learned how `$state` works with simple numbers.  
In **Part 2**, we used `$state` with arrays.

Now we move to one of the most important concepts in real applications: **objects**.

Objects allow us to group related data together, such as:
- user profiles
- form data
- settings
- configuration

This article will teach you how to work with **objects in Svelte 5 Runes** using a clean, realistic example.

---

## What You Will Learn

By the end of this article, you will understand:

- How to store an object using `$state`
- How to update object properties correctly
- Why reassignment is required
- How `$derived` works with object values
- How `$effect` reacts to object changes
- How object state drives the UI

---

## Our Example: A User Profile Card

We will build a **user profile card** that:

- Stores user information in an object
- Updates the user’s age
- Toggles the user’s active status
- Displays derived text
- Updates UI styles automatically
- Logs changes to the console

This mirrors how objects are used in real applications.

---

## Creating a Reactive Object with `$state`

```ts
let user = $state({
	name: 'Alice',
	age: 25,
	active: true
});
````

### What Is Happening Here?

* `user` is a JavaScript object
* It contains related data about one user
* `$state` makes the object **reactive**

Whenever the `user` object is reassigned:

* The UI updates
* Derived values recompute
* Effects run automatically

---

## Reading Object Properties in the Template

```svelte
<span>{user.name}</span>
<span>{user.age}</span>
<span>{user.active ? 'Active' : 'Inactive'}</span>
```

You access object properties just like normal JavaScript:

* `user.name`
* `user.age`
* `user.active`

Svelte tracks these automatically.

---

## Updating Object Properties (The Correct Way)

### ❌ Incorrect Way (Common Mistake)

```ts
user.age++; // ❌ Not reactive
```

This mutates the object but does **not** trigger reactivity.

---

### ✅ Correct Way: Reassign the Object

```ts
const incrementAge = () => {
	user = { ...user, age: user.age + 1 };
};
```

### Why This Works

1. `...user` copies the existing object
2. We override the `age` property
3. A **new object** is created
4. Svelte detects the reassignment

---

## Toggling a Boolean Property

```ts
const toggleActive = () => {
	user = { ...user, active: !user.active };
};
```

This pattern works for:

* booleans
* numbers
* strings
* any property in an object

---

## Using `$derived` with Objects

Derived values are perfect for **computed object data**.

---

### Derived Description Text

```ts
const description = $derived(
	`${user.name} is ${user.age} years old!`
);
```

This value:

* Depends on `user.name` and `user.age`
* Automatically updates when either changes
* Is never manually updated

---

### Derived Status Text

```ts
const statusText = $derived(
	user.active ? 'Active' : 'Inactive'
);
```

This keeps UI text perfectly in sync with state.

---

## Reacting to Object Changes with `$effect`

```ts
$effect(() => {
	console.log('User updated:', user);
});
```

This effect runs whenever:

* Any user property changes
* The object is reassigned

Use `$effect` for:

* Logging
* Debugging
* Analytics
* API calls

⚠️ **Important Rule:**
Do not compute values inside `$effect`.
Use `$derived` instead.

---

## How Object State Drives the UI

This example shows something very important:

> **State controls UI, not the other way around.**

* Changing `user.active` updates:

  * Status badge
  * Status dot color
  * Footer text color
* Changing `user.age` updates:

  * Description text
  * Age display
  * Footer stats

All without manual DOM logic.

---

## Key Pattern to Remember

Every object update follows this pattern:

```ts
state = {
	...state,
	changedProperty: newValue
};
```

This is the **core pattern** for working with objects in Svelte 5.

---

## Mental Model for Objects in Svelte 5

Keep this mental checklist:

1. `$state` makes the object reactive
2. Never mutate object properties directly
3. Always reassign the object
4. `$derived` computes values
5. `$effect` handles side effects

If you follow this model, object state becomes predictable and easy.

---

## Summary

In this article, you learned:

* How to create reactive objects with `$state`
* How to update object properties correctly
* Why reassignment matters
* How `$derived` works with object values
* How `$effect` reacts to object changes
* How UI reacts automatically to state

At this point, you understand the **core of Svelte 5 Runes**.

---

## What’s Next?

In **Part 4**, we’ll cover:

* Nested objects
* Updating deep properties safely
* When to split state
* Real-world state structure

This is where your apps start to feel *real*.

See you in Part 4 🚀
