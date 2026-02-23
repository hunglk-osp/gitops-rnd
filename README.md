# gitops-rnd

GitOps repository triển khai ứng dụng trên Kubernetes sử dụng ArgoCD với pattern **App of Apps**.

---

## App of Apps Pattern

Thay vì apply từng Application thủ công lên ArgoCD, pattern này dùng **một Root Application duy nhất** để quản lý tất cả các Application còn lại. Mọi thay đổi đều thông qua Git.

```
Git repo (source of truth)
    │
    ▼
argocd-bootstrap          ← apply thủ công 1 lần duy nhất
    │   watches: bootstrap/
    │
    └── argocd-apps               ← Root App quản lý tất cả child apps
            │   watches: apps/*.yaml
            │
            ├── cert-manager            (sync-wave: -5)
            ├── cloudnative-pg-operator (sync-wave: -5)
            ├── keycloak-operator       (sync-wave: -4)
            ├── postgresql              (sync-wave: -2)
            └── keycloak                (sync-wave: -1)
```

### Nguyên lý namespace

**Application CRD** và **workload** sống ở 2 namespace khác nhau:

| Loại resource | Namespace |
|---|---|
| Application CRD (argocd-bootstrap, argocd-apps, keycloak, postgresql...) | `argocd` |
| Workloads (pods, services, ingress...) | `rnd-keycloak` |

ArgoCD server mặc định chỉ **watch và manage** Application CRD trong namespace `argocd` — namespace mà nó được cài vào. Vì vậy tất cả Application manifest phải có `metadata.namespace: argocd`.

`spec.destination.namespace` trong Application là nơi **workloads được deploy**, hoàn toàn độc lập với namespace của Application CRD:

```yaml
metadata:
  namespace: argocd          # ArgoCD server watch ở đây
spec:
  destination:
    namespace: rnd-keycloak  # Workloads (pods, services...) chạy ở đây
```

### Cách hoạt động

**1. Bootstrap một lần duy nhất**

```bash
kubectl apply -f bootstrap/argocd-bootstrap.yaml
```

Chỉ cần chạy lệnh này một lần. Từ đó ArgoCD tự quản lý mọi thứ.

**2. argocd-bootstrap** watch thư mục `bootstrap/` và sync các resource trong đó, bao gồm `argocd-apps.yaml`.

**3. argocd-apps** watch thư mục `apps/` với `recurse: false` — chỉ đọc các file Application manifest ở root `apps/*.yaml`, **không** đọc vào subfolder.

**4. Mỗi child Application** trỏ `path` vào subfolder tương ứng (`apps/keycloak/`, `apps/postgresql/`...) và dùng **Kustomize** để render resources.

**5. Sync-wave** đảm bảo thứ tự deploy — operators phải lên trước để CRD sẵn sàng trước khi deploy workload phụ thuộc vào chúng.

### Thêm ứng dụng mới

Chỉ cần tạo 2 thứ và push lên `main`:

```
apps/ten-app.yaml          ← Application manifest (namespace: argocd)
apps/ten-app/
    kustomization.yaml     ← Kustomize entry point
    ...resources...
```

`argocd-apps` tự động phát hiện file mới và tạo child Application. Không cần động vào ArgoCD UI hay chạy lệnh kubectl nào thêm.

---

## Cấu trúc repo

```
gitops-rnd/
├── bootstrap/
│   ├── argocd-bootstrap.yaml           # Root App #1 - apply thủ công 1 lần
│   └── argocd-apps.yaml                # Root App #2 - quản lý child apps
└── apps/
    ├── cert-manager.yaml
    ├── cloudnative-pg-operator.yaml
    ├── keycloak-operator.yaml
    ├── postgresql.yaml
    ├── keycloak.yaml
    ├── keycloak-operator/
    ├── postgresql/
    └── keycloak/
```
