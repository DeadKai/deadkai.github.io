+++
title = "Async Programming in Python: A Practical Guide"
date = 2024-12-10T16:45:00Z
+++

Asynchronous programming in Python has evolved significantly with the introduction of `async` and `await` keywords. Understanding when and how to use async can dramatically improve your application's performance.

## What is Async Programming?

Async programming allows your program to handle multiple operations concurrently without using multiple threads. This is particularly valuable for I/O-bound operations like:

- Network requests
- Database queries
- File operations
- API calls

## Sync vs Async: A Simple Example

### Synchronous Code

```python
import time
import requests

def fetch_url(url):
    response = requests.get(url)
    return response.text

def main():
    urls = ['https://api.example.com/1', 'https://api.example.com/2']
    start = time.time()

    for url in urls:
        data = fetch_url(url)
        print(f"Fetched {len(data)} bytes")

    print(f"Time taken: {time.time() - start:.2f}s")

# Time taken: ~4.0s (2 seconds per request)
```

### Asynchronous Code

```python
import asyncio
import aiohttp
import time

async def fetch_url(session, url):
    async with session.get(url) as response:
        return await response.text()

async def main():
    urls = ['https://api.example.com/1', 'https://api.example.com/2']
    start = time.time()

    async with aiohttp.ClientSession() as session:
        tasks = [fetch_url(session, url) for url in urls]
        results = await asyncio.gather(*tasks)

        for data in results:
            print(f"Fetched {len(data)} bytes")

    print(f"Time taken: {time.time() - start:.2f}s")

asyncio.run(main())
# Time taken: ~2.0s (requests happen concurrently!)
```

The async version completes in half the time because requests run concurrently.

## Core Concepts

### Coroutines

Functions defined with `async def` are coroutines. They don't execute immediately when called—they return a coroutine object:

```python
async def greet(name):
    return f"Hello, {name}"

# This doesn't print anything:
coro = greet("Alice")

# This executes the coroutine:
result = await greet("Alice")  # Must be inside async function
```

### The Event Loop

The event loop manages and executes async tasks. It's the core of async programming:

```python
# Python 3.7+
asyncio.run(main())

# Older approach
loop = asyncio.get_event_loop()
loop.run_until_complete(main())
loop.close()
```

### await Keyword

`await` pauses the coroutine until the awaited operation completes, allowing other coroutines to run:

```python
async def process_data():
    data = await fetch_data()  # Waits here
    result = await transform_data(data)  # Then waits here
    return result
```

### Creating Tasks

Tasks allow coroutines to run concurrently:

```python
async def main():
    # These run concurrently
    task1 = asyncio.create_task(fetch_data(1))
    task2 = asyncio.create_task(fetch_data(2))

    result1 = await task1
    result2 = await task2
```

## Common Patterns

### Gathering Multiple Coroutines

```python
async def main():
    results = await asyncio.gather(
        fetch_data(1),
        fetch_data(2),
        fetch_data(3)
    )
    # All three run concurrently
    print(results)  # [result1, result2, result3]
```

### Handling Timeouts

```python
async def fetch_with_timeout(url):
    try:
        async with aiohttp.ClientSession() as session:
            async with asyncio.timeout(5.0):  # Python 3.11+
                response = await session.get(url)
                return await response.text()
    except asyncio.TimeoutError:
        return None
```

### Running Background Tasks

```python
async def background_task():
    while True:
        await asyncio.sleep(60)
        await cleanup_old_data()

async def main():
    # Start background task
    task = asyncio.create_task(background_task())

    # Do other work
    await handle_requests()

    # Cancel background task when done
    task.cancel()
```

### Async Context Managers

```python
class AsyncDatabase:
    async def __aenter__(self):
        self.connection = await create_connection()
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        await self.connection.close()

async def main():
    async with AsyncDatabase() as db:
        data = await db.query("SELECT * FROM users")
```

## Common Pitfalls

### Blocking the Event Loop

```python
# BAD: This blocks the entire event loop
async def bad_sleep():
    time.sleep(5)  # Blocks everything!

# GOOD: Use async sleep
async def good_sleep():
    await asyncio.sleep(5)  # Allows other tasks to run
```

### Forgetting to await

```python
# BAD: Coroutine never executes
async def process():
    fetch_data()  # Missing await!

# GOOD
async def process():
    await fetch_data()
```

### Mixing Sync and Async

```python
# BAD: Can't use await outside async function
def sync_function():
    result = await async_function()  # SyntaxError!

# GOOD: Use asyncio.run() or make function async
async def async_function_caller():
    result = await async_function()
```

## When to Use Async

### Good Use Cases

- **I/O-bound operations**: Network requests, database queries, file operations
- **Many concurrent operations**: Handling hundreds of simultaneous connections
- **Real-time applications**: WebSockets, chat servers, streaming

### Bad Use Cases

- **CPU-bound operations**: Heavy computation, image processing, data crunching
- **Simple scripts**: Async adds complexity for minimal benefit
- **Legacy codebases**: Retrofitting async can be challenging

For CPU-bound tasks, use `multiprocessing` instead:

```python
from multiprocessing import Pool

def cpu_intensive(n):
    return sum(i*i for i in range(n))

with Pool(4) as p:
    results = p.map(cpu_intensive, [10**7, 10**7, 10**7, 10**7])
```

## Real-World Example: Web Scraper

```python
import asyncio
import aiohttp
from bs4 import BeautifulSoup

async def fetch_page(session, url):
    async with session.get(url) as response:
        return await response.text()

async def parse_page(html):
    soup = BeautifulSoup(html, 'html.parser')
    return soup.find_all('a', href=True)

async def scrape_site(urls):
    async with aiohttp.ClientSession() as session:
        tasks = [fetch_page(session, url) for url in urls]
        pages = await asyncio.gather(*tasks)

        all_links = []
        for html in pages:
            links = await parse_page(html)
            all_links.extend(links)

        return all_links

urls = [f'https://example.com/page{i}' for i in range(10)]
links = asyncio.run(scrape_site(urls))
print(f"Found {len(links)} links")
```

## Conclusion

Async programming in Python is powerful but not a silver bullet. Key takeaways:

- Use async for I/O-bound operations
- Understand the event loop and how it schedules coroutines
- Always `await` your coroutines
- Don't block the event loop with synchronous code
- Consider whether the complexity is worth it for your use case

When used appropriately, async can dramatically improve application performance and responsiveness. Start with simple examples, understand the fundamentals, and gradually incorporate async patterns where they provide clear benefits.
