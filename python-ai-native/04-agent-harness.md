# Python AI Native: Build an Agent Harness from Scratch (Step by Step)

In an AI product, the model is the engine and the harness is everything else: the loop that calls it, the tools it can use, the rules about what it may do, the memory it works with, and the log of what happened. Claude Code, Cursor, and deepagents are all harnesses. Swapping the model changes how smart the agent is. Changing the harness changes whether the agent is safe, affordable and debuggable.

In this tutorial you build one from a blank folder, about 500 lines of Python, and run it on a real model through OpenRouter. Each step has a check, and the whole project ends with 45 passing tests that need no API key. At the end, a table maps every part you wrote to the same part in Claude Code and deepagents, so you can read those tools' docs as a practitioner.

> [!NOTE]
> Chapter 4 of [Python AI Native: Build Production AI](../README.md). Read the MCP server tutorial and the Deep Agent tutorial first if you can. Neither is required, but they show the same ideas from the outside.

---

## What a harness is

The whole idea fits in a few lines. Do not type this, it is the mental model:

```py
messages = [system_prompt, user_task]
while True:
    reply = model(messages, tools)          # 1. ask the model
    messages.append(reply)
    if not reply.tool_calls:                # 2. no tool request means it is finished
        return reply.text
    for call in reply.tool_calls:           # 3. run what it asked for, feed results back
        messages.append(run(call))
```

Everything in this tutorial is what you add so that loop is **safe** (permissions, a jailed workspace), **affordable** (a step cap, output trimming, compaction), **extensible** (tools, hooks, sub-agents) and **debuggable** (a trace of every decision).

| Part | What it does | Step |
| --- | --- | --- |
| Model boundary | One function from messages to a reply | 1 |
| Tools | Python functions plus a JSON schema the model reads | 2 |
| Workspace tools | Files and shell, jailed to one folder | 3 |
| Permissions | Allow, ask or deny before anything runs | 4 |
| Trace | One JSON line per decision | 5 |
| Context management | Trim big outputs, summarize old turns | 6 |
| The loop | Glue for all of the above, plus hooks | 7 |
| Sub-agents | A tool that runs another harness | 8 |
| CLI | Run it for real | 9 |

---

## Step 0: Set up the project

You need uv (it installs Python and manages the project), Python 3.11 or newer, and an OpenRouter API key for the final live run. The tests never call a model, so you can do Steps 1 to 9 without a key.

**0.1 Install uv and check it.** Run in any terminal:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
uv --version
```

Check: a version number prints. On Windows use `powershell -c "irm https://astral.sh/uv/install.ps1 | iex"`.

**0.2 Create the project folder.** Run in the folder where you keep projects. Every command from here on runs inside `harness-lab/` unless a step says otherwise.

```bash
mkdir harness-lab
cd harness-lab
mkdir -p src/harness tests demo
touch src/harness/__init__.py
```

When you finish this tutorial the folder looks like this:

```txt
harness-lab/
├── pyproject.toml
├── .gitignore
├── .env                      (your key, never committed)
├── src/harness/
│   ├── __init__.py
│   ├── model.py              Step 1
│   ├── testing.py            Step 1
│   ├── tools.py              Step 2
│   ├── files.py              Step 3
│   ├── permissions.py        Step 4
│   ├── trace.py              Step 5
│   ├── context.py            Step 6
│   ├── loop.py               Step 7
│   ├── subagent.py           Step 8
│   └── cli.py                Step 9
├── tests/                    one test file per step
└── demo/                     a tiny buggy project for the agent to fix
```

**0.3 Create ****`pyproject.toml`** in `harness-lab/`:

```toml
[project]
name = "harness"
version = "0.1.0"
description = "Build an agent harness from scratch"
requires-python = ">=3.11"
dependencies = ["openai>=3.0,<4"]

[project.scripts]
harness = "harness.cli:run"

[dependency-groups]
dev = ["pytest>=9.0", "ruff>=0.5.6"]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/harness"]

[tool.ruff.lint]
select = ["E", "F", "I", "UP"]
ignore = ["E501"]

[tool.pytest.ini_options]
testpaths = ["tests"]
```

**0.4 Create ****`.gitignore`** in `harness-lab/` so your key and the venv never get committed:

```txt
.venv/
.env
__pycache__/
*.pyc
trace.jsonl
```

**0.5 Create ****`.env`** in `harness-lab/`. Get a key at [openrouter.ai/keys](http://openrouter.ai/keys) and paste it in. You load this file with `uv run --env-file .env`, so you never export variables in each terminal:

```txt
OPENROUTER_API_KEY=sk-or-v1-your-key-here
OPENROUTER_MODEL=anthropic/claude-fable-5-1
```

**0.6 Install and check.**

```bash
uv sync
uv run python -c "import openai; print(openai.__version__)"
```

Check: it prints `3.` followed by numbers. uv created `.venv/` and `uv.lock` for you.

---

## Step 1: The model boundary

The harness must not care which provider is behind the model. So the only code that touches an API is one small class with one method: messages and tool specs in, one `Reply` out. A reply holds text, tool calls, or both.

**1.1 Create ****`src/harness/model.py`****:**

```py
"""The model boundary: messages in, one Reply out. Nothing else knows about the API."""
import os
from dataclasses import dataclass
from typing import Protocol

from openai import OpenAI

BASE_URL = "https://openrouter.ai/api/v1"


@dataclass(frozen=True)
class ToolCall:
    id: str
    name: str
    arguments: str  # raw JSON text, exactly as the model wrote it


@dataclass(frozen=True)
class Reply:
    text: str = ""
    tool_calls: tuple[ToolCall, ...] = ()


class Model(Protocol):
    def complete(self, messages: list[dict], tools: list[dict]) -> Reply: ...


class OpenRouterModel:
    def __init__(self, model: str | None = None, max_tokens: int = 2048):
        self.model = model or os.environ.get("OPENROUTER_MODEL", "anthropic/claude-fable-5-1")
        self.max_tokens = max_tokens
        self._client = OpenAI(
            base_url=BASE_URL,
            api_key=os.environ["OPENROUTER_API_KEY"],
            default_headers={"X-Title": "harness-tutorial"},
        )

    def complete(self, messages: list[dict], tools: list[dict]) -> Reply:
        extra = {"tools": tools} if tools else {}
        resp = self._client.chat.completions.create(
            model=self.model, max_tokens=self.max_tokens, messages=messages, **extra
        )
        msg = resp.choices[0].message
        calls = tuple(
            ToolCall(c.id, c.function.name, c.function.arguments or "{}")
            for c in msg.tool_calls or []
        )
        return Reply(msg.content or "", calls)
