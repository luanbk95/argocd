# ARGOCD

## Install

sudo apt install -y git
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm upgrade --install argocd argo/argo-cd -n argocd --create-namespace

1. kubectl port-forward service/argocd-server -n argocd 8080:443
2. kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

```
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```

argocd login localhost:8080

Retrieve admin password
argocd admin initial-password -n argocd

Change the password using the command:
argocd account update-password

# Operation
Register A Cluster To Deploy Apps

kubectl config get-contexts -o name
argocd cluster add kubernetes-admin@cluster.local


Create An Application From A Git Repository
argocd app create guestbook --repo https://github.com/argoproj/argocd-example-apps.git --path guestbook --dest-server https://kubernetes.default.svc --dest-namespace default

argocd app get guestbook
argocd app sync guestbook


https://kubernetes.default.svc is a cluster where Argo CD is running on



## Projects
The AppProject CRD is the Kubernetes resource object representing a logical grouping of applications

```
cat <<'YAML' > project-demo.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: demo
  namespace: argocd
spec:
  description: Demo project
  sourceRepos:
  - '*'
  destinations:
  - namespace: '*'
    server: https://kubernetes.default.svc
  clusterResourceWhitelist:
  - group: '*'
    kind: '*'
YAML
```

## Applications
The Application CRD is the Kubernetes resource object representing a deployed application instance in an environment
- source: Git
- destination: target cluster and namespace

### Helm app

```
cat <<'YAML' > app-helm-guestbook.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: helm-guestbook
  namespace: argocd  ### argocd only create app in argocd namespace
spec:
  project: demo
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: helm-guestbook
    helm:
      releaseName: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: helm-guestbook  ## namespace that resources will be deployed
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
YAML
```
### Kustomize app
```
cat <<'YAML' > app-kustomize-guestbook.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kustomize-guestbook
  namespace: argocd
spec:
  project: demo
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: kustomize-guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: kustomize-guestbook
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
YAML
```
# App of apps

App of Apps is a pattern in ARGOCD:

You create a root Application and point to a Git repo that includes sub-Application manifest files. When syncing root, Argo CD will apply those sub-Applications, each of them will sync its workloads afterwards














