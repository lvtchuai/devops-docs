# Bài 56: Ingress Controller - Mảnh ghép còn thiếu của Kubernetes

## Giới thiệu

Bài này chính thức giải mã "mảnh ghép còn thiếu" cực kỳ quan trọng mà K8s **cố tình** không cài đặt sẵn: **Ingress Controller** (Người thợ xây).

Như đã nhắc ở [Bài 55](./bai-55-ingress-resource.md): nếu bạn chỉ viết một file Ingress YAML siêu chuẩn định tuyến các kiểu, rồi chạy `kubectl apply`, thì... **chẳng có gì xảy ra cả!** K8s sẽ chỉ lưu bản ghi đó vào bộ nhớ, nhưng không có một dòng traffic nào được xử lý.

Tại sao lại như vậy? Dưới đây là 3 điểm cốt lõi bạn cần nắm.

---

## 1. Sự "trung lập" của Kubernetes (Not opinionated)

Khác với các tài nguyên như `Deployment` hay `Service` (được K8s xử lý tự động dưới nền), **K8s KHÔNG cài sẵn bất kỳ Ingress Controller nào**.

### Lý do

K8s muốn trao lại **100% quyền tự do lựa chọn** cho bạn:

- K8s không ép bạn phải dùng Nginx.
- K8s cũng không ép bạn dùng ALB của AWS hay F5.

Mỗi loại Load Balancer của các hãng khác nhau có những tính năng độc quyền riêng, ví dụ:

- **WAF** (Web Application Firewall)
- **Rate Limiting** (Giới hạn tốc độ request)
- **Whitelisting** (Danh sách IP được phép truy cập)

Bạn cần tính năng của hãng nào, bạn **tự cài Ingress Controller của hãng đó**.

---

## 2. Ai là người tạo ra Ingress Controller?

Chính các công ty phát triển Load Balancer đã viết ra các đoạn phần mềm điều khiển này (thường bằng ngôn ngữ **Go**) và cung cấp cho cộng đồng, ví dụ:

- **Nginx** → Nginx Ingress Controller
- **F5**
- **Traefik**
- **Kong**
- **AWS** → AWS Load Balancer Controller (ALB Ingress Controller)

### Cơ chế hoạt động

Khi bạn cài đoạn phần mềm này lên K8s, nó sẽ chạy ngầm giống như một **người thợ xây liên tục "theo dõi" (watch)**.

```
Ingress Controller (đang chạy ngầm, liên tục "watch")
        │
        │ Phát hiện có Ingress Resource mới được apply
        ▼
Đọc hiểu "bản thiết kế" (Ingress YAML)
        │
        ▼
Bắt tay cấu hình tạo ra Load Balancer vật lý thực sự
```

Khi bạn ném cho nó một cái **Ingress Resource** (Bản thiết kế kiến trúc), nó sẽ đọc hiểu và bắt tay vào cấu hình tạo ra chiếc **Load Balancer vật lý** (Ngôi nhà) thực sự.

---

## 3. Kế hoạch hành động cho chặng cuối dự án

Vì cụm EKS của chúng ta đang nằm trên hạ tầng AWS, giải pháp tối ưu nhất là sử dụng **"thợ xây" chính chủ** từ hệ sinh thái này:

> ✅ **AWS ALB Ingress Controller** (AWS Load Balancer Controller — quản lý Application Load Balancer)

Việc dùng đúng Ingress Controller "chính chủ" của AWS giúp:

- Tận dụng tối đa các tính năng native của AWS ALB (routing theo path/host, TLS termination, tích hợp WAF...).
- Tối ưu hiệu năng và chi phí khi vận hành trên EKS.
- Được AWS hỗ trợ và cập nhật thường xuyên.

---

## Tổng kết

- K8s cố tình **không** cài sẵn Ingress Controller để giữ tính trung lập, cho phép người dùng tự do lựa chọn công nghệ Load Balancer phù hợp.
- Ingress Controller là phần mềm do các hãng (Nginx, F5, Traefik, AWS...) phát triển, chạy ngầm trong cluster để "theo dõi" và biến Ingress Resource (YAML) thành Load Balancer vật lý thật sự.
- Không có Ingress Controller, file Ingress YAML chỉ là cấu hình nằm im, không có tác dụng gì.
- Với hạ tầng AWS EKS, lựa chọn phù hợp nhất là **AWS ALB Ingress Controller** — sẽ được cài đặt và cấu hình ở bài tiếp theo.
