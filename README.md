# ARGOCD

## Install

sudo apt install -y git
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm upgrade --install argocd argo/argo-cd -n argocd --create-namespace

1. kubectl port-forward service/argocd-server -n argocd 8080:443
2. kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64

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


https://kubernetes.default.svc la cluster ma noi argocd dang chay



## Projects
The AppProject CRD is the Kubernetes resource object representing a logical grouping of applications

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


## Applications
The Application CRD is the Kubernetes resource object representing a deployed application instance in an environment
- source: Git
- destination: target cluster and namespace

### Helm app
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

### Kustomize app

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

# App of apps

App of Apps là một “mẫu kiến trúc” (pattern) trong Argo CD: bạn tạo một Application gốc (root) trỏ vào một thư mục Git chứa các manifest Application con. Khi root sync, Argo CD sẽ apply các Application con đó → mỗi app con lại tự sync workload của chính nó






# Argo CD Mastery – 4-Week Hands-On Plan (for a DevOps engineer)

Dưới đây là lộ trình học “đi từ số 0 đến làm chủ Argo CD” trong 4 tuần, thiên về **lab thực chiến**, multi-env, multi-cluster, và các best-practices sản xuất. Mình kèm **bài tập, tiêu chí hoàn thành, checklist kỹ năng** và **capstone** cuối khoá.

> Lưu ý: mọi lệnh mình ghi đều kèm rõ **nơi thực thi** (server).

---

## Mục tiêu sau khoá

* Hiểu vững GitOps & Argo CD core (Application, Project, Sync, Health, Hooks, Waves, Ignore Differences).
* Vận hành **App of Apps**, **ApplicationSet** (multi-env, multi-cluster).
* Tích hợp **Helm/Kustomize**, **Secret management** (Sealed-Secrets/SOPS), **Argo Rollouts**, **Image Updater**, **Notifications**.
* Thiết kế **multi-tenancy**, **RBAC**, **SSO**, **backup/DR**, **HA**, **observability**.
* Xây dựng pipeline GitOps chuẩn Prod (dev/staging/prod), review PR → auto deploy an toàn.

---

## Tuần 0 (chuẩn bị 2–3 giờ)

**Hạ tầng lab (chọn 1 trong 2):**

* **Local**: 1 cluster `kind` để học nhanh; 2 cluster `kind` để luyện multi-cluster.
* **Cloud**: 1–2 EKS (ap-southeast-1) nếu bạn muốn sát Prod AWS.

### Cài nhanh cluster + Argo CD

**A. Tạo cluster kind (local)**

* *Run on: your laptop (Linux/macOS)*

```bash
# Install kind & kubectl (nếu chưa có) - hướng dẫn chuẩn trên trang chủ
# Create a quick single-node cluster
kind create cluster --name=argo-lab
kubectl cluster-info
kubectl get nodes
```

**B. Cài Argo CD (namespace: argocd)**

* *Run on: your laptop (kubectl context -> kind-argo-lab)*

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server
```

**C. Lấy mật khẩu admin & đăng nhập web (port-forward)**

* *Run on: your laptop*

```bash
# Get initial password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo

# Port-forward để dùng UI
kubectl -n argocd port-forward svc/argocd-server 8080:443
# Mở https://localhost:8080 (user: admin, pass ở trên)
```

**D. Cài argocd CLI**

* *Run on: your laptop*

```bash
# Install argocd CLI (Linux)
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd && sudo mv argocd /usr/local/bin/
argocd version
```

---

## Tuần 1 – Nền tảng Argo CD & GitOps (6–8 giờ)

**Kiến thức:**

* GitOps principles, Pull vs Push CD.
* Argo CD objects: **Application**, **AppProject**, Source/Target, SyncPolicy (Auto/Manual), Prune, SelfHeal.
* Health checks, Sync status, Diff, **Ignore Differences**.
* Tích hợp **Helm/Kustomize** cơ bản.

**Lab 1 – App Hello World (Helm & Kustomize)**

1. Tạo repo Git `gitops-demo` chứa 2 app: `helm-guestbook/` và `kustomize-guestbook/`.
2. Tạo **AppProject** “demo”, **Application** trỏ đến repo đó.
3. Bật/tắt **Auto-Sync**, thử sửa chart để thấy self-heal/prune.

* *Run on: your laptop (kubectl context -> argo-lab)*

```bash
# Create a Project
cat <<'YAML' | kubectl apply -f -
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

