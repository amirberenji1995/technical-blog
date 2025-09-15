# 🧪 Measuring HTTP Call Performance in Three Styles: Sync, Async, and Async + RabbitMQ

In this post, we walk through three styles of making repeated HTTP calls and measuring their performance:

1. **Synchronous** execution using `requests`
2. **Asynchronous** execution using `httpx`
3. **Asynchronous** execution with **RabbitMQ** as a decoupled producer-consumer pipeline

#### 📢 Attention! This post is co-authored by GPT-4o from OpenAI.

---

## 🐢 1. Synchronous API Calls (Baseline)

Our first implementation is the most straightforward: make requests one after another using the `requests` library.

### How it works

* Performs requests sequentially (`for` loop)
* Uses `time()` to calculate latency
* Saves each result to a CSV file

### Implementation

```python
# sync.py
import requests
import csv
from time import time
from datetime import datetime

URL = "https://example.com"
NUM_CALLS = 10
CSV_FILE = "results/metrics_sync.csv"


def fetch(url):
    start = time()
    try:
        resp = requests.get(url)
        elapsed = round(time() - start, 2)
        return {"url": url, "status": resp.status_code, "elapsed": elapsed, "error": ""}
    except Exception as e:
        elapsed = round(time() - start, 2)
        return {"url": url, "status": None, "elapsed": elapsed, "error": str(e)}


def run():
    with open(CSV_FILE, "w", newline="") as f:
        writer = csv.DictWriter(
            f, fieldnames=["timestamp", "url", "status", "elapsed", "error"]
        )
        writer.writeheader()
        for _ in range(NUM_CALLS):
            result = fetch(URL)
            result["timestamp"] = datetime.now().isoformat()
            writer.writerow(result)


if __name__ == "__main__":
    run()
```

---

## ⚡ 2. Asynchronous API Calls Using `httpx`

To improve performance, we switch to an async implementation using `httpx.AsyncClient`.

### How it works

* Sends requests concurrently using `asyncio.gather`
* Measures individual latency using `perf_counter`
* Still logs each result to a CSV

### Implementation

```python
# async.py
import httpx
import asyncio
import csv
from datetime import datetime
from time import perf_counter

URL = "https://example.com"
NUM_CALLS = 10
CSV_FILE = "results/metrics_async.csv"


async def fetch(client, url):
    start = perf_counter()
    try:
        resp = await client.get(url)
        elapsed = round(perf_counter() - start, 2)
        return {"url": url, "status": resp.status_code, "elapsed": elapsed, "error": ""}
    except Exception as e:
        elapsed = round(perf_counter() - start, 2)
        return {"url": url, "status": None, "elapsed": elapsed, "error": str(e)}


async def run():
    async with httpx.AsyncClient() as client:
        tasks = [fetch(client, URL) for _ in range(NUM_CALLS)]
        results = await asyncio.gather(*tasks)

    with open(CSV_FILE, "w", newline="") as f:
        writer = csv.DictWriter(
            f, fieldnames=["timestamp", "url", "status", "elapsed", "error"]
        )
        writer.writeheader()
        for result in results:
            result["timestamp"] = datetime.now().isoformat()
            writer.writerow(result)


if __name__ == "__main__":
    asyncio.run(run())
```

---

## 🧵 3. Asynchronous API Calls via RabbitMQ Queue

Here, we decouple the triggering of requests from their execution. A **producer** sends messages to a RabbitMQ queue; an **async consumer** fetches and processes them.

### Producer Implementation

```python
# producer.py
import pika
import json

URL = "https://example.com"
NUM_CALLS = 10

credentials = pika.PlainCredentials("admin", "admin123")
params = pika.ConnectionParameters("localhost", credentials=credentials)
connection = pika.BlockingConnection(params)
channel = connection.channel()
channel.queue_declare(queue="metrics")

for _ in range(NUM_CALLS):
    msg = json.dumps({"url": URL})
    channel.basic_publish(exchange="", routing_key="metrics", body=msg)
    print("Published:", msg)

connection.close()
```

### Consumer Implementation

