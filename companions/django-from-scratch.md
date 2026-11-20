# Django from Scratch (Karpathy-Style): Build a Web Framework in 400 Lines, Then Read the Real One

> [!NOTE]
> Django feels like magic until you have built a tiny one yourself. Starting from a blank file, you will write a working web framework in about 390 lines of standard-library Python (request and response objects, a router, validation, an ORM, middleware and a WSGI entry point), use it to build a real task API with auth, and prove it with tests. Then you will open the real Django source and see the same ideas at full size. Every file below was built and run: lint clean, 22 tests passing, and the live server was exercised with curl. No third-party runtime dependencies.


## The style of this tutorial

This follows a teaching pattern: pick one concept you use but cannot explain, rebuild a minimal version, run it, then compare it with the real thing.

- **Start from a blank file** and add one idea at a time.
- **Small, readable, runnable code.** No framework hiding the point. Every step ends with something you run.
- **Tests are the proof.** A claim like "queries are lazy" or "this is safe from SQL injection" has a test that fails if the claim is false.
- **Reveal at the end.** Only after you built it do you read how Django does it, and you will recognize every piece.

You should know a little Python. If you are coming from JavaScript, the companion tutorial "Python for JavaScript Developers: Build a Django REST API" teaches the language; here, short callouts explain each Python feature the first time it appears.

## What you will build

| Step | You build |
| --- | --- |
| 0 | Install the tools |
| 1 | A web framework in 6 lines (the one idea) |
| 2 | Create the project |
| 3 | `Request`, `Response`, `HttpError` |
| 4 | A router with `<int:id>` patterns |
| 5 | Declarative validation (what serializers and forms do) |
| 6 | An ORM: models, lazy querysets, safe SQL |
| 7 | The app: middleware and the WSGI entry point |
| 8 | A real API on top: users, tokens, tasks, filters, pagination |
| 9 | Run it and call it with curl |
| 10 | Tests that prove each claim |
| 11 | The reveal: your pieces versus Django's |
| 12 | Exercises: break it, extend it, port it |
| 13 | Teaching kit: turn this into a video and a post |

## How to read this tutorial

- **Every path is relative to the ****`nanoweb/`**** folder (the project root)** you create in Step 2. Every command runs from there unless it says otherwise. Each file is introduced with its full path starting at `nanoweb/`.
- Versions used: Python 3.13, pytest 9.2, ruff 0.17. The framework itself uses only the standard library. It needs Python 3.12 or newer.
- Code comments show where each piece maps onto Express or Django, so you can read it either way.

## Step 0: Install everything

You need **uv** (installs Python and manages dependencies) and **Python 3.13**. Do each sub-step in order and run its check.

### 0.1 Open a terminal

- macOS: Terminal (or iTerm)
- Windows: PowerShell. For the curl commands in Step 9 use WSL or Git Bash, or rely on the tests in Step 10.
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

## Step 1: A web framework in six lines

Before building anything, look at the one idea underneath Django, Flask, FastAPI and every other Python web framework. **A web app is a function that takes a request and returns a response.** Python standardized the shape of that function in a spec called **WSGI**. A server (wsgiref, gunicorn, uwsgi) calls your function once per HTTP request, passing in a dict describing the request (`environ`) and a callback for starting the response (`start_response`).

In JavaScript terms: `(req, res) => ...` in Node's `http.createServer`, except the function returns the body instead of writing it.

Create the project first (the next step explains the commands), then create `nanoweb/hello_wsgi.py`:

```py
"""The whole idea of a Python web framework: a function from request to response."""
from wsgiref.simple_server import make_server


def app(environ, start_response):
    start_response("200 OK", [("Content-Type", "text/plain")])
    return [f"{environ['REQUEST_METHOD']} {environ['PATH_INFO']}\n".encode()]


if __name__ == "__main__":
    print("http://127.0.0.1:8800")
    make_server("127.0.0.1", 8800, app).serve_forever()
```

You will run it in Step 2.4. The rest of this tutorial is one question: *what do you need to put around that function to build a real API?* Answer: parse the request into an object, route the path to the right handler, validate the input, talk to a database, and wrap it all in middleware for cross-cutting things like authentication. That is exactly the list of files we will write.

```txt
browser/curl  ->  server (wsgiref)  ->  App.__call__(environ, start_response)
                                          |
                         Request.from_environ(environ)
                                          |
                         Router.resolve(method, path)  ->  handler + path params
                                          |
                         middleware (logging, auth)  ->  handler(request)
                                          |            Schema.validate(...)  QuerySet / Model
                         Response.render() -> status, headers, body bytes
```

## Step 2: Create the project

### 2.1 Project and dependencies

```bash
mkdir nanoweb && cd nanoweb
uv init --bare --name nanoweb --python 3.13
uv add --dev pytest ruff
mkdir -p nano board tests
touch nano/__init__.py board/__init__.py tests/__init__.py
```

There are **no runtime dependencies**, only dev tools. `nano/` will be the framework and `board/` the application built with it. `__init__.py` marks a folder as an importable package.

### 2.2 Tooling config

Append these sections to the end of `nanoweb/pyproject.toml` (open it in your editor and paste at the bottom):

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
pythonpath = ["."]

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B"]
ignore = ["E501"]
```

### 2.3 Settings file

Create two files in the project root, next to `pyproject.toml`: `nanoweb/.env` holds your settings and `nanoweb/.gitignore` keeps secrets and build output out of git. The commands create both (you are already inside `nanoweb/`):

```bash
printf 'PORT=8800\nDB_PATH=board.sqlite3\n' > .env
printf '.env\n.venv/\n__pycache__/\n*.sqlite3\n.pytest_cache/\n.ruff_cache/\n' > .gitignore
```

On Windows, create both files in your editor, saved in the `nanoweb` folder, with the same contents.

### 2.4 Run the six-line framework

```bash
uv run python hello_wsgi.py
```

In a second terminal:

```bash
curl -s http://127.0.0.1:8800/tasks/7
curl -s -X POST http://127.0.0.1:8800/hello
```

Expected: `GET /tasks/7` and `POST /hello`. You have a web server. Stop it with Ctrl+C.

When we are done the project looks like this:

```txt
nanoweb/
├── .env
├── .gitignore
├── pyproject.toml
├── hello_wsgi.py
├── nano/                  # the framework (standard library only)
│   ├── web.py             # Request, Response, HttpError
│   ├── router.py
│   ├── schema.py          # validation
│   ├── orm.py
│   ├── app.py             # middleware + WSGI entry point
│   └── testing.py
├── board/                 # the application built with the framework
│   ├── models.py
│   ├── auth.py
│   ├── services.py
│   ├── api.py
│   ├── app.py
│   └── run.py
└── tests/
    ├── conftest.py
    ├── test_framework.py
    └── test_api.py
```

## Step 3: Request, Response, HttpError

The raw `environ` dict is awkward. Wrap it in a `Request` object, and give handlers a `Response` plus a way to bail out with an error. Create `nanoweb/nano/web.py`:

```py
"""Request, Response and HttpError: the vocabulary of a web framework."""
import json
from dataclasses import dataclass, field
from http import HTTPStatus
from urllib.parse import parse_qsl


class HttpError(Exception):
    """Raise anywhere in a handler to stop and send this response (like Django's Http404)."""

    def __init__(self, status: int, detail):
        super().__init__(str(detail))
        self.status = status
        self.detail = detail