# Create an Application (Helm example)
cat <<'YAML' | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: helm-guestbook
  namespace: argocd
spec:
  project: demo
  source:
    repoURL: https://github.com/<your-user>/<your-repo>.git
    targetRevision: main
    path: helm-guestbook
    helm:
      releaseName: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: guestbook
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
YAML
```

**Bài tập cuối tuần 1 (tiêu chí đạt):**

* Deploy được 1 app Helm và 1 app Kustomize, bật Auto-Sync + Prune + SelfHeal.
* Sửa giá trị trong repo → Argo CD tự reconcile OK.
* Hiểu và dùng **argocd app diff**, **ignoreDifferences** (ví dụ trường managedFields).

---

## Tuần 2 – App of Apps, Multi-Env, Multi-Cluster (6–8 giờ)

**Kiến thức:**

* **App of Apps** pattern (root app quản lý các app con).
* **ApplicationSet**: generators (list, git, cluster, matrix), templates.
* **Cluster management**: đăng ký **external cluster** vào Argo CD, tách env (dev/stg/prod).
* **Sync waves & hooks** (preSync, postSync), **sync options**.

**Lab 2 – App of Apps cho dev/stg/prod**

* Cấu trúc repo:

```
apps/
  root/
    application.yaml (app-of-apps)
  envs/
    dev/app-a/
    stg/app-a/
    prod/app-a/
