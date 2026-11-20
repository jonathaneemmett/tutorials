# Python AI Native: Build the API First with Django (Step by Step)

Every AI feature that touches real data sits on top of an API. The model does not read your database. It calls a tool, the tool calls your API, and your API decides what that caller may see. If you build the model-facing layer first, you end up guessing at the layer underneath it. So this series starts at the bottom: a real HTTP API, built properly, that the next chapters put an AI on top of.

In this chapter you build a **provider directory API** with Django 6.1 and Django Ninja: a database model, token authentication, search with pagination, staff-only writes, rate limits, generated OpenAPI docs and 30 tests that need no network. In chapter 2 an MCP server calls it. In chapter 3 and after, agents use that MCP server.

> [!NOTE]
> Chapter 1 of [Python AI Native: Build Production AI](../README.md). Next: [Python AI Native: Build an MCP Server, Agent and Webhook (Step by Step)](02-mcp-server-agent-webhook.md), which calls the API you build here. You need no API key for this chapter.

---

## Why the API comes first

| Layer | Job | Built in |
| --- | --- | --- |
| API (this chapter) | Owns the data and the rules: who may see what, what is valid, how fast callers may go | Chapter 1 |
| MCP server | Describes the API to a model as tools, and turns its answers into text the model can use | Chapter 2 |
| Agent | Decides which tool to call, in a loop | Chapters 2 to 4 |

Three decisions in this chapter exist because an AI will be the caller:

1. **Authorization lives in the API, not in the prompt.** A prompt that says "do not show internal providers" is a request. A query that never returns them is a rule. Every token belongs to a user, and every query is filtered for that user.
2. **Validation lives in the API.** A model will send an empty name, a 500-row page size or a state called "California". The API refuses each with a clear error the tool can pass back to the model.
3. **Errors are plain and specific.** The model reads them and tries again. A vague 500 teaches it nothing.

**Why Django and Django Ninja?** Django gives you the ORM, migrations, users and a free admin site. Django Ninja adds typed endpoints on top: you declare the input and output shapes with Python type hints, and you get validation and an OpenAPI schema for free. Django REST Framework (current release 3.18) is the older, larger alternative and is a fine choice if your team already uses it. Everything below that is about Django itself, such as models, tokens and visibility rules, carries over unchanged.

> [!WARNING]
> This tutorial targets Django 6.1 (current release 6.1.3, which requires Python 3.12 or newer), django-ninja 1.7 and pytest-django 4.14. Tested with Python 3.13. Two things changed recently and older tutorials get them wrong: Django 6.1 validates a new `MAILERS` setting (the old `EMAIL_*` settings are deprecated), and django-ninja now wants `Status(201, obj)` instead of returning a `(201, obj)` tuple.

## What you will build

| Step | You build |
| --- | --- |
| 0 | Install the tools |
| 1 | Project setup and settings |
| 2 | The data model: providers and API tokens |
| 3 | The API: auth, schemas and endpoints |
| 4 | Seed data, tokens, and a tour with curl |
| 5 | Tests (30, no network) |
| 6 | Make it real: production settings and serving |

---

## Step 0: Install everything

You need two things: **uv** (installs Python and dependencies) and **Python 3.12 or newer**. Nothing else. There is no key and no account in this chapter.

**0.1 Open a terminal.** macOS: Terminal. Windows: PowerShell. Linux: any shell.

**0.2 Install uv.** macOS or Linux (pick one):

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

Expected: a version number such as `uv 0.13.7`. If you see `command not found`, the install directory is not on your `PATH`. Reopen the terminal, or add `~/.local/bin` to your `PATH`.

**0.3 Install Python.** Let uv do it:

```bash
uv python install 3.13
uv python list --only-installed
```

Expected: a `cpython-3.13.x` line. Any version from 3.12 up works, because Django 6.1 needs 3.12 or newer.

**0.4 Check curl.** The tour in Step 4 uses `curl`, which ships with macOS and most Linux systems. On Windows use Git Bash or WSL for those commands.

```bash
curl --version
```

**0.5 Checklist**

- [ ] `uv --version` prints a version
- [ ] `uv python list --only-installed` shows 3.12 or newer
- [ ] `curl --version` prints a version

---

## Step 1: Create the project

Every path in this tutorial is relative to the `directory/` folder (the project root), and every command runs from there unless it says otherwise.

```bash
mkdir directory && cd directory
```

Create `directory/pyproject.toml`:

```toml
[project]
name = "directory"
version = "0.1.0"
description = "Provider directory API (Django + Django Ninja)"
requires-python = ">=3.12"
dependencies = [
    "django>=6.1,<6.2",
    "django-ninja>=1.7,<2",
]

[dependency-groups]
dev = ["pytest>=9.0", "pytest-django>=4.14", "ruff>=0.5.6"]

[tool.pytest.ini_options]
DJANGO_SETTINGS_MODULE = "config.settings"
python_files = ["test_*.py"]
testpaths = ["tests"]

[tool.ruff]
extend-exclude = ["*/migrations/*"]

[tool.ruff.lint]
select = ["E", "F", "I", "UP"]
ignore = ["E501"]
```

Install the dependencies. This creates a `.venv` and a `uv.lock`:

```bash
uv sync
```

Check:

```bash
uv run python -m django --version
uv pip show django-ninja | head -2
```

Expected: `6.1.3` (or a newer 6.1.x), then a `Name: django-ninja` line with version `1.7.x`.