```

Notice three things. `ToolCall.arguments` stays a raw string: models sometimes emit broken JSON, and the harness (Step 7) must be the one to handle that, not the API wrapper. `Model` is a `Protocol`, so anything with a `complete` method counts as a model. And OpenRouter is just the OpenAI client pointed at a different `base_url`.

**1.2 Create ****`src/harness/testing.py`****.** This is a fake model that plays back a script. It is the most important testing tool in this tutorial: it lets you test every harness behavior deterministically, with no network and no cost.

```py
"""A fake model that plays back scripted replies, so harness tests need no network."""
import itertools
import json

from harness.model import Reply, ToolCall

_ids = itertools.count(1)


def call(name: str, /, **arguments) -> ToolCall:
    return ToolCall(f"call_{next(_ids)}", name, json.dumps(arguments))


def say(text: str) -> Reply:
    return Reply(text=text)


def use(*calls: ToolCall, text: str = "") -> Reply:
    return Reply(text=text, tool_calls=tuple(calls))


class ScriptedModel:
    def __init__(self, *replies: Reply):
        self._replies = list(replies)
        self.seen: list[list[dict]] = []  # a copy of the messages at every call

    def complete(self, messages: list[dict], tools: list[dict]) -> Reply:
        self.seen.append([dict(m) for m in messages])
        if not self._replies:
            raise AssertionError("ScriptedModel ran out of replies")
        return self._replies.pop(0)
```

**1.3 Create ****`tests/test_model.py`****.** It uses the real `openai` client with a fake HTTP transport, so it proves your parsing works with the actual SDK response shape. Note that openai 3.x ships its HTTP library as `httpx2`.

```py
import json

import httpx2 as httpx
from openai import OpenAI

from harness.model import BASE_URL, OpenRouterModel, ToolCall


def fake_openrouter(captured: list):
    def handler(request: httpx.Request) -> httpx.Response:
        captured.append(json.loads(request.content))
        return httpx.Response(200, json={
            "id": "gen-1", "object": "chat.completion", "created": 0, "model": "m",
            "choices": [{
                "index": 0, "finish_reason": "tool_calls",
                "message": {
                    "role": "assistant", "content": None,
                    "tool_calls": [{
                        "id": "call_9", "type": "function",
                        "function": {"name": "read_file", "arguments": "{\"path\": \"a.txt\"}"},
                    }],
                },
            }],
        })
    return handler


def test_the_real_sdk_response_maps_to_a_reply(monkeypatch):
    monkeypatch.setenv("OPENROUTER_API_KEY", "test-key")
    monkeypatch.setenv("OPENROUTER_MODEL", "some/model")
    model = OpenRouterModel()
    captured: list = []
    model._client = OpenAI(
        base_url=BASE_URL, api_key="test-key",
        http_client=httpx.Client(transport=httpx.MockTransport(fake_openrouter(captured))),
    )
    reply = model.complete([{"role": "user", "content": "hi"}], [{"type": "function", "function": {"name": "read_file"}}])
    assert reply.tool_calls == (ToolCall("call_9", "read_file", '{"path": "a.txt"}'),)
    assert reply.text == ""
    assert captured[0]["model"] == "some/model"
    assert captured[0]["tools"][0]["function"]["name"] == "read_file"


def test_no_tools_means_no_tools_key(monkeypatch):
    monkeypatch.setenv("OPENROUTER_API_KEY", "test-key")
    model = OpenRouterModel()
    captured: list = []
    model._client = OpenAI(
        base_url=BASE_URL, api_key="test-key",
        http_client=httpx.Client(transport=httpx.MockTransport(fake_openrouter(captured))),
    )
    model.complete([{"role": "user", "content": "hi"}], [])
    assert "tools" not in captured[0]
```

**1.4 Run it:**

```bash
uv run pytest tests/test_model.py -q
```

Check: `2 passed`.

---

## Step 2: Tools

A tool is a Python function plus a JSON schema. The model never sees your code, only the name, the description and the parameter schema, so those are your interface. The `@tool` decorator builds the schema from the function's type hints and uses the docstring as the description. Write docstrings for the model, not for humans.

**2.1 Create ****`src/harness/tools.py`****:**

```py
"""A tool is a Python function plus the JSON schema the model needs to call it."""
import inspect
from collections.abc import Callable
from dataclasses import dataclass
from typing import get_args, get_origin, get_type_hints

_JSON_TYPES = {str: "string", int: "integer", float: "number", bool: "boolean"}


def _schema_for(annotation) -> dict:
    if get_origin(annotation) is list:
        (item,) = get_args(annotation)
        return {"type": "array", "items": _schema_for(item)}
    return {"type": _JSON_TYPES[annotation]}


@dataclass(frozen=True)
class Tool:
    name: str
    description: str
    parameters: dict
    fn: Callable[..., str]
    risk: str  # "read" | "write" | "exec": what the permission layer keys on

    def spec(self) -> dict:
        return {
            "type": "function",
            "function": {
                "name": self.name,
                "description": self.description,
                "parameters": self.parameters,
            },
        }

    def run(self, args) -> str:
        """Never raises: a failed tool becomes text the model can read and react to."""
        if not isinstance(args, dict):
            return "error: arguments must be a JSON object"
        missing = [n for n in self.parameters["required"] if n not in args]
        extra = [n for n in args if n not in self.parameters["properties"]]
        if missing or extra:
            return f"error: missing arguments {missing}, unexpected arguments {extra}"
        try:
            return str(self.fn(**args))
        except Exception as exc:
            return f"error: {type(exc).__name__}: {exc}"


def tool(risk: str = "read"):
    """Turn a typed, documented function into a Tool. The docstring is the model's manual."""

    def wrap(fn: Callable[..., str]) -> Tool:
        hints = get_type_hints(fn)
        props, required = {}, []
        for name, param in inspect.signature(fn).parameters.items():
            props[name] = _schema_for(hints[name])
            if param.default is param.empty:
                required.append(name)
        return Tool(
            name=fn.__name__,
            description=inspect.getdoc(fn) or "",
            parameters={"type": "object", "properties": props, "required": required},
            fn=fn,
            risk=risk,
        )

    return wrap


class Tools:
    def __init__(self, *tools: Tool):
        self._by_name: dict[str, Tool] = {}
        for t in tools:
            self.add(t)

    def add(self, t: Tool) -> None:
        if t.name in self._by_name:
            raise ValueError(f"duplicate tool name {t.name!r}")
        self._by_name[t.name] = t

    def get(self, name: str) -> Tool | None:
        return self._by_name.get(name)

    def __iter__(self):
        return iter(self._by_name.values())

    def names(self) -> list[str]:
        return list(self._by_name)

    def specs(self) -> list[dict]:
        return [t.spec() for t in self._by_name.values()]

    def read_only(self) -> "Tools":
        return Tools(*(t for t in self if t.risk == "read"))
