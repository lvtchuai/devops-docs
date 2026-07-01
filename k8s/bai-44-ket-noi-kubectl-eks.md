# Bài 44: Kết nối `kubectl` từ EC2 đến cụm EKS

## Giới thiệu bài toán

Đây là một bài toán vô cùng kinh điển khi làm việc với Kubernetes trên Cloud: **làm sao để máy tính của bạn (hoặc một con EC2 Bastion Host) có thể "nói chuyện" và điều khiển được cụm EKS vừa tạo?**

Khi bạn vừa cài `kubectl` trên EC2 và gõ:

```bash
kubectl get nodes
```

hệ thống sẽ báo lỗi hoặc không tìm thấy cụm. Lý do là vì `kubectl` đang bị "mù" — nó chưa biết thông tin cấu hình của cụm EKS nằm ở đâu.

Dưới đây là phần giải thích chi tiết các khái niệm cốt lõi và các bước thực hiện.

---

## 1. Bản chất của file `kubeconfig` và `Context`

Để điều khiển các cụm Kubernetes, `kubectl` luôn nhìn vào một file cấu hình mặc định nằm tại đường dẫn `~/.kube/config` (gọi tắt là **kubeconfig**). File này chứa 3 thành phần chính:

| Thành phần | Mô tả |
|---|---|
| **Clusters** | Danh sách các cụm K8s mà bạn có quyền truy cập (bao gồm URL API Server, chứng chỉ CA...). Một công ty có thể có hàng chục cụm (Dev, Staging, Prod). |
| **Users** | Danh sách các tài khoản/thẻ căn cước để đăng nhập vào cụm (Token, Client Key...). |
| **Contexts** (Ngữ cảnh) | Sợi dây liên kết **[1 Cluster + 1 User]**. Context sẽ định nghĩa: *"Tôi là User A, tôi muốn kết nối vào Cluster B"*. |

### Các lệnh kiểm tra ngữ cảnh

```bash
# Xem toàn bộ nội dung file kubeconfig
kubectl config view

# Xem kubectl hiện tại đang trỏ vào cụm nào
kubectl config current-context

# Chuyển đổi nhanh giữa các cụm K8s khác nhau
kubectl config use-context <tên-context>
```

> ⚠️ **Lưu ý quan trọng:** Khi làm việc thực tế, các kỹ sư thường phải quản lý cùng lúc cả cụm Dev và Prod. Việc thành thạo các lệnh kiểm tra `context` này sẽ giúp bạn tránh được những lỗi nguy hiểm như "gõ nhầm lệnh xóa trên cụm Production".

---

## 2. Quy trình 3 bước kết nối từ EC2 đến EKS

### Bước 1: Cài đặt và cấu hình AWS CLI

Vì EKS là dịch vụ do AWS quản lý, `kubectl` cần thông qua AWS CLI để xác thực danh tính.

- Cài đặt AWS CLI trên Linux (unzip gói cài đặt từ AWS).
- Chạy lệnh sau để điền `Access Key` và `Secret Key`:

```bash
aws configure
```

> 💡 Access Key và Secret Key có thể lấy từ file `~/.aws/credentials` dưới máy Local (đã cấu hình ở các bài trước).

### Bước 2: Đồng bộ cấu hình EKS về file `kubeconfig`

Đây là câu lệnh **"thần chú"** quan trọng nhất của bài học:

```bash
aws eks update-kubeconfig --name <tên_cluster_của_bạn> --region us-west-2
```

**Cơ chế hoạt động:**

Lệnh này sẽ:
1. Tự động gọi lên AWS, lấy thông tin **API Endpoint** và **Chứng chỉ bảo mật** của cụm EKS về.
2. Tự động ghi (update) vào file `~/.kube/config` trên con EC2 của bạn.
3. Đồng thời tạo ra một `context` mới tương ứng với cụm này.

### Bước 3: Xác nhận kết nối

Sau khi chạy lệnh trên, `kubectl` đã có đủ thông tin. Gõ:

```bash
kubectl get nodes
```

Hệ thống sẽ trả về danh sách các Worker Nodes (EC2) đang chạy trong cụm EKS của bạn một cách mượt mà.

---

## Tổng kết

Bài học này giúp bạn làm chủ kỹ năng **điều khiển Kubernetes từ xa**, bao gồm:

- Hiểu bản chất file `kubeconfig` và cơ chế `context`.
- Biết cách xác thực EC2 với AWS thông qua AWS CLI.
- Sử dụng thành thạo lệnh `aws eks update-kubeconfig` để kết nối `kubectl` với cụm EKS.
- Nắm được các lệnh kiểm tra/chuyển đổi context để làm việc an toàn giữa nhiều môi trường (Dev/Staging/Prod).