@dataclass
class Request:
    method: str
    path: str
    query: dict = field(default_factory=dict)
    headers: dict = field(default_factory=dict)  # keys are lower-case
    body: bytes = b""
    params: dict = field(default_factory=dict)  # filled by the router: /tasks/<int:id>/
    user: object = None  # filled by the auth middleware

    def json(self) -> dict:
        if not self.body:
            return {}
        try:
            data = json.loads(self.body)
        except json.JSONDecodeError:
            raise HttpError(400, "Body is not valid JSON.") from None
        if not isinstance(data, dict):
            raise HttpError(400, "JSON body must be an object.")
        return data

    @classmethod
    def from_environ(cls, environ: dict) -> "Request":
        """Build a Request from a WSGI environ: the dict every Python web server hands you."""
        length = int(environ.get("CONTENT_LENGTH") or 0)
        headers = {
            key[5:].replace("_", "-").lower(): value
            for key, value in environ.items()
            if key.startswith("HTTP_")
        }
        return cls(
            method=environ["REQUEST_METHOD"],
            path=environ.get("PATH_INFO", "/"),
            query=dict(parse_qsl(environ.get("QUERY_STRING", ""))),
            headers=headers,
            body=environ["wsgi.input"].read(length) if length else b"",
        )


@dataclass
class Response:
    data: object = None
    status: int = 200
    headers: dict = field(default_factory=dict)

    def render(self) -> tuple[str, list[tuple[str, str]], bytes]:
        """Turn this into the three things WSGI wants: status line, headers, body bytes."""
        body = b"" if self.data is None else json.dumps(self.data).encode()
        headers = {"Content-Type": "application/json", "Content-Length": str(len(body)), **self.headers}
        return f"{self.status} {HTTPStatus(self.status).phrase}", list(headers.items()), body
```

> [!WARNING]
> **Python for JS developers.** `@dataclass` writes `__init__`, `__repr__` and `__eq__` for you from the annotated fields (like a TypeScript class with a generated constructor). `field(default_factory=dict)` is how you default to a fresh `{}` per instance. A plain `= {}` would share one dict between all instances. `@classmethod` is a static factory that receives the class as `cls`. `raise ... from None` hides the original traceback, so a client mistake does not look like a server bug.

How HTTP maps onto this: the request line (`POST /tasks/?page=2`) became `method`, `path` and `query`. Headers arrive in the environ as `HTTP_AUTHORIZATION`, and we turn them back into `authorization`. The body is a stream we read `CONTENT_LENGTH` bytes from.

## Step 4: The router

A router answers: which function handles `GET /tasks/42/`, and what is `42`? Patterns like `/tasks/<int:id>/` become regular expressions with named groups. Create `nanoweb/nano/router.py`:

```py
"""Map (method, path) to a handler. Patterns look like /tasks/<int:id>/."""
import re

from .web import HttpError

CONVERTERS = {"int": (r"\d+", int), "str": (r"[^/]+", str)}


class Router:
    def __init__(self):
        self._routes = []  # (method, compiled regex, converters, handler)

    def add(self, method: str, pattern: str, handler) -> None:
        converters = {}

        def to_regex(match):  # a closure: it fills `converters` while re.sub runs
            kind, name = match.groups()
            regex, convert = CONVERTERS[kind]
            converters[name] = convert
            return f"(?P<{name}>{regex})"

        regex = re.compile("^" + re.sub(r"<(\w+):(\w+)>", to_regex, pattern) + "$")
        self._routes.append((method, regex, converters, handler))

    def route(self, method: str, pattern: str):
        """Decorator form, like Express: @router.route("GET", "/tasks/")."""

        def decorator(handler):
            self.add(method, pattern, handler)
            return handler

        return decorator

    def get(self, pattern):
        return self.route("GET", pattern)

    def post(self, pattern):
        return self.route("POST", pattern)

    def patch(self, pattern):
        return self.route("PATCH", pattern)

    def delete(self, pattern):
        return self.route("DELETE", pattern)

    def resolve(self, method: str, path: str):
        path_matched = False
        for route_method, regex, converters, handler in self._routes:
            match = regex.match(path)
            if not match:
                continue
            path_matched = True
            if route_method == method:
                params = {name: converters[name](value) for name, value in match.groupdict().items()}
                return handler, params
        if path_matched:
            raise HttpError(405, "Method not allowed.")
        raise HttpError(404, "Not found.")
```

> [!WARNING]
> **Python for JS developers.** A **decorator** (`@router.get(...)`) is a function that receives your function and returns a replacement (or the same one). It is the same pattern as JavaScript higher-order functions, with nicer syntax. `@router.get("/x")` means `handler = router.get("/x")(handler)`. The dict comprehension in `resolve` is `Object.fromEntries(Object.entries(...).map(...))`.

The difference between `404` and `405` is one boolean: did the path match anything at all? Django's `django/urls/resolvers.py` does this with the same regex idea and more features.

## Step 5: Validation

Never trust request input. A **schema** declares the shape you accept, and `validate` returns cleaned data or every error at once. This is what Django forms and DRF serializers do. Create `nanoweb/nano/schema.py`:

```py
"""Declarative input validation: the idea behind Django forms and DRF serializers."""
from datetime import date

from .web import HttpError


class Field:
    def __init__(self, required=True, default=None, nullable=False):
        self.required, self.default, self.nullable = required, default, nullable

    def clean(self, value):
        """Return the cleaned value or raise ValueError(message)."""
        return value


class Str(Field):
    def __init__(self, max_length=255, min_length=1, strip=True, **kwargs):
        super().__init__(**kwargs)
        self.max_length, self.min_length, self.strip = max_length, min_length, strip

    def clean(self, value):
        if not isinstance(value, str):
            raise ValueError("Must be a string.")
        if self.strip:
            value = value.strip()
        if len(value) < self.min_length:
            raise ValueError("This field may not be blank." if self.min_length == 1 else
                             f"Ensure this field has at least {self.min_length} characters.")
        if len(value) > self.max_length:
            raise ValueError(f"Ensure this field has no more than {self.max_length} characters.")
        return value


class Int(Field):
    def __init__(self, min=None, max=None, **kwargs):
        super().__init__(**kwargs)
        self.min, self.max = min, max

    def clean(self, value):
        if isinstance(value, bool) or not isinstance(value, int):  # True is an int in Python!
            raise ValueError("Must be an integer.")
        if self.min is not None and value < self.min:
            raise ValueError(f"Must be at least {self.min}.")
        if self.max is not None and value > self.max:
            raise ValueError(f"Must be at most {self.max}.")
        return value


class Choice(Field):
    def __init__(self, choices, **kwargs):
        super().__init__(**kwargs)
        self.choices = list(choices)

    def clean(self, value):
        if value not in self.choices:
            raise ValueError(f"Must be one of: {', '.join(self.choices)}.")
        return value


class IsoDate(Field):
    def clean(self, value):
        try:
            return date.fromisoformat(value).isoformat()
        except (TypeError, ValueError):
            raise ValueError("Must be a date like 2026-12-31.") from None


class Schema:
    """Subclass and add Field attributes. __init_subclass__ collects them."""

    _fields: dict

    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
        cls._fields = {name: f for name, f in vars(cls).items() if isinstance(f, Field)}

    @classmethod
    def validate(cls, data: dict, partial: bool = False) -> dict:
        """Return cleaned data, or raise HttpError(400, {field: [messages]})."""
        clean, errors = {}, {}
        for name, field in cls._fields.items():
            if name not in data:
                if field.required and not partial:
                    errors[name] = ["This field is required."]
                elif not field.required and not partial:
                    clean[name] = field.default
                continue
            value = data[name]
            if value is None:
                if field.nullable:
                    clean[name] = None
                else:
                    errors[name] = ["This field may not be null."]
                continue
            try:
                clean[name] = field.clean(value)
            except ValueError as err:
                errors[name] = [str(err)]
        if errors:
            raise HttpError(400, errors)
        return clean