```

Two design decisions matter here. `Tool.run` never raises: a failing tool returns `error: ...` text, because the model can read an error and try something else, but an exception would kill the run. And every tool carries a `risk` label (`read`, `write` or `exec`) that Step 4 uses to decide what needs approval.

**2.2 Create ****`tests/test_tools.py`****:**

```py
import pytest

from harness.tools import Tools, tool


@tool()
def add(a: int, b: int = 1) -> str:
    """Add two numbers."""
    return str(a + b)


def test_schema_comes_from_the_signature():
    spec = add.spec()["function"]
    assert spec["name"] == "add"
    assert spec["description"] == "Add two numbers."
    assert spec["parameters"] == {
        "type": "object",
        "properties": {"a": {"type": "integer"}, "b": {"type": "integer"}},
        "required": ["a"],
    }


def test_list_parameters_get_item_schemas():
    @tool()
    def many(names: list[str]) -> str:
        """Take names."""
        return ",".join(names)

    assert many.parameters["properties"]["names"] == {"type": "array", "items": {"type": "string"}}


def test_run_never_raises():
    assert add.run({"a": 2}) == "3"
    assert add.run({}).startswith("error: missing arguments ['a']")
    assert "unexpected arguments ['c']" in add.run({"a": 1, "c": 2})
    assert add.run({"a": "x"}).startswith("error: TypeError")
    assert add.run([1, 2]).startswith("error: arguments must be")


def test_duplicate_names_are_rejected():
    with pytest.raises(ValueError):
        Tools(add, add)


def test_read_only_filters_by_risk():
    @tool(risk="write")
    def save(text: str) -> str:
        """Save text."""
        return "saved"

    assert Tools(add, save).read_only().names() == ["add"]
```

**2.3 Run it:**

```bash
uv run pytest tests/test_tools.py -q
```

Check: `5 passed`.

---

## Step 3: Workspace tools

Now give the agent hands: list, read and write files, and run shell commands. They are built by a function that takes a `root` folder, so every path is checked against that root (the "jail").

**3.1 Create ****`src/harness/files.py`****:**

```py
"""File and shell tools, jailed to one workspace folder."""
import subprocess
from pathlib import Path

from harness.tools import Tool, tool


def file_tools(root: Path) -> list[Tool]:
    root = root.resolve()

    def safe(relative: str) -> Path:
        path = (root / relative).resolve()  # resolves "..", absolute paths and symlinks
        if not path.is_relative_to(root):
            raise PermissionError(f"{relative!r} is outside the workspace")
        return path

    @tool()
    def list_dir(path: str = ".") -> str:
        """List the files and folders in a workspace directory."""
        names = sorted(p.name + ("/" if p.is_dir() else "") for p in safe(path).iterdir())
        return "\n".join(names) or "(empty)"

    @tool()
    def read_file(path: str) -> str:
        """Read a text file from the workspace."""
        return safe(path).read_text()

    @tool(risk="write")
    def write_file(path: str, content: str) -> str:
        """Create or overwrite a text file in the workspace."""
        target = safe(path)
        target.parent.mkdir(parents=True, exist_ok=True)
        target.write_text(content)
        return f"wrote {len(content)} characters to {path}"

    @tool(risk="exec")
    def run_shell(command: str) -> str:
        """Run a shell command from the workspace root. Returns the exit code, then the output."""
        proc = subprocess.run(
            command, shell=True, cwd=root, capture_output=True, text=True, timeout=30
        )
        return f"exit {proc.returncode}\n{proc.stdout}{proc.stderr}"

    return [list_dir, read_file, write_file, run_shell]
```

`safe()` resolves `..`, absolute paths and symlinks before checking, which are the three ways an agent (or a prompt-injected file it read) could escape the folder.

> [!WARNING]
> The jail covers the file tools only. `run_shell` can run anything your user can run, including commands that touch files outside `root`. That is why Step 4 puts approval in front of it, and why a real deployment runs shell commands in a container or sandbox.

**3.2 Create ****`tests/test_files.py`****:**

```py
import pytest

from harness.files import file_tools
from harness.tools import Tools


def test_read_only_keeps_only_the_looking_tools(tmp_path):
    tools = Tools(*file_tools(tmp_path))
    assert tools.read_only().names() == ["list_dir", "read_file"]


def test_file_tools_round_trip(tmp_path):
    by_name = {t.name: t for t in file_tools(tmp_path)}
    assert by_name["write_file"].run({"path": "a/b.txt", "content": "hi"}) == "wrote 2 characters to a/b.txt"
    assert by_name["read_file"].run({"path": "a/b.txt"}) == "hi"
    assert by_name["list_dir"].run({}) == "a/"
    assert by_name["run_shell"].run({"command": "echo ok"}) == "exit 0\nok\n"


@pytest.mark.parametrize("path", ["../secret.txt", "/etc/passwd", "a/../../x"])
def test_workspace_jail(tmp_path, path):
    (tmp_path.parent / "secret.txt").write_text("nope")
    read = {t.name: t for t in file_tools(tmp_path)}["read_file"]
    assert "outside the workspace" in read.run({"path": path})


def test_symlink_escape_is_blocked(tmp_path):
    outside = tmp_path.parent / "outside.txt"
    outside.write_text("secret")
    (tmp_path / "link").symlink_to(outside)
    read = {t.name: t for t in file_tools(tmp_path)}["read_file"]
    assert "outside the workspace" in read.run({"path": "link"})
```

**3.3 Run it:**

```bash
uv run pytest tests/test_files.py -q
```

Check: `6 passed`.

---

## Step 4: Permissions

The model decides what to try. The harness decides what happens. A `Policy` maps each tool's risk level to `allow`, `ask` or `deny`, and a separate rule list denies dangerous shell phrases outright, even if a human would approve them. Unknown risk levels fall back to `ask`, never `allow`.

When the answer is `ask`, the harness calls an `approver`: a function that returns True or False. The default approver says no, so a harness with nobody watching is safe by default.

**4.1 Create ****`src/harness/permissions.py`****:**

```py
"""The gate between 'the model asked' and 'the machine did it'."""
from collections.abc import Callable
from dataclasses import dataclass, field

from harness.tools import Tool

ALLOW, ASK, DENY = "allow", "ask", "deny"

Approver = Callable[[Tool, dict], bool]


def never_approve(tool: Tool, args: dict) -> bool:
    return False  # the safe default: nobody is around to say yes


