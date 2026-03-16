# DevClaw — Product Plan

> **DevClaw** is an AI-powered multi-agent engineering tool built on OpenManus.
> It has two modes: **`--que`** (default) for conversational knowledge and debugging,
> and **`--run`** for autonomous code implementation and PR creation.
> Both modes share the same repo memory, config, and agent infrastructure.
> **`--run` mode never touches `main` or any default branch. Ever.**

---

## Two modes, one tool

### `--que` mode (default)
Conversational knowledge assistant. Ask questions about your repos in natural language.
Sessions persist — every conversation has a session ID you use to continue the chat.
Covers three use cases: onboarding a new developer, understanding architecture, debugging production issues.

```bash
# --que is the default — these are equivalent
python run_devclaw.py "how does authentication work?"
python run_devclaw.py --que "how does authentication work?"

# Continue a session
python run_devclaw.py "what would break if I changed the token expiry?" --session que-a1b2c3

# Interactive REPL
python run_devclaw.py --interactive --repos backend-api

# List all past sessions
python run_devclaw.py --list-sessions
```

### `--run` mode
Autonomous implementation swarm. Give it a task, it clones repos, writes code,
runs tests, and opens linked PRs — one per repo, same branch name across all.

```bash
python run_devclaw.py --run \
  --prompt "Add Redis caching to the /search endpoint" \
  --branch feat/redis-caching \
  --repos backend-api

python run_devclaw.py --run --issue 142 --issue-repo your-org/backend-api
python run_devclaw.py --run --task tasks/add-rate-limiting.yaml
python run_devclaw.py --run --task tasks/add-rate-limiting.yaml --dry-run
```

---

## What each mode does

### `--que` mode flow
1. Create or resume a session (session ID printed after every response)
2. Run `MemorySyncAgent` per repo — SHA check, update only changed files
3. Load conversation history from the session
4. Classify the question — onboarding, architecture, or debugging
5. `RepoExplorerAgent` finds relevant code and facts (reads from memory, not full repo)
6. `DiagnosticAgent` traces execution paths (debugging use case only)
7. `SynthesisAgent` assembles the answer, building on conversation history
8. Write turn to session store — answer available next time via `--session`

### `--run` mode flow
1. Receive task from prompt, GitHub Issue, or YAML file
2. Score complexity 1–5 — determines team size
3. `PlannerAgent` reads all repos, produces per-repo implementation plans
4. `RepoWorkerAgent` × N — parallel, one per repo: clone → branch → code → test → commit → PR
5. `ReviewAgent` — cross-repo consistency + PR quality (LLM-as-Judge)
6. Back-fill cross-repo PR links
7. Open one PR per repo on the feature branch — never on `main`

---

## Technology foundation

| Layer | Technology |
|---|---|
| Agent framework | OpenManus (MIT) |
| Multi-agent coordination | Claude Code native Task tool |
| LLM | Configurable — GPT-4o, Claude, etc. via OpenManus LLM abstraction |
| GitHub integration | PyGithub |
| Build validation | Docker + pytest via subprocess |
| Memory & sessions | SQLite (`devclaw.db`) — zero infrastructure |
| Config | TOML (`repos.toml` + `config.toml`) |
| Secrets | python-dotenv (`.env` file, never committed) |
| License | Apache 2.0 |

---

## Repository structure

