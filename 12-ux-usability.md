# Chapter 12: UX & Usability

> How to package enterprise agents so people can use them safely without learning the architecture.

---

## Product Principle

Users should not need to understand model loops, token budgets, MCP, OBO, or policy engines. They should understand:

- what the agent can do
- what data it will use
- what action it proposes
- what needs approval
- what happened after it ran

The UI should make risk visible at the moment it matters.

---

## Tenancy and Workspaces

Enterprise agents need explicit boundaries.

```text
Organization
  -> Workspace / business unit
     -> Connected systems
     -> Agents
     -> Users and groups
     -> Policies
     -> Runs and artifacts
```

Default to private runs unless sharing is explicit. A user in one workspace should not see another workspace's sessions, documents, credentials, or artifacts.

---

## Onboarding

Good onboarding asks for the minimum needed to get one safe workflow running.

1. Connect identity provider or invite users.
2. Connect one low-risk system, such as a ticketing or document source.
3. Enable one draft-only agent.
4. Run a guided task.
5. Review trace, sources, and draft output.
6. Add approvals or broader integrations after trust is established.

Avoid starting with admin credentials and broad tool access.

---

## Agent Directory

Users need to know which agent to use.

Each agent card should show:

- purpose
- allowed systems
- action level: observe, recommend, draft, gated execute, autonomous
- required approvals
- owner team
- recent reliability
- data it may access

Example:

```text
Support Drafter
Creates cited ticket-response drafts from ticket history and approved support docs.
Action level: Draft only
Systems: Zendesk, SharePoint
Owner: Support Platform
```

---

## Starting a Run

The run start screen should make scope explicit:

- agent
- task
- workspace
- connected systems
- data sources
- action mode
- approval requirement
- estimated cost/time, when useful

If a task is high-risk, show the gate before the agent starts.

---

## Run View

A run view should show current state without overwhelming the user.

Useful sections:

- task and requester
- current status
- sources used
- draft/action produced
- validation results
- approvals needed
- timeline of important events
- full trace for admins/developers

Do not expose raw hidden prompts to ordinary users. Show operationally meaningful explanations: sources, tools, policy, and proposed action.

---

## Review UX

Reviewers need exact proposed actions, not vague summaries.

A review card should include:

- what will change
- where it will change
- source evidence
- validation status
- policy reason for requiring review
- approve, reject, edit, and request-changes options
- expiry time

For external communication, show the exact message that will be sent. For API actions, show the structured payload or a human-readable diff.

---

## Error Prevention

Make dangerous mistakes hard or impossible.

| User Mistake | Product Control |
|--------------|-----------------|
| Giving broad credentials | Admin-configured integrations and scoped consent |
| Sending draft too early | Separate draft and send controls |
| Reviewing stale output | Expire approvals when source data changes |
| Selecting wrong workspace | Prominent workspace and data-source labels |
| Sharing sensitive run | Private-by-default sessions and explicit share control |
| Trusting unsupported answer | Require citations and source visibility |
| Ignoring failed validation | Block approval or require override reason |

---

## Collaboration

Agent sessions often become team artifacts.

Support:

- comments on runs
- assignment
- handoff to another agent or human
- share links with permission checks
- status filters
- run history by resource
- saved artifacts
- reviewer feedback

Keep the canonical state in the product, even when notifications happen in Slack, Teams, or email.

---

## Admin UX

Admins need control without editing YAML for every change.

Admin surfaces:

- connected systems
- agent definitions and owners
- tool catalog
- policy settings
- approval routes
- data retention
- audit search
- cost controls
- feature flags and preview capabilities

Show effective policy. If a project, tenant, and agent all define constraints, admins need to see the resolved result.

---

## Explainability

Users do not need chain-of-thought. They need grounded operational explanations.

Good explanations:

- "Used these 3 source documents."
- "Created a draft because policy requires human review for high-severity cases."
- "Could not access the contract because you do not have permission."
- "Validation failed because the response lacked a required citation."

Avoid pretending the model is certain. Show sources and controls.

---

## Design Checklist

- [ ] Workspaces, systems, and data scope are visible
- [ ] Agent action level is clear before a run starts
- [ ] Runs are private by default
- [ ] Review cards show exact proposed action
- [ ] Approvals expire when context becomes stale
- [ ] Validation failures block or clearly warn
- [ ] Admins can see effective policy and tool ownership
- [ ] Users see sources, not hidden prompts
