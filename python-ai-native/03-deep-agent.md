# Python AI Native: Build a Deep Agent from Scratch (Step by Step)

> [!NOTE]
> Chapter 3 of [Python AI Native: Build Production AI](../README.md). Next: Build an Agent Harness from Scratch.

> [!NOTE]
> Build a **data-analysis deep agent** one layer at a time: a model with tools, then a workspace with files and code execution, then automatic context compression, on-demand skills, and finally a subagent with a task list. The model runs through **OpenRouter** (Fable by default). Every file below was built and run: lint clean and 10 tests passing, including a test where the agent runs real pandas code and one where it delegates to a subagent.

Based on LangChain's tutorial [Build a data analysis agent from scratch](https://docs.langchain.com/oss/python/langchain/deep-agent-from-scratch). This version uses a local workspace so you do not need a sandbox account, and routes the model through OpenRouter.

## The idea

A "deep agent" is an ordinary tool-calling agent with a few extra capabilities bolted on. In LangChain these are **middleware**: small plug-ins that each add tools and prompt text to the agent. You assemble the harness yourself, one piece at a time:

| Layer | Middleware | What it gives the agent |
| --- | --- | --- |
| 1 | none | A model and a tool loop |
| 2 | `FilesystemMiddleware`  • a backend | Read, write, edit, search files and run shell commands |
| 3 | `SummarizationMiddleware` | Automatic compression of old turns so long sessions do not overflow the context window |
| 4 | `SkillsMiddleware` | Domain knowledge loaded only when needed (progressive disclosure) |
| 5 | `TodoListMiddleware`  • `SubAgentMiddleware` | A task list and a specialist subagent with its own context |

The **backend** is where the agent's files live and where its commands run. Swapping the backend (local folder, remote sandbox, in-memory) changes where work happens without changing the agent.

## What you will build

| Step | You build |
| --- | --- |
| 0 | Install the tools and get an OpenRouter key |
| 1 | Project setup and one shared model factory |
| 2 | Layer 1: the minimal agent |
| 3 | A workspace and sample data |
| 4 | Layer 2: filesystem and code execution |
| 5 | Layer 3: summarization |
| 6 | Layer 4: skills |
| 7 | Layer 5: task list and visualizer subagent |
| 8 | Offline tests |
| 9 | Run it for real |
| 10 | Go to production with a remote sandbox |

## Step 0: Install everything

You need: **uv** (installs Python and dependencies), **Python 3.11+**, and an **OpenRouter API key**. Do each sub-step in order and run its check.

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

Expected: a version number such as `uv 0.10.12`. If you see `command not found`, reopen the terminal or add `~/.local/bin` to your `PATH`.

### 0.3 Install Python 3.11 or newer

Let uv do it:

```bash
uv python install 3.12
uv python list --only-installed
```

Expected: a `cpython-3.12.x` line. Any version 3.11 or newer works (the tutorial was tested on 3.14.8).

### 0.4 Get an OpenRouter API key

