# Article 7: Objects - Structured State

## The Object Problem That Breaks Beginners

You've mastered reactive primitives. You can handle numbers, strings, and booleans with ease. Then you build your first real form—a user profile with name, email, bio, and preferences—and suddenly nothing works the way you expect.

You update `user.name = "Alice"` and the UI doesn't update. You try `user = { ...user, name: "Alice" }` and it works, but it feels clunky. You nest an object inside another object and updates become a nightmare of spread operators. Welcome to the world of reactive objects, where the rules change and many developers get lost.

Here's the truth: **objects are the building blocks of real applications**, and Svelte 5's Runes handle them beautifully—once you understand the mental model. This article will transform you from fighting with object updates to wielding them confidently in production code.

---

## 🎯 Learning Goals

By the end of this article, you will:

- ✅ Create reactive objects with `$state({})`
- ✅ Update object properties using immutable patterns
- ✅ Handle partial updates across multiple properties
- ✅ Navigate nested objects without losing reactivity
- ✅ Choose between `$state()` and `$state.raw()` for objects
- ✅ Build helper functions to simplify object updates
- ✅ Create a production-ready user profile editor

---

## 📋 Prerequisites

Before diving in, you should be comfortable with:

- **Article 2**: `$state()` for primitives
- **Article 3**: `$derived()` for computed values
- JavaScript object fundamentals (spread operator, destructuring)
- Basic understanding of immutability

---

## 🧠 Core Concept: Objects Are Containers of State

### The Fundamental Rule

In Svelte 5, **reactivity tracks the object reference, not its properties**. Let me show you what this means:

```svelte
<script>
let user = $state({ name: "Alice", age: 30 });

// ❌ This does NOT trigger reactivity
user.name = "Bob";

// ✅ This DOES trigger reactivity
user = { ...user, name: "Bob" };
</script>
```

**Why?** Because Svelte 5 with Runes uses *fine-grained reactivity*. When you create `$state({ name: "Alice" })`, Svelte creates a reactive proxy that tracks when the *entire object* is reassigned. It doesn't automatically track individual property mutations.

### Deep Reactivity by Default

However, there's an important nuance: `$state()` creates **deep reactive proxies** by default. This means:

```svelte
<script>
let user = $state({ name: "Alice", age: 30 });

// This WILL trigger reactivity because $state() wraps objects deeply
user.name = "Bob"; // ✅ Works!

// But this is the preferred pattern
user = { ...user, name: "Bob" }; // ✅ More explicit
</script>
```

**So why learn immutable patterns?** Because:
1. They're more predictable and easier to debug
2. They work consistently with `$state.raw()` (covered later)
3. They prepare you for complex nested updates
4. They align with modern JavaScript best practices

---

## 💻 Live Example: User Profile Editor

Let's build a real-world example that demonstrates all the key concepts:Now let's break down the actual Svelte 5 code patterns:

### Basic Object Update Pattern

```svelte
<script>
let user = $state({
  name: "Alice",
  email: "alice@example.com"
});

// ✅ Recommended: Immutable update
function updateName(newName) {
  user = { ...user, name: newName };
}

// ✅ Also works: Direct property mutation (with $state)
function updateEmail(newEmail) {
  user.email = newEmail; // Works because $state creates deep proxy
}
</script>

<input bind:value={user.name} />
<input bind:value={user.email} />
```

### Partial Updates: Multiple Properties

When updating several properties at once, use object spread:

```svelte
<script>
let user = $state({
  name: "Alice",
  email: "alice@example.com",
  age: 30,
  city: "New York"
});

// ✅ Update multiple properties efficiently
function updateProfile(updates) {
  user = { ...user, ...updates };
}

// Usage
updateProfile({ 
  name: "Alice Cooper", 
  city: "Los Angeles" 
});
</script>
```

### Nested Objects: The Spread Cascade

Nested objects require spreading at each level:

