# Step 1 — Install ArgoCD via Helm

Bootstraps ArgoCD on Docker Desktop Kubernetes with the notifications controller enabled.

## Pick a values file

This repo demonstrates two methods of configuring ArgoCD Notifications. The Helm install picks one or the other up front via the `-f` flag.

| Method        | Values file                                | What it adds                                                                                          |
| ------------- | ------------------------------------------ | ----------------------------------------------------------------------------------------------------- |
| **Inline**    | [`values-inline.yaml`](values-inline.yaml) | Base ArgoCD + the full `notifications.notifiers/templates/triggers` blocks (single source of truth).  |
| **Manifest**  | [`values-manifest.yaml`](values-manifest.yaml) | Base ArgoCD only. Notifier/template/trigger config comes from [`../notifications/configmap.yaml`](../notifications/configmap.yaml) applied via `kubectl`. |

`diff helm/values-inline.yaml helm/values-manifest.yaml` shows exactly what the inline path adds on top of the base.

## Install

```sh
# Add the Argo Helm repo
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Inline method
helm upgrade --install argocd argo/argo-cd \
  -n argocd \
  --create-namespace \
  -f helm/values-inline.yaml

# OR — manifest method
helm upgrade --install argocd argo/argo-cd \
  -n argocd \
  --create-namespace \
  -f helm/values-manifest.yaml

# STATUS: deployed
# REVISION: 1
```

## Verify

```sh
# All pods Running (notifications-controller is the one this project cares about)
kubectl get pods -n argocd
# argocd-application-controller-0                     1/1   Running
# argocd-applicationset-controller-5f945f8b66-...     1/1   Running
# argocd-dex-server-...                               1/1   Running
# argocd-notifications-controller-...                 1/1   Running
# argocd-redis-...                                    1/1   Running
# argocd-repo-server-...                              1/1   Running
# argocd-server-...                                   1/1   Running

kubectl get deploy -n argocd argocd-notifications-controller
# argocd-notifications-controller   1/1   1   1
```

## Access the UI

```sh
# Get the auto-generated admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d && echo

# Forward the server port (leave running)
kubectl port-forward svc/argocd-server -n argocd 8080:443
# UI: https://localhost:8080

# CLI login from a second shell
argocd login localhost:8080 --username admin --insecure
```
