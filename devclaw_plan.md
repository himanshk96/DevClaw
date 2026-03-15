# DevClaw — Final Claude Code Input

> **DevClaw** is an AI-driven, repo-safe engineering orchestration swarm built on OpenManus.
> It turns task inputs (CLI prompt, GitHub issue, or YAML) into an automated multi-agent plan,
> executes code changes in parallel across repositories, validates them with tests and Docker,
> and opens linked PRs while enforcing branch safety.
> 
> Key guarantees:
> 1) Works in cloned branches only (never `main`/default branches),
> 2) Uses strict repo allowlists and safety checks,
> 3) Outputs structured results and explicit terminal status messages.

---

## Instructions for Claude Code

You are building DevClaw inside an existing OpenManus repo clone.

**Read this entire document before writing a single line of code.**

Work through the phases in order. Do not skip ahead. After each phase, run
the listed test gate command and confirm it passes before starting the next phase.

If you are unsure about any OpenManus internals, read the relevant source file
before writing code that depends on it. The files that matter most are:
- `app/agent/base.py` — BaseAgent, AgentState, run loop
- `app/agent/toolcall.py` — ToolCallAgent, think(), act()
- `app/tool/base.py` — BaseTool, ToolResult, ToolCollection
- `app/flow/planning.py` — PlanningFlow, create_initial_plan()
- `app/llm/llm.py` — LLM client, ask(), ask_tool()

---

## Step 0 — Create `CLAUDE.md` first

Before any other file, create `CLAUDE.md` in the repo root. Claude Code reads
this file on every invocation to understand project conventions.

**File: `CLAUDE.md`**

```markdown
# DevClaw — Project Guide for Claude Code

## What this project is
DevClaw is a multi-agent swarm built on OpenManus that clones GitHub repos,
implements tasks using specialist AI agents, runs tests, and opens linked PRs.

## What NOT to touch
- Do NOT modify any file under `app/agent/base.py`, `app/agent/toolcall.py`,
  `app/flow/planning.py`, `app/llm/`, or `app/schema/`.
- Do NOT modify `main.py`, `run_flow.py`, `run_mcp.py` — these are OpenManus entry points.
- Do NOT commit secrets. The GitHub token lives in `.env` only.

## Environment setup
cp .env.example .env        # then fill in your GitHub token
pip install -r requirements.txt

## How to run DevClaw
python run_devclaw.py --prompt "your task" --branch feat/branch-name --repos repo-name
python run_devclaw.py --issue 42 --issue-repo your-org/your-repo
python run_devclaw.py --task tasks/example-task.yaml
python run_devclaw.py --task tasks/example-task.yaml --dry-run

## Running tests
pytest tests/devclaw/ -v

## Code conventions
- All new DevClaw files live under app/agent/devclaw_*.py, app/tool/ (new tools),
  app/flow/devclaw_flow.py, app/task_input/
- All agents inherit from ToolCallAgent (app/agent/toolcall.py)
- All tools inherit from BaseTool (app/tool/base.py) and return ToolResult
- All agents must call Terminate() as their final action
- Commit messages: conventional commits format — feat(scope): description
- Every agent outputs structured JSON as its final message before Terminate

## Branch safety rule
GitCommitPushTool raises DevClawSafetyError at the Python level if
branch_name matches any default_branch in repos.toml. This check exists in
Python code — it is NOT just a prompt instruction and cannot be bypassed by any LLM.

## Secrets management
- GitHub token: DEVCLAW_GITHUB_TOKEN env var (loaded from .env)
- Never log, print, or include the token in any string output
- Mask it by replacing with <REDACTED> if it appears in subprocess output
```

---

## Step 0b — Create `.env.example`

**File: `.env.example`**

```bash
# Copy this to .env and fill in your values.
# .env is in .gitignore — never commit it.

# GitHub Personal Access Token
# Required scopes: repo, pull_requests
# Create at: https://github.com/settings/tokens
DEVCLAW_GITHUB_TOKEN=ghp_your_token_here

# Git identity used for commits made by DevClaw
DEVCLAW_GIT_USER_NAME=DevClaw
DEVCLAW_GIT_USER_EMAIL=devclaw@your-company.com

# Working directory where repos are cloned
DEVCLAW_WORK_DIR=/tmp/devclaw
```

Add `.env` to `.gitignore` if not already present.

---

## Step 0c — Exception hierarchy

Create this file before any tool or agent. Every component imports from it.

**File: `app/devclaw_errors.py`**

```python
class DevClawError(Exception):
    """Base exception for all DevClaw errors."""

class DevClawSafetyError(DevClawError):
    """
    Raised when an operation would violate a safety rule.
    Primary use: GitCommitPushTool raises this if branch_name == default_branch.
    This exception is intentionally NOT caught inside tools — it propagates
    up to DevClawFlow which logs it and halts the affected repo's work.
    """

class DevClawConfigError(DevClawError):
    """Raised when repos.toml or config.toml is missing or malformed."""

class DevClawGitError(DevClawError):
    """Raised when a git subprocess command fails with a non-zero exit code."""

class DevClawBuildError(DevClawError):
    """Raised when docker build or pytest exits non-zero after all retries."""

class DevClawRepoNotAllowedError(DevClawSafetyError):
    """Raised when a requested repo is not in repos.toml."""
```

---

## File & folder structure (complete)

```
OpenManus/
├── CLAUDE.md                              # CREATE — Claude Code project guide
├── .env.example                           # CREATE — secrets template
├── .env                                   # NOT committed — gitignored
├── run_devclaw.py                         # CREATE — CLI entry point
├── config/
│   ├── config.toml                        # EDIT — add [devclaw] section
│   └── repos.toml                         # CREATE — repo allowlist
├── tasks/                                 # CREATE directory
│   └── example-task.yaml                  # CREATE — example task file
├── app/
│   ├── devclaw_errors.py                  # CREATE — exception hierarchy
│   ├── agent/
│   │   ├── devclaw_orchestrator.py        # CREATE
│   │   ├── devclaw_planner.py             # CREATE
│   │   ├── devclaw_repo_worker.py         # CREATE
│   │   ├── devclaw_coder.py               # CREATE
│   │   ├── devclaw_tester.py              # CREATE
│   │   └── devclaw_reviewer.py            # CREATE
│   ├── flow/
│   │   └── devclaw_flow.py                # CREATE
│   ├── task_input/
│   │   ├── __init__.py                    # CREATE
│   │   ├── schema.py                      # CREATE
│   │   └── loader.py                      # CREATE
│   └── tool/
│       ├── git_tools.py                   # CREATE
│       ├── github_pr_tool.py              # CREATE
│       ├── build_test_tool.py             # CREATE
│       ├── repo_reader_tool.py            # CREATE
│       └── spawn_agent_tool.py            # CREATE
└── tests/
    └── devclaw/
        ├── __init__.py                    # CREATE
        ├── test_git_tools.py              # CREATE
        ├── test_build_test_tool.py        # CREATE
        ├── test_repo_reader_tool.py       # CREATE
        ├── test_task_input.py             # CREATE
        └── test_safety.py                 # CREATE — safety rules must be tested

Do NOT modify any existing OpenManus files except:
  - config/config.toml  (add [devclaw] section)
  - requirements.txt    (add new deps at bottom)
  - .gitignore          (add .env if missing)
```

