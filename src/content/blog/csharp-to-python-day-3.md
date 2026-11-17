---
title: "From C# Tech Lead to Forward Deployed Engineer: Day 3 - FastAPI through ASP.NET Core eyes"
description: "Building a CRUD API with FastAPI: model binding, response models, dependency injection without a container, and the async lesson that matters most for AI services."
pubDate: 2026-11-16
draft: false
---

[Day 2](./csharp-to-python-day-2) ended with a Pydantic DTO for a motor-insurance quote. Day 3 put it behind an API using **FastAPI**, which is Python's closest equivalent to ASP.NET Core minimal APIs.

## A detour first: Python 3.15 was too new

```bash
uv add "fastapi[standard]"
```

This failed with a wall of build output ending in:

```text
error: Microsoft Visual C++ 14.0 or greater is required.
```

Some Python packages contain C code. Normally you download a prebuilt binary called a **wheel**, compiled for your Python version and OS. I was on Python 3.15.2, which was so new that `pyyaml` hadn't published a Windows wheel for it, so uv fell back to compiling from source.

The .NET equivalent is using a preview SDK before your NuGet packages have shipped builds for it. The fix was to move to a mature version:

```bash
# in pyproject.toml: requires-python = ">=3.13"
uv python pin 3.13
uv add "fastapi[standard]"
```

I hadn't even asked for `pyyaml`. It was a **transitive dependency**, and uv's error showed the chain:

```text
fde-practice → fastapi[standard] → uvicorn[standard] → pyyaml
```

`uvicorn` is the web server that runs FastAPI, roughly what Kestrel is to ASP.NET Core. `uv tree` shows the whole dependency graph, like `dotnet list package --include-transitive`.

The lesson: **production projects use a mature Python version, not the newest one.** Customer environments are usually a version or two behind.

## First endpoint

In C#:

```csharp
app.MapGet("/quotes/{id}", (int id) => new { Id = id, Status = "found" });
```

In FastAPI:

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/quotes/{quote_id}")
def get_quote(quote_id: int):
    return {"quote_id": quote_id, "status": "found"}
```

```bash
uv run fastapi dev api.py
```

Two things stood out:

- **`/docs` comes for free.** Swagger UI is generated from the type hints, with no Swashbuckle and no `AddSwaggerGen()`.
- **Type hints are enforced at the API boundary.** `/quotes/abc` returns a `422` with the same structured error as Pydantic, because FastAPI uses Pydantic for all validation:

```json
{
  "detail": [{
    "type": "int_parsing",
    "loc": ["path", "quote_id"],
    "msg": "Input should be a valid integer, unable to parse string as an integer",
    "input": "abc"
  }]
}
```

`loc` starts with `"path"`, telling the client the bad value was in the URL. (My first attempt returned `{"detail":"Not Found"}` because I'd opened the root URL, which had no route.)

## Request bodies: model binding without attributes

```python
@app.post("/quotes")
def create_quote(request: QuoteRequestDto):
    ...
```

Because the parameter is a Pydantic model, FastAPI reads it from the JSON body, like `[FromBody]` without the attribute. A 15-year-old driver gave:

```json
"loc": ["body", "drivers", 0, "age"]
```

That's the nested path from Day 2, with `"body"` added at the front.

## 422 vs 400

ASP.NET Core with `[ApiController]` returns **400** for validation failures; FastAPI returns **422**. In HTTP terms, 400 means "malformed request" and 422 means "well-formed but invalid values". Neither is wrong, but customers often have an existing contract that expects 400. A global handler changes it, the equivalent of an exception filter:

```python
from fastapi import Request
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse

@app.exception_handler(RequestValidationError)
async def validation_handler(request: Request, exc: RequestValidationError):
    return JSONResponse(status_code=400, content={"errors": exc.errors()})
```

My first try failed with `NameError: name 'Request' is not defined`. I'd missed the import, which is the Day 1 "top to bottom" rule again.

## Importing a file runs it

After importing my Day 2 DTO into the API, the server printed my Day 2 test output every time it reloaded. In Python, **importing a module executes the whole file**, including any test code at the bottom. C# never does this.

The standard fix:

```python
if __name__ == "__main__":
    # runs only with `uv run quote_request.py`, not on import
    ...
