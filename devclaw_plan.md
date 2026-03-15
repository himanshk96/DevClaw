# DevClaw — Product Plan

> **DevClaw** is an AI-powered multi-agent engineering swarm built on OpenManus.
> It accepts tasks from a CLI prompt, GitHub Issue, or YAML file — assembles a
> dynamic team of specialist agents, fans them out across repos in parallel,
> validates with pytest and Docker, and opens linked PRs.
> Coordinated through Claude Code's native multi-agent Task tool.
> **Never touches `main` or any default branch. Ever.**

---

## What DevClaw does

1. **Receives a task** — from a CLI prompt, a GitHub Issue, or a YAML file
2. **Scores complexity** — 1 to 5, determines how many agents to assemble
3. **Plans** — PlannerAgent reads every repo and produces a per-repo implementation plan
4. **Executes in parallel** — one RepoWorkerAgent per repo, all running simultaneously
5. **Validates** — builds with Docker, runs pytest, retries on failure
6. **Reviews** — ReviewAgent checks cross-repo consistency and PR quality using LLM-as-Judge
7. **Opens linked PRs** — one PR per repo, same branch name, each linking to the others

---

## Technology foundation

| Layer | Technology |
|---|---|
| Agent framework | OpenManus (MIT) |
| Multi-agent coordination | Claude Code native Task tool |
| LLM | Configurable — GPT-4o, Claude, etc. via OpenManus LLM abstraction |
| GitHub integration | PyGithub |
| Build validation | Docker + pytest via subprocess |
| Config | TOML (repos.toml + config.toml) |
| Secrets | python-dotenv (.env file, never committed) |
| License | Apache 2.0 |

---

## Repository structure

```
OpenManus/                          ← base framework (do not modify internals)
├── CLAUDE.md                       ← Claude Code project guide (create first)
├── .env.example                    ← secrets template
├── run_devclaw.py                  ← CLI entry point
├── config/
│   ├── config.toml                 ← add [devclaw] section
│   └── repos.toml                  ← your repo allowlist
├── tasks/
│   └── example-task.yaml
├── app/
│   ├── devclaw_errors.py           ← exception hierarchy
│   ├── agent/
│   │   ├── devclaw_orchestrator.py
│   │   ├── devclaw_planner.py
│   │   ├── devclaw_repo_worker.py
│   │   ├── devclaw_coder.py        ← Plan-and-Solve pattern
│   │   ├── devclaw_tester.py
│   │   └── devclaw_reviewer.py     ← LLM-as-Judge pattern
│   ├── flow/
│   │   └── devclaw_flow.py
│   ├── task_input/
│   │   ├── schema.py
│   │   └── loader.py
│   └── tool/
│       ├── git_tools.py
│       ├── github_pr_tool.py
│       ├── build_test_tool.py
│       ├── repo_reader_tool.py
│       └── spawn_agent_tool.py
└── tests/
    └── devclaw/
        ├── test_safety.py
        └── test_task_input.py
```

---

## Task inputs

### CLI prompt
```bash
python run_devclaw.py \
  --prompt "Add Redis caching to the /search endpoint" \
  --branch feat/redis-caching \
  --repos backend-api
```

### GitHub Issue
```bash
python run_devclaw.py --issue 142 --issue-repo your-org/backend-api
```
Issues support YAML front-matter to specify branch and repos:
```
---
branch_name: feat/my-feature
repos: [backend-api, data-pipeline]
---
Task description here...
```

### YAML task file
```bash
python run_devclaw.py --task tasks/add-rate-limiting.yaml
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
python run_devclaw.py --task tasks/add-rate-limiting.yaml --dry-run
# No code pushed, no PRs opened. Prints what would happen.
```

---

## Repo allowlist — `config/repos.toml`

DevClaw will **only ever touch repos listed here**. Remove a repo to lock it out.

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