---

## What to reuse from OpenManus — do not rewrite

| Component | File | How DevClaw uses it |
|---|---|---|
| `BaseAgent` | `app/agent/base.py` | All DevClaw agents inherit `ToolCallAgent` which inherits this |
| `ToolCallAgent` | `app/agent/toolcall.py` | Every DevClaw agent subclasses this directly |
| `PlanningFlow` | `app/flow/planning.py` | `DevClawFlow` extends this |
| `FlowFactory` | `app/flow/base.py` | Unchanged — used to instantiate DevClawFlow |
| `LLM` | `app/llm/llm.py` | All agents use this via `self.llm` inherited from BaseAgent |
| `PythonExecute` | `app/tool/python_execute.py` | Imported and added to CoderAgent and RepoWorkerAgent tool lists |
| `FileSaver` + `FileOperator` | `app/tool/file_operators.py` | CoderAgent uses FileSaver to write files and FileOperator to read them |
| `ToolCollection` | `app/tool/base.py` | All custom tool lists use `ToolCollection([...])` |
| `BaseTool`, `ToolResult` | `app/tool/base.py` | Every custom tool inherits BaseTool and returns ToolResult |
| `Config` singleton | `app/config/config.py` | Extended with `[devclaw]` — accessed via `config.devclaw.*` |
| `Memory`, `Message` | `app/schema/` | Used by ToolCallAgent automatically — no changes needed |
| `Terminate` | `app/tool/terminate.py` | Every agent's final action — imported and added to all tool lists |
| OpenManus `logger` | `app/logger.py` | Use `from app.logger import logger` throughout — never use `print()` |

**File reading in CoderAgent:** Use `FileOperator` (already in OpenManus) to read
existing source files. `FileSaver` is write-only. For reading: check
`app/tool/file_operators.py` for the exact class name and method signatures before
writing the CoderAgent tool list.

---

## Config additions

### `config/config.toml` — add this section

```toml
[devclaw]
# Loaded from .env — do not hardcode token here
# github_token is read from DEVCLAW_GITHUB_TOKEN env var in code
git_user_name      = "DevClaw"
git_user_email     = "devclaw@your-company.com"
work_dir           = "/tmp/devclaw"
clone_depth        = 1
build_timeout      = 300
test_timeout       = 300
max_fix_retries    = 3
parallel_workers   = true
max_agents         = 8
cleanup_after_run  = false   # set true in production to delete cloned repos after PR
dry_run            = false   # overridden by --dry-run CLI flag
```

### `config/repos.toml` — repo allowlist

```toml
# DevClaw will ONLY ever clone or modify repos listed here.
# Removing a repo from this list fully locks it out.

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

[[repos]]
name           = "infra"
url            = "https://github.com/your-org/infra.git"
default_branch = "main"
build_cmd      = "docker compose build"
test_cmd       = ""
language       = "terraform"
```

### `tasks/example-task.yaml`

```yaml
title: "Add Redis caching to search endpoint"
description: |
  The /api/v1/search endpoint currently hits the database on every call.
  Add Redis-based caching with a 5-minute TTL.
  The cache key should be a hash of the query parameters.
  Add a cache-busting endpoint at DELETE /api/v1/search/cache.
  Update integration tests to mock Redis.
branch_name: "feat/redis-search-cache"
repos:
  - backend-api
```

---

## New dependencies — add to `requirements.txt`

```
# DevClaw dependencies
PyGithub>=2.3.0
pyyaml>=6.0
python-dotenv>=1.0.0
tomli>=2.0.1; python_version < "3.11"
```

---

## Component specifications

---

### `app/task_input/schema.py`

```python
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class AgentTask:
    title: str
    description: str
    branch_name: str          # same branch name used across ALL repos
    repos: list[str]          # empty = all repos in repos.toml
    source: str               # "prompt" | "github_issue" | "config_file"
    issue_url: Optional[str] = None
    issue_number: Optional[int] = None
    extra_context: dict = field(default_factory=dict)

@dataclass
class RepoConfig:
    name: str
    url: str
    default_branch: str
    build_cmd: str
    test_cmd: str
    language: str

@dataclass
class RepoPlan:
    """Output of PlannerAgent — one per repo."""
    repo_name: str
    summary_of_changes: str   # what this repo needs to implement
    files_to_modify: list[str]
    dependencies_on: list[str]
    estimated_complexity: int  # 1-5

@dataclass
class RepoResult:
    """Output of RepoWorkerAgent — one per repo."""
    repo_name: str
    pr_url: Optional[str]
    pr_number: Optional[int]
    branch_name: str
    build_passed: bool
    test_passed: bool
    commit_sha: Optional[str]
    error: Optional[str]      # set if the worker failed entirely

@dataclass
class ReviewReport:
    """Output of ReviewAgent."""
    approved: bool
    issues: list[dict]        # [{"repo": str, "severity": "warning"|"error", "description": str}]
    suggested_pr_title_overrides: dict  # {"repo_name": "new title"}
    overall_summary: str
```

---

### `app/task_input/loader.py`

