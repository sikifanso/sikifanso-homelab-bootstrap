# sikifanso-homelab-bootstrap

GitOps bootstrap for a homelab Kubernetes cluster using ArgoCD ApplicationSets.

## How it works

The repository uses a single ArgoCD `ApplicationSet` with a Git file generator to dynamically discover and deploy applications.

1. **`bootstrap/root-app.yaml`** — An `ApplicationSet` that watches for `apps/*/config.yaml` files. For each match, it generates an ArgoCD Application using the values from that config.

2. **`apps/<app-name>/config.yaml`** — A simple YAML file declaring which Helm chart to deploy and where.

No manual Application manifests needed — add a `config.yaml`, push to Git, and ArgoCD handles the rest.

## Adding an application

Create a new directory under `apps/` with a `config.yaml`:

```
apps/
  my-app/
    config.yaml
```

The `config.yaml` provides the values consumed by the ApplicationSet template:

```yaml
name: my-app
repoURL: https://charts.example.com
chart: my-app
targetRevision: 1.0.0
namespace: my-app
```

Once pushed, the ApplicationSet generator picks it up and creates the corresponding ArgoCD Application automatically.

## Bootstrapping

Apply the root ApplicationSet to an ArgoCD-enabled cluster:

```bash
kubectl apply -f bootstrap/root-app.yaml
```

## Sync policy

All generated applications are configured with:

- **Automated sync** — changes in Git are applied automatically
- **Self-heal** — manual drift on the cluster is corrected
- **Prune** — resources removed from Git are deleted from the cluster
- **CreateNamespace** — target namespaces are created if they don't exist
- **ServerSideApply** — uses server-side apply for resource management
