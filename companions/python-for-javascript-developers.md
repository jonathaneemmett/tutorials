# Python for JavaScript Developers: Build a Django REST API (Step by Step)

> [!NOTE]
> Learn Python by mapping it onto what you already know from JavaScript: `.map`, `.filter`, `.every`, objects, classes, modules, npm and Jest. Then build a real, tested Django REST API (projects and tasks with token auth, per-user data, filtering, search, pagination and a summary endpoint). Every file below was built and run: lint clean, 29 tests passing, and the live server was exercised with curl.

## What you will build

| Step | You do |
| --- | --- |
| 0 | Install the tools |
| 1 | Map the JavaScript ecosystem onto Python's |
| 2 | Create the project |
| 3 | Learn the language through runnable lessons (and tests) |
| 4 | Learn the recommended folder structure |
| 5 | Configure Django |
| 6 | Models and migrations |
| 7 | Business logic in plain functions |
| 8 | Serializers (validation) |
| 9 | Filters |
| 10 | Views and URLs |
| 11 | Admin |
| 12 | Run it and call it with curl |
| 13 | Tests |
| 14 | Production notes |

## How to read this tutorial

- JavaScript is shown in comments next to the Python that replaces it, so you can read it left to right.
- **Every path is relative to the `taskboard/` folder (the project root)** you create in Step 2. Every command runs from there unless it says otherwise. Each file is introduced with its full path starting at `taskboard/`.
- Versions used: Python 3.13, Django 6.1, Django REST framework 3.18, django-filter 26.2, pytest 9, pytest-django 4.14, ruff 0.17. Django 6.1 needs Python 3.12 or newer.

## Step 0: Install everything

You need two things: **uv** (installs Python and manages dependencies, like npm plus nvm in one tool) and **Python 3.13**. A code editor with Python support (VS Code with the Python extension, or PyCharm) helps but is optional. Do each sub-step in order and run its check.

### 0.1 Open a terminal

- macOS: Terminal (or iTerm)
- Windows: PowerShell. For the curl commands in Step 12 use WSL or Git Bash, or skip to the tests in Step 13.
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

You never install Python by hand. uv does it:

```bash
uv python install 3.13
uv python list --only-installed
```

Expected: a `cpython-3.13.x` line. Any 3.12 or newer works with this tutorial.

### 0.4 Installation checklist

- [ ] `uv --version` prints a version
- [ ] `uv python list --only-installed` shows Python 3.13 (or 3.12+)

## Step 1: Map the ecosystem

Before any code, here is the translation table. Keep it open while you work.

| JavaScript / Node | Python | Note |
| --- | --- | --- |
| Node.js | CPython (`python`) | The interpreter |
| nvm, `.nvmrc` | `uv python install`, `.python-version` | uv picks the interpreter per project |
| npm / pnpm / yarn | uv (it wraps pip) | `uv add` is `npm install <pkg>` |
| `package.json` | `pyproject.toml` | Dependencies, scripts, tool config |
| `package-lock.json` | `uv.lock` | Commit it |
| `node_modules/` | `.venv/` | A per-project virtual environment. Never commit it |
| `npx cmd` / `npm run` | `uv run cmd` | Runs inside the project's environment, no activation needed |
| `node` REPL | `uv run python` | Interactive prompt |
| Jest / Vitest | pytest | Plain `assert` statements, no `expect` API |
| ESLint + Prettier | ruff (`ruff check`, `ruff format`) | One fast tool for both |
| TypeScript | Type hints (+ mypy or pyright, optional) | Hints are NOT enforced at runtime |
| dotenv | `uv run --env-file .env` | No library needed |
| Express | Django | Django is "batteries included": ORM, auth, admin, migrations |
| Express router | `urls.py` | Maps URLs to view functions or classes |
| Express middleware | `MIDDLEWARE` list in settings | Same idea, configured in one list |
| Prisma / Mongoose / Sequelize | Django ORM (`models.py`) | Migrations are generated from your models |
| Zod / Joi | DRF serializers | Validate input and shape output |
| nodemon | `manage.py runserver` | Auto-reloads on save |
| `console.log` | `print()` |  |
| `JSON.stringify` / `JSON.parse` | `json.dumps` / `json.loads` |  |

Four language differences to absorb right now:

1. **Indentation is syntax.** There are no braces. A colon opens a block and the indented lines are its body. Use 4 spaces.
2. **No `const`, `let` or `var`.** You just assign: `count = 0`. By convention `UPPER_CASE` means a constant, but nothing enforces it.
3. **Naming:** `snake_case` for variables, functions and files, `PascalCase` for classes. Not camelCase.
4. **No semicolons, no `===`.** `==` compares values. Comments start with `#`.

## Step 2: Create the project

```bash
mkdir taskboard && cd taskboard
uv init --bare --name taskboard --python 3.13
uv add django djangorestframework django-filter
uv add --dev pytest pytest-django ruff
```

`uv add` edited `pyproject.toml` and created `.venv/` and `uv.lock` for you, just like `npm install` does. The `--dev` flag is `--save-dev`. Check the imports:

```bash
uv run python -c "import django, rest_framework, django_filters; print(django.__version__, rest_framework.VERSION)"
```

Expected: `6.1.x 3.18.x`. From here, every command uses `uv run`, so you never activate the virtual environment by hand.

Now generate the Django project and app, and create the folders we will fill:

```bash
uv run django-admin startproject config .
uv run python manage.py startapp tasks
rm tasks/tests.py
mkdir -p tasks/tests lessons
touch tasks/tests/__init__.py lessons/__init__.py
```

Run these in order, one block at a time. Then check the result:

```bash
ls . tasks tasks/tests lessons
```

Expected: `config`, `lessons`, `manage.py`, `pyproject.toml`, `tasks` and `uv.lock` in the project root. `tasks/` contains `models.py`, `views.py`, `admin.py`, `apps.py`, `migrations` and `tests`. `tasks/tests/` and `lessons/` each contain `__init__.py`. If `tasks/` does not exist, `startapp` did not run or failed. Run `uv run python manage.py startapp tasks` again from the `taskboard` folder (the one containing `manage.py`), then repeat the `mkdir -p` and `touch` lines.

The trailing `.` in `startproject config .` matters: it puts `manage.py` in the current folder instead of nesting another folder. `__init__.py` marks a folder as an importable package (JavaScript has no equivalent because it uses file paths).

Create two files in the project root, next to `pyproject.toml`: `taskboard/.env` holds your settings and `taskboard/.gitignore` keeps secrets and build output out of git. The commands create both (you are already inside `taskboard/`):

```bash
cat > .env <<'EOF'
DJANGO_DEBUG=1
DJANGO_SECRET_KEY=local-dev-change-me
DJANGO_ALLOWED_HOSTS=localhost,127.0.0.1
EOF
printf '.env\n.venv/\n__pycache__/\ndb.sqlite3\n.pytest_cache/\n.ruff_cache/\n' > .gitignore
```

On Windows, create both files in your editor, saved in the `taskboard` folder, with the same contents.

Finally, append these sections to the end of `taskboard/pyproject.toml` (open it in your editor and paste at the bottom). They configure pytest and ruff, like the `jest` and `eslintConfig` keys in `package.json`:

```toml
[tool.pytest.ini_options]
DJANGO_SETTINGS_MODULE = "config.settings"
python_files = ["test_*.py"]
testpaths = ["tasks", "lessons"]

[tool.ruff]
extend-exclude = ["migrations"]

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B"]
ignore = ["E501"]
```

