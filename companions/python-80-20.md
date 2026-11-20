# Python 80/20: Master the 20% You Use Every Day (Step by Step)

> [!NOTE]
> Most day-to-day Python is a small, repeating set of skills: strings, lists and dicts, comprehensions, functions, exceptions, files, dates, regex, dataclasses, modules and a handful of tools. This tutorial teaches exactly that set, then makes you fast at it with drills that have tests. You apply each skill in one real project: a command-line tool that cleans a messy bank CSV and summarizes it. Every file below was built and run: lint clean, 49 tests passing (25 drill tests, 24 project tests), and every REPL session shown was checked with Python's doctest.

## The 80/20 of Python

The goal here is not to know all of Python. It is to be **expert in the roughly 20% that you will use in 80% of your real work**, so that you type it from memory, spot mistakes on sight and read errors without panic. The 80/20 split is a rule of thumb, not an exact law.

### The 20%: the skills you use every day

| # | Skill | What you do with it | Chapter |
| --- | --- | --- | --- |
| 1 | Strings and f-strings | Clean, split, join, format text and numbers | 1 |
| 2 | Lists, dicts, sets, tuples | Hold and look up data | 2 |
| 3 | Comprehensions, `sorted(key=)`, `enumerate`, `zip`, `any`, `all` | Transform, filter, rank and check data | 2 |
| 4 | Functions | Defaults, keyword arguments, multiple returns, type hints | 3 |
| 5 | Exceptions | `try`/`except`, `raise`, your own error types, reading tracebacks | 4 |
| 6 | Dates and regex | `datetime`, `timedelta`, `re.search`/`findall`/`sub` | 5 |
| 7 | Dataclasses and enums | Model your data; know when not to write a class | 6 |
| 8 | Files, paths, JSON, CSV | `pathlib`, `with open`, `json`, `csv` | 7 |
| 9 | Modules and scripts | Imports, packages, `__main__`, `argparse`, `logging` | 8 |
| 10 | Tools | `uv`, the REPL, `pytest`, `ruff`, `breakpoint()` | 9 |

### The other 80%: look it up when you need it

You can be productive for years without mastering these. Learn each one when a real task demands it:

- `async`/`await`, threads and multiprocessing
- Writing your own decorators, generators and context managers (*using* them is already in the 20%)
- Metaclasses, descriptors, `__slots__`, multiple inheritance
- Advanced typing (`Protocol`, `TypeVar`, generics)
- Packaging and publishing to PyPI
- Whole frameworks and libraries: pandas, Django, FastAPI. They are the next layer on top of the 20%, not part of it.

### How to get fast: the practice rules

- **The 30-second rule.** If you cannot type a pattern from memory in 30 seconds, it belongs in your daily 20%. Drill it until you can.
- **Type it, never paste it.** Typing is how patterns stick.
- **Three passes per drill.** Day 1: write it with the chapter open. Day 3: write it from a blank file with no notes. Day 7: write it again, timed.
- **Read errors bottom-up.** The last line of a traceback names the problem. The lines above show how you got there (Chapter 4 and 9).
- **Ten minutes a day beats three hours on Sunday.** The drills are small on purpose.

## What you will build

The project is `ledger`, a command-line tool that takes a messy bank export and produces a clean summary. It is a typical "real world" script: inconsistent dates and amounts, junk rows, duplicates, files with encoding quirks. Each chapter adds one module that uses that chapter's skills.

| Step | You do |
| --- | --- |
| 0 | Install the tools |
| 1 | Create the project and the sample data |
| 2 to 9 | Chapters 1 to 8: one module and one drill each |
| 10 | Chapter 9: tools (pytest, ruff, REPL, debugger) |
| 11 | Run the finished tool and the full test suite |
| 12 | What to learn next, and the checklist |
| Appendix | Drill solutions |

## How to read this tutorial

- **Every path is relative to the ****`ledger/`**** folder (the project root)** you create in Step 1. Every command runs from there unless it says otherwise. Each file is introduced with its full path starting at `ledger/`.
- Each chapter has four parts: **Daily patterns** (REPL sessions to type along with), **Gotchas** (the mistakes everyone makes), **Build** (the module for the project) and **Drill** (functions you write from a spec; tests tell you when you are right).
- Drill solutions are in the Appendix. Try first. Peeking is fine after a real attempt.
- Versions used: Python 3.13, pytest 9.2, ruff 0.17. Everything uses only the standard library at runtime.

## Step 0: Install everything

You need **uv** (installs Python and manages projects) and **Python 3.13**. Do each sub-step in order and run its check.

### 0.1 Open a terminal

- macOS: Terminal (or iTerm)
- Windows: PowerShell
- Linux: any shell

### 0.2 Install uv

macOS or Linux (pick one):

```bash
brew install uv
# or, without Homebrew:
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Windows (PowerShell):

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

Close and reopen the terminal, then check:

```bash
uv --version
```

Expected: a version number such as `uv 0.13.7`. If you see `command not found`, reopen the terminal or add `~/.local/bin` to your `PATH`.

### 0.3 Install Python 3.13

```bash
uv python install 3.13
uv python list --only-installed
```

Expected: a `cpython-3.13.x` line. Any 3.12 or newer works.

### 0.4 Installation checklist

- [ ] `uv --version` prints a version
- [ ] `uv python list --only-installed` shows Python 3.13 (or 3.12+)

## Step 1: Create the project and the sample data

```bash
mkdir ledger && cd ledger
uv init --package --name ledger --python 3.13
```

`uv init --package` creates a `src/` layout project and also starts a git repository. You will replace its generated files. First replace the whole contents of `ledger/pyproject.toml` (the file that plays the role of `package.json` or `requirements.txt`):

```toml
[project]
name = "ledger"
version = "0.1.0"
description = "Clean a messy bank CSV and summarize it: the everyday Python toolkit"
requires-python = ">=3.12"
dependencies = []

[project.scripts]
ledger = "ledger.cli:main"

[build-system]
requires = ["uv_build>=0.13.7,<0.14.0"]
build-backend = "uv_build"

[dependency-groups]
dev = ["pytest>=9.0", "ruff>=0.17"]

[tool.pytest.ini_options]
testpaths = ["tests"]
pythonpath = ["."]

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B"]
ignore = ["E501"]
```

If `uv init` generated a different `uv_build` version range, keep the range from your generated file. Now create the remaining folders and files:

```bash
rm -f README.md
mkdir -p data drills tests
touch drills/__init__.py tests/__init__.py
printf '.venv/\n__pycache__/\n.pytest_cache/\n.ruff_cache/\nout/\ndist/\n' > .gitignore
```

Replace the whole contents of `ledger/src/ledger/__init__.py` (the generated one holds a hello-world function) with one line:

```py
"""ledger: clean a messy bank CSV and summarize it."""
```

Install the project and its dev tools. This creates `.venv/` and `uv.lock`:

```bash
uv sync
```

Expected: a package list including `pytest` and `ruff`, and `ledger` installed in editable mode. Check:

```bash
uv run python -c "import ledger; print('ok')"
```

Expected: `ok`. From here, every command uses `uv run`, so you never activate the virtual environment by hand.

Create the sample data. This is the messy bank export you will clean. It has mixed date formats, dollar signs, parentheses for negatives, odd capitalization, a duplicate row, and four rows that are broken on purpose. Create `ledger/data/transactions.csv`:

```javascript
date,description,amount,category
2026-01-03,POS PURCHASE 01/02 STARBUCKS #1234,-$4.75,Food & Drink
01/04/2026,  Whole Foods   Market #55 ,"-$82.10",Groceries
2026-01-05,PAYROLL ACME INC,"$3,200.00",Income
05 Jan 2026,DEBIT CARD 01/05 SHELL OIL #771,(45.20),Transport
2026-01-06,NETFLIX,-15.49,  entertainment 
2026-01-06,NETFLIX,-15.49,  entertainment 
2026-01-07,CHECKCARD 01/07 STARBUCKS #9,(5.25),food & drink
2026-01-09,Whole Foods Market #55,-61.80,GROCERIES
2026-01-10,Rent January,"-1,500.00",Housing
2026-01-12,Electric Co,-92.30,Utilities
2026-01-15,PAYROLL ACME INC,"$3,200.00",Income
not-a-date,Mystery charge,-$9.99,Other
2026-01-16,Bad amount,abc,Other
2026-01-17,Empty amount,,Other
2026-01-18,Infinity,inf,Other
2026-01-20,Gym,-30,
2026-02-01,POS PURCHASE 01/31 STARBUCKS #1234,-$4.75,Food & Drink
02/02/2026,Whole Foods Market #55,-70.00,Groceries
02 Feb 2026,Rent February,"-1,500.00",Housing
2026-02-05,PAYROLL ACME INC,"$3,200.00",Income
2026-02-14,Restaurant Le Petit,-64.20,Food & Drink
```

The two NETFLIX rows end with a trailing space after `entertainment`; keep it, the cleaning code will handle it. When we are done, the project looks like this:

```txt
ledger/
├── pyproject.toml
├── uv.lock
├── .gitignore
├── data/
│   └── transactions.csv
├── src/ledger/                 # the project (one module per chapter)
│   ├── __init__.py
│   ├── text.py                 # ch 1: strings and numbers
│   ├── collect.py              # ch 2: lists, dicts, sets
│   ├── report.py               # ch 3: functions
│   ├── errors.py               # ch 4: exceptions
│   ├── parse.py                # ch 5: dates and regex
│   ├── models.py               # ch 6: dataclasses and enums
│   ├── io.py                   # ch 7: files, CSV, JSON
│   ├── analysis.py             # ch 8: putting it together
│   └── cli.py                  # ch 8: argparse and logging
├── drills/                     # you write these (one file per chapter)
│   ├── __init__.py
│   ├── ch01_strings.py         # ... through ch08_cli.py
│   └── ...
└── tests/
    ├── __init__.py
    ├── test_ledger.py          # tests for the project
    └── test_drill_ch01.py      # ... through test_drill_ch08.py
