# Step 5 — GitHub Actions consumer workflow

[`argocd-notify.yml`](argocd-notify.yml) is triggered by `repository_dispatch` events sent by ArgoCD Notifications' webhook notifier. It echoes the event type and `client_payload` so we can confirm both methods (inline vs manifest) end-to-end.

Note: `repository_dispatch` only fires workflows on the repo's **default branch**, so the workflow must be committed to `master` before any test can succeed.

## Manual round-trip test (no ArgoCD involved)

```sh
# Brackets must be quoted — bash globs them otherwise
gh api repos/simonangel-fong/Demo_Argocd_Notification/dispatches \
  -f event_type=argocd-deployed \
  -f 'client_payload[app]=manual-test' \
  -f 'client_payload[status]=Synced' \
  -f 'client_payload[health]=Healthy' \
  -f 'client_payload[revision]=abc1234' \
  -f 'client_payload[path]=manual'
# (no output on success)

# A new "ArgoCD Notification Consumer" run appears within seconds:
#   https://github.com/simonangel-fong/Demo_Argocd_Notification/actions
```

## Watch the ArgoCD side

When ArgoCD itself fires the dispatch, the controller log shows two `Sending notification` lines per sync (one per target):

```sh
kubectl logs -n argocd deploy/argocd-notifications-controller --tail=30
# Trigger 'on-deployed' TRIGGERED | revision: <hash>
# Sending notification ... to '{slack project-gitops-demo}'
# Sending notification ... to '{github }'        <- empty recipient = webhook URL is the destination
```

If only the `{slack ...}` line appears, the `subscribe.<trigger>.github` annotation on the Application CR is missing.

## Debug — PAT permission gotcha

If the controller logs success but no Actions run appears, the issue is almost certainly the GitHub PAT. Verify with the same token ArgoCD uses:

```sh
TOKEN=$(kubectl get secret argocd-notifications-secret -n argocd \
  -o jsonpath='{.data.github-token}' | base64 -d)

curl -sS -i -X POST https://api.github.com/repos/simonangel-fong/Demo_Argocd_Notification/dispatches \
  -H "Authorization: token $TOKEN" \
  -H "Accept: application/vnd.github+json" \
  -d '{"event_type":"argocd-deployed","client_payload":{"app":"diag","status":"Synced","health":"Healthy","revision":"diag1","path":"diag"}}'
```

| HTTP status                                                                                                 | Meaning                                              | Fix                                                                                            |
| ----------------------------------------------------------------------------------------------------------- | ---------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| `204` (no body)                                                                                             | Success — workflow run fires                         | None                                                                                           |
| `403` with `{"message":"Resource not accessible by personal access token"}`                                 | PAT is missing the right permission                  | Edit the fine-grained PAT → **Repository permissions → Contents: Read and write**; click Save  |
| `401`                                                                                                       | Token expired / wrong                                | Regenerate the PAT, update `notifications/secret.yaml`, re-apply                                |
| `404`                                                                                                       | PAT not authorized for this repo, or repo path wrong | Re-scope the fine-grained PAT to include `Demo_Argocd_Notification`                            |
| `422`                                                                                                       | Body malformed                                       | Check the `webhook.github` template for missing `event_type` field or template render error    |

The 403 was the issue we hit during Step 6 — the PAT defaulted to **Contents: Read-only**, which is insufficient for `repository_dispatch`. Required setting:

```
https://github.com/settings/personal-access-tokens
  → Edit token → Repository permissions → Contents: Read and write → Save
```

No need to regenerate the token; permission edits apply immediately.