```python
import os, re, yaml
from github import Github
from app.task_input.schema import AgentTask
from app.devclaw_errors import DevClawConfigError

def load_from_prompt(prompt: str, branch_name: str, repos: list[str]) -> AgentTask:
    """Wrap a raw CLI prompt into an AgentTask."""
    return AgentTask(
        title=prompt[:80],           # first 80 chars as title
        description=prompt,
        branch_name=branch_name,
        repos=repos,
        source="prompt",
    )

def load_from_github_issue(repo_full_name: str, issue_number: int) -> AgentTask:
    """
    Fetch a GitHub Issue and convert it to an AgentTask.

    Looks for YAML front-matter in the issue body between --- markers:
      ---
      branch_name: feat/my-feature
      repos: [backend-api, data-pipeline]
      ---
    Falls back to slugified title as branch_name if front-matter absent.
    Appends issue comments to extra_context["comments"].
    """
    token = os.environ["DEVCLAW_GITHUB_TOKEN"]
    g = Github(token)
    repo = g.get_repo(repo_full_name)
    issue = repo.get_issue(issue_number)

    # Parse YAML front-matter
    front_matter = {}
    body = issue.body or ""
    fm_match = re.match(r"^---\n(.*?)\n---\n", body, re.DOTALL)
    if fm_match:
        front_matter = yaml.safe_load(fm_match.group(1)) or {}
        body = body[fm_match.end():]

    branch_name = front_matter.get(
        "branch_name",
        "feat/" + re.sub(r"[^a-z0-9]+", "-", issue.title.lower()).strip("-")
    )
    repos = front_matter.get("repos", [])
    comments = [c.body for c in issue.get_comments()]

    return AgentTask(
        title=issue.title,
        description=body.strip(),
        branch_name=branch_name,
        repos=repos,
        source="github_issue",
        issue_url=issue.html_url,
        issue_number=issue_number,
        extra_context={"comments": comments, "labels": [l.name for l in issue.labels]},
    )

def load_from_config_file(path: str) -> AgentTask:
    """Load a YAML task file and convert it to an AgentTask."""
    try:
        with open(path) as f:
            data = yaml.safe_load(f)
    except FileNotFoundError:
        raise DevClawConfigError(f"Task file not found: {path}")
    except yaml.YAMLError as e:
        raise DevClawConfigError(f"Invalid YAML in task file {path}: {e}")

    required = ["title", "description", "branch_name"]
    missing = [k for k in required if k not in data]
    if missing:
        raise DevClawConfigError(f"Task file missing required keys: {missing}")

    return AgentTask(
        title=data["title"],
        description=data["description"],
        branch_name=data["branch_name"],
        repos=data.get("repos", []),
        source="config_file",
        extra_context=data.get("extra_context", {}),
    )

def load_repos_toml(path: str = "config/repos.toml") -> list:
    """Load and validate repos.toml. Returns list of RepoConfig."""
    import sys
    if sys.version_info >= (3, 11):
        import tomllib
        opener = lambda p: open(p, "rb")
        loader = tomllib.load
    else:
        import tomli as tomllib
        opener = lambda p: open(p, "rb")
        loader = tomllib.load

    try:
        with opener(path) as f:
            data = loader(f)
    except FileNotFoundError:
        raise DevClawConfigError(f"repos.toml not found at {path}")

    from app.task_input.schema import RepoConfig
    repos = []
    for r in data.get("repos", []):
        repos.append(RepoConfig(
            name=r["name"],
            url=r["url"],
            default_branch=r.get("default_branch", "main"),
            build_cmd=r.get("build_cmd", ""),
            test_cmd=r.get("test_cmd", ""),
            language=r.get("language", "python"),
        ))
    if not repos:
        raise DevClawConfigError("repos.toml has no [[repos]] entries")
    return repos
```

---

### `app/tool/git_tools.py`

All tools use `subprocess.run(capture_output=True, text=True)`.
Import `DevClawSafetyError` and `DevClawGitError` from `app.devclaw_errors`.
Import `load_repos_toml` to check default branches in `GitCommitPushTool`.
Read `DEVCLAW_GITHUB_TOKEN` from `os.environ` — never from a parameter.
Mask the token in all output strings using a helper:

```python
def _mask_token(s: str) -> str:
    token = os.environ.get("DEVCLAW_GITHUB_TOKEN", "")
    return s.replace(token, "<REDACTED>") if token else s
```

**GitCloneTool**
```
name: "git_clone"
description: "Clone a GitHub repo to a local directory"
parameters:
  repo_url: str       — HTTPS URL (token injected internally)
  target_dir: str     — absolute path to clone into
action:
  1. Build authenticated URL: https://<token>@github.com/org/repo.git
     Extract org/repo from repo_url. Handle both https:// and git@github.com: formats.
  2. subprocess: git clone --depth=<clone_depth> <auth_url> <target_dir>
  3. subprocess: git -C <target_dir> config user.name  "<DEVCLAW_GIT_USER_NAME>"
  4. subprocess: git -C <target_dir> config user.email "<DEVCLAW_GIT_USER_EMAIL>"
  5. Return ToolResult(output=target_dir)
  6. On non-zero exit: raise DevClawGitError with masked stderr
```

**GitBranchTool**
```
name: "git_branch"
description: "Create or checkout a feature branch"
parameters:
  repo_dir: str
  branch_name: str
  base_branch: str
action:
  1. SAFETY: if branch_name == base_branch raise DevClawSafetyError
  2. subprocess: git -C <repo_dir> fetch origin
  3. subprocess: git -C <repo_dir> checkout <base_branch>
  4. subprocess: git -C <repo_dir> pull origin <base_branch>
  5. Check if branch exists remotely:
       git -C <repo_dir> ls-remote --heads origin <branch_name>
     If exists: git checkout <branch_name> && git pull origin <branch_name>
                return ToolResult(output="resumed")
     If not:    git checkout -b <branch_name>
                return ToolResult(output="created")
  6. On non-zero exit: raise DevClawGitError
```

**GitCommitPushTool**
```
name: "git_commit_push"
description: "Stage all changes, commit, and push to feature branch"
parameters:
  repo_dir: str
  commit_message: str
  branch_name: str
action:
  1. HARD SAFETY CHECK (Python level, not LLM prompt):
       repos = load_repos_toml()
       default_branches = {r.default_branch for r in repos}
       if branch_name in default_branches:
           raise DevClawSafetyError(
               f"REFUSED: DevClaw will never push to a default branch. "
               f"Attempted branch: '{branch_name}'"
           )
  2. If config.devclaw.dry_run is True:
       return ToolResult(output="DRY RUN — commit skipped")
  3. subprocess: git -C <repo_dir> add -A
  4. subprocess: git -C <repo_dir> commit -m "<commit_message>"
     If exit code 1 and "nothing to commit" in output:
       return ToolResult(output="nothing_to_commit")
  5. subprocess: git -C <repo_dir> push origin <branch_name>
  6. Get commit SHA: git -C <repo_dir> rev-parse HEAD
  7. Get changed files: git -C <repo_dir> diff --name-only HEAD~1 HEAD
  8. Return ToolResult(output=json.dumps({
         "commit_sha": sha, "branch": branch_name, "files_changed": files
     }))
```

**GitStatusTool**
```
name: "git_status"
description: "Show current git status, diff summary, and recent commits"
parameters:
  repo_dir: str
action:
  1. subprocess: git -C <repo_dir> status --short
  2. subprocess: git -C <repo_dir> diff --stat HEAD
  3. subprocess: git -C <repo_dir> log --oneline -5
  4. Return ToolResult(output=combined formatted output)
```