```

> [!WARNING]
> **Python for JS developers.** Here is the trick behind every Django model and serializer. `__init_subclass__` runs the moment a subclass is defined. We use it to scan the class body with `vars(cls)` and collect the `Field` attributes, so you can declare `title = Str(max_length=200)` and the framework discovers it. JavaScript needs decorators or a schema builder call for this. Also: **`True`**** is an ****`int`**** in Python**, which is why `Int.clean` rejects booleans explicitly.

## Step 6: An ORM

The biggest piece, and the most instructive. An ORM turns classes into tables and method calls into SQL. We will build models, fields, and a **lazy, chainable QuerySet** on top of the standard library's `sqlite3`. Create `nanoweb/nano/orm.py`:

```py
"""A tiny ORM on top of sqlite3: fields, models and lazy, chainable querysets."""
import sqlite3


class Database:
    def __init__(self):
        self.conn = None

    def connect(self, path: str = ":memory:") -> None:
        self.conn = sqlite3.connect(path, check_same_thread=False)
        self.conn.row_factory = sqlite3.Row

    def execute(self, sql: str, params=()):
        cursor = self.conn.execute(sql, params)  # values always travel as parameters, never in the SQL text
        self.conn.commit()
        return cursor

    def create_tables(self, *models) -> None:
        for model in models:
            self.execute(model.create_sql())


db = Database()


class Field:
    sql_type = "TEXT"

    def __init__(self, default=None, null=False, unique=False):
        self.default, self.null, self.unique = default, null, unique

    def __set_name__(self, owner, name):  # Python calls this when the class body defines the attribute
        self.name = name

    def ddl(self) -> str:
        return f"{self.name} {self.sql_type}{'' if self.null else ' NOT NULL'}{' UNIQUE' if self.unique else ''}"


class TextField(Field):
    sql_type = "TEXT"


class IntegerField(Field):
    sql_type = "INTEGER"


LOOKUPS = {"exact": "= ?", "ne": "!= ?", "gt": "> ?", "gte": ">= ?", "lt": "< ?", "lte": "<= ?", "contains": "LIKE ?"}


class QuerySet:
    """Describes a query. Nothing hits the database until you iterate, count or slice-and-iterate."""

    def __init__(self, model, where=(), params=(), order=(), limit=None, offset=0):
        self.model, self.where, self.params = model, tuple(where), tuple(params)
        self.order, self.limit, self.offset = tuple(order), limit, offset

    def _clone(self, **changes) -> "QuerySet":
        state = {"where": self.where, "params": self.params, "order": self.order,
                 "limit": self.limit, "offset": self.offset, **changes}
        return QuerySet(self.model, **state)  # querysets are immutable: every call returns a new one

    def _check_field(self, name: str) -> None:
        # Field names go into the SQL text, so they are checked against the model, never trusted.
        if name != "id" and name not in self.model._fields:
            raise ValueError(f"Unknown field {name!r} for {self.model.__name__}")

    def filter(self, **lookups) -> "QuerySet":
        where, params = list(self.where), list(self.params)
        for key, value in lookups.items():
            name, _, op = key.partition("__")
            op = op or "exact"
            self._check_field(name)
            if op == "isnull":
                where.append(f"{name} IS {'' if value else 'NOT '}NULL")
                continue
            if op not in LOOKUPS:
                raise ValueError(f"Unknown lookup {op!r}")
            where.append(f"{name} {LOOKUPS[op]}")
            params.append(f"%{value}%" if op == "contains" else value)
        return self._clone(where=where, params=params)

    def order_by(self, *names: str) -> "QuerySet":
        parts = []
        for name in names:
            field = name.lstrip("-")
            self._check_field(field)
            parts.append(f"{field} {'DESC' if name.startswith('-') else 'ASC'}")
        return self._clone(order=parts)

    def __getitem__(self, key: slice) -> "QuerySet":  # qs[20:40] -> LIMIT 20 OFFSET 20
        if not isinstance(key, slice) or key.step is not None:
            raise TypeError("Only simple slices like qs[a:b] are supported")
        start = key.start or 0
        limit = None if key.stop is None else key.stop - start
        return self._clone(offset=start, limit=limit)

    def _sql(self, select: str) -> tuple[str, tuple]:
        sql = f"SELECT {select} FROM {self.model._table}"
        if self.where:
            sql += " WHERE " + " AND ".join(self.where)
        if self.order:
            sql += " ORDER BY " + ", ".join(self.order)
        if self.limit is not None or self.offset:
            sql += f" LIMIT {-1 if self.limit is None else int(self.limit)} OFFSET {int(self.offset)}"
        return sql, self.params

    def __iter__(self):
        sql, params = self._sql("*")
        for row in db.execute(sql, params).fetchall():
            yield self.model(**dict(row))

    def count(self) -> int:
        sql, params = self._sql("COUNT(*)")
        return db.execute(sql, params).fetchone()[0]

    def first(self):
        return next(iter(self[:1]), None)

    def get(self, **lookups):
        found = list(self.filter(**lookups)[:2])
        if not found:
            raise self.model.DoesNotExist(f"{self.model.__name__} matching {lookups} does not exist")
        if len(found) > 1:
            raise self.model.MultipleObjectsReturned(f"More than one {self.model.__name__} matches {lookups}")
        return found[0]


class Manager:
    """A descriptor: `Model.objects` hands back a fresh QuerySet (Django's Manager works this way)."""

    def __get__(self, instance, owner) -> QuerySet:
        if instance is not None:
            raise AttributeError("Manager isn't accessible via model instances")
        return QuerySet(owner)


class Model:
    objects = Manager()
    _fields: dict
    _table: str

    class DoesNotExist(Exception):
        pass

    class MultipleObjectsReturned(Exception):
        pass

    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
        cls._fields = {name: f for name, f in vars(cls).items() if isinstance(f, Field)}
        cls._table = cls.__name__.lower()
        cls.DoesNotExist = type("DoesNotExist", (Model.DoesNotExist,), {})  # one per model

    def __init__(self, **values):
        self.id = values.pop("id", None)
        for name, field in self._fields.items():
            setattr(self, name, values.pop(name, field.default))
        if values:
            raise TypeError(f"Unknown fields: {sorted(values)}")

    @classmethod
    def create_sql(cls) -> str:
        columns = ["id INTEGER PRIMARY KEY AUTOINCREMENT", *(f.ddl() for f in cls._fields.values())]
        return f"CREATE TABLE IF NOT EXISTS {cls._table} ({', '.join(columns)})"

    def save(self):
        names = list(self._fields)
        values = [getattr(self, name) for name in names]
        if self.id is None:
            marks = ", ".join(["?"] * len(names))
            cursor = db.execute(f"INSERT INTO {self._table} ({', '.join(names)}) VALUES ({marks})", values)
            self.id = cursor.lastrowid
        else:
            assignments = ", ".join(f"{name} = ?" for name in names)
            db.execute(f"UPDATE {self._table} SET {assignments} WHERE id = ?", [*values, self.id])
        return self

    def delete(self) -> None:
        db.execute(f"DELETE FROM {self._table} WHERE id = ?", [self.id])

    def to_dict(self) -> dict:
        return {"id": self.id, **{name: getattr(self, name) for name in self._fields}}

    def __repr__(self) -> str:
        return f"<{type(self).__name__} {self.to_dict()}>"
