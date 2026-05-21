# Step 2 — Notifications config & secret

Holds the secret that both notification methods consume: a Slack **bot token** (`xoxb-…`) and a GitHub **fine-grained PAT** with `Contents: Read and write`.

The real `secret.yaml` is gitignored; commit only `secret.yaml.template`.

## Apply the secret

```sh
# Real values copied from terraform.tfvars into notifications/secret.yaml
kubectl apply -f notifications/secret.yaml

kubectl get secret argocd-notifications-secret -n argocd
# argocd-notifications-secret   Opaque   2   10m
```

## Roll inline config (Helm) into the cluster

The inline notifiers/templates/triggers live in [`../helm/values.yaml`](../helm/values.yaml). After editing them, push the change:

```sh
helm upgrade argocd argo/argo-cd -n argocd -f helm/values.yaml
# REVISION: 2

# Inspect the rendered ConfigMap (look for service.slack, service.webhook.github,
# template.app-deployed, trigger.on-deployed)
kubectl get cm argocd-notifications-cm -n argocd -o yaml

# Force the controller to reload (sometimes needed)
kubectl rollout restart deploy/argocd-notifications-controller -n argocd

# Confirm a clean startup — look for the two "invalidated cache for resource"
# lines (one for the CM, one for the Secret) and no error lines after.
kubectl logs -n argocd deploy/argocd-notifications-controller --tail=50
# Notifications Controller is starting ... version v3.4.2
# loading configuration 9001
# invalidated cache for resource ... argocd-notifications-cm
# invalidated cache for resource ... argocd-notifications-secret
# Controller is running.
```

## Secret key naming gotcha

Secret keys must match the `$<key>` references in the notifier config exactly.

| Notifier (values.yaml)                                      | Required secret key |
| ----------------------------------------------------------- | ------------------- |
| `service.slack: token: $slack-token`                        | `slack-token`       |
| `service.webhook.github: ... Authorization: token $github-token` | `github-token`  |

A mismatch surfaces in the controller log as:
`config referenced '$slack-webhook', but key does not exist in secret`.

---

## Manifest

```sh
kubectl apply -f notifications/configmap.yaml
# configmap/argocd-notifications-cm configured

# confirm
kubectl get cm argocd-notifications-cm -n argocd -o yaml | grep -E "^\s+(service\.|template\.|trigger\.)"
#   service.slack: |
#   service.webhook.github: |
#   template.app-deployed: |
#   template.app-health-degraded: |
#   template.app-sync-failed: |
#   trigger.on-deployed: |
#   trigger.on-health-degraded: |
#   trigger.on-sync-failed: |

# confirm in log
kubectl logs -n argocd deploy/argocd-notifications-controller --tail=20
# {"level":"info","msg":"Start processing","resource":"argocd/sample-app","time":"2026-05-21T21:06:12Z"}
# {"level":"info","msg":"invalidated cache for resource in namespace: argocd with the name: argocd-notifications-secret","time":"2026-05-21T21:06:12Z"}
# {"level":"info","msg":"Trigger 'on-deployed' TRIGGERED | revision: 8c20fad9 | templates: [app-deployed]","resource":"argocd/sample-app","time":"2026-05-21T21:06:12Z"}
# {"level":"info","msg":"Notification about condition 'on-deployed.[0].Yv81TUs0gmPTEyk-v8Lb6ZKEo0g' already sent to '{github }' using the configuration in namespace argocd","resource":"argocd/sample-app","time":"2026-05-21T21:06:12Z"}
# {"level":"info","msg":"Notification about condition 'on-deployed.[0].Yv81TUs0gmPTEyk-v8Lb6ZKEo0g' already sent to '{slack project-gitops-demo}' using the configuration in namespace argocd","resource":"argocd/sample-app","time":"2026-05-21T21:06:12Z"}
# {"level":"info","msg":"Trigger 'on-health-degraded' FAILED | revision:  | templates: [app-health-degraded]","resource":"argocd/sample-app","time":"2026-05-21T21:06:12Z"}
# {"level":"info","msg":"Trigger 'on-sync-failed' FAILED | revision:  | templates: [app-sync-failed]","resource":"argocd/sample-app","time":"2026-05-21T21:06:12Z"}
# {"level":"info","msg":"Processing completed","resource":"argocd/sample-app","time":"2026-05-21T21:06:12Z"}
```