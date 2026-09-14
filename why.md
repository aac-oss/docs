# Why AAC exists

## The problem

Enterprises are accumulating AI agents. Different teams build them with different frameworks (LangGraph, OpenAI Agents SDK, custom code), different model providers (OpenAI, Anthropic, internal vLLM), different tools, and different execution environments. This is normal and expected. There is no reason to force every team onto one agent SDK.

The problem is not that agents exist. The problem is that they are unmanaged.

Building an agent is getting easier every quarter. Running agents reliably, safely, and repeatedly inside an enterprise is not.

Enterprises need to:

- **Launch agents** manually, by API, on a schedule, by event, or as part of a workflow
- **Control credentials** so agents don't embed long-lived secrets in prompts or source code
- **Control tools** so agents only call what they're authorized to call
- **Control model access** so sensitive workloads stay on approved endpoints
- **Choose where agents execute** and under what security boundary
- **Require human approval** before agents take sensitive actions
- **Govern agent-to-agent delegation** so one agent can't silently invoke arbitrary others
- **Handle retries, cancellation, notifications, and workflow state** without rebuilding this infrastructure per agent
- **Reconstruct who initiated an action, what happened, and why** for audit and incident response

Today, each team either ignores most of these requirements or builds bespoke solutions. The result is fragmented operations, inconsistent security posture, and no central visibility.

## What teams do today (and why it breaks)

| Approach | What breaks |
|----------|-------------|
| Cron + bash scripts | No credentials management, no approvals, no audit, no workflow composition |
| Internal wrappers per framework | Maintenance burden multiplies with each framework; no cross-framework consistency |
| Cloud-native agent services (Bedrock, Copilot Studio) | Locked to one cloud and one vendor's agent model; doesn't cover agents outside that ecosystem |
| "Just deploy it on Kubernetes" | Solves placement but not governance: no tool authorization, approval gates, or delegation control |

## Who needs this

### Agent developer

Builds agent implementations using whatever framework fits the problem. Needs to publish agents without rebuilding enterprise controls in application code. Defines what the agent does; the platform handles how it runs.

**Pain without AAC:** every agent needs bespoke credential handling, tool authorization, audit logging, and deployment scripts. Each framework reinvents the same operational scaffolding.

### Platform / automation engineer

Turns agents into repeatable, governed automation. Owns the operational contract between agent developers and the organization.

**Pain without AAC:** builds and maintains custom glue for each agent. No consistent way to schedule, monitor, or compose agents across frameworks.

### Security / org admin

Controls identities, credentials, tools, models, and approval policy across the agent estate.

**Pain without AAC:** each team handles secrets differently. No central view of what agents can access. No audit trail that spans agents, teams, and frameworks. Approval workflows are ad-hoc or nonexistent.

### Operations / business user

Launches approved automation, provides inputs, approves steps, inspects results. Does not write agent code or manage infrastructure.

**Pain without AAC:** needs developer help to run agents. No standard interface for launching, monitoring, or approving agent work.

## When this matters

### Scenarios: Agent Runs

**Scheduled operational analysis.** An SRE team runs an incident-analysis agent every morning at 9am. The agent reviews the last 24 hours of alerts, correlates patterns, and produces a summary. The platform schedules the run, injects credentials for the monitoring API, enforces read-only tool access, notifies the SRE channel on completion, and retains the run history.

**Governed remediation.** An investigation agent analyzes a production incident and proposes a fix that requires modifying a production database. Policy requires human approval before any write operation against production. The workflow pauses, sends an approval request to the on-call engineer via Slack, and waits. On approval, the remediation agent executes. On denial, the agent receives a structured denial and reports it.

**Mixed-framework workflow.** A data pipeline uses three agents from different teams: a LangGraph agent for data extraction, a custom Python agent for transformation, and a deterministic validation step. The workflow template composes all three into a single run with state passing between steps. Each agent runs in its own container with its own policy.

**Event-driven automation.** A webhook from the monitoring system fires when a critical alert triggers. The platform launches a triage workflow. The agent investigates, proposes a remediation if recoverable, escalates to a human if not.

**Controlled delegation.** A supervisor agent needs data from the finance database, but only the finance-data agent has those credentials. Delegation policy allows this specific pair. The finance-data agent runs in its own sandbox, returns results. The full delegation chain is recorded in the audit trail.

### Scenarios: Agent Serving

**Interactive operations assistant.** An engineer opens a session with an operations assistant. The agent has access to read-only monitoring tools, runbook search, and incident history. They work through a problem together over multiple turns. Session policy limits the agent to read-only operations. The full session is audited.

**RAG-backed knowledge endpoint.** The platform exposes a governed endpoint backed by an agent with access to internal documentation and architecture decision records. Teams query it through an API or chat interface. Credentials for the knowledge backend are managed by the platform, not embedded in the agent.

**Customer-facing tool-enabled conversation.** A support agent is served as an endpoint that customers interact with. The agent can look up order status, check shipping, and initiate returns through governed tool calls. Sensitive actions (refunds above a threshold) require escalation to a human agent.

## The deciding question

Can a platform team turn a heterogeneous agent estate into repeatable, authorized, scheduled, and auditable automation without forcing one framework or vendor?

If yes, this product has a reason to exist.
