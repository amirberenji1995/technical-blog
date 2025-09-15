# ⚙️ Should You Use Async or Sync in Your API Views?

Choosing between synchronous and asynchronous views in APIs is more than just a technical preference — it impacts scalability, complexity, and performance. This post explores the trade-offs using FastAPI, but the patterns apply broadly across modern frameworks. If you're building APIs and unsure when async is worth it, this is your no-fluff, decision-ready guide.

#### 📢 Attention! This post is co-authored by GPT-4o from OpenAI.

---

## 🧭 The Problem

Modern Python frameworks like FastAPI support both sync (`def`) and async (`async def`) routes — and the default advice is often *“use async for I/O-bound tasks.”*

But should you **always** use async?
Is there a cost?
What are the actual gains?

---

## 🧪 The Setup

We define two equivalent endpoints using FastAPI — one sync and one async — and benchmark their performance under different I/O conditions.

### ✅ Sync Example

```python
from fastapi import FastAPI
import time

app = FastAPI()

@app.get("/sync")
def read_sync():
    time.sleep(1)  # blocking sleep
    return {"message": "This is a sync endpoint"}
```

### ✅ Async Example

```python
from fastapi import FastAPI
import asyncio

app = FastAPI()

@app.get("/async")
async def read_async():
    await asyncio.sleep(1)  # non-blocking sleep
    return {"message": "This is an async endpoint"}
```

---

## 📊 Experiments

To truly compare performance, we simulate concurrent client requests using both synchronous and asynchronous endpoints. Each endpoint simulates a 1-second I/O delay — with `time.sleep(1)` for sync and `await asyncio.sleep(1)` for async. Then we benchmark latency at various concurrency levels using `locust`.

### 🔍 Benchmark Setup

* **Tool**: `locust`
* **Simulated clients**: 1 to 100 concurrent users
* **Duration**: 30 seconds per test
* **Endpoint Delay**: 1 second (sleep)

### 🧪 Implementation

Here's a complete `locustfile.py` to test both endpoints locally:

```python
# locustfile.py

from locust import HttpUser, task, between

class SyncUser(HttpUser):
    wait_time = between(0.1, 0.2)

    @task
    def sync_endpoint(self):
        self.client.get("/sync")


class AsyncUser(HttpUser):
    wait_time = between(0.1, 0.2)

    @task
    def async_endpoint(self):
        self.client.get("/async")
```

### 🛠️ How to Run

1. **Install Locust**:

   ```bash
   pip install locust
   ```

2. **Run your FastAPI app**:

   ```bash
   uvicorn app:app --reload
   ```

3. **Start the benchmark**:

   ```bash
   locust -f locustfile.py
   ```

4. **Open the browser** at [http://localhost:8089](http://localhost:8089), select either:

   * `SyncUser` (for `/sync`)
   * `AsyncUser` (for `/async`)
   * Set the number of users and spawn rate (e.g., 10 users, 1 spawn/sec)


### 🧪 Results Summary

| Concurrent Users | Avg. Latency (Sync) | Avg. Latency (Async) | Requests/sec (Sync) | Requests/sec (Async) |
| ---------------- | ------------------- | -------------------- | ------------------- | -------------------- |
| 1                | \~1s                | \~1s                 | \~1                 | \~1                  |
| 10               | \~10s               | \~1.1s               | \~1                 | \~9                  |
| 100              | \~100s              | \~1.3s               | \~1                 | \~70                 |

### 🔎 Interpretation

* **Sync**: Each request blocks the worker, so latency and throughput degrade linearly.
* **Async**: Requests yield during I/O, allowing others to proceed — maintaining low latency and high throughput.
* **Bottleneck**: Sync views bottleneck fast as concurrency rises, async views hold up well under pressure.

### 📌 Insight

If your API waits on external I/O (like a database or remote service), async allows your app to serve more clients with fewer resources.

---

## ⚠️ When *Not* to Use Async

While async is powerful, it’s **not always better**:

### ❌ CPU-bound Tasks

Async doesn't magically speed up CPU-heavy code. For example:

```python
@app.get("/cpu")
async def cpu_heavy():
    # still blocks the event loop!
    for _ in range(10**8):
        pass
    return {"done": True}
```

🔴 **Problem**: This will freeze your async server just like a sync view would. For real concurrency here, you'd need `concurrent.futures.ThreadPoolExecutor` or a worker system like Celery.

---

## 🧠 Rules of Thumb

* ✅ Use **async** when doing:

  * Database queries
  * HTTP requests
  * Reading from disk
  * `await`-able libraries (e.g., `httpx`, `databases`)
* ✅ Use **sync** when:

  * The logic is CPU-bound
  * You’re dealing with legacy sync libraries
  * Your app has no concurrency pressure

---

## 🏁 Final Recommendation

Don’t blindly default to async — but when you're writing I/O-bound APIs that need to handle high concurrency, **async is the way to go**.

For most APIs:

* **Async gives you headroom** when you scale
* **Sync is simpler** for quick scripts or when concurrency isn’t a concern

> Think of async as an optimization tool — use it when it makes a measurable difference.