Now let Django create the skeleton, then remove two files you will not use and create the folders the later steps need:

```bash
uv run django-admin startproject config .
uv run python manage.py startapp providers
rm providers/tests.py providers/views.py
mkdir -p tests providers/management/commands
touch tests/__init__.py providers/management/__init__.py providers/management/commands/__init__.py
```

Create `directory/.env` and `directory/.gitignore`. Settings come from the environment, and `uv run --env-file .env` loads this file, so there is nothing to export and nothing to set again in a new terminal:

```bash
cat > .env <<'EOF'
DJANGO_DEBUG=1
DJANGO_SECRET_KEY=dev-only-change-me
DJANGO_ALLOWED_HOSTS=localhost,127.0.0.1
EOF
printf '.env\n.venv/\n__pycache__/\ndb.sqlite3\nstaticfiles/\n' > .gitignore
```

On Windows, create both files in your editor, saved in the `directory` folder, with the same contents.

By the end, the project looks like this:

```txt
directory/
├── .env
├── .gitignore
├── manage.py                        (generated)
├── pyproject.toml
├── config/
│   ├── settings.py                  Step 1 (replaced)
│   ├── urls.py                      Step 3 (replaced)
│   ├── asgi.py, wsgi.py             (generated)
├── providers/
│   ├── models.py                    Step 2
│   ├── admin.py                     Step 2
│   ├── schemas.py                   Step 3
│   ├── auth.py                      Step 3
│   ├── api.py                       Step 3
│   └── management/commands/
│       ├── seed_providers.py        Step 4
│       └── issue_token.py           Step 4
└── tests/
    ├── conftest.py                  Step 5
    ├── test_models.py               Step 5
    ├── test_api.py                  Step 5
    ├── test_commands.py             Step 5
    └── test_throttle.py             Step 5
```

Replace `directory/config/settings.py` with this smaller version. It keeps what the API needs, reads secrets from the environment, and refuses to start in production without a real secret key:

```py
"""Settings for the provider directory API. Development defaults, production by environment."""
import os
from pathlib import Path

from django.core.exceptions import ImproperlyConfigured

BASE_DIR = Path(__file__).resolve().parent.parent

DEBUG = os.environ.get("DJANGO_DEBUG", "1") == "1"

SECRET_KEY = os.environ.get("DJANGO_SECRET_KEY", "")
if not SECRET_KEY:
    if not DEBUG:
        raise ImproperlyConfigured("Set DJANGO_SECRET_KEY when DJANGO_DEBUG=0")
    SECRET_KEY = "dev-only-insecure-key"

ALLOWED_HOSTS = os.environ.get("DJANGO_ALLOWED_HOSTS", "localhost,127.0.0.1").split(",")

INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "providers",
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
STATIC_ROOT = BASE_DIR / "staticfiles"

DEFAULT_AUTO_FIELD = "django.db.models.BigAutoField"

# This API sends no email, but 6.1 validates the MAILERS setting in `check --deploy`.
MAILERS = {"default": {"BACKEND": "django.core.mail.backends.console.EmailBackend"}}

# Production: everything below switches on when DJANGO_DEBUG=0.
if not DEBUG:
    MAILERS = {
        "default": {
            "BACKEND": "django.core.mail.backends.smtp.EmailBackend",
            "OPTIONS": {"host": os.environ.get("SMTP_HOST", "localhost")},
        },
    }
    SECURE_SSL_REDIRECT = True
    SECURE_HSTS_SECONDS = 3600  # raise to a year once you are sure HTTPS works everywhere
    SESSION_COOKIE_SECURE = True
    CSRF_COOKIE_SECURE = True
    # Only correct behind a proxy that always sets this header (most hosts do).
    SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")
```

Check:

```bash
uv run --env-file .env python manage.py check
```

Expected: `System check identified no issues (0 silenced).`

---

## Step 2: The data model

Two tables. A **Provider** is the thing you search. An **ApiToken** is how a caller proves who they are. Create `directory/providers/models.py`:

```py
import hashlib
import secrets

from django.conf import settings
from django.db import models
from django.utils import timezone


class ProviderQuerySet(models.QuerySet):
    def visible_to(self, user):
        """Authorization lives here: staff see everything, everyone else sees public rows."""
        if user.is_staff:
            return self
        return self.filter(visibility=Provider.Visibility.PUBLIC)

    def search(self, text: str):
        """Every word must appear in the name, specialty, city or state."""
        qs = self
        for term in text.split():
            qs = qs.filter(
                models.Q(name__icontains=term)
                | models.Q(specialty__icontains=term)
                | models.Q(city__icontains=term)
                | models.Q(state__icontains=term)
            )
        return qs


class Provider(models.Model):
    class Visibility(models.TextChoices):
        PUBLIC = "public"
        INTERNAL = "internal"

    name = models.CharField(max_length=200)
    specialty = models.CharField(max_length=100)
    city = models.CharField(max_length=100)
    state = models.CharField(max_length=2)
    visibility = models.CharField(
        max_length=10, choices=Visibility, default=Visibility.PUBLIC
    )
    created_at = models.DateTimeField(auto_now_add=True)

    objects = ProviderQuerySet.as_manager()

    class Meta:
        ordering = ["id"]

    def __str__(self) -> str:
        return f"{self.name} ({self.specialty}, {self.city}, {self.state})"


def hash_key(raw: str) -> str:
    return hashlib.sha256(raw.encode()).hexdigest()


class ApiToken(models.Model):
    """A bearer token for one user. Only the hash is stored, so a database leak is not a key leak."""

    user = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="api_tokens"
    )
    label = models.CharField(max_length=100, blank=True)
    key_hash = models.CharField(max_length=64, unique=True)
    created_at = models.DateTimeField(auto_now_add=True)
    revoked_at = models.DateTimeField(null=True, blank=True)

    @classmethod
    def issue(cls, user, label: str = "") -> str:
        """Create a token and return the raw key. This is the only time it exists in plain text."""
        raw = "dir_" + secrets.token_urlsafe(32)
        cls.objects.create(user=user, label=label, key_hash=hash_key(raw))
        return raw

    def revoke(self) -> None:
        self.revoked_at = timezone.now()
        self.save(update_fields=["revoked_at"])

    def __str__(self) -> str:
        return f"token for {self.user} ({self.label or 'unlabelled'})"
```