@dataclass
class Policy:
    by_risk: dict[str, str] = field(
        default_factory=lambda: {"read": ALLOW, "write": ASK, "exec": ASK}
    )
    blocked_phrases: tuple[str, ...] = ("rm -rf", "sudo ", "git push")

    def check(self, tool: Tool, args: dict) -> tuple[str, str]:
        """Returns (action, reason). Deny rules beat everything else."""
        if tool.risk == "exec":
            for value in args.values():
                for phrase in self.blocked_phrases:
                    if isinstance(value, str) and phrase in value:
                        return DENY, f"the command contains {phrase!r}, which is never allowed"
        action = self.by_risk.get(tool.risk, ASK)  # unknown risk levels ask, never allow
        return action, "needs approval" if action == ASK else ""
```

**4.2 Create ****`tests/test_permissions.py`****:**

```py
from pathlib import Path

from harness.files import file_tools
from harness.permissions import ALLOW, ASK, DENY, Policy, never_approve

TOOLS = {t.name: t for t in file_tools(Path("."))}


def test_default_policy_by_risk():
    policy = Policy()
    assert policy.check(TOOLS["read_file"], {"path": "a"}) == (ALLOW, "")
    assert policy.check(TOOLS["write_file"], {"path": "a", "content": ""}) == (ASK, "needs approval")
    assert policy.check(TOOLS["run_shell"], {"command": "ls"}) == (ASK, "needs approval")


def test_blocked_phrases_are_denied():
    action, reason = Policy().check(TOOLS["run_shell"], {"command": "sudo reboot"})
    assert action == DENY and "'sudo '" in reason


def test_a_stricter_policy_can_deny_a_whole_risk_level():
    policy = Policy(by_risk={"read": ALLOW, "write": DENY, "exec": DENY})
    assert policy.check(TOOLS["write_file"], {"path": "a", "content": ""})[0] == DENY


def test_unknown_risk_levels_ask():
    from harness.tools import Tool

    odd = Tool("odd", "", {"type": "object", "properties": {}, "required": []}, lambda: "", risk="network")
    assert Policy().check(odd, {})[0] == ASK


def test_never_approve_is_the_safe_default():
    assert never_approve(TOOLS["run_shell"], {"command": "ls"}) is False
```

**4.3 Run it:**

```bash
uv run pytest tests/test_permissions.py -q
```

Check: `5 passed`.

> [!NOTE]
> A phrase blocklist is a speed bump, not a security boundary: `rm  -rf` with two spaces, or a script that runs the same command, slips past it. Claude Code's docs say the same about their Bash deny rules. Treat deny phrases as guardrails for honest mistakes and rely on a real sandbox for hostile input.

---

## Step 5: The trace

When an agent misbehaves, the question is always "what did it see and what did it decide?" A trace answers that. Every decision becomes one JSON line, kept in memory and optionally appended to a file.

**5.1 Create ****`src/harness/trace.py`****:**

```py
"""Every decision the harness makes, as one JSON line. Debugging an agent means reading this."""
import json
import time
from pathlib import Path


class Trace:
    def __init__(self, path: Path | None = None):
        self.events: list[dict] = []
        self.path = path

    def emit(self, kind: str, **data) -> None:
        event = {"t": round(time.time(), 3), "kind": kind, **data}
        self.events.append(event)
        if self.path:
            with self.path.open("a") as f:
                f.write(json.dumps(event) + "\n")

    def kinds(self) -> list[str]:
        return [e["kind"] for e in self.events]
```

**5.2 Create ****`tests/test_trace.py`****:**

```py
import json

from harness.trace import Trace


def test_events_are_kept_in_memory_and_in_the_file(tmp_path):
    path = tmp_path / "trace.jsonl"
    trace = Trace(path)
    trace.emit("tool", tool="read_file", ok=True)
    trace.emit("stop", reason="done")
    assert trace.kinds() == ["tool", "stop"]
    lines = [json.loads(line) for line in path.read_text().splitlines()]
    assert [e["kind"] for e in lines] == ["tool", "stop"]
    assert lines[0]["tool"] == "read_file" and "t" in lines[0]


def test_no_path_means_memory_only():
    trace = Trace()
    trace.emit("model", step=1)
    assert trace.events[0]["step"] == 1
```

**5.3 Run it:**

```bash
uv run pytest tests/test_trace.py -q
```

Check: `2 passed`.

---

## Step 6: Context management

The model can only see what fits in its context window, and long tool outputs fill it fast. Two defenses:

- **Trim**: cap each tool output, keeping the head and the tail, because the start and the end of a log are usually the useful parts.
- **Compact**: when the conversation passes a budget, replace everything except the last few messages with a summary written by the model itself.

Compaction has one trap. An assistant message that requested a tool call must stay next to its tool results, or the API rejects the conversation. `compact` moves its cut point backwards until it does not land on a tool message.

**6.1 Create ****`src/harness/context.py`****:**

```py
"""Keep the conversation small enough to fit: cut big outputs, summarize old turns."""
import json

from harness.model import Model

MAX_TOOL_OUTPUT = 4000  # characters
KEEP_LAST = 6  # messages that survive compaction untouched

SUMMARY_PROMPT = (
    "Summarize this agent transcript for your future self. Keep the goal, decisions made, "
    "files touched, and anything still unresolved. Be brief."
)


def trim_output(text: str, limit: int = MAX_TOOL_OUTPUT) -> str:
    if len(text) <= limit:
        return text
    half = limit // 2
    return f"{text[:half]}\n... [{len(text) - limit} characters cut] ...\n{text[-half:]}"


def estimate_tokens(messages: list[dict]) -> int:
    return sum(len(json.dumps(m)) for m in messages) // 4  # ~4 characters per token


def compact(messages: list[dict], model: Model, keep_last: int = KEEP_LAST) -> list[dict]:
    """Replace everything but the last few messages with a model-written summary."""
    cut = len(messages) - keep_last
    # A tool result must stay next to the assistant message that asked for it.
    while cut > 1 and messages[cut]["role"] == "tool":
        cut -= 1
    if cut <= 1:
        return messages  # nothing old enough to summarize
    transcript = "\n".join(
        f"{m['role']}: {str(m.get('content') or m.get('tool_calls'))[:500]}"
        for m in messages[1:cut]
    )
    summary = model.complete(
        [{"role": "system", "content": SUMMARY_PROMPT}, {"role": "user", "content": transcript}],
        [],
    ).text
    note = {"role": "user", "content": f"Summary of earlier work:\n{summary}"}
    return [messages[0], note, *messages[cut:]]
```

**6.2 Create ****`tests/test_context.py`****:**

```py
from harness.context import compact, estimate_tokens, trim_output
from harness.testing import ScriptedModel, say


def test_trim_keeps_head_and_tail():
    out = trim_output("A" * 100 + "B" * 100, limit=50)
    assert out.startswith("A" * 25) and out.endswith("B" * 25)
    assert "150 characters cut" in out


