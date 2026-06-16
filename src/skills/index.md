# Chapter: The Hermes Skill System

Hermes extends its behavior not by growing a monolithic core, but by loading **skills** — self-contained markdown documents that teach the agent how to perform specific tasks. A skill is a portable unit of instruction: it declares what it does, when it applies, and how to do it. The skill system is the agent's long-term memory for procedures, conventions, and hard-won experience. It is one of the most heavily trafficked subsystems in the codebase, touching prompt assembly, tool dispatch, background maintenance, and provenance tracking.

This chapter walks through the skill system's design from the ground up: the file format, how skills are discovered and indexed, how they are dispatched at runtime (both via slash commands and system-prompt preload), how bundles compose multiple skills, how the Curator maintains the collection automatically, and how provenance separates agent-created skills from built-ins and hub-installed ones.

---

## 1. Skill Format: What a Skill Is

At minimum, a skill is a directory containing a single file named `SKILL.md`. The directory lives under `~/.hermes/skills/` (or an external directory configured in `config.yaml`). The file follows a conventional markdown-plus-frontmatter layout compatible with the broader agentskills.io ecosystem.

### Directory Layout

```
~/.hermes/skills/
├── my-skill/
│   ├── SKILL.md              # Required: instructions + YAML frontmatter
│   ├── references/           # Supporting docs (API docs, research notes)
│   ├── templates/            # Starter files meant to be copied
│   ├── scripts/              # Runnable helpers (probes, generators)
│   └── assets/               # Supplementary files (images, data)
└── category/
    └── nested-skill/
        └── SKILL.md
```

The four subdirectories (`references/`, `templates/`, `scripts/`, `assets/`) are optional but blessed. When a skill is loaded into the prompt, the system appends a note listing every file it finds in those directories, with absolute paths, so the agent can reference them without a second round-trip.

### SKILL.md Frontmatter

The top of every `SKILL.md` begins with YAML frontmatter between triple-dash delimiters:

```yaml
---
name: skill-name                 # Required. Max 64 chars.
description: Brief description  # Required. Max 1024 chars.
version: 1.0.0                   # Optional
license: MIT                     # Optional (agentskills.io compatible)
platforms: [macos]               # Optional — restrict to OS
tags: [docker, ci]              # Optional search keywords
metadata:
  hermes:
    tags: [deployment]
    related_skills: [docker-build]
    config:
      - key: registry.url
        description: Container registry endpoint
        default: "registry.hub.docker.com"
---

# Skill Title

Full instructions and content here…
```

The frontmatter is parsed by `agent/skill_utils.py::parse_frontmatter()`, which prefers `yaml.CSafeLoader` for performance and falls back to a naive key-splitting parser if the YAML is malformed. This dual-strategy keeps Hermes resilient against hand-edited skills with minor syntax errors.

### Conditional Activation Fields

Skills can declare when they should be offered to the model. These fields live under `metadata.hermes` and are evaluated by `agent/skill_utils.py::extract_skill_conditions()`:

| Field | Meaning |
|-------|---------|
| `fallback_for_toolsets` | Hide this skill when the named toolset is already present |
| `requires_toolsets` | Only show when the named toolset is present |
| `fallback_for_tools` | Hide when the named tool is present |
| `requires_tools` | Only show when the named tool is present |

These are **offer-time** filters. They decide whether a skill appears in the system-prompt skills index; they do **not** block an explicit `skill_view()` or `--skills` preload. An explicit load is treated as explicit consent.

### Platform & Environment Gating

Two separate gating mechanisms decide whether a skill is shown:

1. **`platforms`** (`agent/skill_utils.py::skill_matches_platform()`): A list like `[macos, linux]`. If absent, the skill loads everywhere. Termux receives special treatment: skills tagged `linux` are accepted even on newer Pythons where `sys.platform == "android"`.

2. **`environments`** (`agent/skill_utils.py::skill_matches_environment()`): A relevance gate, not a hard-compatibility gate. Recognized tags are `kanban`, `docker`, and `s6`. A kanban-only skill is hidden from the skills index unless the dispatcher has set `HERMES_KANBAN_TASK` or the profile has opted into the kanban toolset. Like the toolset gates, explicit loads bypass this filter.