1. Create an account at [openrouter.ai](https://openrouter.ai) and add a few dollars of credit.
2. Open **Keys** and click **Create Key**.
3. Copy the key (it starts with `sk-or-`). You will not see it again.
4. Check the model list for the model you want. The default here is `anthropic/claude-fable-5-1`. If that slug differs on OpenRouter, copy the exact slug from the model's page and use it in step 1.
5. Set a spend limit on the key. Agents that run code in loops can use a lot of tokens.

Keep the key handy. In Step 1 you will save it in a `.env` file that stays out of git. Never paste it into code or commit it. Everything works without a key except the live runs in steps 2, 4 and 9.

### 0.5 Installation checklist

- [ ] `uv --version` prints a version
- [ ] `uv python list --only-installed` shows Python 3.11 or newer
- [ ] You have copied an OpenRouter key with a spend limit set

> [!CAUTION]
> **Safety.** In this tutorial the agent writes and runs code on **your machine** inside a workspace folder. A model can write a command that deletes files or reads secrets. Use it only with throwaway data and a dedicated folder, never with production credentials, and never in a server or multi-user app. The agent's shell does not inherit your environment variables, but it can still read any file your user can read, including the project's `.env`. Use a dedicated OpenRouter key with a low spend limit. Step 10 shows the isolated alternative.

## Step 1: Create the project and the model factory

```bash
mkdir analyst && cd analyst
mkdir -p src/analyst tests versions skills/pandas-patterns
touch src/analyst/__init__.py
```

By the end, the project looks like this. **Every path in this tutorial is relative to the `analyst/` folder (the project root)**, and every command runs from there unless it says otherwise. Each file below is introduced with its full path starting at `analyst/`.

```txt
analyst/
├── .env
├── .gitignore
├── pyproject.toml
├── skills/
│   └── pandas-patterns/
│       └── SKILL.md
├── src/
│   └── analyst/
│       ├── __init__.py
│       ├── models.py
│       ├── agent.py
│       ├── workspace.py
│       └── run.py
├── tests/
│   ├── fakes.py
│   └── test_analyst.py
├── versions/          (filled by the cp commands in steps 2 to 7)
└── workspace/         (created automatically when you run the agent)
```

First, create two files directly in the project root: `analyst/.env` holds your settings and `analyst/.gitignore` keeps them out of git. The commands below create both (you are already inside `analyst/`):

```bash
cat > .env <<'EOF'
OPENROUTER_API_KEY=sk-or-paste-your-key-here
OPENROUTER_MODEL=anthropic/claude-fable-5-1
EOF
printf '.env\n.venv/\nworkspace/\n__pycache__/\n' > .gitignore
```

On Windows, create both files in your editor, saved in the `analyst` folder, with the same contents. Replace the placeholder with your real key. Commands that call the model use `uv run --env-file .env`, so nothing needs exporting and nothing is lost when you open a new terminal.

Create `analyst/pyproject.toml` (project root):

```toml
[project]
name = "analyst"
version = "0.1.0"
description = "Build a data-analysis deep agent from scratch"
requires-python = ">=3.11"
dependencies = [
    "deepagents>=0.7,<0.8",
    "langchain-openai>=1.0",
    "pandas>=2.2",
    "matplotlib>=3.9",
    "seaborn>=0.13",
]

[project.optional-dependencies]
dev = ["pytest>=8.0.0", "ruff>=0.5.6"]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/analyst"]

[tool.ruff.lint]
select = ["E", "F", "I", "UP"]
ignore = ["E501"]
```

`pandas`, `matplotlib` and `seaborn` are not for our code. The agent will use them when it writes analysis scripts in its workspace.

Install:

```bash
uv sync --extra dev
```

Expected: a package list ending with lines like `+ deepagents==0.7.x`. Check the imports:

```bash
uv run python -c "import deepagents, langchain, pandas; print('imports ok', deepagents.__version__)"
```

Expected: `imports ok 0.7.x`. Keep the `<0.8` pin: deepagents moves fast and the middleware names below match the 0.7 series.

Now the one place that knows how to reach a model. Create `analyst/src/analyst/models.py`:

```py
"""The one place that knows how to reach a model (OpenRouter, OpenAI-compatible)."""
import os

from langchain_openai import ChatOpenAI

BASE_URL = "https://openrouter.ai/api/v1"
# One knob for the model. Change it in the environment, not in code.
MODEL = os.environ.get("OPENROUTER_MODEL", "anthropic/claude-fable-5-1")


def make_model(**kwargs) -> ChatOpenAI:
    """Every model in the project is built here so key, URL and headers can't drift."""
    return ChatOpenAI(
        model=MODEL,
        base_url=BASE_URL,
        api_key=os.environ["OPENROUTER_API_KEY"],
        default_headers={"X-Title": "analyst-tutorial"},
        **kwargs,
    )
```

OpenRouter speaks the OpenAI API, so LangChain's `ChatOpenAI` works with a different `base_url`. To try another model, change `OPENROUTER_MODEL` in `.env` (copy exact slugs from [openrouter.ai/models](http://openrouter.ai/models)). No code changes.

## Step 2: Layer 1, the minimal agent

Create `analyst/src/analyst/agent.py`:

```py
from langchain.agents import create_agent


def build_agent(model):
    return create_agent(model=model, tools=[])
```

This is a model and an empty tool list. It can chat, but it cannot see files or run code. Save a copy for the tests later:

```bash
cp src/analyst/agent.py versions/agent_step1.py
```

Try it (needs your key):

```bash
uv run --env-file .env python -c "
from analyst.agent import build_agent
from analyst.models import make_model
r = build_agent(make_model()).invoke({'messages': [{'role': 'user', 'content': 'Say hello in five words.'}]})
print(r['messages'][-1].content)
"
```

Expected: a short greeting. If you get `KeyError: 'OPENROUTER_API_KEY'`, you forgot `--env-file .env` or the key is missing from `.env`. If you get a 404 or 400 about the model, fix the slug (step 0.4).

## Step 3: A workspace and sample data

The agent needs a place to work. Create `analyst/src/analyst/workspace.py`:

```py
"""The agent's working area: a folder it can read, write and run code in."""
import csv
import io
import os
from pathlib import Path

from deepagents.backends import LocalShellBackend

SALES_ROWS = [
    ["Date", "Product", "Units", "Revenue"],
    ["2025-08-01", "Widget A", 10, 250],
    ["2025-08-02", "Widget B", 5, 125],
    ["2025-08-03", "Widget A", 7, 175],
    ["2025-08-04", "Widget C", 3, 90],
]


def make_backend(workspace: Path) -> LocalShellBackend:
    """A local workspace with shell access. Throwaway folders only, see the warning in the tutorial."""
    workspace.mkdir(parents=True, exist_ok=True)
    return LocalShellBackend(
        root_dir=workspace,
        env={"PATH": os.environ["PATH"], "MPLBACKEND": "Agg"},  # headless charts
    )


def upload_sales(backend) -> None:
    buf = io.StringIO()
    csv.writer(buf).writerows(SALES_ROWS)
    backend.upload_files([("/sales.csv", buf.getvalue().encode())])


def upload_skills(backend, skills_dir: Path) -> None:
    """Copy every file under skills_dir to /skills/... inside the workspace."""
    files: list[tuple[str, bytes]] = []
    for path in sorted(skills_dir.rglob("*")):
        if path.is_file():
            rel = path.resolve().relative_to(skills_dir.resolve())
            files.append((f"/skills/{rel.as_posix()}", path.read_bytes()))
    backend.upload_files(files)
```

What to notice:

- `LocalShellBackend` is a folder on disk plus the ability to run shell commands in it. It passes only the environment variables you give it (`env=`), so your API key is **not** visible to the agent's commands. We pass `PATH` so `python` finds pandas, and `MPLBACKEND=Agg` so charts render without a display.
- Paths the agent sees are **absolute and virtual**: `/sales.csv` means `workspace/sales.csv`. Upload with absolute paths.
- `upload_skills` is used in step 6. It is here so the workspace code lives in one file.

## Step 4: Layer 2, filesystem and code execution

Replace the whole contents of `analyst/src/analyst/agent.py`:

```py
from deepagents.middleware import FilesystemMiddleware
from langchain.agents import create_agent


def build_agent(model, backend):
    return create_agent(
        model=model,
        tools=[],
        middleware=[FilesystemMiddleware(backend=backend)],
    )
```

`FilesystemMiddleware` gives the agent these tools: `ls`, `read_file`, `write_file`, `edit_file`, `delete`, `glob`, `grep` and `execute` (run a shell command). Now create `analyst/src/analyst/run.py` to drive it:

```py
"""Run the agent on a question: python -m analyst.run "your question"."""
import sys
from pathlib import Path

from analyst.agent import build_agent
from analyst.models import make_model
from analyst.workspace import make_backend, upload_sales, upload_skills

WORKSPACE = Path("workspace")
SKILLS_DIR = Path("skills")

DEFAULT_QUESTION = (
    "Analyze /sales.csv. Report total revenue by product, make a bar chart of it, "
    "and write a short report.md."
)


def main(question: str) -> None:
    backend = make_backend(WORKSPACE)
    upload_sales(backend)
    upload_skills(backend, SKILLS_DIR)
    agent = build_agent(make_model(), backend)
    result = agent.invoke(
        {"messages": [{"role": "user", "content": question}]},
        config={"recursion_limit": 40},
    )
    print(result["messages"][-1].content)
    print("\nFiles in workspace:")
    for p in sorted(WORKSPACE.iterdir()):
        print(" ", p.name)


if __name__ == "__main__":
    main(" ".join(sys.argv[1:]) or DEFAULT_QUESTION)
```

`recursion_limit` caps how many model and tool steps one request may take. Always set one on an agent that runs code.

Save a copy for the tests later:

```bash
cp src/analyst/agent.py versions/agent_step2.py
```

Try the first real task, reading a file only:

```bash
uv run --env-file .env python -m analyst.run "Read /sales.csv and summarize total revenue by product in one sentence. Do not run shell commands."
```

Expected: one sentence like "Widget A brought in $425, Widget B $125 and Widget C $90." The workspace folder now contains `sales.csv`. (Revenue: A = 250 + 175 = 425, B = 125, C = 90.)

> [!WARNING]
> `run.py` already uploads skills and the default question mentions a chart. Those only fully work after steps 6 and 7. That is fine: until then the agent just does the best it can with the layers you have built.

## Step 5: Layer 3, summarization

Long sessions fill the context window. `SummarizationMiddleware` compresses older turns automatically so the agent keeps working without you trimming messages. Replace the whole contents of `analyst/src/analyst/agent.py`:

```py
from deepagents.middleware import FilesystemMiddleware, SummarizationMiddleware
from langchain.agents import create_agent


def build_agent(model, backend):
    return create_agent(
        model=model,
        tools=[],
        middleware=[
            FilesystemMiddleware(backend=backend),
            SummarizationMiddleware(model=model, backend=backend),
        ],
    )
```

Two arguments matter: `model` is the model used to write the summary (you can pass a cheaper one), and `backend` is where the full history is saved so nothing is lost. Nothing looks different on short tasks. It only matters when a conversation grows large.

Save a copy for the tests later:

```bash
cp src/analyst/agent.py versions/agent_step3.py
```

## Step 6: Layer 4, skills

A **skill** is a folder with a `SKILL.md` file: a name, a one-line description, and detailed instructions. The agent always sees the name and description, and reads the full file only when a task needs it. This is **progressive disclosure**, and it keeps the system prompt small.

Create `analyst/skills/pandas-patterns/SKILL.md` (the folder already exists from Step 1):

```markdown
---
name: pandas-patterns
description: Common pandas and matplotlib patterns for data analysis and visualization
---

## Data loading
Use `pd.read_csv()` for CSV files. Always check `df.info()` and `df.describe()` first.

## Visualization
Use `matplotlib` for bar charts, `seaborn` for statistical plots.
Save figures with `plt.savefig("output.png", dpi=150, bbox_inches="tight")`.

## Reporting
Write a markdown summary to `report.md` alongside any generated charts.
```

`run.py` already uploads `skills/` to `/skills/` in the workspace through `upload_skills`. Now turn the middleware on. Replace the whole contents of `analyst/src/analyst/agent.py`:

```py
from deepagents.middleware import (
    FilesystemMiddleware,
    SkillsMiddleware,
    SummarizationMiddleware,
)
from langchain.agents import create_agent


def build_agent(model, backend):
    return create_agent(
        model=model,
        tools=[],
        middleware=[
            FilesystemMiddleware(backend=backend),
            SummarizationMiddleware(model=model, backend=backend),
            SkillsMiddleware(backend=backend, sources=["/skills/"]),
        ],
    )
```

Skills are plain files, so anyone can add one without touching Python. A good skill has a precise `description` (that is what the model uses to decide whether to open it) and a body of concrete steps.

Save a copy for the tests later:

```bash
cp src/analyst/agent.py versions/agent_step4.py
```

## Step 7: Layer 5, task list and a visualizer subagent

Two more pieces:

- `TodoListMiddleware` gives the agent a `write_todos` tool to plan and track multi-step work.
- `SubAgentMiddleware` gives it a `task` tool to hand work to a specialist. The subagent has its own instructions and its own context window, so chart-debugging noise never clutters the main agent's conversation.

Replace the whole contents of `analyst/src/analyst/agent.py` with the finished version:

```py
from deepagents import SubAgent
from deepagents.middleware import (
    FilesystemMiddleware,
    SkillsMiddleware,
    SubAgentMiddleware,
    SummarizationMiddleware,
)
from langchain.agents import create_agent
from langchain.agents.middleware import TodoListMiddleware


def build_agent(model, backend):
    visualizer: SubAgent = {
        "name": "visualizer",
        "description": "Generates charts and visualizations from data files in the workspace.",
        "system_prompt": (
            "You are a data visualization specialist. Write Python scripts using "
            "matplotlib and seaborn. Save all figures as PNG files."
        ),
        "tools": [],
        "model": model,
    }
    return create_agent(
        model=model,
        tools=[],
        middleware=[
            FilesystemMiddleware(backend=backend),
            SummarizationMiddleware(model=model, backend=backend),
            SkillsMiddleware(backend=backend, sources=["/skills/"]),
            TodoListMiddleware(),
            SubAgentMiddleware(backend=backend, subagents=[visualizer]),
        ],
    )
```

The subagent's `description` is how the main agent decides when to delegate, so write it like a job posting. The subagent shares the workspace (same `backend`) but not the conversation. You can give it a cheaper or different `model`.

This finished `build_agent` is the same stack that LangChain's prebuilt `create_deep_agent` assembles for you. You now know what each piece does.

Save the final copy for the tests:

```bash
cp src/analyst/agent.py versions/agent_step5.py
```

## Step 8: Tests that run offline

No key, no network. A scripted fake model stands in for OpenRouter, and everything else is real: the real middleware, a real workspace, and real code execution. Create `analyst/tests/fakes.py`:

```py
"""A scripted chat model so the whole suite runs offline, with no key and no network."""
from langchain_core.language_models.chat_models import BaseChatModel
from langchain_core.messages import AIMessage
from langchain_core.outputs import ChatGeneration, ChatResult
from pydantic import Field


class FakeToolModel(BaseChatModel):
    script: list[AIMessage]
    seen: list = Field(default_factory=list)        # every message list the model was shown
    tool_names: list = Field(default_factory=list)  # names of the tools bound to it
    step: int = 0

    def _generate(self, messages, stop=None, run_manager=None, **kwargs):
        self.seen.append(list(messages))
        msg = self.script[min(self.step, len(self.script) - 1)]
        self.step += 1
        return ChatResult(generations=[ChatGeneration(message=msg)])

    def bind_tools(self, tools, **kwargs):
        for t in tools:
            name = getattr(t, "name", None) or (t.get("function", {}).get("name") if isinstance(t, dict) else None)
            if name and name not in self.tool_names:
                self.tool_names.append(name)
        return self

    @property
    def _llm_type(self) -> str:
        return "fake-tool-model"


def tool_call(name: str, args: dict, call_id: str = "c1") -> AIMessage:
    return AIMessage(content="", tool_calls=[{"name": name, "args": args, "id": call_id}])
```

The tests check each layer separately, so they load the five copies of `agent.py` you saved along the way. Confirm they are all there:

```bash
ls versions/
```

Expected: `agent_step1.py` through `agent_step5.py`. If one is missing, go back to the step that says "Save a copy for the tests" and run its `cp` command.

Create `analyst/tests/test_analyst.py`:

```py
import importlib.util
from pathlib import Path

import pytest
from fakes import FakeToolModel, tool_call
from langchain_core.messages import AIMessage, ToolMessage

from analyst import models
from analyst.agent import build_agent
from analyst.workspace import make_backend, upload_sales, upload_skills

ROOT = Path(__file__).resolve().parent.parent


@pytest.fixture
def backend(tmp_path):
    b = make_backend(tmp_path / "ws")
    upload_sales(b)
    upload_skills(b, ROOT / "skills")
    return b


def ask(agent, text):
    return agent.invoke({"messages": [{"role": "user", "content": text}]},
                        config={"recursion_limit": 20})


def load_step(n: int):
    spec = importlib.util.spec_from_file_location(f"step{n}", ROOT / "versions" / f"agent_step{n}.py")
    mod = importlib.util.module_from_spec(spec)
    spec.loader.exec_module(mod)
    return mod.build_agent


def test_model_points_at_openrouter(monkeypatch):
    monkeypatch.setenv("OPENROUTER_API_KEY", "sk-or-dummy")
    m = models.make_model()
    assert "openrouter.ai/api/v1" in str(m.openai_api_base)
    assert m.model_name == models.MODEL


def test_upload_puts_files_at_absolute_paths(backend):
    assert "Widget A" in backend.read("/sales.csv").file_data["content"]
    assert "pandas-patterns" in backend.read("/skills/pandas-patterns/SKILL.md").file_data["content"]


def test_step1_minimal_agent_has_no_file_tools():
    model = FakeToolModel(script=[AIMessage(content="hi")])
    assert ask(load_step(1)(model), "hello")["messages"][-1].content == "hi"
    assert "read_file" not in model.tool_names


def test_step2_filesystem_tools_and_reading_a_file(backend):
    model = FakeToolModel(script=[
        tool_call("read_file", {"file_path": "/sales.csv"}),
        AIMessage(content="Widget A leads."),
    ])
    result = ask(load_step(2)(model, backend), "Summarize /sales.csv")
    assert {"read_file", "write_file", "edit_file", "glob", "grep", "execute"} <= set(model.tool_names)
    tool_msgs = [m for m in result["messages"] if isinstance(m, ToolMessage)]
    assert "Widget A" in tool_msgs[0].content
    assert result["messages"][-1].content == "Widget A leads."


def test_step3_summarization_builds_and_runs(backend):
    model = FakeToolModel(script=[AIMessage(content="ok")])
    assert ask(load_step(3)(model, backend), "hi")["messages"][-1].content == "ok"


def test_step4_skill_is_advertised_in_the_system_prompt(backend):
    model = FakeToolModel(script=[AIMessage(content="ok")])
    ask(load_step(4)(model, backend), "hi")
    system = str(model.seen[0][0].content)
    assert "pandas-patterns" in system and "/skills/" in system
    # progressive disclosure: only name + description are in the prompt, not the full body
    assert "savefig" not in system


def test_step5_adds_todo_and_subagent_tools(backend):
    model = FakeToolModel(script=[AIMessage(content="ok")])
    ask(load_step(5)(model, backend), "hi")
    assert {"write_todos", "task"} <= set(model.tool_names)


def test_agent_runs_real_pandas_in_the_workspace(backend):
    code = (
        "python -c \"import pandas as pd; "
        "print(pd.read_csv('sales.csv').groupby('Product')['Revenue'].sum().to_string())\""
    )
    model = FakeToolModel(script=[
        tool_call("execute", {"command": code}),
        AIMessage(content="Done."),
    ])
    result = ask(build_agent(model, backend), "Total revenue by product?")
    out = [m for m in result["messages"] if isinstance(m, ToolMessage)][0].content
    assert "Widget A" in out and "425" in out and "Widget C" in out


def test_agent_writes_a_report_file(backend):
    model = FakeToolModel(script=[
        tool_call("write_file", {"file_path": "/report.md", "content": "# Report\nWidget A: 425"}),
        AIMessage(content="Wrote it."),
    ])
    ask(build_agent(model, backend), "Write a report")
    assert "Widget A: 425" in backend.read("/report.md").file_data["content"]


def test_main_agent_delegates_to_the_visualizer_subagent(backend):
    model = FakeToolModel(script=[
        tool_call("task", {"description": "Chart revenue by product", "subagent_type": "visualizer"}),
        AIMessage(content="Chart saved as chart.png."),   # the subagent's reply
        AIMessage(content="The visualizer made the chart."),  # the main agent's final answer
    ])
    result = ask(build_agent(model, backend), "Make a chart")
    task_result = [m for m in result["messages"] if isinstance(m, ToolMessage)][0]
    assert "chart.png" in task_result.content
    assert result["messages"][-1].content == "The visualizer made the chart."
```

Run:

```bash
uv run pytest -q
uv run ruff check .
```

Expected: `10 passed` and `All checks passed!`. (A few `DeprecationWarning` lines from a dependency are harmless.)

What the tests prove: each layer adds exactly the tools it should, skills advertise only name and description, the agent can really run pandas in the workspace and write files, and the main agent can hand work to the subagent. What they cannot prove is whether the model makes good choices. That needs live runs (next step).

## Step 9: Run it for real

With your key in `.env` and the finished `agent.py` in place:

```bash
uv run --env-file .env python -m analyst.run
```

Expected behavior (the model's exact wording will vary): it writes a todo list, reads the pandas-patterns skill, computes revenue by product (A 425, B 125, C 90), delegates or writes a chart script, and saves `output.png` (or similar) and `report.md`. The script ends by listing the files in `workspace/`.

Things to try:

- Ask a harder question: `uv run --env-file .env python -m analyst.run "Which day had the highest revenue? Chart units per product."`
- Swap models without code changes: edit `OPENROUTER_MODEL` in `.env` and rerun.
- Open `workspace/report.md` and the PNG to check the work by hand.
- If it loops, lower or raise `recursion_limit` in `run.py` and read the message trace.

Delete the `workspace` folder to reset.

## Step 10: Go to production with a remote sandbox

The local backend is for learning. For anything shared, run the agent's code in an isolated sandbox instead. The agent code does not change, only the backend does. The original tutorial uses a LangSmith sandbox:

```py
from deepagents.backends.langsmith import LangSmithSandbox
from langsmith.sandbox import SandboxClient

client = SandboxClient()
sandbox = client.create_sandbox(name="analyst", snapshot_name="<your-snapshot>")
backend = LangSmithSandbox(sandbox=sandbox)
```

This needs a LangSmith account, `LANGSMITH_API_KEY` set, and a sandbox snapshot that has Python with pandas, matplotlib and seaborn installed. Then pass `backend` to `build_agent` exactly as before and upload files with `backend.upload_files([("/sales.csv", data)])` using absolute POSIX paths. Relative paths are rejected.

> [!WARNING]
> The sandbox snippet above was not run here (it needs a LangSmith account and snapshot). It follows the original tutorial. Check the current LangChain docs for exact arguments.

Other production hardening:

- Spend limit on the OpenRouter key and a `recursion_limit` on every call.
- Never put secrets in the sandbox environment.
- Use `FilesystemMiddleware(permissions=...)` to restrict which paths the agent may write.
- Keep the offline test suite as your CI gate and add a handful of saved live scenarios you review by hand.
- Log which model OpenRouter actually served per request when you compare models.

## Checklist

- [ ] Workspace is a throwaway folder; no secrets in the agent's environment
- [ ] `recursion_limit` set on every invocation
- [ ] Spend limit on the OpenRouter key
- [ ] One model factory; model chosen by environment variable; `.env` in `.gitignore`
- [ ] `deepagents` pinned to the 0.7 series
- [ ] Tests run offline in CI
- [ ] Production uses an isolated sandbox backend, not `LocalShellBackend`

> [!TIP]
> **Verified:** lint clean and 10 tests passing on the latest releases (deepagents 0.7.34, langchain 1.4.9, langchain-openai 1.7.4, pandas 3.1.1, Python 3.14.8), with `uv run --env-file .env` loading the settings and the agent's shell confirmed not to inherit the API key: all five layers build, each adds the right tools, skills appear in the prompt, the agent runs real pandas in the workspace, writes files, and delegates to the visualizer subagent. The OpenRouter client builds with the right base URL.
>
> **Not verified:** a live model run through OpenRouter (no key in the build environment), the exact OpenRouter slug for Fable, summarization actually triggering on a long conversation, and the LangSmith sandbox in step 10.