def test_short_output_untouched():
    assert trim_output("short", limit=50) == "short"


def conversation(n_turns: int) -> list[dict]:
    messages = [{"role": "system", "content": "sys"}, {"role": "user", "content": "goal"}]
    for i in range(n_turns):
        messages.append({
            "role": "assistant", "content": None,
            "tool_calls": [{"id": f"c{i}", "type": "function", "function": {"name": "t", "arguments": "{}"}}],
        })
        messages.append({"role": "tool", "tool_call_id": f"c{i}", "content": f"result {i}"})
    return messages


def test_compact_summarizes_old_turns_and_keeps_recent_ones():
    messages = conversation(6)  # 14 messages
    model = ScriptedModel(say("the goal was X"))
    out = compact(messages, model, keep_last=4)
    assert out[0] == messages[0]
    assert out[1] == {"role": "user", "content": "Summary of earlier work:\nthe goal was X"}
    assert out[2:] == messages[-4:]


def test_compact_never_orphans_a_tool_result():
    messages = conversation(6)
    out = compact(messages, ScriptedModel(say("s")), keep_last=3)  # a naive cut lands on a tool message
    tail = out[2:]
    assert tail[0]["role"] == "assistant"
    ids_asked = {c["id"] for m in tail if m.get("tool_calls") for c in m["tool_calls"]}
    ids_answered = {m["tool_call_id"] for m in tail if m["role"] == "tool"}
    assert ids_answered <= ids_asked


def test_compact_does_nothing_when_history_is_short():
    messages = conversation(1)
    assert compact(messages, ScriptedModel(), keep_last=6) is messages


def test_estimate_grows_with_content():
    small = [{"role": "user", "content": "hi"}]
    big = [{"role": "user", "content": "hi" * 1000}]
    assert estimate_tokens(big) > 500 > estimate_tokens(small)
```

**6.3 Run it:**

```bash
uv run pytest tests/test_context.py -q
```

Check: `6 passed`.

---

## Step 7: The loop

Now the glue. `Harness.run` is the loop from the mental model, with each safeguard in its place:

1. Compact if the conversation is over budget.
2. Ask the model.
3. If it asked for no tools, return its text.
4. For each tool call: run **hooks**, then the **policy**, then the tool, then trim the output and trace it.
5. Stop after `max_steps` no matter what.

A **hook** is your own function called at a fixed point. A `before_tool` hook can block a call by returning a reason, which makes it the place for project-specific rules ("never touch the migrations folder"). An `after_tool` hook observes the result, which is where you print progress or log metrics.

**7.1 Create ****`src/harness/loop.py`****:**

```py
"""The harness itself: a loop that asks the model, runs what it asks for, and feeds results back."""
import json
from collections.abc import Callable
from dataclasses import dataclass, field

from harness.context import MAX_TOOL_OUTPUT, compact, estimate_tokens, trim_output
from harness.model import Model, Reply
from harness.permissions import ALLOW, ASK, DENY, Approver, Policy, never_approve
from harness.tools import Tools
from harness.trace import Trace

SYSTEM = (
    "You are a coding assistant working inside one project folder. Look before you answer: "
    "use the tools to read files and run commands instead of guessing. Make small, "
    "targeted changes, then say what you did."
)


@dataclass
class Hooks:
    """Your code, run at fixed points. A before_tool hook returns a reason to block the call."""

    before_tool: list[Callable[[str, dict], str | None]] = field(default_factory=list)
    after_tool: list[Callable[[str, dict, str], None]] = field(default_factory=list)


@dataclass
class RunResult:
    text: str
    steps: int
    stopped: str  # "done" | "max_steps"


def assistant_message(reply: Reply) -> dict:
    message = {"role": "assistant", "content": reply.text or None}
    if reply.tool_calls:
        message["tool_calls"] = [
            {"id": c.id, "type": "function", "function": {"name": c.name, "arguments": c.arguments}}
            for c in reply.tool_calls
        ]
    return message


class Harness:
    def __init__(
        self,
        model: Model,
        tools: Tools,
        *,
        system: str = SYSTEM,
        policy: Policy | None = None,
        approver: Approver = never_approve,
        hooks: Hooks | None = None,
        trace: Trace | None = None,
        max_steps: int = 15,
        token_budget: int = 24_000,
    ):
        self.model = model
        self.tools = tools
        self.policy = policy or Policy()
        self.approver = approver
        self.hooks = hooks or Hooks()
        self.trace = trace or Trace()
        self.max_steps = max_steps
        self.token_budget = token_budget
        self.messages: list[dict] = [{"role": "system", "content": system}]

    def run(self, user_text: str) -> RunResult:
        self.messages.append({"role": "user", "content": user_text})
        for step in range(1, self.max_steps + 1):  # a hard cap: the loop can never run away
            self._maybe_compact()
            reply = self.model.complete(self.messages, self.tools.specs())
            self.trace.emit(
                "model", step=step, text=reply.text, calls=[c.name for c in reply.tool_calls]
            )
            self.messages.append(assistant_message(reply))
            if not reply.tool_calls:
                return RunResult(reply.text, step, "done")
            for call in reply.tool_calls:
                result = self._execute(call.name, call.arguments)
                self.messages.append({"role": "tool", "tool_call_id": call.id, "content": result})
        self.trace.emit("stop", reason="max_steps")
        return RunResult("Stopped: step limit reached.", self.max_steps, "max_steps")

    def _execute(self, name: str, raw_arguments: str) -> str:
        tool = self.tools.get(name)
        if tool is None:
            return f"error: unknown tool {name!r}. Available tools: {', '.join(self.tools.names())}"
        try:
            args = json.loads(raw_arguments or "{}")
        except json.JSONDecodeError as exc:
            return f"error: the arguments were not valid JSON ({exc})"

        for hook in self.hooks.before_tool:
            if reason := hook(name, args):
                self.trace.emit("blocked", tool=name, args=args, reason=reason)
                return f"blocked by hook: {reason}"

        action, reason = self.policy.check(tool, args)
        if action == ASK:
            approved = self.approver(tool, args)
            action, reason = (ALLOW, "") if approved else (DENY, "the user declined")
        if action == DENY:
            self.trace.emit("denied", tool=name, args=args, reason=reason)
            return f"denied: {reason}. Try a different approach."

        output = trim_output(tool.run(args), MAX_TOOL_OUTPUT)
        self.trace.emit("tool", tool=name, args=args, output=output[:200])
        for hook in self.hooks.after_tool:
            hook(name, args, output)
        return output

    def _maybe_compact(self) -> None:
        if estimate_tokens(self.messages) <= self.token_budget:
            return
        smaller = compact(self.messages, self.model)
        if smaller is not self.messages:
            self.trace.emit("compact", before=len(self.messages), after=len(smaller))
            self.messages = smaller
