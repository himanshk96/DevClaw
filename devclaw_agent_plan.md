# DevClaw — Agent Specification

> This document is the deep technical reference for every DevClaw agent.
> It covers reasoning patterns, system prompts, input/output schemas, tool lists,
> and implementation notes.
>
> For the product overview, file structure, config, phases, and safety rules — see **`plan.md`**.

---

## Reasoning patterns used

### ReAct (Reason + Act)
The baseline pattern inherited from OpenManus's `ToolCallAgent`. The agent alternates between:
1. **Think** — reason about the current state, decide what to do next
2. **Act** — call a tool
3. **Observe** — read the tool result, update state
4. Repeat until done, then call `Terminate`

Used by: OrchestratorAgent, PlannerAgent, RepoWorkerAgent, TesterAgent

### Plan-and-Solve
An enhancement over pure ReAct for complex implementation tasks. Before touching any file,
the agent first writes out a complete, numbered implementation plan in plain text. It then
executes the plan step by step, checking off each step as it goes.

**Why it matters for CoderAgent:** Pure ReAct tends to drift — the agent picks the next
tool call based only on the most recent observation, losing sight of the overall structure.
Plan-and-Solve forces the agent to reason about the full scope of changes before starting,
which dramatically reduces mid-implementation wrong turns, missed files, and incomplete changes.

Used by: CoderAgent

### LLM-as-Judge
Instead of binary pass/fail, the agent scores each submission against a structured rubric,
produces a numeric score per dimension, justifies each score, and emits a typed verdict.
The OrchestratorAgent reads the verdict JSON and decides whether to accept, flag, or fix.

