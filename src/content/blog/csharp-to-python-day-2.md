---
title: "From C# Tech Lead to Forward Deployed Engineer: Day 2 - Pydantic, the validation layer Python doesn't have"
description: "Type hints don't enforce anything, so Pydantic does. Model binding, ModelState-style errors, UK dates, nested DTOs, and a one-character bug that silently corrupted my data."
pubDate: 2026-11-13
draft: false
---

On [Day 1](./csharp-to-python-day-1) I learned that Python type hints are documentation, not rules. A dataclass happily stored `"lots"` as a premium, and `premium * 2` returned `lotslots` instead of crashing.

Day 2 was about closing that gap with **Pydantic**, the library that validates data at runtime. It's also the foundation for everything LLM-related coming in Week 2: structured outputs and tool calling are built on it.

## Installing a package with uv

```bash
uv add pydantic
```

This adds `pydantic` to `pyproject.toml` and pins exact versions in `uv.lock`, the same flow as adding a NuGet package and getting a lock file entry.

## From dataclass to BaseModel

```python
from datetime import datetime
from decimal import Decimal
from pydantic import BaseModel

class Quote(BaseModel):
    policy_id: str
    premium: Decimal
    start_date: datetime
```

Two differences from the dataclass version:

- `BaseModel` is a base class you inherit from, not a decorator.
- Pydantic models **only accept named arguments**.

I found the second one by getting it wrong:

```text
TypeError: BaseModel.__init__() takes 1 positional argument but 4 were given
```

The "1" is `self`. Python passes the object itself as a hidden first argument, the explicit version of C#'s `this`. So "4 were given" means `self` plus my three values. The fix is keyword arguments, similar to a C# object initialiser:

```python
q = Quote(policy_id="P1", premium="lots", start_date="tomorrow")
```

## All errors at once, like ModelState

Two fields were wrong. Would Pydantic stop at the first, the way a C# exception does? No:

```text
2 validation errors for Quote
premium
  Input should be a valid decimal [type=decimal_parsing, input_value='lots', input_type=str]
start_date
  Input should be a valid datetime or date, input is too short [type=datetime_from_date_parsing, input_value='tomorrow', input_type=str]
```

Every error comes with the field name, a readable message, a stable `type` code you can check in code, and the exact input it received. The .NET equivalent is `ModelState` in ASP.NET Core, which collects every invalid field into one 400 response.

This matters a lot when working with LLMs. If a model returns bad JSON, you can send all the errors back in one retry instead of fixing them one round trip at a time.

## Conversion, not coercion

Next experiment: valid values, but passed as strings.

```python
q = Quote(policy_id="P1", premium="100.50", start_date="2026-11-13T09:00:00")
print(type(q.premium), type(q.start_date))
```

```text
<class 'decimal.Decimal'> <class 'datetime.datetime'>
```

Pydantic **parsed** the strings into real types. My first guess was "it's like JavaScript", but that's wrong:

- JavaScript converts types implicitly everywhere: `"5" * 2` is `10`.
- Python itself is strict: `"5" + 2` raises a `TypeError`.
- Pydantic converts deliberately, only where data enters the model, and only when the target type makes sense. `"100.50"` becomes a `Decimal`; `"lots"` is rejected.

The right .NET comparison is **model binding**: `?premium=100.50` arrives as text and gets bound to a `decimal`.

This is essential because **JSON has no date type**. Every date from an API or an LLM arrives as a string, and the format varies.

## A custom validator for UK dates

Pydantic accepts ISO 8601 out of the box, but not `13/11/2026`. That's deliberate: is `12/11/2026` 12 November or 11 December? Guessing wrong on a policy start date would be the same kind of silent bug as `lotslots`.

So I wrote a validator that accepts the UK format explicitly. My first attempt had four bugs:

```python
field_validator('start_date', mode='before')      # 1. missing @
@classmethod
def validate_start_date(cls, value):
    datetime.strptime(value, "%d/%m/%y")           # 2. %y = 2-digit year
                                                   # 3. no return
                                                   # 4. ISO strings now crash
```

1. Without the `@`, the line is just a function call whose result is thrown away. The validator never ran.
2. `%y` is a two-digit year; `%Y` is four digits. It's the same trap as `MM` vs `mm` in C# format strings.
3. With no `return`, the method returns `None`. A validator's return value becomes the field value.
4. Once it worked, `strptime` raised a `ValueError` on ISO strings.

The working version tries the UK format and falls back to Pydantic's own parsing:

```python
@field_validator("start_date", mode="before")
@classmethod
def validate_start_date(cls, value):
    try:
        return datetime.strptime(value, "%d/%m/%Y")
    except ValueError:
        return value
```

In C#, the closest equivalent is parsing the exact format and falling back to standard parsing:

```csharp
public static DateTime ParseStartDate(string value)
{
    if (DateTime.TryParseExact(value, "dd/MM/yyyy", CultureInfo.InvariantCulture,
                               DateTimeStyles.None, out var ukDate))
        return ukDate;

    return DateTime.Parse(value, CultureInfo.InvariantCulture);   // ISO 8601
}
```

C# has `TryParseExact`, so you don't need an exception for control flow. Python's `strptime` has no "try" version, so `try/except ValueError` is the normal way to write it.

`mode="before"` matters here. The default (`"after"`) runs *after* Pydantic's parsing, which would already have rejected the UK string.

## Field constraints, like data annotations

```python
premium: Decimal = Field(gt=0)
policy_id: str = Field(min_length=2, max_length=20)
```

These do the job of `[Range]` and `[StringLength]`. `gt` means greater than; `ge` means greater than or equal. I used `gt=0` so a £0 premium fails. Pydantic **parses first, then checks**: `"0.00"` becomes a `Decimal` and is then rejected by the constraint.

## Catching ValidationError

A crash with a traceback is fine while learning, but a service needs a clean response:

```python
try:
    q = Quote(...)
except ValidationError as e:
    for error in e.errors():
        field = ".".join([str(p) for p in error["loc"]])
        print(f"{field}: {error['msg']}")
else:
    print(q)
```

The ASP.NET Core equivalent is reading `ModelState`:

```csharp
if (!ModelState.IsValid)
{
    foreach (var (field, entry) in ModelState)
        foreach (var error in entry.Errors)
            Console.WriteLine($"{field}: {error.ErrorMessage}");
}
```

Three things I learned from this short block:

**`else` runs only if no exception was raised.** I first had `print(q)` after the `try/except` and got a `NameError`. The constructor had raised, so `q` was never assigned. Python has no block scope, so a variable created in `try` *is* visible afterwards, but only if the assignment actually happened.

**`loc` is a path, not a name.** It's a tuple like `('premium',)`. For nested models it becomes `('drivers', 1, 'age')`, which is `Drivers[1].Age` in C# terms.

**`join` won't convert types for you.** `".".join(('drivers', 1, 'age'))` fails with `TypeError: sequence item 1: expected str instance, int found`. C#'s `string.Join` calls `ToString()` on each item; Python doesn't. The fix was my first list comprehension:

```python
[str(p) for p in loc]          # Python
loc.Select(p => p.ToString())  // C#
```

## The final exercise: rewriting a C# DTO

I translated a motor-insurance quote request with an enum, a nested driver model, a list and an optional field. Here's the C# original:

```csharp
public enum CoverType { Comprehensive, ThirdPartyFireTheft, ThirdParty }

public class DriverDto
{
    [Required, StringLength(100, MinimumLength = 1)]
    public string FullName { get; set; }

    [Range(17, 99)]
    public int Age { get; set; }

    public int? YearsNoClaims { get; set; }
}

public class QuoteRequestDto
{
    [Required, StringLength(20, MinimumLength = 2)]
    public string PolicyId { get; set; }

    [Required]
    public CoverType Cover { get; set; }

    [Range(0.01, double.MaxValue)]
    public decimal Premium { get; set; }

    public DateTime StartDate { get; set; }   // must accept UK dates too

    [MinLength(1)]
    public List<DriverDto> Drivers { get; set; }
}
```