```

Every failure inside `_execute` returns a string instead of raising: unknown tool, broken JSON, blocked by a hook, denied by policy. The model reads the message and adapts. That is the core of why agents recover from mistakes.

**7.2 Create ****`tests/test_loop.py`****.** This is the biggest test file because the loop is where behavior lives. Each test scripts a model's replies and asserts what the harness did.

```py
from harness.context import SUMMARY_PROMPT, estimate_tokens
from harness.loop import Harness, Hooks
from harness.permissions import Policy
from harness.testing import ScriptedModel, call, say, use
from harness.tools import Tools, tool

calls_made = []


@tool()
def echo(text: str) -> str:
    """Echo text."""
    calls_made.append(text)
    return text


@tool(risk="write")
def save(text: str) -> str:
    """Pretend to save text."""
    calls_made.append(f"saved {text}")
    return "saved"


def make(model, **kw):
    calls_made.clear()
    return Harness(model, Tools(echo, save), **kw)


def test_final_answer_without_tools():
    result = make(ScriptedModel(say("hello"))).run("hi")
    assert (result.text, result.steps, result.stopped) == ("hello", 1, "done")


def test_tool_result_goes_back_to_the_model():
    model = ScriptedModel(use(call("echo", text="ping")), say("it said ping"))
    harness = make(model)
    result = harness.run("try echo")
    assert result.text == "it said ping"
    roles = [m["role"] for m in harness.messages]
    assert roles == ["system", "user", "assistant", "tool", "assistant"]
    assert model.seen[1][-1] == {
        "role": "tool", "tool_call_id": harness.messages[2]["tool_calls"][0]["id"], "content": "ping",
    }


def test_step_limit_stops_a_runaway_model():
    forever = [use(call("echo", text="again")) for _ in range(10)]
    result = make(ScriptedModel(*forever), max_steps=3).run("loop")
    assert (result.stopped, result.steps) == ("max_steps", 3)
    assert len(calls_made) == 3


def test_bad_calls_become_error_messages_not_crashes():
    model = ScriptedModel(
        use(call("nope"), call("echo", wrong=1)),
        say("recovered"),
    )
    harness = make(model)
    assert harness.run("go").text == "recovered"
    unknown, bad_args = (m["content"] for m in harness.messages if m["role"] == "tool")
    assert unknown.startswith("error: unknown tool 'nope'. Available tools: echo, save")
    assert "unexpected arguments ['wrong']" in bad_args


def test_malformed_json_arguments():
    bad = use(call("echo", text="x"))
    object.__setattr__(bad.tool_calls[0], "arguments", "{not json")
    harness = make(ScriptedModel(bad, say("ok")))
    harness.run("go")
    assert harness.messages[3]["content"].startswith("error: the arguments were not valid JSON")


def test_writes_are_denied_without_an_approver():
    harness = make(ScriptedModel(use(call("save", text="x")), say("ok")))
    harness.run("save it")
    assert harness.messages[3]["content"] == "denied: the user declined. Try a different approach."
    assert calls_made == []


def test_approver_can_allow_a_write():
    asked = []
    harness = make(
        ScriptedModel(use(call("save", text="x")), say("ok")),
        approver=lambda tool, args: asked.append(tool.name) or True,
    )
    harness.run("save it")
    assert asked == ["save"]
    assert calls_made == ["saved x"]


def test_reads_never_ask():
    harness = make(ScriptedModel(use(call("echo", text="x")), say("ok")), approver=lambda t, a: 1 / 0)
    harness.run("go")
    assert calls_made == ["x"]


def test_before_hook_can_block():
    hooks = Hooks(before_tool=[lambda name, args: "no echoing today" if name == "echo" else None])
    harness = make(ScriptedModel(use(call("echo", text="x")), say("ok")), hooks=hooks)
    harness.run("go")
    assert harness.messages[3]["content"] == "blocked by hook: no echoing today"
    assert calls_made == []


def test_after_hook_sees_the_output():
    seen = []
    hooks = Hooks(after_tool=[lambda name, args, out: seen.append((name, out))])
    make(ScriptedModel(use(call("echo", text="x")), say("ok")), hooks=hooks).run("go")
    assert seen == [("echo", "x")]


def test_long_tool_output_is_trimmed():
    @tool()
    def big() -> str:
        """Return a lot of text."""
        return "x" * 10_000

    harness = Harness(ScriptedModel(use(call("big")), say("ok")), Tools(big))
    harness.run("go")
    assert "characters cut" in harness.messages[3]["content"]
    assert len(harness.messages[3]["content"]) < 4100


def test_trace_records_the_run():
    harness = make(ScriptedModel(use(call("echo", text="x")), say("done")))
    harness.run("go")
    assert harness.trace.kinds() == ["model", "tool", "model"]


def test_blocked_phrases_beat_approval(tmp_path):
    from harness.files import file_tools

    harness = Harness(
        ScriptedModel(use(call("run_shell", command="sudo reboot")), say("ok")),
        Tools(*file_tools(tmp_path)),
        approver=lambda t, a: True,
        policy=Policy(),
    )
    harness.run("go")
    assert harness.messages[3]["content"].startswith("denied: the command contains 'sudo '")


class SummarizingModel(ScriptedModel):
    """Answers summary requests itself; everything else comes from the script."""

    def complete(self, messages, tools):
        if messages[0]["content"] == SUMMARY_PROMPT:
            return say("summary text")
        return super().complete(messages, tools)


def test_harness_compacts_when_over_budget():
    @tool()
    def noisy() -> str:
        """Return noise."""
        return "n" * 400

    model = SummarizingModel(*[use(call("noisy")) for _ in range(8)], say("finished"))
    harness = Harness(model, Tools(noisy), token_budget=500, max_steps=20)
    result = harness.run("go")
    assert result.text == "finished"
    assert "compact" in harness.trace.kinds()
    assert harness.messages[1]["content"].startswith("Summary of earlier work")
    assert estimate_tokens(harness.messages) < 1000
```

**7.3 Run it:**

```bash
uv run pytest tests/test_loop.py -q
```

Check: `14 passed`.

---

## Step 8: Sub-agents

A sub-agent is just a tool whose implementation is another `Harness`. The parent calls `delegate(task)`. The child starts with an empty context, does the work with its own tool calls, and returns only its final text. The parent never sees the child's intermediate file reads, so its context stays small.

Two rules keep this safe. The child gets only **read-only tools**, and it does not get `delegate`, so it cannot spawn more agents or change anything. And the tool description tells the parent the child cannot see the conversation, so the task must be self-contained.

**8.1 Create ****`src/harness/subagent.py`****:**

```py
"""A sub-agent is a tool whose implementation is another harness with a fresh context."""
from harness.loop import Harness
from harness.model import Model
from harness.tools import Tool, Tools, tool
from harness.trace import Trace

