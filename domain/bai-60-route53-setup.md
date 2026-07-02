# Bài 60: Cấu hình AWS Route 53 để trỏ Domain vào ALB

## Giới thiệu

Bài này hướng dẫn chi tiết cách cấu hình dịch vụ **AWS Route 53** để trỏ tên miền (ví dụ: `abhishek.shop`) vào **Application Load Balancer (ALB)** — cầu nối tiếp theo trong quy trình đã đề cập ở [Bài 59](./bai-59-mua-domain-custom-domain.md).

Dưới đây là các bước thực hiện và những điểm mấu chốt.

---

## 1. Hosted Zone (Vùng lưu trữ DNS)

### Hosted Zone là gì?

Nó giống như một **danh bạ điện thoại trên mạng Internet**. Nó nói cho thế giới biết:

> "Nếu có ai gõ `abhishek.shop`, hãy chuyển hướng họ đến địa chỉ IP hoặc tài nguyên này trên AWS."

### Tạo Hosted Zone

1. Truy cập **AWS Route 53**.
2. Tạo một **Public Hosted Zone** với tên miền đã mua, ví dụ: `abhishek.shop`.

```
AWS Console → Route 53 → Hosted zones → Create hosted zone
  Domain name: abhishek.shop
  Type: Public hosted zone
```

---

## 2. Tạo bản ghi (Alias Record) trỏ về ALB

Đây là bước **quan trọng nhất** để liên kết Route 53 với ALB của EKS.

### Các bước thực hiện

1. Tạo một **Record** mới, ví dụ cấu hình thêm tiền tố `www` để có được tên miền đầy đủ là `www.abhishek.shop`.

2. Bật tính năng **Alias to Application and Classic Load Balancer**:

   - Tính năng này cho phép Route 53 **nhận diện và tự động cập nhật IP của ALB**.
   - Bạn **không cần** phải nhập IP của ALB bằng tay.
   - IP của ALB trên AWS cũng thường xuyên thay đổi, nên dùng **Alias là bắt buộc** (khác với bản ghi CNAME/A truyền thống phải nhập IP tĩnh).

3. Chọn đúng **Region** (`us-west-2`) và chọn đúng cái **ALB** đã được sinh ra bởi Ingress Controller (đã tạo ở [Bài 57](./bai-57-alb-ingress-controller-setup.md) và [Bài 58](./bai-58-ingress-yaml-hosts-file.md)).

4. **Simple Routing:** Sử dụng chính sách định tuyến đơn giản (phân phối traffic đều đặn) — phù hợp cho trường hợp chỉ có 1 ALB đích.

### Tóm tắt cấu hình Record

| Trường | Giá trị |
|---|---|
| Record name | `www` |
| Record type | A (Alias) |
| Alias target | Application and Classic Load Balancer |
| Region | `us-west-2` |
| Load Balancer | ALB được tạo bởi Ingress Controller |
| Routing policy | Simple routing |

---

## 3. Sự thật về Name Servers (Vấn đề chuyển giao quyền lực)

Sau khi thiết lập xong Route 53, có một thực tế quan trọng:

> ⚠️ Route 53 đã sẵn sàng nhận khách (traffic), **nhưng GoDaddy** (nơi bán tên miền) **vẫn đang là bên kiểm soát** hướng đi của cái tên miền đó.

### Name Servers (NS)

Route 53 sẽ cấp cho bạn **4 địa chỉ Name Servers (NS)**, dạng tương tự:

```
ns-123.awsdns-45.com
ns-678.awsdns-90.net
ns-234.awsdns-56.org
ns-789.awsdns-01.co.uk
```

### Nhiệm vụ tiếp theo (Bài 61)

Bạn phải copy 4 cái NS này, mang sang tài khoản **GoDaddy** và dán vào đó. Thao tác này tương đương với việc tuyên bố:

> "Từ nay trở đi, mọi quyết định điều hướng của tên miền này sẽ do **AWS Route 53** toàn quyền quyết định."

```
GoDaddy (nơi mua domain)
        │
        │  hiện đang giữ quyền điều hướng (Name Servers mặc định của GoDaddy)
        ▼
Cập nhật Name Servers → trỏ sang 4 NS của Route 53
        │
        ▼
AWS Route 53 (giờ toàn quyền điều hướng domain)
        │
        ▼
Alias Record → ALB → Ingress Controller → frontend-proxy
```

Đây là quy trình chuẩn mực (**Best Practice**) khi tích hợp tên miền mua từ bên thứ 3 vào hệ sinh thái AWS.

---

## Tổng kết

- **Hosted Zone** trong Route 53 đóng vai trò như một "danh bạ DNS" cho tên miền của bạn trên AWS.
- **Alias Record** là loại bản ghi đặc biệt của Route 53, tự động cập nhật theo IP động của ALB — thay thế cho việc nhập IP thủ công.
- Việc mua domain (GoDaddy/Namecheap) và cấu hình Route 53 là hai bước **tách biệt**; domain vẫn do nhà đăng ký gốc kiểm soát cho đến khi bạn cập nhật **Name Servers**.
- Cập nhật 4 Name Servers của Route 53 sang GoDaddy chính là bước "chuyển giao quyền lực" điều hướng DNS — sẽ được thực hiện ở [Bài 61](./bai-61-godaddy-nameservers.md).