```

## Step 2: Chapter 1, strings, numbers and truthiness

**You use this when:** you clean input, build messages, format output, parse numbers.

### Daily patterns

Open a REPL with `uv run python` and type these. Lines starting with `>>>` are what you type; the line below is Python's answer. Exit with `exit()`.

```txt
>>> name, n = "ada", 3
>>> f"{name.title()} has {n} items ({n / 4:.0%})"
'Ada has 3 items (75%)'
>>> f"{1234567.891:,.2f} | {5:03d} | {'x':>4} | {'x':<4} | {'x':^4}"
'1,234,567.89 | 005 |    x | x    |  x  '
>>> s = "  Hello, World  "
>>> s.strip().lower().replace(",", "").split()
['hello', 'world']
>>> ", ".join(["a", "b", "c"])
'a, b, c'
>>> "World" in s, s.strip().startswith("Hello"), s.find("zzz")
(True, True, -1)
>>> s.strip()[:5], s.strip()[-5:], s.strip()[::-1]
('Hello', 'World', 'dlroW ,olleH')
>>> bool(""), bool("0"), bool([]), bool(0), bool(None), bool("False")
(False, True, False, False, False, True)
>>> 7 / 2, 7 // 2, 7 % 2, 2 ** 10
(3.5, 3, 1, 1024)
>>> 0.1 + 0.2 == 0.3, round(0.1 + 0.2, 2) == 0.3
(False, True)
```

The patterns to own: f-strings with format specs (`:,.2f` thousands separators and two decimals, `:03d` zero padding, `:>4` right align), `strip` / `split` / `join` / `replace` / `startswith` / `in`, slicing (`[:5]`, `[-5:]`, `[::-1]`), and the fact that `/` always gives a float while `//` is whole-number division.

### Gotchas

- **Strings are immutable.** `s.strip()` returns a new string. Writing `s.strip()` on its own line does nothing.
- **Empty things are falsy; non-empty things are truthy, including ****`"0"`**** and ****`"False"`****.** `if not text:` is the idiom for "empty". Use `is None` when `0` or `""` are valid values.
- **Floats are inexact.** `0.1 + 0.2 != 0.3`. Round for display, and use `round()` or the `decimal` module for money.
- **`float()`**** accepts ****`"nan"`**** and ****`"inf"`****.** User input like that passes the conversion and then poisons your sums. Check with `math.isfinite`.
- **Use ****`"".join(parts)`****, not ****`+=`**** in a loop,** to build big strings.

### Build

Create `ledger/src/ledger/text.py`:

```py
"""Chapter 1: strings, numbers and truthiness."""
import math


def clean_label(raw: str) -> str:
    """Collapse whitespace and lower-case: '  Food &   Drink ' -> 'food & drink'."""
    return " ".join(raw.split()).lower()


def parse_amount(raw: str) -> float:
    """'$1,234.50' -> 1234.5, '(12.00)' -> -12.0, '-$5' -> -5.0. Raises ValueError on junk."""
    text = raw.strip()
    if not text:  # "" is falsy, so this catches empty and whitespace-only input
        raise ValueError("empty amount")
    negative = text.startswith("(") and text.endswith(")")
    if negative:
        text = text[1:-1]
    text = text.replace("$", "").replace(",", "").strip()
    value = float(text)  # raises ValueError for things like "abc"
    if not math.isfinite(value):  # float() happily accepts "nan" and "inf"
        raise ValueError(f"not a real amount: {raw!r}")
    return -value if negative else value


def money(amount: float) -> str:
    """-1234.5 -> '-$1,234.50'."""
    sign = "-" if amount < 0 else ""
    return f"{sign}${abs(amount):,.2f}"
```

Notice `" ".join(raw.split())`: `split()` with no argument splits on any run of whitespace and drops empties, so joining with one space collapses all the mess in a single expression. In the f-string, `!r` shows the value with quotes (`repr`), which makes stray whitespace visible in error messages.

### Drill

Create `ledger/drills/ch01_strings.py` with three functions that satisfy this spec, then create the test file and run it.

| Function | Behavior |
| --- | --- |
| `slugify(text)` | `"  Hello, World! 2026 "` becomes `"hello-world-2026"`. Anything that is not a letter or digit separates words. |
| `mask_email(email)` | `"jane.doe@example.com"` becomes `"j*******@example.com"`. `ValueError` unless there is exactly one `@` with text on both sides. |
| `initials(name)` | `"ada  lovelace byron"` becomes `"ALB"`. |

Create `ledger/tests/test_drill_ch01.py`:

```py
import pytest

from drills.ch01_strings import initials, mask_email, slugify


def test_slugify():
    assert slugify("  Hello, World! 2026 ") == "hello-world-2026"
    assert slugify("Already-slugged") == "already-slugged"
    assert slugify("!!!") == ""


def test_mask_email():
    assert mask_email("jane.doe@example.com") == "j*******@example.com"
    for bad in ["nope", "@x.com", "a@", "a@b@c"]:
        with pytest.raises(ValueError):
            mask_email(bad)


def test_initials():
    assert initials("ada  lovelace byron") == "ALB"
    assert initials("") == ""
```

```bash
uv run pytest tests/test_drill_ch01.py -q
```

Before you write the drill file it fails with `ModuleNotFoundError` (that is the point). When you are done: `3 passed`.

## Step 3: Chapter 2, lists, dicts, sets and comprehensions

**You use this when:** almost always. Most programs are "collect data, reshape it, look things up".

### Daily patterns

```txt
>>> nums = [5, 3, 8, 1]
>>> [n * 2 for n in nums if n > 2]
[10, 6, 16]
>>> {n: n * n for n in nums}
{5: 25, 3: 9, 8: 64, 1: 1}
>>> sorted(nums), sorted(nums, reverse=True), nums
([1, 3, 5, 8], [8, 5, 3, 1], [5, 3, 8, 1])
>>> people = [{"name": "Ada", "age": 36}, {"name": "Linus", "age": 28}]
>>> sorted(people, key=lambda p: p["age"])[0]["name"]
'Linus'
>>> max(people, key=lambda p: p["age"])["name"], sum(p["age"] for p in people)
('Ada', 64)
>>> for i, p in enumerate(people, start=1):
...     print(i, p["name"])
1 Ada
2 Linus
>>> list(zip(["a", "b"], [1, 2])), dict(zip(["a", "b"], [1, 2]))
([('a', 1), ('b', 2)], {'a': 1, 'b': 2})
>>> d = {"x": 1}
>>> d.get("y"), d.get("y", 0), d.setdefault("z", []), d
(None, 0, [], {'x': 1, 'z': []})
>>> first, *rest = [1, 2, 3]
>>> first, rest
(1, [2, 3])
>>> any(n > 7 for n in nums), all(n > 0 for n in nums)
(True, True)
>>> a = [1, 2]
>>> b = a          # two names, ONE list
>>> b.append(3)
>>> a
[1, 2, 3]
>>> c = a.copy()   # a real copy
>>> c.append(4)
>>> a
[1, 2, 3]
>>> {1, 2, 3} & {2, 3, 4}, {1, 2} | {2, 3}
({2, 3}, {1, 2, 3})
>>> list(dict.fromkeys([3, 1, 3, 2]))   # dedupe, keep order
[3, 1, 2]
```

The patterns to own: the list comprehension (`[expr for x in xs if cond]`), the dict comprehension, `sorted(..., key=...)` and `max(..., key=...)`, `enumerate(..., start=1)`, `zip`, `d.get(key, default)`, unpacking (`first, *rest = ...`), `any` / `all`, and set operators (`&` intersection, `|` union). Read a comprehension as "the expression, for each item, if the condition".

### Gotchas

- **Assignment never copies.** `b = a` makes two names for one list. Use `a.copy()` or `list(a)` for a shallow copy.
- **`sorted()`**** returns a new list; ****`list.sort()`**** sorts in place and returns ****`None`****.** `items = items.sort()` sets `items` to `None`.
- **`d[key]`**** raises ****`KeyError`**** for a missing key; ****`d.get(key)`**** returns ****`None`****.**
- **Do not change a list or dict while looping over it.** Build a new one with a comprehension instead.
- **Dict keys must be hashable**: strings, numbers and tuples work; lists do not. Sets and dicts give you fast `in` checks; lists scan every element.
- **`all([])`**** is ****`True`**** and ****`any([])`**** is ****`False`****.** Guard the empty case when it matters.

### Build

Create `ledger/src/ledger/collect.py`:

```py
"""Chapter 2: lists, dicts, sets, comprehensions and sorting."""
from collections import Counter, defaultdict


def group_totals(items, key, value) -> dict:
    """Sum value(item) for each key(item). key and value are functions."""
    totals = defaultdict(float)
    for item in items:
        totals[key(item)] += value(item)
    return dict(totals)


def top_n(totals: dict, n: int = 5, *, largest: bool = True) -> list[tuple]:
    """The n biggest (or smallest) entries as (key, total), ties broken by key."""
    sign = -1 if largest else 1
    return sorted(totals.items(), key=lambda kv: (sign * kv[1], kv[0]))[:n]


def count_by(items, key) -> Counter:
    return Counter(key(item) for item in items)


def dedupe(items, key=None) -> list:
    """Drop repeats, keeping the first of each and the original order."""
    seen = set()
    kept = []
    for item in items:
        marker = item if key is None else key(item)
        if marker in seen:
            continue
        seen.add(marker)
        kept.append(item)
    return kept
```

This is the most reusable file in the project. `group_totals` takes *functions* as arguments (`key` and `value`), so one helper groups by anything. `defaultdict(float)` makes missing keys start at `0.0`, so no "if key in dict" boilerplate. `top_n` sorts by a tuple `(sign * total, key)`: the first element ranks, the second breaks ties so output is deterministic. `dedupe` uses a `set` for the "have I seen this?" check, which is fast.

