# Chapter: The Hermes Skill System

Hermes extends its behavior not by growing a monolithic core, but by loading **skills** — self-contained markdown documents that teach the agent how to perform specific tasks. A skill is a portable unit of instruction: it declares what it does, when it applies, and how to do it. The skill system is the agent's long-term memory for procedures, conventions, and hard-won experience. It is one of the most heavily trafficked subsystems in the codebase, touching prompt assembly, tool dispatch, background maintenance, and provenance tracking.

This chapter walks through the skill system's design from the ground up: the file format, how skills are discovered and indexed (including the tiered search roots that decide which copy of a name wins), how they are dispatched at runtime (slash commands, system-prompt preload, and config auto-load), how bundles compose multiple skills, how the Curator maintains the collection automatically, and how provenance separates agent-created skills from built-ins and hub-installed ones.

---

## 1. Skill Format: What a Skill Is

At minimum, a skill is a directory containing a single file named `SKILL.md`. The directory lives under `~/.hermes/skills/` (or a trusted project directory, a configured `skills.create_dir`, or an external directory in `config.yaml`). The file follows a conventional markdown-plus-frontmatter layout compatible with the broader agentskills.io ecosystem.

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

The four subdirectories (`references/`, `templates/`, `scripts/`, `assets/`) are optional but blessed. When a skill is loaded into the prompt, the system appends a note listing every file it finds in those directories, with skill-relative paths, so the agent can load them via `skill_view(name, file_path=...)` or run scripts directly by absolute path without a second round-trip. The four directories are scanned with different glob shapes (`_LINKED_FILE_SPECS` in `tools/skills_tool.py`): `references/` collects top-level markdown, `templates/` collects known file types recursively, `assets/` collects all files recursively, and `scripts/` collects executable source extensions at the top level.

### SKILL.md Frontmatter

The top of every `SKILL.md` begins with YAML frontmatter between triple-dash delimiters:

```yaml
---
name: skill-name                 # Required. Max 64 chars.
description: Brief description  # Required. Max 1024 chars (see the 60-char
                                # system-prompt budget below).
version: 1.0.0                   # Optional
license: MIT                     # Optional (agentskills.io compatible)
platforms: [macos]               # Optional — restrict to host OS
environments: [kanban]           # Optional — offer-time runtime relevance gate
requires_apps: [spotify]         # Optional — hide until the app is registered
session_platforms: [telegram]    # Optional — gateway channels the skill is FOR
deps: [ffmpeg]                   # Optional — pm packages ensured at load time
required_environment_variables:  # Optional — secrets captured at first view
  - name: SPOTIFY_CLIENT_ID
    description: Spotify API client ID
required_credential_files:       # Optional — registered for remote sandboxes
  - path: ~/.spotify/credentials
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

The frontmatter is parsed by `agent/skill_utils.py::parse_frontmatter()`, which strips a leading UTF-8 BOM (Windows editors emit one, and it would defeat the `---` fence check) and then calls the shared `hermes_yaml.safe_load`, falling back to a naive key-splitting parser if the YAML is malformed. This dual-strategy keeps Hermes resilient against hand-edited skills with minor syntax errors; BOM tolerance is applied consistently across frontmatter readers, config readers, and hub-lock readers (`utf-8-sig` throughout).

### Conditional Activation Fields

Skills can declare when they should be offered to the model. These fields live under `metadata.hermes` and are evaluated by `agent/skill_utils.py::extract_skill_conditions()`:

| Field | Meaning |
|-------|---------|
| `fallback_for_toolsets` | Hide this skill when the named toolset is already present |
| `requires_toolsets` | Only show when the named toolset is present |
| `fallback_for_tools` | Hide when the named tool is present |
| `requires_tools` | Only show when the named tool is present |
| `session_platforms` | Only show on the named gateway channel (telegram, discord, …) |

These are **offer-time** filters. They decide whether a skill appears in the system-prompt skills index; they do **not** block an explicit `skill_view()`. An explicit load is treated as explicit consent. With no tool/platform information available, every skill passes the filter (fail open).

### Platform, Environment, and App Gating

Three separate gating mechanisms decide whether a skill is shown:

1. **`platforms`** (`agent/skill_utils.py::skill_matches_platform()`): A list like `[macos, linux]`, normalized through `PLATFORM_MAP` (`macos` → `darwin`, `windows` → `win32`) and prefix-matched against `sys.platform`. If absent, the skill loads everywhere. Unlike the other gates this one is a **hard** compatibility check: `skill_view()` refuses to serve a platform-mismatched skill with an `UNSUPPORTED` readiness status. It is offer-time only in the system-prompt index and slash-command scan, where `environments` and `requires_apps` join it.

2. **`environments`** (`agent/skill_utils.py::skill_matches_environment()`): A relevance gate, not a hard-compatibility gate. Recognized tags are `kanban`, `docker`, and `s6`; unknown tags fail open. Kanban detection requires that this execution *owns* the task (`HERMES_KANBAN_TASK`/`HERMES_KANBAN_BOARD` set and the process is a dispatcher-owned worker — delegate children and in-process cron see the worker's vars) or the profile has opted into the kanban toolset. Verdicts are cached per process, *except* kanban, which is deliberately context-dependent and re-detected on every ask. Like the toolset gates, explicit loads bypass this filter.

3. **`requires_apps`** (`agent/skill_utils.py::skill_matches_apps()`): Every named app must have a registered declaration (`hermes_platform.declaration`, registered by whoever owns the server, e.g. the plugin loader) whose `availability()` says it is offerable — the same check an MCP tool's `check_fn` uses. An unknown name hides the skill (fail closed). Offer-time only, like `environments`.

---

## 2. Discovery and Loading

Skills are discovered by walking the filesystem. The entry point is `agent/skill_utils.py::iter_skill_index_files()`, which performs an `os.walk` rooted at each skills directory, pruning directories listed in `EXCLUDED_SKILL_DIRS` and yielding sorted paths:

```python
EXCLUDED_SKILL_DIRS = frozenset((
    ".git", ".github", ".hub", ".archive", ".curator_backups", ".locks",
    ".venv", "venv", "node_modules", "site-packages", "__pycache__",
    ".tox", ".nox", ".pytest_cache", ".mypy_cache", ".ruff_cache",
))
```

This centralized exclusion list ensures that dependencies, virtualenvs, VCS metadata, caches, curator backups, and lock files never leak into the skill index. The walker follows symbolic links (so a skill can be a symlink to a checked-out repository elsewhere) but trust is still validated at load time.

### Support-Directory Exclusion

When a `SKILL.md` exists in a skill directory, the four blessed subdirectories (`references/`, `templates/`, `scripts/`, `assets/`) are pruned from discovery — they are listed in the skill's loaded content but not scanned as separate skills. `SKILL_SUPPORT_DIRS` and `is_skill_support_path()` / `is_excluded_skill_path()` in `skill_utils.py` enforce this, preventing support files from being misidentified as standalone skills. The check is depth-aware: `skills/scripts/foo/` with no `SKILL.md` above `scripts` remains a discoverable skill root.

### Tiered Search Roots and Cross-Directory Precedence

Name resolution is no longer "local wins by directory order." `agent/skill_utils.py::get_skill_search_roots()` returns `(tier, dir)` pairs in one shared precedence order that the skills list, prompt index, slash commands, `skill_view()`, preload, and cron all consume:

1. **Trusted project** (`TIER_PROJECT`): `<project-root>/.hermes/skills/` and `<project-root>/.agents/skills/`, where the project root is the nearest ancestor containing `.git` (found by walking up from the session's effective cwd; `$HOME` itself never counts as a project, so a dotfiles checkout doesn't make every session project-scoped).
2. **Local profile** (`TIER_LOCAL`): `~/.hermes/skills/` (the active profile's dir).
3. **`skills.create_dir`** (`TIER_CREATE_DIR`): an optional directory where new skills land (see Chapter tooling); counts as a search root only once it exists.
4. **`skills.external_dirs`** (`TIER_EXTERNAL`): team-shared read-only libraries.

The lowest tier wins. Inside one tier, two *different* skills sharing a name are ambiguous and refused — never guessed (`AMBIGUOUS_SKILL_PREFIX` errors tell the user to pass the exact relative path). Identical copies under one root resolve deterministically via `provably_same_skill()` (same resolved file or byte-identical `SKILL.md`) and `skill_candidate_rank()` (a real `SKILL.md` beats a legacy flat `<name>.md`, then the shallower path wins). `resolve_skill_catalog()` annotates every scanned copy with `status`: `unique` (loadable by bare name), `ambiguous` (loadable only by its exact relative path), or `shadowed` (hidden by a higher tier, with a one-time warning per pairing).

### Trusted Project-Local Skills and Quarantine

Project-local skills are a prompt-injection vector if auto-sourced from any clone, so they load only when the project root is listed under `skills.trusted_project_dirs` in `config.yaml` (`hermes skills trust|untrust <path>` manages the list; `skills.project_discovery: false` disables discovery entirely). Trust is a repo-level decision made once, but a `git pull` could inject a malicious skill into an already-trusted repo — so **every** project `SKILL.md`'s parent dir is scanned with the same skills_guard scanner the Skills Hub uses (`agent/skill_utils.py::is_quarantined_project_skill()`). The content-hash–cached scan result lives under `HERMES_HOME/cache/project_skill_scans/`, never inside the repo. A `dangerous` verdict **quarantines** the skill: it is excluded from the index, `skills_list`, `skill_view` (even by explicit name), and slash commands. `caution` verdicts load, matching hub behavior for prose-level keyword hits. A scanner crash or missing scanner quarantines too (fail closed). `iter_project_skill_files()` is the single iteration chokepoint, so the quarantine cannot be bypassed by a call site forgetting the check.

Project entries appear in the skills index labeled with a `[project]` prefix on their description.

### Local, Create-Dir, and External Directories

`agent/skill_utils.py::get_all_skills_dirs()` returns the non-project search path: local `~/.hermes/skills/` first, then `skills.create_dir`, then `skills.external_dirs`. External entries are expanded (`~`, `${VAR}`), anchored at `HERMES_HOME` when relative, resolved absolutely, and deduplicated; non-existent paths are skipped and the local dir is silently dropped if it appears in the external list.

External directories are read-only from the skill system's perspective: they appear in listings and the system prompt, but new skills are written to the profile skills directory or `skills.create_dir`. `is_external_skill_path()` extends read-only ownership to trusted project skills as well: autonomous lifecycle maintenance treats both as externally owned, while user-directed tool calls may still edit them in place.

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

`agent/skill_utils.py::get_disabled_skill_names()` resolves both lists; the platform comes from `HERMES_PLATFORM` or the session's `HERMES_SESSION_PLATFORM`. Config values written as quoted JSON/Python-literal strings (how `hermes config set` stores lists) are parsed back into lists via `parse_config_string_list()` so a stringified list can't silently filter nothing. One name is never disableable: `ESSENTIAL_SKILLS = {"hermes-agent"}` — the agent's own operating manual, pointed at unconditionally by the system prompt.

Disabled matching is path-aware: `is_disabled_entry()` matches a resolved catalog entry by its declared name **or** its `load_name` — the exact path (like `a/one`) a same-tier duplicate's listing row shows and saves. A unique copy elsewhere that merely sits at the same relative path is not matched. `skill_view()` applies the same rule, so `--skills noisy-skill` and bundle invocations cannot inject an operator-disabled skill into the prompt: preload and bundle paths count disabled skills as missing and report them, never raise.

---

## 3. The Skills Index and System Prompt Integration

Skills are not loaded into the system prompt in full — that would explode token usage. Instead, `agent/prompt_builder.py::build_skills_system_prompt()` constructs a compact **index**: a categorized list of skill names and one-line descriptions, rendered inside `<available_skills>` tags.

```
## Skills
Before replying, scan the skills below. If a skill matches or is even partially relevant
…
<available_skills>
  frontend:
    - react-testing: Jest + React Testing Library workflow
    - nextjs-deploy: Vercel deployment steps
  devops:
    - docker-compose: Multi-container local orchestration
  mlops/agents [names only]: axolotl, llama-cpp
