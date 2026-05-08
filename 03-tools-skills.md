# Chapter 3: Tools, Skills & Capabilities

> How agents gain capabilities without gaining unlimited authority.

---

## The Tool Landscape

Agents become useful when they can use tools: search documents, call APIs, run code, update tickets, query databases, browse web apps, and create drafts. They become risky when those tools are broad, poorly typed, or invisible to policy.

Think in capability layers:

| Layer | Example | Risk |
|-------|---------|------|
| **Knowledge** | Search docs, retrieve records, inspect history | Sensitive data exposure |
| **Computation** | Run Python, transform files, calculate forecasts | Data leakage, resource abuse |
| **Drafting** | Create ticket draft, PR, email, report | Incorrect or misleading output |
| **Workflow** | Move task status, request approval, assign owner | Process disruption |
| **External action** | Send email, update customer record, trigger payment | High blast radius |
| **Administration** | Change permissions, configure integrations | Critical blast radius |

Production systems should treat tools as governed product capabilities, not incidental helper functions.

---

## Tool Design Rules

### 1. Prefer Typed Tools

Typed tools make the agent's choices inspectable and enforceable.

```typescript
const createTicketDraft = tool({
  name: "create_ticket_draft",
  description: "Create a draft ticket update. Does not publish it.",
  parameters: z.object({
    ticketId: z.string(),
    title: z.string().max(120),
    body: z.string().max(5000),
    citations: z.array(z.string()).default([]),
  }),
  execute: async (input, ctx) => {
    await ctx.policy.require("ticket:draft", { ticketId: input.ticketId });
    return tickets.createDraft(input);
  },
});
```

### 2. Make Side Effects Explicit

Name tools by what they really do:

| Avoid | Prefer |
|-------|--------|
| `update_ticket` | `create_ticket_update_draft` and `publish_ticket_update` |
| `send_message` | `draft_message` and `send_approved_message` |
| `run_workflow` | `request_workflow_execution` and `execute_approved_workflow` |
| `query_data` | `query_customer_records_readonly` |

The tool name should reveal whether the action is read-only, draft-only, approval-gated, or executable.

### 3. Return Structured Results

Agents handle tool failures better when the result is structured.

```typescript
interface ToolResult<T> {
  ok: boolean;
  data?: T;
  error?: {
    code: 'permission_denied' | 'not_found' | 'rate_limited' | 'validation_failed';
    message: string;
    retryable: boolean;
  };
  auditId: string;
}
```

### 4. Keep Tool Descriptions Operational

Tool descriptions should state:

- what the tool does
- whether it mutates state
- what approval or policy gate applies
- what inputs are required
- what the result means
- what errors are expected

Avoid promotional descriptions. The model needs operational truth.

---

## Tool Categories

| Category | Examples | Default Mode |
|----------|----------|--------------|
| **Document and knowledge search** | SharePoint, Drive, Confluence, Notion, file search | Read-only with document ACLs |
| **Business systems** | Salesforce, ServiceNow, Jira, Zendesk, SAP | Draft or approval-gated |
| **Data and analytics** | SQL, warehouse, BI, metrics | Read-only with row/column policy |
| **Code and repositories** | GitHub, GitLab, Azure DevOps | Branch/PR before merge |
| **Messaging** | Slack, Teams, email | Draft before send |
| **Code execution** | Python, JavaScript, notebooks | Isolated sandbox |
| **Browser automation** | Web app workflows without APIs | Constrained session and audit |
| **Internal APIs** | Entitlements, billing, workflow services | Least-privilege scoped endpoints |

---

## CLI Tools

CLIs are still useful for coding agents, data agents, and automation-heavy internal tools. Use them deliberately.

Safe CLI patterns:

- run in a sandbox with a minimal filesystem
- allowlist binaries and arguments for production tools
- prefer machine-readable output such as JSON
- pin tool versions in the image
- block metadata endpoints and internal networks unless explicitly needed
- inject short-lived credentials at runtime
- wrap high-risk commands in typed tools

Example wrapper:

```typescript
async function runReportCheck(reportPath: string): Promise<ToolResult<CheckSummary>> {
  const result = await execFile("report-lint", ["--json", reportPath], {
    timeout: 30_000,
    env: minimalEnv(),
  });

  return parseReportLintJson(result.stdout);
}
```

Use direct shell access for local development and highly trusted coding agents. Use typed wrappers for repeated enterprise workflows.

---

## Skills

Skills are document-first capability packages. A skill explains when and how to perform a task, what tools to use, what constraints apply, and how to recover from common failures.

```
skills/
|-- customer-escalation-summary/
|   `-- SKILL.md
|-- contract-risk-review/
|   `-- SKILL.md
`-- support-ticket-drafting/
    `-- SKILL.md
```