### Drill

Create `ledger/drills/ch02_collections.py`:

| Function | Behavior |
| --- | --- |
| `most_common_words(text, n=3)` | Lower-case, ignore punctuation (keep apostrophes), return the `n` most frequent `(word, count)` pairs, ties alphabetical. |
| `invert_dict(d)` | `{"a": 1, "b": 1, "c": 2}` becomes `{1: ["a", "b"], 2: ["c"]}`. |
| `merge_dicts(*dicts)` | Merge any number of dicts, later ones win, never modify the inputs. |

Create `ledger/tests/test_drill_ch02.py`:

```py
from drills.ch02_collections import invert_dict, merge_dicts, most_common_words


def test_most_common_words():
    text = "The cat and the hat. The END, and the cat's hat!"
    assert most_common_words(text, 2) == [("the", 4), ("and", 2)]
    assert most_common_words("b a b a c", 3) == [("a", 2), ("b", 2), ("c", 1)]  # ties alphabetical


def test_invert_dict():
    assert invert_dict({"a": 1, "b": 1, "c": 2}) == {1: ["a", "b"], 2: ["c"]}


def test_merge_dicts_later_wins_and_inputs_untouched():
    a, b = {"x": 1, "y": 2}, {"y": 3}
    assert merge_dicts(a, b, {"z": 4}) == {"x": 1, "y": 3, "z": 4}
    assert a == {"x": 1, "y": 2} and b == {"y": 3}
    assert merge_dicts() == {}
```

```bash
uv run pytest tests/test_drill_ch02.py -q
```

Expected when done: `3 passed`.

## Step 4: Chapter 3, functions

**You use this when:** constantly. Good Python is small functions that do one thing and return a value.

### Daily patterns

```txt
>>> def greet(name, greeting="Hello", *, punctuation="!"):
...     return f"{greeting}, {name}{punctuation}"
>>> greet("Ada"), greet("Ada", punctuation="?"), greet(greeting="Hi", name="Ada")
('Hello, Ada!', 'Hello, Ada?', 'Hi, Ada!')
>>> def stats(nums):
...     return min(nums), max(nums), sum(nums) / len(nums)
>>> low, high, mean = stats([2, 4, 9])
>>> (low, high, mean)
(2, 9, 5.0)
>>> def log(*args, **kwargs):
...     return args, kwargs
>>> log(1, 2, a=3)
((1, 2), {'a': 3})
>>> opts = {"punctuation": "."}
>>> greet("Ada", **opts)
'Hello, Ada.'
>>> def add_item(item, bucket=None):   # never write bucket=[]
...     bucket = [] if bucket is None else bucket
...     bucket.append(item)
...     return bucket
>>> add_item(1), add_item(2)
([1], [2])
>>> def total(prices: list[float], *, tax: float = 0.0) -> float:
...     """Sum prices, then add tax (0.1 = 10%)."""
...     return round(sum(prices) * (1 + tax), 2)
>>> total([10, 5.5], tax=0.1)
17.05
>>> total.__doc__
'Sum prices, then add tax (0.1 = 10%).'
>>> sorted(["bb", "a", "ccc"], key=len, reverse=True)
['ccc', 'bb', 'a']
```

The patterns to own: default values, **keyword-only parameters** (everything after a bare `*` must be passed by name, which makes call sites readable), returning several values as a tuple and unpacking them, `*args` / `**kwargs` and unpacking a dict into keyword arguments with `**opts`, the `None` default instead of a mutable default, type hints plus a one-line docstring, and passing a function (`key=len`) as an argument.

### Gotchas

- **Never use a mutable default (****`def f(x, bucket=[])`****).** The list is created once, when the function is defined, and shared by every call. Use `None` and create the list inside.
- **A function with no ****`return`**** returns ****`None`****.** Forgetting `return` is a classic bug, and `None` flows downstream.
- **Type hints are not enforced at runtime.** They document intent and let your editor and tools catch mistakes.
- **Parameters are references.** A function that mutates a list you pass in changes the caller's list. Prefer returning new values.
- **Prefer small functions that take data and return data.** They are the easiest thing to test.

### Build

Create `ledger/src/ledger/report.py`:

```py
"""Chapter 3: functions with defaults, keyword-only arguments and type hints."""
from .text import money


def format_table(rows: list[tuple[str, float]], *, title: str, width: int = 28) -> str:
    """Render (label, amount) rows as aligned text. `*` makes title and width keyword-only."""
    lines = [title, "-" * (width + 14)]
    lines += [f"{label:<{width}}{money(amount):>14}" for label, amount in rows]
    return "\n".join(lines)


def format_summary(summary: dict, *, top: int = 5) -> str:
    """Turn the dict from analysis.summarize() into a printable report."""
    head = [
        f"Transactions: {summary['transactions']}",
        f"Income:   {money(summary['income']):>12}",
        f"Expenses: {money(summary['expenses']):>12}",
        f"Net:      {money(summary['net']):>12}",
    ]
    categories = sorted(summary["by_category"].items(), key=lambda kv: -kv[1])[:top]
    months = sorted(summary["by_month"].items())
    return "\n\n".join([
        "\n".join(head),
        format_table(categories, title="Spending by category"),
        format_table(months, title="Net by month"),
        format_table(summary["top_merchants"][:top], title="Top merchants"),
    ])
```

The interesting bit is `f"{label:<{width}}"`: format specs can contain their own `{}` fields, so the column width is a variable. `from .text import money` is a **relative import**: `.` means "this package". `format_summary` is a pure function from a dict to a string, which is why it is trivially testable.

### Drill

Create `ledger/drills/ch03_functions.py`:

| Function | Behavior |
| --- | --- |
| `chunked(seq, size)` | `chunked([1, 2, 3, 4, 5], 2)` returns `[[1, 2], [3, 4], [5]]`. `ValueError` if `size <= 0`. |
| `first(iterable, predicate=None, default=None)` | First item (that satisfies `predicate`, if given), else `default`. Must work on generators. |
| `clamp(value, *, low=0, high=1)` | Keep `value` inside `[low, high]`. `low` and `high` are keyword-only. `ValueError` if `low > high`. |

Create `ledger/tests/test_drill_ch03.py`:

```py
import pytest

from drills.ch03_functions import chunked, clamp, first


def test_chunked():
    assert chunked([1, 2, 3, 4, 5], 2) == [[1, 2], [3, 4], [5]]
    assert chunked("abcd", 3) == [["a", "b", "c"], ["d"]]
    assert chunked([], 3) == []
    with pytest.raises(ValueError):
        chunked([1], 0)


def test_first_works_on_any_iterable_including_generators():
    assert first([5, 8, 12], lambda n: n > 6) == 8
    assert first([5, 8, 12]) == 5
    assert first([], default="none") == "none"
    assert first((n * n for n in range(10)), lambda n: n > 20) == 25


def test_clamp_is_keyword_only():
    assert clamp(5, low=0, high=3) == 3 and clamp(-2) == 0 and clamp(0.5) == 0.5
    with pytest.raises(TypeError):
        clamp(5, 0, 3)
    with pytest.raises(ValueError):
        clamp(1, low=5, high=2)
```

```bash
uv run pytest tests/test_drill_ch03.py -q
```

Expected when done: `3 passed`.

## Step 5: Chapter 4, exceptions

**You use this when:** input can be wrong, files can be missing, networks fail. In real work that is always.

### Daily patterns

```txt
>>> def to_int(text):
...     try:
...         return int(text)
...     except ValueError:
...         return None
>>> to_int("12"), to_int("twelve")
(12, None)
>>> def parse(text):
...     try:
...         value = int(text)
...     except ValueError:
...         return "bad"
...     else:                    # runs only if the try block raised nothing
...         return f"ok {value}"
...     finally:
...         pass                 # always runs: close files, release locks
>>> parse("5"), parse("x")
('ok 5', 'bad')
>>> try:
...     {}["missing"]
... except KeyError as err:
...     print("KeyError:", err)
KeyError: 'missing'
>>> try:
...     int("x")
... except (TypeError, ValueError) as err:
...     print(type(err).__name__, "-", err)
ValueError - invalid literal for int() with base 10: 'x'
>>> def need_positive(n):
...     if n <= 0:
...         raise ValueError(f"expected a positive number, got {n}")
...     return n
>>> try:
...     need_positive(-1)
... except ValueError as err:
...     print(err)
expected a positive number, got -1
```

The patterns to own: `try` / `except SomeError as err` / `else` / `finally`, catching a tuple of types, and `raise SomeError("clear message")`. Python favors **"easier to ask forgiveness than permission"**: try the operation and handle the failure, instead of checking every condition first.

### Reading a traceback

Save this as a scratch file `bug.py` (anywhere) and run `uv run python bug.py`:

```py
def average(nums):
    return sum(nums) / len(nums)


def report(groups):
    return {name: average(nums) for name, nums in groups.items()}


report({"a": [1, 2], "b": []})
```

```txt
Traceback (most recent call last):
  File "bug.py", line 9, in <module>
    report({"a": [1, 2], "b": []})
    ~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^
  File "bug.py", line 6, in report
    return {name: average(nums) for name, nums in groups.items()}
                  ~~~~~~~^^^^^^
  File "bug.py", line 2, in average
    return sum(nums) / len(nums)
           ~~~~~~~~~~^~~~~~~~~~~
ZeroDivisionError: division by zero
```

**Read it from the bottom.** The last line is the error type and message: division by zero. The frame just above it is where it happened (`average`, line 2). Each frame above that is who called whom, ending at your script's top level. The `^^^^` marks show the exact sub-expression that failed. The empty list for `"b"` is the cause. The fix belongs in `average`, not in the code that called it.

### Gotchas

