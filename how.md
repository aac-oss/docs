# How AAC works

## The AAP analogy

| AAP | AAC | Role |
|-----|-----|------|
| Ansible CLI | AAC CLI | Local developer tool, useful standalone |
| ansible-core | AAC Core (pkg/) | Shared engine: parsing, validation, policy generation, container lifecycle |
| AWX / AAP | AAC Platform | Enterprise control plane |
| Playbook | Agent Playbook | Declarative operational contract |
| Execution Environment | Container (EE image) | Runtime environment for agents |
| SSH | OpenShell | Execution boundary, sandbox isolation |
| Vault | Credential Store | Secrets management |

Ansible didn't build SSH. It used SSH and added everything around it. AAC doesn't build the agent runtime. It uses OpenShell and adds everything around it.

## Architecture

![Architecture](high-level-architecture.png)

The architecture has four parts. CLI and Platform are side by side as consumers of the same core engine, not stacked.

### AAC Core (pkg/)

The shared engine. Equivalent to ansible-core.

- Playbook parsing and validation
- Policy generation (OpenShell YAML from playbook declarations)
- Container lifecycle management (podman/docker)
- Credential resolution (env vars, files, vault)
- Network and volume configuration

Both the CLI and the Platform consume this library. When the playbook format evolves, it evolves in one place.

### AAC CLI

The local developer tool. Works standalone, no platform required.

- `aac init` scaffolds a project with playbooks/, agents/, aac.yaml
- `aac run` translates the playbook into a container execution with policy enforcement
- `aac validate` checks playbooks and config
- `aac inspect` shows resolved configuration and generated policy
- `aac run --dry-run` shows what would happen without executing

The CLI does not schedule, does not send notifications, does not enforce RBAC, does not manage multi-team credentials. It is a single-user local tool for developing and testing agent playbooks.

### AAC Platform

The enterprise control plane. Adds organizational governance on top of the same playbook format.

- Schedules, triggers, and workflow orchestration
- Approval gates (workflow-defined and policy-triggered)
- RBAC, multi-team credential management
- Notifications (Slack, email, PagerDuty, webhooks)
- Alerts (repeated failures, stuck runs, budget breach)
- Cost tracking and budget enforcement
- Delegation governance
- Aggregated audit trail
- Agent Serving (interactive endpoints, sessions)

The platform never executes agents directly. It uses the same core engine and delegates execution to the container runtime.

### OpenShell

NVIDIA's open-source agent execution environment. Sits at the sandbox boundary.

- Container isolation per agent run
- Declarative policy enforcement (filesystem, network, process)
- L7 HTTP proxy intercepting every outbound call with method/path filtering
- Credential injection (secrets never written to disk)
- Audit logging of every permission decision (allow/deny)

OpenShell doesn't know what a playbook is, who launched the run, or whether an approval is needed. It runs what it's told, enforces the policy it's given, and logs what happens.

## How a playbook runs

A playbook is a declarative YAML file that binds everything together:

```yaml
playbook:
  name: weather-assistant
  agent:
    image: aac-ee:latest
  tools:
    - name: geocode
      description: "Look up coordinates for a city"
      parameters:
        type: object
        properties:
          city: {type: string}
        required: ["city"]
      http:
        method: GET
        url: "https://geocoding-api.open-meteo.com/v1/search"
        params:
          name: "{city}"
          count: "1"
        response_path: "results[0]"
  model:
    endpoint: 192.168.1.137
    port: 11434
  policy:
    filesystem:
      read_write: [/tmp]
      read_only: [/usr, /lib, /etc, /aac]
  limits:
    timeout: 3m
  inputs:
    query:
      description: "What to check"
      required: true
```

When you run `aac run weather-assistant --input "query=What's the weather in Madrid?"`:

1. **Core parses** the playbook and validates it
2. **Core generates** an OpenShell policy YAML (filesystem rules, network rules for model + tool endpoints)
3. **Core resolves** credentials from env vars, files, or vault
4. **Core creates** a container with the EE image, mounts the playbook and policy, injects credentials and inputs as env vars
5. **The container runs** the generic runner (or custom agent if `agent.source` is set)
6. **The runner** reads the playbook, sends tools to the model (Ollama), executes HTTP tool calls when the model requests them, loops until done
7. **Output streams** to the terminal. Run result is recorded.

For declarative tools (no custom code), the generic runner handles everything. For complex agents, you provide your own Python script and the runner steps aside.

## Governance at the boundary

There are two enforcement layers. They don't overlap. See [component boundaries](component-boundaries.md) for the full breakdown.

**OpenShell: hard security boundary.** Things that should never happen, regardless of who approves. No exceptions, no human override.

```
Agent -> outbound HTTP call -> OpenShell L7 proxy -> check network policy
                                      |
                      Allowed: forward to endpoint, log decision
                      Denied:  block, log decision, return error
```

The agent can use any HTTP client, any SDK, any framework. OpenShell doesn't care how the call was made. It controls what leaves the sandbox. The playbook declares which tools are allowed. The core translates that into network policy. OpenShell enforces it.

**AAC runner: soft governance layer.** Things that can happen with the right authorization. The model sees all available tools so it can reason and plan, but the runner gates execution of restricted tools until a human approves.

```
Model calls restricted tool -> runner pauses -> notifies approver -> waits
                                                        |
                                        Approved: runner executes the HTTP call
                                        Denied:   runner returns denial to model
```

OpenShell never sees the approval logic. The runner never touches network policy. Two layers, two purposes.

This is framework-agnostic by design. Governance is a property of the boundary, not the agent code.

## Both modes, same foundation

| Capability | Agent Runs | Agent Serving |
|-----------|-----------|---------------|
| Playbook definition | Yes | Yes |
| RBAC | Yes | Yes |
| Credential management | Yes | Yes |
| Policy enforcement | Yes | Yes |
| Audit trail | Yes | Yes |
| Cost tracking | Yes | Yes |
| Approvals | Pre-run gates | Per-action gates |
| Scheduling | Cron/event triggers | Uptime windows |
| Notifications | On completion/failure | On session events |

The core engine and OpenShell layer don't care which mode it is. A served agent is still a container with a policy, credentials, and governed tool access. The difference is lifecycle: runs are ephemeral, served agents are long-lived with session state.

## Boundary rules

1. **The CLI works without the platform.** A developer can `aac init`, write playbooks, `aac run`, and get real work done. No login, no server.
2. **The platform works through the CLI's format.** Same playbook YAML. Publishing pushes the same file.
3. **Connected mode is additive.** `aac run --connect` adds visibility and governance. It does not change what the playbook does.
4. **The CLI does not implement platform features.** No RBAC, no scheduling, no approval workflows in the CLI.
5. **The platform does not reimplement CLI features.** One playbook parser, one policy generator.
6. **OpenShell is an implementation detail.** Users write playbooks. The core translates those into OpenShell configs.