When we are done, the project looks like this:

```txt
taskboard/
├── .env
├── .gitignore
├── pyproject.toml
├── uv.lock
├── manage.py
├── config/                 # project-wide settings and root URLs
│   ├── settings.py
│   └── urls.py
├── lessons/                # Step 3: language lessons, each with tests
│   ├── basics.py
│   ├── arrays.py
│   ├── functions.py
│   ├── classes.py
│   └── test_lessons.py
└── tasks/                  # the Django app
    ├── models.py
    ├── services.py
    ├── serializers.py
    ├── filters.py
    ├── views.py
    ├── urls.py
    ├── admin.py
    ├── migrations/
    └── tests/
        ├── test_services.py
        └── test_api.py
```

## Step 3: Learn Python through runnable lessons

Each lesson is a real file with the JavaScript equivalent in comments, and Step 3.5 runs tests that prove every claim. Read the code, then run it.

### 3.1 Basics and the gotchas

Create `taskboard/lessons/basics.py`:

```py
"""Core language differences a JavaScript developer trips over first."""


def truthiness(value) -> bool:
    # JS:     [] and {} are truthy.   Python: empty containers are falsy.
    # Falsy in Python: None, False, 0, 0.0, "", [], {}, set(), ()
    return bool(value)


def describe_port(port: int | None) -> str:
    # JS:     const p = port ?? 8000     (only null/undefined fall back)
    # Python: no ?? operator. `port or 8000` also replaces 0, like JS `||`.
    p = port if port is not None else 8000
    return f"port {p}"


def nested_lookup(data: dict) -> str | None:
    # JS:     data.user?.profile?.city
    # Python: dict.get with a default, or catch KeyError.
    return data.get("user", {}).get("profile", {}).get("city")


def greeting(name: str, items: list[str]) -> str:
    # JS template literal: `Hi ${name}, you have ${items.length} items`
    return f"Hi {name}, you have {len(items)} items"


def same_value_vs_same_object() -> tuple[bool, bool]:
    # JS has == and ===. Python's == compares values (like a deep ===).
    # `is` asks "same object in memory?" Use it only for None: `x is None`.
    a, b = [1, 2], [1, 2]
    return a == b, a is b


def parse_age(text: str) -> int:
    # JS:     try { ... } catch (e) { ... }     throw new Error("...")
    # Python: try / except / raise, and you catch specific exception types.
    try:
        age = int(text)
    except ValueError:
        raise ValueError(f"not a number: {text!r}") from None
    if age < 0:
        raise ValueError("age cannot be negative")
    return age


def read_first_line(path: str) -> str:
    # JS:     try { fh = await open(path) } finally { await fh.close() }
    # Python: `with` closes the file for you, even if an error is raised.
    with open(path, encoding="utf-8") as fh:
        return fh.readline().rstrip("\n")
```

What to notice:

- **Truthiness is the biggest trap.** In JS `if ([])` is true. In Python an empty list, dict, string or `0` is false. Test for `None` explicitly (`is None`) when `0` or `[]` are valid values.
- **`None` replaces both `null` and `undefined`.** Reading a missing dict key with `d["x"]` raises `KeyError`; `d.get("x")` returns `None` like JS returns `undefined`.
- Type hints like `port: int | None` and `-> str` are documentation plus editor and checker support. Python itself ignores them at runtime.
- Errors are exceptions you `raise`, not `throw`. `except ValueError` catches one type, the way you would check `e instanceof TypeError`.

### 3.2 Arrays and objects: map, filter, every and friends

A **list** is an array. A **dict** is an object (or a `Map`). A list of dicts is an array of objects. The array methods you know are mostly **list comprehensions** and built-in functions:

| JavaScript | Python |
| --- | --- |
| `users.map(u => u.name)` | `[u["name"] for u in users]` |
| `users.filter(u => u.age > 30)` | `[u for u in users if u["age"] > 30]` |
| `users.filter(...).map(...)` | `[f(u) for u in users if cond(u)]` |
| `users.every(u => u.age >= 18)` | `all(u["age"] >= 18 for u in users)` |
| `users.some(u => u.admin)` | `any(u["admin"] for u in users)` |
| `users.find(u => ...)` | `next((u for u in users if ...), None)` |
| `users.findIndex(u => ...)` | `next((i for i, u in enumerate(users) if ...), -1)` |
| `users.reduce((s, u) => s + u.age, 0)` | `sum(u["age"] for u in users)` (or `functools.reduce`) |
| `[...users].sort((a, b) => b.age - a.age)` | `sorted(users, key=lambda u: u["age"], reverse=True)` |
| `arr.flat()` | `list(chain.from_iterable(arr))` |
| `arr.includes(x)` | `x in arr` |
| `arr.slice(1, 3)`, `arr.at(-1)` | `arr[1:3]`, `arr[-1]` |
| `[...a, ...b]` | `[*a, *b]` |
| `{...a, ...b}` | `{**a, **b}` |
| `Object.keys / values / entries(o)` | `list(d)`, `d.values()`, `d.items()` |
| `Object.fromEntries(...)` | `{k: v for k, v in ...}` (dict comprehension) |
| `new Set(arr)` | `set(arr)` |
| `Object.groupBy` | `defaultdict(list)` in a loop |

Create `taskboard/lessons/arrays.py`:

```py
"""JavaScript array and object methods, in Python.

A list of dicts is the Python version of an array of objects.
"""
from collections import Counter, defaultdict
from functools import reduce
from itertools import chain

USERS = [
    {"name": "Ada", "age": 36, "admin": True},
    {"name": "Linus", "age": 28, "admin": False},
    {"name": "Grace", "age": 45, "admin": True},
]


def names(users):  # users.map(u => u.name)
    return [u["name"] for u in users]


def over_30(users):  # users.filter(u => u.age > 30)
    return [u for u in users if u["age"] > 30]


def admin_names_upper(users):  # users.filter(u => u.admin).map(u => u.name.toUpperCase())
    return [u["name"].upper() for u in users if u["admin"]]


def all_adults(users):  # users.every(u => u.age >= 18)
    return all(u["age"] >= 18 for u in users)


def any_admin(users):  # users.some(u => u.admin)
    return any(u["admin"] for u in users)


def find_user(users, name):  # users.find(u => u.name === name)  -> undefined if missing
    return next((u for u in users if u["name"] == name), None)


def find_index(users, name):  # users.findIndex(u => u.name === name)  -> -1 if missing
    return next((i for i, u in enumerate(users) if u["name"] == name), -1)


def total_age(users):  # users.reduce((sum, u) => sum + u.age, 0)
    return reduce(lambda total, u: total + u["age"], users, 0)  # or simply sum(u["age"] for u in users)


def oldest_first(users):  # [...users].sort((a, b) => b.age - a.age)
    return sorted(users, key=lambda u: u["age"], reverse=True)  # sorted() returns a NEW list


def flatten(nested):  # nested.flat()
    return list(chain.from_iterable(nested))


def by_name(users):  # Object.fromEntries(users.map(u => [u.name, u]))
    return {u["name"]: u for u in users}


def group_by_admin(users):  # Object.groupBy(users, u => u.admin)
    groups = defaultdict(list)
    for u in users:
        groups[u["admin"]].append(u["name"])
    return dict(groups)


def count_words(text):  # a Map used as a counter
    return dict(Counter(text.split()))


def slices(items):
    # items.slice(1, 3), items.at(-1), [...items].reverse()
    return items[1:3], items[-1], items[::-1]


def spread_and_destructure():
    first, *rest = [1, 2, 3, 4]  # const [first, ...rest] = [1, 2, 3, 4]
    merged_list = [*rest, 99]  # [...rest, 99]
    merged_dict = {**{"a": 1, "b": 2}, **{"b": 3}}  # {...{a:1,b:2}, ...{b:3}}  (later wins)
    a, b = 1, 2
    a, b = b, a  # [a, b] = [b, a]
    return first, rest, merged_list, merged_dict, (a, b)


def object_helpers():
    d = {"x": 1, "y": 2}
    # Object.keys(d), Object.values(d), Object.entries(d)
    return list(d), list(d.values()), list(d.items())


def unique_sorted(items):  # [...new Set(items)].sort()
    return sorted(set(items))


def pairs(a, b):  # a.map((x, i) => [x, b[i]])
    return list(zip(a, b, strict=True))


def membership(names_list):
    # names.includes("Ada")  and  "key" in obj
    return "Ada" in names_list, "x" in {"x": 1}
```

