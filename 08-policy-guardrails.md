# Chapter 8: Policy & Guardrails

> How to constrain agent behavior with enforceable controls.

---

## Guardrail Layers

Prompt rules help, but they are not enough. Production guardrails need multiple layers.

```
Layer 1: Product policy       What the product allows for this tenant/task
Layer 2: Runtime policy       What tools, models, budgets, and data scopes are allowed
Layer 3: Tool policy          What each tool enforces before execution
Layer 4: Identity policy      What credentials can actually do
Layer 5: External controls    Source-system RBAC, approval systems, audit, DLP
```

If a prompt rule says "do not send emails," the `send_email` tool should still be denied or approval-gated. The model should not be the enforcement boundary.

---

## Autonomy Tiers

| Tier | Name | Allowed Behavior | Example |
|------|------|------------------|---------|
| 0 | Observe | Read and summarize | Summarize a policy document |
| 1 | Recommend | Suggest actions | Recommend next steps for a case |
| 2 | Draft | Create proposed artifacts | Draft email, PR, ticket update |
| 3 | Execute with Gate | Execute after validation and approval | Publish approved ticket response |
| 4 | Bounded Autonomous | Execute within narrow pre-approved limits | Auto-label low-risk tickets |

Each task type should declare its maximum tier. Each tenant can lower that maximum.

---

## Policy Configuration

```yaml
agents:
  research-agent:
    max_tier: 1
    allowed_tools:
      - search_docs
      - read_ticket
      - summarize_file
    denied_tools:
      - send_email
      - update_customer_record
      - execute_workflow
    max_turns: 20
    max_runtime_minutes: 10

  support-drafter:
    max_tier: 2
    allowed_tools:
      - read_ticket
      - search_docs
      - create_ticket_update_draft
    denied_tools:
      - publish_ticket_update
      - change_customer_record
    max_turns: 30

  workflow-operator:
    max_tier: 3
    allowed_tools:
      - read_workflow
      - request_workflow_execution
      - execute_approved_workflow
    approval_required_for:
      - execute_approved_workflow
    max_turns: 40
```

Policy should be versioned. Each run should record the policy version it used.

---

## Policy Decision Pipeline

```typescript
async function evaluateToolCall(call: ToolCall, run: AgentRun): Promise<PolicyDecision> {
  const policy = await loadEffectivePolicy(run.organizationId, run.agentSlug);
  const user = await loadUser(run.userId);
  const data = await classifyRequestedData(call);

  return policyEngine.evaluate({
    policy,
    user,
    run,
    tool: call.name,
    input: call.input,
    dataClassification: data.classification,
    approvalState: run.approvalState,
  });
}
```

Evaluate policy:

- before a run starts
- before data retrieval
- before each tool call
- before credential issuance
- before external action
- before storing long-term memory

---

## Data Guardrails

| Control | Purpose |
|---------|---------|
| Source ACL preservation | Agent cannot read data the identity cannot read |
| Field filtering | Remove fields not needed for the task |
| Data classification | Apply stricter controls to PII, PHI, financial, legal, or secret data |
| Redaction | Prevent sensitive values from entering prompts, traces, and logs |
| Export controls | Restrict file downloads, bulk exports, and copy-paste tools |
| Retention policy | Avoid indefinite storage of retrieved data or memory |

Data policy should run before retrieval and before model context assembly.

---

## Tool Guardrails

Every tool should declare:

```typescript
interface ToolPolicyMetadata {
  name: string;
  sideEffect: 'none' | 'draft' | 'write' | 'external' | 'admin';
  dataClasses: string[];
  requiredCapability: string;
  requiresApproval: boolean;
  maxTier: 0 | 1 | 2 | 3 | 4;
  ownerTeam: string;
}
```

Policy can then make simple deterministic decisions:

- Tier 0 and 1 agents cannot call write tools
- draft tools can run without approval if data classification allows it
- external tools require approval for high-risk tenants
- admin tools require explicit admin approval and separate credentials

---

## Prompt Rules

Prompt rules are useful for intent-level guidance.

Good prompt rules:

- "When answering with customer-specific data, cite the source record."
- "Create drafts only unless the task includes an approved execution ID."
- "If the user asks for data outside their workspace, explain that access is unavailable."
- "Ask for review before external communication on high-severity cases."

Bad prompt rules:

- "Never leak secrets" without redaction and data filtering
- "Do not make mistakes" without validators
- "Only use approved tools" without tool enforcement

Prompt rules should align with hard policy, not substitute for it.

---

## Structural Controls

Convert recurring prompt rules into structural controls.

| Prompt Rule | Structural Control |
|-------------|--------------------|
| "Do not send without review" | No direct send tool; only draft and approved send tools |
| "Cite sources" | Validator rejects outputs without source IDs |
| "Do not access HR data" | Retrieval layer blocks HR collections |
| "Stay under budget" | Runtime max-turn, token, and cost limits |
| "Only use approved wording" | Template or clause library tool |
| "Do not store sensitive memory" | Memory write policy and classification gate |

The goal is to make safe behavior the path of least resistance.

---

## Budget Guardrails

Agents can fail expensively.

Set limits per task type:

| Task Type | Max Turns | Max Runtime | Max Tool Calls | Notes |
|-----------|-----------|-------------|----------------|-------|
| Short summary | 8 | 3 min | 10 | Low-cost path |
| Ticket draft | 20 | 10 min | 25 | Requires citations |
| Code PR | 60 | 60 min | 100 | Requires validation |
| Data analysis | 40 | 30 min | 80 | Watch code sandbox cost |
| Browser workflow | 30 | 30 min | 60 | Watch action gates |

When a limit is reached, the agent should summarize current state and stop. Continuing should be a deliberate user or workflow action.

---

## Approval Policies

Approval policy should consider:

- data classification
- tenant risk tier
- external visibility
- money movement
- customer impact
- permission changes
- production effect
- legal/regulatory sensitivity
- confidence and validation result

Example:

```yaml
approval:
  external_communication:
    high_severity_case: support_lead
    regulated_customer: legal_or_compliance
  customer_record_update:
    low_risk_field: owner
    financial_or_contract_field: manager
  workflow_execution:
    read_only: none
    mutating: workflow_owner
```

---

## Design Checklist

- [ ] Each agent has a maximum autonomy tier
- [ ] Tools declare side-effect level and required capability
- [ ] Policy is checked before tool calls, credentials, retrieval, and external actions
- [ ] Prompt rules have matching structural controls where risk is meaningful
- [ ] Budget limits are set per task type
- [ ] Approval rules are based on risk and data classification
- [ ] Policy versions are recorded in every run
- [ ] Denials are explainable to users and admins
