# Chapter 2: Agent Runtime & Orchestration

> Choosing the model loop, managed platform, task queue, worker model, and recovery strategy.

---

## The Core Problem

An enterprise agent is rarely a single request-response call. It is usually a long-running, stateful workflow that:

- reads context from several systems
- calls tools with different credentials
- may wait for a user, approver, workflow, or external API
- needs to recover from worker restarts
- must emit a trace that explains every decision and action
- may need to pause, resume, or hand off to another agent

Separate two concerns:

1. **Agent runtime**: the inner loop that calls the model, selects tools, handles tool results, manages memory, and stops.
2. **Orchestration layer**: the outer system that dispatches tasks, manages state, retries failures, pauses for humans, and resumes work.

---

## The Agentic Loop

Every framework implements the same loop:

```
user/task input
  -> model call
  -> tool request
  -> policy check
  -> tool execution
  -> tool result
  -> next model call
  -> final output or next tool
```

A production runtime must add:

- typed tool schemas
- max-turn and token budgets
- tool allowlists and denylists
- credential brokering
- trace emission
- structured error handling
- resumable state
- model and prompt versioning

---

## Managed Agent Platforms

Managed platforms are useful when you want the provider to handle hosting, identity integration, tool hosting, memory, observability, or scaling. They do not remove the need for your own product policy, tenant authorization, data classification, and action-control model.

### Option 1: Amazon Bedrock AgentCore

[Amazon Bedrock AgentCore](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/what-is-bedrock-agentcore.html) is AWS's agentic platform for building, deploying, and operating agents securely at scale. AWS describes it as framework- and model-flexible: AgentCore can work with open-source frameworks such as CrewAI, LangGraph, LlamaIndex, Google ADK, OpenAI Agents SDK, and Strands Agents, and with foundation models in or outside Amazon Bedrock.

Core AgentCore services and capabilities:

| Capability | What It Provides |
|------------|------------------|
| **Runtime** | Secure serverless hosting for agents and tools, with session isolation, long-running workloads, streaming, and framework flexibility |
| **Memory** | Short-term and long-term context stores that can be shared across agents |
| **Gateway** | Converts APIs, Lambda functions, existing services, and MCP servers into agent-accessible tools |
| **Identity** | Agent identity, inbound auth, outbound auth, and integration with providers such as Cognito, Okta, Microsoft Entra ID, and Auth0 |
| **Code Interpreter** | Isolated code execution for analysis and task completion |
| **Browser** | Managed browser automation environment for web applications |
| **Observability** | Agent traces, metrics, CloudWatch integration, and OTEL-compatible telemetry |
| **Evaluations** | Automated quality assessment using sessions, traces, and spans |
| **Policy** | Deterministic tool-call controls using natural language rules or Cedar |
| **Registry** | Governed catalog for agents, MCP servers, tools, skills, and custom resources |

[AgentCore Runtime](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/agents-tools-runtime.html) is the part to evaluate first if you already have agent code and need a managed execution environment. The AWS docs highlight framework agnosticism, model flexibility, MCP/A2A protocol support, isolated user sessions, long-running workloads up to 8 hours, persistent filesystem support where available, built-in authentication, and agent-specific observability.

Use AgentCore when:

- your organization is AWS-centered or already uses Bedrock and CloudWatch
- you need managed isolation for sessions, tools, code execution, or browser automation
- you want MCP/A2A support with a governed gateway and registry
- you need centralized observability and evaluation hooks for production agents

Watch out for:

- service-region availability and preview status of individual features
- how AgentCore policy maps to your existing enterprise policy model
- avoiding a split-brain control plane where app policy and platform policy disagree
- cost and latency for long-running sessions or browser/code workloads

### Option 2: Microsoft Foundry Agent Service / Azure AI Foundry