What to notice:

- **Comprehensions read inside out.** `[u["name"] for u in users if u["admin"]]` is "the name of each user, for each user, if admin". The filter goes last, the map goes first.
- **There is no method chaining on lists.** Python favors functions and comprehensions over `.map().filter().reduce()`. You will rarely miss it.
- **`all([])` is `True` and `any([])` is `False`**, exactly like `[].every()` and `[].some()` in JS. The API in Step 7 guards against this.
- **`sorted(x)` returns a new list; `x.sort()` sorts in place and returns `None`.** Writing `items = items.sort()` is a classic bug.
- Dict keys use brackets: `u["name"]`, not `u.name`. (Classes use dots, see 3.4.)
- `lambda` is a one-expression arrow function. For anything longer, write a named `def`.

### 3.3 Functions

Create `taskboard/lessons/functions.py`:

```py
"""Functions: arguments, closures, lambdas and decorators."""
import functools


def greet(name: str, greeting: str = "Hello", *, punctuation: str = "!") -> str:
    # JS:  function greet(name, greeting = "Hello", { punctuation = "!" } = {})
    # The bare `*` makes everything after it keyword-only, like an options object.
    return f"{greeting}, {name}{punctuation}"


def total(*numbers: float) -> float:
    # JS: (...numbers) => numbers.reduce(...)   `numbers` is a tuple here.
    return sum(numbers)


def build_url(path: str, **params: object) -> str:
    # JS: (path, params) => ...   **params collects keyword arguments into a dict.
    query = "&".join(f"{key}={value}" for key, value in sorted(params.items()))
    return f"{path}?{query}" if query else path


def make_counter():
    # A closure, same idea as JS. Rebinding an outer variable needs `nonlocal`.
    count = 0

    def increment() -> int:
        nonlocal count
        count += 1
        return count

    return increment


def buggy_append(item, bucket=[]):  # noqa: B006  (deliberately wrong, see below)
    # GOTCHA: the default list is created ONCE, when the function is defined,
    # and shared by every call. JS evaluates defaults on every call.
    bucket.append(item)
    return bucket


def safe_append(item, bucket=None):
    if bucket is None:
        bucket = []
    bucket.append(item)
    return bucket


square = lambda n: n * n  # noqa: E731  JS: n => n * n   (one expression only; use def for more)


def log_calls(fn):
    """A decorator: a function that takes a function and returns a wrapped one.

    JS equivalent: const logCalls = fn => (...args) => { log(...); return fn(...args) }
    """
    calls: list[str] = []

    @functools.wraps(fn)  # keep the original name and docstring
    def wrapper(*args, **kwargs):
        calls.append(f"{fn.__name__}{args}")
        return fn(*args, **kwargs)

    wrapper.calls = calls
    return wrapper


@log_calls  # same as: add = log_calls(add)
def add(a: int, b: int) -> int:
    return a + b
```

What to notice:

- **Keyword arguments are first class.** Python has no "pass an options object" habit because you can write `greet("Ada", punctuation="?")` directly. Django and DRF use keyword arguments everywhere (`Project.objects.filter(owner=user, name="x")`).
- `*args` collects extra positional arguments (JS rest parameters), `**kwargs` collects extra named ones into a dict.
- **The mutable default argument trap** (`bucket=[]`) bites every JS developer once. Use `None` and create the list inside.
- **Decorators** (`@log_calls`) are higher-order functions with nicer syntax. Django uses them for things like `@action` and `@login_required`.
- Functions are first class, like JS: you can pass them around, return them and store them in dicts.

### 3.4 Classes

Create `taskboard/lessons/classes.py`:

```py
"""Classes: constructors, methods, properties, dataclasses and inheritance."""
from dataclasses import dataclass


class Money:
    # JS: class Money { constructor(amount, currency) { this.amount = amount ... } }
    def __init__(self, amount: int, currency: str = "USD"):
        self.amount = amount  # `self` is the explicit `this`; it is always the first parameter
        self.currency = currency
        self._cache = None  # a leading underscore means "internal". Python has no real private

    def __repr__(self) -> str:  # like toString(), but meant for developers
        return f"Money({self.amount}, {self.currency!r})"

    def __eq__(self, other) -> bool:  # lets `==` compare by value
        return isinstance(other, Money) and (self.amount, self.currency) == (other.amount, other.currency)

    def __add__(self, other: "Money") -> "Money":  # operator overloading: a + b
        if other.currency != self.currency:
            raise ValueError("currency mismatch")
        return Money(self.amount + other.amount, self.currency)

    @property  # JS: get dollars() { ... }  Accessed without parentheses.
    def dollars(self) -> float:
        return self.amount / 100

    @classmethod  # JS: static factory that can build `cls`, so subclasses work too
    def from_dollars(cls, dollars: float, currency: str = "USD") -> "Money":
        return cls(round(dollars * 100), currency)

    @staticmethod  # JS: static method. It gets no self or cls
    def is_valid_currency(code: str) -> bool:
        return len(code) == 3 and code.isupper()


@dataclass(frozen=True)  # generates __init__, __repr__ and __eq__; frozen = immutable
class Point:
    # Closest to a TypeScript interface/type you can also instantiate.
    x: float
    y: float = 0.0


class Animal:
    def __init__(self, name: str):
        self.name = name

    def speak(self) -> str:
        return f"{self.name} makes a sound"


class Dog(Animal):  # JS: class Dog extends Animal
    def speak(self) -> str:
        return super().speak() + ": woof"  # JS: super.speak()


def announce(thing) -> str:
    # Duck typing: no interface needed. If it has .speak(), it works.
    return thing.speak()
```

What to notice:

- **`self` is explicit.** Every method lists it first and you use it for every instance attribute. You create instances without `new`: `Money(150)`.
- **Dunder ("double underscore") methods** hook into the language: `__init__` is the constructor, `__repr__` is debug output, `__eq__` is `==`, `__add__` is `+`. This is how Django models, querysets and DRF fields feel so natural.
- **No `private`.** `_name` is a convention meaning "do not touch from outside".
- **`@dataclass`** removes the boilerplate for data holders. You will meet the same idea as Django models and DRF serializers.
- Inheritance uses parentheses: `class Dog(Animal)`. Python has **duck typing**, so you rarely need interfaces.

### 3.5 Run the lessons

Create `taskboard/lessons/test_lessons.py`. This is your first pytest file. A test is a function named `test_*` that uses plain `assert`:

```py
"""Every claim in the tutorial's language lessons, checked."""
import pytest

from lessons import arrays, basics, classes, functions


def test_truthiness_differs_from_js():
    assert not basics.truthiness([]) and not basics.truthiness({}) and not basics.truthiness("")
    assert not basics.truthiness(0) and not basics.truthiness(None)
    assert basics.truthiness("0") and basics.truthiness([0])


def test_none_vs_zero_fallback():
    assert basics.describe_port(None) == "port 8000"
    assert basics.describe_port(0) == "port 0"  # `0 or 8000` would have given 8000


def test_safe_nested_lookup():
    assert basics.nested_lookup({"user": {"profile": {"city": "Oslo"}}}) == "Oslo"
    assert basics.nested_lookup({}) is None


def test_fstring_and_equality():
    assert basics.greeting("Ada", ["a", "b"]) == "Hi Ada, you have 2 items"
    assert basics.same_value_vs_same_object() == (True, False)


def test_errors(tmp_path):
    assert basics.parse_age("42") == 42
    with pytest.raises(ValueError, match="not a number"):
        basics.parse_age("abc")
    with pytest.raises(ValueError, match="negative"):
        basics.parse_age("-1")
    f = tmp_path / "a.txt"
    f.write_text("first\nsecond\n", encoding="utf-8")
    assert basics.read_first_line(str(f)) == "first"


def test_array_methods():
    u = arrays.USERS
    assert arrays.names(u) == ["Ada", "Linus", "Grace"]
    assert [x["name"] for x in arrays.over_30(u)] == ["Ada", "Grace"]
    assert arrays.admin_names_upper(u) == ["ADA", "GRACE"]
    assert arrays.all_adults(u) and arrays.any_admin(u)
    assert arrays.find_user(u, "Grace")["age"] == 45 and arrays.find_user(u, "Nobody") is None
    assert arrays.find_index(u, "Linus") == 1 and arrays.find_index(u, "Nobody") == -1
    assert arrays.total_age(u) == 109
    assert [x["name"] for x in arrays.oldest_first(u)] == ["Grace", "Ada", "Linus"]
    assert [x["name"] for x in u] == ["Ada", "Linus", "Grace"]  # sorted() did not mutate
    assert arrays.flatten([[1, 2], [3], []]) == [1, 2, 3]
    assert arrays.by_name(u)["Ada"]["age"] == 36
    assert arrays.group_by_admin(u) == {True: ["Ada", "Grace"], False: ["Linus"]}
    assert arrays.count_words("a b a") == {"a": 2, "b": 1}
    assert arrays.slices([1, 2, 3, 4]) == ([2, 3], 4, [4, 3, 2, 1])
    assert arrays.unique_sorted([3, 1, 3, 2]) == [1, 2, 3]
    assert arrays.pairs([1, 2], ["a", "b"]) == [(1, "a"), (2, "b")]
    assert arrays.membership(["Ada"]) == (True, True)


def test_every_and_some_on_empty_lists():
    assert all(x > 0 for x in []) is True  # same as [].every(...)
    assert any(x > 0 for x in []) is False  # same as [].some(...)


def test_spread_destructure_and_objects():
    first, rest, merged_list, merged_dict, swapped = arrays.spread_and_destructure()
    assert (first, rest, merged_list) == (1, [2, 3, 4], [2, 3, 4, 99])
    assert merged_dict == {"a": 1, "b": 3} and swapped == (2, 1)
    assert arrays.object_helpers() == (["x", "y"], [1, 2], [("x", 1), ("y", 2)])


def test_sort_in_place_returns_none():
    items = [3, 1, 2]
    assert items.sort() is None and items == [1, 2, 3]  # list.sort() mutates and returns None


def test_function_arguments():
    assert functions.greet("Ada") == "Hello, Ada!"
    assert functions.greet("Ada", punctuation="?", greeting="Hi") == "Hi, Ada?"
    with pytest.raises(TypeError):
        functions.greet("Ada", "Hi", "?")  # punctuation is keyword-only
    assert functions.total(1, 2, 3) == 6
    assert functions.build_url("/search", q="x", page=2) == "/search?page=2&q=x"
    assert functions.build_url("/search") == "/search"


def test_closure_and_lambda():
    counter = functions.make_counter()
    assert (counter(), counter(), counter()) == (1, 2, 3)
    assert functions.square(4) == 16


def test_mutable_default_gotcha():
    assert functions.buggy_append(1) == [1]
    assert functions.buggy_append(2) == [1, 2]  # surprise: shared list
    assert functions.safe_append(1) == [1]
    assert functions.safe_append(2) == [2]


def test_decorator():
    assert functions.add(2, 3) == 5
    assert functions.add.__name__ == "add"
    assert functions.add.calls == ["add(2, 3)"]


def test_money_class():
    a, b = classes.Money(150), classes.Money(250)
    assert (a + b) == classes.Money(400)
    assert repr(a) == "Money(150, 'USD')"
    assert a.dollars == 1.5
    assert classes.Money.from_dollars(2.5) == classes.Money(250)
    assert classes.Money.is_valid_currency("EUR") and not classes.Money.is_valid_currency("eur")
    with pytest.raises(ValueError):
        a + classes.Money(1, "EUR")


def test_dataclass_and_inheritance():
    p = classes.Point(1.0)
    assert p == classes.Point(1.0, 0.0) and repr(p) == "Point(x=1.0, y=0.0)"
    with pytest.raises(AttributeError):
        p.x = 5  # frozen
    assert classes.announce(classes.Dog("Rex")) == "Rex makes a sound: woof"
```

Run them:

```bash
uv run pytest lessons -q
```

Expected: `15 passed`. Want to experiment? Open a REPL (JS: `node`) and try things:

```bash
uv run python
>>> from lessons import arrays
>>> arrays.over_30(arrays.USERS)
>>> exit()
```

## Step 4: Recommended folder structure

Django has strong conventions, and following them means any Django developer can navigate your project. The rules of thumb:

- **`config/`** (the folder `startproject` made) holds only project-wide wiring: `settings.py`, root `urls.py`, `wsgi.py` and `asgi.py`. Put no business code here.
- **An "app" is a feature folder**, like `tasks/`. It is a Python package that owns its models, URLs and tests. Start with one app and add another when a part of the domain could stand alone (for example `billing/` or `accounts/`).
- **One file per concern inside an app.** This is the layout every Django and DRF project shares:

| File | JavaScript analogy | Purpose |
| --- | --- | --- |
| `models.py` | Prisma schema / Mongoose model | Database tables as classes |
| `migrations/` | `prisma migrate` files | Generated history of schema changes. Commit them |
| `serializers.py` | Zod schema + response DTO | Validate input, shape output |
| `views.py` | Route handlers / controllers | HTTP in, HTTP out. Keep them thin |
| `urls.py` | Express router | Maps URLs to views |
| `filters.py` | Query-string parsing | `?status=done&overdue=true` |
| `services.py` | Service layer / utils | Plain business logic with no HTTP and no database, easy to test |
| `admin.py` | (nothing) | A free admin UI for your models |
| `tests/` | `__tests__/` | pytest files named `test_*.py` |

- **Keep views thin and logic in `services.py`.** A function that takes plain data and returns plain data can be tested without a server or a database (you will see this in Step 7).
- **Tests live inside the app** (`tasks/tests/`) next to the code they cover, or in a top-level `tests/` folder for cross-app tests.
- **Do not commit** `.venv/`, `__pycache__/`, `.env` or `db.sqlite3`. **Do commit** `pyproject.toml`, `uv.lock` and `migrations/`.
- If you build a reusable *library* instead of an app, use the `src/` layout (`src/mypackage/...`). Django apps conventionally sit at the project root as above.

