# Bài 51: Deploy toàn bộ 20 Microservices lên cụm EKS

## Giới thiệu

Bài 51 chính là "giờ G" — thời khắc chúng ta chính thức đẩy toàn bộ hệ thống **20 microservices** của dự án lên cụm EKS thực tế.

Đây là một quy trình deploy (triển khai) cực kỳ bài bản mà bất kỳ DevOps nào cũng phải tuân thủ. Dưới đây là 3 bước thực chiến và 1 "cú lừa" cuối bài.

---

## 1. Quy trình Deploy chuẩn mực

### Bước 1: Kiểm tra chéo (Sanity Check)

Trước khi chạy bất kỳ lệnh deploy nào, luôn thực hiện 2 bước kiểm tra:

```bash
# Chắc chắn bạn đang trỏ đúng vào cụm EKS
# (tránh việc deploy nhầm lên cụm Minikube dưới local)
kubectl config current-context
```

```bash
# Đảm bảo không gian làm việc hoàn toàn sạch sẽ
kubectl get all
```

> ⚠️ Đây là bước cực kỳ quan trọng — deploy nhầm cluster là một trong những lỗi phổ biến và nguy hiểm nhất khi làm việc thực tế với nhiều môi trường (Dev/Staging/Prod).

### Bước 2: Thứ tự sinh tồn — Service Account đi trước

Nhắc lại kiến thức về Kube-RBAC ở [Bài 46](./bai-46-service-account-rbac.md): file `serviceaccount.yaml` phải được apply **ĐẦU TIÊN**.

```bash
kubectl apply -f serviceaccount.yaml
```

> ❗ Nếu bạn apply file Deployment trước, Kubernetes sẽ không tìm thấy "thẻ căn cước" có tên `opentelemetry-demo`, dẫn đến toàn bộ hệ thống bị lỗi (`CrashLoopBackOff`).

### Bước 3: Chiến thuật "Siêu YAML" (Mega YAML)

Thay vì gõ lệnh `kubectl apply` 20 lần cho 20 thư mục, gom toàn bộ 40 file (20 Deployment + 20 Service) vào **một file duy nhất** dài gần 2000 dòng (`complete-deploy.yaml`), ngăn cách nhau bởi `---`.

```yaml
# complete-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cart-deployment
spec:
  ...
---
apiVersion: v1
kind: Service
metadata:
  name: cart-service
spec:
  ...
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shipping-deployment
spec:
  ...
---
# ... (tiếp tục cho 18 microservices còn lại)
```

Bạn chỉ cần gõ đúng **1 lệnh** để đưa toàn bộ hệ thống lên mây:

```bash
kubectl apply -f complete-deploy.yaml
```

---

## 2. Thành quả và "Cú lừa" mang tên ClusterIP

Sau khoảng 1 phút, kiểm tra trạng thái:

```bash
kubectl get pods
kubectl get svc
```

Tất cả các dịch vụ đều báo trạng thái `Running`. Đến đây, mọi thứ dường như hoàn hảo:

- Các microservices nội bộ (như Shipping gọi tới Cart) đã kết nối với nhau trơn tru.
- Điều này khẳng định cơ chế **Service Discovery** bằng Labels và Tên Service (đã học ở [Bài 48](./bai-48-kubernetes-service.md)) đang hoạt động **chính xác 100%**.

### Nhưng... làm sao để khách hàng từ trình duyệt truy cập được vào Frontend?

Thử lọc ra IP của service `frontend-proxy`:

```bash
kubectl get svc frontend-proxy
```

Kết quả trả về là một IP bắt đầu bằng `172.x.x.x` — đây là **Private IP**. Khi copy IP này lên trình duyệt, web hoàn toàn không thể tải được (**Time Out**)!

### Nguyên nhân

Có 2 lý do cộng hưởng khiến bạn không truy cập được:

1. **Kiến trúc VPC:** Nếu bạn nhớ lại kiến trúc VPC đã dựng bằng Terraform, cụm EKS đang được bao bọc an toàn bên trong **Private Subnet**.
2. **Loại Service mặc định:** Kiểu mặc định của Kubernetes Service khi tạo ra là `ClusterIP` — nghĩa là nó **CHỈ phục vụ giao tiếp nội bộ** trong cụm K8s, hoàn toàn cách ly với Internet.

```yaml
spec:
  type: ClusterIP   # <-- Đây là "thủ phạm": chỉ cho phép truy cập nội bộ trong cluster
```

> 🔑 Đây chính là "cú lừa" kinh điển của bài học: hệ thống chạy hoàn hảo bên trong, nhưng lại **vô hình** với thế giới bên ngoài. Đây cũng là bàn đạp để bài tiếp theo giới thiệu các loại Service khác (như `LoadBalancer` hoặc `NodePort`) để mở cổng ra Internet.

---

## Tổng kết

- Quy trình deploy chuẩn: **kiểm tra context → apply ServiceAccount trước → apply Deployment/Service**.
- Dùng "Siêu YAML" với dấu `---` giúp deploy toàn bộ hệ thống nhiều microservices chỉ bằng 1 lệnh `kubectl apply`.
- Tất cả Pod chạy `Running` không có nghĩa là hệ thống đã sẵn sàng phục vụ người dùng thật.
- `ClusterIP` (loại Service mặc định) chỉ phục vụ giao tiếp **nội bộ trong cluster**, không thể truy cập từ Internet — đặc biệt khi cụm EKS nằm trong Private Subnet.
- Vấn đề "làm sao mở cổng ra Internet" sẽ là chủ đề trọng tâm của bài học tiếp theo.
