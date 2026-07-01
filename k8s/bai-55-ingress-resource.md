# Bài 55: Ingress Resource - Bộ định tuyến thông minh cho Traffic đi vào

## Giới thiệu

Bài này chính là câu trả lời trực tiếp cho việc: **Làm thế nào để khắc phục sự thô sơ của `LoadBalancer` Service Type** mà chúng ta vừa phân tích ở [Bài 54](./bai-54-loadbalancer-vs-ingress.md)!

Bài học đi sâu vào khái niệm tài nguyên **Ingress** và cách nó quản lý lưu lượng đi vào (Incoming traffic) một cách thông minh.

Dưới đây là 3 điểm sáng thực tế cực hay từ bài học này.

---

## 1. Rủi ro bảo mật (Red Flag) khi dùng LoadBalancer thông thường

### Ví dụ minh họa

Nếu bạn dùng lệnh `nslookup amazon.com` để lấy IP, sau đó gõ IP đó thẳng lên trình duyệt, hệ thống của Amazon sẽ **ngay lập tức chặn bạn lại**.

```bash
nslookup amazon.com
# Lấy được IP, ví dụ: 176.32.103.205
# Gõ thẳng http://176.32.103.205 lên trình duyệt → bị chặn
```

Việc cho phép người dùng truy cập trực tiếp bằng IP gốc của Load Balancer là một **lỗi bảo mật nghiêm trọng (Red flag)** trong môi trường doanh nghiệp.

### So sánh cách xử lý

| Phương án | Có chặn được truy cập trực tiếp bằng IP không? |
|---|---|
| **LoadBalancer Service** (cũ) | ❌ Không thể chặn. Khách hàng gõ URL hay IP đều vào được ứng dụng. |
| **Ingress** | ✅ Khắc phục triệt để bằng **Host-based routing** (Định tuyến theo tên miền). |

### Cơ chế Host-based routing

Bạn khai báo rõ trong file YAML:

```yaml
host: amazon.com
```

Nếu request gửi tới **không mang đúng tên miền này**, Ingress sẽ chặn đứng ngay lập tức!

---

## 2. Cấu trúc YAML của một Ingress Resource

Ingress đóng vai trò như một bộ **"luật lệ giao thông" (Routing rules)**. Nó hoạt động ở **Tầng 7 (HTTP/HTTPS)** của mô hình OSI — khác với Service (`ClusterIP`, `NodePort`, `LoadBalancer`) vốn hoạt động chủ yếu ở Tầng 4 (TCP/UDP).

Cấu trúc của nó rất trực quan:

- **Khai báo Domain (Host):** Ai được phép vào? (Ví dụ: `foo.bar.com`)
- **Khai báo Path (Đường dẫn):** Ví dụ đường dẫn `/` thì chuyển traffic tới Service `frontend-proxy`, cổng `8080`.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ecommerce-ingress
  annotations:
    kubernetes.io/ingress.class: "alb"
spec:
  rules:
    - host: foo.bar.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-proxy
                port:
                  number: 8080
```

### Chiến thuật cho dự án

Trong số 20 microservices của dự án e-commerce:

- ✅ **CHỈ TẠO 1 INGRESS DUY NHẤT** trỏ vào `frontend-proxy`.
- ✅ **19 microservices còn lại** hoàn toàn không cần Ingress và vẫn giữ nguyên kiểu `ClusterIP` an toàn bên trong nội bộ.

```
                 ┌─────────────┐
Internet ──────▶ │   Ingress    │
                 │ host: shop.  │
                 │ com  path: / │
                 └──────┬──────┘
                        │
                        ▼
                 frontend-proxy (Service)
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
   cart-service   payment-service  shipping-service
   (ClusterIP)      (ClusterIP)      (ClusterIP)
```

---

## 3. Sự thật về Ingress: Nó chỉ là một "Bản thiết kế"

Đây là một khái niệm **dễ gây nhầm lẫn** nhất khi mới học Ingress:

> ⚠️ Bản thân tài nguyên Ingress (cái file YAML bạn viết) **không tự tạo ra Load Balancer**.

Nếu bạn apply file này:

```bash
kubectl apply -f ingress.yaml
```

Hệ thống K8s chỉ **ghi nhận cấu hình** chứ **không có dòng traffic nào được định tuyến cả**.

### Vậy làm sao để nó chạy được?

Để biến "bản thiết kế" này thành hệ thống vật lý chạy được, cụm EKS của bạn **bắt buộc** phải được cài đặt một "người thợ xây" mang tên **Ingress Controller**.

- Ingress Controller sẽ **đọc file Ingress YAML** của bạn.
- Sau đó nó đi **ra lệnh cho AWS** để khởi tạo cấu hình một bộ Load Balancer thật sự (ALB/ELB) theo đúng luật lệ bạn đã viết.

```
Ingress YAML (bản thiết kế)
        │
        ▼
Ingress Controller (người thợ xây, đọc bản thiết kế)
        │
        ▼
AWS ALB/ELB (Load Balancer vật lý thật sự được tạo ra)
```

> 🔑 Nói cách khác: **Ingress = luật lệ (config)**, còn **Ingress Controller = bộ máy thực thi luật lệ đó**. Thiếu Ingress Controller, file Ingress YAML chỉ là một tờ giấy vô nghĩa nằm im trong etcd.

---

## Tổng kết

- Cho phép truy cập trực tiếp bằng IP gốc của Load Balancer là một rủi ro bảo mật; **Ingress** giải quyết vấn đề này bằng **Host-based routing**.
- Ingress hoạt động ở Tầng 7 (HTTP/HTTPS), định tuyến traffic dựa trên `host` (tên miền) và `path` (đường dẫn).
- Trong kiến trúc nhiều microservices, chỉ cần **1 Ingress** trỏ vào service Frontend/Gateway; các service Backend khác vẫn giữ `ClusterIP`.
- **Ingress Resource chỉ là cấu hình (bản thiết kế)** — để nó thực sự hoạt động và tạo ra Load Balancer, cụm K8s bắt buộc phải cài **Ingress Controller**.
