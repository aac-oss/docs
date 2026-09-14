# What AAC is

## Positioning

An automation controller for AI agents. Same concept as Ansible Automation Platform (AAP), but for agents instead of infrastructure automation.

AAP didn't replace Python or SSH. It operationalized automation. AAC doesn't replace agent frameworks or model providers. It operationalizes agents.

The product is not an agent framework, model-serving platform, or governance dashboard. It turns agents into governed, repeatable enterprise automation.

## Two modes of operation

```
AAC Platform
├── Agent Runs          (current focus)
│   ├── On-demand
│   ├── Scheduled
│   └── Event-driven
│
└── Agent Serving       (long-term)
    ├── Interactive endpoints
    ├── Sessions
    ├── Streaming
    ├── RAG / knowledge access
    └── Tool-enabled conversations
```

**Agent Runs** are ephemeral. Fire, execute, exit. A bounded task with a clear start and end.

**Agent Serving** is long-lived. The platform exposes a governed agent as an endpoint that users or systems interact with over multiple turns. Sessions maintain state, support streaming, and can integrate RAG or knowledge backends.

Same playbook format, same policy enforcement, same credentials, same audit trail for both. The difference is lifecycle.

## Functional requirements

### Playbook definition
- Declarative YAML playbook format: agent image, source, model endpoint, tools, credentials, policies, limits, inputs
- Declarative HTTP tool definitions with parameter substitution and response extraction
- Custom agent code when declarative tools are not sufficient
- Validation and inspection before execution

### Execution
- Launch playbooks on-demand, by schedule, by event trigger, or as part of a workflow
- Each run executes in an isolated container with its own policy
- Stream agent output to the caller
- Enforce timeout limits, support cancellation
- Full run history with inputs, outputs, duration, and exit status

### Tools and credentials
- Declare allowed tools per playbook, enforce at the network boundary
- Resolve credentials from env vars, files, or vault; inject into containers
- Platform credential store with scoping, rotation, and audit (platform tier)

### Policy enforcement
- Generate sandbox policies from playbook declarations (filesystem, network)
- Enforcement at the infrastructure level, not in agent code
- Policy-triggered approval gates for sensitive actions (platform tier)

### Governance (platform tier)
- RBAC: who can view, launch, edit, administer playbooks and credentials
- Scheduling and triggers: cron, webhooks, event-driven launch
- Workflows: compose playbooks with conditions, branching, state passing
- Approvals: workflow-defined and policy-triggered, with timeout and escalation
- Notifications: lifecycle event delivery via Slack, email, PagerDuty, webhooks
- Alerts: repeated failures, stuck runs, budget breach, credential expiry
- Cost tracking: token attribution, budget enforcement
- Delegation governance: authorized pairs, depth limits, lineage tracking
- Aggregated audit trail across runs, teams, and workflows

## MVP scope

The MVP is a working CLI. No platform, no server, no account required.

**What ships:**
- Playbook format with declarative tools
- Policy generation from playbook declarations
- Container runtime (podman/docker) with agent source mounting
- Generic tool-calling runner for declarative playbooks
- CLI commands: `init`, `run`, `validate`, `inspect`, `list`
- Credential resolution from env/file
- Dry-run mode

**What doesn't ship (platform tier):**
Scheduling, workflows, approvals, notifications, alerts, RBAC, centralized credentials, cost tracking, delegation governance, Agent Serving, platform UI.

**Success criteria:** a developer can install the CLI, scaffold a project, write a playbook, and run a governed agent without signing up for anything.

## Business model

Open-source CLI for developer adoption (bottom-up). Subscription-based Platform for enterprise governance (top-down). Same model as Red Hat with Ansible/AAP.

- **Free**: CLI, local runs, unlimited playbooks
- **Team**: platform with RBAC, shared credentials, run history, scheduling
- **Enterprise**: approval workflows, delegation governance, cost budgets, SSO, audit export
