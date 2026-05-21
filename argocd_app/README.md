# Step 3 — Sample app + subscriptions

Defines an ArgoCD `Application` (`sample-app`) that pulls the nginx workload from [`../apps/`](../apps/). Subscription annotations on this CR are what wire each trigger to a notification recipient.

## Apply the app

```sh
kubectl apply -f argocd_app/application.yaml
# application.argoproj.io/sample-app created

kubectl get app -n argocd
# NAME         SYNC STATUS   HEALTH STATUS
# sample-app   Synced        Healthy

# Verify the workload landed in the destination namespace
kubectl get pods,svc -n default
# sample-app-<hash>   1/1   Running
# service/sample-app  ClusterIP   80/TCP
```

## Trigger a deploy event (golden path)

The `on-deployed` trigger uses `oncePer: revision` — it only fires once per Git revision. Each retest needs a new commit.

```sh
# Bump apps/deployment.yaml: nginx:1.25 -> nginx:1.26
git add -A
git commit -m "bump image to trigger argocd notification"
git push origin master
argocd app sync sample-app

# Tail the controller — look for TRIGGERED followed by a Sending notification line
kubectl logs -n argocd deploy/argocd-notifications-controller --tail=30
```

## Debug recipes

### Strip the notification dedup state

ArgoCD records which conditions it has already notified in a single annotation on the Application CR (`notified.notifications.argoproj.io`). The controller writes it **before** confirming the downstream delivery, so a failed send (e.g., Slack `channel_not_found`) still marks the condition as "sent". Result: subsequent retries log `already sent` and never actually call the API again.

```sh
# Inspect
kubectl get app sample-app -n argocd -o yaml | grep -E "notified\."

# Strip; the next controller tick will attempt a real send
kubectl annotate app sample-app -n argocd notified.notifications.argoproj.io-
```

### Force-recreate the Application CR

When annotations get out of sync or you want a clean slate without touching helm:

```sh
kubectl replace --force -f argocd_app/application.yaml
# application.argoproj.io "sample-app" deleted from argocd namespace
# application.argoproj.io/sample-app replaced
```

### Verify Slack auth directly (skip ArgoCD)

If the controller log shows `Sending notification to {slack ...}` with no error but the message never appears, prove Slack is the issue with a direct call:

```sh
TOKEN=$(kubectl get secret argocd-notifications-secret -n argocd \
  -o jsonpath='{.data.slack-token}' | base64 -d)

curl -s -X POST https://slack.com/api/chat.postMessage \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json; charset=utf-8" \
  -d '{"channel":"project-gitops-demo","text":"direct curl test"}'
# {"ok":true, ...}          -> token+channel OK; problem must be elsewhere
# {"ok":false,"error":"channel_not_found"}  -> bot not in channel, or name not resolvable
# {"ok":false,"error":"invalid_auth"}       -> wrong token
```

Fix `channel_not_found` by inviting the bot into the channel (`/invite @<bot>`) or by switching the subscription annotation to use the channel **ID** instead of the name.
