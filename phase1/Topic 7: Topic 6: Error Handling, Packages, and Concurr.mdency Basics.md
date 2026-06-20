# Topic 6: Error Handling, Packages, and Concurrency Basics

This guide covers error handling flow control, module structures, virtual environments, and compares threads, processes, and async programming.

---

## 1. Error Handling

Web APIs must be resilient. If a crash occurs (e.g. database goes offline), your server should log the error and return a clean HTTP `500` error to the user rather than crashing the script.

### 1.1 Try / Except / Finally
```python
def read_configuration(file_path):
    file = None
    try:
        file = open(file_path, "r")
        return file.read()
    except FileNotFoundError as exc:
        print(f"Error: Configuration file not found! ({exc})")
        return "{}"
    finally:
        # 'finally' always runs, regardless of whether an error was raised or not.
        # Excellent for closing files or cleanups.
        if file:
            file.close()
            print("File handle closed.")
```

### 1.2 Custom Exceptions & Exception Chains
You can create custom exceptions by inheriting from the base `Exception` class. Use `raise ... from ...` to chain exceptions (tracking the original cause of an error).

```python
class DatabaseConnectionError(Exception):
    """Raised when the database connection fails."""
    pass

def connect_db():
    try:
        # Simulating connection error
        import socket
        socket.create_connection(("localhost", 9999), timeout=1)
    except Exception as original_error:
        # Exception chaining: Raises DatabaseConnectionError caused by original_error
        raise DatabaseConnectionError("Failed to connect to PG database") from original_error
```

---

## 2. Modules, Packages & Virtual Environments

### 2.1 Imports & `__init__.py`
*   **Module**: Any `.py` file containing code.
*   **Package**: A folder containing modules.
*   **`__init__.py`**: 
    *   Tells Python that the directory is a package, allowing you to import modules from it.
    *   It is executed when the package is imported. You can use it to expose package-level APIs (e.g., importing models inside `__init__.py` so users can import them directly from the folder namespace).

### 2.2 Virtual Environments
A sandbox directory containing its own Python binary and library installs.
*   *Why*: Keeps dependencies isolated per project.
*   *Commands*:
    *   `python -m venv .venv` (Create sandbox)
    *   `.venv\Scripts\Activate.ps1` (Activate on Windows PowerShell)

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