- **Never write a bare ****`except:`****.** It also catches Ctrl+C and real bugs. Catch the specific error you expect.
- **Do not swallow errors silently.** `except ValueError: pass` hides problems. Return a value, log it, or re-raise.
- **Keep the ****`try`**** block small,** just the line that can fail, so you do not catch unrelated errors.
- **Use ****`raise ... from None`** to hide a confusing original error, or `raise ... from err` to chain it.
- **A custom exception class is one line** (`class LedgerError(Exception): pass`) and lets callers catch your errors separately from Python's.

### Build

Create `ledger/src/ledger/errors.py`:

```py
"""Chapter 4: your own exceptions."""


class LedgerError(Exception):
    """Base class for expected, user-facing problems (bad file, bad row)."""


class RowError(LedgerError):
    """One CSV row could not be understood."""

    def __init__(self, line: int, message: str):
        super().__init__(f"line {line}: {message}")
        self.line = line
        self.message = message
```

This is also your first **class**: `__init__` is the constructor, `self` is the instance, and `super().__init__(...)` runs the parent's constructor. `RowError` carries structured data (`line`, `message`) plus a readable message. Because it inherits from `LedgerError`, `except LedgerError` catches both kinds.

### Drill

Create `ledger/drills/ch04_errors.py`:

| Function | Behavior |
| --- | --- |
| `safe_int(text, default=None)` | `int(text)`, or `default` when it cannot be converted (bad text, or `None`). |
| `read_json(path)` | Load a JSON file. Raise `LedgerError` with a helpful message if the file is missing (`"file not found"`) or is not valid JSON (`"not valid JSON (line N)"`). Import `LedgerError` from `ledger.errors`. |
| `retry(fn, attempts=3, exceptions=(Exception,))` | Call `fn()` until it succeeds. After the last failed attempt, re-raise the error. Only retry the listed exception types. |

Create `ledger/tests/test_drill_ch04.py`:

```py
import pytest

from drills.ch04_errors import read_json, retry, safe_int
from ledger.errors import LedgerError


def test_safe_int():
    assert safe_int("42") == 42 and safe_int(" 7 ") == 7
    assert safe_int("x") is None and safe_int(None, default=0) == 0 and safe_int("1.5", default=-1) == -1


def test_read_json_gives_friendly_errors(tmp_path):
    good = tmp_path / "good.json"
    good.write_text('{"a": 1}')
    assert read_json(good) == {"a": 1}
    with pytest.raises(LedgerError, match="not found"):
        read_json(tmp_path / "missing.json")
    bad = tmp_path / "bad.json"
    bad.write_text("{oops")
    with pytest.raises(LedgerError, match="not valid JSON"):
        read_json(bad)


def test_retry_succeeds_after_failures_and_reraises_the_last_error():
    calls = []

    def flaky():
        calls.append(1)
        if len(calls) < 3:
            raise ConnectionError("down")
        return "ok"

    assert retry(flaky, attempts=3) == "ok" and len(calls) == 3
    always = []

    def broken():
        always.append(1)
        raise ConnectionError("still down")

    with pytest.raises(ConnectionError):
        retry(broken, attempts=2)
    assert len(always) == 2


def test_retry_only_catches_the_listed_exceptions():
    def wrong_kind():
        raise KeyError("x")

    with pytest.raises(KeyError):
        retry(wrong_kind, attempts=3, exceptions=(ConnectionError,))
```

```bash
uv run pytest tests/test_drill_ch04.py -q
```

Expected when done: `4 passed`. Note `pytest.raises(...)` is how you test that code fails correctly, and `tmp_path` is a pytest feature that gives each test a fresh temporary folder.

## Step 6: Chapter 5, dates and regex

**You use this when:** you parse timestamps, compute deadlines, or pull structured bits out of text.

### Daily patterns

```txt
>>> from datetime import date, datetime, timedelta, timezone
>>> d = date(2026, 3, 14)
>>> d + timedelta(days=30), (date(2026, 12, 25) - d).days, d.weekday(), d.isoformat()
(datetime.date(2026, 4, 13), 286, 5, '2026-03-14')
>>> datetime.strptime("14 Mar 2026 09:30", "%d %b %Y %H:%M")
datetime.datetime(2026, 3, 14, 9, 30)
>>> d.strftime("%A, %B %d, %Y"), f"{d:%b %d}"
('Saturday, March 14, 2026', 'Mar 14')
>>> datetime(2026, 3, 14, 9, 30, tzinfo=timezone.utc).isoformat()
'2026-03-14T09:30:00+00:00'
>>> import re
>>> re.search(r"(\d+) items", "found 42 items").group(1)
'42'
>>> re.findall(r"\b[A-Z]{2,}\b", "NASA and ESA met the BBC")
['NASA', 'ESA', 'BBC']
>>> re.sub(r"\s+", " ", "a   b \n c")
'a b c'
>>> m = re.match(r"(?P<y>\d{4})-(?P<m>\d{2})", "2026-03-14")
>>> m["y"], m.groupdict()
('2026', {'y': '2026', 'm': '03'})
>>> bool(re.fullmatch(r"\d{3}-\d{4}", "555-1234")), re.match(r"\d+", "abc")
(True, None)
```

The patterns to own: subtracting dates gives a `timedelta` (use `.days`), `strptime` to parse and `strftime` or an f-string to format (`%Y-%m-%d`, `%b`, `%A`), `isoformat()` for storage, and the four regex workhorses `search`, `findall`, `sub` and `fullmatch`, plus named groups `(?P<name>...)`. Always write regex as a raw string, `r"..."`.

### Gotchas

- **Dates and datetimes are different types.** `date` has no time; `datetime` does. Subtracting a `date` from a `datetime` raises `TypeError`.
- **Naive versus aware datetimes.** A datetime with no `tzinfo` is "naive" and has no time zone. Store and exchange timestamps in UTC (`timezone.utc`) and convert only for display.
- **`re.match`**** anchors at the start; ****`re.search`**** looks anywhere.** Both return `None` when nothing matches, so check before calling `.group()`.
- **Use raw strings for patterns,** otherwise `\d` and `\b` are mangled by normal string escapes.
- **Do not parse HTML or other nested formats with regex.** It is for flat text patterns.
- **Month and day order is a guess** for a string like `03/04/2026`. Decide a rule and document it.

### Build

Create `ledger/src/ledger/parse.py`:

```py
"""Chapter 5: dates and regular expressions."""
import re
from datetime import date, datetime

DATE_FORMATS = ("%Y-%m-%d", "%m/%d/%Y", "%d %b %Y")


def parse_date(raw: str) -> date:
    """Accept '2026-01-04', '01/04/2026' or '04 Jan 2026'."""
    text = raw.strip()
    for fmt in DATE_FORMATS:
        try:
            return datetime.strptime(text, fmt).date()
        except ValueError:
            continue
    raise ValueError(f"unrecognized date {raw!r}")


def month_key(d: date) -> str:
    """date(2026, 1, 4) -> '2026-01'."""
    return f"{d:%Y-%m}"


NOISE = re.compile(r"^(POS PURCHASE|DEBIT CARD|CHECKCARD)\s+(\d{2}/\d{2}\s+)?", re.IGNORECASE)
STORE_NUMBER = re.compile(r"\s*#\d+\b")


def merchant(description: str) -> str:
    """'POS PURCHASE 01/02 STARBUCKS #1234' -> 'Starbucks'."""
    text = NOISE.sub("", description.strip())
    text = STORE_NUMBER.sub("", text)
    return " ".join(text.split()).title()
```

`parse_date` is the "try each format" pattern: loop over candidates, `return` on the first success, `continue` on failure, `raise` after the loop if none worked. The regexes are compiled once at module level (`re.compile`) and reused. `NOISE` removes the bank's prefix and the embedded date; `STORE_NUMBER` removes `#1234`.

### Drill

Create `ledger/drills/ch05_dates_regex.py`:

| Function | Behavior |
| --- | --- |
| `days_until(target, today)` | Whole days from `today` to `target` (negative if in the past). Both are `date` objects. |
| `extract_emails(text)` | Every email address in the text, in order. Ignore things like `@nobody` or `foo@bar` (no dot in the domain). |
| `redact_phones(text)` | Replace phones like `(555) 123-4567`, `555-123-4567` and `555.123.4567` with `[phone]`. Leave dates like `2026-01-04` alone. |

Create `ledger/tests/test_drill_ch05.py`:

```py
from datetime import date

from drills.ch05_dates_regex import days_until, extract_emails, redact_phones


def test_days_until():
    assert days_until(date(2026, 3, 1), date(2026, 2, 1)) == 28
    assert days_until(date(2026, 1, 1), date(2026, 1, 3)) == -2
    assert days_until(date(2028, 3, 1), date(2028, 2, 1)) == 29  # leap year handled for you


def test_extract_emails():
    text = "Mail ada@example.com or bob.smith+news@mail.co.uk; not this: @nobody or foo@bar"
    assert extract_emails(text) == ["ada@example.com", "bob.smith+news@mail.co.uk"]


def test_redact_phones():
    text = "Call (555) 123-4567 or 555-123-4567 or 555.123.4567. Order 2026-01-04 stays."
    assert redact_phones(text) == "Call [phone] or [phone] or [phone]. Order 2026-01-04 stays."
```

```bash
uv run pytest tests/test_drill_ch05.py -q
```

Expected when done: `3 passed`.

## Step 7: Chapter 6, dataclasses and enums

**You use this when:** you want a named, typed bundle of fields instead of a loose dict.

### Daily patterns

```txt
>>> from dataclasses import dataclass, field, replace
>>> @dataclass(frozen=True)
... class Point:
...     x: float
...     y: float = 0.0
...     def dist(self, other: "Point") -> float:
...         return ((self.x - other.x) ** 2 + (self.y - other.y) ** 2) ** 0.5
>>> p = Point(3, 4)
>>> p, p.dist(Point(0, 0)), p == Point(3, 4), replace(p, y=0)
(Point(x=3, y=4), 5.0, True, Point(x=3, y=0))
>>> @dataclass
... class Team:
...     name: str
...     members: list[str] = field(default_factory=list)
>>> t = Team("core")
>>> t.members.append("ada")
>>> Team("x").members, t
([], Team(name='core', members=['ada']))
>>> from enum import Enum
>>> class Color(Enum):
...     RED = "red"
...     BLUE = "blue"
>>> Color("red"), Color.RED.value, [c.name for c in Color]
(<Color.RED: 'red'>, 'red', ['RED', 'BLUE'])
```

