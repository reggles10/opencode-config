# Python Async Patterns

Production patterns for asyncio and concurrent Python.

## Core Async Patterns

### Basic Async Function

```python
import asyncio

async def fetch_data(url: str) -> dict:
    """Async function that performs I/O."""
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.json()

# Running async code
async def main():
    data = await fetch_data("https://api.example.com/data")
    print(data)

asyncio.run(main())
```

### Concurrent Execution

```python
# Run multiple coroutines concurrently
async def fetch_all(urls: list[str]) -> list[dict]:
    tasks = [fetch_data(url) for url in urls]
    return await asyncio.gather(*tasks)

# With error handling
async def fetch_all_safe(urls: list[str]) -> list[dict | Exception]:
    tasks = [fetch_data(url) for url in urls]
    return await asyncio.gather(*tasks, return_exceptions=True)
```

### Task Groups (Python 3.11+)

```python
async def process_items(items: list[str]) -> list[str]:
    """Process items with structured concurrency."""
    results = []
    
    async with asyncio.TaskGroup() as tg:
        for item in items:
            task = tg.create_task(process_item(item))
            results.append(task)
    
    return [task.result() for task in results]
```

## Error Handling

### Timeout Pattern

```python
async def fetch_with_timeout(url: str, timeout: float = 10.0) -> dict:
    """Fetch with timeout protection."""
    try:
        async with asyncio.timeout(timeout):
            return await fetch_data(url)
    except asyncio.TimeoutError:
        raise TimeoutError(f"Request to {url} timed out after {timeout}s")
```

### Retry Pattern

```python
import asyncio
from typing import TypeVar, Callable, Awaitable

T = TypeVar('T')

async def retry_async(
    func: Callable[[], Awaitable[T]],
    max_retries: int = 3,
    base_delay: float = 1.0,
    max_delay: float = 60.0,
    exponential_base: float = 2.0,
) -> T:
    """Retry async function with exponential backoff."""
    last_exception = None
    
    for attempt in range(max_retries):
        try:
            return await func()
        except Exception as e:
            last_exception = e
            if attempt < max_retries - 1:
                delay = min(base_delay * (exponential_base ** attempt), max_delay)
                await asyncio.sleep(delay)
    
    raise last_exception

# Usage
result = await retry_async(lambda: fetch_data(url), max_retries=3)
```

### Graceful Shutdown

```python
import signal

async def shutdown(signal, loop):
    """Cleanup tasks tied to the service's shutdown."""
    print(f"Received exit signal {signal.name}...")
    
    tasks = [t for t in asyncio.all_tasks() if t is not asyncio.current_task()]
    
    for task in tasks:
        task.cancel()
    
    await asyncio.gather(*tasks, return_exceptions=True)
    loop.stop()

def main():
    loop = asyncio.get_event_loop()
    
    for sig in (signal.SIGTERM, signal.SIGINT):
        loop.add_signal_handler(
            sig, lambda s=sig: asyncio.create_task(shutdown(s, loop))
        )
    
    try:
        loop.run_until_complete(run_server())
    finally:
        loop.close()
```

## Concurrency Control

### Semaphore - Limit Concurrent Operations

```python
async def fetch_with_limit(urls: list[str], max_concurrent: int = 10) -> list[dict]:
    """Fetch URLs with concurrency limit."""
    semaphore = asyncio.Semaphore(max_concurrent)
    
    async def fetch_one(url: str) -> dict:
        async with semaphore:
            return await fetch_data(url)
    
    return await asyncio.gather(*[fetch_one(url) for url in urls])
```

### Lock - Mutual Exclusion

```python
class AsyncCounter:
    def __init__(self):
        self._value = 0
        self._lock = asyncio.Lock()
    
    async def increment(self) -> int:
        async with self._lock:
            self._value += 1
            return self._value
```

### Event - Signaling

```python
async def waiter(event: asyncio.Event):
    print("Waiting for event...")
    await event.wait()
    print("Event received!")

async def setter(event: asyncio.Event):
    await asyncio.sleep(2)
    event.set()
    print("Event set!")

async def main():
    event = asyncio.Event()
    await asyncio.gather(waiter(event), setter(event))
```