## Config — `config/config.toml` additions

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
```

Secrets go in `.env` only — never in `config.toml`:
```bash
DEVCLAW_GITHUB_TOKEN=ghp_your_token_here
DEVCLAW_GIT_USER_NAME=DevClaw
DEVCLAW_GIT_USER_EMAIL=devclaw@your-company.com
DEVCLAW_WORK_DIR=/tmp/devclaw
```

---

## Dynamic team sizing

OrchestratorAgent scores complexity 1–5 and assembles the appropriate team:

| Score | Scenario | Agents spawned |
|---|---|---|
| 1 | Single repo, < 5 files | Orchestrator + 1 RepoWorker (inline) |
| 2 | Single repo, non-trivial | Orchestrator + Planner + 1 RepoWorker |
| 3 | 2–3 repos, moderate | Orchestrator + Planner + N RepoWorkers + Reviewer |
| 4 | 3+ repos or architectural | Orchestrator + Planner + N RepoWorkers (each with Coder + Tester) + Reviewer |
| 5 | Cross-cutting change | Full swarm, all roles, Reviewer runs twice |

---

## PR output

Every PR DevClaw opens:
- Title in conventional commits format: `feat(repo-name): description`
- Body with summary, per-repo changes, related PRs table, test results, task source, run metadata
- Same feature branch name across all repos
- Always targets `default_branch` — never any other branch
- After all PRs are open, each body is updated with cross-links to the others

---

## Safety guarantees

- `GitCommitPushTool` raises `DevClawSafetyError` at the **Python level** if `branch_name` matches any `default_branch` in `repos.toml`. This is enforced in code, not in a prompt.
- `GitHubPRTool` asserts `branch_name != base_branch` before any API call.
- Repos not in `repos.toml` raise `DevClawRepoNotAllowedError` at startup — not mid-run.
- GitHub token is read from environment only — never logged, never in any dataclass field.

---

## What DevClaw reuses from OpenManus

These OpenManus files are **never modified**:

| Component | Used for |
|---|---|
| `BaseAgent` + `ToolCallAgent` | Every DevClaw agent inherits from these |
| `PlanningFlow` | `DevClawFlow` extends this |
| `LLM` abstraction | All agents share this for LLM calls |
| `PythonExecute`, `FileSaver`, `FileOperator` | CoderAgent's implementation tools |
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
```

---

## Implementation phases

| Phase | What gets built | Test gate |
|---|---|---|
| 1 — Scaffolding | CLAUDE.md, errors, schema, loaders, git clone/branch, repo reader | Clone a repo and print summary — no LLM |
| 2 — Single agent | Build/test, git commit/push, GitHub PR, RepoWorkerAgent inline | Open a real PR on a test repo |
| 3 — Specialist agents | PlannerAgent, CoderAgent (Plan-and-Solve), TesterAgent, ReviewAgent (LLM-as-Judge) | Multi-agent run, each agent logged |
| 4 — Full swarm | SpawnAgentTool, OrchestratorAgent, parallel multi-repo, PR back-linking | 2+ repos open linked PRs, same branch |
| 5 — Hardening | Issue/YAML loaders, dry-run, error isolation, cleanup, token masking | All tests pass, all input modes work |

---

## Agent index

For full specifications — system prompts, reasoning patterns, input/output schemas,
tool lists, and implementation notes — see **`agent_plan.md`**.

| Agent | Role | Reasoning pattern |
|---|---|---|
| OrchestratorAgent | Coordinates the swarm, scores complexity, arbitrates conflicts | ReAct |
| PlannerAgent | Reads all repos, outputs per-repo `RepoPlan` JSON | ReAct |
| RepoWorkerAgent | Owns one repo: clone → branch → code → test → commit → PR | ReAct |
| CoderAgent | Implements code changes in one repo | **Plan-and-Solve** |
| TesterAgent | Runs build + tests, outputs structured pass/fail report | ReAct |
| ReviewAgent | Cross-repo consistency + PR quality scoring | **LLM-as-Judge** |

---

## License

Apache 2.0 — chosen for explicit patent grant and enterprise compatibility.
Compatible with OpenManus (MIT). No license conflict.
