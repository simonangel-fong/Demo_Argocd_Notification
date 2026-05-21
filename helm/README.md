```sh
# Add the Argo Helm repo
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Install ArgoCD
helm upgrade --install argocd argo/argo-cd \
  -n argocd \
  --create-namespace \
  -f helm/values.yaml
# NAME: argocd
# LAST DEPLOYED: Thu May 21 13:25:15 2026
# NAMESPACE: argocd
# STATUS: deployed
# REVISION: 1
# TEST SUITE: None
# NOTES:
# In order to access the server UI you have the following options:

# 1. kubectl port-forward service/argocd-server -n argocd 8080:443

#     and then open the browser on http://localhost:8080 and accept the certificate

# 2. enable ingress in the values file `server.ingress.enabled` and either
#       - Add the annotation for ssl passthrough: https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#option-1-ssl-passthrough
#       - Set the `configs.params."server.insecure"` in the values file and terminate SSL at your ingress: https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#option-2-multiple-ingress-objects-and-hosts


# After reaching the UI the first time you can login with username: admin and the random password generated during the installation. You can find the password by running:

# kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# (You should delete the initial secret afterwards as suggested by the Getting Started Guide: https://argo-cd.readthedocs.io/en/stable/getting_started/#4-login-using-the-cli)


# confirm
kubectl get pods -n argocd -w
# NAME                                                READY   STATUS      RESTARTS   AGE
# argocd-application-controller-0                     1/1     Running     0          51s
# argocd-applicationset-controller-5f945f8b66-4hb9j   1/1     Running     0          51s
# argocd-dex-server-75468956d6-dmmrg                  1/1     Running     0          51s
# argocd-notifications-controller-85fcb968f8-4btx8    1/1     Running     0          51s
# argocd-redis-6c6699d6c4-l9ghs                       1/1     Running     0          51s
# argocd-redis-secret-init-6vjh2                      0/1     Completed   0          2m47s
# argocd-repo-server-64958754b6-wvk5s                 1/1     Running     0          51s
# argocd-server-86cfc7f4c9-bvjxj                      1/1     Running     0          51s

# Check the notifications controller is healthy
kubectl get deploy -n argocd argocd-notifications-controller
# NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
# argocd-notifications-controller   1/1     1            1           67s

# Retrieve the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d && echo
# BumBIePtTNeOjQ-B

# port-forward 
kubectl port-forward svc/argocd-server -n argocd 8080:443


# login
argocd login localhost:8080 --username admin --insecure

```
