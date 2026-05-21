# Demo_Argocd_Notification — Implementation Plan

## Project Goal

A demo + reusable tech reference: **ArgoCD Notifications wired up 2 ways to 2 targets**, running on Docker Desktop Kubernetes.

| Configuration method                  | Slack (webhook) | GitHub Actions (`repository_dispatch`) |
| ------------------------------------- | --------------- | -------------------------------------- |
| **Inline** (in `helm/values.yaml`)    | ✅              | ✅                                     |
| **Manifest** (standalone CM + Secret) | ✅              | ✅                                     |

**Approach: vertical slices, test-as-you-go.** Configure the inline method, prove it works (Slack, then GitHub Actions), tear it down, then start the manifest method from a clean slate and prove it the same way. One reusable sample app is used for both methods — same workload, different notification configuration source.

**Cleanup between methods is total:** `helm uninstall` + delete the `argocd` namespace + drop the sample app. This guarantees the second method stands entirely on its own with no leftover state from the first.

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

- [x] Docker Desktop installed, Kubernetes enabled (Settings → Kubernetes → Enable)
- [x] `kubectl` points at `docker-desktop` context (`kubectl config current-context`)
- [x] `helm` v3 installed
- [x] `argocd` CLI installed (in WSL)
- [x] Slack workspace + incoming webhook URL ready → `<SLACK_WEBHOOK_URL>`
- [x] Slack channel chosen → `<SLACK_CHANNEL>` (e.g. `#argocd-demo`)
- [x] GitHub fine-grained PAT created with **Contents: Read** and **Metadata: Read** for this repo → `<GITHUB_PAT>`
- [x] GitHub repo owner/name noted → `<GITHUB_OWNER>/<GITHUB_REPO>`

---

## Step 1 — Install ArgoCD via Helm

**Goal:** Get ArgoCD running locally with the notifications controller enabled.

- [x] Add Helm repo: `helm repo add argo https://argoproj.github.io/argo-helm && helm repo update`
- [x] Flesh out [helm/values.yaml](../helm/values.yaml) with: `server.service.type: ClusterIP`, `configs.params."server.insecure": true`, `notifications.enabled: true`
- [x] Install: `helm install argocd argo/argo-cd -n argocd --create-namespace -f helm/values.yaml`
- [x] Wait for pods: `kubectl get pods -n argocd -w`
- [x] Port-forward: `kubectl port-forward svc/argocd-server -n argocd 8080:443`
- [x] Retrieve admin password: `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d`
- [x] `argocd login localhost:8080`
- [x] Smoke test: open https://localhost:8080, log in, see empty app list

---

## Step 2 — Build the inline notifications path (Slack + GitHub)

**Goal:** Configure both notification targets via the `notifications.*` block in [helm/values.yaml](../helm/values.yaml).

- [x] Create `notifications/secret.yaml.template` documenting the secret shape (real `secret.yaml` is gitignored). Secret keys: `slack-webhook`, `github-token`
- [x] Apply the real (gitignored) `argocd-notifications-secret` to the cluster
- [x] In `values.yaml` `notifications.notifiers`, define:
  - `service.slack` → references `$slack-webhook`
  - `service.webhook.github` → URL `https://api.github.com/repos/<GITHUB_OWNER>/<GITHUB_REPO>/dispatches`, headers `Authorization: token $github-token`, `Accept: application/vnd.github+json`
- [x] In `notifications.templates`, define `app-deployed`, `app-sync-failed`, `app-health-degraded`. Each has:
  - `slack:` block (channel-friendly message text)
  - `webhook.github:` block (method `POST`, body `{"event_type":"argocd-deployed","client_payload":{"app":"{{.app.metadata.name}}","status":"{{.app.status.sync.status}}","revision":"{{.app.status.operationState.syncResult.revision}}"}}`)
  - Use `toJson` filter on any dynamic field that may contain quotes/newlines (e.g., error messages)
- [x] In `notifications.triggers`, bind `on-deployed`, `on-sync-failed`, `on-health-degraded` to the templates
- [x] Upgrade release: `helm upgrade argocd argo/argo-cd -n argocd -f helm/values.yaml`
- [x] Verify: `kubectl get cm argocd-notifications-cm -n argocd -o yaml` shows the inline config
- [x] Tail controller logs: `kubectl logs -n argocd deploy/argocd-notifications-controller -f`

---

## Step 3 — Create the reusable sample app

**Goal:** A minimal nginx workload + an ArgoCD `Application` CR. Same app reused for both methods; only the subscription annotations change between slices.

- [x] Create `apps/deployment.yaml` (nginx:1.25, 1 replica)
- [x] Create `apps/service.yaml` (ClusterIP)
- [x] Create `argocd_app/application.yaml` — `Application` CR pointing at `apps/` in this repo
  - Subscription annotations left as commented placeholders — each slice fills them in
