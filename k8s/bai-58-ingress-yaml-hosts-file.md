# Bài 58: Cấu hình Ingress hoàn chỉnh và thủ thuật Local DNS Spoofing

## Giới thiệu

Bài này chính là lúc "chiêu bài" cuối cùng được tung ra để trình diễn toàn bộ sức mạnh bảo mật của Ingress mà chúng ta đã phân tích ở [Bài 55](./bai-55-ingress-resource.md).

Dưới đây là phần giải phẫu file `Ingress.yaml` và thủ thuật "đánh lừa trình duyệt" cực đỉnh để test Host-based routing mà không cần mua tên miền thật.

---

## 1. Giải phẫu Bản thiết kế Ingress (Ingress Resource YAML)

File `ingress.yaml` có 3 thành phần sống còn sau.

### A. Lựa chọn "Thợ xây" (Ingress Class)

```yaml
spec:
  ingressClassName: alb
```

Vì cụm K8s có thể cài nhiều Ingress Controller cùng lúc (Nginx, Traefik, ALB...), dòng này là **bắt buộc**. Nó nói với hệ thống rằng: *"Này, cái bản thiết kế Ingress này chỉ dành cho ông thợ xây ALB của AWS đọc thôi nhé!"* — đã học ở [Bài 56](./bai-56-ingress-controller.md).

### B. Cấu hình chuyên sâu bằng Annotations

```yaml
metadata:
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
```

Đây chính là sức mạnh **"Declarative"** (Tính khai báo) mà `LoadBalancer` Service Type không làm được (xem lại [Bài 54](./bai-54-loadbalancer-vs-ingress.md)).

| Annotation | Ý nghĩa |
|---|---|
| `scheme: internet-facing` | Yêu cầu AWS tạo ra một Load Balancer mở ra Internet (Public), thay vì Load Balancer nội bộ. |
| `target-type: ip` | Yêu cầu ALB ném traffic **thẳng tới IP của Pod** (ở đây là IP của `frontend-proxy`) để tối ưu hóa tốc độ, thay vì phải đi vòng qua EC2 Node. |

### C. Luật định tuyến (Routing Rules)

```yaml
spec:
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: opentelemetry-demo-frontendproxy
                port:
                  number: 8080
```

Luật này cực kỳ nghiêm ngặt: **Chỉ những ai gõ đúng tên miền `example.com`** trên trình duyệt mới được ALB cho qua cửa để đi vào service `frontend-proxy`.

### File Ingress hoàn chỉnh

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ecommerce-ingress
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: opentelemetry-demo-frontendproxy
                port:
                  number: 8080
```

---

## 2. Màn biểu diễn bảo mật xuất sắc

Sau khi chạy:

```bash
kubectl apply -f ingress.yaml
```

K8s cấp cho một đường dẫn Load Balancer của AWS, ví dụ:

```
k8s-default-frontend-xxxxxxxx.us-west-2.elb.amazonaws.com
```

### Lần 1: Truy cập bằng IP

Lấy IP của Load Balancer gõ thẳng lên trình duyệt.

> ❌ **Bị chặn!**

### Lần 2: Truy cập bằng URL gốc của AWS

Lấy nguyên cái đường dẫn dài ngoằng của AWS gõ lên trình duyệt.

> ❌ **Cũng bị chặn nốt!**

### Lý do

Cả hai cách trên đều **không truyền vào Header tên miền `example.com`** mà luật Routing yêu cầu — đúng chính xác cơ chế Host-based routing đã học ở [Bài 55](./bai-55-ingress-resource.md).

---

## 3. Thủ thuật "Hack" máy tính local (Local DNS Spoofing)

**Câu hỏi:** Làm sao để có tên miền `example.com` mà không tốn tiền mua?

**Trả lời:** Dùng thủ thuật kinh điển của dân IT — **sửa file Hosts**.

### Vị trí file Hosts

| Hệ điều hành | Đường dẫn |
|---|---|
| macOS / Linux | `/etc/hosts` |
| Windows | `C:\Windows\System32\drivers\etc\hosts` |

### Thao tác

Thêm một dòng vào cuối file:

```
[IP_của_LoadBalancer] example.com
```

Ví dụ:

```
54.123.45.67 example.com
```

> 💡 **Lưu ý:** Load Balancer của AWS thường trả về nhiều IP động (không cố định), nên trong thực tế bạn cần `nslookup` hoặc `dig` FQDN của Load Balancer để lấy IP hiện tại trước khi thêm vào file Hosts. Đây chỉ là thủ thuật để **test cục bộ**, không dùng cho môi trường Production thật.

### Cơ chế hoạt động

- Thao tác này đánh lừa trình duyệt Chrome dưới máy tính.
- Khi gõ `example.com`, Chrome sẽ **không lên Internet hỏi đường (DNS) nữa**, mà nó bốc ngay cái IP vừa nhập để kết nối.
- Đồng thời, trình duyệt vẫn gửi kèm cái nhãn (HTTP Host Header) `example.com` đi theo request.

```
Trình duyệt (gõ example.com)
        │
        │ (tra file /etc/hosts → không hỏi DNS)
        ▼
Kết nối tới IP: 54.123.45.67
        │
        │ (nhưng vẫn gửi kèm Host: example.com trong HTTP header)
        ▼
ALB Ingress Controller nhận request, kiểm tra Host header
        │
        │ Host header == "example.com" → khớp luật trong Ingress YAML
        ▼
✅ Mở cửa → Route traffic vào frontend-proxy
```

### Kết quả

ALB Ingress Controller thấy nhãn `example.com` đúng y như luật đã viết trong YAML → **Mở cửa** → Giao diện e-commerce hiện ra thành công rực rỡ! 🎉

---

## Tổng kết

- File `Ingress.yaml` gồm 3 phần cốt lõi: `ingressClassName` (chọn Ingress Controller), `annotations` (cấu hình chuyên sâu của ALB), và `rules` (luật định tuyến theo host/path).
- `scheme: internet-facing` và `target-type: ip` là 2 annotation quan trọng để tạo Load Balancer public và tối ưu tốc độ định tuyến trực tiếp đến Pod.
- Ingress với Host-based routing chặn đứng mọi truy cập không mang đúng tên miền khai báo — kể cả truy cập trực tiếp bằng IP hay URL gốc của AWS.
- Sửa file `/etc/hosts` (hoặc file hosts trên Windows) là thủ thuật phổ biến để giả lập tên miền khi test cục bộ, không cần mua domain thật.