The patterns to own: `@dataclass` generates `__init__`, `__repr__` and `==` from your annotated fields; `frozen=True` makes instances immutable (safer, and hashable); `field(default_factory=list)` for a mutable default; `replace(obj, y=0)` to make a modified copy; and `Enum` for a fixed set of named choices.

**When NOT to write a class.** Most Python code is plain functions and dicts. Reach for a class only when you have *data with behavior that belongs together* (a dataclass), or state that must be carried across several calls. A function that takes inputs and returns a result does not need to be a method of anything.

### Gotchas

- **Use ****`field(default_factory=list)`**** for list and dict defaults.** Writing `members: list = []` in a dataclass raises an error, because it would be shared between instances.
- **Methods take ****`self`**** first,** and you call them without passing it: `p.dist(q)`.
- **Python has no private attributes.** A leading underscore (`_cache`) means "internal, please do not touch".
- **`@property`**** makes a computed attribute** you read without parentheses (`t.month`). Use it for cheap derived values.
- **Inheritance is rarely the answer.** Prefer composition (a class that holds another) and plain functions.

### Build

Create `ledger/src/ledger/models.py`:

```py
"""Chapter 6: dataclasses and enums."""
from dataclasses import dataclass
from datetime import date
from enum import Enum

from .parse import merchant, month_key, parse_date
from .text import clean_label, parse_amount


class Kind(Enum):
    INCOME = "income"
    EXPENSE = "expense"


@dataclass(frozen=True)
class Transaction:
    date: date
    description: str
    amount: float
    category: str = "uncategorized"

    @property
    def kind(self) -> Kind:
        return Kind.INCOME if self.amount >= 0 else Kind.EXPENSE

    @property
    def month(self) -> str:
        return month_key(self.date)

    @classmethod
    def from_row(cls, row: dict) -> "Transaction":
        """Build one from a csv.DictReader row. Missing cells arrive as None, so use `or ""`."""
        return cls(
            date=parse_date(row.get("date") or ""),
            description=merchant(row.get("description") or ""),
            amount=parse_amount(row.get("amount") or ""),
            category=clean_label(row.get("category") or "") or "uncategorized",
        )
```

Three things to notice. `@classmethod` makes `from_row` an alternative constructor: it receives the class as `cls` and returns an instance (call it as `Transaction.from_row(row)`). The pattern `x or ""` turns `None` into an empty string so the parsers can reject it with a clear `ValueError`, and `clean_label(...) or "uncategorized"` uses `or` to supply a default for an empty result. Finally, `kind` and `month` are computed properties, so the data is stored once and the derived values can never get out of sync.

### Drill

Create `ledger/drills/ch06_classes.py`:

| Name | Behavior |
| --- | --- |
| `Status` (an `Enum`) | Members `TODO`, `DOING`, `DONE` with values `"todo"`, `"doing"`, `"done"`. A classmethod `from_text(text)` turns `" Done "` into `Status.DONE`, and raises `ValueError` listing the valid values for anything else. |
| `Cart` (a dataclass) | Holds `items`, a list of `(name, price, qty)` tuples, empty by default (each cart has its own list). `add(name, price, qty=1)` raises `ValueError` if `qty <= 0`. A `total` property returns the sum of `price * qty`, rounded to 2 decimals. |

Create `ledger/tests/test_drill_ch06.py`:

```py
import pytest

from drills.ch06_classes import Cart, Status


def test_status_from_text():
    assert Status.from_text(" Done ") is Status.DONE
    with pytest.raises(ValueError, match="todo"):
        Status.from_text("finished")


def test_cart_total_and_validation():
    cart = Cart()
    cart.add("pen", 1.10, 3)
    cart.add("book", 12.99)
    assert cart.total == 16.29 and len(cart.items) == 2
    with pytest.raises(ValueError):
        cart.add("bad", 1.0, 0)


def test_carts_do_not_share_their_item_list():
    a, b = Cart(), Cart()
    a.add("x", 1.0)
    assert b.items == [] and Cart() == Cart()  # dataclass gives == for free
```

```bash
uv run pytest tests/test_drill_ch06.py -q
```

Expected when done: `3 passed`.

## Step 8: Chapter 7, files, paths, JSON and CSV

**You use this when:** data comes from files and results go to files. That is most scripts.

### Daily patterns

```txt
>>> import csv, json, tempfile
>>> from pathlib import Path
>>> base = Path(tempfile.mkdtemp())
>>> note = base / "notes" / "todo.txt"
>>> note.parent.mkdir(parents=True, exist_ok=True)
>>> _ = note.write_text("buy milk\ncall Ada\n", encoding="utf-8")
>>> note.read_text(encoding="utf-8").splitlines()
['buy milk', 'call Ada']
>>> note.name, note.stem, note.suffix, note.parent.name, note.exists()
('todo.txt', 'todo', '.txt', 'notes', True)
>>> with note.open("a", encoding="utf-8") as fh:
...     _ = fh.write("water plants\n")
>>> sum(1 for _ in note.open(encoding="utf-8"))
3
>>> data = {"name": "Ada", "tags": ["x", "y"]}
>>> _ = (base / "d.json").write_text(json.dumps(data, indent=2), encoding="utf-8")
>>> json.loads((base / "d.json").read_text(encoding="utf-8")) == data
True
>>> with (base / "t.csv").open("w", newline="", encoding="utf-8") as fh:
...     writer = csv.DictWriter(fh, fieldnames=["name", "age"])
...     _ = writer.writeheader()
...     writer.writerows([{"name": "Ada", "age": 36}])
>>> with (base / "t.csv").open(newline="", encoding="utf-8") as fh:
...     rows = list(csv.DictReader(fh))
>>> rows
[{'name': 'Ada', 'age': '36'}]
>>> sorted(f.name for f in base.rglob("*") if f.is_file())
['d.json', 't.csv', 'todo.txt']
```

The patterns to own: `pathlib.Path` with the `/` operator to join paths, `.read_text()` / `.write_text()` for small files, `with path.open(...) as fh:` for anything else (the file is closed for you, even on errors), `mkdir(parents=True, exist_ok=True)`, `rglob("*.py")` for recursive search, `json.dumps` / `json.loads`, and `csv.DictReader` / `csv.DictWriter` (rows as dicts). Always pass `encoding="utf-8"`.

### Gotchas

- **Always use ****`with`****.** It guarantees the file is closed. Forgetting to close leaks handles and can lose buffered writes.
- **Always pass ****`encoding="utf-8"`****.** The default depends on the operating system.
- **Open CSV files with ****`newline=""`****.** The `csv` module needs that to handle line endings and quoted newlines correctly.
- **Excel adds a BOM (a hidden marker) at the start of CSV files.** Open with `encoding="utf-8-sig"` to remove it, otherwise your first column name is `"﻿date"`.
- **Every CSV value is a string.** `"36"` is not `36`. Convert deliberately (and handle failure).
- **Prefer ****`pathlib`**** over string paths and ****`os.path`****.** It reads better and works on every OS.
- **JSON cannot hold dates or sets directly.** `json.dumps(..., default=str)` is a quick fix for dates.

### Build

Create `ledger/src/ledger/io.py`:

```py
"""Chapter 7: files, paths, CSV and JSON."""
import csv
import json
from pathlib import Path

from .collect import dedupe
from .errors import RowError
from .models import Transaction


def read_rows(path: Path) -> list[dict]:
    # newline="" is what the csv module wants; utf-8-sig quietly drops the BOM Excel adds.
    with path.open(newline="", encoding="utf-8-sig") as fh:
        return list(csv.DictReader(fh))


def load_transactions(path: Path, *, drop_duplicates: bool = True) -> tuple[list[Transaction], list[RowError]]:
    """Return (good transactions, per-row errors). One bad row never stops the run."""
    good, bad = [], []
    for line, row in enumerate(read_rows(path), start=2):  # line 1 is the header
        try:
            good.append(Transaction.from_row(row))
        except ValueError as err:
            bad.append(RowError(line, str(err)))
    if drop_duplicates:
        good = dedupe(good, key=lambda t: (t.date, t.description, t.amount))
    return good, bad


def write_json(path: Path, data: dict) -> None:
    path.parent.mkdir(parents=True, exist_ok=True)
    path.write_text(json.dumps(data, indent=2, default=str) + "\n", encoding="utf-8")
```

This file is where chapters 1 to 6 come together. `load_transactions` is the key real-world pattern: **collect errors instead of crashing**. A `try`/`except ValueError` around each row turns one bad line into a `RowError` with its line number, and the loop carries on. A tuple `(date, description, amount)` is used as the dedupe key because tuples are hashable and compare by value. One caution: real bank data can contain two genuinely identical purchases on one day, which is why `drop_duplicates` is a switch you can turn off.

### Drill

Create `ledger/drills/ch07_files.py`:

| Function | Behavior |
| --- | --- |
| `write_lines(path, lines)` | Write one line per item (each ending with a newline), creating parent folders as needed. |
| `read_lines(path)` | Return the stripped, non-blank lines as a list. |
| `count_files(folder, suffix)` | How many files under `folder`, recursively, end with `suffix` (such as `".py"`). Folders do not count. |
| `csv_to_json(csv_path, json_path)` | Convert a CSV with a header row to a JSON list of objects. Handle a BOM. Return the number of rows. |

Create `ledger/tests/test_drill_ch07.py`:

```py
import json

from drills.ch07_files import count_files, csv_to_json, read_lines, write_lines


def test_write_and_read_lines(tmp_path):
    path = tmp_path / "deep" / "er" / "notes.txt"
    write_lines(path, ["one", "two", "three"])
    assert path.read_text() == "one\ntwo\nthree\n"
    path.write_text("  a \n\n b\n   \n")
    assert read_lines(path) == ["a", "b"]


def test_count_files_is_recursive(tmp_path):
    (tmp_path / "a.py").write_text("")
    (tmp_path / "sub").mkdir()
    (tmp_path / "sub" / "b.py").write_text("")
    (tmp_path / "sub" / "c.txt").write_text("")
    assert count_files(tmp_path, ".py") == 2 and count_files(tmp_path, ".md") == 0


def test_csv_to_json(tmp_path):
    src = tmp_path / "in.csv"
    src.write_text("﻿name,age\nAda,36\nLinus,28\n", encoding="utf-8")  # starts with a BOM, like Excel
    out = tmp_path / "out.json"
    assert csv_to_json(src, out) == 2
    assert json.loads(out.read_text()) == [{"name": "Ada", "age": "36"}, {"name": "Linus", "age": "28"}]
```

```bash
uv run pytest tests/test_drill_ch07.py -q
```

Expected when done: `3 passed`.

## Step 9: Chapter 8, modules, scripts, argparse and logging

**You use this when:** you turn a script into a tool other people (or future you) can run.

### Daily patterns

Imports: three forms cover almost everything.

```py
import json                       # use as json.loads(...)
from pathlib import Path          # use as Path(...)
from .text import money           # relative import: from this package
```

A runnable module ends with the `__main__` guard. The block under it runs only when you run the file directly, not when something imports it:

```py
if __name__ == "__main__":
    raise SystemExit(main())
```

Logging beats `print` for anything that is not the program's actual output. Messages have levels, they go to stderr, and you can turn the detail up or down without editing code:

```py
import logging

log = logging.getLogger("ledger")
logging.basicConfig(level=logging.INFO, format="%(levelname)s: %(message)s")
log.info("loaded %s rows from %s", 20, "data.csv")   # %s style: formatting happens only if emitted
log.warning("skipped %s", "line 13")
log.error("file not found: %s", "nope.csv")
```

`argparse` builds a command line with `--help` for free. You will see it in `cli.py` below and write a small one in the drill.

The standard library has a short list of modules you will reach for constantly. Know they exist; look up the details when you need them:

| Module | Reach for it when you need |
| --- | --- |
| `pathlib`, `json`, `csv` | Files and data formats (Chapter 7) |
| `datetime`, `re` | Dates and text patterns (Chapter 5) |
| `collections` | `Counter`, `defaultdict`, `deque` |
| `itertools`, `functools` | `chain`, `islice`, `groupby`; `lru_cache`, `partial` |
| `dataclasses`, `enum` | Data models (Chapter 6) |
| `argparse`, `logging`, `sys`, `os` | Command-line tools, diagnostics, `os.environ`, exit codes |
| `subprocess`, `shutil` | Run other programs; copy and move files |
| `random`, `statistics`, `time` | Sampling, mean and median, timing |

The third-party shortlist you will add soon: `httpx` or `requests` for HTTP, `pandas` for tables, `pydantic` for validated data models. They are not needed for this tutorial.

### Gotchas

- **`import`**** runs the module's top-level code once.** Keep side effects (reading files, starting servers) out of the top level, or put them behind the `__main__` guard.
- **Do not name your file after a standard-library module** (`json.py`, `random.py`). Your file shadows the real one and imports break in confusing ways.
- **Circular imports** (`a` imports `b` imports `a`) are a design smell. Move the shared piece to a third module.
- **Return an exit code from ****`main()`** (0 for success, non-zero for failure) and call `raise SystemExit(main())`. Shell scripts and CI rely on it.
- **Use ****`log.info("... %s", value)`****, not f-strings, in logging calls.** The message is only built if it will be emitted.
- **Put the ****`-m`**** form in muscle memory:** `uv run python -m ledger.cli` runs a module by its import path.

### Build

First the analysis module, which uses the grouping helpers from Chapter 2. Create `ledger/src/ledger/analysis.py`:

```py
"""Chapter 8: put the pieces together."""
from .collect import group_totals, top_n
from .models import Kind, Transaction


def summarize(transactions: list[Transaction]) -> dict:
    expenses = [t for t in transactions if t.kind is Kind.EXPENSE]
    income = [t for t in transactions if t.kind is Kind.INCOME]
    by_category = group_totals(expenses, key=lambda t: t.category, value=lambda t: -t.amount)
    by_month = group_totals(transactions, key=lambda t: t.month, value=lambda t: t.amount)
    by_merchant = group_totals(expenses, key=lambda t: t.description, value=lambda t: -t.amount)
    return {
        "transactions": len(transactions),
        "income": round(sum(t.amount for t in income), 2),
        "expenses": round(sum(-t.amount for t in expenses), 2),
        "net": round(sum(t.amount for t in transactions), 2),
        "by_category": {k: round(v, 2) for k, v in by_category.items()},
        "by_month": {k: round(v, 2) for k, v in by_month.items()},
        "top_merchants": [(k, round(v, 2)) for k, v in top_n(by_merchant, 5)],
    }
```

This is nearly all comprehensions and the Chapter 2 helpers. `t.kind is Kind.EXPENSE` compares enum members with `is`. Money is rounded to cents on the way out because floats accumulate tiny errors (for a real accounting system you would use the `decimal` module or integer cents).

Now the command-line entry point. Create `ledger/src/ledger/cli.py`:

```py
"""Chapter 8: modules, argparse and logging. Run: uv run ledger data/transactions.csv"""
import argparse
import logging
from pathlib import Path

from .analysis import summarize
from .io import load_transactions, write_json
from .report import format_summary

log = logging.getLogger("ledger")


def build_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(prog="ledger", description="Clean and summarize a bank CSV.")
    parser.add_argument("csv", type=Path, help="path to the transactions CSV")
    parser.add_argument("--json", type=Path, help="also write the summary to this JSON file")
    parser.add_argument("--top", type=int, default=5, help="rows per section (default 5)")
    parser.add_argument("--keep-duplicates", action="store_true", help="do not drop repeated rows")
    parser.add_argument("--strict", action="store_true", help="exit with status 1 if any row was skipped")
    parser.add_argument("-v", "--verbose", action="store_true")
    return parser


def main(argv: list[str] | None = None) -> int:
    args = build_parser().parse_args(argv)
    logging.basicConfig(level=logging.DEBUG if args.verbose else logging.INFO, format="%(levelname)s: %(message)s")
    try:
        transactions, bad_rows = load_transactions(args.csv, drop_duplicates=not args.keep_duplicates)
    except FileNotFoundError:
        log.error("file not found: %s", args.csv)
        return 2
    for err in bad_rows:
        log.warning("skipped %s", err)
    summary = summarize(transactions)
    print(format_summary(summary, top=args.top))
    if args.json:
        write_json(args.json, summary)
        log.info("wrote %s", args.json)
    return 1 if bad_rows and args.strict else 0


if __name__ == "__main__":
    raise SystemExit(main())
```