```

Python sets `__name__` to `"__main__"` only when a file is run directly. Think of it as `Main()`, but only when this file is the entry point.

## CRUD, and why Python needs `global`

The first in-memory version:

```python
quotes: dict[int, QuoteRequestDto] = {}
next_quote_id = 1

@app.post("/quotes", status_code=201)
def create_quote(request: QuoteRequestDto):
    global next_quote_id
    quote_id = next_quote_id
    quotes[quote_id] = request
    next_quote_id += 1
    ...
```

`global` confused me at first. The rule exists because **Python has no `var` keyword**: `x = 1` both declares and assigns, so Python decides that any name *assigned* inside a function is local to it. Without `global`, `next_quote_id += 1` raises `UnboundLocalError`.

The subtle part: `quotes[quote_id] = request` doesn't need `global`, because it changes the dict object's contents, like calling `.Add()` on a field. `next_quote_id += 1` rebinds the name to a new integer, so it does.

404s are raised rather than returned:

```python
from fastapi import HTTPException

raise HTTPException(status_code=404, detail="Quote not found")
```

## Response models: why the request DTO isn't enough

Returning the stored request object "worked", but it had problems:

- **No id in the list response.** The id was only the dict key, so a front end couldn't build a `/quotes/2` link. `policy_id` can't stand in for it, because one policy can have many quotes (renewals, re-quotes).
- **Money came back as a string:** `"premium": "0.01"`. Pydantic serialises `Decimal` as a string on purpose, because JSON numbers are read as floating point by most clients. (C#'s `System.Text.Json` writes `decimal` as a number, which a JavaScript front end reads as a `double`.)
- **Accidental exposure.** Any internal field added to the stored model later would leak to every client.
- **Domain correctness.** In a real insurer's system, the client doesn't send the premium. The server calculates it. Separate models make that possible.

So I added a separate response model and a mapper, the Python version of an AutoMapper profile:

```python
def to_response(quote_id: int, quote: QuoteRequestDto) -> QuoteResponseDto:
    return QuoteResponseDto(
        quote_id=quote_id,
        policy_id=quote.policy_id,
        premium=quote.premium,
        drivers=len(quote.drivers),
    )

@app.get("/quotes", response_model=list[QuoteResponseDto])
def get_stored_quotes(store: QuoteStore = Depends(get_quote_store)):
    return [to_response(quote_id, quote) for quote_id, quote in store.list_all()]
```

Before the mapper existed, the GET endpoints returned a **500**. FastAPI validates *outgoing* data against `response_model` too, and refused to send a shape that broke the documented contract.

`dict.items()` returns `(key, value)` pairs, and the comprehension unpacks both, like `quotes.Select(kv => ToResponse(kv.Key, kv.Value))`.

### Why can `to_response` be defined below the endpoints?

On Day 1, calling a class above its definition caused a `NameError`. Here it didn't. The rule is about **when the line runs**: code at the top level runs as the file loads, while code inside a function body runs only when the function is called. By the time a request arrives, the whole file has loaded.

The exceptions are things that run at load time: decorators (so `app = FastAPI()` must come before `@app.get`), base classes, default values, and, on Python 3.13, type hints in a signature. The practical habit is imports, then models, then helpers, then endpoints.

## Dependency injection without a container

In ASP.NET Core:

```csharp
builder.Services.AddSingleton<IQuoteStore, InMemoryQuoteStore>();
app.MapGet("/quotes/{id}", (int id, IQuoteStore store) => store.Get(id));
```

FastAPI has no container. A parameter names a **provider function**:

```python
quote_store = QuoteStore()

def get_quote_store() -> QuoteStore:
    return quote_store

@app.get("/quotes/{quote_id}", response_model=QuoteResponseDto)
def get_quote(quote_id: int, store: QuoteStore = Depends(get_quote_store)):
    quote = store.get(quote_id)
    if quote is None:
        raise HTTPException(status_code=404, detail="Quote not found")
    return to_response(quote_id, quote)