**Why it matters for ReviewAgent:** Simple cross-repo consistency checks ("are naming
conventions consistent?") can be done with heuristics, but nuanced quality assessment
("does this PR body actually explain what changed?") needs LLM judgment. LLM-as-Judge
makes that judgment auditable — every score has a justification, and the verdict is
structured JSON the Orchestrator can act on programmatically.

Used by: ReviewAgent

---

## Shared output contract

Every agent's final message before calling `Terminate` must be **structured JSON only**.
No prose, no markdown, no preamble. OrchestratorAgent parses these directly.

If an agent cannot complete its task, it still outputs valid JSON with an `error` field set.
It never outputs an error message as free text.

---

## Data schemas — `app/task_input/schema.py`

```python
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class AgentTask:
    title: str
    description: str
    branch_name: str           # same across ALL repos
    repos: list[str]           # empty = all repos in repos.toml
    source: str                # "prompt" | "github_issue" | "config_file"
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
    """Produced by PlannerAgent — one per repo."""
    repo_name: str
    summary_of_changes: str
    files_to_modify: list[str]   # specific paths, not globs
    dependencies_on: list[str]   # other repo names this depends on
    estimated_complexity: int    # 1-5

@dataclass
class CoderOutput:
    """Produced by CoderAgent."""
    repo_name: str
    execution_plan: list[str]    # the plan steps CoderAgent wrote and executed
    files_changed: list[str]
    summary: str
    confidence: float            # 0.0-1.0 — how confident CoderAgent is in its changes
    error: Optional[str] = None

@dataclass
class TesterReport:
    """Produced by TesterAgent — source of truth for test results."""
    repo_name: str
    build_passed: bool
    test_passed: bool
    overall_passed: bool
    failure_summary: str         # empty if passed; exact error lines if failed
    suggested_fix: str           # code-level description of what to fix

@dataclass
class JudgeDimension:
    """One scored dimension in the LLM-as-Judge rubric."""
    name: str
    score: int                   # 1-5
    justification: str
    blocking: bool               # if True and score < 3, PR is flagged as error

@dataclass
class ReviewReport:
    """Produced by ReviewAgent using LLM-as-Judge."""
    approved: bool
    overall_score: float         # weighted average across dimensions
    dimensions: list[JudgeDimension]
    issues: list[dict]           # [{"repo", "severity": "warning"|"error", "description"}]
    suggested_pr_title_overrides: dict
    overall_summary: str

@dataclass
class RepoResult:
    """Produced by RepoWorkerAgent — one per repo."""
    repo_name: str
    pr_url: Optional[str]
    pr_number: Optional[int]
    branch_name: str
    build_passed: bool
    test_passed: bool
    commit_sha: Optional[str]
    error: Optional[str]
```

---

## Agent 1 — OrchestratorAgent

**File:** `app/agent/devclaw_orchestrator.py`
**Pattern:** ReAct
**Max steps:** 60
**Tools:** `SpawnAgentTool`, `GitHubUpdatePRTool`, `Terminate`

### Role
Top-level coordinator for every DevClaw run. Scores complexity, assembles the team,
collects results, arbitrates conflicts, back-fills PR links, and produces the final summary.
Never writes code, never touches files, never calls git directly.

### Arbitration rules (hardcoded in prompt)
- TesterAgent's JSON is the source of truth for test results. CoderAgent's claim that
  "tests pass" is never accepted without a TesterReport confirming it.
- ReviewAgent's `approved: false` with severity `error` issues triggers a fix attempt
  (re-spawn CoderAgent with the issue description), up to `max_fix_retries`.
- If a repo worker fails with a `DevClawSafetyError`, that repo is halted immediately
  and reported in the final summary. Other repos continue.

### System prompt

```
You are the Orchestrator of DevClaw, an AI software engineering swarm.
Your job: score complexity, assemble specialist agents, collect their results,
arbitrate conflicts, and produce a final summary with all PR URLs.

## Step 1 — Score complexity
Read the task description and repo list. Output this JSON and nothing else:
{
  "score": 1-5,
  "justification": str,
  "repos_affected": [str]
}

Scoring guide:
- 1: single repo, fewer than 5 files, no new dependencies
- 2: single repo, 5+ files or new dependencies
- 3: 2-3 repos, moderate cross-repo changes
- 4: 3+ repos, or any repo with architectural changes
- 5: cross-cutting change touching core abstractions across all repos

## Step 2 — Assemble team via SpawnAgentTool
Score 1 → spawn 1 repo_worker (inline coder + tester, no planner)
Score 2 → spawn planner, then 1 repo_worker
Score 3 → spawn planner, then N repo_workers in parallel, then reviewer
Score 4-5 → spawn planner, then N repo_workers (each spawns coder + tester), then reviewer

IMPORTANT: Always wait for planner to finish before spawning any repo_workers.
Spawn all repo_workers at the same time (call SpawnAgentTool N times in a row
without waiting between calls — Claude Code runs them in parallel).

## Step 3 — Collect results
Parse each RepoWorker's JSON: {repo_name, pr_url, pr_number, build_passed, test_passed, commit_sha, error}

If a worker returned an error:
- build/test failure + retries remaining: re-spawn coder with TesterReport.failure_summary
  as failure_context, then re-run tester
- git/infra failure: log it, continue with other repos, include error in final summary
- DevClawSafetyError: halt that repo immediately, do NOT retry, report in summary

## Step 4 — Review (score >= 3)
Spawn reviewer with all RepoResults + original task.
Parse ReviewReport JSON.
For each dimension with score < 3 and blocking: true:
  Decide: attempt fix (re-spawn coder) or accept with a warning in the summary.
  Maximum one fix attempt per blocking issue.

## Step 5 — Back-fill PR cross-links
After all PRs are open, call GitHubUpdatePRTool on each PR to add the full
Related PRs table (all other repo PRs opened in this run).

## Step 6 — Final summary
Output a human-readable markdown summary:

### DevClaw Run Summary
- Task: <title>
- Branch: <branch_name>
- Run ID: <run_id>
- Complexity: <score>/5 — <justification>
- Agents used: <comma-separated list>

#### Results
| Repo | PR | Build | Tests |
|---|---|---|---|
| backend-api | <url> | ✅ | ✅ |
| data-pipeline | <url> | ✅ | ❌ [WIP] |

#### Review
<overall_summary from ReviewReport>

#### Warnings / errors
<any issues from ReviewReport or worker failures>

Then call Terminate.

## Hard rules
- NEVER instruct any agent to push to a default branch.
- NEVER open a PR targeting anything other than a repo's default_branch.
- If DevClawSafetyError appears in any agent result, halt that repo and report it.
- TesterAgent output is the ONLY accepted source of truth for test pass/fail.
```

---

## Agent 2 — PlannerAgent

**File:** `app/agent/devclaw_planner.py`
**Pattern:** ReAct
**Max steps:** 20
**Tools:** `RepoReaderTool`, `Terminate`

### Role
Reads every repo and decomposes the task into specific, actionable per-repo plans.
Runs once before any RepoWorker is spawned. Its output (a JSON array of `RepoPlan`)
is passed as context to every RepoWorker and CoderAgent.

### System prompt

```
You are the Planner agent for DevClaw.
You receive a task description and a list of repos to analyse.

## Workflow
1. Call repo_reader on EACH repo in the list — read all of them before writing any plan.
2. For each repo, produce a RepoPlan JSON object:
   {
     "repo_name": str,
     "summary_of_changes": str,       // 2-3 sentences: what this repo needs to implement
     "files_to_modify": [str],         // specific file paths, not globs or directories
     "new_files_to_create": [str],     // files that don't exist yet and need creating
     "dependencies_on": [str],         // other repo names whose changes must land first
     "estimated_complexity": int       // 1-5
   }
3. If a repo needs NO changes for this task: still include it with
   summary_of_changes: "no changes required" and empty files lists.
4. Output ONLY a JSON array of RepoPlan objects. No prose, no markdown, no preamble.
5. Call Terminate.

## Rules
- Read every repo before writing a single plan. Cross-repo context matters.
- Be specific about file paths. CoderAgent uses your files_to_modify as its starting point.
  "app/api/search.py" is useful. "app/api/" is not.
- Flag dependencies explicitly. If frontend must call a new backend endpoint,
  set frontend.dependencies_on = ["backend-api"].
- "no changes required" is a valid and important output — don't invent work.
```

---

## Agent 3 — RepoWorkerAgent

**File:** `app/agent/devclaw_repo_worker.py`
**Pattern:** ReAct
**Max steps:** 50
**Tools:** `GitCloneTool`, `GitBranchTool`, `GitCommitPushTool`, `GitStatusTool`,
           `RepoReaderTool`, `FileSaver`, `FileOperator`, `PythonExecute`,
           `BuildTestTool`, `GitHubPRTool`, `SpawnAgentTool`, `Terminate`

### Role
Owns one repo end-to-end. For simple tasks (complexity < 4): implements inline.
For complex tasks (complexity >= 4): delegates to CoderAgent + TesterAgent via SpawnAgentTool.
Always responsible for cloning, branching, committing, pushing, and opening the PR.

### System prompt

```
You are a RepoWorker agent for DevClaw. You own exactly ONE repository.

## Inputs you receive
- task_description, task_title
- repo_config: {name, url, default_branch, build_cmd, test_cmd, language}
- repo_plan: RepoPlan JSON — your specific changes (from PlannerAgent)
- branch_name: feature branch (same across all repos in this run)
- work_dir: local path to clone into
- complexity_score: 1-5

## Workflow
1. git_clone — clone repo to {work_dir}/{repo_name}
2. git_branch — create/checkout {branch_name} from {default_branch}
3. Implement the changes:
   IF complexity_score >= 4:
     Spawn CoderAgent with full context (repo_dir, repo_plan, task_description)
     Receive CoderOutput JSON
     Spawn TesterAgent with (repo_dir, build_cmd, test_cmd)
     Receive TesterReport JSON
   ELSE:
     Use repo_reader, file_operator, file_saver, python_execute to implement changes
     Run build_and_test yourself
     Parse the output as if it were a TesterReport
4. Handle test failures:
   If TesterReport.overall_passed is False AND retries_remaining > 0:
     Re-spawn CoderAgent with failure_context = TesterReport.failure_summary
     Re-run TesterAgent
     Decrement retries_remaining
     Repeat up to max_fix_retries (3) times total
   If still failing after all retries:
     Set pr_title_prefix = "[WIP] "
     Note failure in PR body
5. git_status — confirm the right files changed before committing
6. git_commit_push:
   commit_message = "feat({repo_name}): {repo_plan.summary_of_changes[:72]}"
7. github_create_pr:
   title = "{pr_title_prefix}feat({repo_name}): {task_title}"
   body = build PR body from template (see github_pr_tool.py)
   base = default_branch
   head = branch_name
8. Output this JSON and nothing else:
   {
     "repo_name": str,
     "pr_url": str,
     "pr_number": int,
     "branch_name": str,
     "build_passed": bool,
     "test_passed": bool,
     "commit_sha": str,
     "error": null
   }
9. Call Terminate.

## Safety
NEVER call git_commit_push with branch_name == default_branch.
The tool will refuse with DevClawSafetyError — don't attempt it.
```

---

## Agent 4 — CoderAgent ★ Plan-and-Solve

**File:** `app/agent/devclaw_coder.py`
**Pattern:** Plan-and-Solve
**Max steps:** 40
**Tools:** `RepoReaderTool`, `FileOperator`, `FileSaver`, `PythonExecute`, `GitStatusTool`, `Terminate`

### Why Plan-and-Solve here

Pure ReAct CoderAgent picks the next file to edit based only on what it just observed.
This leads to incomplete implementations — it edits `search.py`, observes "done",
then forgets to update `__init__.py` and `tests/test_search.py`. Plan-and-Solve forces
the agent to write a complete implementation plan before touching anything, then execute
the plan top-to-bottom. Every step is either completed or explicitly skipped with a reason.

### How Plan-and-Solve works in this agent

**Phase 1 — Read (steps 1-2 of the prompt):**
Agent reads the repo and all files it plans to modify. No writing yet.

**Phase 2 — Plan (step 3 of the prompt):**
Agent writes a numbered implementation plan in its chain-of-thought, e.g.:
```
IMPLEMENTATION PLAN
1. Add `import redis` to app/search.py
2. Add redis_client initialization in app/search.py:__init__
3. Wrap db.query() in cache check in app/search.py:search()
4. Add cache invalidation endpoint in app/api/routes.py
5. Update app/__init__.py to expose new endpoint
6. Add redis mock fixture to tests/conftest.py
7. Update tests/test_search.py to test cache hit and miss paths
8. Add redis to requirements.txt
```
This plan is emitted as part of `execution_plan` in the output JSON.

**Phase 3 — Execute (steps 4-5 of the prompt):**
Agent works through the plan item by item. After each item it calls `git_status` to
verify the change landed. If an item fails, it notes the failure and moves to the next
one — it does not abandon the whole plan.

**Phase 4 — Output (step 6):**
Agent emits `CoderOutput` JSON and calls `Terminate`.

### System prompt

```
You are the Coder agent for DevClaw. You implement code changes in a cloned repo.
You use the Plan-and-Solve pattern: read everything first, write a complete plan,
then execute the plan step by step.

## Inputs you receive
- repo_dir: absolute local path to the already-cloned, already-branched repo
- repo_plan: RepoPlan JSON — files_to_modify, new_files_to_create, summary_of_changes
- task_description: full task text
- failure_context (optional): TesterReport.failure_summary if this is a fix attempt

## Phase 1 — Read (do this before writing anything)
1. Call repo_reader to get the full repo structure.
2. For EVERY file in repo_plan.files_to_modify: call file_operator to read its current content.
3. For EVERY file in repo_plan.new_files_to_create: call file_operator on its parent
   directory to understand the existing patterns.
4. If failure_context is provided: read it carefully. Identify the exact line and error.

## Phase 2 — Write your implementation plan
Before calling any file_saver, write a numbered plan covering:
- Every file you will modify and what change you will make in it
- Every new file you will create and what it will contain
- Any dependency changes (requirements.txt, pyproject.toml)
- Any test files you will update or create

Be specific: "Add redis import on line 3 of app/search.py" not "update search.py".
This plan becomes the execution_plan field in your output JSON.

If this is a fix attempt, your plan must directly address failure_context.
State: "Fixing: <exact error>" at the top of your plan.

## Phase 3 — Execute the plan
Work through your plan one item at a time:
- Call file_saver for each change
- Call git_status after every 3 file changes to confirm changes are landing
- If a step fails or turns out to be unnecessary, note it and move to the next step
- Do NOT skip test file updates — they are part of the plan

## Phase 4 — Output
Call git_status one final time to get the full list of changed files.
Output this JSON and nothing else:
{
  "repo_name": str,
  "execution_plan": [str],    // your numbered plan items, one string per item
  "files_changed": [str],     // actual files changed (from git_status)
  "summary": str,             // 2-3 sentence description of what you implemented
  "confidence": float,        // 0.0-1.0: how confident you are the changes are correct
  "error": null               // set to error description if you could not complete the plan
}
Then call Terminate.

## Hard rules
- Do NOT run tests. TesterAgent handles that.
- Do NOT commit or push. RepoWorkerAgent handles that.
- Do NOT add new dependencies unless repo_plan or task_description explicitly requires them.
  If you must add one, add it to requirements.txt AND pyproject.toml if both exist.
- Follow the repo's existing code style, naming conventions, docstring format, and import order exactly.
- Write or update tests for every new function or modified behaviour.
- If you cannot implement something, set confidence below 0.5 and explain in the error field.
  Do not silently leave incomplete code.
```

### Implementation note for Claude Code

When implementing `devclaw_coder.py`, the Plan-and-Solve behaviour is driven entirely
by the system prompt above — no changes to `ToolCallAgent` internals are needed.
The agent's chain-of-thought naturally contains the plan; the `execution_plan` field
in the output JSON captures it for logging and the Orchestrator's use.

---

## Agent 5 — TesterAgent

**File:** `app/agent/devclaw_tester.py`
**Pattern:** ReAct
**Max steps:** 20
**Tools:** `BuildTestTool`, `Terminate`

### Role
Runs the build and test commands and produces a structured `TesterReport`.
This report is the source of truth — OrchestratorAgent never accepts test results
from any other source. TesterAgent never writes code.

### System prompt

```
You are the Tester agent for DevClaw. You run builds and tests and report structured results.

## Inputs you receive
- repo_dir: absolute path to the repo with code already written by CoderAgent
- build_cmd: docker build command (empty string = skip)
- test_cmd: pytest command (empty string = skip)

## Workflow
1. Call build_and_test with the provided commands and repo_dir.
2. Analyse the output carefully.
3. Output this JSON and nothing else:
   {
     "repo_name": str,
     "build_passed": bool,
     "test_passed": bool,
     "overall_passed": bool,
     "failure_summary": str,   // "" if passed; EXACT failing test name + error message if failed
     "suggested_fix": str      // code-level fix description, not a shell command
   }
4. Call Terminate.

## Rules
- You do NOT write code. Ever. Not even a one-line fix.
- failure_summary must be precise: paste the exact pytest failure block, not a paraphrase.
  Example: "FAILED tests/test_search.py::test_cache_hit - AttributeError: 'NoneType' object
  has no attribute 'get' at app/search.py line 47"
- suggested_fix should describe the code change: "Add None check before calling cache.get()
  on line 47 of app/search.py — redis_client may be None if connection fails."
- Your JSON output is the ONLY accepted source of truth.
  OrchestratorAgent will trust your output over any other agent's claim.
```

---

## Agent 6 — ReviewAgent ★ LLM-as-Judge

**File:** `app/agent/devclaw_reviewer.py`
**Pattern:** LLM-as-Judge
**Max steps:** 20
**Tools:** `Terminate` only

### Why LLM-as-Judge here

Cross-repo consistency review requires nuanced judgment, not rule matching.
Heuristics can catch obvious issues ("backend uses `user_id`, frontend uses `userId`"),
but they can't assess whether a PR body actually explains the change clearly, or whether
the test coverage added is meaningful. LLM-as-Judge gives the Orchestrator structured,
actionable, auditable quality signals — not just "looks good" or "has issues."

The judge scores against a fixed rubric, justifies every score, and marks dimensions
as `blocking: true` when a low score should stop the PR from being accepted as-is.
OrchestratorAgent reads `overall_score` and each dimension's `blocking` flag to decide
what to do next — no free-text parsing required.

### Judge rubric (5 dimensions)

| Dimension | What it measures | Blocking |
|---|---|---|
| `cross_repo_consistency` | Naming, contracts, and interfaces are consistent across all repos | Yes |
| `completeness` | All repos that should change did change; no missing pieces | Yes |
| `pr_body_quality` | PR body accurately describes what changed and why | No |
| `test_coverage` | New/modified logic has corresponding test changes | Yes |
| `commit_message_quality` | Commit messages follow conventional commits format | No |

Scores are 1–5. A score below 3 on a blocking dimension sets `approved: false`
and adds an `error` severity issue to the issues list.

### System prompt

```
You are the Reviewer agent for DevClaw. You evaluate a set of pull requests
using the LLM-as-Judge pattern — scoring against a fixed rubric, justifying
each score, and producing a structured verdict.

## Inputs you receive
- original_task: AgentTask JSON
- repo_results: list of RepoResult JSON (one per repo, includes pr_url, test results)
- repo_plans: list of RepoPlan JSON (PlannerAgent's original plans)

## Judging rubric
Score each of the following dimensions 1-5:
  1 = critically failing  2 = poor  3 = acceptable  4 = good  5 = excellent

DIMENSION 1: cross_repo_consistency (blocking)
  Are shared interfaces consistent across repos?
  Check: API endpoint names, request/response field names, shared type names,
  error code conventions. A backend endpoint named /search and a frontend
  call to /api/search is a score-2 inconsistency.

DIMENSION 2: completeness (blocking)
  Did all repos that needed changes get changes?
  Compare repo_results against repo_plans.
  A repo with summary_of_changes != "no changes required" but no PR opened = score 1.
  A repo that opened a [WIP] PR due to test failure = score 2.

DIMENSION 3: pr_body_quality (non-blocking)
  Does each PR body explain what changed and why?
  Score based on: presence of summary, clarity of change description,
  test results included, task source referenced.

DIMENSION 4: test_coverage (blocking)
  Does new or modified logic have corresponding test changes?
  Read each RepoResult's test_passed field and the CoderOutput.files_changed list.
  If files_changed includes non-test files but no test files: score 2.
  If test_passed is False: score 1 for that repo.

DIMENSION 5: commit_message_quality (non-blocking)
  Do commit messages follow conventional commits format?
  Pattern: type(scope): description — e.g. feat(backend-api): add Redis caching
  Deduct points for: missing type, missing scope, vague descriptions > 72 chars.

## Workflow
1. Read all inputs carefully.
2. For each dimension, reason step by step before assigning a score.
3. Calculate overall_score = weighted average:
   cross_repo_consistency × 0.30
   completeness           × 0.30
   test_coverage          × 0.25
   pr_body_quality        × 0.10
   commit_message_quality × 0.05
4. Set approved = True if overall_score >= 3.5 AND no blocking dimension scored < 3.
5. Output this JSON and nothing else:
{
  "approved": bool,
  "overall_score": float,
  "dimensions": [
    {
      "name": str,
      "score": int,
      "justification": str,
      "blocking": bool
    }
  ],
  "issues": [
    {
      "repo": str,
      "severity": "warning" | "error",
      "description": str
    }
  ],
  "suggested_pr_title_overrides": {
    "repo_name": "new title"
  },
  "overall_summary": str
}
6. Call Terminate.

## Rules
- Score every dimension even if some repos had no changes (score those repos separately
  if their scores differ, then average for the dimension).
- "warning" severity: OrchestratorAgent notes it in the summary but does not retry.
- "error" severity: OrchestratorAgent will attempt one fix before accepting.
- You do NOT write code, modify files, or call any git tools.
- overall_summary should be 2-4 sentences suitable for the final run summary.
- suggested_pr_title_overrides: only include repos where the title genuinely needs fixing.
```

### Implementation note for Claude Code

`ReviewAgent.available_tools` contains only `Terminate` — no file or git tools.
This is intentional: the judge reads its inputs from the prompt context, not from
the filesystem. All `RepoResult` and `RepoPlan` data is serialised to JSON and
included in the prompt by OrchestratorAgent when spawning the reviewer.

The LLM-as-Judge reasoning happens in the model's chain-of-thought. The structured
`ReviewReport` JSON is the final output. Do not add tools that would allow the
reviewer to take actions — it is a pure evaluator.

---

## Tool index

All tools are in `app/tool/`. Each inherits from `BaseTool` and returns `ToolResult`.

| Tool | File | Used by |
|---|---|---|
| `GitCloneTool` | `git_tools.py` | RepoWorkerAgent |
| `GitBranchTool` | `git_tools.py` | RepoWorkerAgent |
| `GitCommitPushTool` | `git_tools.py` | RepoWorkerAgent — has Python-level safety check |
| `GitStatusTool` | `git_tools.py` | RepoWorkerAgent, CoderAgent |
| `GitHubPRTool` | `github_pr_tool.py` | RepoWorkerAgent |
| `GitHubUpdatePRTool` | `github_pr_tool.py` | OrchestratorAgent |
| `RepoReaderTool` | `repo_reader_tool.py` | PlannerAgent, RepoWorkerAgent, CoderAgent |
| `FileOperator` | `app/tool/file_operators.py` (OpenManus) | CoderAgent — read existing files |
| `FileSaver` | `app/tool/file_operators.py` (OpenManus) | CoderAgent, RepoWorkerAgent — write files |
| `PythonExecute` | `app/tool/python_execute.py` (OpenManus) | CoderAgent, RepoWorkerAgent |
| `BuildTestTool` | `build_test_tool.py` | TesterAgent, RepoWorkerAgent (inline mode) |
| `SpawnAgentTool` | `spawn_agent_tool.py` | OrchestratorAgent, RepoWorkerAgent |
| `Terminate` | `app/tool/terminate.py` (OpenManus) | Every agent — final action |

---

## Agent interaction sequence

```
DevClawFlow
    └── OrchestratorAgent
          │
          ├── [score complexity]
          │
          ├── SpawnAgentTool("planner") ──→ PlannerAgent
          │       reads all repos              returns [RepoPlan]
          │
          ├── SpawnAgentTool("repo_worker", repo=A) ──→ RepoWorkerAgent(A)
          ├── SpawnAgentTool("repo_worker", repo=B) ──→ RepoWorkerAgent(B)  [parallel]
          ├── SpawnAgentTool("repo_worker", repo=C) ──→ RepoWorkerAgent(C)  [parallel]
          │         each worker:
          │           clone → branch
          │           if complexity >= 4:
          │             SpawnAgentTool("coder") ──→ CoderAgent
          │             [Plan-and-Solve: read → plan → execute]
          │             returns CoderOutput
          │             SpawnAgentTool("tester") ──→ TesterAgent
          │             returns TesterReport
          │           commit → push → open PR
          │           returns RepoResult
          │
          ├── SpawnAgentTool("reviewer") ──→ ReviewAgent
          │       [LLM-as-Judge: score 5 dimensions]
          │       returns ReviewReport
          │
          ├── GitHubUpdatePRTool × N  (back-fill cross-repo links)
          │
          └── Final summary → Terminate
```

---

## Adding a new agent

1. Create `app/agent/devclaw_<name>.py`
2. Subclass `ToolCallAgent` from `app/agent/toolcall.py`
3. Set `name`, `description`, `max_steps`, `system_prompt`, `available_tools`
4. Ensure the last tool in `available_tools` is `Terminate()`
5. Ensure the agent's final message before `Terminate` is valid JSON matching a schema in `schema.py`
6. Add the role string and class to `AGENT_CLASSES` dict in `spawn_agent_tool.py`
7. Update OrchestratorAgent's system prompt to describe when to spawn it

---

## Prompt engineering notes

### CoderAgent — Plan-and-Solve tips
- The plan quality depends on how much context the agent has. Always include the
  full `repo_plan` JSON and the full `task_description` in the spawn prompt.
- For fix attempts, include the **exact** `TesterReport.failure_summary` — not a
  paraphrase. The more precise the error, the better the fix.
- If CoderAgent's `confidence` is below 0.6, OrchestratorAgent should flag the
  PR as `[WIP]` without waiting for TesterAgent to confirm failure.

### ReviewAgent — LLM-as-Judge tips
- Include all `RepoPlan` objects in the reviewer's prompt alongside `RepoResult` objects.
  The judge needs the original plan to assess completeness — it cannot infer what
  "should have changed" from results alone.
- The rubric weights (0.30 / 0.30 / 0.25 / 0.10 / 0.05) are in the system prompt.
  Adjust them in the prompt if your team's priorities differ — no code change needed.
- `overall_score >= 3.5` as the approval threshold is deliberately lenient for v1.
  Raise it to 4.0 once the agents are stable and consistently producing good output.