---

### `app/tool/github_pr_tool.py`

```
GitHubPRTool
name: "github_create_pr"
description: "Open a pull request on GitHub"
parameters:
  repo_name: str           — short name matching repos.toml (e.g. "backend-api")
  branch_name: str
  title: str
  body: str
  base_branch: str
  linked_pr_urls: list[str]   — cross-repo PR URLs to append (can be empty)
action:
  1. SAFETY: assert branch_name != base_branch
  2. If config.devclaw.dry_run:
       logger.info(f"DRY RUN — would open PR: {title}")
       return ToolResult(output=json.dumps({"pr_url": "DRY_RUN", "pr_number": 0}))
  3. token = os.environ["DEVCLAW_GITHUB_TOKEN"]
     g = Github(token)
     Look up full repo path by matching repo_name against repos.toml url field.
     repo = g.get_repo("org/repo_name")
  4. If linked_pr_urls non-empty, append to body:
       \n\n## Related PRs\n + bullet list of linked_pr_urls
  5. pr = repo.create_pull(title=title, body=body, base=base_branch, head=branch_name)
  6. return ToolResult(output=json.dumps({"pr_url": pr.html_url, "pr_number": pr.number}))

GitHubUpdatePRTool
name: "github_update_pr"
description: "Update the body of an existing pull request"
parameters:
  repo_name: str
  pr_number: int
  new_body: str
action:
  1. If dry_run: log and return immediately
  2. repo.get_pull(pr_number).edit(body=new_body)
  3. return ToolResult(output="updated")
```

---

### `app/tool/repo_reader_tool.py`

```
RepoReaderTool
name: "repo_reader"
description: "Read and summarise a cloned repo's structure and key files for LLM context"
parameters:
  repo_dir: str
  max_chars: int = 60000
action:
  Priority order — include files until max_chars is reached:
    1. README.md or README.rst (full content)
    2. pyproject.toml, setup.py, setup.cfg, requirements*.txt
    3. Dockerfile, docker-compose*.yml
    4. Directory tree — 3 levels deep, skip: .git, __pycache__, node_modules,
       *.pyc, .pytest_cache, dist, build, .venv, venv
    5. All __init__.py files (with file paths as headers)
    6. All *.py files in the top-level package directory

  Format each file as:
    === path/to/file.py ===
    <content>

  After all files, append:
    === DIRECTORY TREE ===
    <tree output>

  Truncate to max_chars total. Add a note at the end if truncated:
    "[TRUNCATED — {total_chars} chars total, showing first {max_chars}]"

output: ToolResult(output=repo_summary_string)
```

---

### `app/tool/build_test_tool.py`

```
BuildTestTool
name: "build_and_test"
description: "Run docker build and/or pytest in a repo directory"
parameters:
  repo_dir: str
  build_cmd: str    — empty string to skip
  test_cmd: str     — empty string to skip
  timeout: int = 300
action:
  results = {build_passed: None, build_output: "", test_passed: None, test_output: ""}

  If build_cmd non-empty:
    proc = subprocess.run(build_cmd, cwd=repo_dir, shell=True,
                          capture_output=True, text=True, timeout=timeout)
    results["build_passed"] = (proc.returncode == 0)
    results["build_output"] = (proc.stdout + proc.stderr)[-4000:]

  If test_cmd non-empty:
    proc = subprocess.run(test_cmd, cwd=repo_dir, shell=True,
                          capture_output=True, text=True, timeout=timeout)
    results["test_passed"] = (proc.returncode == 0)
    results["test_output"] = (proc.stdout + proc.stderr)[-4000:]

  results["overall_passed"] = all(
      v for v in [results["build_passed"], results["test_passed"]] if v is not None
  )

  return ToolResult(output=json.dumps(results))

  On subprocess.TimeoutExpired:
    return ToolResult(output=json.dumps({...all False, "error": "timeout after {timeout}s"}))
```

---

### `app/tool/spawn_agent_tool.py` — the swarm enabler

This is how DevClaw integrates with Claude Code's native multi-agent system.
Claude Code exposes a `Task` tool to sub-agents it spawns. When OrchestratorAgent
(itself a Claude Code sub-agent) calls `SpawnAgentTool`, this tool formats a
`Task` tool call that Claude Code executes, spawning a new Claude Code sub-agent
with the given prompt.

```python
from app.tool.base import BaseTool, ToolResult
from app.agent.devclaw_planner import PlannerAgent
from app.agent.devclaw_repo_worker import RepoWorkerAgent
from app.agent.devclaw_coder import CoderAgent
from app.agent.devclaw_tester import TesterAgent
from app.agent.devclaw_reviewer import ReviewerAgent

AGENT_CLASSES = {
    "planner":     PlannerAgent,
    "repo_worker": RepoWorkerAgent,
    "coder":       CoderAgent,
    "tester":      TesterAgent,
    "reviewer":    ReviewerAgent,
}

class SpawnAgentTool(BaseTool):
    """
    Spawns a DevClaw specialist agent as a Claude Code sub-task.

    Claude Code's multi-agent system works as follows:
    - Claude Code itself is the orchestrating agent
    - It can spawn sub-agents by issuing Task tool calls
    - Each sub-agent runs as an independent Claude Code instance
    - Sub-agents have access to tools defined in their agent class
    - Results are returned to the spawning agent when the sub-agent calls Terminate

    This tool instantiates the appropriate DevClaw agent class and runs it
    with the provided prompt. In production with Claude Code, this naturally
    becomes a parallel Task invocation. In local/test mode, it runs sequentially.

    Parameters:
        agent_role: "planner" | "repo_worker" | "coder" | "tester" | "reviewer"
        agent_prompt: full context + instructions for this agent instance
        repo_name: repo this agent works on ("all" for planner/reviewer)
        task_id: caller-assigned ID for correlating results

    Returns ToolResult with JSON:
        {"task_id": str, "agent_role": str, "result": <agent final output>}
    """
    name = "spawn_agent"
    description = "Spawn a specialist DevClaw sub-agent for a specific role and repo"

    async def execute(self, agent_role: str, agent_prompt: str,
                      repo_name: str, task_id: str) -> ToolResult:
        if agent_role not in AGENT_CLASSES:
            return ToolResult(error=f"Unknown agent role: {agent_role}")

        agent_class = AGENT_CLASSES[agent_role]
        agent = agent_class()
        result = await agent.run(agent_prompt)

        return ToolResult(output=json.dumps({
            "task_id": task_id,
            "agent_role": agent_role,
            "repo_name": repo_name,
            "result": result,
        }))
```