- [x] Commit + push so ArgoCD can pull from the repo
- [x] Apply: `kubectl apply -f argocd_app/application.yaml`
- [x] Verify in ArgoCD UI: the app shows Synced + Healthy

---

## Step 4 — INLINE method × Slack target

**Goal:** Prove the inline notifications config delivers messages to Slack.

- [ ] Add Slack subscription annotation to `argocd_app/application.yaml`:
  - `notifications.argoproj.io/subscribe.on-deployed.slack: project-gitops-demo`
  - `notifications.argoproj.io/subscribe.on-sync-failed.slack: project-gitops-demo`
  - `notifications.argoproj.io/subscribe.on-health-degraded.slack: project-gitops-demo`
- [ ] `kubectl apply -f argocd_app/application.yaml`
- [ ] Trigger an event:
  - Deployed: bump image tag in `apps/deployment.yaml` (e.g. nginx:1.25 → 1.26), commit, push, sync
  - Sync failed: temporarily point image to `nginx:nonexistent-tag`, commit, push, observe degraded notification, then revert
- [ ] Confirm Slack messages arrived (✅ deployed, ⚠️ degraded)
- [ ] Save screenshot to `docs/screenshots/inline-slack-deployed.png` (and one for failure)
- [ ] If nothing arrives: check `kubectl logs -n argocd deploy/argocd-notifications-controller --tail=100` for delivery errors

---

## Step 5 — GitHub Actions consumer workflow

**Goal:** A workflow that fires on `repository_dispatch` and prints the payload. Needed before the GitHub-target slices can be tested.

- [ ] Create `.github/workflows/argocd-notify.yml`
- [ ] Trigger: `on: repository_dispatch: types: [argocd-deployed, argocd-sync-failed, argocd-health-degraded]`
- [ ] Single job, steps: echo `${{ github.event.action }}`, echo `${{ toJSON(github.event.client_payload) }}`, print a friendly message per event type
- [ ] Commit + push so the workflow is on the **default branch** (required for `repository_dispatch` to find it)
- [ ] Manually test with `gh` or `curl` before wiring ArgoCD:
  ```sh
  gh api repos/simonangel-fong/Demo_Argocd_Notification/dispatches \
    -f event_type=argocd-deployed \
    -F client_payload[app]=manual-test -F client_payload[status]=Synced
  ```
- [ ] Verify the workflow run appears under the repo's Actions tab

---

## Step 6 — INLINE method × GitHub Actions target

**Goal:** Prove the inline notifications config delivers `repository_dispatch` events to the workflow.

- [ ] Add GitHub subscription annotation to `argocd_app/application.yaml` (alongside the Slack ones):
  - `notifications.argoproj.io/subscribe.on-deployed.github: ""`
  - `notifications.argoproj.io/subscribe.on-sync-failed.github: ""`
  - `notifications.argoproj.io/subscribe.on-health-degraded.github: ""`
- [ ] `kubectl apply -f argocd_app/application.yaml`
- [ ] Trigger a deploy event (image tag bump, sync)
- [ ] Confirm GitHub Actions run fires — `path: "inline"` should appear in the printed `client_payload`
- [ ] Capture the run URL for the README
- [ ] If nothing arrives: check controller logs for HTTP errors (401 = PAT issue, 404 = wrong repo, 422 = bad JSON body)

---

## Step 7 — Tear down the inline method (clean slate before manifest)

**Goal:** Remove all inline-method state so the manifest method is proven entirely on its own.

- [ ] Delete the sample app: `kubectl delete -f argocd_app/application.yaml`
- [ ] Uninstall ArgoCD: `helm uninstall argocd -n argocd`
- [ ] Delete the namespace (removes the notifications secret too): `kubectl delete ns argocd`
- [ ] Revert `argocd_app/application.yaml` subscription annotations to placeholders (or remove them — manifest slice rewrites them anyway)
- [ ] Verify clean: `kubectl get ns argocd` returns NotFound

---

## Step 8 — Install ArgoCD for the manifest method

**Goal:** Reinstall ArgoCD with notifications enabled but **no inline notifier/template/trigger config**. The manifest will supply everything.

- [ ] Create `helm/values-base.yaml` (or temporarily comment out the `notifications.notifiers/templates/triggers` blocks in `helm/values.yaml`) — keep `notifications.enabled: true` only
- [ ] Decide which: simpler is to keep one `helm/values.yaml` and just comment the inline blocks for this slice. Re-enable them later if you want the final committed state to demonstrate both. **Recommended:** keep `helm/values.yaml` with the inline blocks present and document via the README that you can install with `--set notifications.notifiers=null` (or similar) to demo the manifest path. For the actual run-through here, comment them out temporarily.
- [ ] `helm install argocd argo/argo-cd -n argocd --create-namespace -f helm/values.yaml`
- [ ] Re-apply the secret: `kubectl apply -f notifications/secret.yaml`
- [ ] Verify the `argocd-notifications-cm` exists but has no `service.*` / `template.*` / `trigger.*` keys yet
- [ ] Re-port-forward, re-login