And the Python version:

```python
class CoverType(str, Enum):
    COMPREHENSIVE = "comprehensive"
    THIRD_PARTY_FIRE_THEFT = "third-party-fire-theft"
    THIRD_PARTY = "third-party"

class DriverDto(BaseModel):
    full_name: str = Field(min_length=1, max_length=100)
    age: int = Field(ge=17, le=99)
    years_no_claims: int | None = Field(default=None, ge=0)

class QuoteRequestDto(BaseModel):
    policy_id: str = Field(min_length=2, max_length=20)
    cover_type: CoverType
    premium: Decimal = Field(gt=0)
    start_date: datetime
    drivers: list[DriverDto] = Field(min_length=1)
    # plus the UK date validator
```

| C# | Python |
| --- | --- |
| `enum CoverType` | `class CoverType(str, Enum)` |
| `int?` | `int \| None = None` |
| `List<DriverDto>` | `list[DriverDto]` |
| `[Range(17, 99)]` | `Field(ge=17, le=99)` |

Plain dictionaries in the input became real `DriverDto` objects, the string `"comprehensive"` became an enum member, and a 16-year-old second driver produced:

```text
drivers.1.age: Input should be greater than or equal to 17
```

### The bug of the day: one comma

My first version of the optional field was:

```python
years_no_claims: int | None = None, Field(description="...")
```

It ran without errors. But in Python **a comma creates a tuple**, even without brackets, so the default value was actually `(None, FieldInfo(...))`. Pydantic doesn't validate defaults, so a driver with no claims history would silently get that tuple instead of `None`.

The fix is to put the default inside `Field`:

```python
years_no_claims: int | None = Field(default=None, ge=0)
```

It's the same lesson as Day 1: the dangerous bugs in Python are the ones that don't crash.

## Side notes: C# features Python does differently

**Culture settings.** There's no `CultureInfo`. The `locale` module is process-wide global state and depends on what the OS has installed, so it's best avoided. Use `babel` for culture-aware formatting, and convert everything to ISO 8601 and UTC as soon as it comes in.

**Access modifiers.** Python has none. `_name` signals "internal" by convention (linters warn about it), and `__name` triggers name mangling, which is rarely used. Immutability comes from `frozen=True`, not a `readonly` keyword.

**Exceptions.** `Exception` is the base, like `System.Exception`. The ones I'll see most: `ValueError` (bad value, like `FormatException`), `TypeError` (wrong type), `AttributeError` (including calling a member on `None`, Python's `NullReferenceException`), and `KeyError`. Never use a bare `except:`; it catches even Ctrl+C.

**Functions.** `def` defines every function. A method is a `def` inside a class with `self` (or `cls`) as the first parameter. A function with no `return` returns `None`.

## What I learned

- Pydantic validates at runtime, which is where API and LLM data actually shows up.
- Its errors are collected, structured and machine-readable, like `ModelState`.
- It converts deliberately, like model binding, not like JavaScript coercion.
- `mode="before"` validators let you handle messy real-world formats explicitly.
- A stray comma can quietly turn a value into a tuple.

## Next

Day 1 leftovers before FastAPI: VS Code and `ruff` setup, `pytest`, list comprehensions in more depth, and `async`/`await`, which FastAPI depends on.

Code for this post, in both languages:

- Python: [quote_request.py](https://github.com/ishnetcode/fde-practice/blob/main/quote_request.py)
- C#: [QuoteRequestDto.cs](https://github.com/ishnetcode/fde-practice/blob/main/csharp/QuoteRequestDto.cs)