---

### Agent definitions

All agents inherit from `ToolCallAgent`. Import path: `from app.agent.toolcall import ToolCallAgent`.
All agents must have `Terminate()` as the last tool in their `available_tools` list.

---

#### `app/agent/devclaw_orchestrator.py`

```python
class OrchestratorAgent(ToolCallAgent):
    name: str = "DevClaw-Orchestrator"
    description: str = "Top-level swarm coordinator for DevClaw"
    max_steps: int = 60

    system_prompt: str = """
You are the Orchestrator of DevClaw, an AI software engineering swarm.
Your job: coordinate specialist agents to implement a task across GitHub repos and open PRs.

## Step-by-step workflow

### Step 1 — Score complexity
Read the task description and repo list. Output a JSON complexity assessment:
{"score": 1-5, "justification": str, "repos_affected": [str]}

Scoring guide:
- 1: single repo, < 5 files, no new dependencies
- 2: single repo, 5+ files or new dependencies
- 3: 2-3 repos, moderate changes
- 4: 3+ repos, or any repo with architectural changes
- 5: cross-cutting change affecting all repos or core abstractions

### Step 2 — Assemble team using SpawnAgentTool
Based on score:
- Score 1: spawn 1 repo_worker (it handles coder+tester inline)
- Score 2: spawn planner first, then 1 repo_worker
- Score 3: spawn planner, then N repo_workers in parallel, then reviewer
- Score 4-5: spawn planner, then N repo_workers (each spawns coder+tester), then reviewer

Always wait for planner to finish before spawning repo_workers.
Spawn all repo_workers in parallel (call SpawnAgentTool N times without waiting between calls).

### Step 3 — Collect results
Parse each RepoWorker's JSON output:
{"repo_name", "pr_url", "pr_number", "build_passed", "test_passed", "commit_sha", "error"}

If a worker returned an error:
- Build/test failure AND retries_remaining > 0: spawn a coder to fix, then re-run tester
- Git/infra failure: log it, continue with other repos, include in final summary

### Step 4 — Review (score >= 3)
Spawn ReviewAgent with all RepoResults + original task.
Parse ReviewReport JSON.
For each "error" severity issue: decide whether to fix or note as known issue.

### Step 5 — Back-fill PR links
After all PRs are open, call GitHubUpdatePRTool on each PR to add cross-repo links.

### Step 6 — Final summary
Output a human-readable summary:
- Task: <title>
- Branch: <branch_name>
- Complexity: <score>/5
- Agents used: <list>
- Results per repo:
  - <repo>: PR <url> | build ✅/❌ | tests ✅/❌
- Review: approved/flagged
- Any errors or warnings

Then call Terminate.

## Arbitration rule
YOU make all final decisions.
TesterAgent output is the source of truth for test results.
Never accept CoderAgent's claim that tests pass — only TesterAgent's JSON report.

## Hard safety rules
- Never instruct any agent to push to a default branch.
- If DevClawSafetyError appears in any result, halt that repo, report it, continue others.
- Always verify branch_name != default_branch before spawning any repo_worker.
"""

    available_tools: ToolCollection = ToolCollection([
        SpawnAgentTool(),
        GitHubUpdatePRTool(),
        Terminate(),
    ])
```

---

#### `app/agent/devclaw_planner.py`

```python
class PlannerAgent(ToolCallAgent):
    name: str = "DevClaw-Planner"
    max_steps: int = 20

    system_prompt: str = """
You are the Planner agent for DevClaw. You decompose a task into per-repo implementation plans.

## Workflow
1. Call repo_reader on EACH repo in the provided list.
2. For each repo, output a RepoPlan JSON object:
   {
     "repo_name": str,
     "summary_of_changes": str,
     "files_to_modify": [str],      // specific file paths, not globs
     "dependencies_on": [str],      // other repo names whose changes affect this one
     "estimated_complexity": int    // 1-5
   }
3. Output ONLY a JSON array of RepoPlan objects. No other text.
4. Call Terminate.

## Rules
- Read every repo before writing any plan.
- If a repo needs no changes: include it with summary_of_changes: "no changes required".
- Be specific about file paths — CoderAgent uses your list as its starting point.
- Flag cross-repo dependencies explicitly (e.g. backend adds endpoint, frontend must call it).
"""
    available_tools: ToolCollection = ToolCollection([
        RepoReaderTool(), Terminate(),
    ])
```

---

#### `app/agent/devclaw_repo_worker.py`

```python
class RepoWorkerAgent(ToolCallAgent):
    name: str = "DevClaw-RepoWorker"
    max_steps: int = 50

    system_prompt: str = """
You are a RepoWorker agent for DevClaw. You own ONE repo end-to-end.

## Inputs you receive
- task_description, repo_config (name, url, default_branch, build_cmd, test_cmd)
- repo_plan (from PlannerAgent) — your specific changes
- branch_name — feature branch (same across all repos)
- work_dir — local path for cloning
- complexity_score — 1-5

## Workflow
1. git_clone — clone repo to work_dir/repo_name
2. git_branch — create/checkout branch_name from default_branch
3. If complexity_score >= 4:
     spawn coder agent with your repo context
     wait for coder result
     spawn tester agent
     wait for tester result
   Else:
     Use repo_reader to read the repo
     Use file_saver and python_execute to implement changes per the repo_plan
     Run build_and_test
4. If tests failed:
     fix the code (or re-spawn coder with the failure output as context)
     re-run build_and_test
     Repeat up to max_fix_retries (3) times
     If still failing: set pr_title prefix to "[WIP] "
5. git_status — confirm correct files changed
6. git_commit_push — commit with message: feat(<repo_name>): <summary>
7. github_create_pr — open PR, base=default_branch, head=branch_name
8. Output final JSON (this is your return value to OrchestratorAgent):
   {
     "repo_name": str, "pr_url": str, "pr_number": int,
     "branch_name": str, "build_passed": bool, "test_passed": bool,
     "commit_sha": str, "error": null
   }
9. Call Terminate.

## Safety
NEVER call git_commit_push with branch_name equal to the repo's default_branch.
"""
    available_tools: ToolCollection = ToolCollection([
        GitCloneTool(), GitBranchTool(), GitCommitPushTool(), GitStatusTool(),
        RepoReaderTool(), FileSaver(), FileOperator(), PythonExecute(),
        BuildTestTool(), GitHubPRTool(), SpawnAgentTool(), Terminate(),
    ])
```

---

#### `app/agent/devclaw_coder.py`