```svelte
<script>
let user = $state({
  name: "Alice",
  preferences: {
    theme: "dark",
    notifications: true
  }
});

// ✅ Update nested property
function updateTheme(newTheme) {
  user = {
    ...user,
    preferences: {
      ...user.preferences,
      theme: newTheme
    }
  };
}
</script>
```

**The pattern**: Spread from the outside in, updating the specific property at the deepest level.

---

## ⚠️ Common Mistakes

### Mistake #1: Forgetting to Spread Nested Objects

```svelte
<script>
let user = $state({
  name: "Alice",
  preferences: { theme: "dark", notifications: true }
});

// ❌ WRONG: Loses the notifications property!
function updateTheme(theme) {
  user = {
    ...user,
    preferences: { theme } // Overwrites entire preferences object
  };
}

// ✅ CORRECT: Preserves all properties
function updateTheme(theme) {
  user = {
    ...user,
    preferences: { ...user.preferences, theme }
  };
}
</script>
```

### Mistake #2: Deep Nesting Without Helper Functions

```svelte
<script>
// ❌ Deeply nested updates become unreadable
user = {
  ...user,
  settings: {
    ...user.settings,
    privacy: {
      ...user.settings.privacy,
      showEmail: false
    }
  }
};

// ✅ Create a helper function
function updatePrivacy(key, value) {
  user = {
    ...user,
    settings: {
      ...user.settings,
      privacy: {
        ...user.settings.privacy,
        [key]: value
      }
    }
  };
}
</script>
```

### Mistake #3: Confusing Object Reference vs Property Changes

```svelte
<script>
let user = $state({ name: "Alice" });

// This creates a NEW derived value every time user changes
let derived1 = $derived(user); // ❌ Not useful

// This derives from a PROPERTY of user
let derived2 = $derived(user.name.toUpperCase()); // ✅ Correct
</script>
```

---

## ✨ Best Practices

### Practice #1: Use Helper Functions for Complex Updates

```svelte
<script>
let user = $state({
  name: "",
  email: "",
  preferences: {
    theme: "light",
    notifications: false
  }
});

// Helper: Update top-level properties
function updateUser(updates) {
  user = { ...user, ...updates };
}

// Helper: Update nested preferences
function updatePreference(key, value) {
  user = {
    ...user,
    preferences: { ...user.preferences, [key]: value }
  };
}

// Clean usage in your component
function handleThemeChange(theme) {
  updatePreference('theme', theme);
}
</script>
```

### Practice #2: Validate Before Updating

```svelte
<script>
let user = $state({ email: "", age: 0 });

function updateEmail(newEmail) {
  if (!newEmail.includes('@')) {
    console.error('Invalid email');
    return;
  }
  user = { ...user, email: newEmail };
}

function updateAge(newAge) {
  if (newAge < 0 || newAge > 150) {
    console.error('Invalid age');
    return;
  }
  user = { ...user, age: newAge };
}
</script>
```

### Practice #3: Create Computed Values from Objects

```svelte
<script>
let user = $state({
  name: "Alice",
  email: "alice@example.com",
  bio: ""
});

// Derive validation state
let isValid = $derived(
  user.name.length > 0 && 
  user.email.includes('@')
);

// Derive statistics
let profileCompletion = $derived(() => {
  let filled = 0;
  let total = 3;
  
  if (user.name) filled++;
  if (user.email) filled++;
  if (user.bio) filled++;
  
  return Math.round((filled / total) * 100);
});
</script>

<div>Profile: {profileCompletion()}% complete</div>
```

### Practice #4: When to Use `$state.raw()`

For **flat objects that don't need deep reactivity**, use `$state.raw()` for better performance:

```svelte
<script>
// If you only reassign the entire object, use raw
let config = $state.raw({
  apiKey: "abc123",
  endpoint: "https://api.example.com"
});

// Update the whole object at once
function updateConfig(newConfig) {
  config = newConfig;
}

// ❌ Won't trigger reactivity with .raw()
config.apiKey = "new-key";

// ✅ Will trigger reactivity
config = { ...config, apiKey: "new-key" };
</script>
```

