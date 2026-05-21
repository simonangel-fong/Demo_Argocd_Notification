# Demo_Argocd_Notification — Implementation Plan

## Project Goal

A demo + reusable tech reference: **ArgoCD Notifications wired up 2 ways to 2 targets**, running on Docker Desktop Kubernetes.

| Configuration method                  | Slack (webhook) | GitHub Actions (`repository_dispatch`) |
| ------------------------------------- | --------------- | -------------------------------------- |
| **Inline** (in `helm/values.yaml`)    | ✅              | ✅                                     |
| **Manifest** (standalone CM + Secret) | ✅              | ✅                                     |

Two sample apps each subscribe via one of the two configuration paths — same outcome, two idiomatic paths. Demonstrates flexibility and serves as a personal tech note.

---

## Placeholders to fill at runtime

| Placeholder                    | Where it's used                           | Example                                          |
| ------------------------------ | ----------------------------------------- | ------------------------------------------------ |
| `<SLACK_WEBHOOK_URL>`          | `notifications/secret.yaml` (gitignored)  | `https://hooks.slack.com/services/T.../B.../...` |
| `<SLACK_CHANNEL>`              | sample app `application.yaml` annotations | `#argocd-demo`                                   |
| `<GITHUB_PAT>`                 | `notifications/secret.yaml` (gitignored)  | `github_pat_...` (fine-grained)                  |
| `<GITHUB_OWNER>/<GITHUB_REPO>` | `helm/values.yaml` webhook URL            | `simonangel-fong/Demo_Argocd_Notification`       |

---

## Step 0 — Prerequisites

**Goal:** Confirm the local environment is ready before touching anything.

- [ ] Docker Desktop installed, Kubernetes enabled (Settings → Kubernetes → Enable)
- [ ] `kubectl` points at `docker-desktop` context (`kubectl config current-context`)
- [ ] `helm` v3 installed
- [ ] `argocd` CLI installed
- [ ] Slack workspace + incoming webhook URL ready → `<SLACK_WEBHOOK_URL>`
- [ ] Slack channel chosen → `<SLACK_CHANNEL>` (e.g. `#argocd-demo`)
- [ ] GitHub fine-grained PAT created with **Contents: Read** and **Metadata: Read** for this repo → `<GITHUB_PAT>`
- [ ] GitHub repo owner/name noted → `<GITHUB_OWNER>/<GITHUB_REPO>`

---

## Step 1 — Install ArgoCD via Helm

**Goal:** Get ArgoCD running locally with the notifications controller enabled.

- [ ] Add Helm repo: `helm repo add argo https://argoproj.github.io/argo-helm && helm repo update`
- [ ] Flesh out [helm/values.yaml](../helm/values.yaml) with: `server.service.type: ClusterIP`, `configs.params."server.insecure": true`, `notifications.enabled: true`
- [ ] Install: `helm install argocd argo/argo-cd -n argocd --create-namespace -f helm/values.yaml`
- [ ] Wait for pods: `kubectl get pods -n argocd -w`
- [ ] Port-forward: `kubectl port-forward svc/argocd-server -n argocd 8080:443`
- [ ] Retrieve admin password: `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d`
- [ ] `argocd login localhost:8080`
- [ ] Smoke test: open https://localhost:8080, log in, see empty app list

---

## Step 2 — Build the inline notifications path (Slack + GitHub)

**Goal:** Configure both notification targets via the `notifications.*` block in [helm/values.yaml](../helm/values.yaml).

- [ ] Create `notifications/secret.yaml.template` documenting the secret shape (real `secret.yaml` is gitignored). Secret keys: `slack-webhook`, `github-token`
- [ ] Apply the real (gitignored) `argocd-notifications-secret` to the cluster
- [ ] In `values.yaml` `notifications.notifiers`, define:
  - `service.slack` → references `$slack-webhook`
  - `service.webhook.github` → URL `https://api.github.com/repos/<GITHUB_OWNER>/<GITHUB_REPO>/dispatches`, headers `Authorization: token $github-token`, `Accept: application/vnd.github+json`
- [ ] In `notifications.templates`, define `app-deployed`, `app-sync-failed`, `app-health-degraded`. Each has:
  - `slack:` block (channel-friendly message text)
  - `webhook.github:` block (method `POST`, body `{"event_type":"argocd-deployed","client_payload":{"app":"{{.app.metadata.name}}","status":"{{.app.status.sync.status}}","revision":"{{.app.status.operationState.syncResult.revision}}"}}`)
  - Use `toJson` filter on any dynamic field that may contain quotes/newlines (e.g., error messages)
- [ ] In `notifications.triggers`, bind `on-deployed`, `on-sync-failed`, `on-health-degraded` to the templates
- [ ] Upgrade release: `helm upgrade argocd argo/argo-cd -n argocd -f helm/values.yaml`
- [ ] Verify: `kubectl get cm argocd-notifications-cm -n argocd -o yaml` shows the inline config
- [ ] Tail controller logs: `kubectl logs -n argocd deploy/argocd-notifications-controller -f`

---

## Step 3 — Build the manifest notifications path (Slack + GitHub)

**Goal:** Same notifications, but configured via standalone manifests instead of Helm values. Demonstrates flexibility.

