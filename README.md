# Homelab GitOps

This repository contains the ArgoCD control-plane manifests for the homelab
application deployments.

It does not contain the application Kubernetes manifests themselves. Those stay
in the infrastructure repository:

```text
https://github.com/KeremAR/Homelab-Infrastructure.git
```

This repository tells ArgoCD which environment and service manifest paths it
should watch and sync.

---

## Repository Role

There are two Git repositories in the deployment flow:

```text
Homelab-Infrastructure
  Real Kubernetes manifests
  Jenkins updates image tags here

homelab-gitops
  ArgoCD Application manifests
  ArgoCD bootstraps and tracks apps from here
```

This split is intentional. The infrastructure repository remains the source of
truth for raw Kubernetes YAML files, while this repository is the source of
truth for ArgoCD's application tree.

---

## App of Apps Structure

The current setup uses the App of Apps pattern with an environment layer:

```text
argocd-manifests/
  root-application.yaml
  environments/
    staging.yaml
    production.yaml
    staging/
      staging-user-service.yaml
      staging-todo-service.yaml
      staging-frontend.yaml
    production/
      production-user-service.yaml
      production-todo-service.yaml
      production-frontend.yaml
```

The chain is:

```text
root-app
  -> staging
       -> staging-user-service
       -> staging-todo-service
       -> staging-frontend
  -> production
       -> production-user-service
       -> production-todo-service
       -> production-frontend
```

`root-app` is the only Application that needs to be applied manually during
bootstrap. After that, it creates the environment Applications, and the
environment Applications create the service Applications.

---

## Why Keep Staging And Production Layers

The environment layer is useful because staging and production are separate
operational units.

With this structure:

- Staging services can be added or changed without touching production apps.
- Production sync policy can later become more conservative than staging.
- Environment-wide ownership is visible in the ArgoCD UI.
- Moving to Helm later can be done by adding a separate tree such as
  `argocd-helm/` instead of rewriting this plain-manifest structure.

For a tiny demo, root could point directly to all service Applications. For this
project, the environment layer is worth keeping.

---

## Service Application Flow

Each service Application points to a plain manifest directory in the
infrastructure repository.

Example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: staging-todo-service
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/KeremAR/Homelab-Infrastructure.git
    targetRevision: main
    path: 3-Kubectl-Deploy/staging/todo-service/templates
  destination:
    server: https://kubernetes.default.svc
    namespace: staging
```

That means ArgoCD does not render Helm here. It reads YAML files from the
selected directory and applies them to the target namespace.

---

## Deployment Flow

The desired flow is:

```text
Developer / Jenkins
  -> builds image
  -> pushes image to GHCR
  -> updates image tag in Homelab-Infrastructure
  -> pushes manifest change to main
  -> ArgoCD detects the manifest drift
  -> ArgoCD syncs the target service
```

So Jenkins does not need to run `kubectl apply` for app deployments once GitOps
is fully adopted. Jenkins changes Git, and ArgoCD applies Git.

This keeps the deployment audit trail in Git:

```text
Which service changed?
Which image tag changed?
Who/what committed it?
When did ArgoCD sync it?
```

---

## Sync Policy

Current Applications use automated sync:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

Meaning:

- `automated`: ArgoCD can sync without pressing the UI Sync button.
- `prune`: resources removed from Git can be removed from the cluster.
- `selfHeal`: if a live resource is manually changed in the cluster, ArgoCD can
  restore it back to Git state.

The Sync modal in the ArgoCD UI shows options for a single manual sync
operation. It does not show every field from `syncPolicy` as pre-selected
checkboxes. The actual policy is visible in the Application manifest or live
Application YAML.

---

## Refresh vs Sync

Refresh means:

```text
Read Git and the cluster again, then compare desired state with live state.
```

Refresh does not apply changes.

Sync means:

```text
Apply the desired state from Git to the cluster.
```

Use hard refresh when ArgoCD appears to be using stale repo or manifest cache,
for example after a previously empty repository is pushed for the first time.

---

## Bootstrap

After ArgoCD is installed, apply only the root Application:

```bash
kubectl apply -f argocd-manifests/root-application.yaml
```

Then check the tree:

```bash
kubectl get application -n argocd
```

Expected Applications:

```text
root-app
staging
production
staging-user-service
staging-todo-service
staging-frontend
production-user-service
production-todo-service
production-frontend
```

---

## Future Helm Layout

When the project moves from plain manifests to Helm, keep this directory as the
plain-manifest GitOps history and add a separate tree:

```text
argocd-helm/
  root-application.yaml
  environments/
    staging.yaml
    production.yaml
```

That keeps the migration explicit. ArgoCD can then be pointed at the Helm tree
when the Helm-based deployment model is ready.
