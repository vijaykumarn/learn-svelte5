# Learning Svelte 5 Runes – Part 4  
## Managing Arrays of Nested Objects: User Management Example

In **Part 3**, we learned how to work with a **single nested object** using `$state`.  

In real applications, you often work with **lists of nested objects**, such as users, tasks, or products.  

In this article, we’ll build a **full user management component** using Svelte 5 Runes.

---

## What You Will Learn

By the end of this article, you will understand:

- How to create a **reactive array of nested objects**
- How to **update nested properties** safely
- How to **add and remove items**
- How to use `$derived` for **computed statistics**
- How `$effect` reacts to changes
- When to **split state** for simplicity

---

## Our Example: User Management

We will:

- Display a list of users
- Show name, age, and active status
- Increment age
- Toggle active status
- Add new users
- Remove users
- Display derived statistics: total users, active users, and average age
- Log changes automatically

This mirrors a **real-world scenario**.

---

## Setting Up Reactive State

```ts
let users = $state([
	{ id: 1, name: 'Alice', profile: { age: 25, active: true } },
	{ id: 2, name: 'Bob', profile: { age: 30, active: false } },
	{ id: 3, name: 'Charlie', profile: { age: 35, active: true } }
]);
````

* `users` is an **array of objects**
* Each object has a nested `profile`
* `$state` tracks updates reactively
* Reassignment is **required** to trigger UI updates

---

## Updating Nested Properties

### Increment Age

```ts
const incrementAge = (id: number) => {
	users = users.map(user =>
		user.id === id
			? { ...user, profile: { ...user.profile, age: user.profile.age + 1 } }
			: user
	);
};
```

### Toggle Active Status

```ts
const toggleActive = (id: number) => {
	users = users.map(user =>
		user.id === id
			? { ...user, profile: { ...user.profile, active: !user.profile.active } }
			: user
	);
};
```

**Key Points:**

1. Use `map()` to create a **new array**.
2. Copy the object with `...user`.
3. Copy the nested `profile` object before updating.
4. Reassign the array to trigger reactivity.

---

## Adding and Removing Users

### Add a User

```ts
const addUser = () => {
	const newUser = {
		id: users.length + 1,
		name: `User ${users.length + 1}`,
		profile: { age: 20, active: false }
	};
	users = [...users, newUser];
};
```

### Remove a User

```ts
const removeUser = (id: number) => {
	users = users.filter(user => user.id !== id);
```

* `filter()` creates a **new array** without the removed user
* Always reassign `users` for reactivity

---

## Using `$derived` for Statistics

### Total Users

```ts
const totalUsers = $derived(users.length);
```

### Active Users

```ts
const activeUsers = $derived(users.filter(user => user.profile.active).length);
```

### Average Age

```ts
const averageAge = $derived(
	users.length > 0 ? users.reduce((sum, user) => sum + user.profile.age, 0) / users.length : 0
);
```

All derived values **update automatically** whenever `users` changes.

---

## Reacting to Changes with `$effect`

```ts
$effect(() => {
	console.log(`Users updated: ${users.length} total, ${activeUsers} active`);
});
```

* Runs whenever:

  * A user is added or removed
  * Nested properties are updated
* Useful for debugging, analytics, or API sync

---

## Displaying Users in the Template

```svelte
{#each users as user (user.id)}
	<div class="user-card">
		<h3>{user.name}</h3>
		<p>Age: {user.profile.age}</p>
		<p>Status: {user.profile.active ? 'Active' : 'Inactive'}</p>

		<button on:click={() => incrementAge(user.id)}>+1 Age</button>
		<button on:click={() => toggleActive(user.id)}>Toggle Status</button>
		<button on:click={() => removeUser(user.id)}>Remove</button>
	</div>
{/each}

<button on:click={addUser}>Add User</button>

<p>Total users: {totalUsers}</p>
<p>Active users: {activeUsers}</p>
<p>Average age: {averageAge.toFixed(1)}</p>
```

**Tips:**

* Always use a **unique key** (`user.id`) for each `{#each}` item.
* Nested properties are accessed with **dot notation**.
* All UI updates automatically when `$state` changes.

---

## When to Split State for Simplicity

As your array of nested objects grows, updates can get **verbose and complex**.

For example:

```ts
users = users.map(user =>
	user.id === id
		? { ...user, profile: { ...user.profile, age: user.profile.age + 1 } }
		: user
);
```

Doing this repeatedly can become **hard to read**.

### Signs You Should Split State

* You have multiple independent properties that change frequently
* Nested updates require repetitive `map` + spread operations
* `$derived` values depend only on part of the state
* Debugging nested updates is becoming cumbersome

### How to Split State

Instead of storing everything in a single array:

```ts
let users = $state([...]);
```

You can split into multiple reactive states:

```ts
let userProfiles = $state([
	{ id: 1, age: 25, active: true },
	{ id: 2, age: 30, active: false }
]);
let userNames = $state(['Alice', 'Bob']);
```

**Benefits:**

* Updates are simpler
* Less nested spread operations
* `$derived` computations become easier
* Easier to reason about which state triggers which UI updates

⚠️ **Rule of Thumb:** Split state whenever it improves **clarity and maintainability**.

---

## Key Patterns to Remember

1. **Always reassign arrays or objects** — direct mutation does not trigger reactivity.
2. Use `map()` for updating items, `filter()` for removals.
3. **Copy nested objects** when updating properties.
4. `$derived` handles calculations like totals and averages.
5. `$effect` is for side effects or logging.
6. **Split state** when nested updates become verbose or unclear.

---

## Summary

In this article, you learned:

* How to manage **arrays of nested objects**
* How to update nested properties safely
* How to add or remove items
* How `$derived` keeps statistics in sync
* How `$effect` reacts to changes
* When and why to **split state for simplicity**

This pattern is common in **user management, task lists, and product catalogs**.

---

## Next Steps

In **Part 5**, we’ll cover:

* Reactive forms
* Two-way binding with `$state`
* Validating user input
* Connecting forms to nested state

You’re now ready to build **fully interactive apps** with Svelte 5 Runes 🚀
