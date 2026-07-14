# The Hermes Memory System

Hermes remembers in three layers: a fast, curated, file-backed **core memory** that rides in the system prompt; a durable **session store** indexed by SQLite FTS5 that the agent can search at zero LLM cost; and an optional **pluggable external memory backend** that trades speed for cross-service persistence. All three are coordinated by a single orchestrator, the `MemoryManager`, so that the agent never pays for more than one memory tax per turn.

> Design principle: *per-conversation prompt caching is sacred.*  Any component that mutates past context, swaps toolsets, or rebuilds the system prompt mid-conversation invalidates the cache and multiplies the user's cost. The memory system therefore **freezes the system-prompt snapshot for the entire session**; mid-session writes land on disk (and in external backends) but do not touch the prompt prefix.

---

## 1. Architecture at a Glance

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              AIAgent (run_agent.py)                         │
│  ┌─────────────────────┐  ┌──────────────────┐  ┌────────────────────────┐ │
│  │   MemoryStore       │  │   SessionDB      │  │   MemoryManager        │ │
│  │  (tools/memory_tool)│  │ (hermes_state.py)│  │ (agent/memory_manager) │ │
│  │                     │  │                  │  │                        │ │
│  │  MEMORY.md          │  │  state.db        │  │  builtin provider      │ │
│  │  USER.md            │  │  FTS5 + trigram  │  │  + 0..1 external       │ │
│  └────────┬────────────┘  └────────┬─────────┘  └───────┬────────────────┘ │
│           │                        │                    │                  │
│           ▼                        ▼                    ▼                  │
│     System prompt              session_search        sync_all /              │
│     (frozen snapshot)          tool (FTS5)          prefetch_all            │
│                                                              │              │
│                                                              ▼              │
│                                                     External provider(s)    │
│                                                     (Honcho, Mem0, ...)     │
└─────────────────────────────────────────────────────────────────────────────┘
```

The agent interacts with memory through four surfaces:

| Surface | Lives in | Scope | Cost |
|---|---|---|---|
| Core memory (curated notes) | `tools/memory_tool.py` — `MemoryStore` | Global, across all sessions | File I/O only |
| Session storage | `hermes_state.py` — `SessionDB` | Per-session messages & metadata | SQLite + FTS5 |
| Session search | `tools/session_search_tool.py` | Cross-session recall | Zero LLM cost |
| External memory providers | `plugins/memory/<name>/` | Backend-specific persistence | Network/daemon (async) |

---

## 2. Core Memory: Curated, Compact, Prompt-Resident

The built-in memory is the oldest and most stable layer. It consists of two plain-text files living under `$HERMES_HOME/memories/`:

- `MEMORY.md` — the agent’s own notes (project conventions, learned facts, tool quirks, environment state).
- `USER.md` — what the agent knows about the user (preferences, style, workflow habits).

These files are read **once** at agent startup, parsed into entries delimited by the section sign `§`, and injected into the system prompt as a frozen snapshot. Because the snapshot never changes mid-session, the API provider’s prompt-cache prefix remains byte-identical across every turn.

### 2.1 MemoryStore (`tools/memory_tool.py`)

`MemoryStore` is a small, bounded, file-backed append-only (plus replace/remove) store. Its design priorities are:

1. **Bounded size** — character limits, not tokens, because character counts are model-independent. Defaults: `2200` chars for `MEMORY.md`, `1375` for `USER.md`.
2. **Atomic durability** — writes use `tempfile.mkstemp` + `os.replace` so readers always see either the old complete file or the new one, never a half-written buffer.
3. **External-drift detection** — because users may append to `MEMORY.md` via the `patch` tool, shell redirection, or a concurrent session, `MemoryStore` detects non-roundtrippable content and refuses the mutation, surfacing a backup path so the operator can reconcile.
4. **Security scanning** — every `add`/`replace` is scanned with the shared `tools/threat_patterns.py` library at `"strict"` scope, and load-time entries are also screened so a poisoned on-disk file cannot inject adversarial prompts into the system prompt. Blocked entries are replaced with `[BLOCKED: …]` placeholders in the snapshot but kept in live state so the user can inspect and remove them.

#### Entry lifecycle

Entries are stored as a flat list separated by `\n§\n`. Deduplication (`dict.fromkeys`) runs at load time. The four tool actions are `add`, `replace`, `remove`, and `read`. `replace` and `remove` use substring matching rather than IDs — the model refers to `"old_text"` and the store finds the first entry containing it. When multiple entries match, the tool returns an error with previews so the caller can disambiguate.

#### Batch operations

The `apply_batch(target, operations)` method enables multiple `add`/`replace`/`remove` operations in a single atomic call. The tool exposes this via the `operations` array parameter — each item is `{action, content?, old_text?}`. The batch works on a copy and only commits to disk if every operation validates, so a single poisoned op rejects the whole batch without corrupting the file.

#### Snapshot vs. live state

```python
# Frozen at load_from_disk() — injected into system prompt
_system_prompt_snapshot: {"memory": "…", "user": "…"}

