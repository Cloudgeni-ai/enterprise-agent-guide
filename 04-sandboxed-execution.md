# Chapter 4: Sandboxed Execution

> How to isolate agent workloads so a compromised or confused agent cannot harm the wider environment.

---

## Why Sandboxing Matters

Agents execute code, parse files, call APIs, browse web apps, and read untrusted content. Prompt injection, bad tool descriptions, malicious documents, or model mistakes can turn a helpful agent into an execution path for data exfiltration or unwanted actions.

The sandbox boundary should limit:

- filesystem access
- network access
- credentials
- process lifetime
- CPU, memory, and disk use
- reachable tools
- reachable tenant data
- side effects in external systems

Sandboxing is not a replacement for authorization. It is a second boundary after policy and before tool execution.

---

## Isolation Levels

```
Process sandbox
  < container
  < microVM
  < managed isolated runtime
  < separate account/subscription/project
```

| Level | Best For | Limits |
|-------|----------|--------|
| **Process sandbox** | Local development, trusted internal tools | Weak isolation |
| **Container** | Most worker-hosted enterprise agents | Kernel shared with host |
| **MicroVM** | Higher-risk code execution and tenant isolation | More platform complexity |
| **Managed code interpreter** | Data analysis and generated code | Provider constraints |
| **Managed browser runtime** | Web automation without local browser risk | Session and cost limits |
| **Separate cloud boundary** | High-regulation or high-risk tenants | Operational overhead |

Use stronger isolation as data sensitivity and tool power increase.

---

## Default Sandbox Contract

A production sandbox should provide:

- clean working directory per run
- no inherited developer credentials
- no persistent home directory unless explicitly mounted
- allowlisted tools and package managers
- short-lived scoped credentials injected at runtime
- egress controls
- blocked metadata endpoints
- resource limits
- run timeout
- full command and tool-call logging
- artifact extraction after completion

---

## Container Pattern

Containers are the default for many enterprise workers.

```dockerfile
FROM node:22-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates curl git jq python3 python3-pip \
    && rm -rf /var/lib/apt/lists/*

RUN useradd -m -u 10001 agent
USER agent
WORKDIR /workspace

ENV NODE_ENV=production
CMD ["node", "/app/worker.js"]
```

Dispatch pattern:

```typescript
await sandbox.run({
  image: "enterprise-agent-worker:2026-05-08",
  command: ["node", "/app/run-agent.js"],
  env: {
    RUN_ID: run.id,
    POLICY_DIGEST: policy.digestId,
  },
  secrets: await credentialBroker.issueForRun(run.id),
  limits: {
    timeoutMs: 15 * 60_000,
    memoryMb: 2048,
    cpu: 2,
  },
  networkPolicy: "restricted-egress",
});
```

The worker image should contain stable tool versions. Avoid installing tools dynamically inside the run unless the package source is pinned and policy-approved.

---

## Network Controls

Most agent escapes are data movement problems. Restrict egress.

Typical allowlist:

- your API gateway
- approved identity provider endpoints
- approved tool/MCP endpoints
- object storage for artifacts
- package registries only when needed
- model provider endpoint

Always block:

- cloud metadata endpoints such as `169.254.169.254`
- tenant-private networks unless explicitly required
- arbitrary outbound SMTP
- unreviewed webhook endpoints
- public paste/file-sharing domains

For browser agents, route traffic through a proxy that logs domains and enforces policy.

---

## Code Execution

Code interpreters are useful for data analysis, transformation, charting, and validation. Treat generated code as untrusted.

Controls:

- no ambient credentials
- separate input and output directories
- read-only source mounts unless edits are required
- package install controls
- CPU, memory, disk, and wall-clock limits
- output-size limits
- artifact malware scanning for uploaded/generated files
- redaction before trace/log emission

Managed options include Amazon Bedrock AgentCore Code Interpreter and Microsoft Foundry Code Interpreter. They reduce hosting work, but you still need tenant authorization, data classification, and retention rules around what you upload.

---

## Browser Automation

Browser tools are powerful because they can operate web applications that lack APIs. They are risky because a web page can inject instructions, hide content, or trigger unexpected actions.

Controls:

- use a managed or isolated browser runtime
- start with a clean profile per run
- disable password managers and persistent cookies by default
- record page URL, title, and action summaries
- require confirmation before submit, purchase, send, delete, or permission changes
- restrict domains
- capture screenshots for audit when policy permits
- never expose raw session cookies to the model

Amazon Bedrock AgentCore Browser is one managed option for cloud-hosted browser automation. For self-hosted browser work, use Playwright inside a locked-down container or microVM.

---

## File Handling

Agents often process PDFs, spreadsheets, exports, images, and archives.

Rules:

- scan uploaded files before opening
- unpack archives into size-limited directories
- reject symlinks and path traversal during extraction
- convert files with sandboxed tools
- preserve original files separately from generated outputs
- track provenance for every artifact
- enforce document ACLs before retrieval

For sensitive documents, prefer extracting only the relevant text slices instead of giving the agent the entire file.

---

## Managed Runtime Options

| Option | Strengths | Watch Out For |
|--------|-----------|---------------|
| **Amazon Bedrock AgentCore Runtime** | Managed serverless runtime, isolated sessions, long-running agents, identity, observability, MCP/A2A support | AWS platform fit, feature availability |
| **Microsoft Foundry Agent Service hosted agents** | Azure-managed hosting for code-based agents, identity and RBAC integration | Preview/region status for hosted-agent features |
| **Azure Container Apps Jobs** | Container jobs with Azure networking and managed identity | You own agent runtime semantics |
| **AWS ECS Fargate / Lambda** | Managed AWS compute with IAM integration | Lambda duration limits; ECS orchestration work |
| **Kubernetes Jobs** | Maximum portability and control | You own cluster security and operations |
| **Modal / serverless containers** | Fast iteration and bursty workloads | Vendor-specific runtime and networking model |

Choose the runtime based on data boundary, identity integration, task duration, tool needs, and operational ownership.

---

## Resource Limits

Set budgets per task type.

| Task Type | Timeout | Memory | Network | Notes |
|-----------|---------|--------|---------|-------|
| Document summary | 5 min | 1 GB | docs + model | Low side-effect risk |
| Data analysis | 30 min | 4-16 GB | storage + model | Watch output size and PII |
| Ticket drafting | 10 min | 1 GB | ticket + docs + model | Draft-only default |
| Browser workflow | 30 min | 2-4 GB | approved domains | Approval before submit |
| Code modification | 60 min | 4 GB | repo + package registry | PR-only default |

Budget failures should be explicit: stop, summarize what happened, and ask for continuation if needed.

---

## Design Checklist

- [ ] Sandbox has no ambient developer or host credentials
- [ ] Every run gets a clean workspace
- [ ] Network egress is restricted and logged
- [ ] Metadata endpoints are blocked
- [ ] Code and browser tools run in isolated environments
- [ ] High-risk browser submits require confirmation
- [ ] Artifacts are scanned, tracked, and access-controlled
- [ ] Resource limits and timeout behavior are defined per task type