Why it is written this way:

- **Authorization is a queryset method.** `visible_to(user)` is the only place that knows staff see internal rows and everyone else does not. Every endpoint that reads providers goes through it, so a new endpoint cannot forget the rule.
- **Search is a queryset method too**, so the tests can exercise it without HTTP. Every word must match somewhere, which is how the model's queries ("orthopedic surgery CA") naturally behave.
- **Tokens are stored hashed.** The raw key exists once, at creation. If the database leaks, the keys do not. A SHA-256 hash is enough for a long random key. You would use a slow hash like the ones Django uses for passwords only for secrets a human chose.
- **Tokens can be revoked** without deleting history, and a token for an inactive user stops working.

Register both in the admin so you can see and manage them. Create `directory/providers/admin.py`:

```py
from django.contrib import admin

from providers.models import ApiToken, Provider


@admin.register(Provider)
class ProviderAdmin(admin.ModelAdmin):
    list_display = ("name", "specialty", "city", "state", "visibility")
    list_filter = ("visibility", "state")
    search_fields = ("name", "specialty", "city")


@admin.register(ApiToken)
class ApiTokenAdmin(admin.ModelAdmin):
    list_display = ("user", "label", "created_at", "revoked_at")
    readonly_fields = ("key_hash",)
```

Create the tables:

```bash
uv run --env-file .env python manage.py makemigrations providers
uv run --env-file .env python manage.py migrate
```

Expected: `Create model Provider` and `Create model ApiToken`, then a list of `Applying ... OK` lines ending with `Applying providers.0001_initial... OK`. Check:

```bash
uv run --env-file .env python manage.py shell -c "from providers.models import Provider; print(Provider.objects.count())"
```

Expected: `0`. Django 6 also prints a line such as `14 objects imported automatically` first. That is the new shell auto-import and is harmless.

---

## Step 3: The API

Four small files. First, the shapes. Create `directory/providers/schemas.py`:

```py
from ninja import Field, Schema

from providers.models import Provider


class ProviderIn(Schema):
    name: str = Field(min_length=1, max_length=200)
    specialty: str = Field(min_length=1, max_length=100)
    city: str = Field(min_length=1, max_length=100)
    state: str = Field(pattern=r"^[A-Za-z]{2}$")
    visibility: Provider.Visibility = Provider.Visibility.PUBLIC


class ProviderPatch(Schema):
    name: str | None = Field(default=None, min_length=1, max_length=200)
    specialty: str | None = Field(default=None, min_length=1, max_length=100)
    city: str | None = Field(default=None, min_length=1, max_length=100)
    state: str | None = Field(default=None, pattern=r"^[A-Za-z]{2}$")
    visibility: Provider.Visibility | None = None


class ProviderOut(Schema):
    id: int
    name: str
    specialty: str
    city: str
    state: str
    visibility: str


class ProviderPage(Schema):
    items: list[ProviderOut]
    count: int
    page: int
    page_size: int
```

Each class is a contract. `ProviderIn` is what a caller may send: names must be 1 to 200 characters, and `state` must be exactly two letters. `ProviderPatch` makes every field optional so a caller can change one thing. `ProviderOut` is what comes back, and `ProviderPage` wraps a list with the total count so a caller knows whether there is more.

Next, authentication. Create `directory/providers/auth.py`:

```py
from ninja.security import HttpBearer

from providers.models import ApiToken, hash_key


class TokenAuth(HttpBearer):
    """Authorization: Bearer dir_... -> the User who owns the token. request.auth is that user."""

    def authenticate(self, request, token):
        row = (
            ApiToken.objects.select_related("user")
            .filter(
                key_hash=hash_key(token),
                revoked_at__isnull=True,
                user__is_active=True,
            )
            .first()
        )
        return row.user if row else None
```

`HttpBearer` reads the `Authorization: Bearer ...` header and calls `authenticate`. Whatever you return becomes `request.auth`. Return `None` and the caller gets a 401. Here it returns the **user** who owns the token, which is exactly what `visible_to` needs.

Now the endpoints. Create `directory/providers/api.py`:

```py
from django.shortcuts import get_object_or_404
from ninja import NinjaAPI, Query, Status
from ninja.errors import HttpError
from ninja.throttling import AnonRateThrottle, AuthRateThrottle

from providers.auth import TokenAuth
from providers.models import Provider
from providers.schemas import ProviderIn, ProviderOut, ProviderPage, ProviderPatch

api = NinjaAPI(
    title="Provider Directory API",
    version="1.0.0",
    auth=TokenAuth(),  # every operation needs a token unless it says auth=None
    throttle=[
        AnonRateThrottle("60/min"),  # open endpoints such as /health, counted per IP
        AuthRateThrottle("600/min"),  # callers with a token, counted per token owner
    ],
)


def require_staff(request) -> None:
    if not request.auth.is_staff:
        raise HttpError(403, "Staff only")


@api.get("/health", auth=None, tags=["meta"])
def health(request):
    return {"status": "ok"}


@api.get("/providers", response=ProviderPage, tags=["providers"])
def list_providers(
    request,
    q: str = "",
    state: str = "",
    page: int = Query(1, ge=1),
    page_size: int = Query(20, ge=1, le=100),
):
    """Search providers. Every word in `q` must match name, specialty, city or state."""
    qs = Provider.objects.visible_to(request.auth).search(q)
    if state:
        qs = qs.filter(state__iexact=state)
    start = (page - 1) * page_size
    return {
        "items": list(qs[start : start + page_size]),
        "count": qs.count(),
        "page": page,
        "page_size": page_size,
    }


@api.get("/providers/{provider_id}", response=ProviderOut, tags=["providers"])
def get_provider(request, provider_id: int):
    # Filtering by visibility first means a hidden row looks exactly like a missing one.
    return get_object_or_404(Provider.objects.visible_to(request.auth), id=provider_id)


@api.post("/providers", response={201: ProviderOut}, tags=["providers"])
def create_provider(request, payload: ProviderIn):
    require_staff(request)
    data = payload.model_dump()
    data["state"] = data["state"].upper()
    return Status(201, Provider.objects.create(**data))


@api.patch("/providers/{provider_id}", response=ProviderOut, tags=["providers"])
def update_provider(request, provider_id: int, payload: ProviderPatch):
    require_staff(request)
    provider = get_object_or_404(Provider, id=provider_id)
    changes = payload.model_dump(exclude_unset=True)
    if "state" in changes:
        changes["state"] = changes["state"].upper()
    for field, value in changes.items():
        setattr(provider, field, value)
    provider.save()
    return provider


@api.delete("/providers/{provider_id}", response={204: None}, tags=["providers"])
def delete_provider(request, provider_id: int):
    require_staff(request)
    get_object_or_404(Provider, id=provider_id).delete()
    return Status(204, None)
```

Read it once, slowly. These are the choices that matter:

- **`auth=TokenAuth()` on the whole API.** Every endpoint needs a token unless it opts out with `auth=None`, and only `/health` does. A new endpoint is private by default.
- **A hidden row looks missing.** `get_provider` filters by visibility before it looks up the id, so a reader asking for an internal provider gets the same 404 as for an id that does not exist. A 403 would confirm the row is there.
- **Limits are in the type.** `Query(20, ge=1, le=100)` means a page size of 500 is a 422 with a precise message, before your code runs.
- **Writes are staff only**, checked at the top of each write endpoint.
- **`Status(201, ...)`** sets the response code. Returning a tuple still works in 1.7 but warns that it is deprecated.
- **Throttles are declared once** on the `NinjaAPI`. One of them does less than its name suggests. See Step 5.

Finally, mount it. Replace `directory/config/urls.py`:

```py
from django.contrib import admin
from django.urls import path

from providers.api import api

urlpatterns = [
    path("admin/", admin.site.urls),
    path("api/", api.urls),
]
```

Run the server. Port 8001 matters: the MCP server in chapter 2 uses 8000.

```bash
uv run --env-file .env python manage.py runserver 8001
```

In a second terminal, check the open and closed doors:

```bash
curl -s http://127.0.0.1:8001/api/health
curl -s -w ' [%{http_code}]\n' http://127.0.0.1:8001/api/providers
```

Expected:

```txt
{"status": "ok"}
{"detail": "Unauthorized"} [401]
```

Open `http://127.0.0.1:8001/api/docs` in a browser. Django Ninja built interactive documentation from your type hints, with an **Authorize** button for entering a token. That page, and the JSON behind it at `/api/openapi.json`, always match the code, because they are generated from it.

---

## Step 4: Seed data, tokens, and a tour

Two management commands. The first loads five sample providers (one is `internal`) and is safe to run twice. Create `directory/providers/management/commands/seed_providers.py`:

```py
from django.core.management.base import BaseCommand

from providers.models import Provider

SAMPLE = [
    ("Dr. Ana Reyes", "orthopedic surgery", "San Diego", "CA", "public"),
    ("Dr. Ben Okafor", "cardiology", "Boston", "MA", "public"),
    ("Dr. Chloe Park", "orthopedic surgery", "Los Angeles", "CA", "public"),
    ("Dr. Dev Patel", "dermatology", "Austin", "TX", "public"),
    ("Dr. Eli Stone", "orthopedic surgery", "Sacramento", "CA", "internal"),
]


class Command(BaseCommand):
    help = "Load the sample providers. Safe to run twice."

    def handle(self, *args, **options):
        created = 0
        for name, specialty, city, state, visibility in SAMPLE:
            _, was_created = Provider.objects.get_or_create(
                name=name,
                defaults={
                    "specialty": specialty,
                    "city": city,
                    "state": state,
                    "visibility": visibility,
                },
            )
            created += was_created
        self.stdout.write(f"{created} created, {len(SAMPLE) - created} already there")
```

