# test factory loop

Backfill a test suite onto a legacy codebase without a human writing … or reviewing … individual tests: a dark factory for tests, in the lights-out-manufacturing sense of a plant that runs with nobody on the floor. Naive LLM test generation fails at scale in one specific way: the tautological test that passes, lifts coverage, and asserts nothing. The fix is deterministic. A mutation engine injects a small synthetic fault (a *mutant*: `>=` flipped to `>`, a call deleted, a constant changed) and a generated test is accepted only if it **fails on the mutated code and passes on the real code** … a verified kill. The engine defines the work and verifies the result; the model only drafts the test. Meta runs the pipeline behind this in production as [ACH](https://engineering.fb.com/2025/02/05/security/revolutionizing-software-testing-llm-powered-bug-catchers-meta-ach/), which generated 9,095 mutants across 10,795 Kotlin classes and 571 privacy-hardening tests, 73% of which engineers accepted.

```text
┌───────────────────────────────────────────────────────────────────────┐
│ [2] PROBLEM   legacy code nobody will ever hand-write tests for       │
├───────────────────────┬─────────────────────────┬─────────────────────┤
│ [3] TRIGGER           │ [1] GOAL                │ [6] LIMITS          │
│ nightly mutation      │                         │ tries per mutant;   │
│ sweep; unit = one     │ the new test kills its  │ budget per module;  │
│ surviving mutant      │ mutant, passes on HEAD, │ suite-runtime cap   │
│ [4] ACTIONS           │ isn't flaky, suite      │ [7] CONTROL         │
│ write test files only;│ stays green             │ kill->merge; no     │
│ run suite + mutation  │                         │ kill->discard;      │
│ engine; sandbox       │ (all machine-checked)   │ survivor->park      │
│ [5] STATE             │                         │ [9] MODEL & PROMPT  │
│ mutant inventory;     │                         │ drafts one test per │
│ scores; equivalents   │                         │ mutant; swappable   │
├───────────────────────┴─────────────────────────┴─────────────────────┤
│ [8] OBSERVABILITY   mutation score per module, kill rate, cost/kill   │
└───────────────────────────────────────────────────────────────────────┘
```

## The fields

### [1] Goal

Done for one unit of work: the generated test **kills its target mutant** (fails with the fault applied), passes on HEAD, passes N reruns, and the whole suite stays green. Every check is binary and machine-run. "The test passes" alone proves nothing … a test that can't fail is dead weight; the kill is the proof of value. Done for the codebase: mutation score per module, trending up. Per draft the gate runs the new test and the module's tests; the whole suite runs once, guarding the night's merge batch … a per-draft full-suite run doesn't scale to a legacy suite. One honest limit: a kill proves the test can *detect* the fault, not that HEAD's behavior is *correct*. On legacy code this is characterization testing … a live bug gets pinned as expected behavior, with a green gate. The fault-class choice in [2] and the sampled audit in [8] are the mitigations, not the gate.

```python
from dataclasses import dataclass
import subprocess

@dataclass
class GateResult:
    passed: bool
    detail: str                     # the verdict, handed to the model on a redraft

def pytest_ok(args: list[str]) -> bool:
    return subprocess.run(["pytest", "-q", *args]).returncode == 0

def gate(test_file: str, mutant, reruns: int = 5) -> GateResult:
    if not pytest_ok([test_file]):
        return GateResult(False, "red on HEAD")            # broken test
    with applied(mutant):                                  # harness patches the fault in, never the model
        if pytest_ok([test_file]):
            return GateResult(False, "mutant survived")    # tautology: passes but detects nothing
    if not all(pytest_ok([test_file]) for _ in range(reruns)):
        return GateResult(False, "flaky")
    if runtime_ms(test_file) > PER_TEST_MS:                # [6] a slow test taxes every CI run forever
        return GateResult(False, "too slow")
    if not pytest_ok([tests_for(mutant.module)]):          # module-scoped; the full suite guards the merge batch
        return GateResult(False, "breaks the module's tests")
    return GateResult(True, "kill verified")
```

### [2] Problem

Anyone handed "get this legacy module to 80% coverage" knows how it goes: the tickets lose to feature work for years, and when they do get done the KPI corrupts them … coverage measures what tests *execute*, not what they can *detect*, so assert-thin tests raise the number and catch nothing. Recurring unit of work: one fault the suite provably can't catch. Humans own which modules matter and which fault classes to harden against (Meta pointed ACH at privacy regressions); producing and proving the tests is the loop's job.

### [3] Trigger

The nightly mutation sweep is the order book. The engine (mutmut, PIT, Stryker) mutates target modules and runs the existing suite; every mutant that survives marks a hole … one work order. Merges trigger an incremental sweep over changed files only, because full-repo mutation runs are expensive; a sampled backlog drains the rest over time. Parallel across modules, one lock per module so two units never write the same test file.

```python
def order_book(state) -> list:
    paths = state.changed_since_last_sweep() or state.backlog_sample()
    return [m for m in mutation_sweep(paths)               # survivors only
            if not state.known_equivalent(m) and not state.exhausted(m)]
```

### [4] Actions

The loop may write **test files only**, run the suite and the mutation engine in a fresh worktree, and merge on a green gate. That last authority is the dark-factory step … past open-PR, straight to merge … and it is defensible only because the write scope is enforced in code and the gate is deterministic. No human reviews an individual test; the gate is the reviewer. A stray write to production code is a `PermissionError`, not a judgment call. Worth naming: Meta stopped one step short … ACH's kill-verified tests still went to engineers, who accepted 73% of them. Merge-on-green is the lights-out endgame; start at open-PR and raise authority once the audit sample in [8] stays clean.

```python
TEST_DIRS = ("tests/",)

def assert_in_scope(changed: list[str]) -> None:
    stray = [p for p in changed if not p.startswith(TEST_DIRS)]
    if stray:
        raise PermissionError(f"out-of-scope changes: {stray}")   # production code is untouchable
```

### [5] State

The mutant inventory is the factory's ledger: each mutant under a stable key … a hash of file, enclosing function, and mutation operator, not the line number, which shifts with every edit above it … so nightly resweeps reopen records instead of duplicating them. Each record carries its outcome: `killed` and by which test, `parked` after exhausted tries, `equivalent` (behaviorally identical to the original, unkillable, never retried). Every merged test traces back to the fault it provably catches … an audit trail hand-written suites don't have. Only the harness writes the ledger.

```python
import sqlite3

class State:
    def __init__(self, path="factory.db"):
        self.db = sqlite3.connect(path)
        self.db.execute("CREATE TABLE IF NOT EXISTS mutants("
            "id TEXT PRIMARY KEY, module TEXT, diff TEXT,"
            "outcome TEXT, test TEXT, tries INT)")   # outcome: killed | parked | equivalent

    def known_equivalent(self, m) -> bool:
        row = self.db.execute("SELECT outcome FROM mutants WHERE id=?", (m.id,)).fetchone()
        return row is not None and row[0] == "equivalent"

    def exhausted(self, m) -> bool:
        row = self.db.execute("SELECT outcome, tries FROM mutants WHERE id=?", (m.id,)).fetchone()
        return row is not None and (row[0] == "killed" or row[1] >= MAX_TRIES)

    def record(self, m, outcome, test=None, tries=0):
        self.db.execute("INSERT OR REPLACE INTO mutants VALUES (?,?,?,?,?,?)",
                        (m.id, m.module, m.diff, outcome, test, tries)); self.db.commit()
```

### [6] Limits

- **tries per mutant** ≫ a few drafts, then park. Some mutants are unkillable (equivalent) and no amount of drafting changes that.
- **budget per module, and per sweep** ≫ a module ceiling so one pathological module can't starve the night; a sweep ceiling as the backstop. The backlog is patient.
- **suite-runtime cap** ≫ the factory must not bloat CI. A slow test taxes every future run; the gate rejects it like a failed kill.

```python
MAX_TRIES     = 3           # drafts per mutant, then park
PER_TEST_MS   = 500         # per-test runtime cap, enforced in the gate [1]
MODULE_TOKENS = 300_000     # ceiling per module … one bad module can't starve the sweep
NIGHT_TOKENS  = 5_000_000   # ceiling for the whole night, the backstop

class Budget:
    def __init__(self, cap): self.cap, self.spent = cap, 0
    def charge(self, n): self.spent += n
    def exhausted(self) -> bool: return self.spent >= self.cap
```

### [7] Control

After each gate:

- **kill verified** → merge, record which test kills which mutant, done. No human in the path. Merges land as a nightly batch behind one full-suite run; a red batch gets bisected.
- **passes but no kill** → **discard** … the anti-tautology move. A test that proves nothing never ships, no matter how plausible it reads.
- **red on HEAD / flaky / breaks the module's tests / too slow** → discard, redraft with the verdict as feedback.
- **tries exhausted** → park the mutant. Parked mutants are signal, not noise: often equivalent mutants or dead, unreachable code. A sampled queue goes to a human (Meta layers an LLM equivalence detector on top; the sampled queue is the lean version). Parking isn't permanent: a model swap [9] or a human triage releases the mutant for another round.

```python
def run_unit(mutant, state, budget) -> str:    # budget = the module's Budget(MODULE_TOKENS) [6]
    feedback = None
    for tries in range(1, MAX_TRIES + 1):
        if budget.exhausted():
            return "stopped: budget"
        test_file = write_test(draft_test(mutant, feedback))   # [9] the model's whole job
        assert_in_scope(changed_files())                       # [4]
        result = gate(test_file, mutant)                       # [1]
        if result.passed:
            merge(test_file)                                   # enqueues on the nightly batch [7]
            state.record(mutant, "killed", test_file, tries)
            return "merged"
        discard(test_file); feedback = result.detail
    state.record(mutant, "parked", tries=MAX_TRIES)            # equivalent? dead code? humans sample these
    return "parked"
```

### [8] Observability

Per unit: the mutant diff, each draft, gate verdicts, the decision, cost. Metrics: mutation score per module over time (the factory's output metric), kill rate per draft, discard rate, flake rate of merged tests (should be ~0), cost per kill. Alerts: kill rate collapsing (a model or harness regression), suite runtime growth, any out-of-scope write. Humans read the dashboard and audit a random sample of merged tests … never every test.

```python
import logging, collections
log = logging.getLogger("test-factory")
metrics = collections.Counter()

def trace(mutant, tries, decision, cost):
    log.info("test-factory", extra={                   # one structured record per unit
        "mutant": mutant.id, "module": mutant.module,
        "tries": tries, "decision": decision, "cost_tokens": cost,
    })
    metrics[decision] += 1

def check_alerts():
    done = metrics["merged"] + metrics["parked"]
    if done and metrics["merged"] / done < 0.3:        # kill rate collapsed: model or harness broke
        page("test-factory: kill rate < 30%")
    if suite_runtime_ms() > SUITE_RUNTIME_CAP_MS:      # the factory is bloating CI (see [6])
        page("test-factory: suite runtime over cap")
```

### [9] Model & Prompt

The model never picks what to test (the sweep does), never judges test quality (the gate does), never touches production code (scope does). Its whole job: given the source and one mutant diff, return one test that passes on the original and fails on the mutant. A narrow, mechanically checked task … a mid-tier model is plenty, and swapping it changes nothing else on the canvas. On a redraft it gets the gate verdict, nothing more.

```python
MODEL = "your-mid-tier-model"        # the gate is the source of truth; small is fine

def call_model(model: str, system: str, user: str) -> str:
    return llm.complete(model=model, system=system, user=user)   # the one provider-specific line

SYSTEM = """You write one unit test that catches one specific injected fault.
Given a source file and a mutant diff, return a single test that PASSES on the
original code and FAILS with the mutant applied. Assert observable behavior;
do not mock the code under test; do not copy implementation details into asserts.
Output only the test code, no prose."""

def draft_test(mutant, feedback: str | None) -> str:
    user = (f"Source:\n{mutant.source}\n\nMutant diff:\n{mutant.diff}\n\n"
            f"Last attempt: {feedback or '(first try)'}")
    out = strip_code_fences(call_model(MODEL, SYSTEM, user))
    assert compiles(out), "not even valid code"        # cheap guard; the kill is the real gate
    return out
```

The snippets are illustrative skeletons, not a framework. The mutation engine defines the work and verifies the kill; the model only drafts the test.

## Resources

- [Meta: LLM-powered bug catchers (ACH)](https://engineering.fb.com/2025/02/05/security/revolutionizing-software-testing-llm-powered-bug-catchers-meta-ach/) … mutation-guided test generation in production.
- [Mutation-Guided LLM-based Test Generation at Meta](https://arxiv.org/abs/2501.12862) … the ACH paper: 9,095 mutants, 571 hardening tests.
- [Automated Unit Test Improvement using LLMs at Meta](https://arxiv.org/abs/2402.09171) … TestGen-LLM, the assured-generation predecessor: keep only tests that build, pass, and aren't flaky.
- [PIT](https://pitest.org/) … JVM mutation engine.
- [mutmut](https://mutmut.readthedocs.io/) … Python mutation engine.
- [Stryker](https://stryker-mutator.io/) … JS/TS/C#/Scala mutation engine.
- [qodo-cover](https://github.com/qodo-ai/qodo-cover) … open-source coverage-gated test generation loop (archived, readable).
- [The dark factory pattern](https://hackernoon.com/the-dark-factory-pattern-moving-from-ai-assisted-to-fully-autonomous-coding) … the lights-out framing: humans manage gates and metrics, not output.
