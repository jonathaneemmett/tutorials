# Python AI Native: Build an MCP Server, Agent and Webhook (Step by Step)

> [!NOTE]
> Chapter 2 of [Python AI Native: Build Production AI](../README.md). Previous: [Python AI Native: Build the API First with Django (Step by Step)](01-build-the-api-first-with-django.md), which this chapter calls. Next: Build a Deep Agent from Scratch.

> [!NOTE]
> Build an AI-native Python service step by step: an MCP server whose tools call the Django API from chapter 1, an agent that uses those tools through **OpenRouter** (one API key, any model, Fable by default), and a webhook automation that acts on a model verdict. Every file below was built and run: lint clean, 14 tests passing, and the whole chain (Django API, MCP server, MCP client) was exercised over HTTP, all on current releases (mcp 2.3.0, openai 3.46.0).

## What you will build

| Step | You build |
| --- | --- |
| 0 | Install the tools, start the chapter 1 API and get a token |
| 1 | Project setup |
| 2 | Data layer that calls the API |
| 3 | MCP server with two tools |
| 4 | Run and inspect the server |
| 5 | One shared OpenRouter client, then the agent loop |
| 6 | Webhook automation |
| 7 | Offline tests |
| 8 | Run it end to end |
| 9 | Go to production |

**What an MCP server adds to an API.** The chapter 1 API answers programs. A model needs something different: a short description of each tool, answers as plain text, and errors it can read and act on. The MCP server is that translation layer. The rules (who may see what, what is valid) stay in the API, where the model cannot talk its way around them.

**Why OpenRouter?** It exposes many models (Anthropic, OpenAI, Google and others) behind one OpenAI-compatible API and one key. Switching or comparing models becomes a one-line environment change, not a rewrite. We use the standard `openai` Python SDK pointed at `https://openrouter.ai/api/v1`.

## Step 0: Install everything

You need five things: the **chapter 1 API** (running), **uv** (installs Python and dependencies), **Python 3.11+**, **Node.js** (only for the MCP Inspector in step 4), and an **OpenRouter API key**. Do each sub-step in order and run its check before moving on.

### 0.1 Open a terminal

- macOS: Terminal (or iTerm)
- Windows: PowerShell
- Linux: any shell

### 0.2 Install uv

uv handles Python versions, virtual environments and dependencies in one tool. If you did chapter 1 you already have it, so just run the check.

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

Expected: a version number such as `uv 0.13.7`. If you see `command not found`, the install directory is not on your `PATH`. Reopen the terminal, or add `~/.local/bin` to your `PATH`.

### 0.3 Install Python 3.11 or newer

You do not need to install Python yourself. Let uv do it:

```bash
uv python install 3.13
uv python list --only-installed
```

Expected: a `cpython-3.13.x` line in the list. Any 3.11 or newer works for this chapter (current `starlette` requires 3.11). Chapter 1's Django needs 3.12 or newer, so one 3.13 install serves both.

### 0.4 Install Node.js (for the MCP Inspector only)

```bash
brew install node        # macOS
# Windows / Linux: download the LTS installer from https://nodejs.org
node --version && npx --version
```

Expected: two version numbers. Skip this sub-step if you only want to run the Python code. The Inspector in step 4 is a visual debugging tool and optional.

### 0.5 Get an OpenRouter API key