```
OpenManus/                              ← base framework (do not modify internals)
├── CLAUDE.md                           ← Claude Code project guide (create first)
├── .env.example                        ← secrets template
├── run_devclaw.py                      ← CLI entry point — both modes
├── config/
│   ├── config.toml                     ← add [devclaw] section
│   └── repos.toml                      ← your repo allowlist
├── tasks/                              ← --run mode YAML task files
│   └── example-task.yaml
├── app/
│   ├── devclaw_errors.py               ← exception hierarchy (both modes)
│   ├── cost_tracker.py                 ← token + cost tracking (both modes)
│   ├── audit_logger.py                 ← JSONL audit log (both modes)
│   ├── run_state.py                    ← run state persistence (--run mode)
│   ├── preflight.py                    ← startup checks (both modes)
│   ├── agent/
│   │   │
│   │   │   ── --run mode agents ──
│   │   ├── devclaw_orchestrator.py     ← RunOrchestratorAgent
│   │   ├── devclaw_planner.py          ← PlannerAgent
│   │   ├── devclaw_repo_worker.py      ← RepoWorkerAgent
│   │   ├── devclaw_coder.py            ← CoderAgent (Plan-and-Solve)
│   │   ├── devclaw_tester.py           ← TesterAgent
│   │   ├── devclaw_reviewer.py         ← ReviewAgent (LLM-as-Judge)
│   │   │
│   │   │   ── --que mode agents ──
│   │   ├── devclaw_query_orchestrator.py ← QueryOrchestratorAgent
│   │   ├── devclaw_repo_explorer.py    ← RepoExplorerAgent
│   │   ├── devclaw_synthesis.py        ← SynthesisAgent
│   │   ├── devclaw_diagnostic.py       ← DiagnosticAgent (debugging only)
│   │   │
│   │   │   ── shared agents ──
│   │   ├── devclaw_memory_sync.py      ← MemorySyncAgent (both modes)
│   │   └── devclaw_memory_writer.py    ← MemoryWriterAgent (both modes)
│   │
│   ├── flow/
│   │   ├── devclaw_flow.py             ← DevClawFlow (--run mode)
│   │   └── devclaw_query_flow.py       ← QueryFlow (--que mode)
│   │
│   ├── task_input/                     ← --run mode task loading
│   │   ├── schema.py
│   │   └── loader.py
│   │
│   ├── session/                        ← --que mode session management
│   │   ├── manager.py                  ← SessionManager
│   │   └── schema.py                   ← Session, ConversationTurn dataclasses
│   │
│   ├── memory/                         ← shared by both modes
│   │   ├── store.py                    ← DevClawMemoryStore (SQLite wrapper)
│   │   └── schema.py                   ← MemoryItem, RepoMemoryState
│   │
│   └── tool/
│       │   ── --run mode tools ──
│       ├── git_tools.py                ← clone, branch, commit, push, status
│       ├── github_pr_tool.py           ← create + update PRs
│       ├── build_test_tool.py          ← docker build + pytest
│       ├── secret_scan_tool.py         ← block secrets before push
│       ├── pr_size_check_tool.py       ← flag oversized PRs
│       │
│       │   ── --que mode tools ──
│       ├── conversation_store_tool.py  ← read/write sessions + turns
│       ├── query_memory_tool.py        ← read repo_facts for exploration
│       │
│       │   ── shared tools ──
│       ├── repo_reader_tool.py         ← full repo read (first-run index)
│       ├── spawn_agent_tool.py         ← Claude Code Task tool wrapper
│       └── memory_store_tool.py        ← read/write memory store
│
└── tests/
    └── devclaw/
        ├── test_safety.py
        ├── test_task_input.py
        ├── test_memory.py
        ├── test_sessions.py
        └── test_preflight.py
```

---

## Shared memory store — `devclaw.db`

Both modes read from and write to a single SQLite database. No separate infrastructure.

```
devclaw.db
├── repo_state          ← SHA per repo — freshness tracking (both modes)
├── repo_facts          ← structured knowledge about each repo (both modes)
├── run_log             ← episode log: what --run did in past runs (--run writes, --que reads)
├── sessions            ← conversation sessions (--que mode)
└── conversation_turns  ← full dialogue history per session (--que mode)
```

**How modes share memory:**
- `--run` writes `repo_facts` as it works — CoderAgent discovers patterns, MemoryWriterAgent stores them
- `--que` reads `repo_facts` for context — RepoExplorerAgent retrieves targeted facts instead of re-reading files
- `--que` reads `run_log` — ReviewAgent can see what DevClaw has done to a repo recently
- `MemorySyncAgent` runs at the start of both modes — SHA check ensures facts are always current

### SQLite schema