```

* App root “bootstrap” tạo các Application con theo env.

**Lab 3 – ApplicationSet multi-cluster**

* Đăng ký 2 cluster (local kind và một EKS hoặc 2 kind).

* Dùng **Cluster generator** để tạo Application cho mỗi cluster.

* *Run on: your laptop*

```bash
# (Nếu có cluster thứ 2) đăng ký vào Argo CD
argocd login localhost:8080 --username admin --password <pwd> --insecure    # Run on: your laptop
argocd cluster add <kube-context-name-of-2nd-cluster>                        # Run on: your laptop
```

**Bài tập cuối tuần 2:**

* Hoàn thiện App of Apps quản lý ≥3 app cho dev/stg/prod.
* ApplicationSet rollout cùng cấu hình trên 2 cluster.
* Thực hành **waves** (DB → backend → frontend) và **hooks** (job migrate DB).

---

## Tuần 3 – Secrets, Rollouts, Image Updater, Notifications (6–10 giờ)

**Kiến thức:**

* Secrets: **Sealed-Secrets** (bitnami) hoặc **SOPS + age/KMS** (khuyến nghị với GitOps).
* Progressive Delivery: **Argo Rollouts** (canary/blue-green/traffic shaping).
* **Argo CD Image Updater**: tự động bump image tag theo semver/commit-sha.
* **Argo CD Notifications**: cảnh báo qua Slack/Telegram/Email.

**Lab 4 – Secret Management**

* Chọn **SOPS+age** hoặc **Sealed-Secrets**; commit secret dưới dạng mã hoá; Argo CD sync → secret sống trong cluster.

**Lab 5 – Canary bằng Argo Rollouts**

* Chuyển Deployment → Rollout + Service canary/stable, set bước canary 20/40/60 → promotion.

**Lab 6 – Image Updater + Notifications**

* Gắn annotation vào Application/Helm values để Image Updater theo dõi repo image và bump tag.
* Cấu hình Notifications gửi sự kiện “sync failed/succeeded”.

**Bài tập cuối tuần 3:**

* Bảo mật secret trong Git, rollout canary thành công, image auto-bump khi push image mới.

---

## Tuần 4 – Prod-ready: HA, RBAC/SSO, Policies, Observability, DR (6–10 giờ)

**Kiến thức:**

* **HA install**: replica cho `argocd-server`, `repo-server`, `application-controller`, Redis HA; cấu hình Ingress/ALB + TLS.
* **Multi-tenancy** & **RBAC** (AppProject boundaries, source/destination whitelist), **SSO** (GitHub/GitLab/OIDC).
* **Policy** (OPA/Gatekeeper, Kyverno) – guardrails cho manifests.
* **Observability**: Prometheus metrics, Grafana dashboards, audit logs.
* **Backup/DR**: dùng **Velero** backup CRDs/namespaces; khôi phục khi disaster.
* **Hardening**: NetworkPolicy, restrict cluster-wide, repo-server plugin allowlists.

**Lab 7 – SSO & RBAC**

* Bật SSO OIDC với GitHub/GitLab; map nhóm → role; giới hạn nhóm A chỉ deploy namespace A.

**Lab 8 – Backup/Restore**

* Dùng Velero backup `argocd` + namespaces app; simulate restore sang cluster khác.

**Bài tập cuối tuần 4 (tổng kết):**

* Setup Argo CD HA (ít nhất replicas >1, Ingress/TLS), SSO + RBAC theo team.
* Có monitor cơ bản, dashboard, và kịch bản backup/restore.

---

## Capstone (2–6 giờ): GitOps chuẩn Prod

**Yêu cầu:**

* Monorepo hoặc multi-repo: `infra/`, `apps/`, `environments/` (dev/stg/prod).
* App of Apps bootstrap toàn bộ nền tảng (nginx-ingress, cert-manager, external-dns…) + các app domain.
* ApplicationSet phát hành đến 2 cluster (VN/US).
* Secret bằng SOPS (age key trong KMS), Rollouts canary + metric check, Image Updater, Notifications.
* SSO + RBAC phân quyền team, Velero backup định kỳ, dashboard Prometheus/Grafana.

**Tiêu chí chấm:**

* `git log` cho thấy change → PR review → merge → Argo CD tự deploy đúng thứ tự (waves).
* Canary chạy và rollback được khi metric xấu.
* Secret không lộ plain-text trong repo.
* Backup/restore khôi phục được apps và Argo CD state.

---

## Checklist kỹ năng (tự đánh giá)

* [ ] Tạo/Quản trị **Application**, **AppProject**, hiểu Sync/Health/Diff/Ignore.
* [ ] Dùng **Helm/Kustomize** trong GitOps.
* [ ] **App of Apps**, **ApplicationSet** (multi-env/cluster).
* [ ] **Sync waves/hooks**, pre/post sync jobs.
* [ ] Secret với **SOPS/Sealed-Secrets**.
* [ ] **Argo Rollouts** canary/blue-green.
* [ ] **Image Updater**, **Notifications**.
* [ ] **RBAC/SSO**, **multi-tenancy**.
* [ ] **HA**, **Ingress/TLS**, **Observability**, **Backup/DR**.

---

## Mẫu lệnh nhanh hay dùng (tham khảo)

**argocd CLI cơ bản**

* *Run on: your laptop*

```bash
argocd login <argo-server> --username admin --password <pwd> --grpc-web
argocd app list
argocd app create myapp --repo https://github.com/... --path app --dest-server https://kubernetes.default.svc --dest-namespace default
argocd app sync myapp
argocd app set myapp --sync-policy automated --self-heal --auto-prune
argocd app diff myapp
argocd app history myapp
argocd proj list
```

**Ignore Differences (ví dụ)**

* *Run on: your laptop (apply manifest to cluster)*

```yaml
spec:
  ignoreDifferences:
  - group: "*"
    kind: Deployment
    jsonPointers:
    - /spec/template/spec/containers/0/imagePullPolicy
```

---

## Gợi ý thực chiến & best-practices

* Tách **repo hạ tầng** (bootstrap, CRDs, operators) và **repo ứng dụng** (business services).
* Mỗi env một **branch** hoặc **folder**; dùng **ApplicationSet** để phát hành theo pattern thống nhất.
* Không lưu secret plain-text; chọn **SOPS + age + KMS** nếu dùng cloud.
* Bật **Auto-Sync + SelfHeal + Prune** cho env non-prod; prod có thể dùng manual sync + Rollouts.
* Thiết lập **policy** (Kyverno/Gatekeeper) để chặn cấu hình rủi ro (runAsRoot, hostPath…).
* Quan sát **reconcile loop** và **App health** qua metrics; alert khi sync lỗi.

---
















