# Chapter 5: Credential Management

> Agents should request scoped authority, not hold broad secrets.

---

## The Core Problem

Agents need access to real systems: documents, tickets, CRMs, databases, repositories, messaging platforms, and internal APIs. Static credentials in prompts, environment variables, or tool descriptions turn every prompt-injection bug into a credential leak.

The pattern is a **credential broker**:

```
Agent -> request capability by intent -> policy check -> broker issues scoped grant -> tool call
```

The agent does not choose raw scopes or secrets. It asks for an intent such as:

- read this ticket
- search this workspace
- draft a customer response
- open a pull request
- query account data for this customer
- request workflow execution

The broker maps that intent to the right identity, scope, lifetime, and audit record.

---

## Identity Modes

| Mode | Meaning | Use When |
|------|---------|----------|
| **User-delegated identity** | Agent acts on behalf of a human user | User-specific data access and user-visible actions |
| **Agent identity** | Agent has its own enterprise identity | Scheduled tasks, shared agents, governed automation |
| **Service identity** | Backend service owns the action | Product-owned workflows behind strict APIs |
| **Hybrid** | User context plus agent/service constraints | Most enterprise workflows |

Avoid letting the model decide which identity mode to use. Policy should derive it from task type, tenant settings, user role, and action risk.

---

## Broker Contract

```typescript
interface CredentialRequest {
  runId: string;
  organizationId: string;
  userId?: string;
  intent:
    | 'docs:read'
    | 'ticket:read'
    | 'ticket:draft'
    | 'ticket:publish'
    | 'crm:read'
    | 'crm:update'
    | 'repo:read'
    | 'repo:pull_request'
    | 'workflow:request'
    | 'workflow:execute';
  resource: {
    system: 'sharepoint' | 'jira' | 'salesforce' | 'github' | 'internal-api';
    id?: string;
    tenant?: string;
  };
  requestedTtlSeconds: number;
}

interface CredentialGrant {
  grantId: string;
  token: string;
  expiresAt: string;
  allowedOperations: string[];
  auditContext: Record<string, string>;
}
```

The broker should:

1. authenticate the caller
2. verify the run and task are valid
3. evaluate policy
4. verify user or agent permissions
5. issue a short-lived grant
6. record an audit event
7. support revocation

---

## OAuth, OIDC, and OBO

Most enterprise systems should use OAuth/OIDC rather than API keys.

Common patterns:

- **OAuth authorization code** for user-connected apps
- **On-behalf-of (OBO)** for user-delegated access through backend services
- **Client credentials** for service-owned automation
- **Workload identity federation** for cloud-hosted workers
- **Device or browser flow** only for local developer tools

Microsoft Entra ID, Okta, Auth0, Amazon Cognito, and similar identity providers can all participate in this model. Managed agent platforms also expose identity features: Amazon Bedrock AgentCore Identity handles inbound and outbound auth patterns, and Microsoft Foundry Agent Service supports dedicated agent identities through Microsoft Entra ID.

---

## Scopes by Capability

Define named scopes at the product level. Do not expose raw provider scopes to the agent.

```yaml
capabilities:
  docs-read:
    systems: [sharepoint, google-drive, confluence]
    operations: [search, read]
    max_ttl: 15m

  ticket-draft:
    systems: [jira, zendesk, servicenow]
    operations: [read, create_draft]
    max_ttl: 15m

  ticket-publish:
    systems: [jira, zendesk, servicenow]
    operations: [publish_update]
    max_ttl: 5m
    requires_approval: true

  repo-pr:
    systems: [github, gitlab, azure-devops]
    operations: [read, branch, commit, open_pr]
    max_ttl: 30m
```

Each tool maps to one or more capabilities. Each capability maps to provider-specific scopes and API permissions behind the broker.

---

## Least Privilege by Default

| Agent Task | Required Access | Usually Not Needed |
|------------|-----------------|--------------------|
| Summarize a document | Read selected document slices | Workspace-wide read |
| Draft ticket response | Read ticket, read approved docs, create draft | Publish or delete |
| Analyze customer account | Read selected account fields | Full CRM export |
| Prepare code change | Repo read, branch, open PR | Direct merge |
| Run report | Read scoped dataset | Admin database role |
| Request workflow | Create approval request | Execute without approval |

Least privilege is easier when tools are narrow. A `create_ticket_update_draft` tool can be safe with draft-only permission. A generic `call_jira_api` tool usually cannot.

---

## Secret Storage

Use a managed secret store for static material that cannot be eliminated:

| Store | Good For |
|-------|----------|
| **HashiCorp Vault** | Cross-cloud secret brokerage, dynamic secrets, mature policy |
| **AWS Secrets Manager** | AWS-hosted services and AgentCore-adjacent workloads |
| **Azure Key Vault** | Azure-hosted services and Foundry-adjacent workloads |
| **GCP Secret Manager** | GCP-hosted services |
| **1Password / enterprise vaults** | Human-managed operational secrets |

Static secrets should be source material for short-lived grants, not directly exposed to the agent runtime.

---

## Credential Injection

Safe injection patterns:

- inject only into the tool process that needs the grant
- use environment variables only for short-lived process-local grants
- avoid writing tokens to disk
- scrub tokens from stdout, stderr, traces, and model context
- bind grants to run ID, tenant ID, resource ID, and operation
- expire grants quickly
- revoke all grants when a run ends

Tool wrapper example:

```typescript
async function withGrant<T>(
  request: CredentialRequest,
  fn: (grant: CredentialGrant) => Promise<T>
): Promise<T> {
  const grant = await broker.issue(request);
  try {
    return await fn(grant);
  } finally {
    await broker.revoke(grant.grantId);
  }
}
```

---

## Sensitive Data Handling

Credentials are not the only sensitive data. Retrieved records can contain PII, PHI, financial data, trade secrets, or privileged communications.

Controls:

- classify data before retrieval
- filter fields before passing context to the model
- preserve source ACLs in retrieval
- redact secrets and regulated fields from logs
- record which records were accessed
- prevent unrestricted export/download tools
- isolate tenants at retrieval and storage layers

For high-risk data, prefer server-side tools that answer narrow questions instead of handing raw records to the model.

---

## Revocation and Incident Response

You need a one-click way to stop an agent run.

On cancellation:

- revoke active grants
- stop running sandboxes
- invalidate browser sessions
- prevent pending actions from executing
- mark drafts as abandoned or needing review
- write an audit event
- notify owners if data or external actions were involved

For incidents, the audit trail should answer: who asked, what policy applied, what credentials were issued, what data was read, what tools ran, what changed, and who approved it.

---

## Design Checklist

- [ ] Agents request access by intent, not raw credentials
- [ ] Credential grants are short-lived and scoped to run, tenant, resource, and operation
- [ ] User-delegated, agent, and service identities are explicitly modeled
- [ ] OAuth/OIDC/OBO is preferred over static API keys
- [ ] Secret stores are used only as backing stores for brokered grants
- [ ] Tokens are redacted from logs and never placed in prompts
- [ ] Retrieved data respects source ACLs and data classification
- [ ] Cancelling a run revokes credentials and stops pending actions
