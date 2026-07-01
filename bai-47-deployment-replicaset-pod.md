# Bài 47: Deployment - "Siêu năng lực" của Kubernetes

## Giới thiệu

Bài này giải thích lý do lớn nhất khiến cả thế giới chuyển từ việc chạy Docker thuần túy sang sử dụng Kubernetes. "Siêu năng lực" của K8s nằm ở một tài nguyên (resource) mang tên **Deployment**.

Dưới đây là 3 điểm cốt lõi bạn cần nắm vững để hiểu tại sao 90% các ứng dụng trên K8s đều dùng Deployment.

---

## 1. Nỗi đau của Docker thuần (Không Auto-Healing, Không Auto-Scaling)

### Chữa lành (Healing)

Nếu bạn chạy một container bằng Docker, và container đó bị "sập" do lỗi tràn RAM hoặc crash ứng dụng, nó sẽ **nằm im ở đó**. Dịch vụ của bạn chết.

### Mở rộng (Scaling)

Nếu ứng dụng e-commerce của bạn chạy chiến dịch Flash Sale, lượng truy cập tăng gấp 10 lần. Để container cũ không bị sập vì quá tải, bạn cần nhân bản nó lên 10 container. Việc gõ lệnh thủ công để tạo và cấu hình 10 container ngay lập tức là **bất khả thi**.

---

## 2. Mô hình quản lý 3 tầng của Kubernetes

Để giải quyết bài toán trên, K8s không quản lý trực tiếp Container, mà nó thiết lập một hệ thống phân cấp cực kỳ chặt chẽ:

```
Deployment → ReplicaSet → Pod
```

Hãy tưởng tượng nó như một công xưởng:

| Tầng | Vai trò | Nhiệm vụ |
|---|---|---|
| **Pod** (Công nhân) | Tầng thấp nhất | Một Pod chứa bên trong nó 1 (hoặc vài) Container. Đây là nơi ứng dụng thực sự chạy. |
| **ReplicaSet** (Quản đốc) | Tầng giữa | Nhiệm vụ duy nhất là **"Đếm"**. Nếu chỉ tiêu giao xuống là phải có 3 công nhân làm việc (`replicas: 3`), Quản đốc sẽ liên tục đếm. Nếu 1 Pod bị chết (còn 2), Quản đốc lập tức gọi tuyển thêm 1 Pod mới lấp vào để luôn bảo đảm con số 3. |
| **Deployment** (Giám đốc) | Tầng cao nhất | Đây là tài nguyên mà bạn (kỹ sư DevOps) sẽ trực tiếp viết file YAML để tạo ra. Bạn chỉ cần đưa yêu cầu cho Giám đốc (ví dụ: "Tôi muốn ứng dụng Web chạy 3 bản sao"), Giám đốc sẽ tự động sinh ra Quản đốc (ReplicaSet) để lo phần việc còn lại. |

> ✅ Chính cơ chế "đếm và bù" của ReplicaSet là nguồn gốc của tính năng **Auto-Healing** (Tự chữa lành).

---

## 3. Sức mạnh của file YAML

Khi dùng Deployment, việc nhân bản hệ thống (Scaling) dễ đến mức bạn chỉ cần mở file cấu hình YAML ra, sửa đúng 1 con số:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 10 # <-- Đổi từ 3 lên 10 lúc Flash Sale, và hạ xuống lại 3 khi hết Sale
  selector:
    ...
```

Chỉ cần gõ lệnh:

```bash
kubectl apply -f deployment.yaml
```

K8s sẽ tự động sinh thêm 7 Pod mới trong vài giây.

---

## Tổng kết

- Docker thuần không có cơ chế tự chữa lành (Auto-Healing) hay tự mở rộng (Auto-Scaling).
- Kubernetes giải quyết vấn đề này bằng mô hình 3 tầng: **Deployment → ReplicaSet → Pod**.
- **ReplicaSet** đảm bảo luôn có đúng số lượng Pod mong muốn đang chạy (Auto-Healing).
- **Deployment** là tài nguyên bạn thao tác trực tiếp; chỉ cần đổi `replicas` là có thể scale hệ thống lên/xuống tức thì.

---

## 🔗 Cầu nối sang Bài 48: Sự xuất hiện của Service

Tác giả đặt ra một tình huống để dẫn dắt sang bài tiếp theo:

> Khi một Pod chết đi, ReplicaSet tạo ra một Pod mới để thay thế (Healing). NHƯNG, Pod mới này sẽ mang một địa chỉ IP hoàn toàn mới.
>
> Nếu địa chỉ IP cứ thay đổi liên tục như vậy, làm sao Frontend có thể biết IP mới của Backend để mà gọi API?

Đó chính là bài toán **Service Discovery** (Khám phá dịch vụ) mà tài nguyên **Service** của K8s sẽ giải quyết ở [Bài 48](./bai-48-service.md).