SUB_SYSTEM = (
    "You are a sub-agent. Do the task you were given using your tools, then reply with a short "
    "final answer. Your reply is all the caller will see."
)


def delegate_tool(model: Model, child_tools: Tools, trace: Trace | None = None) -> Tool:
    @tool()
    def delegate(task: str) -> str:
        """Hand a self-contained task to a sub-agent that has a fresh, empty context. It returns
        only its final answer. Include everything it needs: it cannot see this conversation."""
        child = Harness(model, child_tools, system=SUB_SYSTEM, trace=trace, max_steps=8)
        return child.run(task).text

    return delegate
```

**8.2 Create ****`tests/test_subagent.py`****:**

```py
from harness.loop import Harness
from harness.subagent import delegate_tool
from harness.testing import ScriptedModel, call, say, use
from harness.tools import Tools, tool


@tool()
def peek() -> str:
    """Look at something."""
    return "the secret is 42"


def test_sub_agent_returns_only_its_final_answer():
    # one scripted model plays both roles; replies are consumed in the order they are needed
    model = ScriptedModel(
        use(call("delegate", task="find the secret")),  # parent asks to delegate
        use(call("peek")),                              # child uses its tool
        say("it is 42"),                                # child's final answer
        say("The secret is 42."),                       # parent's final answer
    )
    tools = Tools(peek, delegate_tool(model, Tools(peek)))
    parent = Harness(model, tools)
    assert parent.run("what is the secret?").text == "The secret is 42."

    parent_tool_messages = [m["content"] for m in parent.messages if m["role"] == "tool"]
    assert parent_tool_messages == ["it is 42"]  # the child's tool call never enters parent context
    child_first_call = model.seen[1]
    assert [m["role"] for m in child_first_call] == ["system", "user"]  # fresh context
    assert child_first_call[1]["content"] == "find the secret"
```

**8.3 Run it:**

```bash
uv run pytest tests/test_subagent.py -q
```

Check: `1 passed`.

---

## Step 9: The command line

The last file wires everything together: build the tools, hand a read-only set to `delegate`, choose an approver, attach a hook that prints each tool call, and parse arguments. `main` accepts an optional `model` so tests can inject a scripted one.

**9.1 Create ****`src/harness/cli.py`****:**

```py
"""Wire everything together and give it a command line."""
import argparse
import sys
from pathlib import Path

from harness.files import file_tools
from harness.loop import Harness, Hooks
from harness.model import Model, OpenRouterModel
from harness.permissions import Approver
from harness.subagent import delegate_tool
from harness.tools import Tool, Tools
from harness.trace import Trace


def ask_user(tool: Tool, args: dict) -> bool:
    print(f"\n  The agent wants to run {tool.name}({args})")
    return input("  Allow? [y/N] ").strip().lower() in {"y", "yes"}


def approve_all(tool: Tool, args: dict) -> bool:
    return True


def show_tool(name: str, args: dict, output: str) -> None:
    first_line = output.splitlines()[0] if output else ""
    print(f"  -> {name}({args}) {first_line[:70]}")


def build(model: Model, root: Path, *, approver: Approver, trace: Trace, max_steps: int) -> Harness:
    base = Tools(*file_tools(root))
    tools = Tools(*base, delegate_tool(model, base.read_only(), trace))
    return Harness(
        model,
        tools,
        approver=approver,
        hooks=Hooks(after_tool=[show_tool]),
        trace=trace,
        max_steps=max_steps,
    )


def main(argv: list[str] | None = None, model: Model | None = None) -> int:
    parser = argparse.ArgumentParser(prog="harness", description="A tiny agent harness.")
    parser.add_argument("prompt", nargs="*", help="one-shot task; omit for an interactive session")
    parser.add_argument("--root", type=Path, default=Path("."), help="workspace folder")
    parser.add_argument("--yes", action="store_true", help="approve every write and command")
    parser.add_argument("--max-steps", type=int, default=15)
    parser.add_argument("--trace", type=Path, help="append every event to this JSONL file")
    args = parser.parse_args(argv)

    if model is None:
        try:
            model = OpenRouterModel()
        except KeyError:
            print("OPENROUTER_API_KEY is not set. Put it in .env and run with --env-file.", file=sys.stderr)
            return 2

    harness = build(
        model,
        args.root,
        approver=approve_all if args.yes else ask_user,
        trace=Trace(args.trace),
        max_steps=args.max_steps,
    )
    if args.prompt:
        print(harness.run(" ".join(args.prompt)).text)
        return 0
    while True:
        try:
            line = input("\nyou> ").strip()
        except EOFError:
            return 0
        if line in {"exit", "quit"}:
            return 0
        if line:
            print(harness.run(line).text)


def run() -> None:  # the console-script entry point
    sys.exit(main())
```

**9.2 Create the demo project.** This is a tiny project with a bug for the agent to fix. Both files go in `demo/` inside `harness-lab/`.

`demo/calc.py`:

```py
def average(numbers):
    return sum(numbers) / (len(numbers) - 1)
```

`demo/test_calc.py`:

```py
from calc import average


def test_average():
    assert average([2, 4, 6]) == 4
```

Confirm the bug is real. Run this in `harness-lab/`:

```bash
(cd demo && uv run python -m pytest -q)
```

Check: `1 failed` with `assert 6.0 == 4`. The average of 2, 4, 6 should be 4, but the code divides by `len - 1`.

**9.3 Create ****`tests/test_cli.py`****.** The first test replays a complete bug-fix session with a scripted model: read the file, write the fix, run the tests, report.

```py
import json
from pathlib import Path

from harness.cli import build, main
from harness.testing import ScriptedModel, call, say, use
from harness.trace import Trace

DEMO = Path(__file__).parent.parent / "demo"


def test_one_shot_fixes_the_demo_bug(tmp_path, capsys):
    workspace = tmp_path / "demo"
    workspace.mkdir()
    for name in ("calc.py", "test_calc.py"):
        (workspace / name).write_text((DEMO / name).read_text())

    model = ScriptedModel(
        use(call("read_file", path="calc.py")),
        use(call("write_file", path="calc.py", content="def average(numbers):\n    return sum(numbers) / len(numbers)\n")),
        use(call("run_shell", command="python -m pytest -q test_calc.py")),
        say("Fixed: average divided by len - 1."),
    )
    trace_file = tmp_path / "trace.jsonl"
    code = main(
        ["--root", str(workspace), "--yes", "--trace", str(trace_file), "fix", "the", "bug"],
        model=model,
    )
    assert code == 0
    assert "len(numbers)\n" in (workspace / "calc.py").read_text()
    out = capsys.readouterr().out
    assert "-> write_file" in out and "Fixed: average" in out
    kinds = [json.loads(line)["kind"] for line in trace_file.read_text().splitlines()]
    assert kinds == ["model", "tool", "model", "tool", "model", "tool", "model"]