# Mutated by tool calls — persisted to disk immediately
memory_entries: List[str]
user_entries:   List[str]
```

`format_for_system_prompt(target)` returns the snapshot, while `memory_tool(action="read")` returns the live state. This split is the caching contract in action.

### 2.2 Tool wiring (`tools/memory_tool.py` → `model_tools.py`)

The `memory` tool is one of four *agent-level tools* intercepted before registry dispatch because it needs the agent’s live `MemoryStore` instance. `run_agent.py` injects the store into the tool handler; the tool never instantiates its own store.

#### Write-approval gate (`tools/write_approval.py`)

When `memory.write_approval: true` is set in config, mutating actions (`add`, `replace`, `remove`) are gated:

- **Interactive CLI** — an inline `approve / deny / stage` prompt blocks the turn.
- **Gateway / background / script** — the write is *staged* to `$HERMES_HOME/pending/memory/<id>.json` and surfaced via `/memory pending` for later review.

Staging prevents autonomous runs from silently poisoning memory.

---

## 3. Session Storage: The SQLite State Machine (`hermes_state.py`)

Every message, session boundary, compression event, and tool call is persisted to `state.db`, a SQLite database under `$HERMES_HOME`. It replaces the older per-session JSONL files and powers both analytics (dashboard, TUI sidebar) and the zero-cost `session_search` recall tool.

### 3.1 Schema

Five canonical tables plus two FTS5 virtual tables:

```sql
CREATE TABLE sessions (
    id TEXT PRIMARY KEY,
    source TEXT NOT NULL,          -- 'cli', 'telegram', 'discord', 'cron', ...
    user_id TEXT,
    model TEXT,
    model_config TEXT,             -- JSON blob
    system_prompt TEXT,
    parent_session_id TEXT,
    started_at REAL NOT NULL,
    ended_at REAL,
    end_reason TEXT,               -- 'compression', 'branched', 'user_exit', ...
    message_count INTEGER DEFAULT 0,
    tool_call_count INTEGER DEFAULT 0,
    input_tokens INTEGER DEFAULT 0,
    output_tokens INTEGER DEFAULT 0,
    cache_read_tokens INTEGER DEFAULT 0,
    cache_write_tokens INTEGER DEFAULT 0,
    reasoning_tokens INTEGER DEFAULT 0,
    cwd TEXT,
    billing_provider TEXT,
    billing_base_url TEXT,
    billing_mode TEXT,
    estimated_cost_usd REAL,
    actual_cost_usd REAL,
    cost_status TEXT,
    cost_source TEXT,
    pricing_version TEXT,
    title TEXT,
    api_call_count INTEGER DEFAULT 0,
    handoff_state TEXT,
    handoff_platform TEXT,
    handoff_error TEXT,
    compression_fallback_streak INTEGER NOT NULL DEFAULT 0,
    rewind_count INTEGER NOT NULL DEFAULT 0,
    archived INTEGER NOT NULL DEFAULT 0,
    FOREIGN KEY (parent_session_id) REFERENCES sessions(id)
);

CREATE TABLE messages (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT NOT NULL REFERENCES sessions(id),
    role TEXT NOT NULL,
    content TEXT,
    tool_call_id TEXT,
    tool_calls TEXT,               -- JSON blob
    tool_name TEXT,
    effect_disposition TEXT,       -- disposition of a tool's effect (v21)
    timestamp REAL NOT NULL,
    token_count INTEGER,
    finish_reason TEXT,
    reasoning TEXT,
    reasoning_content TEXT,
    reasoning_details TEXT,
    codex_reasoning_items TEXT,
    codex_message_items TEXT,
    platform_message_id TEXT,
    observed INTEGER DEFAULT 0,
    active INTEGER NOT NULL DEFAULT 1
);

CREATE TABLE session_model_usage (
    session_id TEXT NOT NULL REFERENCES sessions(id) ON DELETE CASCADE,
    model TEXT NOT NULL,
    billing_provider TEXT NOT NULL DEFAULT '',
    billing_base_url TEXT NOT NULL DEFAULT '',
    billing_mode TEXT NOT NULL DEFAULT '',
    api_call_count INTEGER NOT NULL DEFAULT 0,
    input_tokens INTEGER NOT NULL DEFAULT 0,
    output_tokens INTEGER NOT NULL DEFAULT 0,
    cache_read_tokens INTEGER NOT NULL DEFAULT 0,
    cache_write_tokens INTEGER NOT NULL DEFAULT 0,
    reasoning_tokens INTEGER NOT NULL DEFAULT 0,
    estimated_cost_usd REAL NOT NULL DEFAULT 0,
    actual_cost_usd REAL NOT NULL DEFAULT 0,
    cost_status TEXT,
    cost_source TEXT,
    first_seen REAL,
    last_seen REAL,
    PRIMARY KEY (session_id, model, billing_provider, billing_base_url, billing_mode)
);

CREATE TABLE state_meta (
    key TEXT PRIMARY KEY,
    value TEXT
);

CREATE TABLE compression_locks (
    session_id TEXT PRIMARY KEY,
    holder TEXT NOT NULL,
    acquired_at REAL NOT NULL,
    expires_at REAL NOT NULL
);

