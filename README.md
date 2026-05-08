# Enterprise Agent Guide

[![License: CC BY 4.0](https://img.shields.io/badge/License-CC_BY_4.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/)
[![Code License: MIT](https://img.shields.io/badge/Code_License-MIT-blue.svg)](https://opensource.org/licenses/MIT)
[![Chapters: 13](https://img.shields.io/badge/Chapters-13-green.svg)](#guide-structure)

> How to design, build, and operate enterprise AI agents safely.

Enterprise agents can read knowledge bases, update tickets, analyze documents, prepare pull requests, call business APIs, run code, coordinate workflows, and notify people. The core engineering problem is not the model. It is letting probabilistic software act inside real enterprise systems without leaking data, bypassing approvals, or taking irreversible actions.

This guide covers the architecture decisions you need to make when building enterprise agents: runtime selection, tool design, identity, context, policy, observability, testing, UX, and go-live risk controls.

The platform notes for **Amazon Bedrock AgentCore** and **Microsoft Foundry Agent Service / Azure AI Foundry** were checked against official AWS and Microsoft documentation on **2026-05-08**.

---

## Who This Is For

- **Engineering leaders** evaluating where agents fit in enterprise products and internal platforms
- **Platform teams** building shared agent runtimes, tool catalogs, and governance layers
- **Application teams** adding agents to business workflows
- **Security, risk, and compliance teams** reviewing how agent actions are constrained and audited
- **Product teams** designing agent experiences for employees or customers

---

## Guide Structure

| # | Chapter | What You'll Learn |
|---|---------|-------------------|
| 1 | [Architecture Overview](./01-architecture.md) | The six planes of an enterprise-agent system |
| 2 | [Agent Runtime & Orchestration](./02-agent-runtime.md) | Runtime options, managed platforms, queues, recovery, and session state |
| 3 | [Tools, Skills & Capabilities](./03-tools-skills.md) | Tool design, MCP, A2A, skill systems, and capability scoping |
| 4 | [Sandboxed Execution](./04-sandboxed-execution.md) | Isolation for code, browsers, file work, and long-running jobs |
| 5 | [Credential Management](./05-credential-management.md) | Agent identity, delegated user access, service access, and short-lived credentials |
| 6 | [The Data Plane](./06-data-plane.md) | Enterprise context, retrieval, permissions, and memory |
| 7 | [Change Control & Action Delivery](./07-change-control.md) | Draft-review-approve-execute workflows and deterministic validation |
| 8 | [Policy & Guardrails](./08-policy-guardrails.md) | Tool restrictions, approval gates, autonomy tiers, and budgets |
| 9 | [Observability & Audit](./09-observability.md) | Traces, action trails, debugging, privacy, and audit records |
| 10 | [Autonomy & Notifications](./10-autonomy-notifications.md) | Scheduled agents, event-driven agents, routing, escalation, and digests |
| 11 | [Testing & Hardening](./11-testing-hardening.md) | Trajectory tests, evaluations, prompt-injection defense, and red-team coverage |
| 12 | [UX & Usability](./12-ux-usability.md) | Tenancy, RBAC, onboarding, review flows, and error prevention |
| 13 | [Risk Framework & Checklists](./13-risk-framework.md) | Decision matrices, readiness checks, and build-vs-buy guidance |

---

## Core Principles

1. **Draft Before Action** - The safest default is for agents to draft work: proposed answers, ticket updates, pull requests, workflow changes, or API requests. Execution requires explicit policy and audit coverage.
2. **Least Privilege by Intent** - Agents request capabilities by intent, such as "read customer profile" or "open support ticket," not raw secrets or broad API keys.
3. **Human Gates Where Risk Changes** - Human review belongs at boundaries where the blast radius changes: external communication, production writes, money movement, sensitive data access, and irreversible changes.
4. **Observability Is Part of the Product** - Every model call, tool call, credential grant, policy decision, approval, and outbound action needs traceability.
5. **The Agent Is Not Special** - Agent work should flow through the same validation, review, and compliance systems as human work.

---

## Quick Start: Mental Model

```
+---------------------------------------------------------+
|                 YOUR ENTERPRISE SYSTEMS                 |
|  Apps, APIs, databases, documents, tickets, messages     |
|  Identity, policy, CI/CD, analytics, customer records    |
+----------------------------+----------------------------+
                             |
                  +----------v----------+
                  |    POLICY PLANE     |  What agents CAN do
                  |  rules, approvals   |
                  +----------+----------+
                             |
            +----------------v----------------+
            |          AGENT RUNTIME          |  How agents THINK
            |  models, memory, tools          |
            +----------------+----------------+
                             |
                  +----------v----------+
                  |    ACTION PLANE     |  How work LANDS
                  | drafts, PRs, APIs   |
                  +----------+----------+
                             |
                  +----------v----------+
                  |   OBSERVABILITY     |  How you SEE it
                  | traces, audit logs  |
                  +---------------------+
```

---

## Alternatives Covered

This guide does not prescribe a single stack. For each architectural layer, it covers multiple approaches:

| Layer | Options Covered |
|-------|----------------|
| **Managed Agent Platforms** | [Amazon Bedrock AgentCore](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/what-is-bedrock-agentcore.html), [Microsoft Foundry Agent Service](https://learn.microsoft.com/en-us/azure/ai-foundry/agents/overview) |
| **LLM Runtime** | Claude Agent SDK, OpenAI Agents SDK / Codex CLI, LangGraph, CrewAI, Pydantic AI, Mastra, direct API wrappers |
| **Tools & Interop** | Function tools, hosted tools, MCP, A2A, OpenAPI tools, code interpreters, browser tools |
| **Task Orchestration** | Redis Streams, BullMQ, AWS SQS, RabbitMQ, Temporal, durable workflows |
| **Sandboxing** | Containers, microVMs, managed code interpreters, browser runtimes, Azure Container Apps, AWS AgentCore Runtime |
| **Identity & Secrets** | Microsoft Entra ID, Okta, Cognito, OAuth/OIDC, HashiCorp Vault, AWS Secrets Manager, Azure Key Vault |
| **Action Delivery** | Pull requests, workflow approvals, ticket drafts, API execution gates, human-in-the-loop checkpoints |
| **Observability** | OpenTelemetry, CloudWatch, Azure Monitor, Grafana, Datadog, New Relic |
| **State Storage** | PostgreSQL, Redis, object storage, vector databases, managed memory services |

---

## Contributing

Found an error? Have a better pattern? Contributions are welcome.

- **Issues** - Open an issue for questions, suggestions, or corrections
- **Pull Requests** - Submit a PR for content improvements or new examples
- **Discussions** - Use GitHub Discussions for broader architectural questions

Please keep contributions focused on reusable patterns and architecture, not vendor-specific marketing.

---

## About

This guide is built by the team at **[Cloudgeni](https://cloudgeni.ai)**.

Every pattern here comes from designing, building, and reviewing production agent systems. We open-sourced it because the same architectural questions keep coming up across enterprise teams.

---

## License

The guide text is released under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Use it, adapt it, share it - just give credit.

Code snippets are released under [MIT](https://opensource.org/licenses/MIT).