**Rule of thumb**: Use `$state.raw()` when you'll always replace the entire object, never mutate properties individually.

---

## 🧩 Mental Model: The Reactive Container

Think of a `$state()` object as a **container with sensors**:

```
┌─────────────────────────────┐
│  Reactive Container (user)   │
│                              │
│  ┌──────────────────────┐   │
│  │ name: "Alice"        │   │ ← Sensor on each property
│  │ email: "alice@..."   │   │
│  │ preferences: {...}   │   │ ← Nested container
│  └──────────────────────┘   │
│                              │
└─────────────────────────────┘
       ↓
   Changes detected
       ↓
   UI updates
```

When you reassign with spread (`user = { ...user, name: "Bob" }`):
1. You create a **new container**
2. The sensors detect the change
3. Svelte updates any UI that depends on `user`

When you mutate directly (`user.name = "Bob"`) with `$state()`:
1. The **deep proxy** detects the property change
2. Sensors trigger
3. UI updates

Both work, but immutable patterns are more predictable.

---

## 📖 Quick Reference

```svelte
<script>
// CREATE reactive object
let obj = $state({ key: "value" });
let flatObj = $state.raw({ key: "value" }); // No deep reactivity

// UPDATE single property
obj = { ...obj, key: "newValue" };

// UPDATE multiple properties
obj = { ...obj, key1: "val1", key2: "val2" };
obj = { ...obj, ...updates };

// UPDATE nested property
obj = {
  ...obj,
  nested: { ...obj.nested, key: "value" }
};

// DERIVE from object
let value = $derived(obj.key);
let computed = $derived(() => obj.key.toUpperCase());

// VALIDATE
let isValid = $derived(obj.email.includes('@'));

// HELPER function
function update(path, value) {
  obj = { ...obj, [path]: value };
}
</script>
```

---

## 💪 Practice Exercise: Build a Settings Panel

Create a settings panel with the following requirements:

1. **State structure**:
```javascript
{
  appearance: { theme: "light", fontSize: 14 },
  privacy: { showEmail: true, showActivity: false },
  notifications: { email: true, push: false, sms: false }
}
```

2. **Features**:
   - Toggle switches for all boolean settings
   - Dropdown for theme (light/dark/auto)
   - Number input for fontSize (12-20)
   - "Reset to Defaults" button
   - Display JSON of current settings

3. **Bonus challenges**:
   - Add a `$derived` value showing how many notifications are enabled
   - Implement a search filter to show only matching settings
   - Add localStorage persistence with `$effect()`

**Starter template**:

```svelte
<script>
let settings = $state({
  appearance: { theme: "light", fontSize: 14 },
  privacy: { showEmail: true, showActivity: false },
  notifications: { email: true, push: false, sms: false }
});

function updateAppearance(key, value) {
  settings = {
    ...settings,
    appearance: { ...settings.appearance, [key]: value }
  };
}

// Add more helper functions...
</script>

<!-- Your UI here -->
```

---

## 🔜 Next Steps

You've mastered reactive objects! You can now handle complex structured state with confidence. But what about when you need **collections of objects**—like a list of users, products, or tasks?

In **Article 8: "Arrays of Objects - Real-World Data"**, we'll tackle:
- Managing arrays of complex objects
- Finding and updating specific items
- Filtering and sorting reactive lists
- Building a complete user management dashboard

Arrays of objects are where things get really interesting—and where most developers struggle. We'll make sure you don't.

---

## 📚 Further Reading

- [Svelte 5 Runes Documentation](https://svelte.dev/docs/svelte/$state)
- [JavaScript Spread Operator (MDN)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax)
- [Immutability in JavaScript](https://developer.mozilla.org/en-US/docs/Glossary/Immutable)

---

**Got questions?** Drop them in the comments below. Struggling with a specific object update pattern? Share your code and we'll figure it out together! 

**Next up**: Article 8 - Arrays of Objects 🚀