## Queue Patterns

### Producer-Consumer

```python
async def producer(queue: asyncio.Queue, items: list):
    """Produce items to queue."""
    for item in items:
        await queue.put(item)
    await queue.put(None)  # Sentinel to signal completion

async def consumer(queue: asyncio.Queue, worker_id: int):
    """Consume items from queue."""
    while True:
        item = await queue.get()
        if item is None:
            queue.task_done()
            break
        
        await process_item(item)
        queue.task_done()

async def main():
    queue = asyncio.Queue(maxsize=100)
    
    # Start producer and consumers
    producer_task = asyncio.create_task(producer(queue, items))
    consumer_tasks = [
        asyncio.create_task(consumer(queue, i))
        for i in range(num_workers)
    ]
    
    await producer_task
    await queue.join()
    
    # Cancel consumers
    for task in consumer_tasks:
        task.cancel()
```

### Bounded Queue with Backpressure

```python
async def rate_limited_producer(queue: asyncio.Queue, items: list):
    """Producer that respects queue capacity."""
    for item in items:
        await queue.put(item)  # Blocks if queue is full
        print(f"Produced: {item}, queue size: {queue.qsize()}")
```

## Context Managers

### Async Context Manager

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def managed_resource():
    """Async context manager for resource management."""
    resource = await acquire_resource()
    try:
        yield resource
    finally:
        await release_resource(resource)

# Usage
async with managed_resource() as resource:
    await resource.do_something()
```

### Database Connection Pool

```python
import asyncpg

class DatabasePool:
    def __init__(self, dsn: str, min_size: int = 5, max_size: int = 20):
        self.dsn = dsn
        self.min_size = min_size
        self.max_size = max_size
        self._pool = None
    
    async def connect(self):
        self._pool = await asyncpg.create_pool(
            self.dsn,
            min_size=self.min_size,
            max_size=self.max_size
        )
    
    async def close(self):
        if self._pool:
            await self._pool.close()
    
    async def execute(self, query: str, *args):
        async with self._pool.acquire() as conn:
            return await conn.execute(query, *args)
    
    async def fetch(self, query: str, *args):
        async with self._pool.acquire() as conn:
            return await conn.fetch(query, *args)
```

## Streaming Patterns

### Async Generator

```python
async def stream_data(url: str):
    """Stream data as async generator."""
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            async for chunk in response.content.iter_chunked(1024):
                yield chunk

# Usage
async for chunk in stream_data(url):
    process_chunk(chunk)
```

### Async Iterator

```python
class AsyncPaginator:
    def __init__(self, client, endpoint: str, page_size: int = 100):
        self.client = client
        self.endpoint = endpoint
        self.page_size = page_size
        self.page = 0
        self.exhausted = False
    
    def __aiter__(self):
        return self
    
    async def __anext__(self):
        if self.exhausted:
            raise StopAsyncIteration
        
        data = await self.client.get(
            self.endpoint,
            params={'page': self.page, 'size': self.page_size}
        )
        
        if not data:
            self.exhausted = True
            raise StopAsyncIteration
        
        self.page += 1
        return data

# Usage
async for page in AsyncPaginator(client, '/api/items'):
    for item in page:
        process_item(item)
```

## Common Anti-Patterns

| Anti-Pattern | Problem | Solution |
|--------------|---------|----------|
| `asyncio.run()` in async | Nested event loops | Use `await` directly |
| Blocking calls in async | Blocks event loop | Use `run_in_executor` |
| No timeout on I/O | Hangs forever | Use `asyncio.timeout()` |
| Unbounded concurrency | Resource exhaustion | Use semaphore |
| Fire-and-forget tasks | Lost exceptions | Track tasks, handle errors |
| Sync locks in async | Deadlocks | Use `asyncio.Lock` |

### Blocking Call in Async

```python
# BAD: Blocks the event loop
async def bad_example():
    result = requests.get(url)  # Blocking!
    return result

# GOOD: Use async library
async def good_example():
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.json()

# GOOD: Run blocking code in executor
async def good_example_executor():
    loop = asyncio.get_event_loop()
    result = await loop.run_in_executor(None, requests.get, url)
    return result
```
