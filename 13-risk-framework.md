# Chapter 13: Risk Framework & Checklists

> Decision matrices and readiness checks for enterprise agents.

---

## Top Risks

| Risk | Why It Happens | Example | Controls |
|------|----------------|---------|----------|
| Prompt injection | Agents read untrusted content | Document tells agent to send data elsewhere | Label untrusted content, tool policy, sandboxing |
| Data leakage | Retrieval is too broad or logs are unsafe | Agent includes restricted fields in answer | ACL preservation, field filtering, redaction |
| Unauthorized action | Tool is too powerful or approval is missing | Agent updates customer record directly | Draft-first tools, approval gates, scoped credentials |
| Hallucinated output | Model fills gaps under uncertainty | Agent invents policy or customer fact | Citations, validation, source-bound answers |
| Tool abuse | Generic API/CLI tool exposes too much | Agent calls admin endpoint | Narrow typed tools, allowlists, policy checks |
| Runaway cost | Long loops or repeated tool failures | Agent retries for hours | Budgets, timeouts, circuit breakers |
| Stale context | Data changes after retrieval or approval | Approved action executes days later | Freshness checks, approval expiry |
| Cross-tenant access | Retrieval or storage lacks tenant boundary | User sees another team's run | Tenant isolation, tests, audit |
| Memory misuse | Sensitive data stored long term | Agent remembers private case details | Memory policy, retention, delete controls |
| Audit gap | Events are incomplete | Cannot explain who approved action | Append-only audit and trace correlation |

---

## Autonomy Decision Matrix

| Question | Low Risk | Medium Risk | High Risk |
|----------|----------|-------------|-----------|
| Data sensitivity | Public/internal | Customer/confidential | Regulated/secret |
| Action side effect | None/draft | Internal write | External/admin/financial |
| Reversibility | Easy | Compensatable | Hard or impossible |
| Validation | Deterministic | Partial | Mostly human judgment |
| User impact | Individual | Team/customer | Many customers/legal |
| Frequency | Rare/manual | Regular | Continuous/autonomous |

Suggested maximum tier:

- mostly low risk: Tier 2 or 3
- any high-risk data or irreversible action: Tier 2 until reviewed
- high-risk action plus weak validation: Tier 1 or 2 only
- bounded, low-risk, validated, reversible actions: Tier 4 may be appropriate

---

## Build vs Buy

Use managed platforms when they materially reduce platform risk or operations burden.

| Need | Consider |
|------|----------|
| AWS-native managed runtime, identity, gateway, memory, browser/code tools, observability, policy, registry | Amazon Bedrock AgentCore |
| Azure/Microsoft identity, Foundry model catalog, prompt/workflow/hosted agents, Teams/M365 distribution, Azure RBAC | Microsoft Foundry Agent Service |
| Full product control and custom UX | Own runtime plus provider APIs/frameworks |
| Durable business workflows | Temporal, Logic Apps, Step Functions, Durable Functions, or equivalent |
| Shared tool ecosystem | MCP/OpenAPI with internal registry |

Managed platform adoption checklist:

- [ ] Region and feature availability verified
- [ ] Preview features identified and risk-accepted
- [ ] Data retention and residency reviewed
- [ ] Identity model mapped to enterprise IAM
- [ ] Tool policy mapped to product policy
- [ ] Observability exported or joined with enterprise telemetry
- [ ] Cost model tested with realistic runs
- [ ] Exit strategy documented for critical workflows

---

## Production Readiness Checklist

### Architecture

- [ ] Ingestion, policy, execution, integration, action control, and observability planes are separated
- [ ] Workers do not hold broad static credentials
- [ ] Queue retry, dead-letter, and dedupe behavior is defined
- [ ] Long-running runs can pause and resume
- [ ] Tenant isolation is enforced in retrieval, storage, and UI

### Policy

- [ ] Every agent has a maximum autonomy tier
- [ ] Every tool declares side-effect level and required capability
- [ ] Policy checks run before retrieval, credentials, tools, and actions
- [ ] Approval gates exist for high-risk actions
- [ ] Budget and timeout limits exist per task type

### Identity and Data

- [ ] User-delegated, agent, and service identities are modeled
- [ ] Credential grants are short-lived and scoped
- [ ] Source ACLs are preserved during retrieval
- [ ] Sensitive fields are filtered or redacted
- [ ] Memory has retention, owner, scope, and deletion rules

### Action Delivery

- [ ] Draft and execute are separate paths for high-risk actions
- [ ] Deterministic validators run before review or execution
- [ ] Pull requests or approval records identify agent authorship
- [ ] Executable tools define idempotency and compensation behavior
- [ ] Stale approvals expire

### Observability

- [ ] Model, tool, policy, data, credential, approval, and action events are traceable
- [ ] Audit records are append-only or tamper-evident
- [ ] Prompt and output logging is redacted and access-controlled
- [ ] Alerts exist for cost spikes, policy denials, tool errors, and data-access anomalies
- [ ] Run summaries are understandable to users and auditors

### Testing

- [ ] Trajectory tests cover common and high-risk tasks
- [ ] Prompt-injection tests use untrusted docs, tickets, HTML, and tool output
- [ ] Data-permission tests cover tenant and source ACL boundaries
- [ ] Tool-abuse tests verify fail-closed behavior
- [ ] Regression tests run before model, prompt, tool, policy, or platform changes

---

## When Not to Use an Agent

Use simpler automation when:

- the workflow is deterministic
- inputs and outputs are fully structured
- the task is high-volume and low-variance
- mistakes are expensive and human judgment is not needed
- a rules engine, workflow, SQL query, or API integration solves it cleanly

| Problem | Usually Better Than an Agent |
|---------|------------------------------|
| Fixed approval routing | Workflow engine |
| Data sync | ETL/job pipeline |
| Known validation rule | Policy engine or validator |
| Repeated calculation | Service function |
| Simple notification | Event rule |
| Stable form filling | Direct API integration |

Agents are best when context is messy, language-heavy, judgment matters, and tool use benefits from reasoning.

---

## Minimum Viable Production Scope

For a first production launch:

1. Choose one narrow task.
2. Use read-only or draft-only tools.
3. Preserve source ACLs.
4. Require citations for factual answers.
5. Record full traces and audit events.
6. Add deterministic validators.
7. Launch to a small workspace.
8. Review traces daily.
9. Add execution only after the draft path is stable.

This is slower than enabling every tool at once, but it creates evidence. Evidence is what lets autonomy expand safely.

---

## Final Summary

The core enterprise-agent architecture is stable across domains:

1. Normalize work through an ingestion plane.
2. Evaluate policy before reasoning and before action.
3. Run agents in isolated runtimes.
4. Give agents scoped tools and short-lived credentials.
5. Retrieve only authorized, relevant context.
6. Deliver changes through drafts, validation, review, and gated execution.
7. Trace everything needed to debug and audit.

If those pieces are missing, the model choice will not save the system. If those pieces are strong, you can change models, frameworks, and managed platforms without rebuilding the governance foundation.