---

## 2. Discovery and Loading

Skills are discovered by walking the filesystem. The entry point is `agent/skill_utils.py::iter_skill_index_files()`, which performs an `os.walk` rooted at each skills directory, pruning directories listed in `EXCLUDED_SKILL_DIRS`:

```python
EXCLUDED_SKILL_DIRS = frozenset({
    ".git", ".github", ".hub", ".archive",
    ".venv", "venv", "node_modules", "site-packages",
    "__pycache__", ".tox", ".nox", ".pytest_cache",
    ".mypy_cache", ".ruff_cache",
})
```

This centralized exclusion list ensures that dependencies, virtualenvs, VCS metadata, and caches never leak into the skill index. The walker follows symbolic links (so a skill can be a symlink to a checked-out repository elsewhere) but still validates the resulting path.

### Local and External Directories

`agent/skill_utils.py::get_all_skills_dirs()` returns the full search path:

1. **Local directory**: `~/.hermes/skills/` (always first, always included).
2. **External directories**: Configured under `skills.external_dirs` in `config.yaml`. Paths are expanded (`~`, `${VAR}`), resolved absolutely, and deduplicated. The local dir is silently skipped if it appears in the external list.

External directories are read-only from the skill system's perspective: they appear in listings and the system prompt, but new skills are always written to the local directory. When a name collision occurs between a local skill and an external one, the local skill wins.

### Disabled Skills

Users can suppress skills globally or per-platform via `config.yaml`:

```yaml
skills:
  disabled:
    - noisy-skill
  platform_disabled:
    telegram:
      - cli-only-skill
```

`agent/skill_utils.py::get_disabled_skill_names()` resolves both lists. A globally disabled skill stays disabled everywhere; platform-specific additions layer on top. The platform is detected from `HERMES_PLATFORM` or `HERMES_SESSION_PLATFORM`.

---

## 3. The Skills Index and System Prompt Integration

Skills are not loaded into the system prompt in full — that would explode token usage. Instead, `agent/prompt_builder.py::build_skills_system_prompt()` constructs a compact **index**: a categorized list of skill names and one-line descriptions, rendered inside `<available_skills>` tags.

```
## Skills (mandatory)
Before replying, scan the skills below. If a skill matches or is even partially relevant
…
<available_skills>
  frontend:
    - react-testing: Jest + React Testing Library workflow
    - nextjs-deploy: Vercel deployment steps
  devops:
    - docker-compose: Multi-container local orchestration
</available_skills>
```

The preamble instructs the model to load matching skills via `skill_view(name)`. This keeps the system prompt small while ensuring the model knows what knowledge is available.

### Two-Layer Cache

Building the index is I/O-bound. Hermes defends against repeated full scans with a two-layer cache:

1. **In-process LRU** (`_SKILLS_PROMPT_CACHE`): Keyed by `(skills_dir, external_dirs, available_tools, available_toolsets, platform, disabled_set, compact_categories)`. Up to 8 entries are retained. The lock-protected OrderedDict is safe for concurrent gateway threads.

2. **Disk snapshot** (`.skills_prompt_snapshot.json`): A JSON file persisted under `HERMES_HOME` keyed by an mtime/size manifest of every `SKILL.md` and `DESCRIPTION.md`. On cold start, if the manifest matches the current filesystem, the snapshot is deserialized and the expensive scan is skipped entirely.

If both layers miss, the code walks the filesystem, builds manifest entries, and writes a fresh snapshot for the next process restart.

### Compact Categories (Posture-Driven Demotion)

When the agent adopts a coding posture (e.g., pair-programming mode), non-coding categories are "demoted" to a names-only line:

```
  social-media [names only]: tweet-thread, linkedin-post
```

The skill names remain visible — the model can still request them by name — but their descriptions are omitted to reduce noise. This is controlled by the `compact_categories` argument to `build_skills_system_prompt()`.

### Category Descriptions

If a category folder contains a `DESCRIPTION.md` with YAML frontmatter, its `description` field is rendered as the category header. This lets skill authors provide context for an entire family of skills.

