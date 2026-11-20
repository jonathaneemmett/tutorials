# Tutorials

Step by step tutorials that build real, tested projects from a blank folder, with install steps, full file paths, and a check after each step.

- **Python AI Native** is a six-chapter path from a Django Ninja API, to an MCP server and agent, to a deep agent, a hand-written agent harness, evals, and agent memory with dreams.
- **Projects** put the chapters to work, starting with an AI second brain built on Obsidian, a nightly dream agent and a Mac mini.
- **Companions** cover the Python and Django underneath: Python 80/20, Python for JavaScript developers, and Django from scratch.

---

# Python AI Native: Build Production AI

A step by step path from a real API to the parts that production AI systems are made of. You build the API first, put an MCP server on it, an agent on that, and then the harness, evals and memory that make the agent trustworthy. Every chapter is a separate tutorial that builds a real, tested project from a blank folder, with install steps, full file paths, and a check after each step.

The path is in the order the chapters should be built, and each one builds on the ideas of the ones above it. Each one also has its own setup steps, so you can start in the middle if you already know the earlier ideas. The one hard dependency is that chapter 2 calls the API from chapter 1, so do those two in order.

---

## Who this is for

You can write Python, or you are coming from JavaScript and want to learn it by building. You want to understand how AI agents work well enough to build, debug and run one, rather than only call an API.

## What every chapter uses

The same small toolkit throughout, so nothing in one chapter surprises you in the next.

| Tool | Why |
| --- | --- |
| uv | Installs Python, creates the project, locks versions, runs everything with `uv run` |
| Python 3.12 or newer | Current language features, current libraries. Django 6.1 in chapter 1 requires 3.12, and the other chapters also run on 3.11 |
| OpenRouter | One API key and one OpenAI-compatible endpoint for any model |
| A `.env` file | Your key and model name live in one file, loaded with `uv run --env-file .env`, and never committed |
| pytest and ruff | Every project ships with passing tests and clean lint |

Each chapter repeats its own setup steps, so you can start at any one.

> [!NOTE]
> Get one OpenRouter key at [openrouter.ai/keys](http://openrouter.ai/keys) and one model name for `OPENROUTER_MODEL`. The tests in these chapters use fake models, so most steps need no key. Only the final live-run step in each chapter spends tokens.

---

## The path

### 1. Build the API First with Django

[Python AI Native: Build the API First with Django (Step by Step)](python-ai-native/01-build-the-api-first-with-django.md)

The data and the rules every later chapter sits on: a provider directory API in Django 6.1 and Django Ninja, with token authentication, per-user visibility, validation, pagination, rate limits, generated OpenAPI docs and 30 tests. **You learn:** why authorization and validation belong in the API and not in the prompt, and how to build and ship a small API properly.

### 2. Build an MCP Server, Agent and Webhook

[Python AI Native: Build an MCP Server, Agent and Webhook (Step by Step)](python-ai-native/02-mcp-server-agent-webhook.md)

Your first AI-native service. You wrap the chapter 1 API as MCP tools, write an agent that calls them through OpenRouter, and receive events through a webhook. **You learn:** tool calling, the MCP protocol, the agent loop in its smallest form.

### 3. Build a Deep Agent from Scratch

[Python AI Native: Build a Deep Agent from Scratch (Step by Step)](python-ai-native/03-deep-agent.md)

A longer-running agent with planning, files and sub-agents, built with LangChain and deepagents. **You learn:** how a framework packages planning, memory and delegation, and what its middleware does.

### 4. Build an Agent Harness from Scratch

[Python AI Native: Build an Agent Harness from Scratch (Step by Step)](python-ai-native/04-agent-harness.md)

The code around the model, written by hand: the loop, tools, permissions, hooks, context compaction, sub-agents and a trace, about 500 lines with 45 tests. **You learn:** what a harness is made of, and where Claude Code and deepagents implement each part.

### 5. Evaluate Your Agent

[Python AI Native: Evaluate Your Agent (Step by Step)](python-ai-native/05-evals.md)

An eval kit on top of the harness: cases, outcome checks, repeated runs, a baseline with a regression gate for CI, and an LLM judge you test before you trust. Adds 29 tests. **You learn:** how to tell whether a change made the agent better or worse, and why outcome checks catch what guardrails miss.

### 6. Give Your Agent Memory and Dreams

[Python AI Native: Give Your Agent Memory and Dreams (Step by Step)](python-ai-native/06-memory-and-dreams.md)

Persistent notes the agent can write and search, saved sessions, and **dreaming**: an offline job that rewrites memory from past sessions into a reviewed copy, never touching the original. Adds 53 tests and a demo that needs no key. **You learn:** how agent memory works, why it must be treated as untrusted input, and how to consolidate it safely.

---

## Projects

Projects put the chapters to work on something you would actually run. Each one lists the chapters it builds on, and each is self-contained, so you can start with it.

### Project 1. An AI Second Brain with Obsidian, a Dream Agent and a Mac mini

[Python AI Native: Build an AI Second Brain with Obsidian, a Dream Agent and a Mac mini (Step by Step)](projects/01-ai-second-brain.md)

An Obsidian vault with a capture agent that files your inbox, a nightly dream agent that proposes merges, links and a morning digest for you to review, and a read-only MCP server so Claude can search your notes. It runs as launchd jobs on a Mac mini, with Obsidian Sync, Tailscale and failure alerts. Adds 51 tests and a demo that needs no key. **Builds on:** chapter 2 (tool calling and MCP), chapter 4 (permission levels) and chapter 6 (dreams you review before adopting). **You learn:** how to run an agent unattended on your own hardware, and how to make a job that reads untrusted text safe to leave running.

---

## Coming next

These chapters are planned and not written yet. They are listed in the order they should be developed: first what the agent can do, then what keeps it safe, then how you watch it, then how you ship it.

- **7. Retrieval over your own data.** Embeddings and semantic search, picking what goes into the context window. Project 1 uses plain keyword search, and this chapter replaces it.
- **8. Guardrails and permissions in depth.** Sandboxing, prompt injection, secrets, and approval flows for real users. This comes before observability and deployment because everything after it puts the agent in front of more people.
- **9. Observe agents in production.** Traces, cost and latency, and finding the one bad run among thousands.
- **10. Deploy it.** Serve an agent behind an API with queues, timeouts and retries.

## Companion tutorials

If Python itself is the gap, these pair well with the path.

- [Python 80/20: Master the 20% You Use Every Day (Step by Step)](companions/python-80-20.md) covers the everyday Python used in all the chapters above.
- [Python for JavaScript Developers: Build a Django REST API (Step by Step)](companions/python-for-javascript-developers.md) maps JavaScript habits to Python and ends with a real API.
- [Django from Scratch (Karpathy-Style): Build a Web Framework in 400 Lines, Then Read the Real One](companions/django-from-scratch.md) rebuilds a web framework to show how they work inside.

## How to work through a chapter

1. Type the code instead of pasting it, at least for the core file of each chapter. The learning is in the typing.
2. Run the check after every step. If a check fails, fix it before moving on, because later steps depend on it.
3. Before the final step, predict what the program will do. Then run it and compare.
4. Change one thing and see what breaks. It is the fastest way to find out whether you understood it.