```python
# async_consumer_with_metrics.py
import aio_pika
import asyncio
import httpx
import csv
from datetime import datetime
from time import perf_counter
import json
import signal

CSV_FILE = "results/metrics_async_rabbitmq.csv"
results = []
stop_event = asyncio.Event()


async def fetch(url):
    start = perf_counter()
    try:
        async with httpx.AsyncClient() as client:
            resp = await client.get(url)
            elapsed = round(perf_counter() - start, 2)
            return {
                "url": url,
                "status": resp.status_code,
                "elapsed": elapsed,
                "error": "",
            }
    except Exception as e:
        elapsed = round(perf_counter() - start, 2)
        return {"url": url, "status": None, "elapsed": elapsed, "error": str(e)}


async def handle_message(message: aio_pika.IncomingMessage):
    async with message.process():
        payload = json.loads(message.body.decode())
        result = await fetch(payload["url"])
        result["timestamp"] = datetime.now().isoformat()
        results.append(result)


async def write_results():
    with open(CSV_FILE, "w", newline="") as f:
        writer = csv.DictWriter(
            f, fieldnames=["timestamp", "url", "status", "elapsed", "error"]
        )
        writer.writeheader()
        for row in results:
            writer.writerow(row)


async def shutdown(loop, connection):
    print("⏹️ Shutting down gracefully...")
    stop_event.set()
    await write_results()
    await connection.close()
    loop.stop()


async def main():
    connection = await aio_pika.connect_robust("amqp://admin:admin123@localhost/")
    channel = await connection.channel()
    queue = await channel.declare_queue("metrics")

    await queue.consume(handle_message)

    loop = asyncio.get_running_loop()
    for sig in (signal.SIGINT, signal.SIGTERM):
        loop.add_signal_handler(
            sig, lambda: asyncio.create_task(shutdown(loop, connection))
        )

    print("✅ Consumer is running. Press Ctrl+C to stop.")
    await stop_event.wait()
---

Let me know if you want:

* Visualizations (bar charts, latency histograms)
* More test cases (e.g., failures, timeouts)
* Integration with FastAPI or other frameworks



if __name__ == "__main__":
    try:
        asyncio.run(main())
    except KeyboardInterrupt:
        pass
```

---

## 🐇 Setting Up RabbitMQ with Docker

To run RabbitMQ locally for testing, use the following `docker-compose.yml`:

```yaml
services:
  rabbitmq:
    image: rabbitmq:4-management-alpine
    restart: unless-stopped
    environment:
      - RABBITMQ_DEFAULT_USER=admin
      - RABBITMQ_DEFAULT_PASS=admin123
    ports:
      - "5672:5672"   # AMQP protocol
      - "15672:15672" # Web UI
```

Then run:

```bash
docker-compose up -d
```

You can access the management dashboard at [http://localhost:15672](http://localhost:15672).

---

## 📦 Required Dependencies

Here’s your `requirements.txt`:

```txt
requests==2.31.0
httpx==0.27.0
aio-pika==9.4.1
pika==1.3.2
```

> 📝 No need to list `asyncio` — it’s included in Python's standard library (3.4+).

---

## 📊 Performance Comparison

### Synchronous Results

| Total Time | Avg Latency | Fastest | Slowest |
| ---------- | ----------- | ------- | ------- |
| \~9.57s    | 0.96s       | 0.57s   | 1.86s   |

### Asynchronous Results

| Total Time | Avg Latency | Fastest | Slowest |
| ---------- | ----------- | ------- | ------- |
| \~2.1s     | 0.91s       | 0.78s   | 2.1s    |

### Async + RabbitMQ Results

| Total Time | Avg Latency | Fastest | Slowest |
| ---------- | ----------- | ------- | ------- |
| \~5.67s    | 1.36s       | 0.65s   | 5.67s\* |

> \*One request spiked in latency (likely queue delay). Others were processed rapidly.

---

## 🧠 Discussion

* **Sync**: Very simple but very slow due to blocking I/O.
* **Async**: Achieves massive speedup by making concurrent requests.
* **RabbitMQ**: Adds flexibility and decoupling. While not faster than pure async, it enables scalable architectures with separate producers/consumers.
