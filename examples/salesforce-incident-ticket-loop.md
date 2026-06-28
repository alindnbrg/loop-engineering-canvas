# salesforce incident ticket loop

Keep business-critical systems running 24/7 when incidents are tracked as Salesforce Cases. When an alert opens a Case, the loop works it like a good on-call engineer: read the Case, pull the service's telemetry, run the runbook most likely to clear the breach, verify the service against its SLO, and either resolve and document the Case or hand it to a human. The deterministic work is the runbooks, the RBAC scope, and the SLO check, not the model … the model only investigates and picks the next runbook.

This mirrors the mid-2026 production pattern: *investigate-and-recommend with human-approved execution*, e.g. [Salesforce Agentforce IT Service](https://www.salesforce.com/news/stories/agentforce-it-service-announcement/) on the Case layer and an SRE agent like [AWS DevOps Agent](https://aws.amazon.com/blogs/devops/automating-incident-investigation-with-aws-devops-agent-and-salesforce-mcp-server/) reading and writing the Case over MCP. Full autonomy is not the bar … on IBM's ITBench the best models resolve only ~11% of SRE incidents end to end, so a good loop scopes itself to known incident classes with runbooks and escalates the rest.

```text
┌───────────────────────────────────────────────────────────────────────┐
│ [2] PROBLEM   24/7 critical systems; incident cases worked fast       │
├───────────────────────┬─────────────────────────┬─────────────────────┤
│ [3] TRIGGER           │ [1] GOAL                │ [6] LIMITS          │
│ alert opens a case;   │                         │ SLA time budget;    │
│ one case per run      │ SLO/health signal       │ confidence; max     │
│                       │ back green, case        │ steps; kill switch  │
│ [4] ACTIONS           │ documented + closed     │ [7] CONTROL         │
│ read case via MCP,    │                         │ low-risk: auto;     │
│ run runbooks; risky   │ (the success check)     │ risky: approve;     │
│ needs approval; RBAC  │                         │ verify SLO, else esc│
│ [5] STATE             │                         │ [9] MODEL & PROMPT  │
│ case timeline; CMDB   │                         │ investigates, picks │
│ runbooks tried + lib  │                         │ runbook; swappable  │
├───────────────────────┴─────────────────────────┴─────────────────────┤
│ [8] OBSERVABILITY   case trace + why/how, MTTR, SLA; alerts           │
└───────────────────────────────────────────────────────────────────────┘
```

Filled goal-first, model last. The engineering is in the goal (the SLO health signal, not the model's say-so), the action boundary (runbooks scoped by RBAC, risky steps behind human approval, Salesforce writes via MCP), and the limits (mitigate within SLA or page a human). The model only investigates and picks the next runbook. That is the point: lean on deterministic runbooks and governance, not on the model.

## The fields

### [1] Goal

Done = the service's SLO/health signal is back green **and** the Case is documented (root cause + actions in the Activity feed) and closed. Deterministic, no judgment call … the SLO is the source of truth, not the model's hunch that it "looks fixed". Verifying mitigation against the health signal is what separates a real resolution from a hopeful one, and it is what the loop checks after every runbook.

```python
from dataclasses import dataclass

@dataclass
class GateResult:
    passed: bool
    detail: str           # what's still breaching, handed to the model

def gate(case) -> GateResult:
    health = check_slo(case.service)             # the SLO signal, not an opinion
    if not health.green:
        return GateResult(False, health.summary)            # still breaching
    if not documented(case):                                # root cause + actions logged
        return GateResult(False, "service green but case not documented")
    return GateResult(True, "SLO green and case documented")
```

### [2] Problem

Business-critical systems must run 24/7, and incidents arrive as Cases at all hours. First-response is largely runbook-driven toil: correlate the alerts, pull telemetry, run known diagnostics and known fixes (restart, scale, reroute, roll back), document, escalate if stuck. The loop handles the routine, known-class incidents; deciding novel architecture or accepting risk on a major incident stays human (the same boundary Salesforce draws by gating major-incident promotion behind a human approval).

### [3] Trigger

An alert (CloudWatch, Datadog, PagerDuty, …) opens or routes to a Salesforce incident Case. One unit of work = one Case. A per-service lock so two runbooks don't fight the same system; unrelated Cases run in parallel.

```python
# Unit of work = one incident Case. An alert opens/routes it; one in-flight
# response per service so two runbooks don't fight the same system.
def on_case(case) -> dict:
    return {"case_id": case.Id, "service": case.Service__c,
            "severity": case.Severity__c, "opened_at": case.CreatedDate}
# per-service lock; unrelated cases run in parallel
```

### [4] Actions

The loop reads the Case over the Salesforce **MCP** server (`soql_query`) and writes findings back to the Activity feed (`create_sobject_record`) … the actual AWS-DevOps-Agent-to-Agentforce pattern. Remediation runs through approved **runbooks**, never ad-hoc shell. Permissions are scoped per step, the way production AI-SRE does it: read-only diagnostics auto, a pod restart notifies, a DB failover or rollback needs explicit human approval. RBAC/IAM bounds the blast radius to the affected service (AWS calls this an Agent Space), and the kill switch lives at the orchestration layer, not in the prompt.

```python
# Read the case over Salesforce MCP; remediate only through approved runbooks.
# Per-step permissions: diagnostics auto, restart notifies, failover needs a human.
RISKY = {"failover", "rollback", "scale_down", "drain_node"}

def read_case(mcp, case_id):
    return mcp.call("soql_query", q=f"SELECT Subject,Service__c,... FROM Case WHERE Id='{case_id}'")

def post_findings(mcp, case_id, note):
    mcp.call("create_sobject_record", sobject="CaseComment",
             fields={"ParentId": case_id, "CommentBody": note})    # the Activity feed

def run_runbook(case, runbook, params):
    if runbook.id in RISKY and not approved(case, runbook, params):
        raise NeedsApproval(runbook, params)                       # human decides risky steps
    assert_in_scope(case.service, runbook.targets)                 # RBAC blast radius
    return runbook.execute(case.service, params)
```

### [5] State

The Salesforce Case is the system of record … its Activity feed is the append-only timeline. The CMDB / Service Graph supplies topology (what depends on what) for correlation. Locally: which runbooks were tried and their outcome (failure memory, so a known-useless step isn't repeated), and the runbook library itself is reusable memory across incidents.

```python
class Incident:
    def __init__(self, case_id, service, mcp):
        self.case_id, self.service, self.mcp, self.tried = case_id, service, mcp, {}
    def record(self, runbook_id, outcome, ts):
        self.tried[runbook_id] = outcome
        post_findings(self.mcp, self.case_id, f"[{ts}] {runbook_id}: {outcome}")  # SFDC = record
# correlation topology comes from the CMDB / Service Graph; the runbook
# library is reusable memory across incidents
```

### [6] Limits

The hard stops, designed in before the incident storm:

- **SLA time budget** ≫ mitigate within the SLA for the severity or page a human; at 24/7 the clock is the real boss.
- **confidence + max steps** ≫ act only above a confidence threshold; cap remediation attempts, then escalate.
- **kill switch** ≫ enforced at the orchestration / RBAC layer, not by asking the model nicely.

```python
SLA_SECONDS = {"P1": 900, "P2": 3600}    # mitigate within SLA or page a human
CONFIDENCE  = 0.7                         # act only above this; else escalate
MAX_STEPS   = 5

def sla_breaching(case, now): return now - case.opened_at > 0.8 * SLA_SECONDS[case.severity]
def no_progress(prev, detail): return prev is not None and prev == detail
# the kill switch is enforced at the orchestration/RBAC layer, not in the prompt
```

### [7] Control

After each runbook + SLO check (Progressive Authorization, the way Google SRE frames it): a low-risk step runs automatically; a risky step is proposed and waits for human approval; SLO green and held → document and close the Case; no-progress or capped or SLA-breaching → escalate and page with the full timeline. Never execute a risky action without approval; never close without the SLO green.

```python
def handle(incident, case, model, now) -> str:   # SLA, CONFIDENCE, MAX_STEPS from [6]
    prev = None
    for step in range(MAX_STEPS):
        if sla_breaching(case, now()):
            return escalate(case, reason="approaching SLA")         # page on-call
        plan = model.next_runbook(incident)         # [9] investigate -> pick runbook
        if plan.confidence < CONFIDENCE:
            return escalate(case, reason="low confidence", plan=plan)
        try:
            run_runbook(case, plan.runbook, plan.params)            # [4] auto, or…
        except NeedsApproval as a:
            return escalate(case, approval=a)                       # risky -> human approves
        result = gate(case)                         # [1] SLO green + documented?
        incident.record(plan.runbook.id, result.detail, now())
        if result.passed:
            close(case); return "resolved"
        if no_progress(prev, result.detail):        # stuck -> escalate
            break
        prev = result.detail
    return escalate(case, reason="not mitigated")
```

### [8] Observability

Per Case: the runbooks run and **why** (and which options were rejected … Google's transparency-over-black-box principle), telemetry snapshots, SLO results, decisions, time-to-mitigate, approvals, escalation. Metrics: auto-resolve rate, MTTR, SLA compliance, escalation rate, steps per incident. Alerts: a service failing repeatedly, SLA budget exceeded, any action attempted outside RBAC scope.

```python
def trace(case, step, plan, result, decision):
    log.info("incident", extra={
        "case": case.Id, "service": case.service, "step": step,
        "runbook": plan.runbook.id, "rationale": plan.rationale,    # why + what was ruled out
        "slo": result.detail[:200], "decision": decision, "elapsed_s": elapsed(case),
    })
# metrics: auto-resolve rate, MTTR, SLA compliance, escalation rate, steps/case
```

### [9] Model & Prompt

The model never runs commands and never writes to Salesforce directly. It investigates … reads the Case and the telemetry … and picks the next runbook and parameters from the approved library, with a confidence and a one-line rationale. The runbooks, the SLO check, and RBAC are the source of truth, so a single mid-tier model is plenty; swap it without touching the rest of the canvas. This is the scoped-LLM design the labs converged on: keep the model's role minimal, let deterministic infrastructure do the governance.

```python
SYSTEM = """You are first responder for one incident, tracked as a Salesforce Case
on a business-critical service.

You do not run commands and you do not write to Salesforce directly. Given the case and
the latest telemetry, choose the next runbook and parameters from the approved library,
with a confidence (0-1) and a one-line rationale (including what you ruled out). Prefer
the least-risky step that could clear the SLO breach. If nothing safe is likely to help,
or the step is risky or novel, escalate. Output {runbook_id, params, confidence, rationale}."""

def next_runbook(client, incident, runbooks):
    user = (f"Case: {incident.summary()}\nTelemetry: {incident.telemetry()}\n"
            f"Tried: {incident.tried}\nRunbooks: {[r.id for r in runbooks]}")
    return client.messages.create(
        model=MODEL,   # runbooks + SLO + RBAC are the source of truth; a small model is fine
        system=SYSTEM,
        messages=[{"role": "user", "content": user}],
    )  # -> {runbook_id, params, confidence, rationale}
```

The snippets are illustrative skeletons, not a framework … the runbooks and the SLO do the work deterministically; the loop only investigates, picks the next step, gates risk behind a human, and stops the moment the service is green and the Case is documented, or escalates fast.

## Resources

- [Salesforce Agentforce IT Service](https://www.salesforce.com/news/stories/agentforce-it-service-announcement/) … agent-first ITSM with an agentic CMDB (GA Oct 2025).
- [AWS DevOps Agent + Salesforce MCP](https://aws.amazon.com/blogs/devops/automating-incident-investigation-with-aws-devops-agent-and-salesforce-mcp-server/) … agent reads and writes the Case over MCP.
- [Google SRE: agentic AI principles](https://cloud.google.com/blog/products/devops-sre/how-google-sre-is-using-agentic-ai-to-improve-operations) … transparency, RBAC, SLOs, progressive authorization.
- [IBM ITBench](https://github.com/itbench-hub/ITBench) … benchmark; top models resolve only ~11% of SRE scenarios.