The second creates a user and a token. The key prints to **stdout** and the warning to **stderr**, so you can capture the key in a shell variable and still see the warning. Create `directory/providers/management/commands/issue_token.py`:

```py
from django.contrib.auth import get_user_model
from django.core.management.base import BaseCommand

from providers.models import ApiToken


class Command(BaseCommand):
    help = "Create an API token for a user (the user is created if it does not exist)."

    def add_arguments(self, parser):
        parser.add_argument("username")
        parser.add_argument("--staff", action="store_true", help="Allow writes and internal rows")
        parser.add_argument("--label", default="")

    def handle(self, *args, username, staff, label, **options):
        user, created = get_user_model().objects.get_or_create(username=username)
        if created:
            user.set_unusable_password()
        if staff:
            user.is_staff = True
        user.save()
        self.stdout.write(ApiToken.issue(user, label))
        self.stderr.write("Copy this token now. It is stored hashed and cannot be shown again.")
```

Run them in the second terminal, while the server from Step 3 keeps running (both use the same database file):

```bash
uv run --env-file .env python manage.py seed_providers
uv run --env-file .env python manage.py seed_providers
READER=$(uv run --env-file .env python manage.py issue_token reader --label tutorial)
ADMIN=$(uv run --env-file .env python manage.py issue_token admin --staff --label tutorial)
echo "$READER" | cut -c1-12
```

Expected: `5 created, 0 already there`, then `0 created, 5 already there`, then the warning twice, then the first 12 characters of a key such as `dir_PixTKPSi`. The variables `READER` and `ADMIN` live only in this terminal. If you close it, issue new tokens.

Now take the tour, in the same terminal. Each request shows one rule from the design.

**4.1 A reader searches.** Two orthopedic providers in California, nobody else:

```bash
curl -s -H "Authorization: Bearer $READER" "http://127.0.0.1:8001/api/providers?q=orthopedic%20CA"
```

```txt
{"items": [{"id": 1, "name": "Dr. Ana Reyes", "specialty": "orthopedic surgery", "city": "San Diego", "state": "CA", "visibility": "public"}, {"id": 3, "name": "Dr. Chloe Park", "specialty": "orthopedic surgery", "city": "Los Angeles", "state": "CA", "visibility": "public"}], "count": 2, "page": 1, "page_size": 20}
```

**4.2 The reader asks for the internal provider directly (id 5).** It looks like it does not exist:

```bash
curl -s -H "Authorization: Bearer $READER" http://127.0.0.1:8001/api/providers/5
```

```txt
{"detail": "Not Found: No Provider matches the given query."}
```

**4.3 Staff see the same search with one more row:**

```bash
curl -s -H "Authorization: Bearer $ADMIN" "http://127.0.0.1:8001/api/providers?q=orthopedic%20CA"
```

The count is 3, and the third provider is Dr. Eli Stone, the internal one.

**4.4 A reader tries to write.** Refused:

```bash
curl -s -X POST -H "Authorization: Bearer $READER" -H 'content-type: application/json' \
  -d '{"name":"Dr. Fay Lin","specialty":"cardiology","city":"Denver","state":"co"}' \
  http://127.0.0.1:8001/api/providers
```

```txt
{"detail": "Staff only"}
```

**4.5 Staff write the same thing.** It works, and `co` is stored as `CO`:

```bash
curl -s -X POST -H "Authorization: Bearer $ADMIN" -H 'content-type: application/json' \
  -d '{"name":"Dr. Fay Lin","specialty":"cardiology","city":"Denver","state":"co"}' \
  http://127.0.0.1:8001/api/providers
```

```txt
{"id": 6, "name": "Dr. Fay Lin", "specialty": "cardiology", "city": "Denver", "state": "CO", "visibility": "public"}
```

**4.6 Bad input is refused with a reason.** A state spelled out in full:

```bash
curl -s -X POST -H "Authorization: Bearer $ADMIN" -H 'content-type: application/json' \
  -d '{"name":"Dr. Gus Roe","specialty":"cardiology","city":"Denver","state":"Colorado"}' \
  http://127.0.0.1:8001/api/providers
```

```txt
{"detail": [{"type": "string_pattern_mismatch", "loc": ["body", "payload", "state"], "msg": "String should match pattern '^[A-Za-z]{2}$'", "ctx": {"pattern": "^[A-Za-z]{2}$"}}]}
```

**4.7 Limits.** A page size of 500 and a missing token:

```bash
curl -s -H "Authorization: Bearer $READER" "http://127.0.0.1:8001/api/providers?page_size=500"
curl -s "http://127.0.0.1:8001/api/providers?page_size=500"
```

```txt
{"detail": [{"type": "less_than_equal", "loc": ["query", "page_size"], "msg": "Input should be less than or equal to 100", "ctx": {"le": 100}}]}
{"detail": "Unauthorized"}
```

The second call shows the order of checks: authentication runs first, so a caller without a token never learns anything about your parameters.

**4.8 Pagination.** The reader sees five providers now (four public originals plus Fay Lin). Page 2 with a page size of 2 holds ids 3 and 4:

```bash
curl -s -H "Authorization: Bearer $READER" "http://127.0.0.1:8001/api/providers?page_size=2&page=2"
```

The response has `"count": 5`, `"page": 2` and two items.

