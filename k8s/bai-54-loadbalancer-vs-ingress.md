# Bài 54: So sánh LoadBalancer Service vs Ingress

## Giới thiệu

Bài này lật ngược lại toàn bộ vấn đề và chỉ ra rằng: cách chúng ta vừa làm ở [Bài 53](./bai-53-deploy-loadbalancer-internet.md) (dùng `type: LoadBalancer`) tuy chạy được, nhưng lại là một **"Bad Practice"** (Thực hành xấu) nếu áp dụng ở quy mô doanh nghiệp.

Đây cũng là một trong những câu hỏi phỏng vấn vị trí DevOps/SRE phổ biến nhất:

> "Tại sao lại dùng Ingress thay vì LoadBalancer Service Type?"

Kiến thức này mang tính lý thuyết hệ thống rất cao và nối tiếp trực tiếp mạch kiến thức của [Bài 48](./bai-48-kubernetes-service.md).

Trong Kubernetes, để mở cửa ứng dụng ra Internet, `type: LoadBalancer` là cách dễ nhất. Tuy nhiên, khi hệ thống mở rộng, nó bộc lộ **4 điểm yếu chí mạng** khiến các kỹ sư phải chuyển sang dùng **Ingress**.

---

## 1. Bốn "Tử huyệt" của LoadBalancer Service Type

### 1.1. Không có tính khai báo (Lack of Declarative Approach)

Kubernetes sinh ra để quản lý hạ tầng bằng Code (**IaC** - Infrastructure as Code, thông qua YAML). Tuy nhiên, file Service YAML chỉ cho phép bạn khai báo mỗi dòng `type: LoadBalancer`.

Nếu sếp yêu cầu bạn:
- Gắn chứng chỉ bảo mật HTTPS (TLS)
- Đổi thuật toán cân bằng tải

...bạn **không thể** viết nó vào file YAML. Bạn bắt buộc phải đăng nhập vào giao diện web của AWS để click chuột thủ công.

> ❌ Điều này phá vỡ hoàn toàn nguyên tắc IaC và gây khó khăn cho việc theo dõi lịch sử chỉnh sửa (Tracking).

### 1.2. Đốt tiền (Cost Ineffectiveness)

Quy tắc của Service Type này là tỷ lệ **1-1**: 1 Service sẽ gọi AWS tạo ra 1 Load Balancer vật lý riêng biệt.

Nếu ứng dụng e-commerce của bạn có **10 microservices** cần mở ra Internet (Frontend, Admin, Payment API...), AWS sẽ tạo ra **10 cái Load Balancers** và tính tiền cả 10.

> 💸 Hóa đơn cuối tháng sẽ cực kỳ khổng lồ.

### 1.3. Trói buộc công nghệ (Lack of Flexibility / Vendor Lock-in)

Bạn bị trói chặt vào công nghệ của nhà cung cấp Cloud:

- Đang chạy trên AWS → K8s sẽ chỉ tạo ra AWS ALB/ELB.
- Bạn **không thể** đổi sang dùng các bộ cân bằng tải nổi tiếng và mạnh mẽ khác như **Nginx**, **F5**, hay **Envoy**.

### 1.4. Phụ thuộc vào CCM (Cloud Controller Manager)

Tính năng này chỉ hoạt động nếu cụm K8s của bạn nằm trên Cloud (như AWS EKS, Google GKE).

Nếu bạn mang dự án này về chạy thử trên:
- Máy local (Minikube, Kind)
- Máy chủ nội bộ (Bare-metal)

...thì `type: LoadBalancer` sẽ **vô dụng** vì không có ai cấp Load Balancer thật cho bạn cả.

---

## 2. Sự "Cứu rỗi" mang tên Ingress

**Ingress không phải là một Service Type**, nó là một **tài nguyên độc lập hoàn toàn mới** của K8s sinh ra để khắc phục mọi nhược điểm trên:

| Ưu điểm | Mô tả |
|---|---|
| **Khai báo 100% bằng YAML** | Cấu hình đường dẫn, gán chứng chỉ HTTPS (TLS), đổi thuật toán... tất cả đều được định nghĩa rõ ràng bằng các `annotations` bên trong file YAML. |
| **Cực kỳ tiết kiệm** | 100 Services cũng chỉ dùng chung **ĐÚNG 1 Load Balancer** vật lý. Ingress sẽ định tuyến thông minh dựa trên đường dẫn (VD: `/api` trỏ vào Backend, `/` trỏ vào Frontend) hoặc theo tên miền (Host-based routing). |
| **Tự do linh hoạt** | Bạn có thể cài đặt bất kỳ `Ingress Controller` nào mình thích (Nginx Ingress, Traefik, ALB Ingress, v.v.). |
| **Chạy mọi nơi** | Hoạt động trơn tru từ Minikube dưới local cho đến các hệ thống Cloud khổng lồ. |

### Minh họa ý tưởng: 1 Load Balancer, nhiều Service

```
                     ┌─────────────────────┐
   Internet  ──────▶ │   Load Balancer      │
                     │   (duy nhất 1 cái)   │
                     └──────────┬───────────┘
                                │
                     ┌──────────▼───────────┐
                     │      Ingress          │
                     │  (định tuyến thông    │
                     │   minh theo path/host)│
                     └──────────┬───────────┘
                ┌────────────────┼────────────────┐
                ▼                ▼                ▼
         /  → frontend    /api → backend    /admin → admin-dashboard
```

---

## 3. Ưu điểm duy nhất của LoadBalancer Service

Ưu điểm duy nhất của nó là **Dễ thiết lập** (Dễ thao tác). Bạn chỉ cần đổi chữ `ClusterIP` thành `LoadBalancer` là xong.

Còn Ingress sẽ yêu cầu bạn cấu hình phức tạp hơn nhiều:
1. Phải cài **Ingress Controller**
2. Viết **Ingress Resource** (YAML định nghĩa routing)

---

## Bảng so sánh tổng hợp

| Tiêu chí | LoadBalancer Service | Ingress |
|---|---|---|
| Tính khai báo (IaC) | ❌ Hạn chế, phải cấu hình thủ công trên Cloud Console | ✅ 100% khai báo qua YAML/annotations |
| Chi phí | ❌ 1 Service = 1 Load Balancer (đắt đỏ khi scale) | ✅ Nhiều Service dùng chung 1 Load Balancer |
| Linh hoạt công nghệ | ❌ Bị khóa theo Cloud Provider | ✅ Tự do chọn Ingress Controller (Nginx, Traefik...) |
| Chạy trên local/on-premise | ❌ Không hoạt động (cần CCM của Cloud) | ✅ Hoạt động mọi nơi |
| Độ dễ thiết lập | ✅ Rất đơn giản | ❌ Phức tạp hơn (cần cài Controller + viết Resource) |

---

## Tổng kết

- `type: LoadBalancer` dễ dùng nhưng có 4 nhược điểm nghiêm trọng ở quy mô lớn: **không khai báo được, tốn kém, khóa công nghệ, phụ thuộc Cloud**.
- **Ingress** là giải pháp thay thế: một tài nguyên K8s riêng biệt cho phép định tuyến thông minh, khai báo đầy đủ qua YAML, và chỉ cần **1 Load Balancer duy nhất** cho toàn bộ hệ thống.
- Đánh đổi của Ingress là độ phức tạp khi thiết lập ban đầu (cần cài Ingress Controller).
- Đây là kiến thức nền tảng cực kỳ quan trọng, thường xuất hiện trong các câu hỏi phỏng vấn DevOps/SRE.