```sql
-- SHA tracking — one row per repo, updated on every run
CREATE TABLE repo_state (
    repo_name       TEXT PRIMARY KEY,
    last_synced_sha TEXT NOT NULL,
    last_synced_at  TEXT NOT NULL,
    last_run_id     TEXT,
    status          TEXT DEFAULT 'empty'   -- 'empty' | 'fresh' | 'syncing'
);

-- Structured facts about each repo — keyed to source file + SHA
CREATE TABLE repo_facts (
    id          TEXT PRIMARY KEY,
    repo_name   TEXT NOT NULL,
    source_file TEXT NOT NULL,
    fact_type   TEXT NOT NULL,   -- 'pattern' | 'convention' | 'warning' | 'structure'
    content     TEXT NOT NULL,
    embedding   FLOAT[1536],     -- sqlite-vec column for similarity search
    synced_sha  TEXT NOT NULL,
    written_by  TEXT NOT NULL,
    created_at  TEXT NOT NULL
);

-- Episode log — what --run mode did in each run
CREATE TABLE run_log (
    id            TEXT PRIMARY KEY,
    run_id        TEXT NOT NULL,
    repo_name     TEXT NOT NULL,
    agent_name    TEXT NOT NULL,
    outcome       TEXT NOT NULL,   -- 'success' | 'failure' | 'wip'
    files_changed TEXT,            -- JSON list
    summary       TEXT,
    pr_url        TEXT,
    created_at    TEXT NOT NULL,
    UNIQUE(run_id, repo_name)
);

-- Conversation sessions — one per --que conversation thread
CREATE TABLE sessions (
    session_id  TEXT PRIMARY KEY,
    title       TEXT,              -- auto-generated from first question
    repos       TEXT NOT NULL,     -- JSON list of repo names in scope
    created_at  TEXT NOT NULL,
    updated_at  TEXT NOT NULL,
    turn_count  INTEGER DEFAULT 0,
    status      TEXT DEFAULT 'active'   -- 'active' | 'archived'
);

-- Full dialogue history — every question and answer
CREATE TABLE conversation_turns (
    id               TEXT PRIMARY KEY,
    session_id       TEXT NOT NULL REFERENCES sessions(session_id),
    turn_number      INTEGER NOT NULL,
    role             TEXT NOT NULL,      -- 'user' | 'assistant'
    content          TEXT NOT NULL,
    use_case         TEXT,               -- 'onboarding' | 'architecture' | 'debugging'
    repos_referenced TEXT,               -- JSON list
    files_referenced TEXT,               -- JSON list
    created_at       TEXT NOT NULL
);
```

---

## `--que` mode — usage and session flow

### Starting a new session

```bash
python run_devclaw.py "how does authentication work across our services?" \
  --repos backend-api frontend

# Output:
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# DevClaw  |  use case: architecture
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
#
# Authentication uses JWT tokens issued by backend-api...
# [full markdown answer]
#
# Files referenced: app/auth/jwt.py, app/middleware/auth.py
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# Session: que-a1b2c3
# Continue: python run_devclaw.py "your next question" --session que-a1b2c3
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### Continuing a session

```bash
python run_devclaw.py "what would break if I changed the token expiry?" \
  --session que-a1b2c3

# DevClaw loads the full conversation history from the session.
# The answer references and builds on the previous turn.
```

### Interactive REPL

```bash
python run_devclaw.py --interactive --repos backend-api frontend

# DevClaw Queue > how does auth work?
# [answer...]
#
# DevClaw Queue > what about mobile clients?
# [contextual answer building on previous...]
#
# DevClaw Queue > exit
# Session saved: que-a1b2c3
```

### List sessions

```bash
python run_devclaw.py --list-sessions

# SESSION        CREATED              TURNS  REPOS                    FIRST QUESTION
# que-a1b2c3     2025-03-15 10:23     4      backend-api, frontend    how does auth work...
# que-d4e5f6     2025-03-14 09:11     7      data-pipeline            walk me through...
```

### Three use cases — auto-detected from the question

| Use case | Detected when question mentions | Agent team |
|---|---|---|
| Onboarding | "new developer", "getting started", "walk me through", "overview", "set up" | Explorer + Synthesis |
| Architecture | "architecture", "design", "why is", "how is", "pattern", "structure" | Explorer + Synthesis |
| Debugging | "error", "bug", "failing", "production", "slow", "timeout", "exception" | Explorer + Diagnostic + Synthesis |

Detection is LLM-based — not keyword matching. QueryOrchestratorAgent classifies the question given the full conversation history.

---

## `--run` mode — usage and task inputs

### CLI prompt

```bash
python run_devclaw.py --run \
  --prompt "Add Redis caching to the /search endpoint" \
  --branch feat/redis-caching \
  --repos backend-api