- [ ] Create `notifications/configmap.yaml` — uses **separate trigger names** (`on-deployed-v2`, `on-sync-failed-v2`, `on-health-degraded-v2`) and template names (`app-deployed-v2`, etc.) to avoid colliding with the inline path
- [ ] Make the manifest-path templates visually distinguishable (e.g., prefix Slack messages with `[via manifest path]`)
- [ ] `notifications/secret.yaml.template` already documents shared secret shape — no new secret needed
- [ ] Apply: `kubectl apply -f notifications/configmap.yaml`
- [ ] Verify both inline + manifest triggers coexist in the merged ConfigMap
- [ ] Re-tail controller logs to confirm no parse errors

---

## Step 4 — GitHub Actions consumer workflow

**Goal:** Workflow that fires on `repository_dispatch` and prints the payload — proves the round-trip.

- [ ] Create `.github/workflows/argocd-notify.yml`
- [ ] Trigger: `on: repository_dispatch: types: [argocd-deployed, argocd-sync-failed, argocd-health-degraded]`
- [ ] Single job, steps: echo `${{ github.event.action }}`, echo `${{ toJSON(github.event.client_payload) }}`, print a friendly message per event type
- [ ] Manually test with `curl` or `gh api` before wiring ArgoCD:
  ```sh
  gh api repos/<OWNER>/<REPO>/dispatches -f event_type=argocd-deployed \
    -F client_payload[app]=test -F client_payload[status]=Synced
  ```
- [ ] Commit + push so the workflow is on the **default branch** (required for `repository_dispatch` to find it)

---

## Step 5 — Sample apps + subscriptions

**Goal:** Two minimal apps, each subscribed via one of the two configuration paths.

- [ ] `sample-app/`: `deployment.yaml` (nginx:1.25), `service.yaml`, `application.yaml`
  - Annotations: `notifications.argoproj.io/subscribe.on-deployed.slack: <SLACK_CHANNEL>`, `notifications.argoproj.io/subscribe.on-deployed.github: ""`
  - Wire `on-sync-failed` and `on-health-degraded` the same way
- [ ] `sample-app-v2/`: same file structure, but annotations use the manifest-path triggers (`on-deployed-v2`, etc.)
- [ ] Apply both Application CRs: `kubectl apply -f sample-app/application.yaml -f sample-app-v2/application.yaml`
- [ ] Watch them sync in the ArgoCD UI

---

## Step 6 — End-to-end demo run (capture artifacts)

**Goal:** Trigger each event type for each app and capture proof.

- [ ] Initial sync of both apps → expect `on-deployed` to Slack + GitHub Actions for both
- [ ] Screenshot Slack messages (both inline and manifest variants — the `[via manifest path]` prefix distinguishes them)
- [ ] Capture link to the GitHub Actions run
- [ ] Bump image tag (`nginx:1.25` → `nginx:1.26`), commit, sync → expect another deployed notification
- [ ] Break one app (set image to `nginx:nonexistent-tag`) → expect `on-sync-failed` or `on-health-degraded`
- [ ] Restore broken app to a good state
- [ ] Save screenshots/links into `docs/` for the README to reference

---

## Step 7 — Documentation

**Goal:** Make the repo a useful public reference.

- [ ] Write [docs/tech.md](tech.md) — critical/common issues + debug recipes (outline below)
- [ ] Refine root [README.md](../README.md) — what this repo is, why a viewer should care, the 2×2 matrix, screenshots, quickstart pointer to this plan

### `docs/tech.md` outline

- ArgoCD admin password retrieval
- Notifications controller not loading new config → check controller logs, restart pod
- Slack messages not arriving → secret key naming (`slack-webhook` must match `$slack-webhook` ref)
- GitHub webhook returns 404 → workflow file must be on default branch; PAT scope wrong
- GitHub webhook returns 422 → JSON body malformed; use `toJson` filter
- Template syntax errors render literally (silent failure) — use `argocd admin notifications template notify` to test
- Trigger name collisions between inline + manifest paths
- Docker Desktop Kubernetes context gotchas (wrong context, resource limits)
- Subscription annotation format quirks (`subscribe.<trigger>.<service>: <recipient>`)

### `README.md` outline

- Header + one-line pitch
- "What this demo shows" — the 2×2 matrix
- Architecture diagram (mermaid)
- Screenshots of Slack + GitHub Actions output
- Quickstart (3–5 commands)
- Links to `docs/plan.md` and `docs/tech.md`
- Repo layout
- Skills demonstrated (GitOps, Helm, ArgoCD, Kubernetes, CI/CD integration)

---

## Final Repo Layout

```
.github/workflows/
  argocd-notify.yml              # repository_dispatch consumer
helm/
  values.yaml                    # ArgoCD + INLINE notifications
  README.md                      # install command
notifications/
  configmap.yaml                 # MANIFEST notifications
  secret.yaml.template           # secret shape (real one gitignored)
sample-app/                      # subscribes via inline triggers
  deployment.yaml
  service.yaml
  application.yaml
sample-app-v2/                   # subscribes via manifest triggers
  deployment.yaml
  service.yaml
  application.yaml
docs/
  plan.md                        # this plan (checklists)
  tech.md                        # troubleshooting & gotchas
.gitignore                       # ignores notifications/secret.yaml, .env
README.md                        # public-facing project page
```

---

## Known gotchas (flagged early)

- **Webhook template body** needs `toJson` filter on dynamic content with quotes/newlines: `{{ .app.status.operationState.message | toJson }}`
- **Template syntax errors render literally** instead of failing — test with `argocd admin notifications template notify` before deploying
- **Trigger names must be unique** across the inline + manifest configs, or they'll fight
- **GitHub `repository_dispatch`** only fires workflows on the default branch
