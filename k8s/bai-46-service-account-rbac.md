# Bài 46: Kube-RBAC và Service Account trong Kubernetes

## Giới thiệu

Bài này đề cập đến một khái niệm bảo mật cực kỳ quan trọng và là câu hỏi kinh điển trong các buổi phỏng vấn DevOps/SRE: **Kube-RBAC (Role-Based Access Control)**, mà cụ thể ở đây là **Service Account**.

Nhiều người mới học K8s thường bỏ qua khái niệm này vì khi làm lab, họ cứ gõ `kubectl apply` là Pod chạy mà chẳng cần cấu hình bảo mật gì thêm. Dưới đây là "sự thật" đằng sau hiện tượng này và 3 điểm cốt lõi bạn cần nắm vững.

---

## 1. Phân biệt User Account và Service Account

| Loại tài khoản | Dành cho | Cơ chế xác thực |
|---|---|---|
| **User Account** (Tài khoản người dùng) | CON NGƯỜI (Dev, DevOps, Admin) | Sử dụng file `kubeconfig` (như đã làm ở [Bài 44](./bai-44-ket-noi-kubectl-eks.md)) để xác thực và gõ lệnh điều khiển cụm K8s từ bên ngoài. |
| **Service Account** (Tài khoản dịch vụ) | MÁY MÓC / ỨNG DỤNG (các Pods/Containers) | Khi một ứng dụng đang chạy bên trong K8s muốn giao tiếp hoặc truy xuất dữ liệu từ hệ thống K8s (API Server), nó phải dùng Service Account làm "thẻ căn cước". |

---

## 2. Sự thật về "Default Service Account"

**Câu hỏi:** Tại sao khi làm lab không tạo Service Account mà Pod vẫn chạy?

**Trả lời:**

- Bản thân Kubernetes có một cơ chế tự động: ngay khi bạn tạo một **Namespace mới**, K8s lập tức tạo sẵn một Service Account có tên là `default` ở bên trong đó.
- Khi bạn deploy một Pod mà quên không chỉ định Service Account, K8s sẽ tự động nhét cái thẻ `default` này vào Pod.
- Chiếc thẻ `default` này có **quyền hạn cực thấp**, chỉ vừa đủ để Pod có thể tồn tại và chạy các tác vụ cơ bản của riêng nó.

---

## 3. Khi nào cần tạo Service Account riêng? (Cơ chế RBAC)

Với các microservices bình thường (như web bán hàng, kết nối DB), chúng không cần quan tâm môi trường K8s xung quanh là gì, nên dùng `default` SA là đủ.

Nhưng giả sử bạn viết một ứng dụng CI/CD (như Jenkins, ArgoCD, hay Prometheus) chạy bên trong K8s. Ứng dụng này cần:

- Quyền đọc các `ConfigMap`
- Quyền xem trạng thái của các Pod khác
- Quyền tự động xóa Pod bị lỗi

Lúc này thẻ `default` sẽ bị K8s từ chối (**Access Denied**). Bạn phải cấp quyền theo 3 bước:

### Bước 1: Tạo thực thể

Tạo một Service Account mới. Ví dụ:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus-sa
  namespace: monitoring
```

### Bước 2: Tạo quyền hạn (Role)

Tạo một `Role` (hoặc `ClusterRole`) — đây là tờ giấy ghi rõ các đặc quyền: *"Được phép đọc ConfigMap, được phép liệt kê danh sách Pod"*.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: prometheus-role
  namespace: monitoring
rules:
  - apiGroups: [""]
    resources: ["pods", "configmaps"]
    verbs: ["get", "list", "watch"]
```

### Bước 3: Gắn quyền (RoleBinding)

Dùng `RoleBinding` (hoặc `ClusterRoleBinding`) để ghim tờ giấy đặc quyền (Role) vào chiếc thẻ căn cước (Service Account).

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: prometheus-rolebinding
  namespace: monitoring
subjects:
  - kind: ServiceAccount
    name: prometheus-sa
    namespace: monitoring
roleRef:
  kind: Role
  name: prometheus-role
  apiGroup: rbac.authorization.k8s.io
```

---

## 💡 Mẹo liên hệ (Mental Model)

Nếu đối chiếu sang AWS mà bạn vừa học ở Terraform:

| Kubernetes | AWS (tương đương) |
|---|---|
| Service Account | IAM Role |
| Role (của K8s) | IAM Policy |
| RoleBinding | Động tác Attach Policy vào Role |

---

## Tổng kết

- **User Account** dùng cho con người, xác thực qua `kubeconfig`.
- **Service Account** dùng cho ứng dụng/Pod, là "thẻ căn cước" khi giao tiếp với API Server.
- Mỗi Namespace tự động có một Service Account `default` với quyền hạn tối thiểu.
- Khi ứng dụng cần quyền cao hơn (đọc ConfigMap, list Pod...), phải tạo riêng: **Service Account → Role → RoleBinding**.
- Đây là kiến thức nền tảng của **RBAC (Role-Based Access Control)** — chủ đề bảo mật cốt lõi và thường gặp trong phỏng vấn DevOps/SRE.