```

### GitHub Issue

```bash
python run_devclaw.py --run --issue 142 --issue-repo your-org/backend-api
```

Issues support YAML front-matter:

```
---
branch_name: feat/my-feature
repos: [backend-api, data-pipeline]
---
Task description here...
```

### YAML task file

```bash
python run_devclaw.py --run --task tasks/add-rate-limiting.yaml
```

```yaml
title: "Add rate limiting"
description: |
  Add Redis-based rate limiting to /api/v1 endpoints.
branch_name: feat/rate-limiting
repos:
  - backend-api
```

### Dry run

```bash
python run_devclaw.py --run --task tasks/add-rate-limiting.yaml --dry-run
# No code pushed, no PRs opened. Prints what would happen.
```

### Dynamic team sizing

`RunOrchestratorAgent` scores complexity 1–5 and assembles the team:

| Score | Scenario | Agents spawned |
|---|---|---|
| 1 | Single repo, < 5 files | Orchestrator + 1 RepoWorker (inline) |
| 2 | Single repo, non-trivial | Orchestrator + Planner + 1 RepoWorker |
| 3 | 2–3 repos, moderate | Orchestrator + Planner + N RepoWorkers + Reviewer |
| 4 | 3+ repos or architectural | Orchestrator + Planner + N RepoWorkers (each with Coder + Tester) + Reviewer |
| 5 | Cross-cutting change | Full swarm, all roles, Reviewer runs twice |

### PR output

- Title: `feat(repo-name): description` (conventional commits)
- Body: summary, per-repo changes, related PRs table, test results, task source, run metadata
- Same feature branch across all repos
- Always targets `default_branch`
- After all PRs open, each body updated with cross-links to the others

---

## Repo allowlist — `config/repos.toml`

Both modes only ever access repos listed here. Remove a repo to lock it out completely.

```toml
[[repos]]
name           = "backend-api"
url            = "https://github.com/your-org/backend-api.git"
default_branch = "main"
build_cmd      = "docker build -t backend-api ."
test_cmd       = "pytest tests/ -v"
language       = "python"

[[repos]]
name           = "data-pipeline"
url            = "https://github.com/your-org/data-pipeline.git"
default_branch = "main"
build_cmd      = ""
test_cmd       = "pytest"
language       = "python"
```

---

## Config — `config/config.toml`

```toml
[devclaw]
git_user_name      = "DevClaw"
git_user_email     = "devclaw@your-company.com"
work_dir           = "/tmp/devclaw"
clone_depth        = 1
build_timeout      = 300
test_timeout       = 300
max_fix_retries    = 3
parallel_workers   = true
max_agents         = 8
cleanup_after_run  = false
dry_run            = false

[devclaw.memory]
enabled        = true
store_path     = "/tmp/devclaw/memory/devclaw.db"
top_k          = 5
embedding_dims = 1536

[devclaw.limits]
max_tokens_per_run    = 500000
max_cost_per_run_usd  = 5.00
max_llm_calls_per_run = 200
warn_at_cost_usd      = 2.00
max_pr_files          = 20
max_pr_lines          = 500

[devclaw.github]
max_retries         = 5
backoff_min_seconds = 4
backoff_max_seconds = 60