</available_skills>
```

The preamble instructs the model to load matching skills via `skill_view(name)`, err on the side of loading, fix broken skills with `skill_manage(action='patch')`, and offer to save new skills after difficult tasks. This keeps the system prompt small while ensuring the model knows what knowledge is available. Descriptions are truncated to a 60-character system-prompt budget (`SKILL_PROMPT_DESC_LIMIT`); skill authors are advised to front-load the trigger into the first ~57 characters.

In one-shot sessions (single query, no follow-up), the preamble swaps for a lighter `ONESHOT_SKILLS_LOAD_GUIDANCE` block — the index itself is unchanged — trimming the self-improvement footprint that an interactive session carries.

### Two-Layer Cache

Building the index is I/O-bound. Hermes defends against repeated full scans with a two-layer cache:

1. **In-process LRU** (`_SKILLS_PROMPT_CACHE`): Sized at 32 entries — one per profile × platform, so a gateway multiplexing a default profile and several bots doesn't thrash. The key carries `(skills_dir, extra_roots, available_tools, available_toolsets, platform_hint, disabled_set, compact_categories, oneshot_variant)`. The lock-protected OrderedDict is safe for concurrent gateway threads. Entries whose skills declare `requires_apps` skip the LRU: app presence is a host fact that changes without any `SKILL.md` changing, so those builds always re-evaluate.

2. **Disk snapshot** (`.skills_prompt_snapshot.json`, version 4): A JSON file persisted under `HERMES_HOME` keyed by an mtime/size manifest of every `SKILL.md` and `DESCRIPTION.md`. Entries carry the skill name, category, frontmatter name, `rel` (the `SKILL.md` path relative to its root — needed for duplicate-name resolution), description, platforms, conditions, and `requires_apps`. On cold start, if the manifest matches the current filesystem, the snapshot is deserialized and the expensive scan is skipped entirely; older snapshot versions are rebuilt.

If both layers miss, the code walks the filesystem, builds manifest entries, and writes a fresh snapshot for the next process restart.

### Cross-Tier Resolution in the Index

The index resolves names exactly the way `skill_view()` does: every tier is scanned (platform-, environment-, and app-gated copies still take part, since `skill_view` must be able to find them), the results run through `resolve_skill_catalog()`, and visible entries are indexed under their `load_name`. Three consequences land in the rendered index:

- A shadowed copy (higher tier owns the name) is hidden with a logged warning.
- A same-tier ambiguous duplicate is listed under its exact relative path (`a/one`), so a row the model picks is a row `skill_view` can load.
- Skills that share *both* name and relative path within one tier can't be listed at all — they get a rename note instead of a row `skill_view` would refuse.

### Per-Session Discovery Cache (`_skill_catalog`)

`_skill_catalog()` in `tools/skills_tool.py` caches its results per-session with a 30-second TTL (`_SKILLS_CACHE_TTL_SECONDS`, keyed separately for the disabled-filtered and with-disabled variants plus hidden rows). The cache key is a scan signature computed by `_skills_scan_signature()`: a tuple of `(directory_path, max_mtime)` for each scanned directory and its immediate children (one `scandir` per dir — catches skill add/remove inside categories, which does NOT bump the root dir's mtime), the disabled-set (config-driven — changes with no filesystem mtime bump), and the platform. If the signature matches and the TTL hasn't expired, the cached list is returned as shallow copies (callers may mutate the returned dicts without poisoning the cache). If a write races the scan, the signature changes, so the next call re-scans rather than serving a torn result. `clear_skills_cache()` drops the catalog for explicit reloads.

### Compact Categories (Posture-Driven Demotion)

When the agent adopts a coding posture (e.g., pair-programming mode), non-coding categories — including their nested subcategories — are "demoted" to a names-only line:

```
  social-media [names only]: tweet-thread, linkedin-post
