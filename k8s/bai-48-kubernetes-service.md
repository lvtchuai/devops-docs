# Bài 48: Kubernetes Service - Lời giải cho bài toán Service Discovery

## Giới thiệu

Bài này chính là mảnh ghép hoàn hảo để trả lời cho câu hỏi đang bỏ ngỏ ở [Bài 47](./bai-47-deployment-replicaset-pod.md): **Làm sao để Frontend tìm được Backend khi địa chỉ IP của Backend cứ thay đổi liên tục do bị Pod chết/tạo mới?**

Bài 48 tiếp tục khẳng định lý do tại sao Kubernetes lại thống trị thế giới Container. Dưới đây là lời giải thích trực quan về bài toán **Khám phá dịch vụ (Service Discovery)** và cách tài nguyên `Service` giải quyết nó.

---

## 1. Nỗi đau đứt gãy kết nối của Docker thuần

Hãy tưởng tượng hệ thống của bạn có:

- **Frontend (FE)** đang chạy ở IP `10.1.2.3`
- **Backend (BE)** đang chạy ở IP `10.1.4.6`

FE được cấu hình (ví dụ qua biến môi trường) để luôn gọi API tới IP `10.1.4.6`.

Nếu BE bị sập và khởi động lại, nó sẽ nhận một IP hoàn toàn mới (ví dụ: `10.1.5.4`). Lúc này, FE vẫn ngây ngô gọi vào IP cũ, dẫn đến **toàn bộ hệ thống tê liệt**. Bạn phải vào FE đổi cấu hình và khởi động lại FE — điều này là thảm họa nếu làm thủ công.

---

## 2. Giải pháp: "Trạm trung chuyển" mang tên Service

Kubernetes giải quyết triệt để vấn đề này bằng cách **không cho FE và BE nói chuyện trực tiếp với nhau thông qua IP của Pod**.

K8s tạo ra một tài nguyên gọi là `Service`. Service đóng vai trò như một **Proxy (hoặc Load Balancer) nội bộ**, đứng chắn ngang giữa FE và BE.

- FE chỉ cần gọi đến **TÊN** của Service (ví dụ: `http://backend-service`).
- K8s có hệ thống **DNS nội bộ** sẽ tự động phân giải tên này thành **IP tĩnh** của Service.
- Service sau đó sẽ đứng ra nhận request và chuyển tiếp (route) đến đúng Pod BE đang sống.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: amazon-backend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

---

## 3. Bí mật hoạt động: Labels và Selectors

**Câu hỏi:** Làm sao Service biết BE vừa đổi sang IP mới để mà chuyển traffic đến?

**Trả lời:** Service hoàn toàn **không quan tâm đến IP của Pod**. Nó hoạt động dựa trên cơ chế **Labels (Nhãn)**:

1. Khi tạo Deployment cho BE, bạn dán cho Pod một cái nhãn (như xăm một hình xăm), ví dụ: `app: amazon-backend`.
2. Service được cấu hình một `Selector` (Bộ lọc) để luôn tìm kiếm những Pod có đúng nhãn `app: amazon-backend`.
3. Dù Pod cũ chết đi, IP thay đổi hàng trăm lần, thì ReplicaSet luôn sinh ra Pod mới mang đúng cái nhãn `app: amazon-backend` đó.
4. Service tự động quét thấy nhãn này và cập nhật danh sách IP đích ngay lập tức dưới nền.

### Ví dụ Deployment với Label tương ứng

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: amazon-backend
  template:
    metadata:
      labels:
        app: amazon-backend   # <-- Nhãn này khớp với Selector của Service
    spec:
      containers:
        - name: backend
          image: my-backend:latest
          ports:
            - containerPort: 8080
```

> 🔑 **Chốt lại:** `Selector` của Service và `labels` của Pod (thông qua Deployment) chính là sợi dây liên kết ngầm giúp Service luôn tìm đúng Pod, bất kể IP thay đổi bao nhiêu lần.

---

## 4. Giao tiếp nội bộ và Mở cửa ra bên ngoài

Service có hai vai trò chính:

| Vai trò | Mô tả |
|---|---|
| **Giao tiếp nội bộ** | Giúp các microservices (FE gọi BE, BE gọi DB) tìm thấy nhau, không bao giờ bị đứt kết nối dù Pod có chết/tạo mới liên tục. |
| **Mở cửa ra ngoài** (External Access) | Service cũng là cánh cổng để người dùng từ Internet truy cập vào ứng dụng của bạn (thường thông qua cấu hình Load Balancer). |

---

## Tổng kết

- Docker thuần không giải quyết được vấn đề IP của container thay đổi liên tục khi restart → gây đứt kết nối giữa các service.
- Kubernetes `Service` đóng vai trò như một **trạm trung chuyển (Proxy/Load Balancer)** có tên và IP tĩnh, đứng giữa các thành phần.
- Cơ chế **Labels & Selectors** là "bí mật" giúp Service luôn tìm đúng Pod đích, bất kể Pod có bị thay thế bao nhiêu lần.
- Service phục vụ cả giao tiếp nội bộ giữa các microservices lẫn việc mở cổng truy cập từ bên ngoài Internet.
