# jvm dependency upgrade loop

Keep a JVM project's dependencies current without hand-migrating every breaking change. The deterministic work is done by [OpenRewrite](https://docs.openrewrite.org/) recipes, not the model: bump a dependency, run the recipe that rewrites the broken APIs, build, open a PR if it's green. The model's only job is to author and refine the recipe when the build still fails. That is the nature of a good loop … a fast, reliable tool does the work, and the model only steers it.

```text
┌───────────────────────────────────────────────────────────────────────┐
│ [2] PROBLEM   version upgrades break APIs across the code; toil       │
├───────────────────────┬─────────────────────────┬─────────────────────┤
│ [3] TRIGGER           │ [1] GOAL                │ [6] LIMITS          │
│ nightly + advisory;   │                         │ budget; max recipe  │
│ one dep/version       │ mvn verify is green:    │ refines; no-progress│
│                       │ it compiles, tests      │                     │
│ [4] ACTIONS           │ pass, build succeeds    │ [7] CONTROL         │
│ run + edit rewrite    │                         │ green->PR; red->    │
│ recipes; mvn verify;  │ (the success check)     │ refine recipe;      │
│ open PR; sandbox      │                         │ else escalate       │
│ [5] STATE             │                         │ [9] MODEL & PROMPT  │
│ tried (dep,ver);      │                         │ authors/refines     │
│ recipe library + log  │                         │ recipes; swappable  │
├───────────────────────┴─────────────────────────┴─────────────────────┤
│ [8] OBSERVABILITY   trace recipe + build results, cost; alerts        │
└───────────────────────────────────────────────────────────────────────┘
```

Filled goal-first, model last. Most of the engineering is the goal, the action boundary, and the limits. The migration itself is OpenRewrite's job; the model only writes and refines the recipe that drives it. That is the point: the loop leans on a deterministic tool, not on the model.

## The fields

### [1] Goal

Done = the upgraded project compiles and `mvn verify` is green (the full test suite passes). Deterministic, no judgment call, and everything else serves it. The recipe either gets the build to green or it doesn't … the goal is the compiler and the tests, not anyone's opinion. The build output is also the feedback the model refines against.

```python
from dataclasses import dataclass
import subprocess

@dataclass
class GateResult:
    passed: bool
    detail: str          # compiler + test output, handed to the model on a refine

def gate() -> GateResult:
    r = subprocess.run(["mvn", "-q", "verify"], capture_output=True, text=True)
    if r.returncode != 0:
        return GateResult(False, tail(r.stdout + r.stderr, 4000))   # the errors to resolve
    return GateResult(True, "green")
```

### [2] Problem

Upgrading a framework or library (Spring Boot 2→3, a major Jackson bump) renames APIs and moves config across the whole codebase. Recurring, high-volume, mechanical … loop-worthy. Deciding *whether* to adopt a major version stays human; carrying out the migration is the loop's job.

### [3] Trigger

Nightly cron plus an advisory webhook. One unit of work = one dependency at one target version. Per-dependency lock so the same one isn't upgraded twice at once; independent upgrades run in parallel.

```python
import subprocess

def discover_outdated() -> list[dict]:
    out = subprocess.run(
        ["mvn", "-q", "versions:display-dependency-updates", "-DoutputFormat=json"],
        capture_output=True, text=True, check=True,
    )
    return parse_updates(out.stdout)   # -> [{"ga": "com.x:y", "target": "3.0.0"}, ...]

def units_of_work(state) -> list[dict]:
    return [u for u in discover_outdated()
            if not state.already_tried(u["ga"], u["target"])]
```

### [4] Actions

The loop acts through OpenRewrite, not freehand edits. It may edit the recipe file (`rewrite.yml`), run the active recipes (`mvn rewrite:run`), build and test (`mvn verify`), and open a PR. It may **not** hand-edit source beyond what a recipe produces, merge, deploy, or change CI. Authority caps at *open PR*; a human merges.

Each run happens in a **fresh git worktree**: a dependency bump plus recipe-generated changes, isolated from your real checkout. The scope boundary is enforced in code, not asked for in the prompt.

```python
RECIPE_FILE = "rewrite.yml"            # the only thing the loop authors

def run_recipe(active: str) -> None:
    # OpenRewrite applies the recipe deterministically across the whole module
    subprocess.run(
        ["mvn", "-q", "org.openrewrite.maven:rewrite-maven-plugin:run",
         f"-Drewrite.activeRecipes={active}"], check=True)

def assert_in_scope(changed: list[str]) -> None:
    stray = [p for p in changed if not allowed(p)]   # build files, rewrite.yml, recipe-touched src
    if stray:
        raise PermissionError(f"out-of-scope changes: {stray}")
```

### [5] State

Which `(dependency, version)` pairs were tried and their outcome, and the recipe that made each one green. That recipe library is the loop's memory … when the same upgrade shows up in another module, the proven recipe runs first and usually lands green with no model call at all. Append-only audit log. The agent writes outcomes; only the system writes the log. (Spend ceilings live in Limits.)