CREATE TABLE async_delegations (
    delegation_id TEXT PRIMARY KEY,
    origin_session TEXT NOT NULL,
    parent_session_id TEXT,
    state TEXT NOT NULL,
    dispatched_at REAL NOT NULL,
    completed_at REAL,
    updated_at REAL NOT NULL,
    event_json TEXT,
    result_json TEXT,
    delivery_state TEXT NOT NULL DEFAULT 'pending',
    delivery_attempts INTEGER NOT NULL DEFAULT 0,
    delivered_at REAL,
    owner_pid INTEGER,
    owner_started_at INTEGER,
    task_json TEXT,
    delivery_claim TEXT,
    delivery_claimed_at REAL
);
```

Two FTS5 virtual tables index message content:

- `messages_fts` — standard `unicode61` tokenizer for English/ASCII search.
- `messages_fts_trigram` — `tokenize='trigram'` for CJK substring search.

Both index `content || ' ' || tool_name || ' ' || tool_calls` so tool-call arguments and tool-result summaries are searchable alongside prose.

### 3.1.1 Per-model usage attribution (`session_model_usage`, v20)

The `sessions` row stores only one `(model, billing_provider)` pair, so a mid-session `/model` switch would attribute every token to the initial model (issue #51607). Schema v20 introduces the `session_model_usage` table to record per-API-call token deltas keyed by `(session_id, model, billing_provider, billing_base_url, billing_mode)`.

`update_token_counts()` now records each incremental call's delta via `_record_model_usage()`, which runs inside the caller's write transaction (after the `sessions` UPDATE) so per-model rows stay consistent with the summary row. When the caller omits the model/provider, `_record_model_usage()` falls back to the values already recorded on the session row — the same COALESCE-from-session behaviour the summary update uses. Absolute cumulative updates (which cannot be split back into per-model routes) are skipped; Insights reconciles any positive residual against the aggregate session row instead.

The v20 migration seeds one usage row per historical session from the aggregate totals on `sessions` using `INSERT OR IGNORE` so it is idempotent: if newer code already wrote a `(session_id, model, provider)` row, the PK conflict skips the stale aggregate rather than doubling it.

### 3.1.2 Durable delegation persistence (`async_delegations`, v21)

The `async_delegations` table provides durable persistence for asynchronous task delegations. Each row records a delegation's lifecycle from dispatch through delivery:

- `delegation_id` — unique identifier for the delegation.
- `origin_session` / `parent_session_id` — the session that spawned the delegation.
- `state` — current delegation state (e.g. dispatched, completed).
- `dispatched_at` / `completed_at` / `updated_at` — lifecycle timestamps.
- `event_json` / `result_json` / `task_json` — the original task, its result, and the triggering event.
- `delivery_state` / `delivery_attempts` / `delivery_claim` / `delivery_claimed_at` — delivery tracking for returning results to the origin session.
- `owner_pid` / `owner_started_at` — the process that owns the delegation at any given time.

An index on `(delivery_state, completed_at)` supports efficient polling for pending deliveries.

### 3.1.3 `effect_disposition` on messages

The `messages` table carries an `effect_disposition TEXT` column (v21) that tracks the disposition of a tool's effect — for example, whether a tool result was a side effect (e.g. a file written, a command executed) or a direct response (e.g. a query result returned inline). `add_message()` accepts `effect_disposition` as an optional parameter and persists it alongside the message. `get_messages()` returns it in each message dict when present. This metadata is used by downstream consumers to distinguish tool calls that changed the world from those that merely returned data.

### 3.1.4 Compression fallback streak

The `sessions.compression_fallback_streak INTEGER NOT NULL DEFAULT 0` column (v21) persists a per-session count of consecutive compression fallbacks — situations where the LLM compressor fails and the system falls back to the local deletion compressor. `get_compression_fallback_streak(session_id)` reads the current streak (returning 0 for unknown sessions or null values), and `set_compression_fallback_streak(session_id, streak)` writes it back, normalising to a non-negative integer. The streak allows the compressor to make informed decisions about when to stop attempting LLM-based compression and commit to the fallback path.

### 3.2 Write resilience

Because the DB is shared across processes (gateway workers, CLI sessions, cron runners, desktop readers), writes use:

- **WAL mode** with `DELETE` fallback on NFS/SMB/FUSE filesystems that cannot support WAL (`journal_mode=WAL` raises `OperationalError("locking protocol")`).
- **Application-level retry with jitter** — `BEGIN IMMEDIATE`, then on `database is locked` sleep a random `20–150 ms` and retry up to 15 times. This breaks the convoy pattern that SQLite's deterministic internal backoff creates.
- **Periodic TRUNCATE checkpoints** every 50 writes to keep the WAL file from growing without bound.
- **macOS `synchronous=FULL` enforcement** — after any successful WAL activation, `_enforce_macos_synchronous_full()` sets `PRAGMA synchronous=FULL` on macOS. macOS's `fsync()` does *not* guarantee data-on-platter or write-ordering (per Apple's `fsync(2)` man page), so the WAL default of `synchronous=NORMAL` can leave the main DB with half-written btree pages during a checkpoint race with process termination (e.g. launchd shutdown), resulting in `btreeInitPage error 11` corruption. `synchronous=FULL` issues a real `fsync()` on every transaction commit. The companion `_apply_macos_checkpoint_barrier()` enables `checkpoint_fullfsync=1`, which forces an `F_FULLFSYNC` barrier at checkpoint boundaries (where WAL frames land in the main DB). Both functions are no-ops on non-Darwin platforms and are best-effort (never raise).

Malformed-schema recovery (duplicate `sqlite_master` rows, typically from FTS5 corruption) is handled automatically: `repair_state_db_schema` attempts a `sqlite_master` de-duplication pass, then falls back to dropping and rebuilding the FTS layer while preserving canonical `sessions`/`messages` rows.

### 3.3 Compression lineage

When a conversation exceeds the model’s context threshold, the `ContextCompressor` splits the session: it ends the old session with `end_reason='compression'`, creates a new child session with `parent_session_id` pointing to the old one, and carries the compressed summary forward. The state store treats compression children differently from branch children (`/branch`) and delegate subagent runs — compression children stay hidden in session pickers so the user sees a single logical conversation, while branch children remain visible.

A `compression_locks` table prevents two compressors from racing on the same session ID (e.g. parent agent + background-review fork both deciding to compress simultaneously). Locks expire after 5 minutes; a crashed compressor never blocks the session forever.

`get_compression_lineage()` walks the parent→child chain to reconstruct the full compression lineage for a session. It uses `ancestors` and `seen` sets to detect and break cycles in malformed compression parent/child chains, preventing infinite loops. If a parent or child has already been visited (its ID is in the set), the walk terminates rather than looping forever.

### 3.4 Declarative schema evolution

Column additions are handled by `_reconcile_columns()`, which diffs live tables against the canonical `SCHEMA_SQL` on every startup and `ALTER TABLE ADD COLUMN`s anything missing. Data migrations that transform existing rows still live in a version-gated chain (`SCHEMA_VERSION = 21`).

### 3.5 Workspace grouping (`workspace_key()`)

The top-level `workspace_key(row)` function returns a session's workspace grouping key: its `git_repo_root` when known, else its `cwd`. Branch is deliberately excluded so checking out a new branch doesn't fragment a workspace's session history. Returns `None` for cwd-less (unbound) sessions. Both fields are already recorded on the `sessions` row — this function just picks the coarser identity for grouping and filtering.

### 3.6 Session recovery (`ws_orphan_reap`)

`find_latest_gateway_session_for_peer()` (used by `recover_session_for_session_key()`) treats `end_reason='ws_orphan_reap'` as recoverable alongside `agent_close`. The `ws_orphan_reap` end_reason is set when a dashboard viewer disconnects (before the fix in #60609), which previously left gateway sessions unrecoverable after a viewer disconnect bug. Treating it as recoverable ensures the session can be resumed rather than silently orphaned.

### 3.7 Session import

The `import_sessions(sessions)` method imports sessions exported by `export_session()` or `export_all()`. It enforces bounded batch limits to prevent abuse or accidental overload:

| Limit | Value |
|---|---|
| `_IMPORT_MAX_SESSIONS` | 500 |
| `_IMPORT_MAX_MESSAGES_PER_SESSION` | 10,000 |
| `_IMPORT_MAX_TOTAL_MESSAGES` | 50,000 |
| `_IMPORT_MAX_SESSION_BYTES` | 5 MB |
| `_IMPORT_MAX_TOTAL_BYTES` | 25 MB |

The method validates each session payload (type-checking, JSON serialisability, size limits, duplicate ID detection), normalises text fields, and parses `model_config` as JSON. Existing session IDs are skipped. Imported child sessions keep their `parent_session_id` only when the parent already exists in the database or is included in the same import payload; otherwise the child is detached (parent set to `NULL`) so partial imports don't fail foreign-key validation.

Live runtime state is intentionally reset on import: gateway routing, handoff state, and rewind state are cleared. This restores conversation history, not ownership of a live channel or process. The method returns a summary dict with `ok`, `imported`, `skipped`, `detached`, and `errors` keys.

---

## 4. Session Search: Zero-LLM Recall (`tools/session_search_tool.py`)

The `session_search` tool is the agent’s primary long-term recall surface. It operates entirely on `state.db` via the FTS5 index — no auxiliary LLM calls, no summarization, no embeddings.

### 4.1 Four calling shapes

The tool infers its mode from the arguments rather than an explicit mode parameter:

1. **DISCOVERY** — `query="…"`  
   Runs FTS5, deduplicates hits by session lineage root, and returns up to `limit` sessions. Each result carries:
   - `snippet` — FTS5-highlighted excerpt (`>>>match<<<`).
   - `bookend_start` — first 3 user/assistant messages (the goal / kickoff).
   - `messages` — ±5 messages around the anchor hit.
   - `bookend_end` — last 3 user/assistant messages (the resolution).
   - `match_message_id`, `messages_before`, `messages_after`.

2. **SCROLL** — `session_id` + `around_message_id`  
   Returns a ±`window` slice centred on the anchor. No FTS5, no bookends. The caller scrolls forward/backward by re-anchoring on the last/first message ID of the returned window.

3. **READ** — `session_id` only (no anchor)  
   Dumps the whole session: first 20 + last 10 messages for large sessions, with a pointer to scroll the middle.

4. **BROWSE** — no arguments  
   Lists recent sessions with titles, previews, timestamps, and message counts.

### 4.2 FTS5 query syntax

The tool exposes FTS5 raw syntax to the model:

- **AND** is the default — `docker deployment` means both words must appear.
- **OR** — `alpha OR beta OR gamma`.
- **Phrases** — `"exact phrase"`.
- **Boolean** — `python NOT java`.
- **Prefix** — `deploy*`.

CJK queries are routed automatically:

- 3+ CJK characters per non-operator token → `messages_fts_trigram` (substring matching with ranking).
- 1–2 CJK characters → `LIKE` substring fallback (trigram needs ≥9 bytes).

Role filtering defaults to `user,assistant` so tool-output noise is excluded from recall unless explicitly requested.

### 4.3 Cross-profile reads

The tool accepts a `profile` argument for read-only access to another profile’s `state.db` (e.g. resolving an `@session:work/2026…` link the user dropped into chat). If the model omits the profile, a `_locate_session_db` scan walks every profile’s DB to find the session ID.

---

## 5. Pluggable Memory Providers (`agent/memory_provider.py`, `agent/memory_manager.py`, `plugins/memory/`)

Beyond the built-in file-backed memory and session DB, Hermes supports external memory backends — Honcho, Mem0 (cloud and self-hosted), Supermemory, Byterover, Hindsight, Holographic, OpenViking, RetainDB, and any user-installed plugin implementing the `MemoryProvider` ABC.

### Mem0 Self-Hosted Mode

Mem0 can run in either cloud or self-hosted mode. The self-hosted mode includes a dashboard backend and configurable recall tuning (rerank, host routing precedence). The setup wizard supports both modes, and the SDK is lazy-installed (like honcho/hindsight) so missing dependencies don't block startup.

### Holographic Memory FTS5 Sanitizer

The Holographic memory provider applies an FTS5 query sanitizer to `search_facts` and natural-language recall queries, preventing malformed FTS5 syntax from causing search failures. This mirrors the same sanitization logic used by `session_search`.

### 5.1 The one-external-provider rule

To prevent tool-schema bloat and conflicting backends, `MemoryManager` enforces a hard limit: **only one external provider may be registered at a time**.

```python
# In MemoryManager.add_provider()
if not is_builtin and self._has_external:
    logger.warning(
        "Rejected memory provider '%s' — external provider '%s' is already "
        "registered. Only one external memory provider is allowed at a time.",
        provider.name, existing,
    )
    return