```

Four ideas to take from this file:

- **Laziness.** `Task.objects.filter(...).order_by(...)` only builds a description of a query. SQL runs when you iterate (`for t in qs`), count, or fetch. That lets you add filters step by step across functions, exactly like Django.
- **Immutability.** Every method returns a *new* QuerySet via `_clone`, so a base queryset can never be accidentally modified by code that narrows it.
- **SQL safety.** Values are always passed as `?` parameters, never pasted into the SQL string. Field names, which must be pasted, are checked against the model first. The tests in Step 10 attack this on purpose.
- **One query for the whole page**, not one per row. The `LIMIT` and `OFFSET` come from slicing: `qs[20:40]`.

> [!WARNING]
> **Python for JS developers.** `__iter__` with `yield` is a **generator** (like `function*` in JS), so `for task in queryset` just works. `__getitem__` makes `qs[20:40]` slicing work. `Manager` is a **descriptor**: an object with `__get__` that runs when you read `Task.objects`, which is how a class attribute can return something fresh each time. `__set_name__` tells a field its own attribute name. Together with `__init_subclass__`, this is the entire "magic" of Django models.

## Step 7: The app: middleware and WSGI

Now the piece that ties it together. **Middleware** is a function that takes a handler and returns a wrapped handler, so you can run code before and after every request without touching any handler. Create `nanoweb/nano/app.py`:

```py
"""The App: routing, middleware and the WSGI entry point."""
import logging

from .router import Router
from .web import HttpError, Request, Response

log = logging.getLogger("nano")


def request_log(next_handler):
    """A middleware is a function that takes a handler and returns a wrapped handler."""

    def inner(request: Request) -> Response:
        response = next_handler(request)
        log.info("%s %s -> %s", request.method, request.path, response.status)
        return response

    return inner


def _error_response(err: HttpError) -> Response:
    body = err.detail if isinstance(err.detail, dict) else {"detail": err.detail}
    return Response(body, err.status)


class App:
    def __init__(self, router: Router, middleware=()):
        self.router = router
        self.middleware = list(middleware)

    def _safe(self, handler):
        """Turn raised errors into responses, so middleware only ever sees responses."""

        def inner(request: Request) -> Response:
            try:
                result = handler(request)
                return result if isinstance(result, Response) else Response(result)
            except HttpError as err:
                return _error_response(err)
            except Exception:
                log.exception("unhandled error")
                return Response({"detail": "Server error."}, 500)

        return inner

    def handle(self, request: Request) -> Response:
        try:
            handler, request.params = self.router.resolve(request.method, request.path)
        except HttpError as err:
            return _error_response(err)
        call = self._safe(handler)
        for wrap in reversed(self.middleware):  # first in the list ends up outermost
            call = wrap(call)
        return call(request)

    def __call__(self, environ, start_response):
        """WSGI: any Python web server (wsgiref, gunicorn, uwsgi) can run this object."""
        status, headers, body = self.handle(Request.from_environ(environ)).render()
        start_response(status, headers)
        return [body]
```

The middleware chain is built by wrapping from the inside out:

```txt
middleware=[A, B]   ->   A( B( safe(handler) ) )
request goes in:  A in -> B in -> handler
response comes out: handler -> B out -> A out
```

This is the same onion model as Express and Koa middleware. The `__call__` method is what makes an `App` instance itself a valid WSGI application: you can hand it to wsgiref today or gunicorn tomorrow. Two safety properties are built in: any unexpected exception becomes a clean `500` instead of crashing the server, and a handler can return a plain dict (turned into a `200` JSON response) or a full `Response`.

One more small file: a test client that calls the app in-process, with no network and no server. Create `nanoweb/nano/testing.py`:

```py
"""A test client: call the app in-process, no network, no server."""
import json
from urllib.parse import parse_qsl

from .app import App
from .web import Request, Response


class Client:
    def __init__(self, app: App):
        self.app = app
        self.token = None

    def request(self, method: str, url: str, body=None) -> Response:
        path, _, query = url.partition("?")
        headers = {"authorization": f"Token {self.token}"} if self.token else {}
        raw = json.dumps(body).encode() if body is not None else b""
        return self.app.handle(Request(method, path, dict(parse_qsl(query)), headers, raw))

    def get(self, url):
        return self.request("GET", url)

    def post(self, url, body=None):
        return self.request("POST", url, body)

    def patch(self, url, body=None):
        return self.request("PATCH", url, body)

    def delete(self, url):
        return self.request("DELETE", url)
```

The framework is done: six files, about 390 lines of code, and no dependencies.

## Step 8: A real API on top

Now use the framework the way you would use Django. We build a task API with registration, login, per-user tasks, filtering, search, ordering and pagination.

### 8.1 Models

Create `nanoweb/board/models.py`:

```py
from datetime import UTC, date, datetime

from nano.orm import IntegerField, Model, TextField


class User(Model):
    username = TextField(unique=True)
    password_hash = TextField()
    token = TextField(unique=True)


class Task(Model):
    owner_id = IntegerField()
    title = TextField()
    status = TextField(default="todo")
    priority = IntegerField(default=2)
    due_date = TextField(null=True)  # ISO date string, sorts correctly as text
    completed_at = TextField(null=True)
    created_at = TextField()

    @property
    def is_overdue(self) -> bool:
        return bool(self.due_date and self.due_date < date.today().isoformat() and self.status != "done")

    def save(self):
        now = datetime.now(UTC).isoformat()
        self.created_at = self.created_at or now
        # Keep completed_at consistent with status, whoever changes the status.
        if self.status == "done" and self.completed_at is None:
            self.completed_at = now
        elif self.status != "done":
            self.completed_at = None
        return super().save()
```

Compare this with a Django model: the shape is the same, because we copied the idea. Overriding `save()` and calling `super().save()` is the standard way to keep an invariant ("`completed_at` is set exactly when the status is done").

### 8.2 Authentication

Create `nanoweb/board/auth.py`. This is for learning how it works. In real projects use your framework's vetted auth rather than writing your own:

```py
"""Passwords, tokens and the login_required guard."""
import functools
import hashlib
import hmac
import os
import secrets

from nano.web import HttpError, Request

from .models import User

ITERATIONS = 600_000  # PBKDF2 cost. Tests lower it to stay fast.


def hash_password(password: str) -> str:
    salt = os.urandom(16)
    digest = hashlib.pbkdf2_hmac("sha256", password.encode(), salt, ITERATIONS)
    return f"pbkdf2_sha256${ITERATIONS}${salt.hex()}${digest.hex()}"


def verify_password(password: str, stored: str) -> bool:
    _, iterations, salt_hex, digest_hex = stored.split("$")
    digest = hashlib.pbkdf2_hmac("sha256", password.encode(), bytes.fromhex(salt_hex), int(iterations))
    return hmac.compare_digest(digest.hex(), digest_hex)  # constant-time comparison


def new_token() -> str:
    return secrets.token_hex(20)


def token_auth(next_handler):
    """Middleware: read `Authorization: Token <key>` and attach the user to the request."""

    def inner(request: Request):
        header = request.headers.get("authorization", "")
        scheme, _, key = header.partition(" ")
        if scheme == "Token" and key:
            request.user = User.objects.filter(token=key).first()
        return next_handler(request)

    return inner


def login_required(handler):
    @functools.wraps(handler)
    def wrapper(request: Request):
        if request.user is None:
            raise HttpError(401, "Authentication credentials were not provided.")
        return handler(request)

    return wrapper
