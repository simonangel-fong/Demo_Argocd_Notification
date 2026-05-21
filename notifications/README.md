
```sh
# create secret
kubectl apply -f notifications/secret.yaml

# confirm
kubectl get secret argocd-notifications-secret -n argocd
# NAME                          TYPE     DATA   AGE
# argocd-notifications-secret   Opaque   2      10m

# update helm
helm upgrade argocd argo/argo-cd \
  -n argocd \
  -f helm/values.yaml
# NAME: argocd
# LAST DEPLOYED: Thu May 21 13:50:49 2026
# NAMESPACE: argocd
# STATUS: deployed
# REVISION: 2
# TEST SUITE: None

# confirm
kubectl get cm argocd-notifications-cm -n argocd -o yaml
kubectl rollout restart deploy/argocd-notifications-controller -n argocd
# deployment.apps/argocd-notifications-controller restarted

kubectl logs -n argocd deploy/argocd-notifications-controller --tail=50
# {"built":"2026-05-12T20:34:57Z","commit":"0dc6b1b57dd5bb925d5b03c3d09419ab9fb4225e","level":"info","msg":"ArgoCD Notifications Controller is starting","namespace":"argocd","time":"2026-05-21T17:52:57Z","version":"v3.4.2"}
# {"level":"info","msg":"serving metrics on port 9001","time":"2026-05-21T17:52:57Z"}
# {"level":"info","msg":"loading configuration 9001","time":"2026-05-21T17:52:57Z"}
# {"level":"info","msg":"invalidated cache for resource in namespace: argocd with the name: argocd-notifications-cm","time":"2026-05-21T17:52:57Z"}
# {"level":"info","msg":"invalidated cache for resource in namespace: argocd with the name: argocd-notifications-secret","time":"2026-05-21T17:52:57Z"}
# {"level":"warning","msg":"Controller is running.","time":"2026-05-21T17:52:57Z"}
```