```python
class CoderAgent(ToolCallAgent):
    name: str = "DevClaw-Coder"
    max_steps: int = 40

    system_prompt: str = """
You are the Coder agent for DevClaw. You implement code changes in a cloned repo.

## Inputs you receive
- repo_dir: absolute path to the already-cloned, already-branched repo
- repo_plan: RepoPlan JSON with files_to_modify and summary_of_changes
- failure_context (optional): TesterAgent failure output if this is a fix attempt

## Workflow
1. repo_reader — read the full repo to understand structure and conventions.
2. For each file in files_to_modify: use file_operator to read current content first.
3. Implement changes using file_saver. Follow the existing code style exactly.
4. If this is a fix attempt: read failure_context carefully, fix the specific error.
5. Write or update tests in tests/ for any new logic.
6. git_status — confirm which files changed.
7. Output ONLY this JSON:
   {"repo_name": str, "files_changed": [str], "summary": str}
8. Call Terminate.

## Rules
- Do NOT run tests. TesterAgent does that.
- Do NOT commit. RepoWorkerAgent does that.
- Do NOT introduce new dependencies unless the plan explicitly requires them.
- If a new dependency is needed: add it to requirements.txt or pyproject.toml.
- Follow existing naming conventions, docstring style, and import ordering.
"""
    available_tools: ToolCollection = ToolCollection([
        RepoReaderTool(), FileOperator(), FileSaver(),
        PythonExecute(), GitStatusTool(), Terminate(),
    ])
```

---

#### `app/agent/devclaw_tester.py`

```python
class TesterAgent(ToolCallAgent):
    name: str = "DevClaw-Tester"
    max_steps: int = 20

    system_prompt: str = """
You are the Tester agent for DevClaw. You run builds and tests and report results.

## Inputs you receive
- repo_dir: absolute path to the cloned repo with code changes already applied
- build_cmd: docker build command (empty string = skip)
- test_cmd: pytest command (empty string = skip)

## Workflow
1. Call build_and_test with the provided commands.
2. Analyse the output carefully.
3. Output ONLY this JSON (your return value — OrchestratorAgent trusts this as truth):
   {
     "repo_name": str,
     "build_passed": bool,
     "test_passed": bool,
     "overall_passed": bool,
     "failure_summary": str,    // empty string if passed; exact error lines if failed
     "suggested_fix": str       // describe the code change needed, not a test command
   }
4. Call Terminate.

## Rules
- You do NOT write code. Ever.
- failure_summary must include the exact failing test name and error message.
- suggested_fix should describe the change at the code level (e.g. "add None check
  before calling .lower() on line 47 of app/search.py").
- Your JSON output is the source of truth. OrchestratorAgent trusts it over all other claims.
"""
    available_tools: ToolCollection = ToolCollection([
        BuildTestTool(), Terminate(),
    ])
```

---

#### `app/agent/devclaw_reviewer.py`

```python
class ReviewAgent(ToolCallAgent):
    name: str = "DevClaw-Reviewer"
    max_steps: int = 20

    system_prompt: str = """
You are the Reviewer agent for DevClaw. You do cross-repo consistency review.

## Inputs you receive
- original_task: AgentTask JSON
- repo_results: list of RepoResult JSON objects (one per repo)

## Workflow
1. For each pair of repos, check consistency:
   - Are shared interface contracts maintained? (API endpoint names, request/response shapes)
   - Are naming conventions consistent across repos?
   - Are there any repos that should have changed but didn't?
2. For each repo's PR, check quality:
   - Clear conventional-commits title?
   - PR body accurately describes changes?
   - Test results included?
3. Output ONLY this JSON:
   {
     "approved": bool,
     "issues": [
       {"repo": str, "severity": "warning" | "error", "description": str}
     ],
     "suggested_pr_title_overrides": {"repo_name": "new title"},
     "overall_summary": str
   }
4. Call Terminate.

## Rules
- "warning": noted but does not block. OrchestratorAgent includes it in summary.
- "error": reported to OrchestratorAgent who decides to fix or accept.
- You do NOT write code or modify files.
- Focus on cross-repo issues — single-repo code quality is CoderAgent's domain.
"""
    available_tools: ToolCollection = ToolCollection([Terminate()])
```

---

### `app/flow/devclaw_flow.py`

```python
import json, shutil, uuid
from dataclasses import asdict
from pathlib import Path
from app.flow.planning import PlanningFlow
from app.agent.devclaw_orchestrator import OrchestratorAgent
from app.task_input.schema import AgentTask, RepoConfig
from app.config.config import config
from app.logger import logger

class DevClawFlow(PlanningFlow):
    """
    Thin entry-point flow. Instantiates OrchestratorAgent and passes it
    the full task context. OrchestratorAgent uses SpawnAgentTool to fan
    out the swarm via Claude Code's Task tool.
    """
    task: AgentTask
    repo_configs: list[RepoConfig]
    run_id: str

    async def run(self) -> str:
        work_dir = Path(config.devclaw.work_dir) / self.run_id
        work_dir.mkdir(parents=True, exist_ok=True)
        logger.info(f"DevClaw run {self.run_id} | work_dir={work_dir}")

        try:
            orchestrator = OrchestratorAgent()
            prompt = self._build_orchestrator_prompt(work_dir)
            result = await orchestrator.run(prompt)
            return result
        finally:
            if getattr(config.devclaw, "cleanup_after_run", False):
                shutil.rmtree(work_dir, ignore_errors=True)
                logger.info(f"Cleaned up {work_dir}")

    def _build_orchestrator_prompt(self, work_dir: Path) -> str:
        active_repos = [
            r for r in self.repo_configs
            if not self.task.repos or r.name in self.task.repos
        ]
        return f"""## DevClaw Task

Title: {self.task.title}
Source: {self.task.source}
{f"Issue: {self.task.issue_url}" if self.task.issue_url else ""}
Feature branch: {self.task.branch_name}
Run ID: {self.run_id}
Work directory: {work_dir}

## Task description
{self.task.description}

## Repos to work on
{json.dumps([asdict(r) for r in active_repos], indent=2)}

## Extra context
{json.dumps(self.task.extra_context, indent=2) if self.task.extra_context else "none"}

Score complexity, assemble your team, execute the task, and report all PR URLs.
"""
```

---

### `run_devclaw.py`

