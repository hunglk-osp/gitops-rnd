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
    ├── argocd-namespace-config   ← cho phép ArgoCD watch namespace argocd-applications
    ├── argocd-project-config     ← cấp quyền AppProject cho namespace argocd-applications
    └── argocd-apps               ← Root App quản lý tất cả child apps
            │   watches: apps/*.yaml
            │
            ├── cert-manager            (sync-wave: -5)
            ├── cloudnative-pg-operator (sync-wave: -5)
            ├── keycloak-operator       (sync-wave: -4)
            ├── postgresql              (sync-wave: -2)
            └── keycloak                (sync-wave: -1)
```

### Cách hoạt động

**1. Bootstrap một lần duy nhất**

```bash
kubectl apply -f bootstrap/argocd-bootstrap.yaml
```

Chỉ cần chạy lệnh này một lần. Từ đó ArgoCD tự quản lý mọi thứ.

**2. argocd-bootstrap** watch thư mục `bootstrap/` và sync các resource trong đó, bao gồm `argocd-apps.yaml`.

**3. argocd-apps** watch thư mục `apps/` với `recurse: false` - chỉ đọc các file Application manifest ở root `apps/*.yaml`, **không** đọc vào subfolder.

**4. Mỗi child Application** trỏ `path` vào subfolder tương ứng (`apps/keycloak/`, `apps/postgresql/`...) và dùng **Kustomize** để render resources.

**5. Sync-wave** đảm bảo thứ tự deploy - operators phải lên trước để CRD sẵn sàng trước khi deploy workload phụ thuộc vào chúng.

### Tại sao namespace `argocd-applications`?

ArgoCD mặc định chỉ quản lý Application CRD trong namespace `argocd`. Để child Applications có thể nằm ở namespace khác (`argocd-applications`), cần 2 config:

- `argocd-namespace-config.yaml` - thêm `application.namespaces: argocd-applications` vào ArgoCD config
- `argocd-project-config.yaml` - thêm `sourceNamespaces: [argocd-applications]` vào AppProject `default`

### Thêm ứng dụng mới

Chỉ cần tạo 2 thứ và push lên `main`:

```
apps/ten-app.yaml          ← Application manifest
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
│   ├── argocd-bootstrap.yaml           # Root App #1
│   ├── argocd-apps.yaml                # Root App #2
│   ├── argocd-namespace-config.yaml
│   └── argocd-project-config.yaml
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
