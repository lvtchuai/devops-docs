# Bài 49: Giải phẫu file cấu hình Deployment YAML

## Giới thiệu

Bài này có thể coi là một "lớp học giải phẫu" chuyên sâu về cách viết file cấu hình `Deployment` YAML trong Kubernetes.

Nhiều người mới học K8s thường bị "ngợp" khi nhìn thấy một file YAML dài 60-100 dòng. Thực chất, file YAML này có một cấu trúc (Skeleton) cực kỳ logic và **lặp đi lặp lại**. Bạn không cần (và không bao giờ) phải học thuộc lòng từng dòng.

Dưới đây là công thức chuẩn để "giải phẫu" bất kỳ file Deployment nào.

---

## 1. Bốn trường bắt buộc (The Big Four)

Bất kể bạn tạo tài nguyên gì trên K8s (Pod, Deployment, Service, ConfigMap...), file YAML luôn phải mở đầu bằng 4 trường này:

```yaml
apiVersion: apps/v1  # Phiên bản API của K8s
kind: Deployment      # Loại tài nguyên muốn tạo (ở đây là Deployment)
metadata:             # Dữ liệu định danh cho cái Deployment này
  name: currency-service-deployment
  labels:
    app: currency
spec:                 # Đặc tả chi tiết (Phần ruột)
  ...
```

| Trường | Ý nghĩa |
|---|---|
| `apiVersion` | Phiên bản API của K8s dùng cho tài nguyên này |
| `kind` | Loại tài nguyên muốn tạo (Deployment, Pod, Service...) |
| `metadata` | Dữ liệu định danh: tên, nhãn (labels)... |
| `spec` | Đặc tả chi tiết — "phần ruột" thực sự cấu hình hành vi |

---

## 2. Giải phẫu phần `spec` (Trái tim của Deployment)

Đây là nơi bạn định nghĩa cấu hình hoạt động. `spec` được chia làm **2 tầng** rõ rệt.

### Tầng 1: Cấu hình của ReplicaSet (Quản đốc)

| Trường | Ý nghĩa |
|---|---|
| `replicas` | Bạn muốn chạy bao nhiêu bản sao? (Ví dụ: `replicas: 3`). K8s sẽ tự động giữ đúng con số này (Auto-healing & Auto-scaling). |
| `selector` | "Bộ lọc" để Quản đốc (ReplicaSet) biết phải quản lý những Pod nào. Nó phải **khớp chính xác** với nhãn (labels) của Pod ở Tầng 2. |

### Tầng 2: Cấu hình của Pod (Công nhân) — nằm trong khối `template`

Tất cả mọi thứ nằm dưới khối `template` thực chất chính là **cấu hình của một Pod**. K8s sẽ dùng cái "khuôn" này để đúc ra số lượng Pod tương ứng với trường `replicas`.

Trong `template`, cấu trúc lại lặp lại y hệt: `metadata` + `spec`.

#### `template.metadata` — Cực kỳ quan trọng cho Service Discovery

Nơi bạn dán "Nhãn" (Labels) cho Pod. Service sẽ dùng nhãn này để tìm đến Pod (xem lại [Bài 48](./bai-48-kubernetes-service.md)).

#### `template.spec` — Đặc tả của Pod

Đây là nơi rất quen thuộc nếu bạn đã từng viết `docker-compose.yml`:

| Trường | Ý nghĩa |
|---|---|
| `serviceAccountName` | Khai báo "thẻ căn cước" cho Pod (xem [Bài 46](./bai-46-service-account-rbac.md)). Nếu bỏ qua, nó sẽ dùng thẻ `default`. |
| `containers` | Danh sách các container chạy trong Pod. |
| `containers[].name` | Tên container. |
| `containers[].image` | Đường dẫn lấy Docker image từ Docker Hub hay AWS ECR. |
| `containers[].ports` | Cổng mở ra để giao tiếp. |
| `containers[].env` | Biến môi trường. *(Đừng sợ khi thấy chục dòng biến môi trường — chỗ này thường là copy từ Dev đưa sang!)* |
| `containers[].resources` | Giới hạn RAM/CPU cho container. |
| `containers[].volumeMounts` | Cấu hình ổ đĩa. |

---

## 3. Ví dụ file Deployment hoàn chỉnh

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: currency-service-deployment
  labels:
    app: currency
spec:
  replicas: 3
  selector:
    matchLabels:
      app: currency
  template:
    metadata:
      labels:
        app: currency
    spec:
      serviceAccountName: default
      containers:
        - name: currency-service
          image: my-registry/currency-service:1.0.0
          ports:
            - containerPort: 8080
          env:
            - name: DB_HOST
              value: "currency-db"
            - name: DB_PORT
              value: "5432"
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
```

---

## 💡 Bài học thực tiễn

### 1. Đừng bao giờ cố gõ tay từ đầu

Cách làm thực tế của mọi DevOps engineer là:

```
Mở trang chủ Kubernetes Docs
        ↓
Copy đoạn YAML mẫu của Deployment
        ↓
Paste vào file của mình
        ↓
Sửa lại tên image, cổng, và biến môi trường
```

### 2. "Siêu YAML" (Super YAML)

Thay vì quản lý 40 file YAML cho 20 microservices (mỗi cái 1 file Deployment, 1 file Service), nhiều công ty dùng cú pháp `---` để gộp tất cả chúng vào **một file YAML khổng lồ**.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: currency-service-deployment
spec:
  ...
---
apiVersion: v1
kind: Service
metadata:
  name: currency-service
spec:
  ...
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cart-service-deployment
spec:
  ...
```

Cách này giúp bạn triển khai cả dự án chỉ bằng **1 câu lệnh duy nhất**:

```bash
kubectl apply -f complete_deploy.yaml
```

---

## Tổng kết

- Mọi file YAML trên K8s luôn bắt đầu với 4 trường: `apiVersion`, `kind`, `metadata`, `spec`.
- `spec` của Deployment chia làm 2 tầng: cấu hình **ReplicaSet** (`replicas`, `selector`) và cấu hình **Pod** (`template`).
- Khối `template` chính là "khuôn đúc" Pod, có cấu trúc `metadata` (labels) + `spec` (containers, env, resources...).
- Thực hành tốt nhất: copy YAML mẫu từ Kubernetes Docs rồi chỉnh sửa, thay vì viết tay từ đầu.
- Dùng dấu `---` để gộp nhiều tài nguyên (Deployment, Service...) vào 1 file YAML duy nhất, giúp deploy toàn bộ hệ thống chỉ với 1 lệnh `kubectl apply`.