```python
#!/usr/bin/env python3
"""
DevClaw — AI multi-agent swarm for automated GitHub PRs.

Usage:
  python run_devclaw.py --prompt "Add Redis caching" --branch feat/redis --repos backend-api
  python run_devclaw.py --prompt "Upgrade to Python 3.12" --branch chore/py312
  python run_devclaw.py --issue 142 --issue-repo your-org/backend-api
  python run_devclaw.py --task tasks/example-task.yaml
  python run_devclaw.py --task tasks/example-task.yaml --dry-run
"""
import argparse, asyncio, os, uuid
from dotenv import load_dotenv
load_dotenv()  # load .env before anything else

from app.task_input.loader import (
    load_from_prompt, load_from_github_issue,
    load_from_config_file, load_repos_toml,
)
from app.flow.devclaw_flow import DevClawFlow
from app.config.config import config
from app.logger import logger
from app.devclaw_errors import DevClawConfigError, DevClawRepoNotAllowedError

def parse_args():
    p = argparse.ArgumentParser(description="DevClaw — AI engineering swarm")
    group = p.add_mutually_exclusive_group(required=True)
    group.add_argument("--prompt", help="Natural language task description")
    group.add_argument("--issue", type=int, help="GitHub Issue number")
    group.add_argument("--task", help="Path to YAML task file")
    p.add_argument("--branch", help="Feature branch name (required with --prompt)")
    p.add_argument("--repos", nargs="+", default=[], help="Repo names to work on")
    p.add_argument("--issue-repo", help="GitHub repo for --issue (e.g. org/repo)")
    p.add_argument("--dry-run", action="store_true")
    return p.parse_args()

async def main():
    args = parse_args()

    if not os.environ.get("DEVCLAW_GITHUB_TOKEN"):
        raise DevClawConfigError(
            "DEVCLAW_GITHUB_TOKEN not set. Copy .env.example to .env and fill it in."
        )

    if args.dry_run:
        logger.info("DRY RUN — no code will be pushed or PRs opened")
        config.devclaw.dry_run = True

    # Load task
    if args.prompt:
        if not args.branch:
            raise SystemExit("--branch is required when using --prompt")
        task = load_from_prompt(args.prompt, args.branch, args.repos)
    elif args.issue:
        if not args.issue_repo:
            raise SystemExit("--issue-repo is required when using --issue")
        task = load_from_github_issue(args.issue_repo, args.issue)
        if args.repos:
            task.repos = args.repos
    else:
        task = load_from_config_file(args.task)
        if args.repos:
            task.repos = args.repos

    # Load and validate repos
    repo_configs = load_repos_toml("config/repos.toml")
    if task.repos:
        allowed = {r.name for r in repo_configs}
        unknown = set(task.repos) - allowed
        if unknown:
            raise DevClawRepoNotAllowedError(
                f"Repos not in repos.toml (add them first): {unknown}"
            )

    run_id = str(uuid.uuid4())[:8]
    logger.info(f"🦀 DevClaw | task={task.title!r} | branch={task.branch_name} | run={run_id}")

    flow = DevClawFlow(task=task, repo_configs=repo_configs, run_id=run_id)
    summary = await flow.run()
    print(summary)

if __name__ == "__main__":
    asyncio.run(main())
```

---

## Test files to create

Create these test files in `tests/devclaw/`. They validate the safety rules and
tool mechanics — they do NOT require a real GitHub token or real repo.

### `tests/devclaw/test_safety.py` — most important test file

```python
"""Safety rules must be bullet-proof. These tests mock subprocess and GitHub API."""
import pytest
from unittest.mock import patch, MagicMock
from app.tool.git_tools import GitCommitPushTool
from app.devclaw_errors import DevClawSafetyError
from app.task_input.schema import RepoConfig

MOCK_REPOS = [RepoConfig("backend-api", "https://...", "main", "", "", "python")]

@pytest.mark.asyncio
async def test_git_commit_push_refuses_default_branch():
    """GitCommitPushTool must raise DevClawSafetyError if branch == default_branch."""
    tool = GitCommitPushTool()
    with patch("app.tool.git_tools.load_repos_toml", return_value=MOCK_REPOS):
        with pytest.raises(DevClawSafetyError):
            await tool.execute(
                repo_dir="/tmp/test",
                commit_message="test commit",
                branch_name="main",      # <-- this is a default branch
            )

@pytest.mark.asyncio
async def test_git_commit_push_allows_feature_branch():
    """GitCommitPushTool must proceed normally for non-default branches."""
    tool = GitCommitPushTool()
    with patch("app.tool.git_tools.load_repos_toml", return_value=MOCK_REPOS):
        with patch("subprocess.run") as mock_run:
            mock_run.return_value = MagicMock(returncode=0, stdout="abc123\n", stderr="")
            result = await tool.execute(
                repo_dir="/tmp/test",
                commit_message="feat(test): add thing",
                branch_name="feat/my-feature",   # <-- safe
            )
    assert result.error is None

@pytest.mark.asyncio
async def test_github_pr_tool_refuses_default_branch():
    """GitHubPRTool must refuse to create a PR from a default branch."""
    from app.tool.github_pr_tool import GitHubPRTool
    tool = GitHubPRTool()
    with pytest.raises((DevClawSafetyError, AssertionError)):
        await tool.execute(
            repo_name="backend-api",
            branch_name="main",        # <-- must be refused
            title="test",
            body="test",
            base_branch="main",
            linked_pr_urls=[],
        )
```

### `tests/devclaw/test_task_input.py`

```python
"""Test all three task loaders."""
import pytest, yaml, os, tempfile
from app.task_input.loader import load_from_prompt, load_from_config_file, load_repos_toml
from app.devclaw_errors import DevClawConfigError

def test_load_from_prompt():
    task = load_from_prompt("Add caching", "feat/cache", ["backend-api"])
    assert task.source == "prompt"
    assert task.branch_name == "feat/cache"
    assert task.repos == ["backend-api"]

def test_load_from_config_file_valid():
    data = {"title": "T", "description": "D", "branch_name": "feat/x", "repos": ["api"]}
    with tempfile.NamedTemporaryFile(mode="w", suffix=".yaml", delete=False) as f:
        yaml.dump(data, f)
        path = f.name
    task = load_from_config_file(path)
    assert task.title == "T"
    assert task.repos == ["api"]

def test_load_from_config_file_missing_required_fields():
    data = {"title": "T"}   # missing description and branch_name
    with tempfile.NamedTemporaryFile(mode="w", suffix=".yaml", delete=False) as f:
        yaml.dump(data, f)
        path = f.name
    with pytest.raises(DevClawConfigError):
        load_from_config_file(path)

def test_load_repos_toml_missing_file():
    with pytest.raises(DevClawConfigError):
        load_repos_toml("/nonexistent/path/repos.toml")
```

---

## Implementation phases

Work through these in order. Do not start the next phase until the test gate passes.

### Phase 1 — Scaffolding (no LLM, no GitHub)

