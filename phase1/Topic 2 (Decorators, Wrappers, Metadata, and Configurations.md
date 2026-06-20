# Topic 2: Python Decorators in Depth

Decorators are one of Python's most powerful features. FastAPI uses them everywhere (e.g., `@app.get("/")`, `@app.post("/")`, `@app.middleware("http")`). Understanding how they wrap code is crucial.

---

## 1. What is a Decorator?

A **decorator** is a function that takes another function as input, extends or modifies its behavior, and returns a new function wrapper.

It relies on **First-Class Functions**, meaning functions can be:
1. Passed as arguments to other functions.
2. Returned from other functions.
3. Defined inside other functions (nested functions).

---

## 2. Step-by-Step Decorator Syntax

Let's write a simple decorator that logs when a function starts and finishes.

### The Wrapper Mechanism:
```python
def log_calls(func):
    # This inner wrapper function replaces the original function
    def wrapper(*args, **kwargs):
        print(f"[LOG] Executing function: {func.__name__}")
        
        # Execute the original function and save the output
        result = func(*args, **kwargs)
        
        print(f"[LOG] Finished executing: {func.__name__}")
        return result  # Return the output of the original function
        
    return wrapper  # Return the wrapper function object
```

### The Syntactic Sugar (`@`):
Using the `@` symbol above a function is the standard way to apply a decorator:

```python
@log_calls
def add(a: int, b: int) -> int:
    return a + b
```

This is exactly equivalent to wrapping the function manually:
```python
def add(a: int, b: int) -> int:
    return a + b

add = log_calls(add)  # 'add' is now pointing to the 'wrapper' function!
```

---

## 3. Preserving Metadata with `functools.wraps`

When you wrap a function, its name and documentation get replaced by the inner `wrapper` function:

```python
@log_calls
def calculate_area(radius):
    """Calculates the area of a circle."""
    return 3.14159 * radius * radius

print(calculate_area.__name__)  # Prints: "wrapper"
print(calculate_area.__doc__)   # Prints: None (The docstring is lost!)
```

To prevent this, you **must** use the `@wraps` decorator from the built-in `functools` module:

```python
from functools import wraps

def log_calls_improved(func):
    @wraps(func)  # Copies the name, docstring, and annotations from the original function
    def wrapper(*args, **kwargs):
        print(f"[LOG] Running {func.__name__}")
        return func(*args, **kwargs)
    return wrapper
```

---

## 4. Decorators with Arguments (Configurable Decorators)

If you want your decorator to accept configurations (like `@repeat(num_times=3)`), you need a **three-level nested function**:
1. **Outer Level:** Receives the decorator arguments.
2. **Middle Level:** Receives the target function.
3. **Inner Level:** The actual wrapper that runs.

```python
from functools import wraps

def repeat(num_times: int):
    # Middle level takes the function
    def decorator_repeat(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            last_result = None
            for _ in range(num_times):
                last_result = func(*args, **kwargs)
            return last_result
        return wrapper
    return decorator_repeat

# Usage:
@repeat(num_times=3)
def greet():
    print("Hello!")

# greet() will run 3 times
greet()
```

---

## 5. How FastAPI Uses Decorators

In FastAPI, you write:
```python
@app.get("/items")
def read_items():
    return []
```
*Note:* FastAPI does **not** modify or run wrapper codes on your route function here. Instead, it uses the decorator for **routing registration**. It says: *"Take the function `read_items` and add it to our internal routing table, mapping the path `/items` to it."* It returns your function unchanged, but saves a reference to it in the background router map.
