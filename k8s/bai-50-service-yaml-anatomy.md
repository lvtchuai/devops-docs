# Bài 50: Giải phẫu file cấu hình Service YAML

## Giới thiệu

Bài này chính là "chìa khóa" để giải quyết toàn bộ lý thuyết về **Service Discovery** mà chúng ta đã nói ở [Bài 48](./bai-48-kubernetes-service.md). Bài học chỉ ra cách viết file `Service.yaml` và cách nó "bắt tay" chính xác với file `Deployment.yaml` (đã học ở [Bài 49](./bai-49-deployment-yaml-anatomy.md)).

Nếu như file Deployment dài dòng phức tạp thì file Service lại cực kỳ ngắn gọn. Tuy nhiên, nó chứa **2 điểm mấu chốt** dễ gây nhầm lẫn nhất mà bất kỳ ai mới học K8s cũng phải vượt qua.

---

## 1. Cấu trúc chuẩn của một Service

Giống như mọi tài nguyên K8s khác, nó cũng bắt đầu bằng 4 trường cơ bản:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: currency-service-name
  labels:
    app: currency
spec:
  type: ClusterIP        # Loại Service (ClusterIP, NodePort, LoadBalancer)
  ports:
    - port: 8080          # Cổng của Service
      targetPort: 8080    # Cổng của Container (Pod)
  selector:
    app.kubernetes.io/name: opentelemetry-demo-currencyservice   # Bộ lọc tìm Pod
```

---

## 2. Điểm mấu chốt #1: Sự khác biệt giữa `port` và `targetPort`

Đây là câu hỏi rất hay gặp khi phỏng vấn hoặc thi CKA:

| Trường | Ý nghĩa |
|---|---|
| `targetPort` (Cổng đích) | Cổng mà ứng dụng/container của bạn **đang thực sự chạy** bên trong Pod. Bắt buộc phải khớp với trường `containerPort` đã khai báo bên trong file Deployment. |
| `port` (Cổng dịch vụ) | Cổng **ảo** do Service tạo ra. Bất kỳ ứng dụng Frontend nào muốn gọi Backend này thì phải gọi qua cổng `port` này. K8s sẽ tự động chuyển tiếp (forward) traffic từ `port` vào `targetPort`. |

> 💡 **Ví dụ minh họa:** Bạn hoàn toàn có thể đặt `port: 80` và `targetPort: 8080`. Khi đó Frontend chỉ cần gọi `http://currency-service-name:80`, traffic sẽ tự động chui vào cổng 8080 của container.

```yaml
ports:
  - port: 80          # Frontend gọi vào cổng 80
    targetPort: 8080  # Traffic được forward vào cổng 8080 trong container
```

---

## 3. Điểm mấu chốt #2: Ma thuật mang tên `selector`

**Câu hỏi:** Làm sao file Service biết phải chuyển traffic cho Pod nào giữa hàng nghìn Pod trong Cluster?

**Trả lời:** Đó là nhờ trường `selector` (Bộ lọc).

1. Trong file `Deployment.yaml`, ở phần `template.metadata.labels`, bạn đã dán cho Pod một cái nhãn. Ví dụ:

```yaml
# Trong Deployment.yaml
template:
  metadata:
    labels:
      app.kubernetes.io/name: opentelemetry-demo-currencyservice
```

2. Trong file `Service.yaml`, bạn chỉ cần bê **y nguyên** cái nhãn đó thả vào trường `selector`:

```yaml
# Trong Service.yaml
spec:
  selector:
    app.kubernetes.io/name: opentelemetry-demo-currencyservice
```

3. **Kết quả:** Service giống như một "thợ săn", nó sẽ liên tục rà quét toàn bộ Cluster. Cứ Pod nào ngoi lên mà mang đúng cái nhãn đó, Service sẽ tự động thêm IP của Pod đó vào danh sách phân phối traffic. Pod chết thì nó tự gạch IP đi. **Bạn không cần làm gì cả!**

---

## 4. Cặp Deployment + Service hoàn chỉnh

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: currency-service-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app.kubernetes.io/name: opentelemetry-demo-currencyservice
  template:
    metadata:
      labels:
        app.kubernetes.io/name: opentelemetry-demo-currencyservice   # <-- Nhãn Pod
    spec:
      containers:
        - name: currency-service
          image: my-registry/currency-service:1.0.0
          ports:
            - containerPort: 8080   # <-- Khớp với targetPort bên dưới
---
apiVersion: v1
kind: Service
metadata:
  name: currency-service-name
spec:
  type: ClusterIP
  ports:
    - port: 8080
      targetPort: 8080              # <-- Khớp với containerPort ở trên
  selector:
    app.kubernetes.io/name: opentelemetry-demo-currencyservice   # <-- Khớp với labels ở trên
```

---

## Tổng kết

- File `Service.yaml` cũng tuân theo cấu trúc 4 trường chuẩn: `apiVersion`, `kind`, `metadata`, `spec`.
- `port` là cổng ảo mà client (Frontend) gọi vào; `targetPort` là cổng thật mà container đang lắng nghe — hai giá trị này **không nhất thiết phải giống nhau**.
- `targetPort` phải khớp với `containerPort` trong Deployment.
- `selector` trong Service phải khớp chính xác với `labels` trong `template.metadata` của Deployment — đây chính là sợi dây kết nối giúp Service luôn tìm đúng Pod, kể cả khi Pod liên tục bị thay thế.