## Step 5: Configure Django

Replace the whole contents of `taskboard/config/settings.py`. This version is trimmed and reads per-environment values from environment variables:

```py
"""Django settings. Per-environment values come from environment variables."""
import os
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent

DEBUG = os.environ.get("DJANGO_DEBUG", "1") == "1"
SECRET_KEY = os.environ.get("DJANGO_SECRET_KEY", "dev-only-insecure-key")
if not DEBUG and SECRET_KEY == "dev-only-insecure-key":
    raise RuntimeError("Set DJANGO_SECRET_KEY when DJANGO_DEBUG=0")

ALLOWED_HOSTS = [
    host.strip()
    for host in os.environ.get("DJANGO_ALLOWED_HOSTS", "localhost,127.0.0.1").split(",")
    if host.strip()
]

INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "rest_framework",
    "rest_framework.authtoken",
    "django_filters",
    "tasks",
]

MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
]

ROOT_URLCONF = "config.urls"

TEMPLATES = [
    {
        "BACKEND": "django.template.backends.django.DjangoTemplates",
        "DIRS": [],
        "APP_DIRS": True,
        "OPTIONS": {
            "context_processors": [
                "django.template.context_processors.request",
                "django.contrib.auth.context_processors.auth",
                "django.contrib.messages.context_processors.messages",
            ],
        },
    },
]

WSGI_APPLICATION = "config.wsgi.application"

DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.sqlite3",
        "NAME": BASE_DIR / "db.sqlite3",
    }
}

AUTH_PASSWORD_VALIDATORS = [
    {"NAME": "django.contrib.auth.password_validation.UserAttributeSimilarityValidator"},
    {"NAME": "django.contrib.auth.password_validation.MinimumLengthValidator"},
    {"NAME": "django.contrib.auth.password_validation.CommonPasswordValidator"},
    {"NAME": "django.contrib.auth.password_validation.NumericPasswordValidator"},
]

LANGUAGE_CODE = "en-us"
TIME_ZONE = "UTC"
USE_I18N = True
USE_TZ = True

STATIC_URL = "static/"
DEFAULT_AUTO_FIELD = "django.db.models.BigAutoField"

REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework.authentication.TokenAuthentication",
    ],
    "DEFAULT_PERMISSION_CLASSES": ["rest_framework.permissions.IsAuthenticated"],
    "DEFAULT_FILTER_BACKENDS": [
        "django_filters.rest_framework.DjangoFilterBackend",
        "rest_framework.filters.SearchFilter",
        "rest_framework.filters.OrderingFilter",
    ],
    "DEFAULT_PAGINATION_CLASS": "rest_framework.pagination.PageNumberPagination",
    "PAGE_SIZE": 20,
}
```

Things to notice, as a JS developer:

- `settings.py` is a plain Python module of constants. There is no JSON or YAML config. Anything you can compute, you can use here (see the `ALLOWED_HOSTS` list comprehension).
- `os.environ.get("X", default)` is `process.env.X ?? default`.
- `REST_FRAMEWORK` sets the defaults for every API view: token auth, "must be logged in", filtering, search, ordering and pagination of 20 per page.
- The guard at the top refuses to start with the dev secret key when `DEBUG` is off.

Replace the whole contents of `taskboard/config/urls.py`:

```py
from django.contrib import admin
from django.urls import include, path

urlpatterns = [
    path("admin/", admin.site.urls),
    path("api/", include("tasks.urls")),
]
```

`include("tasks.urls")` is like `app.use("/api", tasksRouter)` in Express. Check the configuration is valid:

```bash
uv run --env-file .env python manage.py check
```

Expected: `System check identified no issues (0 silenced).`

## Step 6: Models and migrations

A **model** is a class that becomes a database table. Replace the whole contents of `taskboard/tasks/models.py`:

```py
from datetime import date

from django.conf import settings
from django.db import models
from django.utils import timezone


class Project(models.Model):
    owner = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="projects"
    )
    name = models.CharField(max_length=120)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ["name"]
        constraints = [
            models.UniqueConstraint(fields=["owner", "name"], name="unique_project_name_per_owner"),
        ]

    def __str__(self) -> str:
        return self.name


class Task(models.Model):
    class Status(models.TextChoices):
        TODO = "todo", "To do"
        DOING = "doing", "Doing"
        DONE = "done", "Done"

    class Priority(models.IntegerChoices):
        LOW = 1, "Low"
        MEDIUM = 2, "Medium"
        HIGH = 3, "High"

    project = models.ForeignKey(Project, on_delete=models.CASCADE, related_name="tasks")
    title = models.CharField(max_length=200)
    status = models.CharField(max_length=10, choices=Status, default=Status.TODO)
    priority = models.IntegerField(choices=Priority, default=Priority.MEDIUM)
    due_date = models.DateField(null=True, blank=True)
    completed_at = models.DateTimeField(null=True, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ["-created_at", "-id"]

    def __str__(self) -> str:
        return self.title

    @property
    def is_overdue(self) -> bool:
        return bool(
            self.due_date and self.due_date < date.today() and self.status != self.Status.DONE
        )

    def save(self, *args, **kwargs):
        # Keep completed_at consistent with status, whoever changes the status.
        if self.status == self.Status.DONE and self.completed_at is None:
            self.completed_at = timezone.now()
        elif self.status != self.Status.DONE:
            self.completed_at = None
        super().save(*args, **kwargs)
```

What to notice:

- Fields are class attributes. `models.CharField(max_length=120)` is a column definition, like a Prisma schema line.
- `ForeignKey(..., related_name="tasks")` lets you write `project.tasks.all()` to get a project's tasks.
- `class Meta` (a nested class) holds table options: default ordering and a unique constraint (one owner cannot have two projects with the same name).
- `@property` makes `task.is_overdue` read like a field. `save()` overrides the built-in and calls `super().save(...)`, so the invariant "completed_at is set exactly when status is done" holds no matter who changes the status.

Generate and apply the database migration. Django diffs your models against the last migration and writes the SQL for you:

```bash
uv run --env-file .env python manage.py makemigrations tasks
uv run --env-file .env python manage.py migrate
```

Expected: `Create model Project`, `Create model Task`, `Create constraint unique_project_name_per_owner on model project`, then a list of `Applying ... OK` lines. `makemigrations` wrote `taskboard/tasks/migrations/0001_initial.py`. Commit that file. Whenever you change a model, run both commands again.

Try the ORM in a shell (JS: a REPL with your models loaded):

```bash
uv run --env-file .env python manage.py shell
>>> from django.contrib.auth import get_user_model
>>> from tasks.models import Project, Task
>>> u = get_user_model().objects.create_user("scratch", password="s3cret-pass-99")
>>> p = Project.objects.create(owner=u, name="Demo")
>>> Task.objects.create(project=p, title="hello", priority=3)
>>> Task.objects.filter(project=p, priority__gte=2).count()
>>> exit()
```

The `priority__gte=2` double-underscore syntax is how queries express operators (`>=` here). `Task.objects.filter(...)` is the equivalent of `prisma.task.findMany({ where })`. Delete `db.sqlite3` afterwards to start clean (`rm db.sqlite3`, then run `migrate` again).

## Step 7: Business logic as plain functions

This file knows nothing about HTTP or the database. It takes any iterable of objects with `.status`, `.priority` and `.is_overdue`, which makes it trivial to test. Create `taskboard/tasks/services.py`:

```py
"""Business logic that does not need HTTP. Plain functions, easy to test."""
from collections import Counter

STATUSES = ("todo", "doing", "done")


def summarize(tasks) -> dict:
    """Summarize any iterable of objects with .status, .priority and .is_overdue."""
    tasks = list(tasks)
    counts = Counter(t.status for t in tasks)
    open_tasks = [t for t in tasks if t.status != "done"]
    return {
        "total": len(tasks),
        "by_status": {status: counts.get(status, 0) for status in STATUSES},
        "overdue": sum(1 for t in tasks if t.is_overdue),
        "all_done": bool(tasks) and all(t.status == "done" for t in tasks),
        "highest_open_priority": max((t.priority for t in open_tasks), default=None),
    }
```

This is Step 3 in action: a `Counter` (a counting `Map`), a dict comprehension, `filter` as a comprehension, `sum` with a generator for `.reduce`, `all` for `.every`, and `max(..., default=None)` because `max` of an empty sequence raises. Note `bool(tasks) and all(...)`: `all([])` is `True`, so an empty project would be reported as "all done" without that guard.

Create `taskboard/tasks/tests/test_services.py`:

```py
from types import SimpleNamespace

from tasks.services import summarize


def task(status, priority=2, overdue=False):
    return SimpleNamespace(status=status, priority=priority, is_overdue=overdue)


def test_summarize_counts_and_flags():
    result = summarize([task("todo", 3), task("doing", 1, overdue=True), task("done", 3)])
    assert result == {
        "total": 3,
        "by_status": {"todo": 1, "doing": 1, "done": 1},
        "overdue": 1,
        "all_done": False,
        "highest_open_priority": 3,
    }


def test_summarize_empty_is_not_all_done():
    # all([]) is True in Python (like [].every(...) in JS), so we guard it explicitly.
    result = summarize([])
    assert result["all_done"] is False
    assert result["highest_open_priority"] is None


def test_summarize_all_done():
    assert summarize([task("done"), task("done")])["all_done"] is True
```

Run it:

```bash
uv run pytest tasks/tests/test_services.py -q
```

Expected: `3 passed`. `SimpleNamespace` is a quick stand-in object (JS: a plain object literal). No database, no server.

## Step 8: Serializers

A **serializer** validates incoming JSON and turns model objects back into JSON. Think Zod schema plus response DTO in one class. Create `taskboard/tasks/serializers.py`:

```py
from django.contrib.auth import get_user_model
from django.contrib.auth.password_validation import validate_password
from rest_framework import serializers

from .models import Project, Task

User = get_user_model()


class ProjectSerializer(serializers.ModelSerializer):
    # Filled by .annotate(task_count=...) in the view; 0 when the instance was just created.
    task_count = serializers.IntegerField(read_only=True, default=0)

    class Meta:
        model = Project
        fields = ["id", "name", "created_at", "task_count"]
        read_only_fields = ["created_at"]

    def validate_name(self, value: str) -> str:
        owner = self.context["request"].user
        clash = Project.objects.filter(owner=owner, name__iexact=value)
        if self.instance:
            clash = clash.exclude(pk=self.instance.pk)
        if clash.exists():
            raise serializers.ValidationError("You already have a project with this name.")
        return value


class TaskSerializer(serializers.ModelSerializer):
    is_overdue = serializers.BooleanField(read_only=True)

    class Meta:
        model = Task
        fields = [
            "id", "project", "title", "status", "priority",
            "due_date", "completed_at", "is_overdue", "created_at",
        ]
        read_only_fields = ["completed_at", "created_at"]

    def get_fields(self):
        fields = super().get_fields()
        request = self.context.get("request")
        if request is not None:  # only the caller's own projects are valid choices
            fields["project"].queryset = Project.objects.filter(owner=request.user)
        return fields


class RegisterSerializer(serializers.Serializer):
    username = serializers.CharField(max_length=150)
    password = serializers.CharField(write_only=True)

    def validate_username(self, value: str) -> str:
        if User.objects.filter(username__iexact=value).exists():
            raise serializers.ValidationError("That username is taken.")
        return value

    def validate_password(self, value: str) -> str:
        validate_password(value)  # Django's configured password rules
        return value

    def create(self, validated_data):
        return User.objects.create_user(**validated_data)
```

What to notice:

- `ModelSerializer` generates fields from the model, so `Meta.fields` is a whitelist of what the API exposes. Fields not listed never leak.
- A method named `validate_<field>` runs automatically for that field. Raise `serializers.ValidationError` and DRF returns a `400` with the message under that field name.
- **Security by construction:** `get_fields` restricts the `project` choices to the caller's own projects. Posting a task into someone else's project fails validation as "object does not exist", so you do not even reveal that it exists.
- `**validated_data` unpacks a dict into keyword arguments, the same as `{...obj}` spreading into function arguments in JS.

## Step 9: Filters

Create `taskboard/tasks/filters.py`. This turns query strings like `?status=done&overdue=true&priority__gte=2` into database filters:

```py
from datetime import date

import django_filters

from .models import Project, Task


def _own_projects(request):
    return Project.objects.filter(owner=request.user)


class TaskFilter(django_filters.FilterSet):
    project = django_filters.ModelChoiceFilter(queryset=_own_projects)
    overdue = django_filters.BooleanFilter(method="filter_overdue")

    class Meta:
        model = Task
        fields = {"status": ["exact"], "priority": ["exact", "gte"]}

    def filter_overdue(self, queryset, name, value):
        overdue = queryset.filter(due_date__lt=date.today()).exclude(status=Task.Status.DONE)
        return overdue if value else queryset.exclude(pk__in=overdue.values("pk"))
```

`ModelChoiceFilter(queryset=_own_projects)` takes a function, so `?project=` can only name the caller's own projects. `overdue` is a custom filter whose logic lives in `filter_overdue`.

## Step 10: Views and URLs

A **viewset** bundles list, retrieve, create, update, partial update and delete for one resource into a single class, which is why an entire CRUD API fits in a few lines. Create `taskboard/tasks/views.py`:

```py
from django.db.models import Count
from rest_framework import status, viewsets
from rest_framework.authtoken.models import Token
from rest_framework.decorators import action
from rest_framework.permissions import AllowAny
from rest_framework.response import Response
from rest_framework.views import APIView

from . import services
from .filters import TaskFilter
from .models import Project, Task
from .serializers import ProjectSerializer, RegisterSerializer, TaskSerializer


class RegisterView(APIView):
    authentication_classes = []
    permission_classes = [AllowAny]

    def post(self, request):
        serializer = RegisterSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        user = serializer.save()
        token, _ = Token.objects.get_or_create(user=user)
        return Response({"token": token.key}, status=status.HTTP_201_CREATED)


class ProjectViewSet(viewsets.ModelViewSet):
    serializer_class = ProjectSerializer
    search_fields = ["name"]
    ordering_fields = ["name", "created_at"]

    def get_queryset(self):
        return (
            Project.objects.filter(owner=self.request.user)
            .annotate(task_count=Count("tasks"))
            .order_by("name")  # annotate() ignores Meta.ordering, so say it here
        )

    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)

    @action(detail=True, methods=["get"])
    def summary(self, request, pk=None):
        project = self.get_object()
        return Response(services.summarize(project.tasks.all()))


class TaskViewSet(viewsets.ModelViewSet):
    serializer_class = TaskSerializer
    filterset_class = TaskFilter
    search_fields = ["title"]
    ordering_fields = ["due_date", "priority", "created_at"]

    def get_queryset(self):
        return Task.objects.filter(project__owner=self.request.user).select_related("project")

    @action(detail=True, methods=["post"])
    def complete(self, request, pk=None):
        task = self.get_object()
        task.status = Task.Status.DONE
        task.save()
        return Response(self.get_serializer(task).data)
```

