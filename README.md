This document describes a simple (not realistic) scenario where ArgoCD and OCP are integrated. 

1. First, deploy OCP4.12
2. % oc login *.*.*.*:6443 -u kubeadmin -p *****
3. % oc create namespace argocd
4. % oc apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
5. argocd-redis pod will fail to be created due to the following failure.
```
% oc get event
LAST SEEN   TYPE      REASON         OBJECT                              MESSAGE
37s         Warning   FailedCreate   replicaset/argocd-redis-8f7689686   Error creating: pods "argocd-redis-8f7689686-" is forbidden: unable to validate against any security context constraint: [provider "anyuid": Forbidden: not usable by user or serviceaccount, spec.containers[0].securityContext.runAsUser: Invalid value: 999: must be in the ranges: [1000670000, 1000679999], provider "restricted": Forbidden: not usable by user or serviceaccount, provider "nonroot-v2": Forbidden: not usable by user or serviceaccount, provider "nonroot": Forbidden: not usable by user or serviceaccount, provider "hostmount-anyuid": Forbidden: not usable by user or serviceaccount, provider "machine-api-termination-handler": Forbidden: not usable by user or serviceaccount, provider "hostnetwork-v2": Forbidden: not usable by user or serviceaccount, provider "hostnetwork": Forbidden: not usable by user or serviceaccount, provider "hostaccess": Forbidden: not usable by user or serviceaccount, provider "node-exporter": Forbidden: not usable by user or serviceaccount, provider "privileged": Forbidden: not usable by user or serviceaccount]
```
To fix this, I needed to update runAsUser in the deployment.
```
% oc edit deployment.apps/argocd-redis

Before:
        runAsUser: 999

After:
        runAsUser: 1000670001
```

6. brew install argocd
```
% argocd version
argocd: v2.6.7+5bcd846.dirty
  BuildDate: 2023-03-23T17:23:10Z
  GitCommit: 5bcd846fa16e4b19d8f477de7da50ec0aef320e5
  GitTreeState: dirty
  GoVersion: go1.20.2
  Compiler: gc
  Platform: darwin/arm64
FATA[0000] Failed to establish connection to localhost:8080: dial tcp [::1]:8080: connect: connection refused 
```

7. oc port-forward svc/argocd-server -n argocd 8080:443

8. generate password
```
% argocd admin initial-password -n argocd
```

9. login
```
% argocd login localhost:8080
WARNING: server certificate had error: tls: failed to verify certificate: x509: certificate signed by unknown authority. Proceed insecurely (y/n)? y
Username: admin
Password: 
'admin:login' logged in successfully
Context 'localhost:8080' updated
```

10. update password
```
% argocd account update-password
*** Enter password of currently logged in user (admin): 
*** Enter new password for user admin: 
*** Confirm new password for user admin: 
Password updated
Context 'localhost:8080' updated
yamayoshi@yamadayoshikis-MacBook-Pro ~ %
```

11. % oc config set-context --current --namespace=argocd





Reference:
https://argo-cd.readthedocs.io/en/stable/getting_started/