---

## Step 9 — Apply the manifest notifications config

**Goal:** Configure notifiers, templates, triggers via standalone Kubernetes manifests applied with `kubectl`.

- [ ] Create `notifications/configmap.yaml` — a complete `argocd-notifications-cm` ConfigMap with `service.slack`, `service.webhook.github`, all three templates, all three triggers. Use the **same names** as the inline path (`on-deployed`, `app-deployed`, etc.) — no `-v2` suffix needed because the inline path no longer exists in this slice
- [ ] Prefix Slack template messages with `[manifest]` so screenshots are distinguishable from the inline slice
- [ ] Apply: `kubectl apply -f notifications/configmap.yaml`
- [ ] Restart controller + tail logs: `kubectl rollout restart deploy/argocd-notifications-controller -n argocd && kubectl logs -n argocd deploy/argocd-notifications-controller --tail=50`

---

## Step 10 — MANIFEST method × Slack target

**Goal:** Prove the manifest-configured notifications deliver to Slack.

- [ ] Re-apply the sample app with the Slack subscription annotations:
  - `notifications.argoproj.io/subscribe.on-deployed.slack: project-gitops-demo` (etc.)
- [ ] `kubectl apply -f argocd_app/application.yaml`
- [ ] Trigger deploy and failure events as in Step 4
- [ ] Confirm Slack messages arrived with `[manifest]` prefix
- [ ] Save screenshot to `docs/screenshots/manifest-slack-deployed.png`

---

## Step 11 — MANIFEST method × GitHub Actions target

**Goal:** Prove the manifest-configured notifications deliver `repository_dispatch` events.

- [ ] Add GitHub subscription annotations to `argocd_app/application.yaml` (alongside Slack)
- [ ] `kubectl apply -f argocd_app/application.yaml`
- [ ] Trigger a deploy event
- [ ] Confirm GitHub Actions run fires — `path: "manifest"` should appear in the printed `client_payload`
- [ ] Capture the run URL

---

## Step 12 — Final committed state + documentation

**Goal:** Leave the repo in a state a public viewer can read top-to-bottom. Both methods' artifacts are committed; the README explains how to choose between them.

- [ ] Restore `helm/values.yaml` to its full state (inline notifications block present) — so the file demonstrates the inline method even when the cluster is currently running the manifest method
- [ ] Sample app `application.yaml` ends up with the final subscription annotations chosen (recommend: leave it with the **manifest-method** annotations since that's the more configurable real-world pattern, and call this out in the README)
- [ ] Write [docs/tech.md](tech.md) — troubleshooting (outline below)
- [ ] Refine root [README.md](../README.md) — 2×2 matrix, screenshots, quickstart, repo layout, skills demonstrated
- [ ] Add `docs/screenshots/` with the captured proof images

### `docs/tech.md` outline

- ArgoCD admin password retrieval
- Notifications controller not loading new config → check logs, restart pod
- Slack messages not arriving → secret key naming (`slack-webhook` must match `$slack-webhook` ref)
- GitHub webhook returns 401/404/422 — root causes per code
- Template syntax errors render literally (silent failure) — test with `argocd admin notifications template notify`
- Inline vs manifest tradeoffs (what we learned by trying both)
- Docker Desktop Kubernetes context gotchas
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
  README.md                      # install command log
notifications/
  configmap.yaml                 # MANIFEST notifications
  secret.yaml.template           # secret shape (real one gitignored)
  README.md                      # apply/verify command log
apps/                            # reusable workload manifests
  deployment.yaml
  service.yaml
argocd_app/                      # ArgoCD Application CR for the workload
  application.yaml
  README.md                      # apply/verify command log
docs/
  plan.md                        # this plan
  tech.md                        # troubleshooting & gotchas
  screenshots/                   # proof images
.gitignore                       # ignores notifications/secret.yaml, *.tfvars
README.md                        # public-facing project page
```

---

## Known gotchas (flagged early)

- **Webhook template body** needs `toJson` filter on dynamic content with quotes/newlines: `{{ .app.status.operationState.message | toJson }}`
- **Template syntax errors render literally** instead of failing — test with `argocd admin notifications template notify` before deploying
- **GitHub `repository_dispatch`** only fires workflows on the default branch
- **Helm vs kubectl ownership** — applying a ConfigMap with `kubectl` over a Helm-managed one causes drift on next `helm upgrade`. The vertical-slice approach (full uninstall between methods) sidesteps this entirely
