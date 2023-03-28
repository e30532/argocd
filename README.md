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

12. install open liberty operator

13. deploy a liberty operator application via argocd.
```
% argocd app create myola --repo https://github.com/e30532/argocd.git --path ola --dest-server https://kubernetes.default.svc --dest-namespace default
application 'myola' created
```

14. sync the app
```
% argocd app sync myola      

Name:               argocd/myola
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          default
URL:                https://localhost:8080/applications/myola
Repo:               https://github.com/e30532/argocd.git
Target:             
Path:               ola
SyncWindow:         Sync Allowed
Sync Policy:        <none>
Sync Status:        Synced to  (9e81421)
Health Status:      Healthy

Operation:          Sync
Sync Revision:      9e81421a27979217da8b993a4d95af1709b40ba8
Phase:              Succeeded
Start:              2023-03-29 00:34:11 +0900 JST
Finished:           2023-03-29 00:34:13 +0900 JST
Duration:           2s
Message:            successfully synced (all tasks run)

GROUP                KIND                    NAMESPACE  NAME         STATUS  HEALTH  HOOK  MESSAGE
apps.openliberty.io  OpenLibertyApplication  default    libertydiag  Synced                openlibertyapplication.apps.openliberty.io/libertydiag created
```

15. check the app in arg CD console(https://localhost:8080)

<img width="1490" alt="image" src="https://user-images.githubusercontent.com/22098113/228290838-2ec0cae1-5194-4b63-ad85-595c404a44c8.png">

16. check the corresponding OCP object

```
% oc get OpenLibertyApplications -n default
NAME          IMAGE                     EXPOSED   RECONCILED   AGE
libertydiag   quay.io/ibm/libertydiag   true      True         2m48s
% oc get deployment -n default             
NAME          READY   UP-TO-DATE   AVAILABLE   AGE
libertydiag   1/1     1            1           2m48s
% oc get pod -n default                    
NAME                           READY   STATUS    RESTARTS   AGE
libertydiag-6bc8487d5d-bxb29   1/1     Running   0          2m51s
```

17. If we update the applicate state in the github repository and sync the application, the state of OCP objects is also updated 
<img width="411" alt="image" src="https://user-images.githubusercontent.com/22098113/228292003-6fe94869-ddfc-43c5-93fd-2d5b11683d14.png">

<img width="1255" alt="image" src="https://user-images.githubusercontent.com/22098113/228292242-c16fc9d0-4e19-43a0-9544-71ba02b95a91.png">

```
% oc get pod -n default
NAME                           READY   STATUS    RESTARTS   AGE
libertydiag-6bc8487d5d-bxb29   1/1     Running   0          6m27s
libertydiag-6bc8487d5d-h5hcq   1/1     Running   0          2m3s
```


Conculsion: 
By using Argo CD, we can controle the application definition in Github. 

<img width="692" alt="image" src="https://user-images.githubusercontent.com/22098113/228293205-c139b99b-5004-4482-bbd9-5c64890e8b62.png">


Reference:
https://argo-cd.readthedocs.io/en/stable/getting_started/

