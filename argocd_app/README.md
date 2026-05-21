
```sh
# Apply the Application
kubectl apply -f argocd_app/application.yaml
# application.argoproj.io/sample-app created

kubectl get app -n argocd
# NAME         SYNC STATUS   HEALTH STATUS
# sample-app   Synced        Healthy

kubectl get pods
# NAME                         READY   STATUS    RESTARTS   AGE
# sample-app-d46dc5456-tr4sq   1/1     Running   0          68s
kubectl get svc

kubectl get svc
# NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
# kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP   71m
# sample-app   ClusterIP   10.104.209.170   <none>        80/TCP    83s

# test notification
# update image: nginx:1.25 -> nginx:1.26; git push
```

- debug

```sh
kubectl logs -n argocd deploy/argocd-notifications-controller --tail=100


kubectl replace --force -f argocd_app/application.yaml
# application.argoproj.io "sample-app" deleted from argocd namespace
# application.argoproj.io/sample-app replaced
```

- slack debug

```sh
# get token
TOKEN=$(kubectl get secret argocd-notifications-secret -n argocd -o jsonpath='{.data.slack-token}' | base64 -d)
echo "Token starts with: ${TOKEN:0:10}..."

# test
curl -s -X POST https://slack.com/api/chat.postMessage   -H "Authorization: Bearer $TOKEN"   -H "Content-Type: application/json; charset=utf-8"   -d '{"channel":"project-gitops-demo","text":"direct curl test"}'
```