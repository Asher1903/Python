# Topic 5: Python Functions, Closures & Generators

This guide covers advanced function behaviors, argument passing, scope nesting (closures), and memory-efficient stream processing (generators).

---

## 1. Argument Passing: `*args` and `**kwargs`

In Python, you can write functions that accept a dynamic number of arguments.

*   `*args`: Collects extra positional arguments as a **Tuple**.
*   `**kwargs`: Collects extra keyword (named) arguments as a **Dictionary**.

### Code Example:
```python
def print_order(customer_name, *items, **metadata):
    print(f"Customer: {customer_name}")
    print(f"Items (Tuple): {items}")
    print(f"Metadata (Dict): {metadata}")

# Calling the function:
print_order(
    "Alice", 
    "Sword", "Shield", "Potion",  # Positional arguments collected into *items
    delivery="Express", priority=1 # Keyword arguments collected into **metadata
)
```

### Unpacking Operators
You can also use `*` and `**` to unpack lists and dicts when calling a function:
```python
items = ["Book", "Pen"]
config = {"theme": "dark", "zoom": 1.2}

# Calls print_order("Bob", "Book", "Pen", theme="dark", zoom=1.2)
print_order("Bob", *items, **config)
```

---

## 2. Closures

A **closure** is an inner function that remembers and has access to variables from its outer (enclosing) scope, even after the outer function has finished executing.

### How it works:
```python
def make_multiplier(factor):
    # This inner function remembers the 'factor' variable from its parent scope
    def multiplier(number):
        return number * factor
    return multiplier

double = make_multiplier(2)  # 'make_multiplier' finishes here
triple = make_multiplier(3)

print(double(5))  # Output: 10 (still remembers factor=2)
print(triple(5))  # Output: 15 (still remembers factor=3)
```

---

## 3. Iterables vs. Iterators

Before learning about Generators, you must understand how Python loops over data.

### 1. Iterable
An **Iterable** is any object that you can loop over (like a list, dictionary, tuple, or string). Under the hood, an iterable is an object that has an `__iter__` method which returns an **Iterator**.

### 2. Iterator
An **Iterator** is the actual "pointer" object that travels through the collection. It remembers its current state during iteration and has:
*   An `__iter__` method (which returns itself).
*   A `__next__` method (which returns the next item in the collection). When no items are left, it raises a `StopIteration` exception to signal the loop to stop.

### Under the Hood of a `for` Loop:
When you write:
```python
my_list = [10, 20]
for item in my_list:
    print(item)
```

Python actually translates it to this behind the scenes:
```python
# 1. Get the iterator from the list
iterator_obj = iter(my_list)  # calls my_list.__iter__()

# 2. Run an infinite loop to call next()
while True:
    try:
        item = next(iterator_obj)  # calls iterator_obj.__next__()
        print(item)
    except StopIteration:
        # 3. Catch StopIteration exception to end the loop
        break
```

---

## 4. Generators (`yield`)

A **generator** is a function that returns an iterator. Instead of computing all values and returning them as a list (which loads everything into RAM), a generator returns values **one at a time, on demand**, using the `yield` keyword.

### Why it matters:
If you are reading a 10GB log file from disk, loading it into a Python list will crash your server due to Out of Memory (OOM) errors. A generator reads one line at a time, keeping RAM consumption near 0MB.

### Code Example:
```python
# Traditional list function (Loads everything into RAM)
def count_to_three_list():
    return [1, 2, 3]

# Generator function (Generates lazily)
def count_to_three_generator():
    print("Generating 1...")
    yield 1
    print("Generating 2...")
    yield 2
    print("Generating 3...")
    yield 3

gen = count_to_three_generator()
print(gen)  # Output: <generator object ...>

# We get values using next() or a loop:
print(next(gen))  # Output: "Generating 1..." followed by 1
print(next(gen))  # Output: "Generating 2..." followed by 2
```
When a generator hits `yield`, it pauses execution and remembers its state. When `next()` is called again, it resumes exactly where it left off.