Create in this order:
1. `CLAUDE.md`
2. `.env.example` + update `.gitignore`
3. `app/devclaw_errors.py`
4. `app/task_input/schema.py`
5. `app/task_input/__init__.py` (empty)
6. `app/task_input/loader.py` — `load_from_prompt` and `load_repos_toml` only
7. `config/repos.toml` — add your real repos
8. `app/tool/repo_reader_tool.py`
9. `app/tool/git_tools.py` — `GitCloneTool` and `GitBranchTool` only
10. `tests/devclaw/__init__.py` (empty)
11. `tests/devclaw/test_task_input.py`
12. Minimal `run_devclaw.py` — load task, load repos, clone, print repo summary, exit

**Test gate:**
```bash
pytest tests/devclaw/test_task_input.py -v
python run_devclaw.py --prompt "test task" --branch feat/test --repos <your-first-repo> --dry-run
# Should clone the repo and print its file summary. No LLM calls. No errors.
```

---

### Phase 2 — Single-agent baseline (first LLM calls)

13. `app/tool/build_test_tool.py`
14. `app/tool/git_tools.py` — add `GitCommitPushTool` + `GitStatusTool`
15. `app/tool/github_pr_tool.py` — `GitHubPRTool` + `GitHubUpdatePRTool`
16. `tests/devclaw/test_safety.py` ← write and pass this BEFORE wiring agents
17. `app/agent/devclaw_repo_worker.py` — inline mode only (no `SpawnAgentTool`)
18. `app/flow/devclaw_flow.py` — runs `RepoWorkerAgent` directly (no orchestrator yet)
19. Update `run_devclaw.py` to use `DevClawFlow`

**Test gate:**
```bash
pytest tests/devclaw/test_safety.py -v
# All safety tests must pass before moving on.

python run_devclaw.py \
  --prompt "Add a hello_world() function to the utils module" \
  --branch feat/hello-world \
  --repos <your-test-repo> \
  --dry-run
# Should show what it would do. No actual push or PR.

python run_devclaw.py \
  --prompt "Add a hello_world() function to the utils module" \
  --branch feat/hello-world \
  --repos <your-test-repo>
# Should open a real PR. Verify it on GitHub.
```

---

### Phase 3 — Specialist agents

20. `app/agent/devclaw_planner.py`
21. `app/agent/devclaw_coder.py`
22. `app/agent/devclaw_tester.py`
23. `app/agent/devclaw_reviewer.py`
24. Update `RepoWorkerAgent` to spawn Coder + Tester when complexity >= 4

**Test gate:**
```bash
python run_devclaw.py \
  --prompt "Add input validation and unit tests to the API endpoint handlers" \
  --branch feat/validation \
  --repos <your-test-repo>
# Should use PlannerAgent (score >= 2), CoderAgent, TesterAgent, ReviewAgent.
# Check logs to confirm each agent ran. Verify the PR body has test results.
```

---

### Phase 4 — Full swarm (multi-repo + orchestrator)

25. `app/tool/spawn_agent_tool.py`
26. `app/agent/devclaw_orchestrator.py`
27. Update `DevClawFlow` to route through `OrchestratorAgent`
28. Multi-repo parallel execution
29. PR back-linking (OrchestratorAgent calls `GitHubUpdatePRTool` on all PRs)

**Test gate:**
```bash
python run_devclaw.py \
  --prompt "Add structured logging with correlation IDs to all API endpoints" \
  --branch feat/structured-logging \
  --repos backend-api data-pipeline
# Should open 2 PRs with the same branch name.
# Each PR body should link to the other PR.
# Verify both PRs on GitHub.
```

---

### Phase 5 — Hardening

30. `load_from_github_issue` in `loader.py`
31. `load_from_config_file` in `loader.py`
32. `tasks/example-task.yaml`
33. Dry-run mode verification (confirm `--dry-run` makes zero GitHub API calls)
34. Error isolation: wrap each RepoWorker result in try/except in OrchestratorAgent
35. Work directory cleanup: implement and test `cleanup_after_run = true`
36. Token masking: add `_mask_token()` helper, apply to all subprocess output

**Final test gate:**
```bash
pytest tests/devclaw/ -v
# All tests must pass.

python run_devclaw.py --issue 1 --issue-repo your-org/your-repo --dry-run
python run_devclaw.py --task tasks/example-task.yaml --dry-run
# Both should parse successfully and print a plan without making any real calls.
```

---

## Key design decisions (do not deviate from these)

**Claude Code is the message bus.** `SpawnAgentTool` wraps Claude Code's native
`Task` tool. No custom IPC, no shared database, no Redis. Sub-agents return
structured JSON as their final message. OrchestratorAgent parses it.

**Branch safety is enforced at the Python level.** `GitCommitPushTool` raises
`DevClawSafetyError` in Python before calling git if `branch_name` matches any
`default_branch` in `repos.toml`. This is not a prompt instruction. It cannot be
overridden by any LLM output. The safety test in `test_safety.py` must pass.

**Secrets from environment only.** `DEVCLAW_GITHUB_TOKEN` is read from `os.environ`.
It is never logged, never passed as a function parameter, never stored in any
dataclass. The `_mask_token()` helper in `git_tools.py` must be applied to all
subprocess stdout/stderr before logging or returning.

**Structured JSON outputs everywhere.** PlannerAgent outputs `[RepoPlan]`.
RepoWorkerAgent outputs `RepoResult`. TesterAgent outputs its JSON report.
ReviewAgent outputs `ReviewReport`. OrchestratorAgent parses these — it never
does string matching on free-form text.

**TesterAgent is the source of truth.** OrchestratorAgent's system prompt
explicitly states it trusts TesterAgent's JSON over all other claims.
`test_safety.py` should also test that OrchestratorAgent rejects a "passed"
claim from CoderAgent when TesterAgent reports failure.

**FileOperator for reading, FileSaver for writing.** CoderAgent uses both.
Read the file first with `FileOperator`, then write the modified version with
`FileSaver`. Do not use `FileSaver` to read.

**Idempotent git operations.** `GitBranchTool` checks if the branch exists
remotely before creating it. Re-running a failed task resumes the existing branch.
`GitCommitPushTool` handles "nothing to commit" gracefully (returns a result, not an error).

**Work directory isolation.** Every run uses `/tmp/devclaw/<run_id>/` so
concurrent runs and re-runs never collide.

**Conventional commits everywhere.** All commit messages follow the format:
`<type>(<scope>): <description>` — e.g. `feat(backend-api): add Redis caching`.
This is in the system prompts for RepoWorkerAgent and CoderAgent.