Your API works. Stop the server with Ctrl+C.

---

## Step 5: Tests

Tests prove the rules in Step 3 hold for every future change. They use a test database that Django creates and throws away, so they never touch `db.sqlite3`, and they need no network and no server. Add them one file at a time and check the count each time.

**5.1 Fixtures and model tests.** `conftest.py` holds the shared setup: sample providers, and a client for a reader, a staff user and an anonymous caller. Each client sends a real token through the real authentication code. Create `directory/tests/conftest.py`:

```py
import pytest
from django.contrib.auth import get_user_model
from django.core.cache import cache
from ninja.testing import TestClient

from providers.api import api
from providers.models import ApiToken, Provider


@pytest.fixture(autouse=True)
def _fresh_throttle_counters():
    """Throttle counts live in the cache; start every test from zero."""
    cache.clear()


@pytest.fixture
def providers(db):
    rows = [
        ("Dr. Ana Reyes", "orthopedic surgery", "San Diego", "CA", "public"),
        ("Dr. Ben Okafor", "cardiology", "Boston", "MA", "public"),
        ("Dr. Chloe Park", "orthopedic surgery", "Los Angeles", "CA", "public"),
        ("Dr. Eli Stone", "orthopedic surgery", "Sacramento", "CA", "internal"),
    ]
    return [
        Provider.objects.create(
            name=n, specialty=s, city=c, state=st, visibility=v
        )
        for n, s, c, st, v in rows
    ]


def _client_for(username: str, *, staff: bool) -> TestClient:
    user = get_user_model().objects.create(username=username, is_staff=staff)
    raw = ApiToken.issue(user, "test")
    return TestClient(api, headers={"Authorization": f"Bearer {raw}"})


@pytest.fixture
def reader(db):
    return _client_for("reader", staff=False)


@pytest.fixture
def admin(db):
    return _client_for("admin", staff=True)


@pytest.fixture
def anonymous():
    return TestClient(api)
```

Create `directory/tests/test_models.py`:

```py
from django.contrib.auth import get_user_model

from providers.models import ApiToken, Provider, hash_key


def test_search_requires_every_word(providers):
    names = {p.name for p in Provider.objects.search("orthopedic CA")}
    assert names == {"Dr. Ana Reyes", "Dr. Chloe Park", "Dr. Eli Stone"}


def test_search_is_case_insensitive_and_spans_fields(providers):
    assert [p.name for p in Provider.objects.search("BOSTON cardiology")] == ["Dr. Ben Okafor"]


def test_blank_search_returns_everything(providers):
    assert Provider.objects.search("").count() == 4


def test_visibility_hides_internal_rows_from_non_staff(providers):
    User = get_user_model()
    plain = User(username="p", is_staff=False)
    staff = User(username="s", is_staff=True)
    assert Provider.objects.visible_to(plain).count() == 3
    assert Provider.objects.visible_to(staff).count() == 4


def test_token_is_stored_hashed(db):
    user = get_user_model().objects.create(username="u")
    raw = ApiToken.issue(user)
    row = ApiToken.objects.get()
    assert raw.startswith("dir_")
    assert row.key_hash == hash_key(raw)
    assert raw not in row.key_hash
```

```bash
uv run --env-file .env pytest -q
```

Check: `5 passed`.

**5.2 API tests.** Django Ninja's `TestClient` calls your endpoints directly, without a server. The last test uses Django's own client instead, which goes through `urls.py` and the middleware like a real request, and checks that the OpenAPI schema lists your paths. Create `directory/tests/test_api.py`:

```py
from django.contrib.auth import get_user_model
from ninja.testing import TestClient

from providers.api import api
from providers.models import ApiToken, Provider


def test_health_needs_no_token(anonymous):
    assert anonymous.get("/health").json() == {"status": "ok"}


def test_missing_token_is_401(anonymous):
    assert anonymous.get("/providers").status_code == 401


def test_wrong_token_is_401(db):
    bad = TestClient(api, headers={"Authorization": "Bearer dir_nope"})
    assert bad.get("/providers").status_code == 401


def test_revoked_token_is_401(db):
    user = get_user_model().objects.create(username="gone")
    raw = ApiToken.issue(user)
    ApiToken.objects.get().revoke()
    client = TestClient(api, headers={"Authorization": f"Bearer {raw}"})
    assert client.get("/providers").status_code == 401


def test_inactive_user_is_401(db):
    user = get_user_model().objects.create(username="off", is_active=False)
    raw = ApiToken.issue(user)
    client = TestClient(api, headers={"Authorization": f"Bearer {raw}"})
    assert client.get("/providers").status_code == 401


def test_search_returns_a_page(providers, reader):
    body = reader.get("/providers", query_params={"q": "orthopedic CA"}).json()
    assert [p["name"] for p in body["items"]] == ["Dr. Ana Reyes", "Dr. Chloe Park"]
    assert body["count"] == 2
    assert (body["page"], body["page_size"]) == (1, 20)


def test_staff_sees_internal_rows(providers, admin):
    body = admin.get("/providers", query_params={"q": "orthopedic CA"}).json()
    assert body["count"] == 3


def test_state_filter(providers, reader):
    body = reader.get("/providers", query_params={"state": "ma"}).json()
    assert [p["name"] for p in body["items"]] == ["Dr. Ben Okafor"]


def test_pagination_slices_and_counts_the_whole_set(providers, reader):
    first = reader.get("/providers", query_params={"page_size": 2}).json()
    second = reader.get("/providers", query_params={"page_size": 2, "page": 2}).json()
    assert len(first["items"]) == 2 and len(second["items"]) == 1
    assert first["count"] == second["count"] == 3


def test_page_size_is_capped(providers, reader):
    assert reader.get("/providers", query_params={"page_size": 500}).status_code == 422
    assert reader.get("/providers", query_params={"page": 0}).status_code == 422


def test_get_one(providers, reader):
    got = reader.get(f"/providers/{providers[0].id}").json()
    assert got["name"] == "Dr. Ana Reyes"


def test_hidden_row_looks_missing_to_a_reader(providers, reader):
    internal = providers[3]
    assert reader.get(f"/providers/{internal.id}").status_code == 404


def test_unknown_id_is_404(providers, reader):
    assert reader.get("/providers/99999").status_code == 404


def test_reader_cannot_write(providers, reader):
    payload = {"name": "X", "specialty": "y", "city": "z", "state": "tx"}
    assert reader.post("/providers", json=payload).status_code == 403
    assert reader.patch(f"/providers/{providers[0].id}", json={"city": "Reno"}).status_code == 403
    assert reader.delete(f"/providers/{providers[0].id}").status_code == 403
    assert Provider.objects.count() == 4


def test_staff_creates_and_state_is_uppercased(db, admin):
    payload = {"name": "Dr. Fay Lin", "specialty": "cardiology", "city": "Denver", "state": "co"}
    res = admin.post("/providers", json=payload)
    assert res.status_code == 201
    assert res.json()["state"] == "CO"
    assert Provider.objects.get(name="Dr. Fay Lin").state == "CO"


def test_create_validates_input(db, admin):
    bad_state = {"name": "A", "specialty": "b", "city": "c", "state": "California"}
    assert admin.post("/providers", json=bad_state).status_code == 422
    assert admin.post("/providers", json={"name": ""}).status_code == 422


def test_patch_changes_only_the_fields_sent(providers, admin):
    target = providers[0]
    res = admin.patch(f"/providers/{target.id}", json={"city": "Reno"})
    assert res.status_code == 200
    target.refresh_from_db()
    assert (target.city, target.name, target.state) == ("Reno", "Dr. Ana Reyes", "CA")


def test_staff_deletes(providers, admin):
    target = providers[0]
    assert admin.delete(f"/providers/{target.id}").status_code == 204
    assert not Provider.objects.filter(id=target.id).exists()


def test_real_urls_and_openapi_schema(providers, client):
    """Django's own test client goes through urls.py and middleware, the way a real request does."""
    assert client.get("/api/health").json() == {"status": "ok"}
    assert client.get("/api/providers").status_code == 401
    paths = client.get("/api/openapi.json").json()["paths"]
    assert "/api/providers" in paths and "/api/providers/{provider_id}" in paths
```

```bash
uv run --env-file .env pytest -q
```

Check: `24 passed` (5 plus 19).

**5.3 Command tests.** Create `directory/tests/test_commands.py`:

```py
from io import StringIO

from django.contrib.auth import get_user_model
from django.core.management import call_command

from providers.models import ApiToken, Provider


def test_seed_is_idempotent(db):
    call_command("seed_providers", stdout=StringIO())
    call_command("seed_providers", stdout=StringIO())
    assert Provider.objects.count() == 5


def test_issue_token_prints_a_working_key_once(db):
    out = StringIO()
    call_command("issue_token", "newbie", "--staff", stdout=out, stderr=StringIO())
    raw = out.getvalue().strip()
    user = get_user_model().objects.get(username="newbie")
    assert user.is_staff and not user.has_usable_password()
    assert ApiToken.objects.get(user=user).key_hash != raw
```

Check: `26 passed`.

**5.4 Throttle tests.** The rate limits from Step 3 are only real if something fails when they break. These tests lower the limit to 2 for the duration of one test, so they run instantly. Create `directory/tests/test_throttle.py`:

```py
from ninja.throttling import AnonRateThrottle, AuthRateThrottle

from providers.api import api


def _limit(throttle_class, n, monkeypatch):
    for t in api.throttle:
        if isinstance(t, throttle_class):
            monkeypatch.setattr(t, "num_requests", n)


def test_open_endpoints_are_limited_per_ip(anonymous, monkeypatch):
    _limit(AnonRateThrottle, 2, monkeypatch)
    assert [anonymous.get("/health").status_code for _ in range(4)] == [200, 200, 429, 429]


def test_bad_tokens_are_rejected_before_any_throttle_runs(anonymous, monkeypatch):
    """Auth runs first, so a flood of bad tokens is never throttled here. Limit it at the proxy."""
    _limit(AnonRateThrottle, 1, monkeypatch)
    assert [anonymous.get("/providers").status_code for _ in range(4)] == [401] * 4


def test_a_valid_token_is_not_counted_against_the_anonymous_limit(providers, reader, monkeypatch):
    _limit(AnonRateThrottle, 1, monkeypatch)
    assert [reader.get("/providers").status_code for _ in range(5)] == [200] * 5


def test_each_token_owner_has_their_own_budget(providers, reader, admin, monkeypatch):
    _limit(AuthRateThrottle, 2, monkeypatch)
    assert [reader.get("/providers").status_code for _ in range(3)] == [200, 200, 429]
    assert admin.get("/providers").status_code == 200
```

