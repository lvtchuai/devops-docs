# Bài 63: Fix lỗi bất đồng bộ "www" và nghiệm thu dự án cuối cùng

## Giới thiệu

Sau khi đã kiên nhẫn chờ đợi và troubleshoot theo 4 bước ở [Bài 62](./bai-62-dns-propagation-troubleshooting.md), đây là bài học cuối cùng của chuỗi **Ultimate DevOps Project**: phát hiện và khắc phục một "hạt sạn" phút chót, rồi chính thức nghiệm thu toàn bộ dự án.

---

## 1. "Hạt sạn" phút chót: Lỗi bất đồng bộ tiền tố `www`

Sau 6-7 tiếng chờ đợi, lệnh `nslookup` trên máy đã trả về đúng IP. Tuy nhiên, **web vẫn không vào được!**

### Nguyên nhân (Lỗi bất đồng bộ)

Có sự **không khớp** giữa cấu hình ở hai nơi:

| Bài học | Nơi cấu hình | Giá trị đã cấu hình |
|---|---|---|
| [Bài 60](./bai-60-route53-setup.md) | Route 53 — Alias Record | `www.abhishek.shop` (**có** `www`) |
| [Bài 61](./bai-61-nameservers-godaddy-ingress-update.md) | `ingress.yaml` — trường `host` | `abhishek.shop` (**thiếu** `www`) |

### Hậu quả

```
Người dùng gõ:  www.abhishek.shop
                       │
                       ▼
         Route 53 phân giải đúng → trỏ về ALB
                       │
                       ▼
         ALB Ingress Controller kiểm tra Host header
                       │
                       ▼
    Host header nhận được: "www.abhishek.shop"
    Luật trong ingress.yaml chỉ chấp nhận: "abhishek.shop"
                       │
                       ▼
              ❌ KHÔNG KHỚP → Từ chối request!
```

Khi người dùng gõ `www.abhishek.shop`, ALB Ingress Controller check file YAML thấy **không khớp quy tắc** nên đóng cửa từ chối!

---

## 2. Cách khắc phục

### Bước 1: Sửa file `ingress.yaml`

Sửa lại dòng `host` cho khớp với bản ghi trên Route 53:

```yaml
# TRƯỚC (sai — thiếu www)
spec:
  rules:
    - host: abhishek.shop
      ...
```

```yaml
# SAU (đúng — khớp với Route 53)
spec:
  rules:
    - host: www.abhishek.shop
      ...
```

### Bước 2: Áp dụng lại cấu hình

```bash
kubectl apply -f ingress.yaml
```

### Bước 3: Đợi và kiểm tra

Đợi vài phút để ALB Ingress Controller cập nhật lại rule, sau đó load lại trình duyệt tại địa chỉ:

```
http://www.abhishek.shop
```

**Kết quả:** Giao diện OpenTelemetry e-commerce hiện ra hoàn hảo qua đường dẫn Internet Public! 🎉

---

## 3. Bài học rút ra

> 🔑 **Nguyên tắc quan trọng:** Khi cấu hình domain xuyên suốt nhiều lớp hạ tầng (DNS Record ở Route 53, Ingress Resource trong K8s), giá trị tên miền phải **khớp chính xác tuyệt đối** ở tất cả các nơi — kể cả việc có hay không có tiền tố `www`. Đây là một dạng lỗi rất dễ mắc phải nhưng lại khó phát hiện, vì mỗi phần riêng lẻ (Route 53, ALB, DNS) đều báo trạng thái "hoạt động bình thường".

---

## Tổng kết toàn bộ chuỗi Custom Domain Configuration

Dự án **Ultimate DevOps Project** đã chính thức khép lại. Việc bạn tự tay đưa được hệ thống **20 microservices** — từ lúc còn là những dòng code nằm im lìm — lên thẳng môi trường Kubernetes phân tán, kết nối ra Internet và gán domain thành công là một minh chứng rất rõ ràng cho năng lực vận hành **Cloud Native** của một kỹ sư hệ thống.

### Hành trình đã đi qua (tóm lược)

| Giai đoạn | Nội dung |
|---|---|
| Kết nối cluster | `kubectl` + EKS ([Bài 44](./bai-44-ket-noi-kubectl-eks.md)) |
| Bảo mật | Service Account & RBAC ([Bài 46](./bai-46-service-account-rbac.md)) |
| Triển khai ứng dụng | Deployment, ReplicaSet, Pod ([Bài 47](./bai-47-deployment-replicaset-pod.md), [Bài 49](./bai-49-deployment-yaml-anatomy.md)) |
| Kết nối nội bộ | Service & Service Discovery ([Bài 48](./bai-48-kubernetes-service.md), [Bài 50](./bai-50-service-yaml-anatomy.md)) |
| Deploy thực tế | 20 microservices lên EKS ([Bài 51](./bai-51-deploy-microservices-eks.md)) |
| Mở cổng Internet | Service Types & LoadBalancer ([Bài 52](./bai-52-service-types.md), [Bài 53](./bai-53-deploy-loadbalancer-internet.md)) |
| Định tuyến thông minh | Ingress & Ingress Controller ([Bài 54](./bai-54-loadbalancer-vs-ingress.md) → [Bài 58](./bai-58-ingress-yaml-hosts-file.md)) |
| Custom Domain | Route 53, Name Servers, DNS Propagation ([Bài 59](./bai-59-mua-domain-custom-domain.md) → [Bài 63](./bai-63-fix-domain-mismatch-final.md)) |

Chúc mừng bạn đã hoàn thành trọn vẹn hành trình triển khai một hệ thống microservices thực tế lên Kubernetes trên AWS EKS, với domain riêng và truy cập công khai qua Internet!
