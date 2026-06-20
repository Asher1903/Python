# Topic 4: Async/Await & Asynchronous Programming

FastAPI is built from the ground up as an asynchronous framework. Understanding how `async` and `await` operate is the single most important concept for building high-performance APIs.

---

## 1. Synchronous vs. Asynchronous (Conceptual)

### Synchronous (Sync) Code: "Single-File Queue"
In synchronous programming, your code runs line-by-line. If a line of code has to wait for something slow (like a database query, reading a file, or hitting an external API), the entire server freezes and waits.

```
Request 1 ---> [ DB Query (200ms) ] ===(Server Blocks)===> Response 1
Request 2 ---> [ Must wait for Request 1 to finish ] ---> Response 2
```
*To handle multiple users concurrently, sync servers must spawn multiple heavy operating system threads or processes, which consumes significant memory.*

### Asynchronous (Async) Code: "The Juggler"
In asynchronous programming, when your code reaches a slow operation, it "yields" control. The server doesn't sit idle; it works on other incoming requests while waiting for the database or network to respond.

```
Request A ---> [ Start DB Query ] --(Yield Control)
                                        | ---> Event Loop runs Request B
Request A <--- [ DB Finished ] <--------+ ---> Response A
```
*An async server can handle thousands of concurrent requests on a **single CPU thread** because it never wastes time waiting.*

---

## 2. Key Terms: What is What?

### A. Coroutine (`async def`)
A function defined with `async def` is called a **coroutine**. 

When you call an async function, it **does not run the code inside**. Instead, it returns a **Coroutine Object** (which represents a task that will run in the future).

```python
async def get_user_data():
    return {"user": "Alice"}

# Calling this does NOT print the dictionary!
coro = get_user_data()
print(coro)  # Output: <coroutine object get_user_data at 0x...>
```

### B. The `await` Keyword
To actually run a coroutine and get its returned data, you must **`await`** it. 
*   `await` tells Python: *"Pause this function here, hand control back to the Event Loop, and don't wake me up until this task is completed and has returned its data."*
*   **Important rule:** You can **only** use `await` inside an `async def` function.

```python
async def main():
    # Execute the coroutine and get the dictionary
    data = await get_user_data()
    print(data)  # Output: {"user": "Alice"}
```

### C. The Event Loop
The **Event Loop** is the engine running in the background. It is a continuous loop that monitors active tasks. If Task A is waiting for a database response, the Event Loop pauses Task A and executes Task B. When the database responds, the loop resumes Task A.

---

## 3. The Golden Rule of Async

> [!WARNING]
> **NEVER run blocking (synchronous) code inside an `async def` function.**

If you run a blocking operation inside an async function, you block the **Event Loop itself**. Since the loop is single-threaded, the entire server freezes, and **no other user can connect** until the blocking operation completes.

### The Bad Way (Blocks the whole server):
```python
import time

async def fetch_profile():
    # time.sleep is synchronous. It blocks the entire thread running the event loop!
    # During these 2 seconds, the server cannot respond to any other user.
    time.sleep(2) 
    return {"status": "done"}
```

### The Good Way (Yields control):
```python
import asyncio

async def fetch_profile():
    # asyncio.sleep is asynchronous. It pauses this function and yields control.
    # During these 2 seconds, the event loop can process hundreds of other requests!
    await asyncio.sleep(2) 
    return {"status": "done"}
```

---

## 4. Run it Yourself (Executable Example)

Create a file named `run_async.py` in this directory or run the code block below to see the difference between synchronous blocking and asynchronous concurrency:

```python
import asyncio
import time

async def fetch_data(task_id: int, delay: int):
    print(f"Task {task_id}: Started (Waiting {delay}s...)")
    # Yield control to the event loop
    await asyncio.sleep(delay)
    print(f"Task {task_id}: Finished!")
    return f"Result {task_id}"

async def main():
    start_time = time.perf_counter()
    
    # Run three tasks concurrently (at the same time)
    results = await asyncio.gather(
        fetch_data(1, 2),
        fetch_data(2, 3),
        fetch_data(3, 1)
    )
    
    end_time = time.perf_counter()
    print(f"\nAll tasks completed in {end_time - start_time:.2f} seconds!")
    print(f"Results: {results}")

# Run the Event Loop
asyncio.run(main())
```

### Expected Output:
Even though the sum of wait times is 6 seconds (2s + 3s + 1s), they run concurrently, completing in **exactly 3 seconds** (the time of the longest task):
```text
Task 1: Started (Waiting 2s...)
Task 2: Started (Waiting 3s...)
Task 3: Started (Waiting 1s...)
Task 3: Finished!
Task 1: Finished!
Task 2: Finished!

All tasks completed in 3.00 seconds!
Results: ['Result 1', 'Result 2', 'Result 3']
```
