# coverage gap test generation loop

Close the coverage gaps in a codebase without hand-writing every missing test. The deterministic work is done by the test runner and a mutation-testing tool ([PIT](https://pitest.org/), [Stryker](https://stryker-mutator.io/), mutmut), not the model: find an under-tested target, write a test, and keep it only if it catches a fault the existing suite misses. The model's only job is to author and refine the test against the mutants that survive. The trap this loop is built to avoid: line coverage is a weak gate … a suite can hit 100% coverage and still catch almost nothing (one study measured a 4% mutation score at 100% coverage). So the gate is mutation score, not coverage. That is the nature of a good loop … a reliable tool decides what counts, and the model only steers toward it.

```text
┌───────────────────────────────────────────────────────────────────────┐
│ [2] PROBLEM   coverage gaps hide regressions; good tests are toil     │
├───────────────────────┬─────────────────────────┬─────────────────────┤
│ [3] TRIGGER           │ [1] GOAL                │ [6] LIMITS          │
│ low-coverage CI/scan; │                         │ budget; max tries;  │
│ one target or gap     │ a test that builds,     │ mutant cap; flaky   │
│                       │ passes and kills a      │ guard; no-progress  │
│ [4] ACTIONS           │ mutant the old suite    │ [7] CONTROL         │
│ write tests only;     │ missed                  │ kills mutant->PR;   │
│ run cov + mutation;   │                         │ cov-only->reject;   │
│ open PR; sandbox      │ (the success check)     │ flaky->drop; else…  │
│ [5] STATE             │                         │ [9] MODEL & PROMPT  │
│ tried targets +       │                         │ writes tests from   │
│ surviving mutants     │                         │ live mutants; swap  │
├───────────────────────┴─────────────────────────┴─────────────────────┤
│ [8] OBSERVABILITY   trace coverage + mutation deltas, cost; alerts     │
└───────────────────────────────────────────────────────────────────────┘
```

## The fields

### [1] Goal

Done = the new test compiles, passes reliably across repeated runs, and **kills at least one mutant the existing suite already missed**. Raising line coverage is not enough … a test can execute a branch without asserting anything about it. The mutation tool is the success check: it makes small changes to the production code and asks whether the new test now fails. If it does, the test detects a real fault; if coverage went up but no new mutant died, the test proved nothing and is rejected. The surviving mutants are also the feedback the model refines against.

```python
from dataclasses import dataclass

@dataclass
class GateResult:
    passed: bool
    detail: str            # surviving mutants + build/test output, handed back to the model

def gate(target, candidate) -> GateResult:
    if not builds(candidate):                             # compiles at all
        return GateResult(False, build_errors(candidate))
    if not passes_reliably(candidate, runs=5):            # not flaky: green N times, no source change
        return GateResult(False, "flaky or failing")
    killed = newly_killed_mutants(target, candidate)      # mutants the OLD suite missed, this test kills
    if not killed:                                        # coverage may have risen; fault-detection did not
        return GateResult(False, still_surviving(target)) # Goodhart guard: coverage-only is rejected
    return GateResult(True, f"kills {len(killed)} new mutants")
```

### [2] Problem

Critical or freshly-changed code carries branches no test ever exercises, and that is where regressions hide. Finding the gaps is cheap; writing tests that actually catch faults is tedious, high-volume, and mechanical … loop-worthy. But the obvious metric lies: a suite can reach 100% coverage and still catch almost nothing (a measured 4% mutation score at 100% coverage). Deciding *what* deserves hardening (which modules are critical) stays human or policy-driven; writing and proving the tests is the loop's job. Humans own the merge.

### [3] Trigger

The CI coverage report on a pull request, or a scheduled scan of hot/critical modules. One unit of work = one target (a class or function) under the coverage floor on lines that changed. Per-target lock so the same one isn't worked twice at once; independent targets run in parallel.

```python
COV_FLOOR = 0.8

def discover_gaps(coverage_report, changed_files) -> list[dict]:
    # a target is in scope if changed code sits under the coverage floor
    return [{"target": t.id, "uncovered": t.uncovered_lines}
            for t in coverage_report.targets
            if t.file in changed_files and t.line_coverage < COV_FLOOR]

def units_of_work(state) -> list[dict]:
    return [g for g in discover_gaps(load_coverage(), changed_files())
            if not state.done(g["target"])]
```

### [4] Actions

The loop acts through the test framework, the coverage tool, and the mutation tool … never freehand source edits. It may write test files, run the suite, run coverage, run mutation testing, and open a PR. It may **not** touch production source (that would let it change behavior to make a test pass, or weaken the very code under test), delete or relax existing tests, merge, or deploy. Authority caps at *open PR*; a human merges.

Each run happens in a **fresh git worktree**. The mutation tool rewrites production code, but only inside the sandbox and only transiently … the real tree stays clean. The test-only write scope is enforced in code, not asked for in the prompt.

```python
def assert_test_only(changed: list[str]) -> None:
    stray = [p for p in changed if not is_test_file(p)]   # production source is off-limits
    if stray:
        raise PermissionError(f"loop tried to edit non-test files: {stray}")

def run_mutation(target, candidate) -> set:
    # the mutation tool mutates production code IN THE SANDBOX only, runs the suite,
    # and returns which mutants SURVIVED (were not caught). the real tree is never changed.
    return mutation_tool.surviving(target, extra_tests=candidate)   # e.g. PIT / Stryker / mutmut
```

### [5] State

Which targets were attempted and their outcome, and the **mutants still surviving** for each one. That surviving-mutant set is the loop's memory … it is the exact shape of the fault gap, so the next attempt is prompted to aim straight at it instead of rediscovering it. Append-only audit log. The agent writes outcomes; only the system writes the log. (Spend ceilings live in Limits.)

```python
import json, sqlite3

class State:
    def __init__(self, path="testgen.db"):
        self.db = sqlite3.connect(path)
        self.db.execute("CREATE TABLE IF NOT EXISTS targets("
            "id TEXT PRIMARY KEY, surviving TEXT, outcome TEXT, ts TEXT)")

    def done(self, target) -> bool:
        row = self.db.execute("SELECT outcome FROM targets WHERE id=?", (target,)).fetchone()
        return row is not None and row[0] in ("shipped", "abandoned")

    def surviving(self, target) -> list:                  # the fault gap = the loop's memory
        row = self.db.execute("SELECT surviving FROM targets WHERE id=?", (target,)).fetchone()
        return json.loads(row[0]) if row else []

    def remember(self, target, surviving, outcome, ts):
        self.db.execute("INSERT OR REPLACE INTO targets VALUES (?,?,?,?)",
                        (target, json.dumps(surviving), outcome, ts)); self.db.commit()
```

### [6] Limits

The hard stops, set before the first runaway. Mutation testing is expensive, and a test that raises coverage but never kills a mutant would otherwise refine forever:

- **per-run budget** ≫ one target can't spend more than its share.
- **max refinements** ≫ the iteration cap. A few tries, then escalate.
- **mutant cap** ≫ sample a bounded, targeted set of mutants per target rather than an exhaustive run (Meta's ACH deliberately generates relatively few, concern-specific mutants).
- **flaky / no-progress breaker** ≫ if two refinements leave the *same* mutants alive, the model is stuck … stop instead of burning tries.

```python
MAX_REFINES    = 3           # test refinements before escalating
MUTANT_CAP     = 40          # a bounded, targeted sample; mutation testing is expensive
PER_RUN_TOKENS = 150_000     # hard spend ceiling for one target

class Budget:
    def __init__(self, cap=PER_RUN_TOKENS): self.cap, self.spent = cap, 0
    def charge(self, n): self.spent += n
    def exhausted(self) -> bool: return self.spent >= self.cap

def no_progress(prev_surviving, surviving) -> bool:
    return prev_surviving is not None and prev_surviving == surviving   # killed nothing new
```

### [7] Control

After each gate:

- **kills a new mutant** (builds + reliable + fault-detection improved) → open PR, record, done.
- **coverage up but no new mutant killed** → hand the surviving mutants to the model to **refine the test**, within the caps. Never accept a coverage-only gain.
- **flaky** → discard the candidate.
- **stuck or capped** → record `abandoned` and **escalate** with the residual mutants.
- **over budget** → stop.
- never auto-merge, never weaken an existing test.

```python
def run_unit(unit, state, budget, now) -> str:      # MAX_REFINES, no_progress, Budget from [6]
    target = unit["target"]
    prev = None
    candidate = draft_test(target, state.surviving(target))   # aim at the known fault gap [5][9]
    for _ in range(MAX_REFINES + 1):
        if budget.exhausted():
            return "stopped: budget"
        assert_test_only(files_touched(candidate))            # [4] production source is off-limits
        result = gate(target, candidate)                      # [1] build + reliable + kills new mutant
        if result.passed:
            state.remember(target, run_mutation(target, candidate), "shipped", now)
            return f"shipped: {open_pr(target, candidate)}"
        if no_progress(prev, result.detail):                  # [6] coverage may rise, mutants don't die
            break
        prev = result.detail
        candidate = refine_test(candidate, result.detail)     # [9] the model only steers the assertions
    state.remember(target, prev, "abandoned", now)
    escalate(target, surviving=prev)
    return "escalated"
```

### [8] Observability

Per run: the target, each attempt, the coverage delta, the mutation-score delta, which mutants died, flaky rejects, and the PR link or escalation. Metrics: PR-open rate, new mutants killed per PR, and the **coverage-up-but-mutation-flat rejection rate** … the single most telling number here, because a rising count means the model is learning to game coverage and the gate is holding the line. Also refinements per target, escalation rate, cost per target. Alerts: a target failing repeatedly, budget exceeded, any out-of-scope write (the `PermissionError` above).

```python
import logging, collections
log = logging.getLogger("testgen-loop")
metrics = collections.Counter()                       # cheap in-process metrics; ship to your TSDB

def trace(target, attempt, gate_result, decision, cost):
    log.info("testgen-loop", extra={                  # one structured record per attempt
        "target": target, "attempt": attempt,
        "result": gate_result.detail[:200], "decision": decision, "cost_tokens": cost,
    })
    metrics[decision] += 1                            # shipped / coverage-only-rejected / escalated
    metrics["cost_tokens"] += cost

def check_alerts():
    runs = metrics["shipped"] + metrics["escalated"]
    if runs and metrics["escalated"] / runs > 0.5:           # most targets not improving detection
        page("testgen-loop: escalation rate > 50%")
    if metrics["cost_tokens"] > DAILY_TOKEN_BUDGET:          # budget guard (see [6])
        page("testgen-loop: over daily token budget")
```

### [9] Model & Prompt

The model never decides whether a test is any good … the mutation tool and the runner do. Its job is narrow: given the target and the list of surviving mutants, write assertions that fail against those mutants and pass against the correct code (mutation feedback in the prompt is the MUTGEN insight). Because the gate is the source of truth, the model stays small and swappable: a single mid-tier model is plenty. Swap it cheaper or stronger without touching the rest of the canvas.

Wire the model behind one adapter so the loop is model-agnostic. The system prompt pins the job and the output format; the user message hands over the current test and the mutants still alive; a cheap compile check guards the result before the mutation run re-checks it for real.

```python
MODEL = "your-mid-tier-model"        # the mutation tool is the source of truth; small is fine

def call_model(model: str, system: str, user: str) -> str:
    # the loop's ONLY provider-specific line; return the completion text
    return llm.complete(model=model, system=system, user=user)   # drop in your SDK

SYSTEM = """You write one unit test that catches a specific fault.
You never edit production code: you return only a test file. You are given the target under
test and a list of SURVIVING MUTANTS … small code changes the current suite fails to catch.
Write assertions that FAIL against each mutant and PASS against the correct code.
Do not write assertion-free or trivially-passing tests. Output only the test file, no prose."""

def refine_test(candidate: str, surviving: str) -> str:
    user = f"Current test:\n{candidate}\n\nStill-surviving mutants to kill:\n{surviving}"
    out = strip_code_fences(call_model(MODEL, SYSTEM, user))   # models love to wrap code in ```
    assert compiles(out), "model returned an uncompilable test"   # cheap guard; mutation is the real gate
    return out

def draft_test(target: str, surviving: str) -> str:            # the first pass
    return refine_test(f"# test for {target}", surviving or "(coverage gap; no mutants yet)")
```

The snippets are illustrative skeletons, not a framework … the point is that the mutation tool and the test runner decide whether a test is worth anything, and the loop only refines the assertions against the goal.

## Resources

- [Automated Unit Test Improvement using LLMs at Meta (TestGen-LLM)](https://arxiv.org/abs/2402.09171) … Assured LLMSE: keep a generated test only if it measurably improves the suite. Reels/Stories: 75% built, 57% passed reliably, 25% raised coverage; 73% of recommendations accepted.
- [Mutation-Guided LLM-based Test Generation at Meta (ACH)](https://arxiv.org/abs/2501.12862) … generate targeted mutants, then tests that kill them; 10,795 Kotlin classes, 73% of tests accepted.
- [Mutation-Guided Unit Test Generation with an LLM (MUTGEN)](https://arxiv.org/abs/2506.02954) … coverage is a weak fault-detection signal; some suites hit 100% coverage at only 4% mutation score; feed mutation feedback into the prompt.
- [Qodo Cover-Agent](https://www.qodo.ai/blog/we-created-the-first-open-source-implementation-of-metas-testgen-llm/) … an open-source implementation of TestGen-LLM's assured, improvement-gated approach.
- [PIT (pitest)](https://pitest.org/) … the deterministic mutation-testing engine for the JVM.
- [Stryker Mutator](https://stryker-mutator.io/) … mutation testing for JavaScript/TypeScript, C#, and Scala.