```

The lifetime depends on how you write the provider:

| Provider | ASP.NET Core |
| --- | --- |
| Returns one shared instance | `AddSingleton` |
| Creates a new object per call | `AddScoped` / `AddTransient` |
| Uses `yield` for setup and cleanup | Scoped `DbContext` with `Dispose` |

Moving state into a `QuoteStore` class also removed the `global`. `self._next_id += 1` updates an attribute of the object rather than rebinding a module-level name.

### A bug: mixing two "not found" designs

At one point I changed `store.get()` to raise `ValueError` for a missing quote, while the endpoint still checked `if quote is None`. The check became dead code, and a missing quote returned **500** instead of 404.

There are two valid designs: return `None` and let the caller decide (like `FirstOrDefault`), or raise a domain exception such as `QuoteNotFoundError` and map it to 404 in one handler. The bug came from mixing them. `ValueError` was also the wrong exception, since it means "bad value", not "not found". I kept `None` for now.

## The most important lesson: async on one thread

The syntax is familiar from C#:

```python
await asyncio.sleep(5)    # await Task.Delay(5000)
time.sleep(5)             # Thread.Sleep(5000)
```

The model underneath is not the same. C# runs async code on a **thread pool**. Python runs it on **one thread**, the event loop. I tested two endpoints by firing two requests at each, at the same moment, with `asyncio.gather` (Python's `Task.WhenAll`):

```python
@app.get("/slow-blocking/{n}")
async def slow_blocking(n: int):
    time.sleep(5)            # blocking call inside async def

@app.get("/slow-async/{n}")
async def slow_async(n: int):
    await asyncio.sleep(5)   # non-blocking wait
```

```text
slow-blocking: each [5.1, 10.0]s, total 10.1s
slow-async: each [5.0, 5.0]s, total 5.0s
```

The blocking version handled requests **one at a time**. (My first test with browser tabs gave confusing results; a script with `perf_counter` gave clean numbers.)

### The analogy that made it click

Picture a restaurant with **one waiter** (the event loop thread). The kitchen (LLM calls, databases, HTTP) takes 5 seconds per dish.

- **`await`:** the waiter passes table 1's order to the kitchen and goes straight to table 2 and table 3. All three dishes cook together, and everyone eats after about 5 seconds.
- **Blocking inside `async def`:** the waiter stands at the kitchen hatch staring at table 1's pan. Table 3 waits 15 seconds for a 5-second dish.
- **Plain `def`:** FastAPI sends that work to a back room with temporary staff (a thread pool), so the main waiter stays free.

```text
Blocking inside async def          With await
R1 [stuck 5s]                      R1 [cooking 5s]
R2 [queued  ][stuck 5s]            R2 [cooking 5s]
R3 [queued            ][stuck 5s]  R3 [cooking 5s]
   0s       5s       10s      15s     0s       5s
```

C# hides this because it has many waiters by default. In Python, one blocking call (the sync `requests` library, a sync database driver, a sync LLM client) quietly turns an AI service into a single-user system. It looks fine in a demo and falls over when 50 underwriters use it at once.

The rule: use **`async def`** only if everything slow inside it is `await`ed; otherwise use **plain `def`**.

## What I learned

- Use a mature Python version: brand-new ones may lack prebuilt wheels.
- FastAPI is ASP.NET Core minimal APIs plus model binding plus Swagger, all driven by type hints.
- Importing a Python file runs it, so guard test code with `if __name__ == "__main__":`.
- Python has no `var`, so assigning inside a function makes a name local unless declared `global`.
- Separate request and response models are about the API's contract, not just today's output.
- `Depends` gives you DI with lifetimes, without a container.
- Async in Python runs on one thread: never block inside `async def`.

## Next

Day 4: `asyncio` and `httpx` in more depth, and testing this API with `pytest`, including swapping the store with `app.dependency_overrides`.

Code for this post:

- [api.py](https://github.com/ishnetcode/fde-practice/blob/main/api.py)
- [quote_store.py](https://github.com/ishnetcode/fde-practice/blob/main/quote_store.py)
- [quote_response.py](https://github.com/ishnetcode/fde-practice/blob/main/quote_response.py)
- [timing_test.py](https://github.com/ishnetcode/fde-practice/blob/main/timing_test.py)