```

The builtin provider (name `"builtin"`) is always accepted; external providers are selected via `memory.provider` in `config.yaml`.

### 5.2 MemoryProvider ABC

```
MemoryProvider (ABC)
  ├── name: str
  ├── is_available() -> bool
  ├── initialize(session_id, **kwargs)
  ├── system_prompt_block() -> str        (static prompt text)
  ├── prefetch(query, session_id="") -> str
  ├── queue_prefetch(query, session_id="")
  ├── sync_turn(user_content, assistant_content, *, session_id, messages)
  ├── get_tool_schemas() -> List[Dict]
  ├── handle_tool_call(tool_name, args, **kwargs) -> str
  ├── shutdown()
  └── optional hooks:
      ├── on_turn_start(turn_number, message, **kwargs)
      ├── on_session_end(messages)
      ├── on_session_switch(new_session_id, parent_session_id, reset, rewound)
      ├── on_pre_compress(messages) -> str
      ├── on_memory_write(action, target, content, metadata)
      └── on_delegation(task, result, child_session_id)
```

- `initialize()` receives `hermes_home`, `platform`, `agent_context`, `agent_identity`, `user_id`, `parent_session_id`, and other runtime context so providers can scope their storage correctly.
- `sync_turn()` is called after every completed turn to persist the exchange. The manager strips skill-scaffolding text before forwarding so providers don’t ingest prompt-injection artifacts.
- `prefetch()` runs before the next API call, returning context text to inject into the user-visible turn. Results from all providers are merged and wrapped in `<memory-context>` fences with a system note instructing the model to treat the content as authoritative reference data.
- `queue_prefetch()` triggers a background recall immediately after the turn, so the next `prefetch()` can return a cache hit.

### 5.3 MemoryManager orchestration

`MemoryManager` is the single integration point in `run_agent.py`:

```python
self._memory_manager = MemoryManager()
self._memory_manager.add_provider(plugin_provider)   # optional external

# System prompt
prompt_parts.append(self._memory_manager.build_system_prompt())

# Pre-turn
context = self._memory_manager.prefetch_all(user_message)