```

The design: **middleware identifies, the decorator authorizes.** `token_auth` only sets `request.user` (or leaves it `None`). `login_required` is a decorator on each protected handler. Passwords are stored as a salted, slow PBKDF2 hash, never as text, and compared in constant time so timing does not leak information.

### 8.3 Business logic

Plain functions with no HTTP and no database, so they are trivial to test. Create `nanoweb/board/services.py`:

```py
"""Business logic with no HTTP and no database: plain functions, easy to test."""
from collections import Counter

STATUSES = ("todo", "doing", "done")


def summarize(tasks) -> dict:
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

### 8.4 The API

Create `nanoweb/board/api.py`:

```py
"""The HTTP API: routes, validation schemas and handlers."""
from datetime import date

from nano.router import Router
from nano.schema import Choice, Int, IsoDate, Schema, Str
from nano.web import HttpError, Request, Response

from . import services
from .auth import hash_password, login_required, new_token, verify_password
from .models import Task, User

router = Router()


class CredentialsSchema(Schema):
    username = Str(max_length=150)
    password = Str(max_length=128, min_length=8, strip=False)


class TaskSchema(Schema):
    title = Str(max_length=200)
    status = Choice(["todo", "doing", "done"], required=False, default="todo")
    priority = Int(min=1, max=3, required=False, default=2)
    due_date = IsoDate(required=False, nullable=True)


@router.post("/register/")
def register(request: Request):
    data = CredentialsSchema.validate(request.json())
    if User.objects.filter(username=data["username"]).count():
        raise HttpError(400, {"username": ["That username is taken."]})
    user = User(username=data["username"], password_hash=hash_password(data["password"]), token=new_token())
    user.save()
    return Response({"token": user.token}, 201)


@router.post("/login/")
def login(request: Request):
    body = request.json()
    user = User.objects.filter(username=str(body.get("username", ""))).first()
    if user is None or not verify_password(str(body.get("password", "")), user.password_hash):
        raise HttpError(400, "Unable to log in with the provided credentials.")
    return {"token": user.token}


def int_param(request: Request, name: str, default: int, low: int, high: int) -> int:
    try:
        value = int(request.query.get(name, default))
    except ValueError:
        raise HttpError(400, {name: ["Must be an integer."]}) from None
    return min(max(value, low), high)


ORDERING = {"due_date", "priority", "created_at"}


@router.get("/tasks/")
@login_required
def list_tasks(request: Request):
    q = request.query
    tasks = Task.objects.filter(owner_id=request.user.id)
    if "status" in q:
        tasks = tasks.filter(status=q["status"])
    if "priority" in q:
        tasks = tasks.filter(priority=int_param(request, "priority", 2, 1, 3))
    if "priority_gte" in q:
        tasks = tasks.filter(priority__gte=int_param(request, "priority_gte", 1, 1, 3))
    if "search" in q:
        tasks = tasks.filter(title__contains=q["search"])
    if q.get("overdue") == "true":
        tasks = tasks.filter(due_date__lt=date.today().isoformat(), status__ne="done")
    ordering = q.get("ordering", "-id")
    if ordering.lstrip("-") not in ORDERING | {"id"}:
        raise HttpError(400, {"ordering": [f"Must be one of: {', '.join(sorted(ORDERING))} (prefix - to reverse)."]})
    page = int_param(request, "page", 1, 1, 10**6)
    size = int_param(request, "page_size", 20, 1, 100)
    tasks = tasks.order_by(ordering, "id")
    start = (page - 1) * size
    return {"count": tasks.count(), "page": page, "results": [_out(t) for t in tasks[start : start + size]]}


@router.post("/tasks/")
@login_required
def create_task(request: Request):
    data = TaskSchema.validate(request.json())
    return Response(_out(Task(owner_id=request.user.id, **data).save()), 201)


def own_task(request: Request) -> Task:
    try:  # Scoping the lookup to the owner is the authorization: other people's ids are simply 404.
        return Task.objects.get(id=request.params["id"], owner_id=request.user.id)
    except Task.DoesNotExist:
        raise HttpError(404, "Not found.") from None


@router.get("/tasks/<int:id>/")
@login_required
def get_task(request: Request):
    return _out(own_task(request))


@router.patch("/tasks/<int:id>/")
@login_required
def update_task(request: Request):
    task = own_task(request)
    for name, value in TaskSchema.validate(request.json(), partial=True).items():
        setattr(task, name, value)
    return _out(task.save())


@router.delete("/tasks/<int:id>/")
@login_required
def delete_task(request: Request):
    own_task(request).delete()
    return Response(None, 204)


@router.post("/tasks/<int:id>/complete/")
@login_required
def complete_task(request: Request):
    task = own_task(request)
    task.status = "done"
    return _out(task.save())


@router.get("/summary/")
@login_required
def summary(request: Request):
    return services.summarize(Task.objects.filter(owner_id=request.user.id))


def _out(task: Task) -> dict:
    return {**task.to_dict(), "is_overdue": task.is_overdue}
```

Things to notice:

- **Decorator order matters.** `@router.get(...)` is on the outside, so it registers the function already wrapped by `@login_required`. Swap them and the route would skip the login check.
- **Authorization is scoping the query.** `own_task` looks up by `id` *and* `owner_id`. Someone else's task id is a `404`, not a `403`, so you do not reveal that it exists.
- **Validate once, at the edge.** `TaskSchema.validate(...)` returns clean data. After that, handlers trust it.
- `partial=True` is how `PATCH` works: validate only the fields that were sent.
- `Task(owner_id=request.user.id, **data)` unpacks a dict into keyword arguments, like spreading an object into a function call.
- `overdue=false` is deliberately not supported. It needs `OR` and `NOT`, which our tiny `filter` cannot express. That is exactly why Django has `Q` objects.

### 8.5 Wire it together and run

Create `nanoweb/board/app.py`:

```py
from nano.app import App, request_log

from .api import router
from .auth import token_auth

app = App(router, middleware=[request_log, token_auth])
```

Create `nanoweb/board/run.py`:

```py
"""Run the dev server: uv run --env-file .env python -m board.run"""
import logging
import os
from wsgiref.simple_server import make_server

from nano.orm import db

from .app import app
from .models import Task, User


def main() -> None:
    logging.basicConfig(level=logging.INFO, format="%(message)s")
    db.connect(os.environ.get("DB_PATH", "board.sqlite3"))
    db.create_tables(User, Task)
    port = int(os.environ.get("PORT", "8800"))
    print(f"Serving on http://127.0.0.1:{port}")
    make_server("127.0.0.1", port, app).serve_forever()


if __name__ == "__main__":
    main()
```

The `request_log` middleware comes first in the list, so it is outermost and logs every response, including `401`s produced further in.

## Step 9: Run it and call it

Start the server:

```bash
uv run --env-file .env python -m board.run
```

Expected: `Serving on http://127.0.0.1:8800`. In a **second terminal** in the `nanoweb` folder, register and copy the token out of the response:

```bash
curl -s -X POST http://127.0.0.1:8800/register/ -d '{"username":"demo","password":"s3cret-pass-99"}'
```

Expected: `{"token": "<40 hex characters>"}`. Set up shell variables and make some requests:

