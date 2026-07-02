# Bài 61: Chuyển giao Name Servers và cập nhật Ingress với Domain thật

## Giới thiệu

Bài này là bước thực hành "chuyển giao quyền lực" DNS — hoàn tất bước đã chuẩn bị ở [Bài 60](./bai-60-route53-setup.md) — và cập nhật lại hệ thống K8s để nó chính thức nhận diện tên miền thật.

Dưới đây là 3 thao tác kỹ thuật quan trọng nhất.

---

## 1. Trỏ Name Servers (NS) từ GoDaddy về AWS

### Các bước thực hiện

1. Quay lại giao diện quản lý DNS của **GoDaddy** (nơi mua tên miền).
2. Tìm đến mục **Name Servers** và chọn **"Use my own name servers"** (Dùng máy chủ định danh tùy chỉnh).
3. Copy chính xác **4 bản ghi Name Server** mà AWS Route 53 cấp ở [Bài 60](./bai-60-route53-setup.md) và dán vào đây, sau đó lưu lại.

```
GoDaddy → My Domains → DNS → Nameservers
  → Change → "Use my own name servers"
  → Nhập 4 NS từ Route 53:
      ns-123.awsdns-45.com
      ns-678.awsdns-90.net
      ns-234.awsdns-56.org
      ns-789.awsdns-01.co.uk
  → Save
```

### ⏳ Lưu ý thực tế: DNS Propagation

Thao tác này kích hoạt quá trình **cập nhật DNS toàn cầu (DNS Propagation)**:

- **Không có tác dụng ngay lập tức.**
- Hệ thống mạng trên thế giới sẽ cần một khoảng thời gian (từ vài phút đến **24 giờ**) để đồng bộ thông tin rằng AWS đang quản lý tên miền này.

> 💡 Trong lúc chờ, bạn có thể dùng `nslookup abhishek.shop` hoặc các công cụ online (như whatsmydns.net) để theo dõi tiến trình đồng bộ.

---

## 2. Cập nhật "Luật định tuyến" trong file Ingress

Trong lúc chờ DNS đồng bộ, bạn phải quay lại báo cho Kubernetes biết về sự thay đổi này, vì K8s vẫn đang đinh ninh khách hàng sẽ truy cập qua `example.com` (như cấu hình ở [Bài 58](./bai-58-ingress-yaml-hosts-file.md)).

### Các bước thực hiện

1. Mở file `ingress.yaml` của dịch vụ `frontend-proxy`.
2. Cập nhật trường `host` từ `example.com` thành tên miền thật (ví dụ: `abhishek.shop`).
3. Áp dụng lại cấu hình:

```bash
kubectl apply -f ingress.yaml
```

### So sánh trước và sau

```yaml
# TRƯỚC (Bài 58)
spec:
  rules:
    - host: example.com
      ...
```

```yaml
# SAU (Bài 61)
spec:
  rules:
    - host: abhishek.shop
      ...
```

---

## 3. Bí kíp Debug: Kiểm tra Logs của "Người thợ xây"

**Câu hỏi:** Làm sao để biết ALB Ingress Controller đã thực sự nhận diện và cấu hình ALB theo tên miền mới?

### Kỹ năng Troubleshooting cơ bản và hiệu quả

**Bước 1:** Lấy tên Pod của ALB Controller:

```bash
kubectl get pods -n kube-system
```

**Bước 2:** Đọc log của Pod đó:

```bash
kubectl logs -n kube-system <tên-pod-alb>
```

**Bước 3:** Tìm dòng log xác nhận tên miền mới đã được xử lý, ví dụ:

```
abhishek.shop is processed
```

> ✅ Nếu thấy dòng log này, điều đó chứng tỏ "người thợ xây" (ALB Ingress Controller) đã hoàn thành xuất sắc nhiệm vụ cấu hình lại Load Balancer trên AWS!

---

## Tổng kết

- Chuyển giao Name Servers từ GoDaddy sang Route 53 là bước "trao quyền" điều hướng DNS thực sự cho AWS — cần thời gian để đồng bộ toàn cầu (DNS Propagation).
- Song song đó, phải cập nhật `host` trong file `Ingress.yaml` để khớp với domain thật, rồi `kubectl apply` lại.
- Kiểm tra log của Pod ALB Ingress Controller (`kubectl logs -n kube-system <pod>`) là cách xác thực nhanh chóng rằng cấu hình domain mới đã được xử lý thành công.
- Với các bước này, mọi nền tảng kỹ thuật đã sẵn sàng — bước cuối cùng ở [Bài 62](./bai-62-nghiem-thu-custom-domain.md) sẽ là gõ tên miền thật lên trình duyệt để nghiệm thu thành quả.
