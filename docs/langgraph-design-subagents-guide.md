# LangGraph Design Subagents: Best-Practice Guide

A practical guide for coding agents building a **Supervisor → Discovery → Design** system with [LangGraph](https://docs.langchain.com/oss/python/langgraph/overview) and [LangChain Deep Agents](https://docs.langchain.com/oss/python/deepagents/overview) (`deepagents`). Examples are Python, minimal, and labeled **From docs** vs **Adapted** for a Postgres shared-namespace design.

**Packages (current docs):** `deepagents`, `langgraph`, `langchain`, `langgraph-checkpoint-postgres` (for `PostgresStore` / Postgres checkpointer).

---

## Overview / when to use subagents vs one fat agent

### The problem

A single Design agent that loads every platform skill (AKS, RHEL10, …) and all Discovery artifacts will:

- Fill the context window with skill bodies and intermediate tool output
- Lose focus across long multi-platform journeys
- Mix unrelated platform reasoning in one transcript

Deep Agents address this with **filesystem offloading**, optional **skills** (progressive disclosure), optional **todo planning**, and **subagents** (context quarantine via the `task` tool).

### When to use subagents

Use subagents when ([Subagents docs](https://docs.langchain.com/oss/python/deepagents/subagents)):

- Multi-step work would clutter the main agent’s context
- Domains need **different instructions, tools, or skills** (e.g. AKS vs RHEL10)
- You want the parent focused on **orchestration**, not raw research/design dumps
- Parallel specialist work is useful

### When *not* to use subagents

- Simple one-step tasks
- You must keep intermediate tool traces in the parent context
- Overhead of delegation exceeds the benefit

### Subagents vs skills

| Mechanism | What it isolates | Cost |
| --- | --- | --- |
| **Skills** | Full instructions until needed (metadata always in prompt) | Cheap; still one agent context |
| **Subagents** | Entire tool-call history; parent sees final result | Higher; true context quarantine |

**Rule of thumb for Design:** keep platform know-how in **skills**; run each platform (or heavy skill) in a **subagent** so AKS tool noise never pollutes RHEL10 (or the Design supervisor).

---

## Recommended architecture mapped to Supervisor / Discovery / Design

```text
┌─────────────────────────────────────────────────────────────┐
│ Parent LangGraph                                            │
│  nodes: supervisor | discovery | design                     │
│  shared: thread_id, user_id, Postgres store (artifacts)     │
└──────────────┬──────────────────┬───────────────────────────┘
               │                  │
               ▼                  ▼
        Discovery Deep Agent   Design Deep Agent (supervisor)
        - API gather tools     - TodoListMiddleware
        - writes /discovery/   - task() → platform subagents
                               - skills per subagent (aks, rhel10)
               │                  │
               └────────┬─────────┘
                        ▼
              Shared Postgres namespace
              (user_id, thread_id, "artifacts")
              /discovery/...  /design/...  /plans/todos.md
```

| Role | Pattern | Sees | Writes |
| --- | --- | --- | --- |
| **Supervisor** (parent graph node) | Orchestration + user edit requests | User messages, high-level status, summaries from Discovery/Design | Routing decisions; optional edit instructions into shared FS |
| **Discovery** | Deep Agent (often *without* platform skills) | APIs + its tools; thread/user context | `/discovery/*` artifacts in shared store |
| **Design** (supervisor agent) | Deep Agent + todos + `task` tool | Discovery artifacts (read), plan/todos, subagent summaries | `/design/*`, `/plans/*`; delegates via `task` |
| **Platform subagents** | Declarative `SubAgent` or dynamic `task` | One platform skill + relevant discovery files | Platform-scoped design files; return summary only |

**Why this shape**

1. **Parent LangGraph** owns stage routing (discover → design → user edits) — clear, testable edges.
2. **Design Deep Agent** owns *within-design* fan-out (per platform) — Deep Agents’ strength.
3. **Shared Postgres-backed filesystem** is the contract between Discovery and Design (not giant message payloads).

---

## Step-by-step: Deep Agent + skills

### 1. Install and create a Deep Agent

**From docs** ([Overview](https://docs.langchain.com/oss/python/deepagents/overview), [create_deep_agent](https://reference.langchain.com/python/deepagents/deepagents/create_deep_agent)):

```python
from deepagents import create_deep_agent

agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-6",  # prefer explicit model; default is deprecated
    system_prompt="You coordinate design work. Prefer delegation over doing everything yourself.",
)
```

Built-in tools (when middleware/backends support them): filesystem (`ls`, `read_file`, `write_file`, `edit_file`, `glob`, `grep`), optional `execute`, and `task` (when subagents are enabled).

### 2. Add skills (progressive disclosure)

Skills are directories containing `SKILL.md` with YAML frontmatter. At startup the agent only gets **name + description**; it reads the full body when relevant ([Skills](https://docs.langchain.com/oss/python/deepagents/skills)).

**From docs** — layout and creation:

```text
skills/
  aks/
    SKILL.md
    references/
      networking.md
  rhel10/
    SKILL.md
    references/
      hardening.md
```

```markdown
---
name: aks
description: Design user journeys and deployment patterns for Azure Kubernetes Service (AKS). Use when the target platform is AKS or Azure Kubernetes.
---

# AKS design skill

## Instructions
1. Read discovery artifacts under /discovery/ for this thread.
2. Propose journeys specific to AKS (ingress, identity, node pools, ...).
3. Write outputs under /design/aks/.
```

**From docs** — wire skills with a filesystem backend:

```python
from deepagents import create_deep_agent
from deepagents.backends.filesystem import FilesystemBackend

backend = FilesystemBackend(root_dir="./my-project", virtual_mode=True)

design_agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-6",
    backend=backend,
    skills=["./skills/"],  # directory of skill dirs; not a single SKILL.md path
)
```

**Important (docs):**

- Paths use POSIX `/` and are **relative to the backend root**.
- With default `StateBackend`, provide skill files via `invoke(files={...})` using `create_file_data()` — raw strings are not supported.
- **Custom subagents do not inherit** the parent’s `skills`. Only the default `general-purpose` subagent inherits parent skills. Pass `skills=[...]` on each declarative subagent.

**Adapted** — Design supervisor holds *catalog* skills lightly; platform subagents get *one* skill each (see spawning section).

---

## Step-by-step: Todo / planning middleware and shared task list

### Opt-in planning (v0.7+)

Task planning is **opt-in**. Earlier Deep Agents versions included it by default; current docs require `TodoListMiddleware` ([Overview — Task planning](https://docs.langchain.com/oss/python/deepagents/overview), [Todo list UI](https://docs.langchain.com/oss/python/deepagents/frontend/todo-list)).

**From docs:**

```python
from deepagents import create_deep_agent
from langchain.agents.middleware import TodoListMiddleware

design_agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-6",
    middleware=[TodoListMiddleware()],
)
```

This adds a `write_todos` tool and a `todos` **state channel**. Statuses move `pending` → `in_progress` → `completed`. UIs can read `stream.values.todos`.

### Shared task list across agents (docs vs your design)

| What docs guarantee | What they do *not* guarantee |
| --- | --- |
| `todos` live in **that agent’s LangGraph state** when middleware is enabled | Automatic sync of `todos` into Postgres / another agent’s state |
| Subagents sharing the same Deep Agent run share **StateBackend files** with the parent | Cross-node (Discovery ↔ Design) todo sync |

**Adapted best practice for a shared namespace:** treat the **filesystem plan file** as the source of truth across Supervisor / Discovery / Design, and use `TodoListMiddleware` *inside* the Design agent for in-run progress.

```python
# Adapted: Design agent system prompt contract
PLAN_PROMPT = """
Maintain two layers of planning:
1) In-run: use write_todos for the current Design invocation.
2) Shared: keep /plans/design-todos.md in sync (checkbox list) so Supervisor
   and future Design runs can read the same plan from the shared store.
When you spawn platform subagents, assign one todo per platform and mark it
in_progress only while that subagent runs.
"""
```

**Adapted** — optional helper the Design agent (or Supervisor) can call so the shared file stays aligned with state:

```python
# Adapted sketch — not a Deep Agents built-in API
def sync_todos_to_shared_fs(todos: list[dict], backend) -> None:
    lines = ["# Design plan", ""]
    for t in todos:
        mark = "x" if t.get("status") == "completed" else " "
        lines.append(f"- [{mark}] ({t.get('status')}) {t.get('content')}")
    backend.write("/plans/design-todos.md", "\n".join(lines))
```

If you need structured shared todos without files, put a `todos` field on **parent graph state** with a reducer and have Design nodes merge updates — that is a LangGraph state pattern, not Deep Agents middleware.

---

## Step-by-step: Spawning dynamic subagents (per-platform / per-skill)

### Why: context isolation

Subagents solve **context bloat**: the parent receives the **final result**, not dozens of intermediate tool calls ([Deep Agents Subagents](https://docs.langchain.com/oss/python/deepagents/subagents)). Default mode is `mode: "isolated"` — the subagent only sees the delegated task description.

### Pattern A — Declarative platform subagents (recommended starting point)

**From docs:**

```python
from deepagents import create_deep_agent
from langchain.agents.middleware import TodoListMiddleware

aks_subagent = {
    "name": "aks-designer",
    "description": (
        "Design user journeys and topology for Azure Kubernetes Service. "
        "Use when platforms include AKS."
    ),
    "system_prompt": (
        "You design for AKS only. Read /discovery/ as needed. "
        "Write under /design/aks/. Return a concise summary to the parent."
    ),
    "skills": ["/skills/aks/"],  # does NOT inherit parent skills
    # "tools": [...],  # optional; if set, replaces inherited tools entirely
    # "model": "openai:gpt-5.5",  # optional override
}

rhel_subagent = {
    "name": "rhel10-designer",
    "description": (
        "Design user journeys for RHEL 10 targets. "
        "Use when platforms include RHEL10."
    ),
    "system_prompt": (
        "You design for RHEL 10 only. Read /discovery/ as needed. "
        "Write under /design/rhel10/. Return a concise summary."
    ),
    "skills": ["/skills/rhel10/"],
}

design_agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-6",
    system_prompt=(
        "You are the Design supervisor. Decompose work with write_todos. "
        "Delegate each platform to the matching subagent via the task tool. "
        "Do not load every platform skill yourself."
    ),
    middleware=[TodoListMiddleware()],
    subagents=[aks_subagent, rhel_subagent],
    skills=["/skills/design-orchestration/"],  # thin coordinator skill only
)
```

The parent calls the built-in **`task`** tool with `subagent_type` set to the subagent `name`.

### Pattern B — CompiledSubAgent for custom graphs

**From docs** — when the worker is already a compiled LangGraph / `create_agent` graph (must have a `"messages"` state key):

```python
from deepagents import CompiledSubAgent, create_deep_agent
from langchain.agents import create_agent

custom_graph = create_agent(
    model="anthropic:claude-sonnet-4-6",
    tools=[],  # specialized tools
    system_prompt="You are a specialized AKS designer...",
)

aks = CompiledSubAgent(
    name="aks-designer",
    description="Specialized AKS design graph",
    runnable=custom_graph,
)

agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-6",
    subagents=[aks],
)
```

### Pattern C — Dynamic fan-out (“workflow” + code interpreter)

**From docs:** with subagents + `CodeInterpreterMiddleware`, the agent can dispatch many `task()` calls from generated code. Phrasing the user request as a **“workflow”** biases toward this path. Requires `langchain-quickjs>=0.2.0` and Python 3.11+.

```python
from deepagents import create_deep_agent
from langchain_quickjs import CodeInterpreterMiddleware

agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-6",
    subagents=[aks_subagent, rhel_subagent],
    middleware=[CodeInterpreterMiddleware()],
)

# Trigger example from docs style:
# "Run a workflow that designs journeys for each platform in /discovery/platforms.json"
```

Use Pattern C when platforms are **unknown until runtime** (Discovery writes a list; Design fans out). Use Pattern A when the platform set is **fixed and small**.

### Files shared with subagents

**From docs** ([Backends — StateBackend](https://docs.langchain.com/oss/python/deepagents/backends)): with the default `StateBackend`, the filesystem is **shared** between the supervisor agent and its subagents. Files a subagent writes remain available to the parent after the subagent finishes.

For durable cross-thread sharing, use `StoreBackend` / `CompositeBackend` (next section) — subagents of the same agent still see those routes if they use the same backend configuration.

---

## Shared Postgres filesystem / namespace patterns (`thread_id`, `user_id`)

### What Deep Agents provide (documented)

There is **no first-party “PostgresFilesystemBackend”** in Deep Agents docs. The supported durable path is:

1. LangGraph **`BaseStore`** implementation — e.g. **`PostgresStore`**
2. Deep Agents **`StoreBackend`** wrapping that store, with a **`namespace` factory**
3. Optional **`CompositeBackend`** to keep scratch files in `StateBackend` and durable paths in `StoreBackend`

**From docs** — StoreBackend + namespace ([Backends](https://docs.langchain.com/oss/python/deepagents/backends)):

```python
from deepagents import create_deep_agent
from deepagents.backends import StoreBackend
from langgraph.store.memory import InMemoryStore  # swap for PostgresStore in prod

agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-6",
    backend=StoreBackend(
        namespace=lambda rt: (rt.server_info.user.identity,),
    ),
    store=InMemoryStore(),
)
```

Runtime fields for namespaces (`deepagents>=0.5.2` uses `Runtime`):

- `rt.context` — your context schema (e.g. `user_id`)
- `rt.execution_info.thread_id` — conversation/thread
- `rt.server_info` — LangGraph Server metadata when applicable

**From docs** — common patterns:

```python
# Per-thread
StoreBackend(namespace=lambda rt: (rt.execution_info.thread_id,))

# Per-user + per-thread (docs explicitly suggest combining)
# Adapted to context_schema user_id:
StoreBackend(
    namespace=lambda rt: (rt.context.user_id, rt.execution_info.thread_id, "filesystem"),
)
```

**From docs** — Composite routing:

```python
from deepagents.backends import CompositeBackend, StateBackend, StoreBackend

backend = CompositeBackend(
    default=StateBackend(),
    routes={
        "/memories/": StoreBackend(namespace=lambda _rt: ("memories",)),
    },
)
```

Always keep **internal** paths (large tool results, conversation history) on ephemeral `StateBackend` as default; route only intentional artifact prefixes to the store.

### Postgres store wiring

**From docs** ([LangGraph add-memory](https://docs.langchain.com/oss/python/langgraph/add-memory), [Long-term memory](https://docs.langchain.com/oss/python/langchain/long-term-memory)):

```python
# pip install -U "psycopg[binary,pool]" langgraph langgraph-checkpoint-postgres
from langgraph.store.postgres import PostgresStore

DB_URI = "postgresql://postgres:postgres@localhost:5432/postgres?sslmode=disable"

with PostgresStore.from_conn_string(DB_URI) as store:
    store.setup()  # first time
    # pass store=store into create_deep_agent / graph.compile
```

### Adapted layout for Discovery ↔ Design shared artifacts

```text
# Virtual paths (same Postgres store; same namespace factory on both agents)
/discovery/raw/...
/discovery/summary.md
/design/aks/...
/design/rhel10/...
/plans/design-todos.md
```

```python
# Adapted — shared factory used by Discovery and Design agents
from dataclasses import dataclass
from deepagents import create_deep_agent
from deepagents.backends import CompositeBackend, StateBackend, StoreBackend
from langchain.agents.middleware import TodoListMiddleware
from langgraph.store.postgres import PostgresStore

@dataclass
class RunContext:
    user_id: str

def artifact_namespace(rt):
    # Shared across Discovery and Design for this user+thread
    return (rt.context.user_id, rt.execution_info.thread_id, "artifacts")

def make_shared_backend():
    return CompositeBackend(
        default=StateBackend(),
        routes={
            "/discovery/": StoreBackend(namespace=artifact_namespace),
            "/design/": StoreBackend(namespace=artifact_namespace),
            "/plans/": StoreBackend(namespace=artifact_namespace),
            # optional: skills from store or leave on FilesystemBackend elsewhere
        },
    )

def build_discovery_agent(store: PostgresStore):
    return create_deep_agent(
        model="anthropic:claude-sonnet-4-6",
        name="discovery",
        system_prompt="Gather artifacts from APIs. Write under /discovery/ only.",
        backend=make_shared_backend(),
        store=store,
        context_schema=RunContext,
        # typically no heavy design skills
    )

def build_design_agent(store: PostgresStore, subagents: list):
    return create_deep_agent(
        model="anthropic:claude-sonnet-4-6",
        name="design",
        system_prompt="Read /discovery/. Plan with todos. Delegate platforms via task.",
        middleware=[TodoListMiddleware()],
        backend=make_shared_backend(),
        store=store,
        context_schema=RunContext,
        subagents=subagents,
    )
```

Invoke with the same `thread_id` and `context=RunContext(user_id=...)` so namespaces align.

### Custom Postgres VFS (if StoreBackend is not enough)

**From docs** — implement `BackendProtocol` (`ls`, `read`, `write`, `edit`, `glob`, `grep`, optional `delete`); return result objects with `error` fields — do not raise ([Backends — Custom](https://docs.langchain.com/oss/python/deepagents/backends)). Community backends exist (Azure Blob, MongoDB VFS); a dedicated Postgres row-store VFS would follow the same protocol. Prefer **`PostgresStore` + `StoreBackend`** unless you need a different schema.

### Permissions (narrow what subagents can touch)

**From docs** — `permissions=` on `create_deep_agent` / per-subagent `permissions` ([create_deep_agent](https://reference.langchain.com/python/deepagents/deepagents/create_deep_agent)). Subagent `permissions` **replace** parent rules entirely when set. Use this so RHEL subagents cannot write `/design/aks/`.

---

## Supervisor orchestration patterns (LangGraph parent graph + agent nodes)

You have **two** supervisor layers; keep them distinct:

1. **Parent graph Supervisor** — stages: Discovery vs Design vs user-edit loops  
2. **Design Deep Agent** — platforms/skills via `task`

### Option 1 — Deep Agents as subgraph nodes (shared `messages`)

**From docs** ([Subgraphs](https://docs.langchain.com/oss/python/langgraph/use-subgraphs)): if parent and subgraph share keys, pass the compiled graph to `add_node`.

```python
# Adapted — create_deep_agent returns a CompiledStateGraph
from typing import Literal
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

class ParentState(TypedDict):
    messages: list
    stage: Literal["discover", "design", "done"]

discovery = build_discovery_agent(store)  # CompiledStateGraph
design = build_design_agent(store, subagents=[...])

def route(state: ParentState) -> Literal["discovery", "design", "__end__"]:
    if state["stage"] == "discover":
        return "discovery"
    if state["stage"] == "design":
        return "design"
    return "__end__"

builder = StateGraph(ParentState)
builder.add_node("discovery", discovery)
builder.add_node("design", design)
builder.add_conditional_edges(START, route, ["discovery", "design", END])
# After discovery, set stage=design in a small node or Command update — Adapted
parent = builder.compile(store=store, checkpointer=checkpointer)
```

If schemas differ, **call the subgraph inside a node** and map state explicitly (same LangGraph docs page).

### Option 2 — Tool-wrapped workers (LangChain multi-agent “subagents” pattern)

**From docs** ([LangChain multi-agent subagents](https://docs.langchain.com/oss/python/langchain/multi-agent/subagents), [migrate from langgraph-supervisor](https://docs.langchain.com/oss/python/migrate/langgraph-supervisor)): wrap specialists as `@tool`s on a main `create_agent`. Prefer Deep Agents’ built-in `task` **inside** Design; use tool-wrapping at the **parent** if you want a thin Supervisor without Deep Agents features.

### Handoffs

**From docs** ([Handoffs](https://docs.langchain.com/oss/python/langchain/multi-agent/handoffs)): use `Command(goto=..., graph=Command.PARENT)` when agents must transfer control as graph nodes. For Supervisor → Discovery → Design **pipelines**, conditional edges are usually clearer than handoff tools. Use handoffs when the **user** converses with different agents turn-by-turn.

### User edit requests

**Adapted pattern:** Supervisor node receives “please change the AKS journey”; it writes `/plans/edit-request.md` (or appends to messages) and re-enters the Design node with a focused instruction so Design re-runs only the `aks-designer` subagent via `task`.

---

## Context isolation checklist

| Actor | Should see | Should *not* see |
| --- | --- | --- |
| Parent Supervisor | User intent, stage, short summaries, edit requests | Full Discovery API dumps, full skill bodies, per-platform tool traces |
| Discovery agent | API tools, `/discovery/` write access, thread/user ids | Design skills (AKS/RHEL), `/design/*` writes (optional deny via permissions) |
| Design supervisor | `/discovery/` reads, `/plans/`, subagent **summaries**, thin orchestration skill | Full body of every platform skill at once; raw subagent tool traces |
| Platform subagent | One platform skill, relevant `/discovery/` files, `/design/<platform>/` | Sibling platform skills; other platforms’ design drafts; unrelated user chit-chat (isolated mode) |
| All | Same `(user_id, thread_id, …)` store namespace for shared artifacts | Other users’ namespaces |

**Checklist actions**

- [ ] Design parent skills list is thin; platform skills only on subagents  
- [ ] Subagents use default `mode: "isolated"` unless you explicitly need `fork`  
- [ ] `permissions` deny cross-platform writes  
- [ ] Large tool results stay on filesystem / offload — not pasted into Supervisor messages  
- [ ] Todos: in-run state *plus* `/plans/` file if cross-node visibility is required  
- [ ] Do not put secrets in shared artifact paths without deny rules  

---

## Minimal end-to-end skeleton

**Labeling:** structure and Postgres/shared paths are **Adapted**; `create_deep_agent`, `TodoListMiddleware`, `SubAgent` dicts, `CompositeBackend` / `StoreBackend`, and subgraph `add_node` are **From docs**.

```python
"""Minimal Supervisor → Design supervisor → platform subagents skeleton."""

from dataclasses import dataclass
from typing import Annotated, Literal

from deepagents import create_deep_agent
from deepagents.backends import CompositeBackend, StateBackend, StoreBackend
from langchain.agents.middleware import TodoListMiddleware
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.store.postgres import PostgresStore
from typing_extensions import TypedDict

MODEL = "anthropic:claude-sonnet-4-6"
DB_URI = "postgresql://postgres:postgres@localhost:5432/postgres?sslmode=disable"


@dataclass
class RunContext:
    user_id: str


class ParentState(TypedDict):
    messages: Annotated[list, add_messages]
    stage: Literal["discover", "design", "done"]


def ns(rt):
    return (rt.context.user_id, rt.execution_info.thread_id, "artifacts")


def shared_backend():
    return CompositeBackend(
        default=StateBackend(),
        routes={
            "/discovery/": StoreBackend(namespace=ns),
            "/design/": StoreBackend(namespace=ns),
            "/plans/": StoreBackend(namespace=ns),
            "/skills/": StoreBackend(namespace=lambda rt: ("skills",)),  # or FilesystemBackend
        },
    )


aks = {
    "name": "aks-designer",
    "description": "AKS platform design. Use for Azure Kubernetes targets.",
    "system_prompt": "AKS only. Read /discovery/. Write /design/aks/. Summarize for parent.",
    "skills": ["/skills/aks/"],
}

rhel = {
    "name": "rhel10-designer",
    "description": "RHEL10 platform design. Use for RHEL 10 targets.",
    "system_prompt": "RHEL10 only. Read /discovery/. Write /design/rhel10/. Summarize for parent.",
    "skills": ["/skills/rhel10/"],
}


def build_graph(store: PostgresStore):
    discovery = create_deep_agent(
        model=MODEL,
        name="discovery",
        system_prompt="Collect artifacts; write markdown under /discovery/.",
        backend=shared_backend(),
        store=store,
        context_schema=RunContext,
    )

    design = create_deep_agent(
        model=MODEL,
        name="design",
        system_prompt=(
            "Design supervisor. write_todos for platforms. "
            "Delegate with task to aks-designer / rhel10-designer. "
            "Mirror the plan to /plans/design-todos.md."
        ),
        middleware=[TodoListMiddleware()],
        backend=shared_backend(),
        store=store,
        context_schema=RunContext,
        subagents=[aks, rhel],
    )

    def after_discovery(state: ParentState) -> dict:
        return {"stage": "design"}

    def after_design(state: ParentState) -> dict:
        return {"stage": "done"}

    def router(state: ParentState) -> str:
        return {
            "discover": "discovery",
            "design": "design",
            "done": END,
        }[state["stage"]]

    g = StateGraph(ParentState)
    g.add_node("discovery", discovery)
    g.add_node("mark_design", after_discovery)
    g.add_node("design", design)
    g.add_node("mark_done", after_design)
    g.add_conditional_edges(START, router, ["discovery", "design", END])
    g.add_edge("discovery", "mark_design")
    g.add_edge("mark_design", "design")
    g.add_edge("design", "mark_done")
    g.add_edge("mark_done", END)
    return g.compile(store=store)


# Usage sketch:
# with PostgresStore.from_conn_string(DB_URI) as store:
#     store.setup()
#     app = build_graph(store)
#     app.invoke(
#         {"messages": [...], "stage": "discover"},
#         config={"configurable": {"thread_id": "t1"}},
#         context=RunContext(user_id="u1"),
#     )
```

**Uncertainty:** exact parent-graph edge helpers (`mark_design`) and whether your deployed runtime passes `execution_info.thread_id` the same way as LangGraph Server should be verified in your environment (`deepagents>=0.5.2` Runtime API). If `thread_id` is missing on `rt.execution_info`, fall back to reading it from the invoke `config` inside a custom namespace factory — confirm against your installed package.

---

## Pitfalls and anti-patterns

1. **One fat Design agent with all skills loaded “just in case”** — skills metadata still costs tokens; large bodies get pulled into one context. Prefer per-platform subagents.
2. **Assuming custom subagents inherit `skills=`** — they do not; only `general-purpose` inherits. Always set `skills` on each platform subagent.
3. **Assuming `TodoListMiddleware` syncs across Discovery and Design nodes** — it does not. Use shared files or parent state.
4. **Putting `FilesystemBackend` alone on a web server** — docs warn against this; use `StateBackend` / `StoreBackend` / sandbox.
5. **Routing everything to `StoreBackend` including `/large_tool_results/`** — keep Composite default as `StateBackend`.
6. **Forgetting `store.setup()`** on first Postgres use.
7. **Omitting namespace factories** — legacy default can share storage across all users of an assistant.
8. **Using `mode: "fork"` for platform work** — pulls parent history into the subagent; use isolated mode for quarantine.
9. **Relying on deprecated `langgraph-supervisor`** — migrate to subagents / Deep Agents `task` ([migration guide](https://docs.langchain.com/oss/python/migrate/langgraph-supervisor)).
10. **Passing `skills` as a path to `SKILL.md` itself** — sources must be directories that *contain* skill directories.
11. **Inventing a non-existent `PostgresBackend` import from `deepagents`** — use `PostgresStore` + `StoreBackend`, or implement `BackendProtocol`.
12. **Expecting `task` without subagents** — disable path requires `GeneralPurposeSubagentProfile(enabled=False)` *and* no sync subagents; otherwise the default general-purpose subagent remains.

---

## Doc links appendix (verified)

| Topic | URL |
| --- | --- |
| Deep Agents overview | https://docs.langchain.com/oss/python/deepagents/overview |
| `create_deep_agent` reference | https://reference.langchain.com/python/deepagents/deepagents/create_deep_agent |
| Subagents (Deep Agents) | https://docs.langchain.com/oss/python/deepagents/subagents |
| Skills | https://docs.langchain.com/oss/python/deepagents/skills |
| Backends / Composite / Store / custom protocol | https://docs.langchain.com/oss/python/deepagents/backends |
| Customize Deep Agents | https://docs.langchain.com/oss/python/deepagents/customization |
| Context engineering | https://docs.langchain.com/oss/python/deepagents/context-engineering |
| Todo list (frontend + middleware) | https://docs.langchain.com/oss/python/deepagents/frontend/todo-list |
| Prebuilt middleware (`TodoListMiddleware`) | https://docs.langchain.com/oss/python/langchain/middleware/built-in |
| Backend integrations catalog | https://docs.langchain.com/oss/python/integrations/backends |
| LangGraph subgraphs | https://docs.langchain.com/oss/python/langgraph/use-subgraphs |
| LangGraph Graph API / `Command` | https://docs.langchain.com/oss/python/langgraph/graph-api |
| LangGraph add memory / PostgresStore | https://docs.langchain.com/oss/python/langgraph/add-memory |
| Long-term memory / PostgresStore | https://docs.langchain.com/oss/python/langchain/long-term-memory |
| Multi-agent subagents (LangChain) | https://docs.langchain.com/oss/python/langchain/multi-agent/subagents |
| Handoffs | https://docs.langchain.com/oss/python/langchain/multi-agent/handoffs |
| Migrate from langgraph-supervisor | https://docs.langchain.com/oss/python/migrate/langgraph-supervisor |

---

## API uncertainties (read before coding)

1. **Shared todos in Postgres:** Not a built-in Deep Agents feature; use `/plans/` files or parent graph state (adapted).
2. **`rt.execution_info.thread_id` / `rt.context.user_id`:** Documented for `deepagents>=0.5.2`; older 0.5.x used `BackendContext`. Confirm against your installed version.
3. **Skills on StoreBackend paths:** Docs support StoreBackend and uploading skill files; exact upload bootstrap for Postgres-backed skills should be validated in a small spike.
4. **Dynamic subagents via code interpreter:** Documented but optional (`langchain-quickjs`); declarative `subagents=` is enough for a fixed platform set.
5. **Embedding three Deep Agents in one parent graph:** Supported via subgraph patterns in principle (`create_deep_agent` returns `CompiledStateGraph`); shared-state vs mapped-state depends on whether you unify on `messages` and custom channels — test checkpointing and store injection on the parent `compile(store=...)`.