---

## 4. Skill Dispatch at Runtime

There are three ways a skill enters an active conversation:

1. **Explicit tool call**: The model calls `skill_view(name)` from inside the agent loop.
2. **Slash command**: The user types `/skill-name` (or `/bundle-name`) in the CLI or a messaging gateway.
3. **CLI preload**: The user passes `-s skill-name` at startup, injecting the skill into the system prompt for the whole session.

All three paths converge on `agent/skill_commands.py::_build_skill_message()`, which formats the loaded skill into a user-facing message block.

### Loading a Skill (`agent/skill_commands.py`)

`_load_skill_payload(skill_identifier, task_id)` resolves an identifier (name, relative path, or absolute path) to a loaded skill dictionary. Resolution order:

1. If the identifier is an absolute path, check that it lies under a trusted root (`SKILLS_DIR` or an external dir). Prefer the lexical path before symlink resolution, so a trusted symlink is accepted.
2. Otherwise, normalize to a relative path and call `skill_view()` from `tools/skills_tool.py`.

The returned dictionary carries:
- `name`, `content`, `raw_content`
- `setup_needed`, `setup_note`, `setup_skipped`
- `linked_files` — a map of subdirectory → file list
- `skill_dir` — absolute path to the skill root

### Message Assembly (`_build_skill_message`)

Once loaded, the skill content is processed through a pipeline:

1. **Template substitution**: `agent/skill_preprocessing.py::substitute_template_vars()` replaces `${HERMES_SKILL_DIR}` and `${HERMES_SESSION_ID}`.
2. **Inline shell expansion**: If `skills.inline_shell: true` in config, every ``!`cmd` `` snippet is executed with `bash -c`, output capped at 4000 characters, and substituted back into the text.
3. **Activation note**: A prominent banner tells the model this is a skill invocation, e.g.:
   > `[IMPORTANT: The user has invoked the "docker-compose" skill. The full skill content is loaded below.]`
4. **Skill directory hint**: The absolute `skill_dir` is injected so the agent can resolve relative script paths.
5. **Config injection**: If the frontmatter declares `metadata.hermes.config` variables, their resolved values are printed so the agent knows what is configured.
6. **Supporting files list**: Discovered `references/`, `templates/`, `scripts/`, and `assets/` files are enumerated with absolute paths.
7. **Setup notes**: If prerequisites are missing, a `[Skill setup note: …]` block is appended.

### Slash Commands (`agent/skill_commands.py`)

`scan_skill_commands()` walks the skills directories and builds a map from `/slug` to skill metadata. Slugs are normalized:
- Lowercase
- Spaces and underscores → hyphens
- Strip invalid characters (anything outside `[a-z0-9-]`)
- Collapse multiple hyphens

This normalization is shared with `agent/skill_bundles.py` so skills and bundles coexist in the same command namespace.

The cached `_skill_commands` map is invalidated when the platform scope changes (e.g., a gateway process serving both Telegram and Discord), ensuring each platform sees its own `platform_disabled` view.

### Bundles (`agent/skill_bundles.py`)

A **skill bundle** is a YAML file under `~/.hermes/skill-bundles/*.yaml` that references multiple skills by name:

```yaml
name: backend-dev
skills:
  - github-code-review
  - test-driven-development
instruction: |
  Optional extra guidance to inject above the skill bodies.
```

Bundles are resolved **before** single skills in slash-command dispatch: if `/research` matches both a bundle and a skill, the bundle wins.

`build_bundle_invocation_message()` loads each referenced skill, builds its activation block, and concatenates everything under a header:

```
[IMPORTANT: The user has invoked the "backend-dev" skill bundle,
loading 3 skills together. Treat every skill below as active guidance for this turn.]
```

Missing skills are noted but do not abort the bundle; the agent receives the ones that exist.

---

## 5. Preprocessing Pipeline (`agent/skill_preprocessing.py`)

Before a skill reaches the model, it passes through `preprocess_skill_content()`:

### Template Variables
- `${HERMES_SKILL_DIR}` → absolute path to the skill directory.
- `${HERMES_SESSION_ID}` → current session/task ID.

Unresolved tokens are left in place so authors can spot them.

### Inline Shell
Syntax: ``!`bash_command` ``. Each command is executed in the skill's directory with a configurable timeout (default 10s). Failure modes are captured gracefully:
- Timeout → `[inline-shell timeout after 10s: cmd]`
- Bash missing → `[inline-shell error: bash not found]`
- General error → `[inline-shell error: <msg>]`

Output longer than 4000 characters is truncated with `...[truncated]`. This prevents a runaway command from blowing out the context window.

---

## 6. Provenance: Who Made This Skill?

Not all skills are equal in the eyes of the maintenance system. Hermes distinguishes three provenance classes:

| Provenance | Source | Curator manages? |
|------------|--------|------------------|
| **Bundled built-in** | Seeded from the bundled `skills/` directory at install/update time | Only if `curator.prune_builtins` is enabled |
| **Hub-installed** | Downloaded from the Skills Hub (`tools/skills_hub.py`) | **Never** — external upstream owner |
| **Agent-created** | Written by the agent via `skill_manage(action='create')` | **Yes** — primary curation target |

Provenance is tracked in `tools/skill_usage.py`, not inside the skill markdown. This keeps user-facing content clean and avoids merge conflicts during hub updates.

### How Provenance Is Determined

1. **Bundled**: Names are read from `~/.hermes/skills/.bundled_manifest`, a line-delimited file of `name:hash` entries created during `hermes update`.
2. **Hub-installed**: Names are read from `~/.hermes/skills/.hub/lock.json`, which records the upstream source and local install path of every hub skill.
3. **Agent-created**: A usage record with `created_by: "agent"` (set by `mark_agent_created()` when `skill_manage` creates a skill).

Manually authored skills (a user dropping a `SKILL.md` into the directory) are **not** inferred as agent-created. They receive usage telemetry, but the Curator ignores them unless they are explicitly opted in. This prevents the curator from archiving a user's hand-crafted notes.

### Suppression List for Built-ins

When the curator archives a bundled built-in, its name is appended to `~/.hermes/skills/.curator_suppressed`. The `hermes update` re-seeder consults this list and skips suppressed names, ensuring a prune decision survives across updates. Restoring a skill clears its suppression entry.

---

## 7. The Curator: Automatic Skill Maintenance

The Curator (`agent/curator.py`) is a background orchestrator that reviews agent-created skills and maintains the collection. It runs when the agent is idle and enough time has passed since the last review.

### Lifecycle States

Every curator-managed skill moves through states tracked in `~/.hermes/skills/.usage.json`:

| State | Meaning | Transition trigger |
|-------|---------|-------------------|
| `active` | Default, healthy | — |
| `stale` | Unused for `stale_after_days` (default 30) | Auto-transition |
| `archived` | Unused for `archive_after_days` (default 90); directory moved to `.archive/` | Auto-transition or LLM consolidation |

Pinned skills (`pinned: true` in their usage record) bypass all automatic transitions. Additionally, `PROTECTED_BUILTIN_SKILLS` (currently just `plan`) are never touched regardless of any flag.

### Automatic Transitions

`apply_automatic_transitions()` walks every curation-eligible skill and compares its latest activity timestamp to cutoffs derived from `config.yaml`:

- `anchor <= archive_cutoff` and not already archived → archive
- `anchor <= stale_cutoff` and currently active → mark stale
- `anchor > stale_cutoff` and currently stale → reactivate

The "anchor" is the latest of `last_used_at`, `last_viewed_at`, or `last_patched_at`. If none exist, `created_at` is used so brand-new skills don't immediately archive themselves.

Built-ins that become eligible for the first time (when `prune_builtins` flips on) are **seeded** with a fresh record anchored to "now," so they are archived only after a full inactivity window, not instantly.

### LLM Consolidation Pass

After the pure time-based transitions, the curator spawns a **forked AIAgent** running a dedicated review prompt (`CURATOR_REVIEW_PROMPT`). This is an "umbrella-building" pass, not a passive audit. Its goals:

- Identify prefix clusters of related skills (`hermes-config-*`, `pr-*`, `python-*`, etc.)
- Merge narrow skills into broad **umbrella** skills with labeled subsections
- Move session-specific detail into `references/`, `templates/`, or `scripts/` under the umbrella
- Archive the now-redundant siblings

The prompt enforces hard rules:
1. Do not touch bundled or hub skills.
2. Do not delete — only archive (recoverable).
3. Do not touch pinned or protected skills.
4. Judge overlap on **content**, not on usage counters (which are often zero for new skills).
5. Produce a structured YAML summary of `consolidations:` and `prunings:`.

### Dry-Run Mode

`hermes curator run --dry-run` skips all mutations and injects a `DRY-RUN` banner into the LLM prompt. The curator still produces a `REPORT.md` under `~/.hermes/logs/curator/`, allowing users to preview what would happen.

### Reports and Recovery

Every run writes a timestamped directory under `logs/curator/{YYYYMMDD-HHMMSS}/` containing:
- `run.json` — machine-readable full record
- `REPORT.md` — human-readable markdown summary
- `cron_rewrites.json` — only when scheduled jobs had skill references updated

Recovery commands:
- `hermes curator restore <name>` — move a skill from `.archive/` back to the live directory.
- `hermes curator pin <name>` — protect a skill from future auto-transitions.

### Cron Job Rewrites

When the curator consolidates skill X into umbrella Y, any scheduled cron job that listed X would fail to load it. The curator automatically rewrites those job definitions in-place, substituting Y for X (or dropping X if it was pruned without a target). This keeps scheduled tasks working across consolidation passes.

---

### Security Guards in `skill_view()`

`tools/skills_tool.py::skill_view()` implements several defense-in-depth checks before returning skill content:

1. **Path validation** (`_skill_lookup_path_error()`): Rejects absolute paths, Windows drive letters (`C:\\skills\\foo`), and `..` traversal components. This prevents a malicious model from requesting arbitrary files outside the skills directory.

2. **Collision detection**: If more than one skill matches a bare name across local and external directories, the function refuses to guess and returns an error with all matching paths. Silent shadowing is treated as a bug class, not a convenience.

3. **Trusted-directory check**: After resolving the skill file, it verifies that the resolved path lies under a trusted root (`SKILLS_DIR` or an external dir). Skills loaded from outside are served with a security warning.

4. **Prompt-injection scan**: A fixed list of suspicious substrings (`_INJECTION_PATTERNS`) is tested case-insensitively. If any match, the skill is still served (since many legitimate skills quote examples containing the word "ignore"), but a `WARNING`-level log line is emitted so operators can audit.

5. **Traversal guard for linked files**: When `file_path` is provided, `tools.path_security.validate_within_dir()` confirms the resolved target is still inside the skill directory. `..` components are rejected outright.

---

### Plugin-Provided Skills

Skills are not limited to the filesystem. Plugins can register skills via the plugin manager (`hermes_cli.plugins`). A qualified name like `plugin:skill-name` routes to `_serve_plugin_skill()` instead of the local directory walk.

Plugin skills receive:
- A **bundle-context banner** listing sibling skills from the same plugin, so the agent knows what else is available.
- The same template-variable and inline-shell preprocessing as local skills.
- The same prompt-injection scan and logging.

Plugin skill collisions are handled by the same namespace validation used elsewhere: namespaces must match `^[a-zA-Z0-9_-]+$`. Invalid namespaces or missing plugin skills fall through to the local scan with a clear error message.

---

### Skill Manager Tool (`tools/skill_manager_tool.py`)

The agent interacts with the skill library through the `skill_manage` tool. It supports several actions:

| Action | Purpose |
|--------|---------|
| `create` | Write a new `SKILL.md` into `~/.hermes/skills/<name>/` |
| `patch` | Apply a unified-diff style patch to an existing skill |
| `write_file` | Add a supporting file under `references/`, `templates/`, `scripts/`, or `assets/` |
| `delete` | Archive a skill (move to `.archive/`) |
| `view` | Alias for `skill_view` — read skill content |
| `edit` | Read + rewrite a skill with a new string |

