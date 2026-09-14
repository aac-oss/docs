# Backlog

Ideas and future capabilities. Not committed to a timeline.

## Tool collections and registry

Tools in AAC are declarative YAML (method, URL, params, response_path), which makes them trivially shareable. A community registry would let teams reuse standard tool definitions instead of rewriting them for every playbook.

Common candidates: MongoDB, PostgreSQL, Slack, PagerDuty, Jira, GitHub, Datadog, Prometheus, AWS, GCP.

Collections bundle related operations together. A `mongodb` collection gives you `find`, `insert`, `update`, `delete` in one import.

```yaml
tools:
  - collection: aac-oss/mongodb
    version: "1.2"
  - collection: aac-oss/pagerduty
    version: "0.5"
  - name: my-custom-thing
    http:
      method: POST
      url: "https://internal.api/do-stuff"
```

Open questions:
- Versioning and dependency resolution
- Where collections live (Git repos, OCI registry, dedicated registry)
- How credentials map to collection tools (collection declares what it needs, playbook provides it)
- Trust and verification for community-contributed collections

Same pattern as Ansible Galaxy, Terraform Registry, Homebrew taps.

## Prehook agents

Gate execution before the main agent runs. Multiple types: script, playbook, approval check, policy evaluation. A prehook failure blocks the main run.

Use cases: cost estimation before expensive runs, input validation, compliance checks, rate limiting.