Notes: `main(argv=None)` takes the argument list as a parameter so tests can call `main([...])` without touching the real command line. `type=Path` makes argparse convert the text for you. `action="store_true"` makes a yes/no flag. The summary report goes to **stdout** via `print` (it is the program's output); status messages go to **stderr** via logging, so you can pipe the report to a file without the noise. `main` returns an exit code: `0` fine, `1` bad rows in strict mode, `2` file not found.

### Drill

Create `ledger/drills/ch08_cli.py`: a tiny greeting tool.

| Part | Behavior |
| --- | --- |
| `main(argv=None) -> int` | An `argparse` parser with a positional `name`, an optional `--times N` (int, default 1) and a `--shout` flag. Print `Hello, <name>!` that many times, upper-cased if `--shout`. Return `0`. |
| `__main__` guard | `raise SystemExit(main())` so the file also works as `uv run python -m drills.ch08_cli Ada`. |

Create `ledger/tests/test_drill_ch08.py`:

```py
import pytest

from drills.ch08_cli import main


def test_greets(capsys):
    assert main(["Ada"]) == 0
    assert capsys.readouterr().out == "Hello, Ada!\n"


def test_options(capsys):
    assert main(["Ada", "--times", "2", "--shout"]) == 0
    assert capsys.readouterr().out == "HELLO, ADA!\nHELLO, ADA!\n"


def test_missing_name_exits_with_usage_error(capsys):
    with pytest.raises(SystemExit) as exit_info:
        main([])
    assert exit_info.value.code == 2
```

```bash
uv run pytest tests/test_drill_ch08.py -q
```

Expected when done: `3 passed`. (`capsys` is a pytest fixture that captures what your code prints.)

## Step 10: Chapter 9, the tools you use every day

**You use this when:** every working session. The tools are part of the 20%.

### uv: projects and dependencies

| You want to | Command |
| --- | --- |
| Make a project | `uv init --package --name myproj` |
| Add a library | `uv add httpx` |
| Add a dev-only tool | `uv add --dev pytest` |
| Remove a library | `uv remove httpx` |
| Install everything from the lockfile | `uv sync` |
| Run anything inside the project environment | `uv run <command>` |
| Run a Python snippet | `uv run python -c "print(1 + 1)"` |
| Open the REPL | `uv run python` |

Think of `pyproject.toml` as `package.json` and `uv.lock` as the lockfile. Commit both. Never commit `.venv/`.

### The REPL

`uv run python` is your scratchpad. Try an idea there before putting it in a file. Useful tricks: `_` holds the last result, `help(str.split)` shows documentation, `dir(obj)` lists what an object offers, and `type(x)` tells you what you are holding.

### pytest: the habits

You have been using pytest all along. These are the parts you will use every day:

```py
import pytest


@pytest.mark.parametrize(("raw", "expected"), [("$1,234.50", 1234.5), ("(12.00)", -12.0)])
def test_many_cases_one_function(raw, expected):
    assert parse_amount(raw) == expected          # plain assert, no special API


def test_it_fails_correctly():
    with pytest.raises(ValueError):
        parse_amount("abc")


def test_files(tmp_path):                          # a fresh temp folder per test
    (tmp_path / "a.txt").write_text("hi")


def test_printing(capsys):                         # captures print() output
    print("hello")
    assert capsys.readouterr().out == "hello\n"
```

Useful flags: `uv run pytest -q` (quiet), `-x` (stop at the first failure), `-k name` (run tests matching a name), `-vv` (full detail on failures), `--lf` (re-run only what failed last time).

### ruff: lint and format

```bash
uv run ruff check .          # find mistakes: unused imports, undefined names, bad patterns
uv run ruff check . --fix    # auto-fix what is safe
uv run ruff format .         # reformat code to one consistent style
```

Run `ruff check` before every commit. It is the habit that catches the most bugs for the least effort.

### The debugger: `breakpoint()`

When `print` is not enough, drop a `breakpoint()` where you want to stop and run the program. You get a prompt inside the running code:

| Key | What it does |
| --- | --- |
| `p expr` | Print an expression's value, for example `p row` |
| `n` | Run the next line |
| `s` | Step into a function call |
| `c` | Continue until the next breakpoint |
| `l` | Show the surrounding source |
| `q` | Quit |

### Gotchas

- **Run everything with ****`uv run`****.** A bare `python` may be a different Python without your dependencies.
- **Do not edit ****`uv.lock`**** by hand;** change dependencies with `uv add` / `uv remove`.
- **If a test passes alone but fails in the suite,** two tests share state (a global, a file). Give each test its own `tmp_path` and its own data.
- **Fix the first error first.** Later errors are often consequences.

## Step 11: Run the finished tool and the full test suite

Create the project's own tests. Create `ledger/tests/test_ledger.py`; they cover every chapter's module:

```py
import json
from datetime import date
from pathlib import Path

import pytest

from ledger.analysis import summarize
from ledger.cli import main
from ledger.collect import count_by, dedupe, group_totals, top_n
from ledger.errors import LedgerError, RowError
from ledger.io import load_transactions, read_rows, write_json
from ledger.models import Kind, Transaction
from ledger.parse import merchant, month_key, parse_date
from ledger.report import format_summary, format_table
from ledger.text import clean_label, money, parse_amount

SAMPLE = Path(__file__).resolve().parent.parent / "data" / "transactions.csv"


# Chapter 1: strings and numbers
@pytest.mark.parametrize(
    ("raw", "expected"),
    [("$1,234.50", 1234.5), ("(12.00)", -12.0), ("-$5", -5.0), (" 7 ", 7.0), ("-1,500.00", -1500.0)],
)
def test_parse_amount(raw, expected):
    assert parse_amount(raw) == expected


@pytest.mark.parametrize("raw", ["", "   ", "abc", "nan", "inf", "-inf", "$"])
def test_parse_amount_rejects_junk(raw):
    with pytest.raises(ValueError):
        parse_amount(raw)


def test_clean_label_and_money():
    assert clean_label("  Food &   Drink ") == "food & drink"
    assert money(-1234.5) == "-$1,234.50" and money(0) == "$0.00" and money(3200) == "$3,200.00"


# Chapter 2: collections
def test_collect_helpers():
    rows = [("a", 1), ("b", 5), ("a", 2), ("c", 5)]
    totals = group_totals(rows, key=lambda r: r[0], value=lambda r: r[1])
    assert totals == {"a": 3.0, "b": 5.0, "c": 5.0}
    assert top_n(totals, 2) == [("b", 5.0), ("c", 5.0)]  # tie broken by key
    assert top_n(totals, 1, largest=False) == [("a", 3.0)]
    assert count_by(rows, key=lambda r: r[0]) == {"a": 2, "b": 1, "c": 1}
    assert dedupe([3, 1, 3, 2, 1]) == [3, 1, 2]
    assert dedupe(["a", "A", "b"], key=str.lower) == ["a", "b"]


# Chapter 3: functions
def test_report_functions_use_keyword_only_options():
    text = format_table([("rent", 1500.0)], title="T", width=10)
    assert text.splitlines()[0] == "T"
    assert text.splitlines()[2] == "rent".ljust(10) + "$1,500.00".rjust(14)  # the width options at work
    with pytest.raises(TypeError):
        format_table([("rent", 1.0)], "T")  # title must be passed by name


# Chapter 4: errors
def test_row_error_message_carries_the_line_number():
    err = RowError(7, "bad date")
    assert str(err) == "line 7: bad date" and err.line == 7 and isinstance(err, LedgerError)


# Chapter 5: dates and regex
def test_parse_date_accepts_three_formats():
    assert parse_date("2026-01-04") == parse_date("01/04/2026") == parse_date(" 04 Jan 2026 ") == date(2026, 1, 4)
    with pytest.raises(ValueError):
        parse_date("not-a-date")
    assert month_key(date(2026, 1, 4)) == "2026-01"


def test_merchant_strips_bank_noise():
    assert merchant("POS PURCHASE 01/02 STARBUCKS #1234") == "Starbucks"
    assert merchant("  Whole Foods   Market #55 ") == "Whole Foods Market"
    assert merchant("checkcard 01/07 Shell Oil #771") == "Shell Oil"
    assert merchant("NETFLIX") == "Netflix"


# Chapter 6: dataclasses
def test_transaction_from_row_and_properties():
    t = Transaction.from_row({"date": "05 Jan 2026", "description": "DEBIT CARD 01/05 SHELL OIL #771",
                              "amount": "(45.20)", "category": "  Transport "})
    assert t == Transaction(date(2026, 1, 5), "Shell Oil", -45.2, "transport")
    assert t.kind is Kind.EXPENSE and t.month == "2026-01"
    assert Transaction.from_row({"date": "2026-01-05", "description": "x", "amount": "1"}).category == "uncategorized"
    with pytest.raises(ValueError):
        Transaction.from_row({"date": None, "description": "x", "amount": "1"})  # a missing cell arrives as None
    with pytest.raises(AttributeError):
        t.amount = 0  # frozen


# Chapter 7: files
def test_read_rows_handles_a_bom(tmp_path):
    path = tmp_path / "a.csv"
    path.write_text("﻿date,amount\n2026-01-01,5\n", encoding="utf-8")
    assert read_rows(path) == [{"date": "2026-01-01", "amount": "5"}]


def test_load_transactions_collects_errors_instead_of_crashing(tmp_path):
    path = tmp_path / "t.csv"
    path.write_text("date,description,amount,category\n"
                    "2026-01-01,A,-5,x\n"
                    "bad,B,-5,x\n"
                    "2026-01-02,C,,x\n"
                    "2026-01-01,A,-5,x\n"
                    "2026-01-03,D\n", encoding="utf-8")  # short row: missing cells
    good, bad = load_transactions(path)
    assert [t.description for t in good] == ["A"]  # the exact repeat was dropped
    assert [e.line for e in bad] == [3, 4, 6]
    assert len(load_transactions(path, drop_duplicates=False)[0]) == 2


def test_write_json_creates_folders(tmp_path):
    out = tmp_path / "a" / "b" / "s.json"
    write_json(out, {"when": date(2026, 1, 1), "n": 1})
    assert json.loads(out.read_text()) == {"when": "2026-01-01", "n": 1}


# Chapter 8: put it together
def test_sample_file_summary_matches_the_tutorial():
    good, bad = load_transactions(SAMPLE)
    assert (len(good), len(bad)) == (16, 4)
    s = summarize(good)
    assert (s["income"], s["expenses"], s["net"]) == (9600.0, 3475.84, 6124.16)
    assert s["by_category"]["housing"] == 3000.0 and s["by_category"]["uncategorized"] == 30.0
    assert s["by_month"] == {"2026-01": 4563.11, "2026-02": 1561.05}
    assert s["top_merchants"][0] == ("Rent February", 1500.0)
    assert "Net:" in format_summary(s)


def test_cli_end_to_end(tmp_path, capsys, caplog):
    out = tmp_path / "summary.json"
    assert main([str(SAMPLE), "--json", str(out), "--top", "3"]) == 0
    text = capsys.readouterr().out
    assert "Spending by category" in text and "$6,124.16" in text
    assert json.loads(out.read_text())["transactions"] == 16
    assert sum("skipped" in r.message for r in caplog.records) == 4
    assert main([str(SAMPLE), "--strict"]) == 1  # bad rows + --strict
    assert main([str(tmp_path / "nope.csv")]) == 2  # missing file
```

Run the tool on the sample data:

```bash
uv run ledger data/transactions.csv --json out/summary.json
```

The four broken rows are reported as warnings (on stderr), then the report prints:

```txt
WARNING: skipped line 13: unrecognized date 'not-a-date'
WARNING: skipped line 14: could not convert string to float: 'abc'
WARNING: skipped line 15: empty amount
WARNING: skipped line 16: not a real amount: 'inf'
INFO: wrote out/summary.json
Transactions: 16
Income:      $9,600.00
Expenses:    $3,475.84
Net:         $6,124.16

Spending by category
------------------------------------------
housing                          $3,000.00
groceries                          $213.90
utilities                           $92.30
food & drink                        $78.95
transport                           $45.20

Net by month
------------------------------------------
2026-01                          $4,563.11
2026-02                          $1,561.05

Top merchants
------------------------------------------
Rent February                    $1,500.00
Rent January                     $1,500.00
Whole Foods Market                 $213.90
Electric Co                         $92.30
Restaurant Le Petit                 $64.20
```

The log lines and the report arrive in a slightly different order in your terminal because they use different output streams; the content is the same. Try a few variations:

```bash
uv run ledger data/transactions.csv --keep-duplicates   # 17 transactions: the NETFLIX repeat is kept
uv run ledger data/transactions.csv --strict            # same report, but exit status 1 because rows were skipped
uv run ledger nope.csv                                  # ERROR: file not found: nope.csv (exit status 2)
uv run ledger --help                                    # argparse wrote this for you
```

Now run the whole suite and the linter. Once every drill file from Chapters 1 to 8 exists:

```bash
uv run pytest -q
uv run ruff check .
```

Expected: `49 passed` and `All checks passed!`. (25 drill tests plus 24 project tests.)

## Step 12: What to learn next, and the checklist

When the daily 20% feels automatic, learn the next layer based on what you actually build, not in a fixed order:

- **Working with tables of data:** pandas (and Matplotlib for charts).
- **Calling web APIs:** `httpx`, plus `json` and `try`/`except`. You already know the hard part.
- **Building web services:** FastAPI (typed, modern) or Django (batteries included).
- **Typed, validated data:** `pydantic`, and optional static checking with `mypy` or `pyright`.
- **Waiting on the network or many tasks at once:** `async` / `await`.
- **Writing your own decorators, generators and context managers:** when you notice repeated setup and cleanup code.

A useful test for "should I learn this now?": *did I need it this week?* If not, it is in the other 80%.

### Checklist

- [ ] `uv run pytest -q` shows `49 passed`
- [ ] You wrote all eight drill files yourself, then compared with the Appendix
- [ ] You can type an f-string with a format spec from memory
- [ ] You can write a list comprehension, a dict comprehension and `sorted(..., key=...)` without looking
- [ ] You can explain why `def f(x, items=[])` is a bug
- [ ] You catch specific exceptions, never a bare `except:`
- [ ] You can read a traceback from the bottom up
- [ ] You use `with` and `pathlib` for every file
- [ ] You can write a `@dataclass` and say when a plain function is better
- [ ] You run `uv run ruff check .` and `uv run pytest -q` before every commit
- [ ] Repeat each drill on day 3 and day 7 from a blank file (the 30-second rule)

## Appendix: drill solutions

Try each drill first. Then compare. There is more than one right answer; if your version passes the tests and reads clearly, it is right.

### Chapter 1: `ledger/drills/ch01_strings.py`

```py
def slugify(text: str) -> str:
    """' Hello, World! 2026 ' -> 'hello-world-2026'."""
    cleaned = "".join(ch if ch.isalnum() else " " for ch in text.lower())
    return "-".join(cleaned.split())


def mask_email(email: str) -> str:
    """'jane.doe@example.com' -> 'j*******@example.com'. ValueError unless there is exactly one '@'."""
    local, sep, domain = email.partition("@")
    if not sep or not local or not domain or "@" in domain:
        raise ValueError(f"not an email: {email!r}")
    return local[0] + "*" * (len(local) - 1) + "@" + domain


def initials(name: str) -> str:
    """'ada  lovelace byron' -> 'ALB'."""
    return "".join(word[0] for word in name.split()).upper()
```

### Chapter 2: `ledger/drills/ch02_collections.py`

```py
from collections import Counter, defaultdict


def most_common_words(text: str, n: int = 3) -> list[tuple[str, int]]:
    """Lower-case, ignore punctuation, ties sorted alphabetically."""
    letters = "".join(ch if ch.isalpha() or ch == "'" else " " for ch in text.lower())
    counts = Counter(letters.split())
    return sorted(counts.items(), key=lambda kv: (-kv[1], kv[0]))[:n]


def invert_dict(d: dict) -> dict:
    """{'a': 1, 'b': 1, 'c': 2} -> {1: ['a', 'b'], 2: ['c']}."""
    inverted = defaultdict(list)
    for key, value in d.items():
        inverted[value].append(key)
    return dict(inverted)


def merge_dicts(*dicts: dict) -> dict:
    """Later dicts win. Never changes its inputs."""
    merged = {}
    for d in dicts:
        merged.update(d)
    return merged
```

### Chapter 3: `ledger/drills/ch03_functions.py`

```py
def chunked(seq, size: int) -> list[list]:
    """chunked([1, 2, 3, 4, 5], 2) -> [[1, 2], [3, 4], [5]]. ValueError if size <= 0."""
    if size <= 0:
        raise ValueError("size must be positive")
    items = list(seq)
    return [items[i : i + size] for i in range(0, len(items), size)]


def first(iterable, predicate=None, default=None):
    """The first item (that satisfies predicate, if given), else default. Works on any iterable."""
    for item in iterable:
        if predicate is None or predicate(item):
            return item
    return default


def clamp(value, *, low=0, high=1):
    """Keep value inside [low, high]. low and high are keyword-only."""
    if low > high:
        raise ValueError("low must not exceed high")
    return max(low, min(high, value))
```

### Chapter 4: `ledger/drills/ch04_errors.py`

```py
import json
from pathlib import Path

from ledger.errors import LedgerError


def safe_int(text, default=None):
    """int(text), or default when it cannot be converted (bad text or None)."""
    try:
        return int(text)
    except (TypeError, ValueError):
        return default


def read_json(path):
    """Load a JSON file. Raise LedgerError (with a helpful message) if missing or invalid."""
    try:
        return json.loads(Path(path).read_text(encoding="utf-8"))
    except FileNotFoundError:
        raise LedgerError(f"file not found: {path}") from None
    except json.JSONDecodeError as err:
        raise LedgerError(f"{path} is not valid JSON (line {err.lineno})") from err


def retry(fn, attempts: int = 3, exceptions: tuple = (Exception,)):
    """Call fn() until it succeeds; after the last failed attempt, re-raise the error."""
    if attempts < 1:
        raise ValueError("attempts must be at least 1")
    for attempt in range(1, attempts + 1):
        try:
            return fn()
        except exceptions:
            if attempt == attempts:
                raise
```

### Chapter 5: `ledger/drills/ch05_dates_regex.py`

```py
import re
from datetime import date

EMAIL = re.compile(r"[\w.+-]+@[\w-]+(?:\.[\w-]+)+")
PHONE = re.compile(r"(?:\(\d{3}\)\s*|\d{3}[-.\s])\d{3}[-.]\d{4}")


def days_until(target: date, today: date) -> int:
    """Whole days from today to target (negative if target is in the past)."""
    return (target - today).days


def extract_emails(text: str) -> list[str]:
    """Every email address in the text, in order."""
    return EMAIL.findall(text)


def redact_phones(text: str) -> str:
    """Replace phones like (555) 123-4567, 555-123-4567 and 555.123.4567 with [phone]."""
    return PHONE.sub("[phone]", text)
```

### Chapter 6: `ledger/drills/ch06_classes.py`

```py
from dataclasses import dataclass, field
from enum import Enum


class Status(Enum):
    TODO = "todo"
    DOING = "doing"
    DONE = "done"

    @classmethod
    def from_text(cls, text: str) -> "Status":
        """' Done ' -> Status.DONE. ValueError (listing the valid values) for anything else."""
        try:
            return cls(text.strip().lower())
        except ValueError:
            raise ValueError(f"unknown status {text!r}; expected one of {[s.value for s in cls]}") from None


@dataclass
class Cart:
    items: list[tuple[str, float, int]] = field(default_factory=list)

    def add(self, name: str, price: float, qty: int = 1) -> None:
        if qty <= 0:
            raise ValueError("qty must be positive")
        self.items.append((name, price, qty))

    @property
    def total(self) -> float:
        return round(sum(price * qty for _, price, qty in self.items), 2)
```

### Chapter 7: `ledger/drills/ch07_files.py`

```py
import csv
import json
from pathlib import Path


def write_lines(path, lines) -> None:
    """Write one line per item, creating parent folders as needed."""
    path = Path(path)
    path.parent.mkdir(parents=True, exist_ok=True)
    path.write_text("".join(f"{line}\n" for line in lines), encoding="utf-8")


def read_lines(path) -> list[str]:
    """Stripped, non-blank lines."""
    return [line.strip() for line in Path(path).read_text(encoding="utf-8").splitlines() if line.strip()]


def count_files(folder, suffix: str) -> int:
    """How many files under folder (recursively) end with suffix, e.g. '.py'."""
    return sum(1 for p in Path(folder).rglob(f"*{suffix}") if p.is_file())


def csv_to_json(csv_path, json_path) -> int:
    """Convert a CSV (header row) to a JSON list of objects. Returns the number of rows."""
    with Path(csv_path).open(newline="", encoding="utf-8-sig") as fh:
        rows = list(csv.DictReader(fh))
    Path(json_path).write_text(json.dumps(rows, indent=2) + "\n", encoding="utf-8")
    return len(rows)
```

### Chapter 8: `ledger/drills/ch08_cli.py`

```py
import argparse


def main(argv: list[str] | None = None) -> int:
    parser = argparse.ArgumentParser(prog="greet")
    parser.add_argument("name")
    parser.add_argument("--times", type=int, default=1)
    parser.add_argument("--shout", action="store_true")
    args = parser.parse_args(argv)
    message = f"Hello, {args.name}!"
    if args.shout:
        message = message.upper()
    for _ in range(args.times):
        print(message)
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

> [!TIP]
> **Verified:** every file in this tutorial was built and run on Python 3.13 with pytest 9.2.0 and ruff 0.17.4: lint clean and 49 tests passing (25 drill tests against the Appendix solutions, 24 project tests). The drill tests were also confirmed to fail with import errors when the drill files do not exist. The `ledger` command was run on the sample CSV and produced the report shown, with exit codes 0, 1 (`--strict`) and 2 (missing file). Every REPL session in the Daily patterns sections was executed with Python's doctest and matched exactly.
>
> **Not verified:** the `uv init --package` output on versions other than uv 0.13.7 (the tutorial tells you to keep your generated `uv_build` range), the Windows-specific commands, the `breakpoint()` key table (standard pdb commands, not run for this page), and the "what to learn next" advice, which is opinion. The `ledger` tool is a teaching project: it stores money as floats and drops identical rows, which is not how you would build accounting software.