[Microsoft Foundry Agent Service](https://learn.microsoft.com/en-us/azure/ai-foundry/agents/overview), surfaced through Azure AI Foundry / Microsoft Foundry docs and portals, is Microsoft's managed platform for building, deploying, and scaling agents on Azure. Microsoft describes Agent Service as handling hosting, scaling, identity, observability, and enterprise security while supporting many models from the Foundry model catalog and frameworks such as Microsoft Agent Framework, LangGraph, or custom code.

Agent Service supports several build styles:

| Agent Type | What It Is | Best For |
|------------|------------|----------|
| **Prompt agents** | Configuration-driven agents with instructions, model, and tools | Fast no-code or low-code agents |
| **Workflow agents** | Declarative multi-step workflows with branching, human steps, and multi-agent patterns | Repeatable business processes and approvals |
| **Hosted agents** | Code-based agents deployed as managed containers | Custom orchestration, custom frameworks, and complex integrations |

The [Foundry tool catalog](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/tool-catalog?view=foundry) includes built-in tools such as web search, Code Interpreter, File Search, and function calling, plus custom tools such as MCP, A2A, and OpenAPI. Microsoft also documents toolboxes as curated bundles of tools exposed through an MCP-compatible endpoint, with centralized authentication and versioning.

Enterprise capabilities to evaluate:

- dedicated agent identities with Microsoft Entra ID
- Azure RBAC for who can create, invoke, and manage agents
- private networking options for supported agent types
- content-safety controls for unsafe output and prompt-injection risks
- tracing, evaluation, publishing, monitoring, and distribution through Microsoft 365 and Teams paths

Use Foundry Agent Service when:

- your enterprise identity, data, and collaboration stack is Microsoft-centered
- you need Entra/RBAC-native governance and Azure networking controls
- you want agents distributed into Teams, Microsoft 365, or Azure-hosted products
- prompt agents and workflow agents cover many use cases before custom runtime code is needed

Watch out for:

- preview status on subfeatures such as hosted agents, workflow agents, A2A, memory, or toolboxes
- [region, model, quota, and tool availability](https://learn.microsoft.com/en-us/azure/ai-foundry/agents/concepts/limits-quotas-regions?view=foundry); Microsoft documents Agent Service as generally available while some subfeatures remain preview
- boundaries between Foundry-managed conversation state and your product's data-retention rules
- whether external tools need delegated user access or an agent/service identity

---

## Framework and SDK Runtimes

Use a framework runtime when you want more control over hosting, app integration, model choice, or tool execution.

### OpenAI Agents SDK / Codex CLI

The OpenAI Agents SDK is a TypeScript and Python framework for defining agents, typed tools, handoffs, sessions, tracing, and guardrails. It is a good fit when you are building application-level agents around GPT models and want a conventional code framework.

Codex CLI is a terminal coding agent. It is useful when the agent works inside a local or ephemeral code checkout and needs file edits, command execution, and sandboxing.

Generic tool example:

```typescript
import { Agent, run, tool } from "@openai/agents";
import { z } from "zod";

const createTicketDraft = tool({
  name: "create_ticket_draft",
  description: "Create a draft ticket update for human review.",
  parameters: z.object({
    ticketId: z.string(),
    body: z.string(),
  }),
  execute: async ({ ticketId, body }) => {
    return await tickets.createDraft({ ticketId, body });
  },
});

const agent = new Agent({
  name: "Support Escalation Agent",
  instructions: "Analyze the case history and draft a concise next-step update.",
  tools: [createTicketDraft],
});

const result = await run(agent, "Prepare next steps for escalation CASE-123");
console.log(result.finalOutput);
```

### Anthropic Claude Agent SDK

Claude Agent SDK is useful when you want a managed agentic loop around Claude, built-in file/shell tools, MCP integration, hooks, budgets, and session resume. It fits coding, document, and operational workflows where you want a capable local or worker-hosted agent runtime.

```typescript
import { query, tool, createSdkMcpServer } from "@anthropic-ai/claude-agent-sdk";
import { z } from "zod";

const lookupAccount = tool(
  "lookup_account",
  "Fetch scoped account context for the current task.",
  { accountId: z.string() },
  async ({ accountId }) => {
    const account = await crm.lookupAccount(accountId);
    return { content: [{ type: "text", text: JSON.stringify(account) }] };
  }
);

const businessTools = createSdkMcpServer({
  name: "business-tools",
  tools: [lookupAccount],
});

for await (const message of query({
  prompt: "Summarize renewal risk for account ACME-001.",
  options: {
    systemPrompt: hardRules + policyDigest,
    allowedTools: ["Read", "Grep"],
    mcpServers: { "business-tools": businessTools },
    maxTurns: 20,
  },
})) {
  console.log(message);
}
```

### LangGraph, CrewAI, Pydantic AI, Mastra, and Direct APIs

| Runtime | Best For | Watch Out For |
|---------|----------|---------------|
| **LangGraph** | Stateful graphs, durable checkpoints, human-in-the-loop, conditional routing | More framework concepts to learn |
| **CrewAI** | Role-based multi-agent collaboration | Overusing multi-agent patterns where one agent is enough |
| **Pydantic AI** | Type-safe Python agents with validated inputs and outputs | You own orchestration and hosting |
| **Mastra** | TypeScript agents, workflows, memory, RAG, MCP, provider flexibility | Platform maturity and integration fit |
| **Direct API loop** | Maximum control over prompts, tools, retries, and state | You own every edge case |

Start with the simplest runtime that lets you enforce policy, inspect traces, and test trajectories. Add graph orchestration or multiple agents only when a single agent becomes too broad to evaluate reliably.

---

## Runtime Selection Matrix

| Need | Strong Candidates |
|------|-------------------|
| AWS-native managed agent platform | Amazon Bedrock AgentCore |
| Microsoft identity, Teams/M365 distribution, Azure governance | Microsoft Foundry Agent Service |
| Custom app agent in TypeScript or Python on GPT models | OpenAI Agents SDK |
| Claude-centered worker or coding agent | Claude Agent SDK |
| Durable multi-step workflow with human checkpoints | LangGraph or Temporal plus a model runtime |
| Strict typed outputs in Python | Pydantic AI |
| Full control and low dependency surface | Direct API wrapper |

---

## Orchestration Layer

The outer orchestration layer owns dispatch, retries, recovery, and lifecycle. It should be deterministic.

### Queue Options

| Technology | Strengths | Weaknesses | Best For |
|------------|-----------|------------|----------|
| **Redis Streams** | Fast, consumer groups, persistent enough for many apps | Requires Redis durability discipline | Low-latency worker dispatch |
| **BullMQ** | Rich job features for Node.js | Redis dependency, Node-centric | Product teams already on Node |
| **AWS SQS** | Managed queue, dead-letter queues, scales well | Message size limits, eventual consistency | AWS-hosted batch and event tasks |
| **Azure Service Bus** | Managed queues/topics, sessions, dead lettering | Azure-specific concepts | Azure-hosted enterprise workflows |
| **RabbitMQ** | Mature routing patterns | Operational overhead | Complex routing and fanout |
| **Temporal** | Durable workflows, retries, timers, human waits | Heavier runtime and learning curve | Multi-day or high-value workflows |
| **PostgreSQL SKIP LOCKED** | No extra service | Polling and throughput limits | Small systems and early products |

### Worker Models

| Model | Use When | Tradeoff |
|-------|----------|----------|
| **Long-lived daemon** | Interactive agents need low latency and warm context | You must manage leaks, restarts, and isolation |
| **Per-task job** | Batch work, clear boundaries, simple cleanup | Higher cold-start cost |
| **Durable workflow** | Work pauses for hours or days | More orchestration code |
| **Managed agent runtime** | You want the platform to own hosting and scaling | Platform constraints and lock-in |

---

## Session State and Recovery

Agents need several kinds of state:

| State | Examples | Where to Store |
|-------|----------|----------------|
| **Run metadata** | status, task type, user, policy decision | Control database |
| **Conversation state** | model turns, tool results, summaries | Runtime session store or database |
| **Artifacts** | drafts, files, reports, generated patches | Object storage or document store |
| **Long-term memory** | durable preferences, prior outcomes | Memory service, vector DB, or relational tables |
| **Audit events** | immutable action history | Append-only log or audit table |

Recovery rule: after a crash, the system should know whether to retry, resume, ask a human, or mark the run failed. Do not leave that decision to the model.

---

## Failure Handling

Define deterministic handling for common failure modes:

| Failure | Handling |
|---------|----------|
| Tool timeout | Retry with backoff, then ask for human input or fail the step |
| Credential expired | Request a fresh scoped grant from the broker |
| Validation failed | Feed structured error back to the agent, with iteration limits |
| Approval rejected | Stop or branch to a safer draft path |
| Model exceeded budget | Summarize state, checkpoint, and require explicit continuation |
| Ambiguous data permission | Deny by default and explain what access is missing |

---

## Design Checklist

- [ ] Runtime choice is documented with model, hosting, and data-residency implications
- [ ] Every tool call passes through policy and audit hooks
- [ ] Queue semantics are clear: retry, dead letter, dedupe, and ordering
- [ ] Long-running runs can pause and resume deterministically
- [ ] Session state is separated from audit state
- [ ] Model and prompt versions are recorded for each run
- [ ] Managed-platform feature preview/GA status is tracked before production use