On `create`, the tool calls `tools/skill_usage.py::mark_agent_created()` to set `created_by: "agent"`, enrolling the skill in curator management. It also writes out a basic frontmatter scaffold so the new skill is immediately discoverable.

Security guards:
- Absolute paths are rejected unless they lie under a trusted root.
- Traversal components (`..`) are blocked via `tools/path_security.py`.
- Prompt-injection patterns (e.g., `ignore previous instructions`) are scanned in created/patched content before the skill is written.

---

## 8. Scaffolding Markers and Memory Hygiene

When a slash command or bundle expands into a user message, the resulting text includes scaffolding headers. If a memory provider stores the raw turn, it would capture the entire skill body rather than what the user actually asked. `agent/skill_commands.py::extract_user_instruction_from_skill_message()` reverses this by looking for canonical marker strings:

- Single skill: `The full skill content is loaded below.` → look backward for the user instruction.
- Bundle: `User instruction:` → extract everything before the first loaded skill block.

If no instruction was provided (a bare `/skill-name`), the function returns `None`, signaling memory providers to skip the turn entirely. These markers are asserted in tests to stay byte-identical with the builders, so drift between `_build_skill_message()` and the extractor is caught early.

---

## 9. Skill Usage Telemetry (`tools/skill_usage.py`)

Every significant interaction with a skill is recorded in a sidecar JSON file, `~/.hermes/skills/.usage.json`, keyed by skill name. This file is deliberately **not** embedded in the skill markdown — it keeps operational telemetry out of user-authored content and avoids merge conflicts during hub updates.

### Record Schema

Each entry is a dict with the following fields:

| Field | Purpose |
|-------|---------|
| `created_by` | `"agent"` for agent-created skills; `null` otherwise |
| `use_count` | Times the skill was loaded into an active turn |
| `view_count` | Times the skill was inspected via `skill_view()` |
| `patch_count` | Times the skill was edited via `skill_manage` |
| `last_used_at` | ISO timestamp of the most recent load |
| `last_viewed_at` | ISO timestamp of the most recent view |
| `last_patched_at` | ISO timestamp of the most recent edit |
| `created_at` | When the record was first seeded |
| `state` | `active`, `stale`, or `archived` |
| `pinned` | Boolean — exempts the skill from auto-transitions |
| `archived_at` | Timestamp when the skill was archived |

The sidecar is read and written under a cross-process file lock (`fcntl.flock` on Unix, `msvcrt.locking` on Windows), so concurrent gateway sessions and CLI invocations do not corrupt it.

### Telemetry Hooks

Three bump functions are called from the tool layer:

- `bump_view(name)` — called by `skill_view()` every time a skill is read.
- `bump_use(name)` — called when a skill is actually injected into a conversation (slash command, bundle, or `--skills` preload).
- `bump_patch(name)` — called by `skill_manage(action='patch'|'edit'|'write_file')`.

All three are best-effort: a failure logs at `DEBUG` and returns silently so a broken sidecar never breaks the underlying tool call.

### Seeding Built-ins

When `curator.prune_builtins` is enabled, built-in skills suddenly become curation-eligible. But they have no usage record. `seed_record_if_missing()` writes a fresh record with `created_at` set to "now" the first time the curator sees a built-in. This anchors the inactivity clock to the moment of eligibility, preventing a mass archive of all built-ins the instant the flag flips on.

### Archive and Restore

`archive_skill(name)` moves the skill directory to `~/.hermes/skills/.archive/`. If a collision exists, a timestamp suffix is appended. Restoration is the inverse: `restore_skill(name)` scans `.archive/` for an exact or prefix match, moves the directory back to the live tree, and clears any suppression list entry so the re-seeder may manage it again.

Both operations refuse to act on:
- Protected built-ins (`plan`)
- Hub-installed skills (external upstream owner)
- Bundled built-ins when `prune_builtins` is disabled

---

## 10. Architecture Diagrams

### Skill Data Flow