# Post-turn
self._memory_manager.sync_all(user_msg, assistant_response)
self._memory_manager.queue_prefetch_all(user_msg)
```

**Failure isolation** — a provider throwing an exception is caught, logged, and skipped. One broken backend never blocks another.

**Background sync** — `sync_all` and `queue_prefetch_all` dispatch work to a single-worker `ThreadPoolExecutor` (created lazily) so a wedged daemon cannot stall the turn-completion path. The executor is serialised (max_workers=1) so turn N is guaranteed to land before turn N+1 in the provider’s store.

**Draining on exit** — `shutdown_all()` first submits a sentinel to the executor and waits up to `_SYNC_DRAIN_TIMEOUT_S` (5 seconds) so the final turn of a session has a chance to persist before providers are torn down.

### 5.4 Tool injection

Providers may expose their own tools (e.g. `honcho_add_memory`, `mem0_search`). `MemoryManager.get_all_tool_schemas()` collects schemas from all registered providers and injects them into the agent’s tool surface via `inject_memory_provider_tools()`. Reserved core tool names (`clarify`, `delegate_task`, etc.) are skipped — builtins always win.

### 5.5 Provider discovery (`plugins/memory/__init__.py`)

Discovery scans two directories:

1. **Bundled** — `plugins/memory/<name>/` (in-repo providers).
2. **User-installed** — `$HERMES_HOME/plugins/<name>/` (standalone repos, pip entry points).

Bundled providers take precedence on name collision. The active provider is loaded by `load_memory_provider(configured_name)` and registered with `MemoryManager.add_provider()`.

> **Policy (May 2026):** the set of in-tree memory providers is closed. New backends must ship as standalone plugin repos that implement the same `MemoryProvider` ABC and register through the same discovery path.

### 5.6 Context fencing

Provider output that reaches the UI must not leak raw `<memory-context>` tags or injected system notes. `MemoryManager` handles this in two ways:

- **One-shot scrubbing** — `sanitize_context()` strips fence tags, pre-wrapped context blocks, and system-note lines from provider output using compiled regexes.
- **Streaming scrubbing** — `StreamingContextScrubber` maintains a small state machine across streaming deltas so that a `<memory-context>` opened in one chunk and closed in a later chunk is fully suppressed. Partial-tag tails are held back in an internal buffer and either discarded or emitted on `flush()`.

### 5.7 Skill-scaffolding stripping

When the user invokes a `/skill` or `/bundle`, Hermes expands the turn into a model-facing message embedding the entire skill body. Feeding that verbatim to memory providers would pollute their stores with prompt scaffolding. `MemoryManager._strip_skill_scaffolding()` extracts just the user’s actual instruction before passing it to `prefetch_all` and `sync_all`. This stripping is applied to **all** providers, not just a single provider — it is a universal preprocessing step.

---

## 6. The Auxiliary Client (`agent/auxiliary_client.py`)

While not strictly part of the memory subsystem, the auxiliary client is the bridge that memory providers (and session compression, title generation, web extraction, vision analysis) use when they need a side LLM call. It provides a single resolution chain so every consumer picks up the best available backend without duplicating fallback logic.

### 6.1 Resolution order (auto mode)

For **text** tasks:
1. User’s main provider + main model
2. OpenRouter (`OPENROUTER_API_KEY`)
3. Nous Portal (`~/.hermes/auth.json` active provider)
4. Custom endpoint (`base_url` + `OPENAI_API_KEY`)
5. Native Anthropic
6. Direct API-key providers (z.ai/GLM, Kimi/Moonshot, MiniMax, …)

For **vision/multimodal** tasks the order is similar but skips providers without vision support (e.g. Kimi Coding Plan routes).

Per-task overrides live in `config.yaml` under the `auxiliary:` section (e.g. `auxiliary.compression.provider`, `auxiliary.vision.model`).

### 6.2 Credit-exhaustion fallback

When a resolved provider returns HTTP 402 or a credit-related error, `call_llm()` automatically retries with the next provider in the chain. This handles the common case where a user depletes OpenRouter balance but still has Codex OAuth or another provider available.

### 6.3 Temperature and compaction overrides

Certain models have strict server-side contracts. The auxiliary client returns `OMIT_TEMPERATURE` for Kimi/Moonshot models (whose gateways manage temperature internally) and clamps Arcee Trinity Large Thinking to `0.5`. It also raises the context-compression threshold to `0.85` for Codex GPT-5.5 because the backend caps the effective window at 272K, making the default 50% trigger wastefully early.

---

## 7. What Gets Saved, and Where

| Data | Destination | Trigger | Notes |
|---|---|---|---|
| Curated memory entries (`add`/`replace`/`remove`/`batch`) | `MEMORY.md`, `USER.md` | Tool call | Atomic temp-file + rename; batch is all-or-nothing |
| Full message history | `state.db` → `messages` | Every turn | Includes tool_calls JSON, reasoning fields |
| Session metadata (model, tokens, cost, title) | `state.db` → `sessions` | Session start / end / compression | `update_token_counts` increments per API call |
| Per-model usage attribution | `state.db` → `session_model_usage` | Every API call (incremental) | Per-call deltas keyed by `(session_id, model, provider)`; fixes mid-session `/model` switch attribution |
| FTS5 index | `messages_fts`, `messages_fts_trigram` | Triggered on INSERT/UPDATE/DELETE | Rebuilt on schema migration |
| Turn to external provider | Provider backend | End of turn (`sync_all`) | Background ThreadPool; skill scaffolding stripped |
| Prefetch context for next turn | Provider backend | End of turn (`queue_prefetch`) | Cached, consumed by `prefetch()` on next turn |
| End-of-session extraction | Provider backend | Session boundary (`on_session_end`) | Full message list passed; not per-turn |
| Subagent delegation observation | Parent provider | Delegate tool result | Task + result forwarded via `on_delegation` |
| Built-in memory writes | External provider mirror | `memory_tool` mutation | `on_memory_write(action, target, content, metadata)` |

### 7.1 What external providers receive

- **Turn sync**: `user_text` (stripped of skill scaffolding), `assistant_text`, `session_id`, and optionally the full `messages` list (OpenAI-style conversation history as of the completed turn, including tool calls and tool results).
- **Prefetch**: the cleaned user query; providers return arbitrary text that the manager wraps in `<memory-context>` fences.
- **Session end**: the full `messages` list for end-of-session summarization or fact extraction.
- **Session switch**: fired on `/resume`, `/branch`, `/reset`, `/new`, and context compression. Providers update cached per-session state (document IDs, turn counters) so subsequent writes land in the correct session record.
- **Pre-compress**: providers may extract insights from messages about to be summarized; extracted text is included in the compression summary prompt so the compressor preserves provider-relevant facts.

---

## 8. Security Boundaries

The memory system sits at the intersection of prompt injection and data exfiltration risk. Several hard boundaries protect it:

1. **Threat-pattern scanning** (`tools/threat_patterns.py`) — `"strict"` scope on memory writes catches promptware, SSH backdoors, credential exfil patterns, and exfiltration URLs. Matched entries are blocked from the tool and replaced with `[BLOCKED: …]` placeholders in the system-prompt snapshot.

2. **Prompt-cache stability** — because the system-prompt snapshot is frozen at session start, a poisoned memory entry cannot retroactively alter past turns. It only affects new sessions, and only if the user does not notice and remove it via `memory(action=read)` + `remove`.

3. **Tool-schema isolation** — `MemoryManager` rejects provider tools that shadow reserved core names (`clarify`, `delegate_task`, etc.), preventing a malicious or buggy provider from intercepting critical agent operations.

4. **Context fencing** — `sanitize_context()` and `StreamingContextScrubber` prevent provider output from leaking raw `<memory-context>` tags or system notes into the user-visible stream, which could be used to trick the model or downstream UI into treating injected text as privileged memory.

5. **Drift detection** — `MemoryStore._detect_external_drift()` catches non-tool writes (patch tool, manual edit, concurrent session) and refuses to overwrite them, preventing silent data loss.

---

## 10. Lifecycle Diagram

The diagram below shows the system prompt as a **static header** above the loop (it is assembled once at startup and never modified mid-session), and the **per-turn flow** as a cycle that includes the internal tool-call sub-loop.

```
┌──────────────────────────────────────────────────────────────────────┐
│  SYSTEM PROMPT (assembled once at startup — frozen for the session) │
│  · identity, date/time, tool schemas                               │
│  · MEMORY.md / USER.md snapshot                                    │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│                         PER-TURN LOOP                                │
│                                                                      │
│   ┌─────────────┐                                                    │
│   │  prefetch   │ ← warmed by queue_prefetch from PREVIOUS turn      │
│   │  external   │                                                    │
│   │  memory     │                                                    │
│   └──────┬──────┘                                                    │
│          ▼                                                           │
│   inject <memory-context> into user message                        │
│          │                                                           │
│          ▼                                                           │
│   ┌─────────────────────────────┐                                    │
│   │       USER MESSAGE          │                                    │
│   └─────────────┬───────────────┘                                    │
│                 │                                                      │
│                 ▼                                                      │
│   ┌──────────────────────────────────────────────────────────────┐  │
│   │              TOOL-CALL SUB-LOOP (within one turn)            │  │
│   │                                                              │  │
│   │   ┌─────────┐                                               │  │
│   │   │  LLM    │───► generates assistant message               │  │
│   │   │generate │     (may contain tool_calls[])                │  │
│   │   └────┬────┘                                               │  │
│   │        │                                                      │  │
│   │        ▼                                                      │  │
│   │   tool_calls present?                                         │  │
│   │        │                                                      │  │
│   │    YES ▼                                                      │  │
│   │   execute tool(s) ──► append results ──► back to LLM        │  │
│   │        │                                    generate          │  │
│   │        └────────────────────────────┐                       │  │
│   │                                       │  (repeat)           │  │
│   │    NO  ▼                              │                       │  │
│   │   final assistant text ───────────────┘                       │  │
│   │                                                               │  │
│   └──────────────────────┬────────────────────────────────────────┘  │
│                          │                                            │
│                          ▼                                            │
│   ┌──────────────────────────────────────────────────────────────┐   │
│   │  FINALISE TURN                                                │   │
│   │  · append user + assistant messages to SessionDB              │   │
│   │  · sync_all(user, assistant) ─────► background executor       │   │
│   │  · queue_prefetch(user) ─────────► background executor       │   │
│   │    (warms cache for NEXT turn)                                │   │
│   └──────────────────────┬───────────────────────────────────────┘   │
│                          │                                            │
│                          └───────────────────────────────────────────►│
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

