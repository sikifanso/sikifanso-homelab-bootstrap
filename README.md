# sikifanso-bootstrap

GitOps bootstrap for AI agent infrastructure on Kubernetes using ArgoCD ApplicationSets.

## How it works

The repository uses ArgoCD `ApplicationSet` resources with Git file generators to dynamically discover and deploy applications. There are three tracks:

1. **`bootstrap/root-catalog.yaml`** -- An `ApplicationSet` that watches `catalog/*.yaml` files. Only entries with `enabled: true` generate ArgoCD Applications. Each catalog entry specifies a Helm chart and is paired with values from `catalog/values/`.

2. **`bootstrap/root-app.yaml`** -- An `ApplicationSet` that watches `apps/coordinates/*.yaml` for custom Helm charts added by the user.

3. **`bootstrap/root-agents.yaml`** -- An `ApplicationSet` that watches `agents/*.yaml` for agent sandbox deployments. Each agent gets an isolated namespace with ResourceQuotas and NetworkPolicies via the agent-template Helm chart, with values from `agents/values/`.

## AI Agent Infrastructure Catalog

The catalog contains curated tools for running AI agents:

| Category | Tools | Purpose |
|----------|-------|---------|
| **gateway** | LiteLLM Proxy | LLM API routing, cost tracking, rate limiting |
| **observability** | Alloy, Langfuse, Prometheus+Grafana, Loki, Tempo | Telemetry collection, LLM tracing, metrics, logs, distributed tracing |
| **guardrails** | Guardrails AI, NeMo Guardrails, Presidio | Output validation, safety rails, PII redaction |
| **rag** | Qdrant, Text Embeddings Inference, Unstructured | Vector DB, embeddings, document parsing |
| **runtime** | Temporal, External Secrets, OPA | Workflow orchestration, secrets, policy |
| **models** | Ollama | Local LLM inference |
| **storage** | CNPG Operator, PostgreSQL (CNPG), Valkey (Redis) | CloudNativePG operator, managed PostgreSQL clusters, in-memory data store |

Enable a tool:

```bash
sikifanso catalog enable litellm-proxy
```

Each catalog entry is a YAML file:

```yaml
name: litellm-proxy
category: gateway
description: LLM API gateway with multi-provider routing, cost tracking, and rate limiting
repoURL: https://richardoc.github.io/litellm_helm-chart
chart: litellm
targetRevision: "0.0.4"
namespace: gateway
enabled: false
```

## Agent sandboxes

Agent sandboxes provide isolated namespaces for running AI agents. Each agent gets its own namespace with ResourceQuotas, NetworkPolicies, and a ServiceAccount via the agent-template Helm chart.

```
agents/
  my-agent.yaml
  values/
    my-agent.yaml
```

Manage agents with the CLI:

```bash
sikifanso agent add       # create a new agent sandbox
sikifanso agent list      # list agent sandboxes
sikifanso agent remove    # remove an agent sandbox
```

## Adding a custom application

Each custom application requires two files:

```
apps/
  coordinates/
    my-app.yaml
  values/
    my-app.yaml
```

The coordinate file provides the values consumed by the ApplicationSet template:

```yaml
name: my-app
repoURL: https://charts.example.com
chart: my-app
targetRevision: 1.0.0
namespace: my-app
```

The values file contains Helm value overrides:

```yaml
# Helm values for my-app
replicaCount: 2
```

### Using the sikifanso CLI

The `sikifanso` CLI can manage apps directly:

```bash
sikifanso app add      # add a new application
sikifanso app list     # list managed applications
sikifanso app remove   # remove an application
```

### Manual workflow

You can also create the two YAML files by hand and push them to Git. The result is the same -- ArgoCD picks up the changes automatically.

## Bootstrapping

Apply the root ApplicationSets to an ArgoCD-enabled cluster:

```bash
kubectl apply -f bootstrap/root-app.yaml
kubectl apply -f bootstrap/root-catalog.yaml
kubectl apply -f bootstrap/root-agents.yaml
```

## Sync policy

All generated applications are configured with:

- **Automated sync** -- changes in Git are applied automatically
- **Self-heal** -- manual drift on the cluster is corrected
- **Prune** -- resources removed from Git are deleted from the cluster
- **CreateNamespace** -- target namespaces are created if they don't exist
- **ServerSideApply** -- uses server-side apply for resource management
- **ApplicationsSync** -- the `applicationsSync: sync` policy ensures apps are deleted when their generator entry is removed (e.g. when a catalog entry is disabled)

The catalog ApplicationSet additionally uses `ignoreDifferences` to suppress spurious diffs on StatefulSet VolumeClaimTemplate fields and CNPG Cluster image names, paired with `RespectIgnoreDifferences` to prevent self-heal from reverting those fields.