1. Create an account at [openrouter.ai](https://openrouter.ai) and add a few dollars of credit.
2. Open **Keys** and click **Create Key**.
3. Copy the key (it starts with `sk-or-`). You will not see it again.
4. Open the model list and confirm the model you want is available to your account. The default in this tutorial is `anthropic/claude-fable-5-1`. If that slug differs on OpenRouter, copy the exact slug from the model's page and use it in step 5.

Keep the key handy. In Step 1 you will save it in a `.env` file that stays out of git. Never paste it into code or commit it.

### 0.6 Start the chapter 1 API and get a token

This chapter's tools call the provider directory API you built in chapter 1. If you have not built it, do chapter 1 first.

In the `directory/` folder from chapter 1, start the API on port 8001 (port 8000 belongs to the MCP server in this chapter) and leave it running:

```bash
cd directory
uv run --env-file .env python manage.py seed_providers
uv run --env-file .env python manage.py runserver 8001
```

In a second terminal, in the same `directory/` folder, issue a token for the MCP server. Use a **non-staff** user, so the model can only see public providers and can never write:

```bash
uv run --env-file .env python manage.py issue_token mcp-server --label "chapter 2 mcp server"
```

Expected: one line starting with `dir_`, and a warning on the next line that it cannot be shown again. Copy the `dir_...` line. You will paste it into `.env` in Step 1.

Check that the API answers with your token (replace the placeholder):

```bash
curl -s -H "Authorization: Bearer dir_paste-your-token-here" "http://127.0.0.1:8001/api/providers?q=orthopedic"
```

Expected: a JSON object with an `items` list and a `count`. A `{"detail": "Unauthorized"}` means the token was pasted wrong.

### 0.7 Optional: curl and openssl

Step 6 sends a signed test request with `curl` and `openssl`. Both ship with macOS and most Linux systems. Check with `curl --version && openssl version`. On Windows, use Git Bash or WSL for that one command.

### 0.8 Installation checklist

- [ ] `uv --version` prints a version
- [ ] `uv python list --only-installed` shows Python 3.11 or newer
- [ ] `node --version` prints a version (optional)
- [ ] The chapter 1 API is running on port 8001 and your token returned a JSON page
- [ ] You have copied an OpenRouter key (steps 5, 6, 8 only)

Step 7 (the tests) needs nothing running. Steps 4 and 8 need the API running. Only the live runs in steps 5, 6 and 8 need the OpenRouter key.

## Step 1: Create the project

Open a **new terminal**, next to your `directory/` folder, not inside it:

```bash
mkdir ainative && cd ainative
mkdir -p src/ainative tests
touch src/ainative/__init__.py
```

By the end, the project looks like this. **Every path in this tutorial is relative to the `ainative/` folder (the project root)**, and every command runs from there unless it says otherwise. Each file below is introduced with its full path starting at `ainative/`.

```txt
ainative/
├── .env
├── .gitignore
├── pyproject.toml
├── src/
│   └── ainative/
│       ├── __init__.py
│       ├── data.py
│       ├── server.py
│       ├── llm.py
│       ├── agent.py
│       └── webhook.py
└── tests/
    └── test_all.py
```

First, create two files directly in the project root, next to where `pyproject.toml` will go: `ainative/.env` holds your settings and `ainative/.gitignore` keeps them out of git. The commands below create both (you are already inside `ainative/`):

```bash
cat > .env <<'EOF'
OPENROUTER_API_KEY=sk-or-paste-your-key-here
OPENROUTER_MODEL=anthropic/claude-fable-5-1
WEBHOOK_SECRET=dev-secret
DIRECTORY_API_URL=http://127.0.0.1:8001/api
DIRECTORY_API_TOKEN=dir_paste-your-token-here
EOF
printf '.env\n.venv/\n__pycache__/\n' > .gitignore
```

On Windows, create `.env` and `.gitignore` in your editor, saved in the `ainative` folder, with the same contents. Open `.env` and replace both placeholders: the OpenRouter key from 0.5 and the `dir_...` token from 0.6. From here, every `uv run` command that needs settings uses `--env-file .env`, so there is nothing to export and nothing to re-set in a new terminal. In production you set real environment variables on the host instead of shipping a `.env` file.

Create `ainative/pyproject.toml` (project root):

```toml
[project]
name = "ainative"
version = "0.1.0"
description = "Python AI-native tutorial: MCP server over the directory API, agent, webhook"
requires-python = ">=3.11"
dependencies = [
    "mcp>=2.3,<3",
    "openai>=3.0,<4",
    "httpx>=0.28",
    "starlette>=1.0",
    "uvicorn>=0.30",
]

[project.optional-dependencies]
dev = ["pytest>=9.0", "pytest-asyncio>=1.0", "ruff>=0.5.6"]

[project.scripts]
ainative-server = "ainative.server:main"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/ainative"]

[tool.pytest.ini_options]
asyncio_mode = "auto"
asyncio_default_fixture_loop_scope = "function"

[tool.ruff.lint]
select = ["E", "F", "I", "UP"]
ignore = ["E501"]
```

> [!WARNING]
> This tutorial targets the current major versions: `mcp` 2.x (`MCPServer` and `Client`, snake_case fields such as `input_schema`) and `openai` 3.x. Tested with mcp 2.3.0, openai 3.46.0 and Python 3.13. Older tutorials that import `FastMCP` or `ClientSession` target mcp 1.x.

Install the project's dependencies (this creates a `.venv` and a `uv.lock` for you):

```bash
uv sync --extra dev
```

Expected: a list of installed packages including `+ mcp==2.3.0` and `+ openai==3.46.0` (or newer 2.x and 3.x releases). Then confirm the tools import:

```bash
uv run python -c "from mcp.server import MCPServer; import openai; print('imports ok')"
```

Expected: `imports ok`. If you see `cannot import name 'MCPServer'`, you have an old mcp 1.x. Re-check the `mcp>=2.3,<3` line in `pyproject.toml` and run `uv sync --extra dev` again.

All commands from here run from the `ainative` folder and use `uv run`, so you never have to activate the virtual environment by hand.

## Step 2: The data layer

The data layer is the only code that knows the API exists. Everything above it asks for providers and gets plain Python back. It does four jobs:

1. Builds one HTTP client with the base URL and your bearer token.
2. Turns every way the API can fail into a short sentence the model can read.
3. Returns `None` for "not found", which is a normal answer, not an error.
4. Never retries on its own. A rate limit or an outage is reported, and the model decides what to do.

Create `ainative/src/ainative/data.py`:

```py
"""The data layer: talks to the Provider Directory API you built in chapter 1."""
import os

import httpx


class DirectoryError(Exception):
    """Something the model can be told about in plain words."""


class Directory:
    def __init__(self, client: httpx.AsyncClient):
        self.client = client

    @classmethod
    def from_env(cls) -> "Directory":
        try:
            token = os.environ["DIRECTORY_API_TOKEN"]
        except KeyError:
            raise DirectoryError("DIRECTORY_API_TOKEN is not set") from None
        return cls(
            httpx.AsyncClient(
                base_url=os.environ.get("DIRECTORY_API_URL", "http://127.0.0.1:8001/api"),
                headers={"Authorization": f"Bearer {token}"},
                timeout=10.0,
            )
        )

    async def _get(self, path: str, **params) -> httpx.Response:
        try:
            res = await self.client.get(path, params=params)
        except httpx.TransportError:
            raise DirectoryError("the directory API is not reachable") from None
        if res.status_code in (401, 403):
            raise DirectoryError("the directory API rejected our token")
        if res.status_code == 429:
            raise DirectoryError("the directory API is rate limiting us; try again shortly")
        if res.status_code >= 500:
            raise DirectoryError(f"the directory API failed ({res.status_code})")
        return res

    async def search(self, query: str, page: int, page_size: int) -> dict:
        res = await self._get("/providers", q=query, page=page, page_size=page_size)
        res.raise_for_status()
        return res.json()  # {"items": [...], "count": n, "page": n, "page_size": n}

    async def get(self, provider_id: int) -> dict | None:
        res = await self._get(f"/providers/{provider_id}")
        if res.status_code == 404:
            return None
        res.raise_for_status()
        return res.json()
```

Notice what is **not** here: no filtering, no permission checks, no validation of the data. The API already did all of that. If a model tricks the tool into asking for something it should not have, the API says no.

## Step 3: The MCP server

Tools are plain async functions. **The docstring is the prompt**: the model reads it to decide when and how to call the tool. Create `ainative/src/ainative/server.py`:

```py
"""A minimal MCP server exposing provider tools."""
from mcp.server import MCPServer

from ainative.data import Directory, DirectoryError

mcp = MCPServer("ainative-demo")

_directory: Directory | None = None


def get_directory() -> Directory:
    """Built on first use, so importing this module needs no settings."""
    global _directory
    if _directory is None:
        _directory = Directory.from_env()
    return _directory


def set_directory(directory: Directory | None) -> None:
    """Tests swap in a directory that talks to a fake API."""
    global _directory
    _directory = directory


def _fmt(p: dict) -> str:
    return f"[{p['id']}] {p['name']} | {p['specialty']} | {p['city']}, {p['state']}"


@mcp.tool()
async def search_providers(query: str, page: int = 1, page_size: int = 20) -> str:
    """Search healthcare providers by name, specialty, city or state.
    Examples: 'orthopedic surgery CA', 'cardiology Boston'.
    Returns matching providers with their IDs.

    Args:
        query: Search text (minimum 3 characters).
        page: Page number (default 1).
        page_size: Results per page (default 20, max 100).
    """
    try:
        query = query.strip()
        if len(query) < 3:
            return "Error: query must be at least 3 characters long."
        page, page_size = max(page, 1), min(max(page_size, 1), 100)
        result = await get_directory().search(query, page, page_size)
        if not result["items"]:
            return f"No providers found for '{query}'."
        lines = [_fmt(p) for p in result["items"]]
        shown = (page - 1) * page_size + len(result["items"])
        if result["count"] > shown:
            lines.append(f"({shown} of {result['count']} shown. Ask for page {page + 1} for more.)")
        return "\n".join(lines)
    except DirectoryError as e:
        return f"Error: {e}."
    except Exception as e:  # tools return errors, they never raise
        return f"Error searching providers: {e}"


@mcp.tool()
async def get_provider_details(provider_id: int) -> str:
    """Get full details for one provider by the ID returned from search_providers."""
    try:
        p = await get_directory().get(provider_id)
        return _fmt(p) if p else f"Error: no provider with id {provider_id}."
    except DirectoryError as e:
        return f"Error: {e}."
    except Exception as e:
        return f"Error fetching provider: {e}"


def main() -> None:
    mcp.run(transport="streamable-http")  # http://127.0.0.1:8000/mcp


if __name__ == "__main__":
    main()
```

Why it is written this way:

- **Return strings, never raise.** The model can read `Error: ...` and recover. A stack trace ends the turn.
- **Validate and clamp inside the tool** (min 3 chars, `page_size <= 100`). The API enforces the same limits, but a clamped request saves a round trip, and the model gets a clear message.
- **Format for the model**: short, labelled lines that include IDs so the next tool call can use them. When there is more than one page, the last line says how many are shown and which page to ask for next. Without it the model believes the first page is everything.
- **Build the data layer lazily.** `get_directory()` creates the client on first use, so importing the module never needs a token, and tests swap in a fake with `set_directory`.
- **Two kinds of error.** A `DirectoryError` carries a sentence written for the model. Anything else is a bug, and is reported too, but never raised.

## Step 4: Run and inspect the server

The chapter 1 API must still be running on port 8001. The server reads the API URL and token from `.env`:

```bash
uv run --env-file .env ainative-server
```

In a second terminal:

```bash
npx @modelcontextprotocol/inspector
```

Choose **Streamable HTTP**, enter `http://127.0.0.1:8000/mcp`, click **Connect**, then **List Tools**. Run `search_providers` with `{"query": "orthopedic CA"}`. Expected output:

```txt
[1] Dr. Ana Reyes | orthopedic surgery | San Diego, CA
[3] Dr. Chloe Park | orthopedic surgery | Los Angeles, CA
```

Now try `get_provider_details` with `{"provider_id": 5}`. The reply is `Error: no provider with id 5.` Provider 5 exists, but it is marked internal, and your token belongs to a non-staff user. The API hides it, so to the model it does not exist. This is the point of chapter 1.

Stop the API (Ctrl+C in its terminal) and run `search_providers` again. The reply is `Error: the directory API is not reachable.` Start the API again. Try a deliberately wrong `DIRECTORY_API_TOKEN` too and you get `Error: the directory API rejected our token.`

> [!NOTE]
> The MCP server itself has no authentication. That is fine on [localhost](http://localhost) only. Before exposing it on a network, add auth (see step 9).

## Step 5: One OpenRouter client, then the agent loop

### 5.1 The shared client

Put all OpenRouter configuration in one file. Every other module calls `make_client()` and reads `MODEL`, so the base URL, key and headers can never drift between call sites. Create `ainative/src/ainative/llm.py`:

```py
"""The one place that knows how to talk to OpenRouter (OpenAI-compatible API)."""
import os

from openai import AsyncOpenAI

BASE_URL = "https://openrouter.ai/api/v1"
# One knob for the model. Change it in the environment, not in code.
MODEL = os.environ.get("OPENROUTER_MODEL", "anthropic/claude-fable-5-1")


def make_client() -> AsyncOpenAI:
    """Every client goes through here so key, base URL and headers can't drift."""
    return AsyncOpenAI(
        base_url=BASE_URL,
        api_key=os.environ["OPENROUTER_API_KEY"],
        default_headers={"X-Title": "ainative-tutorial"},
    )
```

To try another model, change one line in `.env`, no code edit (copy exact slugs from [openrouter.ai/models](http://openrouter.ai/models)):

```txt
OPENROUTER_MODEL=anthropic/claude-sonnet-4.5
```

### 5.2 The agent

The model decides, the tools do. OpenRouter speaks the OpenAI chat format: tools are `{"type": "function", ...}`, the model returns `tool_calls`, and you reply with `role: "tool"` messages. Create `ainative/src/ainative/agent.py`:

```py
"""A tool-using agent: the model decides, MCP tools do."""
import asyncio
import json
import sys

from mcp import Client

from ainative import llm

MAX_ROUNDS = 10
SYSTEM = "You are a provider-directory assistant. Use the tools to answer; never guess provider data."


def to_openai_tools(listed) -> list[dict]:
    return [
        {
            "type": "function",
            "function": {
                "name": t.name,
                "description": t.description or "",
                "parameters": t.input_schema,
            },
        }
        for t in listed.tools
    ]


def _text(result) -> str:
    return "".join(c.text for c in result.content if getattr(c, "type", "") == "text")


async def run_agent(question: str, session, client, tools: list[dict]) -> str:
    messages = [
        {"role": "system", "content": SYSTEM},
        {"role": "user", "content": question},
    ]
    for _ in range(MAX_ROUNDS):  # hard cap: never loop forever
        resp = await client.chat.completions.create(
            model=llm.MODEL, max_tokens=2048, tools=tools, messages=messages
        )
        msg = resp.choices[0].message
        if not msg.tool_calls:
            return msg.content or ""
        messages.append({
            "role": "assistant",
            "content": msg.content,
            "tool_calls": [
                {
                    "id": c.id,
                    "type": "function",
                    "function": {"name": c.function.name, "arguments": c.function.arguments},
                }
                for c in msg.tool_calls
            ],
        })
        for call in msg.tool_calls:
            try:
                args = json.loads(call.function.arguments or "{}")
            except json.JSONDecodeError:
                args = {}
            out = await session.call_tool(call.function.name, args)
            messages.append({"role": "tool", "tool_call_id": call.id, "content": _text(out)})
    return "Stopped: too many tool rounds."


async def main(question: str, url: str = "http://127.0.0.1:8000/mcp") -> None:
    client = llm.make_client()  # reads OPENROUTER_API_KEY
    async with Client(url) as session:  # connects and initializes for you
        tools = to_openai_tools(await session.list_tools())
        print(await run_agent(question, session, client, tools))


if __name__ == "__main__":
    asyncio.run(main(" ".join(sys.argv[1:]) or "Find orthopedic surgeons in CA"))
```

How the loop works:

1. List the server's tools and convert them to the OpenAI tool format.
2. Send the question. If the reply has `tool_calls`, run each one through the MCP session.
3. Append the assistant message and one `role: "tool"` message per result, then call the model again.
4. Stop when the model answers without tool calls. `MAX_ROUNDS` caps runaway loops.
5. Malformed tool arguments fall back to `{}` instead of crashing. Never trust model output.
6. `run_agent` takes the session and client as arguments so tests can inject fakes.

Run it (the API from chapter 1 and the server from step 4 must both be running):

```bash
uv run --env-file .env python -m ainative.agent "Who are the orthopedic surgeons in California?"
```

## Step 6: Webhook automation

An event arrives, the model judges it, and plain code acts. This example triages support tickets. Create `ainative/src/ainative/webhook.py`:

```py
"""Event-driven automation: webhook -> model verdict -> action."""
import hashlib
import hmac
import json
import os

from starlette.applications import Starlette
from starlette.requests import Request
from starlette.responses import JSONResponse
from starlette.routing import Route

from ainative import llm

VERDICT_TOOL = {
    "type": "function",
    "function": {
        "name": "submit_verdict",
        "description": "Report whether the ticket has enough detail to investigate.",
        "parameters": {
            "type": "object",
            "properties": {
                "complete": {"type": "boolean"},
                "missing": {"type": "array", "items": {"type": "string"}},
            },
            "required": ["complete", "missing"],
        },
    },
}


def verify_signature(body: bytes, signature: str, secret: str) -> bool:
    expected = hmac.new(secret.encode(), body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, signature or "")


async def evaluate(client, title: str, description: str) -> dict:
    resp = await client.chat.completions.create(
        model=llm.MODEL,
        max_tokens=512,
        tools=[VERDICT_TOOL],
        tool_choice={"type": "function", "function": {"name": "submit_verdict"}},
        messages=[{
            "role": "user",
            "content": f"Is this ticket complete enough to investigate?\n\nTitle: {title}\n\n{description}",
        }],
    )
    call = resp.choices[0].message.tool_calls[0]
    try:
        v = json.loads(call.function.arguments)
    except json.JSONDecodeError:
        v = {}
    # shape-guard: never trust model output
    return {"complete": bool(v.get("complete")), "missing": [str(m) for m in v.get("missing", [])]}


def make_app(client=None, secret: str | None = None, act=None) -> Starlette:
    client = client or llm.make_client()
    secret = secret or os.environ.get("WEBHOOK_SECRET", "")
    seen: set[str] = set()  # idempotency (use a DB in production)

    async def default_act(issue_id: str, verdict: dict) -> None:
        print(f"{issue_id}: {verdict}")

    act = act or default_act

    async def handle(request: Request) -> JSONResponse:
        body = await request.body()
        if not verify_signature(body, request.headers.get("x-signature", ""), secret):
            return JSONResponse({"error": "bad signature"}, status_code=401)
        event = json.loads(body)
        issue = event.get("data", {})
        if event.get("source") != "asks":  # cheap filter before spending tokens
            return JSONResponse({"ok": True, "skipped": True})
        if issue["id"] in seen:
            return JSONResponse({"ok": True, "duplicate": True})
        seen.add(issue["id"])
        verdict = await evaluate(client, issue.get("title", ""), issue.get("description", ""))
        await act(issue["id"], verdict)
        return JSONResponse({"ok": True, "verdict": verdict})

    return Starlette(routes=[Route("/webhook", handle, methods=["POST"])])


def build() -> Starlette:
    return make_app()
```

The five rules this encodes:

1. **Verify the signature first.** Webhooks are public endpoints.
2. **Filter before you spend tokens.** Only `asks` events reach the model.
3. **Force structured output.** `tool_choice` pins the model to `submit_verdict`, so you get JSON that matches a schema, not prose. Function calling is the most widely supported structured-output route across OpenRouter providers.
4. **Shape-guard the result.** Coerce types and never crash on junk.
5. **Be idempotent.** Webhooks retry. The `seen` set is the simplest version.

Run it:

```bash
uv run --env-file .env uvicorn ainative.webhook:build --factory --port 8787
```

Send a signed event (in a second terminal; `dev-secret` must match `WEBHOOK_SECRET` in your `.env`). Port 8787 avoids clashing with the many dev servers that default to 3000:

```bash
BODY='{"source":"asks","data":{"id":"A1","title":"Search broken","description":"It errors."}}'
SIG=$(printf '%s' "$BODY" | openssl dgst -sha256 -hmac dev-secret | awk '{print $NF}')
curl -s -X POST localhost:8787/webhook -H "x-signature: $SIG" -d "$BODY"
```

Expected: `{"ok":true,"verdict":{"complete":false,"missing":[...]}}`. Resend it and you get `"duplicate":true`.

## Step 7: Tests that run offline

No keys, no network, no running API. A fake model stands in for OpenRouter. A fake API (an `httpx.MockTransport`) stands in for chapter 1, and records every request so tests can check the token and the paging. The MCP test uses a real in-memory session against your real server. Create `ainative/tests/test_all.py`:

```py
import hashlib
import hmac
import json
from types import SimpleNamespace as NS

import httpx
import pytest
from mcp import Client
from starlette.testclient import TestClient

from ainative import agent, server
from ainative.data import Directory
from ainative.server import get_provider_details, mcp, search_providers
from ainative.webhook import make_app

ANA = {"id": 1, "name": "Dr. Ana Reyes", "specialty": "orthopedic surgery",
       "city": "San Diego", "state": "CA", "visibility": "public"}
CHLOE = {"id": 3, "name": "Dr. Chloe Park", "specialty": "orthopedic surgery",
         "city": "Los Angeles", "state": "CA", "visibility": "public"}


class FakeApi:
    """Stands in for the chapter 1 API. Records every request so tests can inspect it."""

    def __init__(self, handler):
        self.requests: list[httpx.Request] = []
        self._handler = handler

    def __call__(self, request: httpx.Request) -> httpx.Response:
        self.requests.append(request)
        return self._handler(request)


def install(handler) -> FakeApi:
    api = FakeApi(handler)
    client = httpx.AsyncClient(
        base_url="http://api.test/api",
        headers={"Authorization": "Bearer dir_test"},
        transport=httpx.MockTransport(api),
    )
    server.set_directory(Directory(client))
    return api


def page(items, count=None):
    return httpx.Response(200, json={"items": items, "count": count or len(items), "page": 1, "page_size": 20})


@pytest.fixture(autouse=True)
def _reset_directory():
    yield
    server.set_directory(None)


async def test_short_query_rejected_without_calling_the_api():
    api = install(lambda r: page([]))
    assert (await search_providers("ab")).startswith("Error")
    assert api.requests == []


async def test_search_formats_the_api_page():
    install(lambda r: page([ANA, CHLOE]))
    out = await search_providers("orthopedic CA")
    assert out == (
        "[1] Dr. Ana Reyes | orthopedic surgery | San Diego, CA\n"
        "[3] Dr. Chloe Park | orthopedic surgery | Los Angeles, CA"
    )


async def test_search_sends_the_token_and_clamped_paging():
    api = install(lambda r: page([ANA]))
    await search_providers("orthopedic", page=0, page_size=500)
    request = api.requests[0]
    assert request.headers["authorization"] == "Bearer dir_test"
    assert request.url.path == "/api/providers"
    assert dict(request.url.params) == {"q": "orthopedic", "page": "1", "page_size": "100"}


async def test_search_tells_the_model_when_there_is_more():
    install(lambda r: page([ANA], count=5))
    out = await search_providers("orthopedic", page_size=1)
    assert out.endswith("(1 of 5 shown. Ask for page 2 for more.)")


async def test_no_results_message():
    install(lambda r: page([]))
    assert await search_providers("zzz nothing") == "No providers found for 'zzz nothing'."


@pytest.mark.parametrize(
    ("response", "expected"),
    [
        (httpx.Response(401, json={"detail": "Unauthorized"}), "rejected our token"),
        (httpx.Response(429, json={"detail": "Throttled"}), "rate limiting"),
        (httpx.Response(503), "failed (503)"),
    ],
)
async def test_api_failures_become_error_strings(response, expected):
    install(lambda r: response)
    out = await search_providers("orthopedic")
    assert out.startswith("Error:") and expected in out


async def test_unreachable_api_becomes_an_error_string():
    def down(request):
        raise httpx.ConnectError("refused")

    install(down)
    assert "not reachable" in await search_providers("orthopedic")


async def test_provider_details_found_and_missing():
    install(lambda r: httpx.Response(200, json=ANA) if r.url.path.endswith("/1") else httpx.Response(404))
    assert (await get_provider_details(1)).startswith("[1] Dr. Ana Reyes")
    assert await get_provider_details(99) == "Error: no provider with id 99."


class FakeModel:
    """Scripted model: first asks for a tool, then answers."""
    def __init__(self):
        self.calls = 0
        self.chat = self
        self.completions = self

    async def create(self, **kw):
        self.calls += 1
        if self.calls == 1:
            call = NS(id="t1", function=NS(
                name="search_providers", arguments='{"query": "orthopedic CA"}'))
            msg = NS(content=None, tool_calls=[call])
        else:
            msg = NS(content="Found 2 surgeons.", tool_calls=None)
        return NS(choices=[NS(message=msg)])


async def test_agent_loop_over_real_mcp_session():
    api = install(lambda r: page([ANA, CHLOE]))
    async with Client(mcp) as session:  # in-memory: no network, real MCP server
        tools = agent.to_openai_tools(await session.list_tools())
        assert {t["function"]["name"] for t in tools} == {"search_providers", "get_provider_details"}
        model = FakeModel()
        assert await agent.run_agent("find", session, model, tools) == "Found 2 surgeons."
        assert model.calls == 2
    assert len(api.requests) == 1  # the tool really went through the data layer


class FakeVerdictModel:
    def __init__(self):
        self.chat = self
        self.completions = self

    async def create(self, **kw):
        args = '{"complete": false, "missing": ["steps to reproduce"]}'
        call = NS(function=NS(arguments=args))
        return NS(choices=[NS(message=NS(tool_calls=[call]))])


def _sign(body: bytes, secret: str) -> str:
    return hmac.new(secret.encode(), body, hashlib.sha256).hexdigest()


def test_webhook_flow():
    acted = []

    async def act(i, v):
        acted.append((i, v))

    client = TestClient(make_app(FakeVerdictModel(), "s3cret", act))
    body = json.dumps({"source": "asks", "data": {"id": "A1", "title": "Bug", "description": "broken"}}).encode()
    assert client.post("/webhook", content=body, headers={"x-signature": "nope"}).status_code == 401
    ok = client.post("/webhook", content=body, headers={"x-signature": _sign(body, "s3cret")})
    assert ok.json()["verdict"] == {"complete": False, "missing": ["steps to reproduce"]}
    dup = client.post("/webhook", content=body, headers={"x-signature": _sign(body, "s3cret")})
    assert dup.json()["duplicate"] is True
    other = json.dumps({"source": "manual", "data": {"id": "B"}}).encode()
    assert client.post("/webhook", content=other, headers={"x-signature": _sign(other, "s3cret")}).json()["skipped"]
    assert len(acted) == 1


def test_openrouter_client_config(monkeypatch):
    monkeypatch.setenv("OPENROUTER_API_KEY", "sk-or-test")
    from ainative import llm

    client = llm.make_client()
    assert str(client.base_url).startswith("https://openrouter.ai/api/v1")


def test_directory_needs_a_token(monkeypatch):
    monkeypatch.delenv("DIRECTORY_API_TOKEN", raising=False)
    from ainative.data import DirectoryError

    with pytest.raises(DirectoryError, match="DIRECTORY_API_TOKEN"):
        Directory.from_env()
```

Run:

```bash
uv run pytest -q
uv run ruff check .
```

Expected: `14 passed` and `All checks passed!`.

What these tests cover: input validation, the exact request sent to the API (path, query and bearer token), result formatting and the "more pages" hint, every API failure becoming an error string, a missing token, the full agent loop against the real tool list, signature rejection, idempotency, and the skip filter. What they do not cover is whether the API really behaves the way the fake says it does, and whether the model picks good tools or writes good answers. The first is what Step 8 is for. The second needs a small set of saved scenarios you run live and review by hand, which is the subject of the evals chapter.

## Step 8: Run it end to end

```txt
Terminal 0:  (in directory/)  uv run --env-file .env python manage.py runserver 8001
Terminal 1:  uv run --env-file .env ainative-server
Terminal 2:  uv run --env-file .env python -m ainative.agent "Who are the orthopedic surgeons in California?"
Terminal 3:  uv run --env-file .env uvicorn ainative.webhook:build --factory --port 8787
```

The agent should call `search_providers`, read the two results, and answer with Dr. Ana Reyes and Dr. Chloe Park. It will not mention Dr. Eli Stone, the internal provider, because the API never returned him.

## Step 9: Make it real

**Use different models for different jobs.** Because everything goes through `llm.py`, add one setting per role (for example a strong model for the agent and a cheaper one for the ticket triage) and have each caller read its own. Keep one client factory so transport config stays in one place.

**Harden OpenRouter calls.** Set `timeout` and `max_retries` on the client for latency-sensitive paths. If you rely on structured output or tool calling, pass OpenRouter's provider preference `extra_body={"provider": {"require_parameters": True}}` so requests only route to providers that honor the parameters you send. Test that combination, because it can leave zero eligible providers for some parameters.

**Forward the real caller's credentials.** This chapter's server holds one token for everyone who talks to it, so every user sees what that token's owner sees. In production, the server should pass each end user's own token to the API, so the model can only see what that user is allowed to see. The pieces are already there: the API issues one token per user, and the data layer takes its client as an argument, so you build a client per request instead of one per process. This chapter did not build or test that.

**Add authentication to the MCP server itself.** Do not expose it on a network without bearer tokens or OAuth. Authorization belongs in your platform, not in the prompt.

**Add instructions for multi-step workflows.** Tools say what is possible. For repeatable procedures (for example "create a record, but check for duplicates first"), write the steps down as a reusable instruction file for the model, and state clearly when it should and should not be used.

**Grow the agent.** When one loop is not enough, move to an agent framework such as LangGraph, which also works with OpenRouter through its OpenAI-compatible client. Useful patterns: a supervisor that dispatches to specialist sub-agents, a verification step that checks claims against the tools that can settle them, and an async start-then-poll API for answers that take minutes.

**Deploy.** Package the API, the server and the webhook as containers or run them on a managed host. Set secrets as environment variables, expose a `/health` route (the API already has `/api/health`), and keep the offline test suite as your CI gate.

## Production checklist

- [ ] Every tool returns an error string instead of raising
- [ ] Inputs validated and clamped inside the tool, and again in the API
- [ ] The API enforces who may see what; the MCP server holds a least-privilege (non-staff) token
- [ ] The API's rate limits are understood, and a 429 reaches the model as a readable error
- [ ] Iteration cap and cost cap on every agent loop
- [ ] Webhook signatures verified, handlers idempotent
- [ ] Model output shape-guarded (tool arguments, JSON)
- [ ] One OpenRouter client factory, model set by environment variable
- [ ] Spend limit set on the OpenRouter key
- [ ] Suite runs offline in CI
- [ ] Dependencies bounded to the majors you tested (`mcp>=2.3,<3`, `openai>=3.0,<4`) and `uv.lock` committed
- [ ] `.env` in `.gitignore`; production secrets come from the host or a secrets manager, never from code

> [!TIP]
> **Verified:** lint clean, 14 tests passing. With the chapter 1 API running on port 8001 and the MCP server on port 8000, a real MCP `Client` over HTTP got correct search results, a "1 of 2 shown" paging hint, `no provider with id 5` for the internal provider, `rejected our token` for a wrong token, and `not reachable` with the API stopped. `uv run --env-file .env` loads the settings, and the webhook rejects bad signatures and skips non-matching events.
>
> **Not verified:** a live call through OpenRouter (no key in the build environment), the exact OpenRouter slug for Fable, per-user token forwarding, and Windows commands. Run steps 5 and 6 with your key. If the default `anthropic/claude-fable-5-1` is rejected, set `OPENROUTER_MODEL` to the slug shown on the model's OpenRouter page.