What to notice:

- **`get_queryset` is the authorization boundary.** Both viewsets only ever see rows that belong to `request.user`. Every detail route (`/projects/5/`) goes through it, so another user's id returns `404`, not `403`.
- `@action` adds extra routes to a viewset: `GET /projects/{id}/summary/` and `POST /tasks/{id}/complete/`.
- `annotate(task_count=Count("tasks"))` computes the count in SQL for the whole list in one query. `select_related("project")` avoids an extra query per task (the N+1 problem you know from ORMs in JS).
- The class attributes `search_fields` and `ordering_fields` enable `?search=` and `?ordering=` with no extra code.
- The view calls `services.summarize` for the logic. It only handles HTTP.

Create `taskboard/tasks/urls.py`:

```py
from django.urls import path
from rest_framework.authtoken.views import obtain_auth_token
from rest_framework.routers import DefaultRouter

from .views import ProjectViewSet, RegisterView, TaskViewSet

router = DefaultRouter()
router.register("projects", ProjectViewSet, basename="project")
router.register("tasks", TaskViewSet, basename="task")

urlpatterns = [
    path("auth/register/", RegisterView.as_view()),
    path("auth/token/", obtain_auth_token),
    *router.urls,
]
```

The router generates all the standard routes from the viewsets (`/projects/`, `/projects/{id}/`, `/tasks/`, ...). `*router.urls` is the spread operator again. The full route table:

| Method and path | What it does |
| --- | --- |
| `POST /api/auth/register/` | Create a user, returns `{"token": ...}` |
| `POST /api/auth/token/` | Log in with username and password, returns a token |
| `GET/POST /api/projects/` | List (search, order, paginate) or create |
| `GET/PATCH/DELETE /api/projects/{id}/` | Read, update, delete |
| `GET /api/projects/{id}/summary/` | Counts by status, overdue, all-done flag |
| `GET/POST /api/tasks/` | List (filter, search, order, paginate) or create |
| `GET/PATCH/DELETE /api/tasks/{id}/` | Read, update, delete |
| `POST /api/tasks/{id}/complete/` | Mark a task done |

## Step 11: Admin

Django ships an admin UI for your models. Create `taskboard/tasks/admin.py` (replace the generated file if it exists):

```py
from django.contrib import admin

from .models import Project, Task


@admin.register(Project)
class ProjectAdmin(admin.ModelAdmin):
    list_display = ["name", "owner", "created_at"]


@admin.register(Task)
class TaskAdmin(admin.ModelAdmin):
    list_display = ["title", "project", "status", "priority", "due_date"]
    list_filter = ["status", "priority"]
```

To use it, create an admin user and start the server (Step 12), then open `http://127.0.0.1:8000/admin/`:

```bash
uv run --env-file .env python manage.py createsuperuser
```

## Step 12: Run it and call it

Start the dev server (it reloads when you save a file, like nodemon):

```bash
uv run --env-file .env python manage.py runserver
```

Expected: `Starting development server at http://127.0.0.1:8000/`. In a **second terminal**, in the `taskboard` folder, register a user. The response contains your token:

```bash
curl -s -X POST http://127.0.0.1:8000/api/auth/register/ \
  -H 'Content-Type: application/json' \
  -d '{"username":"demo","password":"s3cret-pass-99"}'
```

