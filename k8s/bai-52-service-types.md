# Bài 52: 3 Cấp độ mở cửa mạng trong Kubernetes (Service Types)

## Giới thiệu

Bài này chính thức giải đáp câu hỏi lớn để ngỏ ở cuối [Bài 51](./bai-51-deploy-microservices-eks.md): **Tại sao web đã chạy xanh lè mà gõ IP lên trình duyệt lại báo lỗi (Time Out)?**

Mạng của Kubernetes là một thế giới **hoàn toàn khép kín**. Để "đục tường" cho phép truy cập vào bên trong, K8s cung cấp **3 cấp độ mở cửa** thông qua 3 loại Service (Service Types).

Dưới đây là bản tóm tắt 3 cấp độ này từ thấp đến cao.

---

## 1. ClusterIP — Đóng cửa bảo nhau

| Thuộc tính | Chi tiết |
|---|---|
| **Phạm vi** | Chỉ nội bộ trong cụm Kubernetes |
| **Đặc điểm** | Loại **mặc định**. Cực kỳ bảo mật vì không ai từ bên ngoài (kể cả một con EC2 khác nằm cùng mạng VPC nhưng không thuộc cụm K8s) có thể truy cập được. |
| **Ứng dụng** | Dùng cho các dịch vụ Backend, Database, Redis... — những thứ tuyệt đối không được phơi ra ngoài Internet. |

```yaml
apiVersion: v1
kind: Service
metadata:
  name: cart-service
spec:
  type: ClusterIP   # Mặc định, có thể bỏ qua dòng này
  ports:
    - port: 8080
      targetPort: 8080
  selector:
    app: cart
```

---

## 2. NodePort — Mở cửa nội bộ công ty / VPC

| Thuộc tính | Chi tiết |
|---|---|
| **Phạm vi** | Bất kỳ ai có thể kết nối đến IP của máy chủ EC2 (Worker Node) |
| **Cơ chế hoạt động** | K8s mở một cổng ngẫu nhiên (thường từ `30000` đến `32767`) trên **TẤT CẢ** các Node (EC2). Bất kỳ ai gõ `<IP_của_Node>:<NodePort>` đều sẽ được định tuyến thẳng vào Pod bên trong. |
| **Ứng dụng** | Thường dùng để test, hoặc khi công ty có mạng VPN nội bộ và muốn nhân viên truy cập thẳng vào môi trường Staging/Dev mà không cần Public ra Internet. |

```yaml
apiVersion: v1
kind: Service
metadata:
  name: cart-service
spec:
  type: NodePort
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30080   # Cổng mở trên mọi Node, trong khoảng 30000-32767
  selector:
    app: cart
```

---

## 3. LoadBalancer — Mở toang cửa đón khách Internet

| Thuộc tính | Chi tiết |
|---|---|
| **Phạm vi** | Toàn cầu (Internet) |

### Cơ chế hoạt động (Rất quan trọng cho phỏng vấn)

1. Bản thân K8s **KHÔNG tự sinh ra được** Load Balancer. Khi bạn khai báo `type: LoadBalancer`, K8s API Server sẽ đi gọi một thành phần gọi là **CCM (Cloud Controller Manager)**.
2. CCM sẽ nói chuyện trực tiếp với AWS (hoặc Azure, GCP) và bảo: *"Ê AWS, cấp cho tao một cái Load Balancer thật sự đi!"*
3. AWS sẽ tự động khởi tạo một bộ Cân bằng tải (như Classic ELB hoặc Network Load Balancer), lấy cái Domain/FQDN (ví dụ: `http://abc.us-west-2.elb.amazonaws.com`) và trả về cho K8s. Khách hàng dùng URL này để truy cập.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-proxy
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: frontend-proxy
```

> ⚠️ **Lưu ý:** Chế độ này **không hoạt động** nếu bạn chạy Minikube hay Kind dưới máy local (vì máy bạn không phải là AWS để mà cấp Load Balancer được, trừ khi cài thêm tool giả lập).

---

## Bảng so sánh nhanh

| Loại Service | Phạm vi truy cập | Bảo mật | Dùng khi nào |
|---|---|---|---|
| `ClusterIP` | Chỉ trong cluster | Cao nhất | Backend, DB, các service nội bộ |
| `NodePort` | IP của Worker Node (VPC/nội bộ) | Trung bình | Test, truy cập qua VPN nội bộ |
| `LoadBalancer` | Internet | Thấp nhất (public) | Frontend, API Gateway cần public |

---

## 💡 Bước tiếp theo

Áp dụng vào kiến trúc e-commerce 20 microservices của dự án:

- **19 microservices Backend** (Cart, Shipping, Payment...) sẽ giữ nguyên là `ClusterIP`.
- **Duy nhất 1 microservice** là `frontend-proxy` sẽ phải được đổi `type` thành `LoadBalancer`.

```yaml
# Chỉ cần sửa 1 dòng trong Service của frontend-proxy
spec:
  type: LoadBalancer   # <-- Đổi từ ClusterIP sang LoadBalancer
```

---

## Tổng kết

- Mạng Kubernetes mặc định là một thế giới khép kín, hoàn toàn tách biệt với Internet.
- Có 3 cấp độ mở cửa mạng, tương ứng với 3 `type` của Service: `ClusterIP` (nội bộ cluster) → `NodePort` (nội bộ VPC/công ty) → `LoadBalancer` (Internet).
- `LoadBalancer` hoạt động dựa trên cơ chế **Cloud Controller Manager (CCM)** — K8s không tự tạo Load Balancer mà "nhờ" Cloud Provider (AWS/Azure/GCP) tạo hộ.
- Trong kiến trúc microservices, chỉ nên public (`LoadBalancer`) cho service Frontend/Gateway, còn lại giữ `ClusterIP` để đảm bảo an toàn.
