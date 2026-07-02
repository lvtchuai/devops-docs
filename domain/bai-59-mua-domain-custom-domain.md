# Bài 59: Mua tên miền (Domain) để cấu hình Custom Domain cho ứng dụng K8s

## Giới thiệu

Ở [Bài 58](./bai-58-ingress-yaml-hosts-file.md), chúng ta đã dùng thủ thuật "lừa" trình duyệt bằng file `/etc/hosts` với tên miền `example.com`. Nhược điểm của cách này là **chỉ một mình máy tính của bạn truy cập được**.

Bài 59 này sẽ hướng dẫn cách để **cả thế giới** có thể truy cập vào trang web e-commerce của bạn — mở đầu cho chuỗi bài về **Custom Domain Configuration**.

Dưới đây là 3 điểm đúc kết từ bài học.

---

## 1. Sự thật trong doanh nghiệp

Kỹ sư DevOps/SRE **rất hiếm khi** phải tự quẹt thẻ tín dụng cá nhân để mua tên miền. Việc này thường do:

- Phòng IT
- Phòng Marketing
- Ban Giám đốc

...mua sẵn rồi bàn giao lại cho team kỹ thuật cấu hình.

> ⚠️ Tuy nhiên, bạn **bắt buộc phải hiểu quy trình này** để biết cách lấy thông tin cấu hình (Name Servers/DNS Records) khi cần làm việc với domain do người khác mua.

---

## 2. Chiến thuật mua Domain để làm Lab (Cực kỳ tiết kiệm)

Nếu bạn muốn thực hành thật để nghiệm thu dự án này, dưới đây là các mẹo tiết kiệm chi phí.

### Tránh xa các đuôi phổ biến

Các đuôi `.com`, `.net`, `.vn` có giá khá cao (thường từ **$10 - $15/năm**).

### Tuyệt chiêu giá rẻ

Hãy lên **GoDaddy**, **Namecheap** hoặc **Hostinger** và tìm các đuôi mở rộng như:

- `.shop`
- `.online`
- `.xyz`

Ví dụ: tên miền `abhishek.shop` có giá chỉ khoảng **$1 - $1.5/năm** (tương đương ~30.000 VNĐ).

### Bỏ qua các dịch vụ đi kèm

Khi thanh toán, các nhà mạng sẽ gạ gẫm bạn mua thêm:

- **Domain Protection** (Bảo vệ ẩn danh)
- **Email Hosting**

> 💡 Hãy **uncheck (bỏ chọn)** toàn bộ để tránh phát sinh chi phí, vì chúng ta chỉ cần tên miền để test K8s.

### Cảnh báo về "Free Domain"

Không nên dùng các trang đăng ký tên miền miễn phí (như Freenom ngày xưa) vì chúng:

- Cực kỳ chập chờn, hay lỗi
- Đòi hỏi quá nhiều thông tin cá nhân

---

## 3. Bước chuẩn bị cho Route 53

Việc bạn mua tên miền ở GoDaddy hay Hostinger **mới chỉ là bước xác nhận "sở hữu"**. Để cái tên miền đó thực sự "chỉ đường" được vào cái **Application Load Balancer (ALB)** của AWS mà chúng ta đã tạo ở [Bài 58](./bai-58-ingress-yaml-hosts-file.md), chúng ta cần một **trạm trung chuyển DNS**.

```
Domain đã mua (GoDaddy/Namecheap/Hostinger)
        │
        │  (chỉ mới xác nhận quyền sở hữu, chưa "chỉ đường" được)
        ▼
    AWS Route 53 (trạm trung chuyển DNS)
        │
        │  (cấu hình DNS Record trỏ về)
        ▼
   Application Load Balancer (ALB) của Ingress trên K8s
```

Ở [Bài 60](./bai-60-route53-setup.md), chúng ta sẽ tìm hiểu **AWS Route 53** — dịch vụ phân giải tên miền (DNS) quyền lực nhất của AWS — để mang tên miền vừa mua (ví dụ `devopsbyabi.shop`) gán vào Route 53 và "trói" nó vào ALB của Kubernetes.

---

## Tổng kết

- Trong môi trường doanh nghiệp, việc mua domain thường thuộc trách nhiệm của phòng ban khác, nhưng DevOps engineer vẫn cần hiểu quy trình để cấu hình DNS.
- Khi tự làm lab, nên chọn các đuôi domain rẻ (`.shop`, `.online`, `.xyz`) từ các nhà cung cấp uy tín (GoDaddy, Namecheap, Hostinger) thay vì domain miễn phí kém tin cậy.
- Nhớ bỏ chọn các dịch vụ đi kèm không cần thiết (Domain Protection, Email Hosting) để tiết kiệm chi phí.
- Mua domain chỉ là bước xác nhận sở hữu; cần một dịch vụ DNS (như AWS Route 53) để thực sự trỏ domain vào hạ tầng (ALB) của ứng dụng.