```bash
TOKEN=<paste the token here>
B=http://127.0.0.1:8800
AUTH="Authorization: Token $TOKEN"
curl -s -X POST $B/tasks/ -H "$AUTH" -d '{"title":"Write docs","priority":3,"due_date":"2020-01-01"}'
curl -s "$B/tasks/?overdue=true" -H "$AUTH"
curl -s -X POST $B/tasks/1/complete/ -H "$AUTH"
curl -s $B/summary/ -H "$AUTH"
curl -s -o /dev/null -w "%{http_code}\n" $B/tasks/
curl -s -X POST $B/tasks/ -H "$AUTH" -d '{"title":"","priority":9}'
```

Expected, in order: the new task with `"is_overdue": true`; a list with `{"count": 1, "page": 1, "results": [...]}`; the same task with `"status": "done"` and a `completed_at`; `{"total": 1, "by_status": {"todo": 0, "doing": 0, "done": 1}, "overdue": 0, "all_done": true, "highest_open_priority": null}`; `401` (no token); and `{"title": ["This field may not be blank."], "priority": ["Must be at most 3."]}`. The server terminal prints one log line per request from the `request_log` middleware, such as `POST /tasks/ -> 201`. Ids and timestamps will differ. Stop the server with Ctrl+C and delete `board.sqlite3` to reset.

## Step 10: Tests that prove each claim

The test client calls the app in-process with a fresh in-memory database per test, so the suite takes a fraction of a second. Create `nanoweb/tests/conftest.py` (pytest loads this file automatically; it holds shared fixtures):

```py
import pytest

from board import auth
from board.app import app as board_app
from board.models import Task, User
from nano.orm import db
from nano.testing import Client


@pytest.fixture(autouse=True)
def fresh_db(monkeypatch):
    monkeypatch.setattr(auth, "ITERATIONS", 1_000)  # fast password hashing in tests
    db.connect(":memory:")
    db.create_tables(User, Task)


def signed_in(username="ada") -> Client:
    client = Client(board_app)
    res = client.post("/register/", {"username": username, "password": "s3cret-pass-99"})
    assert res.status == 201, res.data
    client.token = res.data["token"]
    return client


@pytest.fixture
def client():
    return signed_in()
```

Create `nanoweb/tests/test_framework.py`. These tests check the framework itself, including the claims we made about laziness, immutability, middleware order and SQL injection:

```py
import io
import json
from wsgiref.util import setup_testing_defaults

import pytest

from board.app import app as board_app
from board.models import Task, User
from nano.app import App
from nano.orm import Model, TextField, db
from nano.router import Router
from nano.schema import Choice, Int, IsoDate, Schema, Str
from nano.web import HttpError


def test_router_extracts_and_converts_params():
    router = Router()

    @router.get("/tasks/<int:id>/notes/<str:slug>/")
    def handler(request):
        pass

    found, params = router.resolve("GET", "/tasks/42/notes/hello/")
    assert found is handler and params == {"id": 42, "slug": "hello"}


def test_router_404_and_405():
    router = Router()
    router.add("GET", "/a/", lambda r: None)
    with pytest.raises(HttpError) as e404:
        router.resolve("GET", "/nope/")
    with pytest.raises(HttpError) as e405:
        router.resolve("POST", "/a/")
    with pytest.raises(HttpError):  # <int:id> will not match letters
        router.add("GET", "/t/<int:id>/", lambda r: None)
        router.resolve("GET", "/t/abc/")
    assert (e404.value.status, e405.value.status) == (404, 405)


def test_middleware_order_first_in_list_is_outermost():
    order = []

    def tag(name):
        def wrap(next_handler):
            def inner(request):
                order.append(f"{name} in")
                response = next_handler(request)
                order.append(f"{name} out")
                return response

            return inner

        return wrap

    router = Router()
    router.add("GET", "/", lambda r: order.append("handler") or {"ok": True})
    from nano.testing import Client

    assert Client(App(router, [tag("A"), tag("B")])).get("/").data == {"ok": True}
    assert order == ["A in", "B in", "handler", "B out", "A out"]


def test_unhandled_exception_becomes_500_not_a_crash():
    router = Router()
    router.add("GET", "/", lambda r: 1 / 0)
    from nano.testing import Client

    res = Client(App(router)).get("/")
    assert res.status == 500 and res.data == {"detail": "Server error."}


class DemoSchema(Schema):
    title = Str(max_length=5)
    priority = Int(min=1, max=3, required=False, default=2)
    status = Choice(["a", "b"], required=False, default="a")
    due = IsoDate(required=False, nullable=True)


def test_schema_cleans_defaults_and_reports_every_error():
    assert DemoSchema.validate({"title": "  hi  "}) == {"title": "hi", "priority": 2, "status": "a", "due": None}
    with pytest.raises(HttpError) as err:
        DemoSchema.validate({"title": "", "priority": True, "status": "z", "due": "2026-13-40"})
    assert set(err.value.detail) == {"title", "priority", "status", "due"}  # all at once, not one by one
    with pytest.raises(HttpError) as missing:
        DemoSchema.validate({})
    assert missing.value.detail == {"title": ["This field is required."]}


def test_schema_partial_updates_only_validate_what_is_sent():
    assert DemoSchema.validate({"priority": 3}, partial=True) == {"priority": 3}


def test_orm_crud_lookups_and_ordering():
    for i, title in enumerate(["b task", "a task", "c thing"]):
        Task(owner_id=1, title=title, priority=i + 1).save()
    assert Task.objects.filter(title__contains="task").count() == 2
    assert [t.title for t in Task.objects.filter(priority__gte=2).order_by("-priority")] == ["c thing", "a task"]
    assert [t.title for t in Task.objects.order_by("title")[1:3]] == ["b task", "c thing"]
    task = Task.objects.get(title="a task")
    task.title = "renamed"
    task.save()
    assert Task.objects.get(id=task.id).title == "renamed"
    task.delete()
    assert Task.objects.count() == 2
    with pytest.raises(Task.DoesNotExist):
        Task.objects.get(id=task.id)
    assert Task.DoesNotExist is not User.DoesNotExist  # one exception class per model


def test_querysets_are_lazy_and_immutable():
    statements = []
    db.conn.set_trace_callback(statements.append)
    base = Task.objects.filter(owner_id=1)
    narrowed = base.filter(status="done").order_by("-id")
    assert statements == []  # nothing has hit the database yet
    assert base.where != narrowed.where  # filter() returned a NEW queryset
    list(narrowed)
    assert len(statements) == 1


def test_orm_is_safe_against_sql_injection():
    Task(owner_id=1, title="x'); DROP TABLE task; --").save()
    assert Task.objects.filter(title="x'); DROP TABLE task; --").count() == 1  # values are parameters
    with pytest.raises(ValueError):
        Task.objects.filter(**{"title = 1 OR 1": "x"})  # field names are checked against the model
    with pytest.raises(ValueError):
        Task.objects.order_by("title; DROP TABLE task")
    with pytest.raises(ValueError):
        Task.objects.filter(title__regex="x")
    assert Task.objects.count() == 1


def test_manager_is_a_descriptor_and_model_rejects_unknown_fields():
    with pytest.raises(AttributeError):
        _ = Task(owner_id=1, title="t").objects
    with pytest.raises(TypeError):
        Task(owner_id=1, title="t", nope=1)


def test_a_brand_new_model_works_with_no_extra_code():
    class Note(Model):
        body = TextField()

    db.create_tables(Note)
    Note(body="hi").save()
    assert Note.objects.get(body="hi").id == 1


def call_wsgi(app, method, path, body=None, token=None):
    environ = {"REQUEST_METHOD": method, "PATH_INFO": path, "QUERY_STRING": ""}
    setup_testing_defaults(environ)
    raw = json.dumps(body).encode() if body is not None else b""
    environ.update({"CONTENT_LENGTH": str(len(raw)), "wsgi.input": io.BytesIO(raw)})
    if token:
        environ["HTTP_AUTHORIZATION"] = f"Token {token}"
    seen = {}
    chunks = app(environ, lambda status, headers: seen.update(status=status, headers=dict(headers)))
    return seen["status"], seen["headers"], b"".join(chunks)


def test_app_speaks_real_wsgi():
    status, headers, body = call_wsgi(board_app, "POST", "/register/", {"username": "wsgi", "password": "s3cret-pass-99"})
    assert status == "201 Created" and headers["Content-Type"] == "application/json"
    token = json.loads(body)["token"]
    assert int(headers["Content-Length"]) == len(body)
    status, _, body = call_wsgi(board_app, "GET", "/summary/", token=token)
    assert status == "200 OK" and json.loads(body)["total"] == 0
    assert call_wsgi(board_app, "GET", "/summary/")[0] == "401 Unauthorized"
```

