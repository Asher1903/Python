# Comprehensive Study Report: Topic 6 (Error Handling, Packages, and Concurrency Basics)

This study report provides an in-depth reference for Python's exception system, module structure, and concurrency mechanics, compiling all key theories and operational rules.

---

# Part 1: Error Handling in Python (Deep Mechanics)

Error handling is how we prevent our application from crashing when something goes wrong. In Python, errors are represented as **Exception Objects**.

## 1.1 The Complete `try-except-else-finally` Block
While most developers know `try` and `except`, Python actually supports a four-part structure:

```python
try:
    # 1. Code that might fail
    file = open("data.txt", "r")
    data = file.read()
except FileNotFoundError:
    # 2. Runs ONLY if a FileNotFoundError occurs
    data = "{}"
    print("File not found. Using empty database.")
else:
    # 3. Runs ONLY if the try block completed with NO exceptions
    print("File read successfully. Processing data...")
finally:
    # 4. ALWAYS runs, regardless of what happened above
    # Used for cleaning up resources
    if 'file' in locals() and not file.closed:
        file.close()
        print("File closed safely.")
```

### The Return Value Trap in `finally`
If you return a value inside the `try` block, Python pauses that return, runs the code inside `finally`, and then completes the return.
If the `finally` block itself contains a `return` or `raise` statement, **it will overwrite the previous return**. The original return value is lost.

---

## 1.2 Exception Hierarchy and Matching Rules
In Python, all exceptions inherit from a base class named `BaseException`.

```
                  BaseException
                        |
       +----------------+----------------+
       |                                 |
   Exception                    KeyboardInterrupt / SystemExit
       |                         (Used to close terminal applications)
       +--------+--------+
                |        |
    ArithmeticError   TypeError / ValueError
        |
   ZeroDivisionError
```

### The Order of `except` Blocks Matters
When an exception is raised, Python checks `except` blocks from **top to bottom**. It matches the first exception class that is the class of the raised error or a parent class of it.

```python
# WARNING: BAD ORDER
try:
    result = 10 / 0
except Exception:  # Matches EVERYTHING because Exception is the parent class
    print("Caught general exception")
except ZeroDivisionError:  # This will NEVER run!
    print("Caught division by zero")
```
*Rule:* Always list specific exceptions (children) at the top, and general exceptions (parents) at the bottom.

---

## 1.3 Custom Exceptions & Exception Chaining
*   **Custom Exceptions:** In a FastAPI backend, you should create custom exceptions for your business logic (e.g., `UserBannedError`, `InvalidCouponError`) by inheriting from the `Exception` class.
*   **Exception Chaining:** When you catch an error and want to raise a different, higher-level error, you should use `raise NewError from old_error`.

```python
class DatabaseError(Exception):
    pass

class RegistrationFailed(Exception):
    pass

def save_user_to_db():
    try:
        # Code that fails to connect to database
        raise DatabaseError("Connection refused by PG Host")
    except DatabaseError as db_err:
        # Chaining them together
        raise RegistrationFailed("Could not register user.") from db_err
```
This sets the `__cause__` attribute. In the logs, you will see both the database error and the registration failure, allowing you to trace the exact cause of the issue.

---

# Part 2: Module and Package Architecture

As your backend codebase grows beyond a single file, you must understand how Python loads and structures modules.

## 2.1 Modules vs. Packages
*   **Module:** Any single `.py` file containing code.
*   **Package:** A folder containing multiple modules. To make a folder a package, it must contain a file named `__init__.py`.

### The Role of `__init__.py`
When a package is imported, its `__init__.py` file is executed automatically. You can use it to:
1.  **Expose simple import paths:**
    If you have `models/user.py` containing a `User` class, you can write `from .user import User` inside `models/__init__.py`. Now, other files can import it cleanly as `from models import User` instead of `from models.user import User`.
2.  **Define `__all__`:**
    You can define a list of strings named `__all__` inside `__init__.py` to control what gets imported when a developer runs `from package import *`:
    ```python
    __all__ = ["User", "Post"]  # Only User and Post will be imported
    ```

---

## 2.2 How Python Locates Imports (`sys.path`)
When you run `import my_module`, Python searches for the module in directories listed in `sys.path` in this order:
1.  The directory containing the input script (your current folder).
2.  The `PYTHONPATH` environment variable (if set).
3.  Standard library directories (e.g., `os`, `sys`, `json`).
4.  Site-packages (where third-party libraries like `fastapi` are installed inside your virtual environment).

### Absolute vs. Relative Imports
*   **Absolute Import:** Uses the full path from the project root folder.
    ```python
    from app.routers import users
    ```
*   **Relative Import:** Uses dot notation relative to the current file.
    ```python
    from . import models  # Same folder
    from ..db import engine  # Parent folder
    ```
    *Warning:* Relative imports only work within packages. If you execute a file using relative imports directly (e.g., `python models.py`), it will fail with `ImportError: attempted relative import with no known parent package`.

---

## 2.3 The Circular Import Nightmare
A circular import occurs when **Module A** imports **Module B**, and **Module B** imports **Module A**.

```
[Module A] ---> (Imports B) ---> [Module B] ---> (Imports A) ---> [CRASH]
```