```python
import sqlite3

class State:
    def __init__(self, path="loop.db"):
        self.db = sqlite3.connect(path)
        self.db.execute("CREATE TABLE IF NOT EXISTS attempts("
            "ga TEXT, version TEXT, recipe TEXT, outcome TEXT, ts TEXT,"
            "PRIMARY KEY(ga, version))")

    def already_tried(self, ga, version) -> bool:
        row = self.db.execute("SELECT outcome FROM attempts WHERE ga=? AND version=?",
                              (ga, version)).fetchone()
        return row is not None and row[0] in ("failed", "shipped")

    def known_recipe(self, ga, version):
        row = self.db.execute("SELECT recipe FROM attempts "
            "WHERE ga=? AND version=? AND outcome='shipped'", (ga, version)).fetchone()
        return row[0] if row else None

    def remember(self, ga, version, recipe, outcome, ts):
        self.db.execute("INSERT OR REPLACE INTO attempts VALUES (?,?,?,?,?)",
                       (ga, version, recipe, outcome, ts)); self.db.commit()
```

### [6] Limits

The hard stops, set before the first runaway. A recipe that almost-but-never converges would otherwise refine forever:

- **per-run budget** ≫ one upgrade can't spend more than its share.
- **max recipe refinements** ≫ the iteration cap. A few tries, then escalate.
- **no-progress / circuit breaker** ≫ if two refinements leave the *same* compile errors, the model is stuck … stop instead of burning tries.

```python
MAX_REFINES = 3              # recipe refinements before escalating
PER_RUN_TOKENS = 200_000     # hard spend ceiling for one (dependency, version)

class Budget:
    def __init__(self, cap=PER_RUN_TOKENS): self.cap, self.spent = cap, 0
    def charge(self, n): self.spent += n
    def exhausted(self) -> bool: return self.spent >= self.cap

def no_progress(prev_errors, errors) -> bool:
    return prev_errors is not None and prev_errors == errors
```

### [7] Control

After each `mvn verify`:

- **green** → open PR, record the working recipe, done.
- **red** → hand the compiler/test errors to the model to **refine the recipe**, then re-run, within the Limits caps.
- **stuck or capped** → record `failed` and **escalate** with the residual errors.
- **over budget** → stop.
- never auto-merge.

```python
def run_unit(unit, model, state, budget, now) -> str:   # MAX_REFINES, no_progress, Budget from [6]
    ga, ver = unit["ga"], unit["target"]
    recipe = state.known_recipe(ga, ver) or model.draft_recipe(ga, ver)   # reuse memory if any
    prev = None
    for _ in range(MAX_REFINES + 1):
        if budget.exhausted():
            return "stopped: budget"
        write(RECIPE_FILE, recipe); run_recipe("upgrade")   # [4] OpenRewrite does the migration
        result = gate()                                     # [1] mvn verify
        if result.passed:
            state.remember(ga, ver, recipe, "shipped", now)
            return f"shipped: {open_pr(ga, ver)}"
        if no_progress(prev, result.detail):                # [6] circuit breaker
            break
        prev = result.detail
        recipe = model.refine_recipe(recipe, result.detail) # [9] the model only steers the rule
    state.remember(ga, ver, recipe, "failed", now)
    escalate(ga, ver, errors=prev)
    return "escalated"
```

### [8] Observability

Per run: the dependency/version, the recipe and each refinement, compile + test results, iterations, cost, and the PR link or escalation. Metrics: PR-open rate, share that converged on a known recipe with no model call, refinements per upgrade, escalation rate, cost per upgrade. Alerts: a dependency failing N nights running, budget exceeded, any out-of-scope write (the `PermissionError` above).

```python
import logging
log = logging.getLogger("dep-loop")

def trace(unit, refinement, gate_result, decision, cost):
    log.info("dep-loop", extra={
        "ga": unit["ga"], "version": unit["target"],
        "refinement": refinement, "build": gate_result.detail[:200],
        "decision": decision, "cost_tokens": cost,
    })
```

### [9] Model & Prompt

The model never migrates code by hand. It reads the build failure and writes or refines an **OpenRewrite recipe** … a deterministic rule that does the migration when re-run. OpenRewrite carries it out across the whole codebase; the model only adjusts the rule. So the model stays small and swappable: a single mid-tier model is plenty, because the recipe and the compiler are the source of truth, not the model's cleverness. Swap it cheaper or stronger without touching the rest of the canvas.

The prompt gives it the dependency, the current recipe, and the build errors, and constrains its output to a recipe.

```python
SYSTEM = """You upgrade one JVM dependency by writing an OpenRewrite recipe.

You never edit source files directly. You produce or refine a recipe (rewrite.yml)
that OpenRewrite then applies deterministically. Given the target dependency and the
build errors from the last run, return an updated recipe that resolves them, composed
from existing recipes where possible (UpgradeDependencyVersion, ChangeType,
ChangeMethodName, migration recipes). Output only the recipe YAML."""

def refine_recipe(client, recipe: str, build_errors: str) -> str:
    user = (f"Current recipe:\n{recipe}\n\nBuild still failing:\n{build_errors}\n\n"
            "Return an updated recipe that resolves these errors.")
    return client.messages.create(
        model="claude-haiku-4-5-20251001",   # recipe + compiler are the source of truth; a small model is fine
        system=SYSTEM,
        messages=[{"role": "user", "content": user}],
    ).content
```

The snippets are illustrative skeletons, not a framework … the point is that the deterministic engine does the migration, and the loop refines the rule against the goal.