```

The skill names remain visible — the model can still request them by name — but their descriptions are omitted to reduce noise. The rendered note explains that demoted categories are outside the current coding context and still load normally. Nothing is ever dropped: agent-created skills are the model's project memory, and it would not rediscover them via `skills_list`. This is controlled by the `compact_categories` argument to `build_skills_system_prompt()`.

### Category Descriptions

If a category folder contains a `DESCRIPTION.md` with YAML frontmatter, its `description` field is rendered as the category header (both on the local scan and for external roots). This lets skill authors provide context for an entire family of skills.

---

## 4. Skill Dispatch at Runtime

There are several ways a skill enters an active conversation:

1. **Explicit tool call**: The model calls `skill_view(name)` from inside the agent loop.
2. **Slash command**: The user types `/skill-name` (or `/bundle-name`) in the CLI or a messaging gateway.
3. **CLI preload**: The user passes `-s skill-name` at startup, injecting the skill into the system prompt for the whole session.
4. **Config auto-load**: `skills.auto_load` lists canonical skill names that are loaded into every new session on any surface (gateway auto-load renders each payload with an `[IMPORTANT: The "X" skill is auto-loaded…]` scaffold followed by the user's typed text).
5. **`/learn`**: The `learn` command distills a source (a directory, URL, this chat, notes) into a new skill — a foreground create that is *not* handed to curator management (see §6).

All of these converge on `agent/skill_commands.py::_build_skill_message()`, which formats the loaded skill into a user-facing message block.

### Loading a Skill (`agent/skill_commands.py`)

`_load_skill_payload(skill_identifier, task_id)` resolves an identifier (name, relative path, or absolute path) to a loaded skill dictionary. Absolute paths are normalized through `agent/skill_utils.py::normalize_skill_lookup_name()`: a path under a trusted root becomes a relative identifier, preferring the lexical path before symlink resolution so a trusted symlink is accepted; anything else passes through unchanged and is rejected by `skill_view()`.

The returned dictionary carries:
- `name`, `content` (optionally preprocessed), `raw_content`
- readiness fields (`setup_needed`, `setup_note`, `setup_skipped`, `readiness_status`, missing env vars / credential files)
- `linked_files` — a map of subdirectory → file list
- `skill_dir` — absolute path to the skill root
- `deps_note` — populated when declared `deps` could not be ensured

### Message Assembly (`_build_skill_message`)

Once loaded, the skill content is processed through a pipeline:

1. **Template substitution and inline shell**: `agent/skill_preprocessing.py` renders `${HERMES_SKILL_DIR}` / `${HERMES_SESSION_ID}` and expands ``!`cmd` `` snippets (§5) before any downstream block sees the content.
2. **Activation note**: A prominent banner tells the model this is a skill invocation, e.g.:
   > `[IMPORTANT: The user has invoked the "docker-compose" skill, indicating they want you to follow its instructions. The full skill content is loaded below.]`
3. **Skill directory hint**: The absolute `skill_dir` is injected with a footer note telling the agent to resolve relative paths against it and run scripts with the terminal tool.
4. **Config injection**: If the frontmatter declares `metadata.hermes.config` variables, their resolved values are printed in a `[Skill config (from ~/.hermes/config.yaml): …]` block so the agent knows what is configured.
5. **Setup notes**: If prerequisites are missing, a `[Skill setup note: …]` block is appended; a skipped setup gets its own "explain any reduced functionality" note.
6. **Supporting files list**: Discovered `references/`, `templates/`, `scripts/`, and `assets/` files are enumerated with skill-relative paths plus the `skill_view(name, file_path=…)` recipe.
7. **User instruction + runtime note**: If the invocation carried an instruction, it is appended after the body; the *stable scaffold prefix* ending at the instruction marker is registered with `agent.prompt_cache_boundary` so the provider's cache planner can put a breakpoint there instead of caching the whole message as one atomic block.

### Skill Readiness (env vars, credential files, pm deps)

`tools/skills_tool.py::_skill_readiness()` resolves three kinds of prerequisites at view time. `required_environment_variables` prompts for missing secrets where the surface allows (the vault capture path), persists them into `HERMES_HOME/.env`, and registers the names that are actually set for sandbox passthrough so `execute_code`/`terminal` can use them. `required_credential_files` registers existing host files for remote sandboxes and flags missing ones as `setup_needed`; remote-backed environments get an extra note that the requirements must exist inside the remote environment too. The `deps` frontmatter is the activation moment for pm packages: each declared name runs through `pm.ensure`, and failures never block the skill content — a `deps_note` carries the remedy (`hermes pm install …` and reload).

### Repeat-View Dedup

A same, unchanged skill file already loaded in this session returns a short stub instead of the full body (mirroring `read_file`'s unchanged-stub behavior; the cache is cleared on context compression). The background-review fork opts out: a stub there would skip the read-mark its read-before-write guard requires.

### Slash Commands (`agent/skill_commands.py`)

`scan_skill_commands()` builds a map from `/slug` to skill metadata over the shared skill catalog — only names whose resolution is `unique` get a command, so a bare slug never guesses between copies. Slugs are normalized by `slugify_skill_name()`:

- Lowercase; spaces and underscores → hyphens
- Invalid characters stripped — but `\w` keeps Unicode letters, so a CJK name registers its own `/command` instead of slugging to empty
- Collapse multiple hyphens; trim leading/trailing hyphens

Two collision rules apply. First, a skill whose slug would shadow a **core command** (name or alias) is *not* auto-registered: the skill map is consulted before built-in handlers, and `skill_command_collision_note()` renders a user-facing note ("slash command /x unavailable — name taken by built-in; use /skill x") in the `/skills` listing and command palette. Second, two skills normalizing to the same slug keep the first — which preserves the project > local > external precedence of the shared catalog.

The scan cache is **multi-slot**: keyed by `(platform, hermes_home, project_root)`, so one gateway process serving Telegram and Discord sees each platform's own `platform_disabled` view, a Desktop profile switch rescans, and two sessions in different repos each see their own project skills. `reload_skills()` (`/reload-skills`) clears every slot, drops the shared catalog cache and the plugin projection, and returns an `added`/`removed`/`unchanged`/`total` diff.

### Bundles (`agent/skill_bundles.py`)

A **skill bundle** is a YAML file under `~/.hermes/skill-bundles/*.yaml` (or `.yml`) that references multiple skills by name:

```yaml
name: backend-dev
description: Optional one-liner for listings
skills:
  - github-code-review
  - test-driven-development
instruction: |
  Optional extra guidance to inject above the skill bodies.
```

Bundles are resolved **before** single skills in slash-command dispatch: if `/research` matches both a bundle and a skill, the bundle wins. The bundle cache rescans when a bundle file or the directory mtime changes, and duplicate slugs keep the first alphabetically.

`build_bundle_invocation_message()` loads each referenced member through the shared `_load_skill_blocks()` loader, builds its activation block, and concatenates everything under a `_scaffold_header()` header:

```
[IMPORTANT: The user has invoked the "backend-dev" skill bundle,
loading 3 skills together. Treat every skill below as active guidance for this turn.]
```

Missing members are noted (`Skills missing (skipped): …`) as are platform-disabled ones, and neither aborts the bundle; the agent receives the ones that exist. Bundle files also get CLI CRUD (`save_bundle`/`delete_bundle`, surfaced as `hermes bundles`), and `reload_bundles()` returns the same diff shape as `reload_skills()`.

### Stacked Slash-Skill Invocations

Multiple skills can be invoked in a single slash command: `/skill-a /skill-b do XYZ`. `split_stacked_skill_commands()` consumes up to four additional leading skill tokens (stopping at the first non-skill or repeated token, which starts the user instruction; native messaging callers never resolve plugin commands here), and `build_stacked_skill_invocation_message()` assembles all skill bodies into one user message reusing the bundle scaffolding markers — the memory extractor needs no new plumbing. This enables ad-hoc multi-skill composition without defining a bundle YAML.

---

## 5. Preprocessing Pipeline (`agent/skill_preprocessing.py`)

Before a skill reaches the model, it passes through `preprocess_skill_content()`, gated by two config switches (`skills.template_vars`, default **on**; `skills.inline_shell`, default **off**):

### Template Variables
- `${HERMES_SKILL_DIR}` → absolute path to the skill directory.
- `${HERMES_SESSION_ID}` → current session/task ID.

Unresolved tokens are left in place so authors can spot them.

### Inline Shell
Syntax: ``!`bash_command` `` (single-line snippets only). Each command is executed with `bash -c` in the skill's directory, with the delegated-child subprocess environment and a configurable timeout (`skills.inline_shell_timeout`, default 10s). Failure modes are captured gracefully as markers instead of raising:
- Timeout → `[inline-shell timeout after 10s: cmd]`
- Bash missing → `[inline-shell error: bash not found]`
- Nonzero exit with no output at all → `[inline-shell exit N with no output: cmd]` (the "interpreter never ran the command" signature, e.g. a WSL stub without a distro)
- General error → `[inline-shell error: <msg>]`

Output longer than 4000 characters is elided (`agent.compression_marker.elide`). This prevents a runaway command from blowing out the context window.

### Community-Skill Trust Scoping

The inline-shell DSL scans as high severity — it is exactly what the Skills Hub install gate blocks community skills for. So a hub-installed skill whose lock entry carries `trust_level: "community"` **never** auto-executes `!`cmd`` snippets on view, even when it reached disk via `--force` or an older install: auto-running it would re-arm what the gate refused. Trusted and builtin lock entries (and anything without a lock entry — user-created, project, external) keep the flag's contract.

---

## 6. Provenance: Who Made This Skill?

Not all skills are equal in the eyes of the maintenance system. Hermes distinguishes provenance classes:

| Provenance | Source | Curator manages? |
|------------|--------|------------------|
| **Bundled built-in** | Seeded from the bundled `skills/` directory at install/update time | Only if `curator.prune_builtins` is enabled |
| **Hub-installed** | Downloaded from the Skills Hub (`tools/skills_hub.py`) | **Never** — external upstream owner |
| **Agent-created** | Written by the background self-improvement review fork via `skill_manage` | **Yes** — primary curation target |
| **Learn / foreground-created** | Written at the user's request (`/learn`, `skill_manage create` outside the review fork) | **No** — belongs to the user until adopted |

Provenance is tracked in `tools/skill_usage.py`, not inside the skill markdown. This keeps user-facing content clean and avoids merge conflicts during hub updates.

### How Provenance Is Determined

1. **Bundled**: Names are read from `~/.hermes/skills/.bundled_manifest`, a line-delimited file of `name:hash` entries created during startup/update sync — plus the curator suppression list, which only ever records built-ins, so a pruned built-in remains non-agent-authored even after its manifest entry is cleaned.
2. **Hub-installed**: Names are read from `~/.hermes/skills/.hub/lock.json`, which records the upstream source and local install path of every hub skill; the frontmatter name of each in-tree install path is included too.
3. **Agent-created**: A usage record with `created_by: "agent"` — an explicit **curator-management opt-in**, written by `record_created()` when the background review fork creates a skill or by `mark_agent_created()`/`adopt_skill()` when the user hands a skill over.

Manually authored skills (a user dropping a `SKILL.md` into the directory, or a foreground `/learn` or `skill_manage create`) are **never inferred** as agent-created. Foreground creates are stamped `created_by: "learn"` — a learning-signal marker, not the curation opt-in. `curator adopt <name>` is the declared handover that flips the marker (the inactivity clock is not reset). This prevents the curator from archiving a user's hand-crafted notes; `list_unmanaged_skill_names()` and `unmanaged_report()` surface eligible-but-unadopted skills for exactly this decision.

The `provenance()` label used by reports and the UI is one of `hub` | `bundled` | `external` | `agent`. `external` covers skills mounted from `skills.external_dirs` and absent from the local store — externally authored, not learned; local hand-made skills still count as `agent` (the label splits origin, not mutability).

### Suppression List for Built-ins

When the curator archives a bundled built-in, its name is appended to `~/.hermes/skills/.curator_suppressed`. The update-time re-seeder consults this list and skips suppressed names, ensuring a prune decision survives across updates. Restoring a skill clears its suppression entry.

---

## 7. The Curator: Automatic Skill Maintenance

The Curator (`agent/curator.py`) is a background orchestrator that reviews curator-managed skills and maintains the collection. It is **inactivity-triggered, not cron-driven**: when the agent is idle (at least `min_idle_hours`, default 2) and the last run is older than `interval_hours` (default one week), `maybe_run_curator()` runs a pass. A cross-process claim file (`skills/.locks/curator-run`, O_EXCL create with a one-hour stale takeover) guarantees one automatic pass per home; `curator.enabled` (default on) and the persisted `paused` flag in `.curator_state` gate it.

### Lifecycle States

Every curator-managed skill moves through states tracked in `~/.hermes/skills/.usage.json`:

| State | Meaning | Transition trigger |
|-------|---------|-------------------|
| `active` | Default, healthy | — |
| `stale` | Inactive for `stale_after_days` (default **14**) | Auto-transition |
| `archived` | Inactive for `archive_after_days` (default **30**); directory moved to `.archive/` | Auto-transition or LLM consolidation |

Pinned skills (`pinned: true` in their usage record) bypass all automatic transitions, as do skills referenced by active cron jobs (treated as pinned: usage only bumps when a job fires, so a paused or rare job must not age its skill out). `PROTECTED_BUILTIN_SKILLS` in `tools/skill_usage.py` is the hard floor — skills the curator never archives or consolidates regardless of any flag — and is currently **empty**: `/plan` graduated from bundled skill to built-in command on every surface, removing the last protected name. The mechanism stays tiny and code-level; the curator review prompt still carries its protected-list wording for future use.

### Automatic Transitions

`apply_automatic_transitions()` walks every row of `curated_report()` (curator-managed skills plus pinned-but-unmanaged ones) and compares its latest activity timestamp to cutoffs derived from `config.yaml`:

- `anchor <= archive_cutoff` and not already archived → archive
- `anchor <= stale_cutoff` and currently active → mark stale
- `anchor > stale_cutoff` and currently stale → reactivate

The "anchor" is the latest of `last_used_at`, `last_viewed_at`, or `last_patched_at`. If none exist, `created_at` is used so brand-new skills don't immediately archive themselves. Two additional guardrails: a skill whose record was never persisted is seeded (`first_seen_at` = now) and deferred one full window, and a **never-used skill (use_count 0) is never archived while younger than `stale_after_days`** — use=0 is absence of evidence, not staleness.

Bundled built-ins that become eligible for the first time (when `prune_builtins` flips on) are **re-anchored** by `reanchor_clock()`: their telemetry record can predate the curator's first sight by months, so without this every never-used built-in would go stale on the first pass. The re-anchor resets `created_at`/`first_seen_at` to now and reactivates any artifact-stale record. Config day/hour values are floored at 1 (`_bounded_count`), because a 0 would collapse the cutoffs onto "now" and mass-transition every skill unconfirmed.

Curator-actor archives go through `_archive_as_curator()`, which tags the audit ledger entry as actor `curator` so autonomous transitions are distinguishable from foreground calls.

### External Skill Protection

External skills (from `skills.external_dirs`) and trusted project skills are protected from background curation: the write guards prevent the curator from modifying or archiving skills it doesn't own, and the LLM candidate list is pre-filtered to local curator-managed skills so the fork never burns tool calls on refusals.

### Restore Semantics

`restore_skill()` moves an archived skill back to the flat live layout. Resolution is exact: the archive directory named exactly like the skill, then its `<name>-YYYYMMDDHHMMSS` timestamped collision duplicate (a bare prefix match would let restoring "git" steal "git-helpers"), then a frontmatter-`name` fallback for older nested archives. Restoration refuses a name that has since become hub-installed or re-bundled, and clears the suppression list entry so the re-seeder may manage it again.

### LLM Consolidation Pass

The LLM consolidation (umbrella-building) pass is **opt-in** — it is off by default (`DEFAULT_CONSOLIDATE = False`). When off, a curator run does only the deterministic inactivity prune (mark stale / archive long-unused skills) and skips the forked aux-model review entirely. Set `curator.consolidate: true` in config (or pass `--consolidate` on the CLI) to opt into the LLM pass that merges overlapping skills into class-level umbrellas.

When enabled, the curator spawns a **forked AIAgent** running a dedicated review prompt (`CURATOR_REVIEW_PROMPT`). This is an "umbrella-building" pass, not a passive audit. Its goals:

- Identify prefix clusters of related skills (`hermes-config-*`, `pr-*`, `python-*`, etc.)
- Merge narrow skills into broad **umbrella** skills with labeled subsections
- Distill session-specific detail into `references/`/`templates/`/`scripts/` under the umbrella (moved *through* the ledgered `skill_manage` surface — `write_file`, then `remove_file`, then `delete` — never a shell `mv`, which would write unledgered bytes and hollow out rollback)
- Archive the now-redundant siblings

The prompt enforces hard rules:
1. Do not touch bundled, hub-installed, or external-dir skills (bundled skills are excluded from the candidate list even under `prune_builtins` — the deterministic pass owns their archival).
2. Do not delete — only archive (recoverable). During a consolidation, `skill_manage delete` routes through the recoverable archive primitive rather than a hard rmtree, so a misjudged consolidation is undoable.
3. Do not touch pinned or protected skills; never prune a cron-referenced skill outright (consolidation-with-rewrite is allowed).
4. Judge overlap on **content**, not on usage counters (which are often zero for new skills) — and never prune a young never-used skill.
5. Produce a structured YAML summary of `consolidations:` and `prunings:`.

The fork is deliberately constrained: `enabled_toolsets=["skills"]` and **no `terminal` toolset** (every filesystem mutation must be ledgered and rollback-able), `max_iterations=9999` (an umbrella pass over hundreds of skills takes 50–100 API calls), the auxiliary task slot for provider/model resolution, and the background-review write-origin ContextVar so `skill_manage`'s unattended-write guards fire. Classification of what happened is reconciled from three signals, most authoritative first: the per-delete `absorbed_into` declaration, the model's structured YAML block, and a tool-call audit heuristic that catches hallucinated umbrellas and omissions.

The provider/model for the fork resolves through the canonical auxiliary task plumbing — `auxiliary.curator.{provider,model}` when both are set, falling back to the deprecated `curator.auxiliary.*` (with a migration warning), then the main chat model.

### Dry-Run Mode

`hermes curator run --dry-run` skips the stale/archive transitions **and** instructs the fork to report only (a banner forbids every mutating action while encouraging reads). It does not bump `last_run_at`/`run_count` — a preview must not push the next real pass out — but still writes a `REPORT.md` under `~/.hermes/logs/curator/`, so users can preview what would happen.

### Reports, Snapshots, and Recovery

Every run writes a timestamped directory under `logs/curator/{YYYYMMDD-HHMMSS}/` (suffixed `-2`, `-3`, … when a crash-rerun lands in the same second) containing:
- `run.json` — machine-readable full record
- `REPORT.md` — human-readable markdown summary
- `cron_rewrites.json` — only when scheduled jobs had skill references updated

Before an LLM consolidation run, `agent/curator_backup.py` takes a whole-tree snapshot of the skills directory (best-effort; the deterministic prune only renames directories, which is its own undo). Recovery commands:
- `hermes curator restore <name>` — move a skill from `.archive/` back to the live directory.
- `hermes curator pin <name>` / `unpin` — protect a skill from future auto-transitions.
- `hermes curator rollback <entry-id>` — undo one ledgered mutation; `--list` lists whole-tree snapshots; `hermes curator ledger` reads the audit trail (§10).

### Cron Job Rewrites

When the curator consolidates skill X into umbrella Y, any scheduled cron job that listed X would fail to load it. The curator automatically rewrites those job definitions in-place via `cron.jobs.rewrite_skill_refs()`, substituting Y for X (or dropping X if it was pruned without a target) — the `absorbed_into` declaration is what drives the mapping, which is why the review prompt requires it on every consolidation delete. This keeps scheduled tasks working across consolidation passes.

---

### Security Guards in `skill_view()`

`tools/skills_tool.py::skill_view()` implements several defense-in-depth checks before returning skill content:

1. **Path validation** (`_skill_lookup_path_error()`): Rejects absolute paths, Windows drive letters (`C:\\skills\\foo` — their `:` would be misread as a plugin namespace separator), and `..` traversal components. This prevents a malicious model from requesting arbitrary files outside the skills directory. The validation runs *before* the `:` plugin dispatch for exactly that reason.

2. **Tiered collision resolution**: Bare names resolve through the shared catalog precedence — lowest tier wins, identical copies under one root resolve to the best rank (real `SKILL.md` over legacy flat markdown, shallower path). If more than one *different* skill matches a bare name in the same tier, the function refuses to guess and returns an error with all matching paths and their loadable relative paths. Silent shadowing is treated as a bug class, not a convenience.

3. **Hard platform gate**: A `platforms:` mismatch refuses the load outright (`UNSUPPORTED` readiness status). This is the one offer-time-only gate that *is* enforced again on explicit load.

4. **Disabled gate**: Refuses a skill disabled by declared name or by the exact `load_name` path of this copy, so a same-tier duplicate can't ride an unrelated copy's enablement.

5. **Project quarantine gate**: A project-tier skill with a `dangerous` scan verdict cannot load even by explicit name (the same chokepoint the index and `skills_list` use).

6. **Security warnings (warn, never block)**: A `WARNING`-level log fires when the skill file resolves outside every trusted root, and/or when the content contains any `_INJECTION_PATTERNS` substring (a short fixed list: "ignore previous instructions", "you are now", `<system>`, and friends). Many legitimate skills quote examples containing these phrases, so the check is audit-only — on both the local and plugin-skill paths.

7. **Traversal guard for linked files**: When `file_path` is provided, the resolved target is validated to still sit inside the skill directory; `..` components are rejected outright.

8. **Load-time dependency ensure**: Declared `deps` are `pm.ensure`d at view time, with failures surfaced as a `deps_note` rather than blocking the content.

### Skills Guard Scan Caching (`tools/skills_guard.py`)

`scan_skill_cached()` provides attestation-bound scan caching for the skills security scanner (`SCANNER_VERSION = "skills-guard-v8"`). It computes a `full_content_hash()` — a canonical SHA-256 over relative paths and exact file bytes — and caches the scan result, by default in a `.scan-cache/` directory beside the skill (project-skill scans pass an explicit cache dir under `HERMES_HOME` instead, so nothing is written into a user checkout). On a cache hit where the bundle hash, scanner version, source, and source URL all match, the cached `ScanResult` is returned with its `scan_provenance` metadata (`fresh: False`). This avoids re-scanning unchanged skills on every load.

The scanner classifies sources into trust levels — `builtin` (never scanned), `trusted` (the openai/anthropics/huggingface/NVIDIA skills repos: caution allowed), `community` (any finding blocks), and `agent-created` (dangerous findings become an `ask` error instead of a silent block, but only when `skills.guard_agent_created` is on). `INSTALL_POLICY` maps (trust level × verdict) → allow/block, and `should_allow_install()` is the single decision point the hub install path uses.

### Slash Command Collision Guard

`agent/skill_commands.py` guards skill slash commands against collisions with core commands and slug collisions between skills. `skill_command_collision_note()` is the single source of the collision predicate: `scan_skill_commands` uses it to skip auto-registration (the skill map is consulted before built-in handlers), and the `/skills` listing plus the command palette render the note so the skipped skill is explained where the user looks, not only in the log. Slug dedup between skills keeps the first (catalog precedence).

---

### Plugin-Provided Skills

Skills are not limited to the filesystem. Plugins can register skills via the plugin manager (`hermes_cli.plugins`). A qualified name like `plugin:skill-name` routes to `_serve_plugin_skill()` instead of the local directory walk. A Windows drive path is rejected first so its `:` can't be reinterpreted as a namespace separator; namespaces must match `^[a-zA-Z0-9_-]+$`. Invalid namespaces or missing plugin skills fall through to the local scan with a clear error message (a `p:missing-skill` lookup against a plugin that exists lists that plugin's available skills).

Plugin skills receive:
- A **bundle-context banner** listing sibling skills from the same plugin, so the agent knows what else is available.
- The same template-variable and inline-shell preprocessing as local skills.
- The same prompt-injection scan and logging.
- Readiness gating: platform, environment, and app gates are re-evaluated from the SKILL.md the plugin will actually load, not registration-time hints.

Memory-provider skills load through `plugins.memory` rather than the general registry; `skill_view` lazy-loads the namespaced provider once when the qualified name matches the active memory provider and clears stale registry entries whose file was deleted out of band.

Plugin skills also project into the **interactive-only** slash namespace (`/plugin:skill`) via a cached projection (`get_plugin_skill_commands()`), reused while its identity tag holds — the projection sits on the per-keystroke completion path, so re-parsing every plugin SKILL.md per Tab would cost real latency. Config disable lists (`skills.disabled`, `plugins.disabled`) are applied on every call so a config-only disable takes effect without a rescan. Plugin slash commands are CLI/TUI/desktop only — messaging gateways and native command menus never see them, and filesystem commands win merge collisions.

---

### Skill Manager Tool (`tools/skill_manager_tool.py`)

The agent interacts with the skill library through the `skill_manage` tool. The advertised call shape is an **operations array** — each op is one of:

| Action | Purpose |
|--------|---------|
| `create` | Write a new `SKILL.md` (full frontmatter + body) into the profile skills directory or `skills.create_dir`, optionally under a `category` |
| `patch` | Apply a targeted `old_string`/`new_string` fix (or a full `content` rewrite) to a skill's `SKILL.md` or a supporting file |
| `write_file` | Add a supporting file under `references/`, `templates/`, `scripts/`, or `assets/` (the skill must already exist) |
| `remove_file` | Drop a supporting file |
| `delete` | Archive a skill (move to `.archive/`); curator consolidations must pass `absorbed_into=<umbrella>` — a delete naming a missing or self target is refused |

There is no `view` action — reading goes through `skill_view`. A legacy flat single-op shape is still accepted for old transcripts and staged-write replay, but the batch array is what the schema advertises, and it applies atomically (up to 20 ops; any failure rolls every touched skill back; per-op locks are acquired in one stable path order).

Provenance and safety on writes:
- Every successful mutation lands in the append-only audit ledger (§10), clears the skills system-prompt cache, and bumps curator telemetry (`record_created` on create; `bump_patch` on patch/edit/write_file/remove_file). Hard deletes forget the usage record; curator archives keep theirs as `STATE_ARCHIVED`.
- On `create`, the **origin decides** provenance: a background-review (curator fork) create calls `record_created(agent_created=True)` and is enrolled in curator management; a foreground create is stamped `created_by: "learn"` — the user's skill, not the curator's.
- Absolute paths are rejected unless they lie under a trusted root; `..` traversal is blocked via `tools/path_security.py`; per-skill mutation locks serialize read-modify-write cycles.
- Content passes `_validate_frontmatter()` (name/description presence and limits, size caps) and the advisory `tools/skill_linter.py` linter — findings like banned shell-utility prose (`grep`/`cat`/`sed` should name `search_files`/`read_file`/`patch`), marketing adjectives, and POSIX-only primitives in scripts without a `platforms:` gate are attached to the result as `lint_warnings` with a hint, never blocking.
- Skills are too large to review inline, so **every** write is staged through the approval write gate regardless of origin (`skills.write_approval`; replay of an approved staged write bypasses it), and the background-review guards enforce read-before-write: a patch/remove/write on content not loaded via `skill_view` in the same review turn is refused.

---

## 8. Scaffolding Markers and Memory Hygiene

When a slash command or bundle expands into a user message, the resulting text includes scaffolding headers. If a memory provider stores the raw turn, it would capture the entire skill body rather than what the user actually asked. `agent/skill_commands.py::extract_user_instruction_from_skill_message()` reverses this by looking for canonical marker strings:

- Single skill: `The full skill content is loaded below.` → the instruction follows the body — the **last** marker (a body may quote the marker) is the user's.
- Bundle: `User instruction:` → extract everything before the first loaded skill block — the **first** marker.

If no instruction was provided (a bare `/skill-name`), the function returns `None`, signaling memory providers to skip the turn entirely. `describe_skill_invocation()` re-renders the turn the way the user typed it (`/work — fix the title leak`) for previews and titles, including the gateway **auto-load scaffold** (a channel-bound skill prepended to a new session's first message): there the footer note's closing sentence marks where the payload ends and the typed request begins. These markers are asserted in tests to stay byte-identical with the builders, so drift between `_build_skill_message()` and the extractor is caught early.

---

## 9. The Skill Ledger and Backups (`tools/skill_ledger.py`)

Every skill mutation — any actor — appends one JSONL entry to `~/.hermes/skills/.curator_ledger.jsonl` with before/after file manifests whose contents are stored content-addressed (sha256-deduped) under `~/.hermes/.curator_backups/blobs/`. JSONL rather than a state DB: durable, greppable, and it survives DB resets. Actors are `curator`, `agent`, or `user`.

The ledger is **telemetry, not a gate**: every public write path swallows and logs failures, so a broken ledger never breaks a tool call — except `rollback_entry()`, which takes a fresh safety capture first and fails closed if that capture fails. Whole-tree snapshots (`agent/curator_backup.py`) complement per-mutation rollback: `hermes curator rollback <id>` undoes one ledger entry, `--list` (or `--id`) restores a snapshot. Deletes and archives are captured as *complete packages* — a consolidation may have re-homed support files first, so a disk-only capture would restore a hollow skill; the capture is filled from the newest curator backup. The ledger size is bounded (`skills.ledger_max_bytes`, default 5 MB) and its blobs are garbage-collected with a grace window for in-flight captures.

---

## 10. Skill Usage Telemetry (`tools/skill_usage.py`)

Every significant interaction with a skill is recorded in a sidecar JSON file, `~/.hermes/skills/.usage.json`, keyed by skill name. This file is deliberately **not** embedded in the skill markdown — it keeps operational telemetry out of user-authored content and avoids merge conflicts during hub updates.

### Record Schema

Each entry is a dict with the following fields:

| Field | Purpose |
|-------|---------|
| `created_by` | `"agent"` (curator opt-in), `"learn"` (foreground create), `"installed"` (hub install), or `null` |
| `use_count` | Times the skill was loaded into an active turn |
| `view_count` | Times the skill was inspected via `skill_view()` |
| `patch_count` | Times the skill was edited via `skill_manage` |
| `patch_generation` | Monotonic edit counter used to detect reuse-after-patch |
| `last_reused_patch_generation` | Generation a use was first observed after a patch |
| `last_used_at` | ISO timestamp of the most recent load |
| `last_viewed_at` | ISO timestamp of the most recent view |
| `last_patched_at` | ISO timestamp of the most recent edit |
| `created_at` | When the record was first seeded |
| `first_seen_at` | When the curator first anchored this skill's inactivity clock |
| `state` | `active`, `stale`, or `archived` |
| `pinned` | Boolean — exempts the skill from auto-transitions |
| `archived_at` | Timestamp when the skill was archived |

The sidecar is read and written atomically under a cross-process file lock (`fcntl.flock` on Unix, `msvcrt.locking` on Windows; the lock helper is re-entrant per thread and degrades to a no-op where neither exists), so concurrent gateway sessions and CLI invocations do not corrupt it. Records are backfilled on read: an older record missing newer keys gets fresh defaults.

### Telemetry Hooks

Three bump functions are called from the tool layer:

- `bump_view(name)` — called by `skill_view()` every time a skill is read.
- `bump_use(name)` — called when a skill is actually injected into a conversation (slash command, bundle, or preload). `skill_view()` bumps it too: viewing is actively loading the skill to act on it, and the curator's stale timer keys off `last_used_at`. It also computes the `reuse_after_patch` fact (a use observed after a later patch generation) for the lifecycle hook.
- `bump_patch(name)` — called by `skill_manage` for patch/edit/write_file/remove_file.

All three are best-effort: a failure logs at `DEBUG` and returns silently so a broken sidecar never breaks the underlying tool call. Successful lifecycle writes (`created`, `loaded`, `patched`, `archived`, `restored`, `stale`) also emit a plugin-facing `on_skill_lifecycle` hook with the provenance label and reuse facts, and the telemetry layer is provenance-agnostic by design: counters are recorded for every skill on disk, while lifecycle mutators refuse to write state onto skills that aren't curation-eligible.

### Seeding Built-ins

When `curator.prune_builtins` is enabled, built-in skills suddenly become curation-eligible. But they may have no usage record — or one months old from incidental telemetry. `seed_record_if_missing()` writes a fresh record with `first_seen_at` set to "now" the first time the curator sees a built-in, and `reanchor_clock()` (§7) fixes the pre-anchored case. Together these anchor the inactivity clock to the moment of eligibility, preventing a mass archive of all built-ins the instant the flag flips on.

### Archive and Restore

`archive_skill(name)` moves the skill directory to `~/.hermes/skills/.archive/`, flattened **under the skill name** (the skill `huggingface-accelerate` in `mlops/training/` archives as `archive/huggingface-accelerate`, so restore/list/purge key on the name, not the directory layout). If a collision exists, a timestamp suffix is appended, and the archive copy's mtime is refreshed so `curator purge` ages archives by archive time rather than last edit. Restoration is the inverse (§7): exact name, then timestamped duplicate, then frontmatter-name fallback, with suppression-list cleanup.

Both operations refuse to act on:
- Hub-installed skills (external upstream owner)
- External-dir skills (read-only to the curator)
- Bundled built-ins when `prune_builtins` is disabled

---

## 11. Architecture Diagrams

### Skill Data Flow

```
┌─────────────────┐
│  User / Model   │
│  (slash cmd or  │
│  skill_view)    │
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
│        build_skills_system_prompt()  │
│         (agent/prompt_builder.py)    │
└─────────────────┬────────────────────┘
                  │
     ┌────────────┼────────────┐
     ▼            ▼            ▼
┌─────────┐ ┌──────────┐ ┌────────────┐
│ Layer 1 │ │ Layer 2  │ │  Cold Path │
│ In-proc │ │  Disk    │ │ Filesystem │
│  LRU    │ │ Snapshot │ │    Scan    │
│ (32 max)│ │ JSON (v4)│ │ + manifest │
└─────────┘ └──────────┘ └────────────┘
```

### Curator Lifecycle

```
┌─────────┐     ┌──────────┐     ┌───────────┐
│ active  │────▶│  stale   │────▶│ archived  │
│         │     │(14 days) │     │(30 days)  │
└─────────┘     └──────────┘     └───────────┘
    ▲                                  │
    │     skill used again             │
    └──────────────────────────────────┘
           (reactivate)
```

---

## 12. Configuration Variables

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

`agent/skill_utils.py::resolve_skill_config_values()` reads these declarations and returns a map of logical keys to current values. Path-like values are expanded against the **subprocess HOME** — `~` and `$HOME` are resolved to the HOME Hermes injects into tool subprocesses, not the control process's — so in a container where the two differ, a default path lands somewhere the tools can actually read. When a skill is loaded, `_inject_skill_config()` appends a `[Skill config: ...]` block to the message so the agent sees the live values without opening `config.yaml` itself. Declarations missing `key` or `description` are skipped; `prompt` defaults to the description.

---

## 13. External Skill Directories

Advanced users or teams can share skill libraries via `skills.external_dirs`:

```yaml
skills:
  external_dirs:
    - ~/company-hermes-skills
    - /opt/shared/hermes/skills
```

Paths are expanded and anchored at `HERMES_HOME` when relative. Duplicates and non-existent paths are ignored. External skills:
- Appear in `skills_list()` and the system prompt index.
- Can be loaded via `skill_view()` and slash commands.
- Are **not** writable by `skill_manage`'s autonomous paths — creations go to the local directory or `skills.create_dir`; `is_external_skill_path()` also covers trusted project skills.

In the tiered precedence, `external_dirs` is the lowest tier: a project, profile, or `create_dir` skill shadows a same-named external skill (with a one-time warning). This separation lets organizations maintain a canonical shared skill set while allowing individual users to supplement it with personal skills.

---

## 14. Summary

The Hermes skill system is a deliberate architectural bet: **capability lives at the edges**. Rather than teaching the core agent every new workflow, Hermes teaches the agent to read instructions from the filesystem at runtime. This yields several powerful properties:

- **Progressive disclosure**: The system prompt carries only an index; full skill bodies are loaded on demand, with repeat views deduplicated to stubs.
- **Precedence-aware naming**: Skill names resolve through one shared tier order — trusted project > profile > create_dir > external — so the index, listings, slash commands, and cron all agree on which copy of a name is real, and ambiguity is refused rather than guessed.
- **Platform sensitivity**: Skills gate themselves by host OS, runtime environment, gateway channel, and registered app availability without central coordination.
- **Injection defense**: Project-local skills load only from trusted repos, are scanned on every content change, and are quarantined on a dangerous verdict; all skills pass warn-only injection audits.
- **Automatic maintenance**: The Curator ages out stale skills and consolidates overlapping ones (opt-in), with every mutation ledgered and rollback-able.
- **Provenance safety**: Built-ins, hub skills, learn-created skills, and agent-created skills are kept in separate lanes so automatic maintenance never destroys upstream, user-taught, or load-bearing content.
- **Composability**: Bundles and stacked slash invocations let users alias multi-skill workflows to a single command, and template variables + inline shell let skills adapt dynamically to the current session.

Understanding the skill system is essential for anyone extending Hermes, debugging why a skill does (or doesn't) appear in the index, or designing new background maintenance workflows. The core invariant is simple: a skill is just a directory with a `SKILL.md`, and everything else — tiered discovery, indexing, dispatch, caching, curation, provenance, the ledger — is infrastructure around that single convention.