Create `nanoweb/tests/test_api.py`. These test the application end to end, including that users cannot touch each other's data:

```py
from datetime import date, timedelta

from board import auth
from board.models import User
from nano.testing import Client
from nano.web import Request
from tests.conftest import signed_in


def new_task(client, **fields):
    res = client.post("/tasks/", {"title": "T", **fields})
    assert res.status == 201, res.data
    return res.data


def test_passwords_are_hashed_and_verified():
    stored = auth.hash_password("hunter2hunter2")
    assert "hunter2" not in stored and stored.startswith("pbkdf2_sha256$")
    assert auth.verify_password("hunter2hunter2", stored) and not auth.verify_password("wrong", stored)
    assert auth.hash_password("same") != auth.hash_password("same")  # random salt


def test_register_login_and_validation(client):
    anon = Client(client.app)
    assert anon.post("/register/", {"username": "ada", "password": "s3cret-pass-99"}).status == 400  # taken
    assert anon.post("/register/", {"username": "bob", "password": "short"}).status == 400
    assert anon.post("/register/", {}).data == {
        "username": ["This field is required."], "password": ["This field is required."]}
    assert User.objects.get(username="ada").password_hash.startswith("pbkdf2_sha256$")
    login = anon.post("/login/", {"username": "ada", "password": "s3cret-pass-99"})
    assert login.status == 200 and login.data["token"] == client.token
    assert anon.post("/login/", {"username": "ada", "password": "nope"}).status == 400


def test_auth_required_and_bad_token():
    anon = Client(signed_in().app)
    assert anon.get("/tasks/").status == 401
    anon.token = "not-a-real-token"
    assert anon.get("/tasks/").status == 401


def test_task_crud(client):
    task = new_task(client, title="  Write docs  ", priority=3)
    assert task["title"] == "Write docs" and task["status"] == "todo" and task["completed_at"] is None
    assert client.get(f"/tasks/{task['id']}/").data["priority"] == 3
    assert client.patch(f"/tasks/{task['id']}/", {"priority": 1}).data["priority"] == 1
    assert client.patch(f"/tasks/{task['id']}/", {"status": "nope"}).status == 400
    assert client.delete(f"/tasks/{task['id']}/").status == 204
    assert client.get(f"/tasks/{task['id']}/").status == 404


def test_validation_and_bad_input(client):
    res = client.post("/tasks/", {"title": "", "status": "nope", "priority": 9})
    assert res.status == 400 and set(res.data) == {"title", "status", "priority"}
    raw = client.app.handle(
        Request("POST", "/tasks/", headers={"authorization": f"Token {client.token}"}, body=b"{not json")
    )
    assert raw.status == 400 and raw.data == {"detail": "Body is not valid JSON."}
    assert client.post("/nope/").status == 404 and client.delete("/tasks/").status == 405


def test_users_cannot_touch_each_others_tasks(client):
    task = new_task(client)
    eve = signed_in("eve")
    assert eve.get("/tasks/").data["count"] == 0
    assert eve.get(f"/tasks/{task['id']}/").status == 404
    assert eve.patch(f"/tasks/{task['id']}/", {"title": "hacked"}).status == 404
    assert eve.delete(f"/tasks/{task['id']}/").status == 404
    assert eve.post(f"/tasks/{task['id']}/complete/").status == 404
    assert client.get(f"/tasks/{task['id']}/").data["title"] == "T"


def test_complete_sets_completed_at_and_reopen_clears_it(client):
    task = new_task(client)
    done = client.post(f"/tasks/{task['id']}/complete/").data
    assert done["status"] == "done" and done["completed_at"]
    assert client.patch(f"/tasks/{task['id']}/", {"status": "doing"}).data["completed_at"] is None


def test_filters_search_ordering(client):
    yesterday = (date.today() - timedelta(days=1)).isoformat()
    new_task(client, title="Write docs", priority=1)
    new_task(client, title="Fix bug", priority=3, due_date=yesterday)
    finished = new_task(client, title="Old thing", priority=2, due_date=yesterday)
    client.post(f"/tasks/{finished['id']}/complete/")

    def titles(query):
        return [t["title"] for t in client.get(f"/tasks/?{query}").data["results"]]

    assert titles("overdue=true") == ["Fix bug"]  # done tasks are never overdue
    assert titles("status=done") == ["Old thing"]
    assert sorted(titles("priority_gte=2")) == ["Fix bug", "Old thing"]
    assert titles("search=docs") == ["Write docs"]
    assert titles("ordering=-priority") == ["Fix bug", "Old thing", "Write docs"]
    assert titles("status=todo&ordering=priority") == ["Write docs", "Fix bug"]
    assert client.get("/tasks/?ordering=title;DROP").status == 400
    assert client.get("/tasks/?page=abc").status == 400


def test_pagination(client):
    for i in range(25):
        new_task(client, title=f"t{i}")
    page1 = client.get("/tasks/").data
    assert page1["count"] == 25 and len(page1["results"]) == 20 and page1["page"] == 1
    assert len(client.get("/tasks/?page=2").data["results"]) == 5
    assert len(client.get("/tasks/?page_size=500").data["results"]) == 25  # clamped to 100, only 25 exist


def test_summary(client):
    yesterday = (date.today() - timedelta(days=1)).isoformat()
    new_task(client, priority=3, due_date=yesterday)
    done = new_task(client, priority=1)
    client.post(f"/tasks/{done['id']}/complete/")
    assert client.get("/summary/").data == {
        "total": 2, "by_status": {"todo": 1, "doing": 0, "done": 1},
        "overdue": 1, "all_done": False, "highest_open_priority": 3}
```

Run everything:

```bash
uv run pytest -q
uv run ruff check .
```

Expected: `22 passed` and `All checks passed!`. Useful flags: `uv run pytest -k orm` runs matching tests, `uv run pytest -x` stops at the first failure, `uv run pytest -vv` shows full diffs.

## Step 11: The reveal: your pieces versus Django's

You now know what every layer does. Here is the map to the real thing:

| You built | Django / DRF equivalent | What the real one adds |
| --- | --- | --- |
| `Request`, `Response` (`nano/web.py`) | `HttpRequest`, `HttpResponse`, `JsonResponse` | Cookies, file uploads, streaming, content negotiation, character encodings |
| `Request.from_environ`, `App.__call__` | `WSGIHandler` (and an ASGI twin for async) | Signals, DB connection management, request-finished cleanup |
| `Router` | `urls.py`, `path()`, `include()`, `reverse()` | Nested includes, reverse URL lookup, more converters, custom converters |
| `HttpError` | `Http404`, `PermissionDenied`, DRF exceptions | Pluggable error pages and exception handlers |
| middleware list in `App` | `MIDDLEWARE` in settings | Security headers, sessions, CSRF protection, per-view and async hooks |
| `Schema` and fields | DRF serializers, Django forms | Nested objects, relations, model-backed fields, translated messages |
| `Model`, `Field` | `django.db.models.Model` and fields | Relations (joins), migrations, transactions, many database backends, signals |
| `QuerySet` | `django.db.models.QuerySet` | `Q` objects for OR/NOT, `F` expressions, aggregation, `select_related`, caching |
| `Manager` descriptor | `models.Manager` | Custom managers and querysets |
| `login_required`, `token_auth` | `permission_classes`, authentication classes | Sessions, OAuth integrations, permissions, groups |
| `Client` | `django.test.Client`, DRF `APIClient` | Transaction-per-test isolation, test databases |
| `board/api.py` | DRF `ViewSet`, `Router`, filters, pagination | Browsable API, schema generation, throttling |

**Go read the real source.** You will find it far less intimidating now. These files exist in the Django package (find them with `uv run python -c "import django; print(django.__file__)"` in the Django project from the companion tutorial, then open the folder):

- `django/core/handlers/wsgi.py` and `django/core/handlers/base.py`: the WSGI entry point and the request-to-response loop (your `App`).
- `django/urls/resolvers.py`: URL matching (your `Router`).
- `django/http/request.py` and `django/http/response.py`: your `Request` and `Response`.
- `django/db/models/base.py`: the `Model` class and its metaclass machinery (your `Model` and `__init_subclass__`).
- `django/db/models/manager.py` and `django/db/models/query.py`: `Manager` and `QuerySet` (your lazy, chainable, immutable queryset).
- In the REST framework package: `rest_framework/serializers.py` and `rest_framework/fields.py` (your `Schema` and `Field`).

A good exercise: open `query.py`, find `_chain` and `filter`, and see how a real queryset clones itself before applying a filter. It is your `_clone` at scale.

## Step 12: Exercises

Rank-ordered from easy to hard. Do at least one from each group.

**Break it (see why each design choice exists)**

1. In `nano/orm.py`, change `QuerySet.filter` to paste values into the SQL string with an f-string instead of using `?`. Run the injection test and watch it fail.
2. Swap the order of `@router.get("/tasks/")` and `@login_required` on `list_tasks`. Which test catches it? (Think about why the answer is "none of them", then write the test that would.)
3. Remove `.strip()` handling and `bool` rejection from the schema fields and see which tests fail.
4. Make `_clone` mutate `self` instead of returning a new queryset. Which tests fail, and what real bug would this cause?

**Extend it**

5. Add a `Project` model and a `/projects/` endpoint, with `project_id` on `Task`. Add a join to the ORM or fetch related rows with a second query, then notice the N+1 problem.
6. Add `exclude(**lookups)` and then `overdue=false`. Why does it need `OR`? Sketch a tiny `Q` class.
7. Add `Content-Type: application/json` validation to `Request.json()` and return `415` otherwise.
8. Add a rate-limit middleware (N requests per token per minute) using a dict and `time.monotonic()`.
9. Replace `Database.execute`'s commit-per-call with a `with db.transaction():` context manager built on `contextlib.contextmanager`.

**Port it**

10. Re-implement the task API with real Django and DRF (the companion tutorial has the finished version). Run *the same scenarios* from `tests/test_api.py` against it and note every difference in behavior. Each difference is something the real framework decided for you.
11. Serve `board.app.app` with gunicorn instead of wsgiref to see that a WSGI app is portable (not tested here).

## Step 13: Teaching kit

The fastest way to find out whether you understand something is to explain it from a blank screen. Use this kit to turn what you just built into a video and a post.

### The four-week plan

- **Week 1: choose the concept.** Pick one thing you use but cannot explain. Candidates from this tutorial: "how a Django QuerySet stays lazy", "what middleware really is", "what `__init_subclass__` does for models". Write a one-sentence promise: "After this you can explain X without notes."
- **Week 2: code the minimal version.** Redo that one piece from scratch without looking, in under 100 lines, with a test. When you get stuck, that is the exact part you do not understand yet. Go back to the Django source.
- **Week 3: record from a blank screen.** Use the script below. Do not read the finished code; type it, and let mistakes stay in if you can recover from them out loud.
- **Week 4: publish.** Repo with the code and tests, the video, and a short written reflection.

### A 15-minute blank-screen script (QuerySet version)

| Time | What you do and say |
| --- | --- |
| 0:00 | State the question: "Why does `Task.objects.filter(...).filter(...)` not run any SQL?" Show the finished output you are aiming for. |
| 1:00 | Open an empty `orm.py`. Write the `Database` class with one `execute` method. Explain parameters (`?`) versus string pasting in one sentence. |
| 3:00 | Write `Model` with `__init_subclass__` and `Field`. Show that declaring `title = TextField()` is discovered automatically. Run it. |
| 6:00 | Write a naive `filter` that runs SQL immediately. Show that it cannot be chained. This is the problem. |
| 8:00 | Rewrite `filter` to return a new `QuerySet` that only stores `where` and `params`. Add `__iter__`. Run it, and show the SQL logged only on iteration. |
| 11:00 | Add the injection test, run it green, then break it live by pasting values into the SQL and watch it go red. |
| 13:00 | Open Django's `django/db/models/query.py`. Find `_chain` and `filter`. Say what is the same and what is bigger. |
| 14:30 | Recap in three sentences. Point to the repo. |

### Publish checklist

- [ ] The repo runs from a clean clone with the commands in Step 2 and `uv run pytest -q` passes
- [ ] The README states the one-sentence promise and links the video
- [ ] Every claim in the video has a test or a printed output
- [ ] You show at least one mistake and how you debugged it
- [ ] A "what this leaves out" section is honest about what Django does that your version does not
- [ ] The video starts with the finished result, not with setup

### Reflection post template

```txt
Title: I rebuilt <concept> in <N> lines. Here is what I got wrong.

1. The question I could not answer: <one sentence>
2. The smallest version that worked: <link to repo, N lines>
3. The thing that surprised me: <one paragraph>
4. The bug I introduced and how I found it: <short story>
5. What the real implementation does that mine does not: <3 bullets>
6. Try it: <exercise you want readers to attempt>
```

## Checklist

- [ ] Python and uv installed, `uv run pytest -q` shows `22 passed`
- [ ] You can explain WSGI in one sentence
- [ ] You can explain why `QuerySet.filter` returns a new object
- [ ] You can explain why values are `?` parameters but field names are checked
- [ ] You opened at least one real Django source file from Step 11
- [ ] You did at least one exercise from each group in Step 12

> [!TIP]
> **Verified:** every file above was built and run on Python 3.13 with pytest 9.2.0 and ruff 0.17.4: lint clean, 22 tests passing (router, schema, ORM laziness and SQL safety, middleware order, a real WSGI call, and the full API including cross-user isolation). The six-line `hello_wsgi.py` and the full `board` server were run with wsgiref and called with curl. The Django and REST framework source files listed in Step 11 were checked to exist in the installed packages (Django 6.1.3, djangorestframework 3.18.3).
> **Not verified:** the descriptions of what real Django adds in the Step 11 table (they come from general Django knowledge, not from reading each source file for this page), gunicorn serving the app (exercise 11), and the exercises themselves. The nanoweb server is a teaching tool: single-threaded, no migrations, no CSRF, no real transactions. Do not deploy it.
