# Loop Engineering Canvas

A template to think through what an agentic loop needs before you build it. It doesn't build anything … it just helps you sort out what you're trying to do, and what could go wrong, while changing your mind is still cheap.
## The idea

Stop using the agent to solve the problem; build the system that solves it … the thing that builds the thing. At the heart is the **goal**: a definition of done that something other than you can check. Everything else … when it runs, what it may touch, when it stops … exists to hit that goal reliably. The model and prompt are the last, most swappable part: the engine you drop in once the harness around it holds.

```text
trigger ─▶ state ─▶ agent ─▶ effect ─▶ gate ─▶ control ─┐
   ▲                                                    │
   └──────────── state update, next run ◀───────────────┘
```

Read it left to right, but it cycles. A **trigger** starts a run; the loop loads its **state** (what earlier runs left behind) and the **agent** acts, producing an **effect**. The **gate** checks that effect against the **goal**; **control** either loops back with a new prompt or stops … continue, retry, repair, escalate, stop, ship. It repeats until the goal is met or a limit fires.

## The canvas

The whole loop fits on one page … enough to sketch it, challenge it, and find the gaps before you build. Position carries meaning; read the canvas as a map, not the definitions (those are in the table below).

```text
┌─────────────────────────────────────────────────────────────────────┐
│ [2] PROBLEM   what recurring work justifies a loop, what stays human│
├─────────────────────┬─────────────────────────┬─────────────────────┤
│ [3] TRIGGER         │ [1] GOAL                │ [6] LIMITS          │
│ how work enters,    │                         │ budget, max iters,  │
│ unit of work        │ what "done" means       │ circuit breaker     │
│                     │ as a check anyone       │                     │
│ [4] ACTIONS         │ or anything can run     │ [7] CONTROL         │
│ read/write/tools,   │                         │ retry, repair,      │
│ scope, isolation    │ everything serves       │ escalate, stop, ship│
│                     │ this                    │                     │
│ [5] STATE           │                         │ [9] MODEL & PROMPT  │
│ what persists       │                         │ the swappable       │
│ between runs        │                         │ engine, chosen last │
│                     │                         │                     │
├─────────────────────┴─────────────────────────┴─────────────────────┤
│ [8] OBSERVABILITY   trace, metrics, alerts to watch every run       │
└─────────────────────────────────────────────────────────────────────┘
```

## The fields

Each field, the question it answers, and what to watch for.

| # | Field | The question it answers | Watch for |
|---|-------|-------------------------|-----------|
| 1 | **Goal** | What outcome does the whole loop exist to produce, and how is *done* checked? | The heart … everything else serves this, so define it first. Write it as a check something other than you can run: not "handle the ticket" but "classify into one of five, draft a reply citing the policy section, escalate if confidence < 0.7". A weak goal ships work that looks done but misses intent, every run. Build the check before you trust anything to hit it. |
| 2 | **Problem** | What recurring work justifies a loop, and what stays human-owned? | If you can't name one repeating unit of work, you don't need a loop yet. |
| 3 | **Trigger** | How does work enter, and what is one unit of work? | Schedule / event / queue / manual / watcher. Set the concurrency policy: single-flight, per-item lock, or parallel. |
| 4 | **Actions** | What may the agent read, write, call, and decide on its own, and where does it run? | Read/write scope, allowed/forbidden tools, network, authority (draft → comment → open PR → merge → deploy), rollback, and isolation: run real actions in a sandbox / isolated worktree, since every file or page it reads is untrusted input. The boundary between useful autonomy and uncontrolled behavior … enforce it in code, not just the prompt. |
| 5 | **State** | What persists between runs, and who may write it? | Cursor, run state, work memory, failure memory, human decisions, audit log. Be explicit about what's append-only, what the agent may write, and what only the system may write. |
| 6 | **Limits** | When must the loop stop, no matter what it thinks? | Token/cost budget per run and per day, max iterations, circuit-breaker on token-velocity, no-progress detection. Design these in from the start, not after the incident … the classic failure is a loop calling a broken tool hundreds of times or burning the budget before anyone looks. |
| 7 | **Control** | After each result, which move is allowed? | Continue, retry (with feedback), repair, escalate, pause, stop done, stop failed, roll back, ship. Plus what triggers escalate vs rollback. (The hard stops live in Limits.) |
| 8 | **Observability** | Can you reconstruct any run after the fact? | Trace: trigger, unit of work, tools called, things touched, goal/gate results, control decisions, cost, approvals, final status. Metrics: success rate, retries, escalation rate, cost per run. Alerts on repeated failure, budget exceeded, policy violation. |
| 9 | **Model & Prompt** | *Last:* which model and prompt run inside everything above? | The swappable engine … "the model is the engine, the harness is the car." Pick it last and expect to change it: model and tier, the prompt, single vs multi agent, tools, and the inner reasoning loop (reason-act-observe / reflection / evaluator-optimizer). The prompt is not the only place safety, permissions, state, and the goal live. |

## The order

Work the numbers in order. It's not the order a run executes … it's outside-in: pin down what success is, then build the harness around it, then drop in the engine.

| Order | Fill in | In one line |
|-------|---------|-------------|
| 1 | **Goal** | the definition of done everything else serves … start here |
| 2 | **Problem** | what recurring work justifies a loop at all, and what stays human |
| 3 | **Trigger** | how work enters, and what one unit of work is |
| 4 | **Actions** | what it may touch, and where it runs |
| 5 | **State** | what it remembers between runs |
| 6 | **Limits** | when it must stop, regardless |
| 7 | **Control** | what to do after each result |
| 8 | **Observability** | what you watch on every run |
| 9 | **Model & Prompt** | the swappable engine … chosen last, once the harness around it holds |

Position is a second lens: the **goal** sits at the center because everything serves it, **problem** frames the page from the top, **observability** records every run along the bottom.

## The examples

- [jvm dependency upgrade loop](examples/jvm-dependency-upgrade-loop.md) ≫ OpenRewrite migrates the breaking changes; the model only steers the recipe.
- [salesforce incident ticket loop](examples/salesforce-incident-ticket-loop.md) ≫ runbooks mitigate the incident; the model only picks the next one.