Other lifecycle events (outside the hot turn path):
· on_turn_start(turn#, msg)               — optional provider hook
· on_pre_compress(messages) ──► compress ──► on_session_switch(...)
· on_session_end(messages) / shutdown_all() — on /reset, /new, exit
```

### Tool-call sub-loop

Inside a single user-visible turn, the LLM may request zero or more tool calls. When tool calls are present:

1. The agent **pauses completion generation**.
2. It **executes every tool call** in the `tool_calls` array (some in parallel if safe, some serialised if stateful).
3. It appends each result back to the conversation history as a `role="tool"` or tool-result message (encoding varies by API provider — OpenAI Chat Completions, Anthropic Messages, or Codex Responses).
4. It sends the **expanded message list** back to the LLM for another completion.
5. This repeats until the assistant returns a message with no `tool_calls`, or the safety iteration cap is reached.

The user sees the final assistant text only after this sub-loop terminates. Tool-call iterations are not billed or logged as separate turns; they are internal assistant-generation passes within the same turn boundary.

### Why this matters for caching

The system prompt is frozen and identical on every API request, so providers with prefix caching (OpenAI, Anthropic, DeepSeek) serve it from cache. The growing portion is the `user` + `assistant` + `tool_result` pairs. Because the system prompt never moves, grows, or changes, the cache key remains stable for the session lifetime.

---

## 10b. Testing Philosophy

The Hermes codebase treats memory-related tests as integration-critical:

- **E2E state DB tests** (`tests/test_hermes_state.py`) exercise real SQLite operations against temp databases, including FTS5 queries, CJK fallback, malformed-schema recovery, WAL fallback, and compression-lock races.
- **Memory provider contract tests** (`tests/agent/test_memory_provider.py`) verify the `MemoryManager` fan-out, tool-name conflict rejection, skill-scaffolding stripping, metadata mode negotiation, and background sync serialization.
- **Session search tests** (`tests/tools/test_session_search.py`) validate the four calling shapes, cross-profile resolution, lineage deduplication, and scroll anchoring.
- **Write-approval tests** (`tests/tools/test_write_approval.py`) gate memory mutations inline and via staging.

No mock is substituted for real SQLite or real `ThreadPoolExecutor` behavior in tests that assert ordering, locking, or schema migration correctness.

---

## 11. The Auxiliary Client: LLM Side Tasks for Memory Work

Many memory operations need a side LLM call: compressing conversation context before storing it, generating session titles, extracting structured facts from a long transcript, or summarising an entire session for an external provider. Rather than letting each subsystem invent its own client construction and fallback logic, Hermes centralises this in `agent/auxiliary_client.py`.

### 11.1 Design goals

- **Single resolution chain** — every consumer (context compression, session search title generation, web extraction, vision analysis, browser vision, memory flush) picks up the best available backend without duplicating fallback logic.
- **Credit-exhaustion resilience** — when a provider returns HTTP 402 or a credit-related error, `call_llm()` walks to the next candidate automatically.
- **Lazy openai import** — the module does not import the OpenAI SDK at top level (it costs ~240 ms cold). Instead it exposes `_OpenAIProxy`, a lazy proxy that loads the real class on the first `OpenAI(...)` invocation.
- **Temperature and compaction overrides** — certain models have strict server-side contracts. The client returns `OMIT_TEMPERATURE` for Kimi/Moonshot models (whose gateways manage temperature internally) and clamps Arcee Trinity Large Thinking to `0.5`. It also raises the context-compression threshold to `0.85` for Codex GPT-5.5 because the backend caps the effective window at 272K, making the default 50% trigger wastefully early.

### 11.2 Resolution order (auto mode)

For **text** tasks:
1. User’s main provider + main model
2. OpenRouter (`OPENROUTER_API_KEY`)
3. Nous Portal (`~/.hermes/auth.json` active provider)
4. Custom endpoint (`base_url` + `OPENAI_API_KEY`)
5. Native Anthropic
6. Direct API-key providers (z.ai/GLM, Kimi/Moonshot, MiniMax, …)

For **vision/multimodal** tasks the order is similar but skips providers without vision support (e.g. Kimi Coding Plan routes, which have no image input capability).

Per-task overrides live in `config.yaml` under the `auxiliary:` section (e.g. `auxiliary.compression.provider`, `auxiliary.vision.model`).

### 11.3 Codex adapter

The Codex backend (`chatgpt.com/backend-api/codex`) speaks the OpenAI **Responses** API, not the standard `chat.completions` surface. The auxiliary client therefore ships `_CodexCompletionsAdapter`, a shim that:
- Translates `chat.completions.create(messages=[...])` into `responses.create(input=[...], instructions=..., model=...)`.
- Uses the shared `_chat_messages_to_responses_input()` converter from `agent/codex_responses_adapter.py` so tool history is encoded identically to the main agent transport.
- Supports a total-timeout timer that closes the client and evicts the cached instance on expiry, preventing a dead Responses stream from hanging the CLI indefinitely.
- Wraps both sync and async surfaces (`CodexAuxiliaryClient` and `AsyncCodexAuxiliaryClient`) so every consumer can call it without knowing the transport shape.

### 11.4 Portal attribution

Calls through Nous Portal carry `extra_body={"tags": [...]}` for product attribution. The tags are computed at call time (not cached at import) so a hot-reloaded version string is reflected without restarting long-running processes.

### 11.5 Why this matters for memory

External memory providers that do summarisation or fact extraction rely on the auxiliary client for their heavy lifting. By centralising the client, Hermes guarantees that a memory provider never accidentally:
- Burns the user’s main-model credits on a side task.
- Deadlocks because the chosen backend is offline.
- Sends mismatched temperature / context-window settings to a provider with strict server-side rules.

---

## 12. Detailed Data Flow: A Complete Turn

Tracing a single non-interrupted turn from user input to persisted memory clarifies how the layers interact.

```
User sends message "remind me about the docker refactor we discussed"
    │
    ▼
┌────────────────────────────────────────────────────────────────┐
│  1.  Agent receives message                                    │
│      run_conversation() begins                                   │
└────────────────┬───────────────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────────────────┐
│  2.  MemoryManager.prefetch_all(query)                         │
│      • Strips skill scaffolding                                  │
│      • Fan-out to every registered provider                      │
│      • External provider returns cached recall (or empty)        │
│      • Manager merges results, wraps in <memory-context>         │
│      • Injected into the prompt just before the user message     │
└────────────────┬───────────────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────────────────┐
│  3.  LLM turn executes                                           │
│      • System prompt (frozen snapshot incl. MEMORY.md)         │
│      • <memory-context> block from prefetch                      │
│      • User message                                              │
│      • Assistant response (+ possible tool calls)                │
└────────────────┬───────────────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────────────────┐
│  4.  Turn finalises successfully (not interrupted)               │
│      _sync_external_memory_for_turn() fires                     │
│      • Flattens multimodal user msg to plain text (sep="\n")   │
│      • Flattens assistant response likewise                      │
│      • Calls MemoryManager.sync_all(user_text, response_text)  │
│      • Background executor serialises per-provider sync_turn() │
│      • Calls MemoryManager.queue_prefetch_all(user_text)       │
│      • Background executor warms next-turn recall               │
└────────────────┬───────────────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────────────────┐
│  5.  SessionDB persists the turn                               │
│      • append_message() inserts user + assistant rows           │
│      • update_token_counts() increments input/output tokens     │
│      • FTS5 triggers auto-update messages_fts + trigram indexes │
└────────────────────────────────────────────────────────────────┘
```

If the turn is **interrupted** (Ctrl-C, timeout, reset), steps 4 and 5 are skipped entirely for external memory — interrupted state is not durable conversational truth. The session DB still retains the partial messages (they are already in the hot transcript), but they are marked with `active=0` on rewind.

---

## 13. Session Management Boundaries

Sessions in Hermes are not monolithic. Three kinds of lineage split the conversation state:

| Split kind | Trigger | Visibility | State preservation |
|---|---|---|---|
| **Compression** | Context window threshold reached | Hidden child; user sees continuous lineage | Old session ended, child carries summary forward |
| **Branch** | `/branch` command | Visible sibling | Forks from a point in time; both branches remain browsable |
| **Delegate subagent** | `delegate_task` tool | Hidden child (orphan-on-delete) | Subagent runs with `skip_memory=True`; parent observes via `on_delegation` |

`SessionDB` stores `parent_session_id` on every row. Child sessions created **after** the parent ended with `end_reason='compression'` are considered compression continuations; those created while the parent is still live are either branches or delegate runs. The `_LISTABLE_CHILD_SQL` predicate uses both the `_branched_from` JSON marker and the legacy `end_reason='branched'` heuristic to decide which children surface in session pickers.

When a session is **deleted** (e.g. `/delete` or prune), delegate subagent runs are cascade-deleted via the `_delegate_from` marker, while branch children are preserved. Orphaned untagged children (older sessions with no title and ≤25 messages that contain tool rows) are also left alone because they might be subagent runs whose parent was already deleted.

Gateway session recovery via `find_latest_gateway_session_for_peer()` treats `end_reason='agent_close'` and `end_reason='ws_orphan_reap'` as recoverable — sessions ended by these reasons can be resumed rather than orphaned. The `ws_orphan_reap` reason (from dashboard viewer disconnect bugs, fixed in #60609) was previously unrecoverable, leaving gateway sessions silently abandoned after a viewer disconnect.

---

## 14. Cron, Gateway, and Non-CLI Memory Behaviour

Different agent contexts treat memory differently:

- **CLI foreground** — full memory stack active: `MemoryStore`, `SessionDB`, external provider, inline write-approval prompts.
- **CLI background / script** — same stack, but write-approval stages instead of prompting inline.
- **Gateway** (Telegram, Discord, Slack, …) — `MemoryStore` and `SessionDB` active; external provider receives `platform`, `user_id`, `chat_id`, etc. for per-user/per-chat scoping. Cron jobs pass `skip_memory=True` by default because system-prompt scaffolding would corrupt user representations.
- **TUI / Desktop** — `SessionDB` is read-only on the sidebar; memory mutations happen through the CLI backend.
- **Subagent** — created with `skip_memory=True`; has no `MemoryStore` and no external provider session. The parent’s provider gets the task + result via `on_delegation()` so the parent still learns from delegated work.

---

## 15. Performance Characteristics

| Operation | Latency | Bottleneck |
|---|---|---|
| `MemoryStore.load_from_disk()` | ~1–5 ms | File read + split by `§` |
| `MemoryStore.add()` | ~5–20 ms | File lock + temp write + atomic rename |
| `SessionDB.append_message()` | ~1–5 ms | SQLite WAL write + trigger fire |
| `SessionDB.search_messages()` | ~10–50 ms | FTS5 MATCH + JOIN to sessions |
| `SessionDB.get_anchored_view()` | ~5–20 ms | Two indexed range scans |
| `MemoryManager.prefetch_all()` | ~0–5 ms | Cached result from last `queue_prefetch` |
| `MemoryManager.sync_all()` (hot path) | ~0 ms | Returns immediately; work queued to background thread |
| External provider `sync_turn()` | 50 ms – 5 s | Network RTT + provider indexing latency |
| External provider `prefetch()` | 50 ms – 2 s | Same; result cached for next turn |

The agent is designed so that the external provider, which is by far the slowest component, never blocks the user-facing turn path. Even a provider that takes 298 seconds to fail (observed with a misconfigured Hindsight daemon) only fills the background executor queue; the user sees their response instantly and the failure is logged.

---

## 16. Failure Modes and Recovery

| Symptom | Likely cause | Automatic recovery |
|---|---|---|
| Session DB not available | NFS/SMB filesystem blocking WAL | Falls back to `journal_mode=DELETE`; logs one warning per process |
| btree corruption on macOS (`btreeInitPage error 11`) | macOS `fsync()` doesn't guarantee durability; WAL checkpoint race with process termination (launchd shutdown) | `_enforce_macos_synchronous_full()` sets `synchronous=FULL`; `_apply_macos_checkpoint_barrier()` enables `checkpoint_fullfsync=1` (F_FULLFSYNC at checkpoint boundaries) |
| FTS5 module missing | Python built without FTS5 | Disables full-text search; message persistence still works |
| Malformed schema (duplicate messages_fts) | Crash during FTS5 creation | Auto-repair: de-duplicate `sqlite_master`, or drop FTS and rebuild from messages |
| Memory file drift | Patch tool / manual edit / concurrent session | Detected at mutation time; backup created; mutation refused with remediation instructions |
| External provider timeout | Network / backend overload | Background executor absorbs it; turn completes for user |
| Memory provider tool name collision | Plugin registers `clarify` or `delegate_task` | Rejected at registration; core tools always win |
| Poisoned memory entry | Threat pattern in MEMORY.md | Entry blocked from system prompt snapshot; user sees `[BLOCKED: …]` placeholder and can remove it |

---

## 17. Code Map

| File | Responsibility |
|---|---|
| `tools/memory_tool.py` | `MemoryStore`, `memory_tool()` entry point, write-approval gate, threat scanning, drift detection |
| `hermes_state.py` | `SessionDB`, `workspace_key()`, WAL fallback, macOS `synchronous=FULL` enforcement, schema reconciliation, FTS5 setup, message CRUD (incl. `effect_disposition`), per-model usage attribution, compression locks, compression lineage cycle detection, session import |
| `tools/session_search_tool.py` | `session_search()` tool with four shapes (discover, scroll, read, browse), cross-profile resolution |
| `agent/memory_manager.py` | `MemoryManager`, provider orchestration, background executor, context fencing, tool injection, skill-scaffolding strip |
| `agent/memory_provider.py` | `MemoryProvider` ABC defining the lifecycle contract for all external backends |
| `plugins/memory/__init__.py` | Provider discovery (bundled + user-installed), `load_memory_provider()`, synthetic package registration |
| `agent/auxiliary_client.py` | `call_llm()`, provider resolution chains, Codex adapter, Portal attribution, temperature/compaction overrides |
| `agent/agent_init.py` | Wiring `MemoryStore`, `MemoryManager`, external provider init, profile-scoped kwargs assembly |
| `run_agent.py` | `_sync_external_memory_for_turn()`, `shutdown_memory_provider()`, `commit_memory_session()`, interrupt gating |
| `tools/write_approval.py` | Inline / staged approval gate for memory and skill writes |
| `tools/threat_patterns.py` | Injection/exfil pattern scanner used by memory, skills, and context-file loaders |

---

## 18. Summary

Hermes’ memory system is deliberately layered:

- **Core memory** (`MemoryStore`) gives the agent fast, bounded, curated notes that ride in the system prompt — frozen for the entire session to protect the API prefix cache.
- **Session storage** (`SessionDB`) gives durable, full-text-searchable conversation history with zero LLM overhead, powering the `session_search` tool, the dashboard sidebar, and all analytics.
- **Pluggable providers** (`MemoryManager` + `MemoryProvider` ABC) give users the option to sync conversation state to external backends (Honcho, Mem0, Supermemory, …), but only one at a time to keep the tool surface small and predictable.
- **Auxiliary client** (`agent/auxiliary_client.py`) gives memory-related side tasks a reliable, fallback-rich LLM client without duplicating provider logic in every subsystem.

All three layers are coordinated through a single manager, background workers isolate slow backends from the hot path, and multiple hard boundaries (threat scanning, context fencing, drift detection, WAL-fallback, schema self-repair, compression-locking) keep the system resilient against corruption, attack, environmental variance, and provider outage.