### Why it fails:
Python executes modules from top to bottom.
1.  Python starts executing Module A.
2.  It hits the line `import B` and pauses Module A to load Module B.
3.  It starts executing Module B and hits the line `import A`.
4.  It jumps back to Module A. But since Module A is only half-evaluated, it does not contain the variables or classes that Module B is trying to import. Python crashes with `AttributeError` or `ImportError`.

### How to resolve circular imports:
1.  **Lazy Imports (Local Imports):** Move the import statement inside the function that uses it, rather than at the top of the file:
    ```python
    # Inside Module A
    def process_data():
        from B import get_b_data  # Imported only when the function is executed
        return get_b_data()
    ```
2.  **Restructuring:** Move the shared resource that both A and B need into a third module (e.g. `C`), which both A and B can import safely.
3.  **Type Checking Guard:** If the import is only needed for type annotations, wrap it in a `TYPE_CHECKING` block:
    ```python
    from typing import TYPE_CHECKING
    if TYPE_CHECKING:
        from B import TypeB  # Evaluated only by static checkers like mypy
    ```

---

# Part 3: Concurrency Internals (Threads vs. Processes vs. Async)

Concurrency is how we handle multiple tasks at the same time. Different tasks require different concurrency models depending on where the bottleneck is.

```
                         Hardware Resource Bottleneck?
                                      |
                     +----------------+----------------+
                     |                                 |
                 CPU Bound                         I/O Bound
              (Calculations)                    (Waiting on Net/Disk)
                     |                                 |
                     v                                 v
             Multi-processing                   Async / Await
            (Separate CPU Cores)              (Event Loop / Single Thread)
```

---

## 3.1 The Global Interpreter Lock (GIL)
CPython (the standard implementation of Python) uses **Reference Counting** to manage memory. Every time an object is referenced, a counter increments; when a reference is removed, it decrements. If it hits zero, Python frees the memory.

To prevent race conditions where two threads try to change the reference count of the same object at the same time (which would cause memory leaks or crashes), CPython uses the **GIL**. 

The GIL is a single mutex lock that ensures **only one operating system thread can run Python bytecode at any given instant**.

---

## 3.2 Operating System Threads vs. Coroutines (Async)

### OS Threads (Used in Multi-threading)
*   **Management:** Scheduled by the operating system kernel.
*   **Context Switch:** Expensive. The CPU must halt execution, save the register states of the current thread, load the register states of the next thread, and resume (called *preemptive* context switching).
*   **Memory Cost:** High. Each thread requires about **8MB of RAM** just to allocate its stack memory.
*   **Impact of the GIL:** Because of the GIL, multiple threads cannot run CPU calculations in parallel. Multi-threading is only useful for synchronous waiting tasks.

### Coroutines (Used in Async / Await)
*   **Management:** Scheduled by the application code (Python's Event Loop).
*   **Context Switch:** Extremely cheap. Tasks switch context voluntarily when they hit the `await` keyword (called *cooperative* context switching). This is simply a Python function swap.
*   **Memory Cost:** Low. A coroutine is just a lightweight function object, costing only **bytes**. You can run 100,000 coroutines in the RAM space it takes to run 10 OS threads.

---

## 3.3 Bottlenecks: CPU-bound vs. I/O-bound

### 1. CPU-bound (Calculation Bottleneck)
*   *Bottleneck:* The speed of your CPU processing calculations (e.g. hashing passwords, mathematical calculations, processing image sizes).
*   *How to scale:* **Multi-processing**. Spawns completely separate processes. Each process has its own memory space and its own Python interpreter (and therefore its own GIL). This allows you to run calculations in parallel across all CPU cores.

### 2. I/O-bound (Input/Output Wait Bottleneck)
*   *Bottleneck:* Waiting for networks or disks (e.g. querying a database, waiting for an HTTP API response, reading a file). The CPU is sitting idle doing nothing.
*   *How to scale:* **Async / Await**. Since the process is spent waiting rather than calculating, a single thread running an Event Loop can manage thousands of paused connections concurrently without spawning heavy threads.

---

## 3. Concurrency Basics: Threads vs. Processes vs. Async

Understanding what to use for different performance bottlenecks:

| Concurrency Method | Execution Model | Bound Type | When to Use | GIL Constraint |
| :--- | :--- | :--- | :--- | :--- |
| **Multi-processing** | Spawns separate OS processes. Runs code on different CPU cores. | **CPU-Bound** | Image processing, machine learning models, heavy math calculations. | **None** (Each process has its own GIL) |
| **Multi-threading** | Runs threads in one process. Shares memory. | **I/O-Bound** (Sync) | Reading/writing multiple files to disk, running blocking third-party network libraries. | **Yes** (Only one thread runs Python bytecode at a time) |
| **Asynchronous (async/await)** | Single thread using an event loop. Non-blocking. | **I/O-Bound** (Async) | High-throughput web APIs, WebSockets, database calls. | **Yes** (But runs completely non-blocking, so GIL isn't a bottleneck) |

### The GIL (Global Interpreter Lock)
CPython (the standard implementation of Python) uses a mutex lock called the **GIL**. It ensures only one thread executes Python bytecode at a time. This means multi-threading in Python cannot run CPU tasks in parallel. For parallel CPU tasks, you **must** use Multi-processing.