Example:

```markdown
# Support Ticket Drafting

## When to Use
Use when asked to prepare a customer-facing update for an existing support ticket.

## Constraints
- Create drafts only. Do not publish.
- Cite source records used in the answer.
- Do not include internal notes or private account fields.
- Ask for human review if severity is critical.

## Required Tools
- read_ticket
- search_knowledge_base
- create_ticket_update_draft
```

Skills are good for reusable behavior and domain conventions. They are not enforcement by themselves. Pair them with tool policy and runtime gates.

---

## Subagents and Role Bundles

Subagents package role, prompt, model, tool scope, budget, memory access, and skills into a reusable definition.

| Role | Typical Capabilities |
|------|----------------------|
| `research-analyst` | Search, summarize, cite, no writes |
| `support-drafter` | Read tickets, search docs, create drafts |
| `workflow-operator` | Request execution, monitor status, escalate |
| `code-reviewer` | Read repo, comment on PR, no merge |
| `risk-reviewer` | Inspect high-risk actions and explain policy concerns |

Managed role bundles are useful for enterprise standardization because they make the effective configuration reviewable. They do not replace sandboxing, identity, validation, or audit trails.

---

## MCP

Model Context Protocol (MCP) is a common way to expose tools, resources, and prompts to model runtimes. Use it when tools need to be shared across agents or maintained outside the application team.

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server";

const server = new McpServer({ name: "customer-tools" });

server.tool(
  "lookup_customer",
  { customerId: z.string() },
  async ({ customerId }) => {
    const customer = await scopedCustomerLookup(customerId);
    return { content: [{ type: "text", text: JSON.stringify(customer) }] };
  }
);
```

Operational guidance:

- pin server versions and tested protocol versions
- require authentication for remote MCP servers
- route tool calls through policy
- keep an internal allowlist or registry
- log tool inputs and outputs with redaction
- treat tool descriptions from third parties as untrusted until reviewed

MCP registry or catalog presence is discoverability, not trust.

---

## A2A

Agent-to-agent protocols such as A2A are useful when the remote system owns its own state, tools, policy, lifecycle, and approvals.

Use A2A when:

- work is long-running and task-oriented
- the remote agent owns credentials you should not proxy
- responses may include artifacts, follow-up questions, or status updates
- a separate product or team owns the execution boundary

Use MCP when:

- you are exposing tools or resources to a calling runtime
- calls are narrow and subordinate to the calling agent
- the caller owns the task lifecycle

In practice, many enterprise systems use MCP inside one agent runtime and A2A across product boundaries.

---

## OpenAPI and Function Tools

OpenAPI tools are useful when existing HTTP APIs already describe their request and response shapes. Function tools are useful when you want tighter code-level control.

Safe OpenAPI usage:

- expose a reduced spec, not your entire internal API surface
- remove destructive endpoints unless explicitly approved
- add per-operation policy metadata
- require OAuth scopes or service entitlements per operation
- test every operation with realistic authorization failures

---

## Tool Catalogs

As the number of tools grows, you need a catalog.

Each tool entry should include:

- owner team
- runtime endpoint
- version
- input and output schema
- auth mode
- data classification
- side-effect level
- required policy
- test environment
- audit fields
- rollback or compensation behavior

Managed platforms increasingly include catalog concepts. For example, Amazon Bedrock AgentCore includes Gateway and Registry capabilities, while Microsoft Foundry exposes a tool catalog and toolboxes. Even if you use a provider catalog, keep your own policy metadata close to the application.

---

## Capability Matrix

```typescript
const capabilityMatrix = {
  "research-agent": {
    allowedTools: ["search_docs", "read_ticket", "summarize_file"],
    deniedTools: ["send_email", "publish_ticket_update", "execute_workflow"],
    maxTurns: 20,
  },
  "support-drafter": {
    allowedTools: ["read_ticket", "search_docs", "create_ticket_update_draft"],
    deniedTools: ["publish_ticket_update", "change_customer_record"],
    maxTurns: 30,
  },
  "workflow-operator": {
    allowedTools: ["read_workflow", "request_workflow_execution", "notify_team"],
    deniedTools: ["execute_workflow_without_approval"],
    maxTurns: 40,
  },
};
```

Start restrictive. Expand after observing real run traces and failure modes.

---

## Design Checklist

- [ ] Tool names reveal side effects
- [ ] Tool inputs and outputs are typed
- [ ] Tool calls route through policy and audit hooks
- [ ] High-risk tools have approval gates
- [ ] Tool results use structured error codes
- [ ] Skills are versioned and reviewed
- [ ] MCP/OpenAPI servers are allowlisted
- [ ] Catalog entries include owner, auth mode, side-effect level, and test coverage
