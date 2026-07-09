# Agent Spawning & Delegation

Hermes can create additional agent instances to work in parallel. There are two distinct mechanisms for this: **synchronous delegation** (`delegate_task`) and **asynchronous process spawning** (running a new `hermes` process). This chapter explains both, their isolation boundaries, and when each is used.

## 3.1 Two Mechanisms Compared

| Dimension | `delegate_task` | Spawning `hermes` process |
|-----------|----------------|----------------------------|
| **Isolation** | Separate conversation, shared process | Fully independent OS process |
| **Duration** | Minutes (bounded by parent loop) | Hours or days |
| **Tool access** | Subset of parent's tools | Full tool access |
| **Interactivity** | No — batch only | Yes — supports PTY mode |
| **Parent blocking** | Parent waits for summary | Parent continues immediately |
| **Use case** | Quick parallel subtasks | Long autonomous missions |

## 3.2 Synchronous Delegation (`delegate_task`)

Implemented in the `delegation` toolset (exposed as `delegate_task`), this is the lighter-weight of the two mechanisms.

### Invocation

```python
# Single task
delegate_task(goal="Research GRPO papers", context="Focus on 2024+")

# Batch (parallel)
delegate_task(tasks=[
    {"goal": "Backend API design", "context": "FastAPI"},
    {"goal": "Frontend scaffolding", "context": "React"},
])
```

### Lifecycle

1. **Parent sends goal + context** — The parent Hermes instance calls `delegate_task`.
2. **Child agent instantiated** — A new `AIAgent` is created with:
   - Isolated conversation history (empty except for the system prompt and the goal)
   - The parent's full toolset — subagents **always inherit the parent's toolsets** automatically. The model-facing `toolsets` argument was removed; the child gets the parent's tool definitions without the model needing to specify them. MCP toolsets are inherited by default (`delegation.inherit_mcp_toolsets`, default true).
   - Its own `IterationBudget` capped at `delegation.max_iterations` (default 50)
3. **Child runs independently** — The child executes the loop until it returns text or hits its iteration budget.
4. **Summary returned** — The child returns a condensed summary (not the full conversation). The parent receives this summary as a tool result.
5. **Parent resumes** — The parent sees the summary in its next loop iteration and continues.

### Security: No Model-Controlled ACP Transport

The delegate ACP (Agent Context Protocol) transport was removed from model control. The model cannot choose to spawn subagents via ACP — delegation is always handled through the internal `delegate_task` tool path, preventing a compromised model from exfiltrating context through an ACP channel.

### Isolation Guarantees

- **Conversation isolation** — The child cannot see the parent's history, and vice versa.
- **Terminal isolation** — The child gets its own terminal session (separate working directory and shell state).
- **Memory isolation** — By default, `skip_memory=True` prevents the child from reading or writing persistent memory. The child still gets the user's profile injected in its system prompt.
- **Process isolation** — The child runs in the same Python process but in a separate thread. If the parent is interrupted, the child is cancelled.

### Roles: Leaf vs Orchestrator

- **Leaf** (default) — A focused worker that cannot re-delegate. Suitable for single concrete tasks.
- **Orchestrator** — Can call `delegate_task` itself to spawn further workers, bounded by `delegation.max_spawn_depth`. Useful for hierarchical task decomposition. (Note: orchestrator spawning is currently disabled by default — `max_spawn_depth=1` — and can be enabled via config.)

### Configuration

```yaml
delegation:
  model: "gpt-4o-mini"        # Model for subagents (can differ from parent)
  provider: "openai"
  base_url: ""
  api_key: ""
  max_iterations: 50
  max_concurrent_children: 3    # Parallel batch limit
  reasoning_effort: "medium"
```

## 3.3 Asynchronous Process Spawning

For work that must outlive the current turn, Hermes can spawn a completely new OS process running `hermes chat -q` or an interactive `hermes` session inside tmux.

### One-Shot Mode

Fire-and-forget for tasks that run unattended:

```python
terminal(
    command="hermes chat -q 'Set up CI/CD for ~/myapp'",
    timeout=300
)
```

### Background Mode

Runs silently with notification on completion:

```python
terminal(
    command="hermes chat -q 'Monitor trading bot'",
    background=True,
    notify_on_complete=True
)
```

### Interactive PTY Mode (tmux)

For tasks requiring interactive prompting or long-running TUI sessions:

```python
# Start
terminal(command="tmux new-session -d -s agent1 -x 120 -y 40 'hermes'")

# Send work
terminal(command="tmux send-keys -t agent1 'Build auth service' Enter")

# Read output
terminal(command="tmux capture-pane -t agent1 -p")

# Clean up
terminal(command="tmux send-keys -t agent1 '/exit' Enter")
terminal(command="tmux kill-session -t agent1")
```

### Multi-Agent Coordination

Spawning is used for parallel backend/frontend development, where two agents work on the same codebase but in different tmux sessions, with a human or a coordinator agent relaying context between them.

## 3.4 Worktree Mode (`-w`)

When spawning agents that edit code, use the `-w` flag:

```bash
hermes -w
```

This creates an isolated git worktree, preventing conflicts when two agents modify the same repository. Each spawned agent gets its own branch and working directory while sharing the same `.git` object database.

## 3.5 Why Spawn?

Hermes spawns agents for three architectural reasons:

1. **Capacity** — A single agent is bounded by `max_turns` (default 90). Complex multi-step tasks burn through this budget quickly. Spawning allows horizontal scaling of reasoning steps.
2. **Parallelism** — Some tasks are naturally parallel (research multiple topics, audit multiple files). Delegation runs subagents concurrently.
3. **Durability** — The parent conversation may end (user disconnects, gateway restarts). A spawned process survives because it's independent.

## 3.6 Kanban: The Durable Work Queue

For durable multi-agent coordination, Hermes provides `kanban` — a SQLite-backed work queue with dispatcher-driven worker spawning.

### Architecture

```
┌─────────────┐      ┌──────────────┐      ┌──────────────┐
│   Board     │◄────►│  Dispatcher  │◄────►│   Workers    │
│  (SQLite)   │      │  (gateway)   │      │  (spawned)   │
└─────────────┘      └──────────────┘      └──────────────┘
```

1. Tasks are created on the board with `kanban_create`.
2. The dispatcher (running inside the gateway) reclaims stale claims and promotes ready tasks.
3. Workers are spawned with `HERMES_KANBAN_TASK` set in their environment.
4. Workers see a focused `kanban_*` toolset to update task status.
5. After ~5 consecutive spawn failures on a task, the dispatcher auto-blocks it (circuit breaker).

Workers are fully isolated: they get `HERMES_KANBAN_BOARD` pinned in their environment, and the board ID is the hard boundary for context sharing.

## 3.7 Spawning Diagram

```
┌─────────────────┐
│  Parent Agent   │
│  (main loop)    │
└─────────────────┘
         │
    ┌────┴────┬─────────────┐
    │         │             │
    ▼         ▼             ▼
┌────────┐ ┌────────┐ ┌────────────┐
│delegate│ │ tmux   │ │  cronjob   │
│_task   │ │ spawn  │ │  scheduler │
└────────┘ └────────┘ └────────────┘
    │         │             │
    ▼         ▼             ▼
┌────────┐ ┌────────┐ ┌────────────┐
│ Leaf   │ │ Full   │ │  Script or │
│ Agent  │ │ Hermes │ │  LLM agent │
│ (sync) │ │ (async)│ │  (timed)   │
└────────┘ └────────┘ └────────────┘
```