```bash
uv run --env-file .env pytest -q -W error
uv run ruff check .
```

Check: `30 passed` and `All checks passed!`. The `-W error` flag turns warnings into failures, which is how the deprecated tuple return from older tutorials would have shown up.

> [!NOTE]
> **Read the second throttle test.** Django Ninja runs authentication before throttling, so a request with a bad token gets a 401 and is never counted. The "anonymous" limit therefore only protects endpoints that opt out of auth, such as `/health`. A flood of bad tokens is not slowed down by it. The random 256-bit keys make guessing hopeless, but each bad attempt still costs a database lookup. Limit that at your proxy or load balancer. I found this by writing the obvious test and watching it fail.

---

## Step 6: Make it real

**Check the deployment settings.** Django has a built-in audit. Run it with production values:

```bash
DJANGO_DEBUG=0 DJANGO_SECRET_KEY="$(python3 -c 'import secrets; print(secrets.token_urlsafe(60))')" \
  uv run python manage.py check --deploy
```

Expected: two warnings, about `SECURE_HSTS_INCLUDE_SUBDOMAINS` and `SECURE_HSTS_PRELOAD`, and nothing else. Those two are decisions only you can make, because they affect every subdomain of your domain. Read what each does before turning it on. With the settings from Step 1, everything else Django checks is already on.

**Serve it with a real server.** `runserver` is for development. Add gunicorn and run the WSGI app:

```bash
uv add "gunicorn>=23"
uv run --env-file .env gunicorn config.wsgi --bind 127.0.0.1:8002 --workers 2
```

In another terminal, `curl -s http://127.0.0.1:8002/api/health` should print `{"status": "ok"}`.

**Production environment.** Set these as real environment variables on the host, not in a `.env` file you ship:

| Variable | Value |
| --- | --- |
| `DJANGO_DEBUG` | `0` |
| `DJANGO_SECRET_KEY` | a long random string, different from every other environment |
| `DJANGO_ALLOWED_HOSTS` | your domain, for example `api.example.com` |
| `SMTP_HOST` | only if you add features that send email |

With `DJANGO_DEBUG=0` the settings turn on the HTTPS redirect, secure cookies and HSTS, and refuse to start without a secret key.

**Other things to change before real users arrive:**

- **Use a real database.** SQLite is fine for this chapter. For production, set `DATABASES` from an environment variable and use PostgreSQL. Django 6.1 dropped support for PostgreSQL 14, so use 15 or newer. The models here use nothing SQLite-specific, but this chapter did not test PostgreSQL.
- **Run migrations on deploy** with `python manage.py migrate`, as a separate step before the new code takes traffic.
- **Add `/api/health` to your host's health check**, and keep the rest behind HTTPS.
- **Give each caller its own token.** The MCP server in chapter 2 will hold one. Issue it with a descriptive `--label`, and revoke it from the admin (set `revoked_at`) the moment it leaks.
- **Plan for the admin.** `/admin/` is a login page on a public URL. Restrict it by network, or remove it from `urls.py` if you manage data another way.

---

## What you hand to chapter 2

Chapter 2 builds an MCP server whose tools call this API. It needs three things from you:

| Chapter 2 setting | Value |
| --- | --- |
| `DIRECTORY_API_URL` | `http://127.0.0.1:8001/api` |
| `DIRECTORY_API_TOKEN` | a token from `issue_token`, for a non-staff user |
| The response shape | `{"items": [...], "count": n, "page": n, "page_size": n}` and `{"detail": "..."}` for errors |

Give the MCP server a **non-staff** token. Then the model can never see internal providers or write anything, even if someone tricks it into trying. That is the point of putting the rules here.

## Production checklist

- [ ] Every endpoint requires a token unless it opts out on purpose
- [ ] Tokens stored hashed, revocable, one per caller, with a label
- [ ] Authorization is applied in the query, not after it
- [ ] Hidden rows look missing, not forbidden
- [ ] Input limits and formats declared in the types
- [ ] Writes restricted to staff
- [ ] Rate limits set, and bad-token floods limited at the proxy
- [ ] `check --deploy` clean apart from decisions you made on purpose
- [ ] `DJANGO_DEBUG=0` and a real `DJANGO_SECRET_KEY` in production
- [ ] A real database, with migrations run on deploy
- [ ] Tests run in CI on every change, with `-W error`

## What is intentionally missing

- **Token scopes.** Every token has its owner's full rights. Real systems add scopes such as `providers:read`.
- **Token expiry and rotation.** Tokens live until revoked.
- **Filtering and sorting beyond `q` and `state`.** Add them as query parameters the same way.
- **Per-caller forwarding from the MCP server.** Chapter 2 uses one service token. Passing the end user's own token through is the production version, noted there.
- **Async views.** Everything here is synchronous. Django Ninja supports `async def` endpoints when you need them.

## What was verified

All 30 tests pass with warnings treated as errors and ruff is clean, run in a fresh folder by adding these files one step at a time, with the per-step counts shown above. The Step 4 outputs are real output from a running server on a freshly migrated database. `check --deploy` was run with production settings, and gunicorn served `/api/health`, `/admin/login/` and `/api/docs` with 200. The API was also called end to end by the chapter 2 MCP server over HTTP.

Not verified: PostgreSQL, deployment to a real host behind TLS, Windows commands, and the admin site beyond loading its login page.
