# Topic 1: How Python Stores Data & Memory Mutability

In Python, understanding how data is stored in memory is critical. If you do not understand memory references and mutability, you will write code with subtle bugs that are extremely hard to track down.

---

## 1. Everything is an Object & Variables are References

In Python, a variable is not a container that holds a value (like in C++ or Java). Instead, a variable is a **name reference** (or pointer) pointing to a physical object in memory.

### Code Example:
```python
x = [1, 2, 3]
y = x
```

Here:
1. Python creates a list object `[1, 2, 3]` at a specific address in memory.
2. It assigns the name `x` to point to that address.
3. When you run `y = x`, Python makes the name `y` point to the **exact same address** in memory. It does not copy the list.

```
       +-----------------------+
x ---> | List Object: [1, 2, 3] | <--- y
       +-----------------------+
```

---

## 2. Mutability vs. Immutability

Python objects are divided into two categories: **Mutable** (can be changed after creation) and **Immutable** (cannot be changed).

### Mutable Types (List, Dict, Set)
When you modify a mutable object, it changes **in place**. Any other variables pointing to that same object will see the change.

```python
x = [1, 2, 3]
y = x

y.append(4)  # Modifying the list via 'y'

print(x)  # Output: [1, 2, 3, 4] (x changed too!)
print(x is y)  # Output: True (They are still the exact same object in memory)
```

### Immutable Types (Int, Float, Str, Tuple, Bool)
When you attempt to modify an immutable object, Python **cannot** change it in place. Instead, it creates a **brand new object** in memory and points your variable to it.

```python
a = 5
b = a

a = a + 1  # This creates a brand new integer object '6' and points 'a' to it.

print(a)  # Output: 6
print(b)  # Output: 5 (b still points to the original integer 5)
print(a is b)  # Output: False (They point to different memory locations now)
```

---

## 3. The Mutable Default Argument Bug

The most common Python trap happens when you use a mutable object (like a list `[]` or dictionary `{}`) as a default argument in a function definition.

### The Buggy Code:
```python
# The default list [] is created ONCE when Python reads this file, NOT when the function runs.
def add_to_inventory(item, items=[]):
    items.append(item)
    return items

# First call:
user_1_items = add_to_inventory("sword")
print(user_1_items)  # Output: ['sword']

# Second call (for a different user, expecting an empty list default):
user_2_items = add_to_inventory("shield")
print(user_2_items)  # Output: ['sword', 'shield']  <-- BUG! User 1's items carried over!
```

### The Correct Way (Using `None`):
To prevent this, always set the default value to `None` and initialize the mutable object inside the function block:

```python
def add_to_inventory_safe(item, items=None):
    if items is None:
        items = []  # A brand new list is created every time the function runs without args
    items.append(item)
    return items

user_1 = add_to_inventory_safe("sword")
print(user_1)  # Output: ['sword']

user_2 = add_to_inventory_safe("shield")
print(user_2)  # Output: ['shield']  <-- Fixed!
```
