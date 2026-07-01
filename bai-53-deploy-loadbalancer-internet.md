# Bài 53: Đưa ứng dụng E-commerce ra Internet với LoadBalancer

## Giới thiệu

Bài 53 chính là "trái ngọt" sau bao nhiêu công sức setup kiến trúc hạ tầng và viết YAML. Chúng ta đã chính thức đưa được ứng dụng e-commerce ra ngoài Internet!

Đây là một bài thực hành rất thú vị với một vài thủ thuật (trick) thực tế. Dưới đây là 3 điểm nhấn của bài học này.

---

## 1. Thủ thuật sửa cấu hình "nóng" (`kubectl edit`)

Thay vì mở file YAML gốc trong thư mục, sửa lại `type: LoadBalancer`, rồi gõ `kubectl apply`, có một cách nhanh hơn rất nhiều của dân vận hành K8s thực thụ:

```bash
kubectl edit svc frontend-proxy
```

- Lệnh này sẽ mở cấu hình của Service **đang chạy trực tiếp trên bộ nhớ K8s** bằng trình soạn thảo (như `vim` hoặc `nano`).
- Cuộn xuống cuối, đổi `ClusterIP` thành `LoadBalancer`, lưu lại và thoát.
- K8s ngay lập tức ghi nhận sự thay đổi này.

```yaml
spec:
  type: LoadBalancer   # <-- Sửa trực tiếp tại đây rồi lưu (:wq trong vim)
```

> 💡 **Lưu ý:** Cách này tiện cho việc test nhanh, nhưng trong môi trường thực tế (đặc biệt là Production), nên sửa lại trong file YAML gốc và commit vào Git để đảm bảo tính nhất quán (Infrastructure as Code), tránh tình trạng cấu hình "trôi" (config drift) không ai kiểm soát được.

---

## 2. Sự "trễ nhịp" của Cloud (Tại sao phải đợi 5 phút?)

Khi gõ:

```bash
kubectl get svc
```

Bạn sẽ thấy K8s trả về **ngay lập tức** một đường dẫn tên miền (FQDN) rất dài của AWS, ví dụ:

```
xxxxxxxx.us-west-2.elb.amazonaws.com
```

Tuy nhiên, nếu bạn copy dán lên trình duyệt ngay, web vẫn sẽ báo lỗi.

### Tại sao?

Ở dưới nền, thành phần **CCM (Cloud Controller Manager)** — đã nhắc đến ở [Bài 52](./bai-52-service-types.md) — đang gọi API yêu cầu AWS tạo một bộ Load Balancer thực sự. Bạn có thể vào giao diện **EC2 > Load Balancers** trên AWS Console để kiểm tra trạng thái này.

Quá trình khởi tạo thiết bị mạng này trên AWS mất khoảng **2-5 phút**. Khi nó báo trạng thái `Active`, bạn gắn thêm cổng `:8080` vào đuôi URL thì giao diện web mới chính thức hiện ra:

```
http://xxxxxxxx.us-west-2.elb.amazonaws.com:8080
```

---

## 3. Khẳng định sự thành công của kiến trúc Microservices

Việc giao diện Web tải lên thành công mới chỉ chứng minh **Frontend hoạt động**. Nhưng các thao tác sau mới thực sự quan trọng:

- ✅ **Add to Cart**
- ✅ **Đổi Currency (Tiền tệ)**
- ✅ **Place Order (Đặt hàng)**

### Điều này chứng minh điều gì?

Tất cả các file `Service.yaml` và `Deployment.yaml` nội bộ (type `ClusterIP`) đã cài đặt ở [Bài 51](./bai-51-deploy-microservices-eks.md) đang hoạt động **hoàn hảo**:

- Frontend tìm được Cart
- Checkout tìm được Shipping

...mà không hề bị đứt gãy kết nối mạng — đúng như cơ chế **Service Discovery** đã học ở [Bài 48](./bai-48-kubernetes-service.md).

---

## 🤔 Nhưng khoan đã, tại sao khóa học chưa kết thúc ở đây?

Nếu kiến trúc hiện tại đã chạy tốt, tại sao phải đẻ thêm khái niệm **Ingress** làm gì cho phức tạp?

### Gợi ý trước cho Bài 54: Yếu điểm của LoadBalancer

Hãy tưởng tượng, nếu công ty bạn không chỉ muốn phơi bày `frontend-proxy` ra Internet, mà muốn mở thêm cổng cho:

- `admin-dashboard`
- `payment-gateway`

Nếu dùng `type: LoadBalancer` cho cả 3 service, AWS sẽ tạo ra **3 cái Load Balancer vật lý khác nhau**, và bạn phải trả tiền cho cả 3 — **rất đắt đỏ!**

Đây chính là "Yếu điểm của LoadBalancer" sẽ được phân tích ở Bài 54, làm tiền đề để giới thiệu **Ingress** — một bộ định tuyến siêu thông minh giúp dùng chung **1 Load Balancer** cho hàng trăm dịch vụ.

---

## Tổng kết

- `kubectl edit svc <tên-service>` là cách nhanh để sửa cấu hình một tài nguyên đang chạy trực tiếp, phù hợp cho việc test nhanh.
- Khi đổi Service sang `type: LoadBalancer`, AWS cần vài phút để thực sự khởi tạo thiết bị Load Balancer vật lý — FQDN trả về ngay không đồng nghĩa với việc đã sẵn sàng phục vụ.
- Việc test được các luồng nghiệp vụ xuyên suốt nhiều service (Add to Cart, đổi Currency, Đặt hàng) là bằng chứng rõ ràng nhất cho thấy kiến trúc Service Discovery nội bộ hoạt động chính xác.
- Dùng `LoadBalancer` riêng cho từng service sẽ phát sinh chi phí lớn khi có nhiều service cần public — đây là động lực để tìm hiểu **Ingress** ở bài tiếp theo.
