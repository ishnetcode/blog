---
title: "From C# Tech Lead to Forward Deployed Engineer: Day 1"
description: "18 years of C#/.NET, an 8-week plan to become an FDE, and the first surprise: Python type hints don't stop anything."
pubDate: 2026-11-12
draft: true
---

## Why I'm doing this

I've spent 18 years writing software, almost all of it in C# and .NET. For the last 12, I've been a Technical Lead embedded with a FTSE 100 insurer through Tata Consultancy Services, designing microservices, migrating platforms to AWS EKS, and shipping systems that handle millions of requests a day.

Lately I've been drawn to a role that has exploded across London's AI companies: the **Forward Deployed Engineer (FDE)**. An FDE sits with a customer, scopes a messy real-world problem, and ships production code inside their environment, often wiring LLMs, retrieval and agents into legacy systems.

When I looked closely, I realised I've been doing the "forward deployed" part for over a decade. What I haven't done is prove I can ship AI systems in the stack these teams use. So I wrote an honest gap analysis:

- **Strengths:** client-embedded delivery, production scale, cloud migration, insurance domain knowledge, and the habit of framing work in business outcomes.
- **Gaps:** Python fluency, a shipped LLM application (RAG, tool use, agents, evals), and a public portfolio. All my work so far lives in private client codebases.

The plan is 8 weeks: Python and LLM APIs, then RAG, agents, MCP and evals, then a flagship insurance-AI project, then interviews. I'll write up each session here, mistakes included. The code lives at [github.com/ishnetcode/fde-practice](https://github.com/ishnetcode/fde-practice).

## Day 1: Python, through C# eyes

### Setting up with uv

My first question was why everyone uses `uv` instead of just installing Python. The answer made sense once I mapped it to .NET. The `dotnet` CLI handles the SDK, packages, isolation and running code. Python historically spread those jobs across several tools. `uv` brings them back together:

| Job | .NET | uv |
| --- | --- | --- |
| Pin the language version | `global.json` | `.python-version` |
| Project definition | `.csproj` | `pyproject.toml` |
| Locked dependencies | `packages.lock.json` | `uv.lock` |
| Per-project packages | automatic | `.venv/` (automatic) |
| Run | `dotnet run` | `uv run` |

The key concept is isolation. By default Python installs packages machine-wide, and a **virtual environment** (`.venv`) is the fix. uv creates and manages it for you, which gives the per-project behaviour I'm used to from NuGet.

First, install uv. On Windows (PowerShell):

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

On macOS or Linux:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Then open a new terminal so the `uv` command is on your PATH, and check it worked:

```bash
uv --version
```

You don't need to install Python separately. The first time you run a project, uv downloads a suitable Python version for you (I got CPython 3.15.2). Next, create a project and run it:

```bash
uv init fde-practice
cd fde-practice
uv run main.py
```

My first error came straight away:

```text
error: Failed to spawn: main.py
  cause: program not found
```

My version of uv had created a package-style project (a `src/` folder) with no `main.py`. When uv can't find the file, it tries to launch `main.py` as a program instead. Creating the file fixed it.

### Translating a C# record

In C# I'd write:

```csharp
public record Quote(string PolicyId, decimal Premium, DateTime StartDate);
```

My first attempt at Python had five mistakes: the wrong module name (`dataclass` instead of `dataclasses`), a missing colon, camelCase fields, a made-up `num` type, and `date` where C#'s `DateTime` maps to `datetime`. The corrected version:

```python
from dataclasses import dataclass
from datetime import datetime
from decimal import Decimal

@dataclass
class Quote:
    policy_id: str
    premium: Decimal
    start_date: datetime
```

Conventions I'm carrying forward:

- `@dataclass` plays the role of `record`: it generates the constructor, equality and string representation.
- Indentation replaces braces (a bit like YAML, except this indentation holds code that runs).
- snake_case for fields and functions, PascalCase for classes.
- Even standard types like `Decimal` and `datetime` need an import.

### Order matters

My first run failed with:

```text
NameError: name 'Quote' is not defined
```

I'd put the test code above the class. C# compiles the whole file before running, so declaration order doesn't matter. Python executes the file **top to bottom at runtime**, so when it reached my test line, `Quote` didn't exist yet.

### The surprise: type hints don't enforce anything

Then came the real experiment. I passed obviously wrong data:

```python
q = Quote("P1", "lots", "tomorrow")
print(q)
```

I predicted a compile error. Instead, Python printed:

```text
Quote(policy_id='P1', premium='lots', start_date='tomorrow')
```

No error at all. **Type hints in Python are documentation, not rules.** Nothing checks them when the code runs.

It gets worse:

```python
print(q.premium * 2)
```

```text
lotslots
```

Multiplying a string by 2 repeats it. A premium was "doubled" into nonsense, and nothing complained.

This was the most important lesson of the day: **a crash is not the worst outcome; a plausible-looking wrong answer is.** Picture `"lots"` arriving from a customer CSV or an LLM's output, flowing through a pricing calculation, and ending up in a quote. In C# the compiler protects you. In Python, you validate data at the boundary where it comes in.

There are two ways to fill that gap:

- **Static checkers** (`mypy`, Pylance) read the hints before the code runs, which is the closest thing to the C# compiler.
- **Pydantic** validates data at runtime when an object is created. For FDE work this matters most, because API responses, customer files and LLM outputs only exist at runtime.

### Shipping it to GitHub

Pushing the repo produced two more small lessons:

- uv had initialised git with a `master` branch. I renamed it with `git branch -M main` to match GitHub's default.
- `git push` returned "Repository not found" because I hadn't created the empty repo on GitHub first. Git pushes to a repo; it doesn't create one. (`gh repo create --source=. --push` does both in one step.)

## What I learned

- uv is the Python equivalent of the `dotnet` CLI plus NuGet, and `.venv` is how Python isolates projects.
- Python runs top to bottom at runtime, so declaration order matters.
- Type hints aren't enforced, and bad data can get through silently.
- Wrong types don't always crash. Sometimes they quietly produce a wrong result.

## Next

Pydantic. I'll swap `@dataclass` for a Pydantic `BaseModel`, pass `"lots"` again, and watch it get rejected. It's the foundation for the structured outputs and tool calling coming in Week 2.