Expected: `{"token":"<40 hex characters>"}`. Save it in a shell variable and use it as a header, like a Bearer token in JS (DRF's word is `Token`):

```bash
TOKEN=<paste the token here>
API=http://127.0.0.1:8000/api
AUTH="Authorization: Token $TOKEN"
```

Create a project, then a task that is already overdue:

```bash
curl -s -X POST $API/projects/ -H "$AUTH" -H 'Content-Type: application/json' -d '{"name":"Launch"}'
curl -s -X POST $API/tasks/ -H "$AUTH" -H 'Content-Type: application/json' \
  -d '{"project":1,"title":"Write docs","priority":3,"due_date":"2020-01-01"}'
```

Expected: the project with `"task_count":0` and the task with `"status":"todo"`, `"is_overdue":true` and `"completed_at":null` (ids and timestamps will differ). Query it:

```bash
curl -s "$API/tasks/?overdue=true" -H "$AUTH"
curl -s "$API/tasks/?search=docs&ordering=-priority" -H "$AUTH"
curl -s -X POST $API/tasks/1/complete/ -H "$AUTH"
curl -s $API/projects/1/summary/ -H "$AUTH"
curl -s -o /dev/null -w "%{http_code}\n" $API/projects/
```

Expected, in order: a paginated list (`{"count":1,"next":null,"previous":null,"results":[...]}`), the same list matched by search, the task now `"status":"done"` with a `completed_at` timestamp, then `{"total":1,"by_status":{"todo":0,"doing":0,"done":1},"overdue":0,"all_done":true,"highest_open_priority":null}`, and finally `401` because the last request had no token.

Try sending bad input to see validation errors: `-d '{"project":1,"title":"","status":"nope"}'` to `/api/tasks/` returns a `400` with a message per bad field.

Stop the server with Ctrl+C. To reset your data, delete `db.sqlite3` and run `migrate` again.

## Step 13: Tests for the API

The tests drive the real API through an in-process client, so no server needs to be running and each test gets a fresh throwaway database. Create `taskboard/tasks/tests/test_api.py`:

```py
from datetime import date, timedelta

import pytest
from rest_framework.test import APIClient

pytestmark = pytest.mark.django_db


def make_client(username="ada"):
    client = APIClient()
    res = client.post("/api/auth/register/", {"username": username, "password": "s3cret-pass-99"})
    assert res.status_code == 201
    client.credentials(HTTP_AUTHORIZATION=f"Token {res.data['token']}")
    return client


@pytest.fixture
def client():
    return make_client()


@pytest.fixture
def project(client):
    return client.post("/api/projects/", {"name": "Launch"}).data


def new_task(client, project, **fields):
    res = client.post("/api/tasks/", {"project": project["id"], "title": "T", **fields}, format="json")
    assert res.status_code == 201, res.data
    return res.data


def test_auth_is_required():
    assert APIClient().get("/api/projects/").status_code == 401


def test_register_rejects_weak_password_and_duplicate_username(client):
    anon = APIClient()
    assert anon.post("/api/auth/register/", {"username": "bob", "password": "123"}).status_code == 400
    assert anon.post("/api/auth/register/", {"username": "ADA", "password": "s3cret-pass-99"}).status_code == 400


def test_login_returns_a_working_token(client):
    anon = APIClient()
    res = anon.post("/api/auth/token/", {"username": "ada", "password": "s3cret-pass-99"})
    assert res.status_code == 200 and res.data["token"]


def test_project_crud_and_task_count(client, project):
    assert project["task_count"] == 0
    new_task(client, project)
    new_task(client, project)
    listing = client.get("/api/projects/").data
    assert listing["count"] == 1 and listing["results"][0]["task_count"] == 2
    assert client.patch(f"/api/projects/{project['id']}/", {"name": "Ship"}).data["name"] == "Ship"
    assert client.delete(f"/api/projects/{project['id']}/").status_code == 204


def test_duplicate_project_name_is_a_400_not_a_500(client, project):
    res = client.post("/api/projects/", {"name": "launch"})
    assert res.status_code == 400 and "name" in res.data


def test_users_cannot_see_or_use_each_others_data(client, project):
    task = new_task(client, project)
    eve = make_client("eve")
    assert eve.get("/api/projects/").data["count"] == 0
    assert eve.get(f"/api/projects/{project['id']}/").status_code == 404
    assert eve.get(f"/api/tasks/{task['id']}/").status_code == 404
    res = eve.post("/api/tasks/", {"project": project["id"], "title": "sneaky"}, format="json")
    assert res.status_code == 400 and "project" in res.data
    assert eve.get(f"/api/tasks/?project={project['id']}").status_code == 400


def test_task_validation_errors(client, project):
    res = client.post("/api/tasks/", {"project": project["id"], "title": "", "status": "nope"}, format="json")
    assert res.status_code == 400
    assert {"title", "status"} <= set(res.data)


def test_complete_action_sets_completed_at_and_reopen_clears_it(client, project):
    task = new_task(client, project)
    done = client.post(f"/api/tasks/{task['id']}/complete/").data
    assert done["status"] == "done" and done["completed_at"]
    reopened = client.patch(f"/api/tasks/{task['id']}/", {"status": "doing"}, format="json").data
    assert reopened["completed_at"] is None


def test_filters_search_and_ordering(client, project):
    yesterday = (date.today() - timedelta(days=1)).isoformat()
    new_task(client, project, title="Write docs", priority=1)
    new_task(client, project, title="Fix bug", priority=3, due_date=yesterday)
    finished = new_task(client, project, title="Old thing", priority=2, due_date=yesterday)
    client.post(f"/api/tasks/{finished['id']}/complete/")

    def titles(query):
        return [t["title"] for t in client.get(f"/api/tasks/?{query}").data["results"]]

    assert titles("overdue=true") == ["Fix bug"]  # the done task is not overdue
    assert sorted(titles("overdue=false")) == ["Old thing", "Write docs"]
    assert titles("status=done") == ["Old thing"]
    assert sorted(titles("priority__gte=2")) == ["Fix bug", "Old thing"]
    assert titles("search=docs") == ["Write docs"]
    assert titles("ordering=-priority") == ["Fix bug", "Old thing", "Write docs"]
    assert titles(f"project={project['id']}&status=todo&ordering=priority") == ["Write docs", "Fix bug"]


def test_pagination(client, project):
    for i in range(25):
        new_task(client, project, title=f"t{i}")
    page1 = client.get("/api/tasks/").data
    assert page1["count"] == 25 and len(page1["results"]) == 20 and page1["next"]
    assert len(client.get("/api/tasks/?page=2").data["results"]) == 5


def test_project_summary_endpoint(client, project):
    yesterday = (date.today() - timedelta(days=1)).isoformat()
    new_task(client, project, priority=3, due_date=yesterday)
    done = new_task(client, project, priority=1)
    client.post(f"/api/tasks/{done['id']}/complete/")
    assert client.get(f"/api/projects/{project['id']}/summary/").data == {
        "total": 2,
        "by_status": {"todo": 1, "doing": 0, "done": 1},
        "overdue": 1,
        "all_done": False,
        "highest_open_priority": 3,
    }
```

What to notice:

- **Fixtures** (`@pytest.fixture`) are pytest's `beforeEach`. A test asks for what it needs by naming it as a parameter: `def test_x(client, project)`. pytest injects them. There is no `describe` or `it`; each test is just a function.
- `pytestmark = pytest.mark.django_db` allows database access, and every test runs in a rolled-back transaction, so tests never leak data into each other.
- Notice the security tests: another user cannot read, modify or even reference your data.

Run everything:

```bash
uv run pytest -q
uv run ruff check .
```

Expected: `29 passed` (15 lessons, 3 services, 11 API) and `All checks passed!`. Useful flags: `uv run pytest -k filters` runs matching tests only, `uv run pytest -x` stops at the first failure, `uv run pytest -vv` shows full diffs. `uv run ruff format .` is Prettier: it rewrites files to a consistent style.

## Step 14: Production notes

The app above is a solid foundation. Before real users, work through this list. Items marked (not tested here) are standard Django practice that I did not run in this build.

- **Settings via environment.** Set `DJANGO_DEBUG=0`, a long random `DJANGO_SECRET_KEY` and your real `DJANGO_ALLOWED_HOSTS`. The settings file already refuses to start without a secret key when debug is off. Generate one with `uv run python -c "from django.core.management.utils import get_random_secret_key as g; print(g())"`.
- **Real database.** SQLite is fine for learning. Use PostgreSQL in production (not tested here): add a driver with `uv add "psycopg[binary]"` and change `DATABASES` in `settings.py`.
- **A production server.** `runserver` is for development only. Use gunicorn (`uv add gunicorn`, then `uv run gunicorn config.wsgi`) or an ASGI server (not tested here).
- **Rate limiting.** DRF has built-in throttles. Add `DEFAULT_THROTTLE_CLASSES` and `DEFAULT_THROTTLE_RATES` to `REST_FRAMEWORK`, especially on the register and token endpoints.
- **A JavaScript frontend on another origin** needs CORS: `uv add django-cors-headers` (not tested here).
- **API docs.** `drf-spectacular` generates an OpenAPI schema you can feed to Swagger UI or a TypeScript client generator (not tested here).
- **Types.** Add `uv add --dev mypy` or pyright if you want TypeScript-style checking of the type hints.
- **Alternatives to know about.** FastAPI (async-first, type-hint driven, closest to Express plus Zod) and Django Ninja (Django plus FastAPI-style typing). DRF is the long-established default.
- **CI.** Your gate is `uv run ruff check . && uv run pytest`. Commit `uv.lock` so CI installs exactly what you tested.

## JavaScript to Python cheat sheet

- [ ] Indentation, not braces. Colons open blocks.
- [ ] `snake_case` names, `PascalCase` classes.
- [ ] Empty list, dict, string and `0` are falsy. Use `is None` to test for none.
- [ ] `==` compares values. `is` is identity, for `None` only.
- [ ] `d["key"]` raises if missing. `d.get("key")` returns `None`.
- [ ] Comprehensions replace `.map` and `.filter`. `all` and `any` replace `.every` and `.some`.
- [ ] `sorted(x)` returns a new list. `x.sort()` mutates and returns `None`.
- [ ] Never use a list or dict as a default argument. Use `None`.
- [ ] Methods take `self` explicitly. Create objects without `new`.
- [ ] Type hints are not enforced at runtime.
- [ ] `import` pulls names from modules: `from tasks.services import summarize`. There is no default export.
- [ ] Raise and catch exceptions with `raise` and `try` / `except`.
- [ ] Always run things with `uv run` so the project's environment is used.

> [!TIP]
> **Verified:** every file above was built and run on Python 3.13.12 with Django 6.1.3, djangorestframework 3.18.3, django-filter 26.2, pytest 9.2.0, pytest-django 4.14.0 and ruff 0.17.4: lint clean, 29 tests passing, migrations generated, and the live `runserver` was exercised with curl (register, create, filter, complete, summary, 401 without a token). The project-creation commands in Step 2 were run from an empty folder.  
> **Not verified:** the Django admin in a browser (the page is generated by Django and the registration code is standard), the interactive shell session in Step 6, and everything marked "not tested here" in Step 14 (PostgreSQL, gunicorn, CORS, throttling, drf-spectacular).