def test_without_yes_a_write_needs_approval(tmp_path, monkeypatch, capsys):
    monkeypatch.setattr("builtins.input", lambda prompt="": "n")
    model = ScriptedModel(use(call("write_file", path="x.txt", content="hi")), say("ok"))
    main(["--root", str(tmp_path), "write", "a", "file"], model=model)
    assert not (tmp_path / "x.txt").exists()
    assert "The agent wants to run write_file" in capsys.readouterr().out


def test_missing_key_is_a_clear_error(monkeypatch, capsys):
    monkeypatch.delenv("OPENROUTER_API_KEY", raising=False)
    assert main(["hello"]) == 2
    assert "OPENROUTER_API_KEY is not set" in capsys.readouterr().err


def test_the_sub_agent_gets_read_only_tools(tmp_path):
    model = ScriptedModel(
        use(call("delegate", task="write a file")),
        use(call("write_file", path="x.txt", content="hi")),  # the child tries to write
        say("could not"),
        say("done"),
    )
    harness = build(model, tmp_path, approver=lambda t, a: True, trace=Trace(), max_steps=5)
    assert harness.tools.names() == ["list_dir", "read_file", "write_file", "run_shell", "delegate"]
    harness.run("go")
    child_view = model.seen[2][-1]["content"]
    assert child_view == "error: unknown tool 'write_file'. Available tools: list_dir, read_file"
    assert not (tmp_path / "x.txt").exists()
```

**9.4 Run the whole suite and the linter:**

```bash
uv run pytest -q
uv run ruff check .
```

Check: `45 passed` and `All checks passed!`. The demo test is not collected, because `pyproject.toml` limits pytest to `tests/`.

---

## Step 10: Run it on a real model

This is the only step that needs your API key and spends tokens. Run in `harness-lab/`:

```bash
uv run --env-file .env harness --root demo --trace trace.jsonl "Run the tests, find the bug, and fix it."
```

The harness asks before it writes a file or runs a command. Answer `y`. You should see one line per tool call, then the model's summary. Your exact output will differ, since a different model run will take different steps, but it will look like this:

```txt
  -> run_shell({'command': 'python -m pytest -q'}) exit 1
  -> read_file({'path': 'calc.py'}) def average(numbers):
  -> write_file({'path': 'calc.py', 'content': '...'}) wrote 72 characters to calc.py
  -> run_shell({'command': 'python -m pytest -q'}) exit 0
The bug was in average(): it divided by len(numbers) - 1. I changed it to len(numbers) and the test passes.
```

Now read the trace, which is the record of every decision:

```bash
cat trace.jsonl
```

Try these experiments, each one teaching a different part of the harness:

- **Add ****`--yes`** and watch it run without asking. That is what "bypass permissions" modes do in other tools.
- **Decline a write** (answer `n`). The model gets `denied: the user declined` and tries something else or explains.
- **Run with ****`--max-steps 2`****.** It stops with `Stopped: step limit reached.` That is your protection against a runaway loop.
- **Ask it to read ****`../.env`****.** The jail refuses with `outside the workspace`.
- **Run with no prompt** for an interactive session: `uv run --env-file .env harness --root demo`, type tasks, and `exit` to quit.

To reset the demo after it fixes the bug, put the bug back: in `demo/calc.py` change `len(numbers)` to `(len(numbers) - 1)`.

---

## How this maps to the real tools

You wrote the same parts that every production harness has. Here is where to find each one.

| Your harness | Claude Code | deepagents |
| --- | --- | --- |
| `Harness.run` loop | The main agent loop | The agent graph built by `create_deep_agent` |
| `@tool` functions | Built-in tools (Read, Edit, Bash, and so on) plus MCP servers | Your tools plus the filesystem tools from `FilesystemMiddleware` |
| `Policy` and approver | Permission rules in `settings.json`, checked deny first, then ask, then allow, plus modes such as default, acceptEdits, plan, auto, dontAsk and bypassPermissions | `interrupt_on` with `HumanInTheLoopMiddleware` |
| `before_tool` and `after_tool` hooks | `PreToolUse` and `PostToolUse` hooks configured in settings. A hook that exits with code 2 blocks the call | Middleware that wraps model and tool calls |
| `compact` and `trim_output` | Automatic compaction and `/compact` | `SummarizationMiddleware` |
| `delegate` tool | Subagents defined as Markdown files in `.claude/agents/`, which get a fresh context and return a summary | `SubAgentMiddleware` and its `task` tool |
| System prompt | `CLAUDE.md` files and the built-in system prompt | `system_prompt`, plus `MemoryMiddleware` and `SkillsMiddleware` |
| `Trace` | Transcripts and hook events | LangSmith traces |

Two facts from the Claude Code docs line up with choices you made. A blocking hook runs before permission rules, so it wins even over an allow rule, which is the order in `_execute`. And deny rules are checked before ask rules before allow rules, which is why a blocked phrase in `Policy.check` beats approval.

---

## What is intentionally missing

This harness teaches the shape. A production one adds more:

- **A plan tool.** A `write_todos` style tool that lets the agent keep a checklist (deepagents ships one). It is a tool that stores a list.
- **Streaming and parallel tool calls.** You run calls one after another and wait for whole replies.
- **Retries and rate limits.** One failed API call ends your run.
- **A real sandbox.** Containers or VMs for `run_shell`, not a phrase blocklist.
- **Prompt-injection defenses.** Tool output is untrusted text. A file the agent reads can contain instructions. The permission layer, not the prompt, is your protection.
- **Evals.** A harness without tests of agent behavior over many tasks is a demo. That is the next tutorial in this series.

## What was verified

All 45 tests pass and ruff is clean, run from a fresh folder using the files on this page. The tests cover the loop, hooks, permissions, the workspace jail (including symlinks), compaction, sub-agents, the CLI, and the response parsing of the real openai 3.x SDK against a fake HTTP transport. The step 9 test replays a full bug-fix session with a scripted model.

Not verified in this page: a live call to OpenRouter (Step 10 needs your key), the `anthropic/claude-fable-5-1` model slug (confirm it on [openrouter.ai/models](http://openrouter.ai/models), or set `OPENROUTER_MODEL` to any tool-calling model), and Windows commands. The Claude Code and deepagents comparisons come from their current docs and installed packages, so check them again after major releases.