```
┌─────────────────┐
│  User / Model   │
│  (slash cmd or  │
│   skill_view)   │
└────────┬────────┘
         │
         ▼
┌──────────────────────────┐
│ agent/skill_commands.py  │
│  _load_skill_payload()   │
│  _build_skill_message()  │
└────────┬─────────────────┘
         │
    ┌────┴────┐
    ▼         ▼
┌────────┐  ┌─────────────────────────┐
│prompt  │  │ agent/skill_preprocessing │
│builder │  │  template vars + inline   │
│        │  │         shell             │
└────────┘  └─────────────────────────┘
         │
         ▼
┌──────────────────────────┐
│  Conversation context     │
│  (system or user message) │
└──────────────────────────┘
```

### Skills Index Cache Layers

```
┌──────────────────────────────────────┐
│         build_skills_system_prompt() │
│         (agent/prompt_builder.py)   │
└─────────────────┬────────────────────┘
                  │
     ┌────────────┼────────────┐
     ▼            ▼            ▼
┌─────────┐ ┌──────────┐ ┌────────────┐
│ Layer 1 │ │ Layer 2  │ │  Cold Path │
│ In-proc │ │  Disk    │ │ Filesystem │
│  LRU    │ │ Snapshot │ │    Scan    │
│ (8 max) │ │  JSON    │ │ + mtime    │
└─────────┘ └──────────┘ └────────────┘
```

### Curator Lifecycle

```
┌─────────┐     ┌──────────┐     ┌───────────┐
│ active  │────▶│  stale   │────▶│ archived  │
│         │     │(30 days) │     │(90 days)  │
└─────────┘     └──────────┘     └───────────┘
    ▲                                  │
    │     skill used again             │
    └──────────────────────────────────┘
           (reactivate)
```

---

## 11. Configuration Variables

Skills can declare configuration variables that the user sets in `config.yaml` under `skills.config.<key>`:

```yaml
metadata:
  hermes:
    config:
      - key: deploy.target
        description: Default deployment host
        default: "staging.internal"
        prompt: Which host to deploy to?
```

`agent/skill_utils.py::resolve_skill_config_values()` reads these declarations and returns a map of logical keys to current values, expanding `~` and environment variables. When a skill is loaded, `_inject_skill_config()` appends a `[Skill config: ...]` block to the message so the agent sees the live values without opening `config.yaml` itself.

---

## 12. External Skill Directories

Advanced users or teams can share skill libraries via `skills.external_dirs`:

```yaml
skills:
  external_dirs:
    - ~/company-hermes-skills
    - /opt/shared/hermes/skills
```

Paths are expanded and resolved against `HERMES_HOME` when relative. Duplicates and non-existent paths are ignored. External skills:
- Appear in `skills_list()` and the system prompt index.
- Can be loaded via `skill_view()` and slash commands.
- Are **not** writable by `skill_manage` — creations and edits go to the local directory.

This separation lets organizations maintain a canonical shared skill set while allowing individual users to supplement it with personal skills.

---

## 13. Summary

The Hermes skill system is a deliberate architectural bet: **capability lives at the edges**. Rather than teaching the core agent every new workflow, Hermes teaches the agent to read instructions from the filesystem at runtime. This yields several powerful properties:

- **Progressive disclosure**: The system prompt carries only an index; full skill bodies are loaded on demand.
- **Platform sensitivity**: Skills gate themselves by OS and runtime environment without central coordination.
- **Automatic maintenance**: The Curator ages out stale skills and consolidates overlapping ones, preventing the library from degrading into an unmaintainable pile of session artifacts.
- **Provenance safety**: Built-ins, hub skills, and agent-created skills are kept in separate lanes so automatic maintenance never destroys upstream or load-bearing content.
- **Composability**: Bundles let users alias multi-skill workflows to a single slash command, and template variables + inline shell let skills adapt dynamically to the current session.

Understanding the skill system is essential for anyone extending Hermes, debugging why a skill does (or doesn't) appear in the index, or designing new background maintenance workflows. The core invariant is simple: a skill is just a directory with a `SKILL.md`, and everything else — indexing, dispatch, caching, curation, provenance — is infrastructure around that single convention.
