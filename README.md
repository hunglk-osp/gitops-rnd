# gitops-rnd

GitOps repository triển khai ứng dụng trên Kubernetes sử dụng ArgoCD với pattern **App of Apps**.

---

## Cấu trúc repo

```
gitops-rnd/
├── bootstrap/                          # Root Applications - ArgoCD tự bootstrap
│   ├── argocd-bootstrap.yaml           # Root App #1: quản lý thư mục bootstrap/
│   ├── argocd-apps.yaml                # Root App #2: quản lý thư mục apps/
│   ├── argocd-namespace-config.yaml    # Cho phép ArgoCD watch namespace argocd-applications
│   ├── argocd-project-config.yaml      # Cấu hình AppProject default cho multi-namespace
│   └── argocd-project-config.yaml
└── apps/
    ├── cert-manager.yaml               # App: cert-manager (Helm)
    ├── cloudnative-pg-operator.yaml    # App: CloudNativePG operator (Helm)
    ├── keycloak-operator.yaml          # App: Keycloak operator (Kustomize)
    ├── postgresql.yaml                 # App: PostgreSQL cluster (Kustomize)
    ├── keycloak.yaml                   # App: Keycloak (Kustomize)
    ├── keycloak-operator/              # Kustomize resources: Keycloak operator
    │   └── kustomization.yaml
    ├── postgresql/                     # Kustomize resources: PostgreSQL
    │   ├── kustomization.yaml
    │   ├── keycloak-db-credentials.yaml
    │   └── postgresql-cluster.yaml
    └── keycloak/                       # Kustomize resources: Keycloak
        ├── kustomization.yaml
        ├── cluster-issuer-selfsigned.yaml
        ├── keycloak-cluster.yaml
        └── keycloak-ingress.yaml
```

---

## App of Apps Pattern

```
argocd-bootstrap  (namespace: argocd)
    └── argocd-apps  (namespace: argocd, path: bootstrap/)
            ├── cert-manager           (sync-wave: -5) → namespace: cert-manager
            ├── cloudnative-pg-operator(sync-wave: -5) → namespace: cnpg-system
            ├── keycloak-operator      (sync-wave: -4) → namespace: argocd-applications
            ├── postgresql             (sync-wave: -2) → namespace: argocd-applications
            └── keycloak               (sync-wave: -1) → namespace: argocd-applications
```

Thứ tự deploy theo sync-wave đảm bảo:
1. Operators lên trước (CRD sẵn sàng)
2. PostgreSQL lên sau operators (CNPG CRD đã có)
3. Keycloak lên cuối (DB đã sẵn sàng)

---

## Yêu cầu cài đặt trước

Các thành phần cần có trên cluster trước khi bootstrap:

- ArgoCD (namespace `argocd`)
- Traefik Ingress Controller
- cert-manager (hoặc để ArgoCD tự deploy qua `cert-manager.yaml`)

---

## Bootstrap lần đầu

### 1. Tạo namespace

```bash
kubectl create namespace argocd-applications
```

### 2. Apply Root App

```bash
kubectl apply -f bootstrap/argocd-bootstrap.yaml
```

ArgoCD sẽ tự động:
- Sync thư mục `bootstrap/` → tạo `argocd-apps` và các config
- `argocd-apps` sync thư mục `apps/` → tạo tất cả child Applications
- Mỗi child Application tự sync resources của nó theo sync-wave

### 3. Kiểm tra trạng thái

```bash
# Xem tất cả Applications
kubectl get applications -A

# Xem pods trong namespace chính
kubectl get pod -n argocd-applications
```

---

## Deploy PostgreSQL + Import Database

Nếu cần import database từ SQL dump (thực hiện **sau khi PostgreSQL Running**, **trước khi Keycloak deploy**):

### 1. Suspend Keycloak để tránh connect DB sớm

```bash
kubectl patch application keycloak -n argocd-applications \
  --type merge -p '{"spec":{"syncPolicy":{"automated":null}}}'
kubectl delete keycloak keycloak -n argocd-applications
```

### 2. Lấy password DB

```bash
PG_PASS=$(kubectl get secret keycloak-db-credentials -n argocd-applications \
  -o jsonpath='{.data.password}' | base64 -d)
PG_SUPER_PASS=$(kubectl get secret keycloak-db-superuser -n argocd-applications \
  -o jsonpath='{.data.password}' | base64 -d)
```

### 3. Drop và recreate database

```bash
# Drop
kubectl exec -n argocd-applications keycloak-db-1 -- bash -c \
  "PGPASSWORD='$PG_PASS' psql -h localhost -U keycloak -d postgres -c 'DROP DATABASE IF EXISTS keycloak;'"

# Recreate
kubectl exec -n argocd-applications keycloak-db-1 -- bash -c \
  "PGPASSWORD='$PG_SUPER_PASS' psql -h localhost -U postgres -d postgres -c 'CREATE DATABASE keycloak WITH OWNER keycloak;'"
```

### 4. Import SQL dump

```bash
cat /path/to/keycloak_backup.sql | kubectl exec -i -n argocd-applications keycloak-db-1 -- \
  bash -c "PGPASSWORD='$PG_PASS' psql -h localhost -U keycloak -d keycloak"
```

### 5. Re-enable Keycloak

```bash
kubectl patch application keycloak -n argocd-applications \
  --type merge -p '{"spec":{"syncPolicy":{"automated":{"prune":true,"selfHeal":true}}}}'
kubectl annotate application keycloak -n argocd-applications \
  argocd.argoproj.io/refresh=normal --overwrite
```

---

## Thêm ứng dụng mới

1. Tạo Application manifest `apps/<ten-app>.yaml`
2. Tạo thư mục `apps/<ten-app>/` với `kustomization.yaml` và resources
3. Push lên `main` → `argocd-apps` tự động phát hiện và deploy

---

## /etc/hosts (laptop)

Thêm vào file hosts trên máy truy cập:

```
192.168.1.18  keycloak.local
192.168.1.18  argocd.rnd
192.168.1.18  rancher.rnd
```