[devclaw.security]
secret_scan_enabled     = true
scan_generic_patterns   = true
```

Secrets go in `.env` only:

```bash
DEVCLAW_GITHUB_TOKEN=ghp_your_token_here
DEVCLAW_GIT_USER_NAME=DevClaw
DEVCLAW_GIT_USER_EMAIL=devclaw@your-company.com
DEVCLAW_WORK_DIR=/tmp/devclaw
```

---

## Safety guarantees (--run mode)

- `GitCommitPushTool` raises `DevClawSafetyError` at the **Python level** if `branch_name` matches any `default_branch` in `repos.toml`
- `GitHubPRTool` asserts `branch_name != base_branch` before any API call
- `SecretScanTool` blocks commits containing secret patterns before any push
- Repos not in `repos.toml` raise `DevClawRepoNotAllowedError` at startup
- GitHub token read from environment only — never logged, never in any dataclass field
- Branch names sanitised at input time — invalid git characters replaced before any git operation

---

## What DevClaw reuses from OpenManus

These OpenManus files are **never modified**:

| Component | Used for |
|---|---|
| `BaseAgent` + `ToolCallAgent` | Every DevClaw agent inherits from these |
| `PlanningFlow` | `DevClawFlow` and `QueryFlow` extend this |
| `LLM` abstraction | All agents share this for LLM calls |
| `PythonExecute`, `FileSaver`, `FileOperator` | CoderAgent + DiagnosticAgent tools |
| `ToolCollection`, `BaseTool`, `ToolResult` | All custom tools follow this interface |
| `Terminate` | Every agent's final action |
| OpenManus `logger` | Used throughout — never `print()` |

---

## New dependencies

```
PyGithub>=2.3.0          # GitHub REST API
pyyaml>=6.0              # YAML task files
python-dotenv>=1.0.0     # .env loading
tomli>=2.0.1             # repos.toml (Python < 3.11 only)
tenacity>=8.0.0          # GitHub API retry + backoff
sqlite-vec>=0.1.0        # vector search in SQLite (memory store)
```

---

## Implementation phases

| Phase | What gets built | Modes affected |
|---|---|---|
| 1 — Scaffolding | CLAUDE.md, errors, schema, loaders, git clone/branch, repo reader, memory store schema | Both |
| 2 — `--run` baseline | Build/test, git commit/push, GitHub PR, RepoWorkerAgent inline, preflight, secret scan, rate limiting | `--run` |
| 3 — `--run` specialist agents | PlannerAgent, CoderAgent (Plan-and-Solve), TesterAgent, ReviewAgent (LLM-as-Judge), cost controls, audit log | `--run` |
| 4 — `--run` full swarm | SpawnAgentTool, RunOrchestratorAgent, parallel multi-repo, PR back-linking, idempotency/resume | `--run` |
| 5 — Memory | MemorySyncAgent, MemoryWriterAgent, SHA-based freshness, repo_facts population | Both |
| 6 — `--que` mode | SessionManager, QueryFlow, QueryOrchestratorAgent, RepoExplorerAgent, SynthesisAgent, DiagnosticAgent, ConversationStoreTool | `--que` |
| 7 — Hardening | Issue/YAML loaders, dry-run, error isolation, cleanup, token masking, branch sanitisation, PR size guardrail | Both |

---

## Agent index

For full specifications — system prompts, reasoning patterns, schemas, tool lists — see **`agent_plan.md`**.

### `--run` mode agents

| Agent | Role | Pattern |
|---|---|---|
| RunOrchestratorAgent | Coordinates the swarm, scores complexity, arbitrates conflicts | ReAct |
| PlannerAgent | Reads all repos, outputs per-repo `RepoPlan` JSON | ReAct |
| RepoWorkerAgent | Owns one repo: clone → branch → code → test → commit → PR | ReAct |
| CoderAgent | Implements code changes in one repo | Plan-and-Solve |
| TesterAgent | Runs build + tests, outputs structured pass/fail report | ReAct |
| ReviewAgent | Cross-repo consistency + PR quality scoring | LLM-as-Judge |

### `--que` mode agents

| Agent | Role | Pattern |
|---|---|---|
| QueryOrchestratorAgent | Manages session, classifies question, coordinates query team | ReAct |
| RepoExplorerAgent | Finds relevant code and facts from memory + targeted file reads | ReAct |
| SynthesisAgent | Assembles answer from findings, building on conversation history | ReAct |
| DiagnosticAgent | Traces execution paths for debugging use case | ReAct |

### Shared agents (both modes)

| Agent | Role | Pattern |
|---|---|---|
| MemorySyncAgent | SHA check at run start — incremental update of repo_facts | ReAct |
| MemoryWriterAgent | Writes new facts + run log after task completes | ReAct |

---

## License

Apache 2.0 — chosen for explicit patent grant and enterprise compatibility.
Compatible with OpenManus (MIT). No license conflict.
