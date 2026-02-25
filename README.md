# sikifanso-homelab-bootstrap

GitOps bootstrap for a homelab Kubernetes cluster using ArgoCD ApplicationSets.

## How it works

The repository uses a single ArgoCD `ApplicationSet` with a Git file generator to dynamically discover and deploy applications. Each app is defined by two files: a coordinate file that specifies which Helm chart to deploy, and a values file that provides Helm value overrides.

1. **`bootstrap/root-app.yaml`** — An `ApplicationSet` that watches for `apps/coordinates/*.yaml` files. For each match, it generates a multi-source ArgoCD Application that combines the chart coordinates with the corresponding values file from `apps/values/`.

2. **`apps/coordinates/<name>.yaml`** — A YAML file declaring which Helm chart to deploy and where.

3. **`apps/values/<name>.yaml`** — A YAML file containing Helm value overrides for the application.

No manual Application manifests needed — add the two YAML files, push to Git, and ArgoCD handles the rest.

## Adding an application

Each application requires two files:

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

Once pushed, the ApplicationSet generator picks up the coordinate file and creates the corresponding ArgoCD Application automatically. The multi-source setup ensures the values file is applied to the Helm release.

### Using the sikifanso CLI

The `sikifanso` CLI can manage apps directly:

```bash
sikifanso app add      # add a new application
sikifanso app list     # list managed applications
sikifanso app remove   # remove an application
```

### Manual workflow

You can also create the two YAML files by hand and push them to Git. The result is the same — ArgoCD picks up the changes automatically.

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
