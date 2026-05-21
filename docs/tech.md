# Tech Notes — ArgoCD Notifications

Troubleshooting and gotchas hit while building this demo. Organized by the moment-of-failure: log line or symptom → root cause → fix.

Companion to [plan.md](plan.md) (the implementation steps) and the per-folder step READMEs ([helm/](../helm/README.md), [notifications/](../notifications/README.md), [argocd_app/](../argocd_app/README.md), [.github/workflows/](../.github/workflows/README.md)).

---

## ArgoCD basics

### Initial admin password

```sh
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d && echo
```

Delete the secret after first login per the [official guide](https://argo-cd.readthedocs.io/en/stable/getting_started/#4-login-using-the-cli) — it stays in the cluster otherwise.

### Notifications controller config reload

The controller watches `argocd-notifications-cm` and `argocd-notifications-secret` and reloads automatically — usually. When in doubt, force it:

```sh
kubectl rollout restart deploy/argocd-notifications-controller -n argocd
kubectl logs -n argocd deploy/argocd-notifications-controller --tail=20
```

A clean reload looks like:
```
ArgoCD Notifications Controller is starting ... version v3.4.2
loading configuration 9001
invalidated cache for resource ... argocd-notifications-cm
invalidated cache for resource ... argocd-notifications-secret
Controller is running.
```

---

## Slack delivery problems

### Symptom: `invalid_auth` in controller logs

```
Failed to notify recipient {slack project-gitops-demo}: invalid_auth
```

**Root cause:** `service.slack` (the named notifier) calls `slack.com/api/chat.postMessage` with `Authorization: Bearer <value>`. It expects a **bot token** (`xoxb-…`), not a Slack incoming webhook URL.

If you have an incoming webhook URL and try to use it as a `service.slack` token, Slack rejects it as invalid auth.

**Two fixes, pick one:**

| Path                          | Notifier type            | Token value needed              |
| ----------------------------- | ------------------------ | ------------------------------- |
| Bot token (this repo's choice) | `service.slack`          | Bot User OAuth Token (`xoxb-…`) |
| Webhook URL                   | `service.webhook.slack`  | The incoming webhook URL itself |

If switching to bot token: create a Slack app at [api.slack.com/apps](https://api.slack.com/apps), add `chat:write` scope, install to workspace, copy the Bot User OAuth Token, store as `slack-token` in the secret, reference as `$slack-token` in the notifier config.

### Symptom: log says `Sending notification` (success), but no Slack message appears

The controller treats Slack's HTTP-200 envelope as success regardless of `ok:false` inside. So a `channel_not_found` response from Slack still looks like a successful send in the log.

**Diagnose directly with curl:**

```sh
TOKEN=$(kubectl get secret argocd-notifications-secret -n argocd \
  -o jsonpath='{.data.slack-token}' | base64 -d)

curl -s -X POST https://slack.com/api/chat.postMessage \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json; charset=utf-8" \
  -d '{"channel":"project-gitops-demo","text":"direct curl test"}'
```

| Response                                                     | Root cause                                                                 | Fix                                                                                                                                |
| ------------------------------------------------------------ | -------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| `{"ok":true, ... }` + message appears                        | Token + channel + bot membership are all fine                              | Problem is elsewhere — likely the dedup annotation (see next section)                                                              |
| `{"ok":false,"error":"channel_not_found"}`                   | Bot is not a channel member, or can't resolve the channel name from token  | `/invite @<bot>` in the channel, **or** use the channel ID (`C0XXXXXXX`) instead of the name in the subscription annotation        |
| `{"ok":false,"error":"not_in_channel"}`                      | Bot installed but not invited                                              | `/invite @<bot>`                                                                                                                   |
| `{"ok":false,"error":"missing_scope"}`                       | Token lacks `chat:write`                                                   | Add scope at [api.slack.com/apps](https://api.slack.com/apps) → OAuth & Permissions, then **reinstall app** to refresh token       |
| `{"ok":false,"error":"invalid_auth"}`                        | Wrong/expired token, or token is actually a webhook URL                    | Verify the secret content; switch to webhook notifier path if you only have a URL                                                  |

### Secret key naming gotcha

Notifier configs reference secret keys with `$<key>`. The `<key>` must match what's in the secret exactly — and the *name* of the value (`slack-webhook` vs `slack-token`) does NOT influence what protocol is used. The notifier *type* (`service.slack` vs `service.webhook.slack`) decides that.

| Notifier config                              | Required secret key | What value goes in |
| -------------------------------------------- | ------------------- | ------------------ |
| `service.slack: token: $slack-token`         | `slack-token`       | Bot token `xoxb-…` |
| `service.webhook.slack: url: $slack-webhook` | `slack-webhook`     | Webhook URL        |

A mismatch surfaces in the controller log as:
```
config referenced '$slack-webhook', but key does not exist in secret
```

---

## The dedup annotation

### Symptom: log shows `already sent to ...` even though no message arrived

```
Notification about condition 'on-deployed.[0].XXXX' already sent to '{slack project-gitops-demo}' using the configuration in namespace argocd
```

**Root cause:** ArgoCD Notifications records sent conditions in a single annotation on the Application CR:

```sh
kubectl get app sample-app -n argocd -o yaml | grep -E "notified\."
# notified.notifications.argoproj.io: '{"<revision>:<trigger>:<condition>:<recipient>":<unix-ts>}'
```

This annotation is written **before** delivery is confirmed — so if Slack returns an error, the controller still considers the send "complete" and refuses to retry. The next sync sees the annotation and logs `already sent`.

**Fix:**

```sh
kubectl annotate app sample-app -n argocd notified.notifications.argoproj.io-
```

Then immediately re-trigger (`argocd app sync sample-app`, or push a new commit). Watch the next tick — it should log `Sending notification ...` (not `already sent`).

### Why simply re-syncing doesn't clear it

The dedup key includes the **revision hash** (`oncePer: app.status.operationState.syncResult.revision`), but it persists across the lifecycle of that revision. To force a re-send for the same revision, you have to strip the annotation. To re-send for a new revision, you need a new git commit + sync.

---

## GitHub webhook problems

### Symptom: controller logs `Sending notification ... to {github }` but no Actions run appears

The webhook notifier logs success on any HTTP response that isn't a connection error. Authentication and authorization errors at GitHub are silent from ArgoCD's perspective.

**Diagnose directly with curl, using the same token ArgoCD uses:**

```sh
TOKEN=$(kubectl get secret argocd-notifications-secret -n argocd \
  -o jsonpath='{.data.github-token}' | base64 -d)

curl -sS -i -X POST https://api.github.com/repos/<owner>/<repo>/dispatches \
  -H "Authorization: token $TOKEN" \
  -H "Accept: application/vnd.github+json" \
  -d '{"event_type":"argocd-deployed","client_payload":{"app":"diag"}}'
```

| HTTP status                                                                  | Root cause                                                  | Fix                                                                                                                              |
| ---------------------------------------------------------------------------- | ----------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `204` no body                                                                | Success                                                     | If workflow still doesn't fire: workflow file isn't on default branch, or `event_type` doesn't match `on: repository_dispatch:`  |
| `403 Resource not accessible by personal access token`                       | PAT lacks **Contents: Read and write**                      | Edit fine-grained PAT → Repository permissions → Contents → Read and write → Save (no need to regenerate the token)              |
| `401 Bad credentials`                                                        | Token expired/invalid                                       | Regenerate, update `notifications/secret.yaml`, `kubectl apply`, restart controller                                              |
| `404 Not Found`                                                              | PAT not authorized for the repo, or repo path wrong         | Re-scope the fine-grained PAT to include this repo                                                                               |
| `422 Unprocessable Entity`                                                   | Body malformed — `event_type` missing or unquoted, etc.     | Check the `webhook.github.body` template for template-render errors                                                              |

### Symptom: `repository_dispatch` accepted (204) but workflow doesn't run

**Root cause:** `repository_dispatch` events only fire workflows on the **default branch**. If the workflow file is on a feature branch, GitHub silently accepts the dispatch but doesn't run anything.

**Fix:** merge/push the workflow to `master` (or whatever your default is).

### Symptom: `gh api ... -f client_payload[app]=...` returns "accepts 1 arg(s)"

```
gh api ... -f client_payload[app]=manual-test
# accepts 1 arg(s), received 7
```

**Root cause:** bash/zsh glob expansion on the square brackets. Either no file matches (works) or there are matching files (breaks).

**Fix:** quote the bracket args:

```sh
gh api repos/<owner>/<repo>/dispatches \
  -f event_type=argocd-deployed \
  -f 'client_payload[app]=manual-test' \
  -f 'client_payload[status]=Synced'
```

---

## Template & trigger gotchas

### Symptom: `cannot fetch phase from <nil>` error in controller log

```
failed to execute when condition: cannot fetch phase from <nil>
 | app.status.operationState.phase in ['Succeeded'] and app.status.health.status == 'Healthy'
```

**Root cause:** A fresh Application that has never been synced has `app.status.operationState == nil`. The expression dereferences `.phase` on nil → crash.

**Fix:** null-guard the expression:

```yaml
when: app.status.operationState != nil and app.status.operationState.phase in ['Succeeded'] and app.status.health.status == 'Healthy'
```

### Template syntax errors render literally (silent failure)

If a template's Go-template syntax is malformed (e.g., unbalanced `{{...}}`), the controller still tries to send the message — it just substitutes the literal source text for that block. Slack receives a message containing `{{.app.metadata.name}}` instead of the resolved name.

**Test templates locally before deploying:**

```sh
argocd admin notifications template notify \
  app-deployed sample-app \
  --recipient slack:project-gitops-demo
```

### `toJson` filter for multiline / quoted dynamic content

When interpolating fields that may contain quotes, newlines, or unescaped JSON-breaking characters (e.g., `.app.status.operationState.message`), pipe through `toJson`:

```yaml
body: |
  {
    "message": {{ .app.status.operationState.message | toJson }}
  }
```

Without `toJson`, a multiline error message produces invalid JSON and the receiving webhook returns 422.

---

## Subscription annotations

Live on the **Application CR**, not anywhere else. Format:

```
notifications.argoproj.io/subscribe.<trigger>.<service>: <recipient>
```

Recipient meaning depends on the service:

| Service                | Recipient value     | Example                                             |
| ---------------------- | ------------------- | --------------------------------------------------- |
| `slack`                | Channel name or ID  | `project-gitops-demo` or `C0XXXXXXXXX`              |
| `webhook.<name>`       | Empty string `""`   | URL is fixed in the notifier config                 |

Example for both services together:

```yaml
metadata:
  annotations:
    notifications.argoproj.io/subscribe.on-deployed.slack: project-gitops-demo
    notifications.argoproj.io/subscribe.on-deployed.github: ""
```

---

## Inline vs manifest tradeoffs

| Aspect                                              | Inline (Helm `values-inline.yaml`)                                | Manifest (`notifications/configmap.yaml` + `kubectl`)              |
| --------------------------------------------------- | ----------------------------------------------------------------- | ------------------------------------------------------------------ |
| Source of truth                                     | One file: helm values                                             | Two files: helm values (base) + ConfigMap                          |
| Update workflow                                     | `helm upgrade -f values-inline.yaml`                              | `kubectl apply -f notifications/configmap.yaml`                    |
| Diff between methods                                | `diff helm/values-inline.yaml helm/values-manifest.yaml`          | Same                                                               |
| Fits GitOps                                         | Yes — the entire stack is one Helm release                        | Yes — but ownership is split between Helm and standalone manifests |
| Risk of drift                                       | Low (one tool manages it)                                         | Moderate — applying configmap.yaml over Helm-rendered CM causes Helm to want to revert it on next upgrade |
| Bootstrap order                                     | Single `helm install` provisions everything                       | `helm install` then `kubectl apply -f configmap.yaml`              |
| Real-world fit                                      | Common in greenfield Helm-managed clusters                        | Common when notifications are owned by a different team/tool than ArgoCD |

This repo demonstrates both because real-world stacks tend to be hybrid — production Helm deployments often have per-environment ConfigMap patches applied separately.

---

## Docker Desktop Kubernetes gotchas

- **Wrong kubectl context** — `kubectl config get-contexts` should show `docker-desktop` as current. If you also use kind/minikube, double-check before running ArgoCD commands.
- **Port-forward must stay running** — closing the shell that runs `kubectl port-forward svc/argocd-server -n argocd 8080:443` kills UI access. Use a dedicated terminal or `nohup` it.
- **Resource limits** — Docker Desktop defaults are usually fine for this stack (one ArgoCD install, one nginx app), but if pods stay `Pending`, check Docker Desktop → Settings → Resources for CPU/memory allocation.

---

## Quick diagnostic flowchart

When a notification doesn't arrive:

1. **Is the trigger firing?** `kubectl logs -n argocd deploy/argocd-notifications-controller --tail=30` — look for `Trigger 'X' TRIGGERED`. If not → check trigger `when:` clause, app status, null-guard.
2. **Is the controller attempting to send?** Look for `Sending notification ... to '{<service> <recipient>}'`. If you see `already sent` → strip the dedup annotation.
3. **Does the controller log show `Failed to notify`?** The error message tells you exactly what went wrong.
4. **Does the log say success but the message never arrives?** Bypass ArgoCD: hit the upstream API (`chat.postMessage` for Slack, `/dispatches` for GitHub) with the same secret from `notifications/secret.yaml` via curl. The upstream's error code tells you the truth.
