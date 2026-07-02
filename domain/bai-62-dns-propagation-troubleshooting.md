# Bài 62: DNS Propagation và 4 bước Troubleshooting kinh điển

## Giới thiệu

Đây là bài học "thử thách sự kiên nhẫn" kinh điển nhất của dân làm hạ tầng mạng: **chờ đợi cập nhật DNS toàn cầu (DNS Propagation)**.

Sau khi đã hoàn tất mọi cấu hình ở [Bài 61](./bai-61-nameservers-godaddy-ingress-update.md), thực tế phũ phàng là: cấu hình xong xuôi, code chạy hoàn hảo, nhưng gõ tên miền lên trình duyệt web vẫn báo lỗi `Site can't be reached`. Đây là lúc các kỹ năng **Troubleshooting** (Gỡ lỗi) lên ngôi.

---

## 1. Tại sao web chưa chạy ngay? (DNS Propagation)

Khi bạn đổi Name Server từ GoDaddy sang AWS Route 53, sự thay đổi này **không diễn ra ngay lập tức**.

Các nhà mạng trên toàn thế giới (VNPT, FPT, Viettel...) và các bộ định tuyến cần thời gian để cập nhật lại bộ nhớ tạm (**DNS Cache**) của họ.

> ⏳ Quá trình này có thể mất từ **2 giờ đến 48 giờ**.

---

## 2. Bốn bước Troubleshooting để "kê cao gối ngủ"

Thay vì ngồi F5 liên tục trong hoang mang, dưới đây là 4 cách để chắc chắn rằng bạn đã cấu hình đúng và chỉ việc đợi.

### Check 1: Kiểm tra toàn cầu (whatsmydns.net)

Truy cập web [whatsmydns.net](https://www.whatsmydns.net), nhập tên miền vào.

- Nếu thấy các khu vực trên thế giới (như Mỹ, châu Âu) bắt đầu trả về đúng các IP của AWS Load Balancer → DNS đang được lan truyền tốt.

### Check 2: Kiểm tra ALB trên AWS

```
AWS Console → EC2 → Load Balancer → Tab "Listeners and rules"
```

Nếu thấy có rule ghi rõ:

```
IF Host header IS abhishek.shop THEN forward to <target-group>
```

→ Chứng tỏ Ingress Controller đã làm tròn nhiệm vụ cấu hình ALB.

### Check 3: Test ép HTTP Header bằng `curl --resolve`

Dùng lệnh `curl --resolve` để tự ép máy tính phân giải tên miền thành IP của ALB và gửi request thẳng lên đó, **bỏ qua** việc chờ DNS thật:

```bash
curl --resolve abhishek.shop:80:<IP_của_ALB> http://abhishek.shop
```

- Nếu Terminal trả về cục code HTML dài ngoằng → hệ thống K8s bên dưới đã **thông suốt 100%**, vấn đề chỉ còn nằm ở DNS chưa lan truyền.

### Check 4: Check trên máy cá nhân

Mở Terminal trên máy tính đang dùng (**không phải con EC2**) và gõ:

```bash
nslookup abhishek.shop
```

- Nếu nó báo `server can't find` → cục router wifi ở nhà bạn chưa chịu cập nhật DNS.
- Việc duy nhất cần làm lúc này là **đi ngủ hoặc đi chơi** (đợi DNS Propagation hoàn tất).

---

## Bảng tóm tắt 4 bước kiểm tra

| Bước | Công cụ | Mục đích |
|---|---|---|
| 1 | whatsmydns.net | Kiểm tra DNS đã lan truyền toàn cầu chưa |
| 2 | AWS Console (ALB Listeners/Rules) | Xác nhận ALB đã có rule đúng theo Host header |
| 3 | `curl --resolve` | Test trực tiếp hệ thống K8s, bỏ qua DNS thật |
| 4 | `nslookup` (máy cá nhân) | Kiểm tra DNS cục bộ đã cập nhật chưa |

---

## Tổng kết

- DNS Propagation là quá trình cần thời gian (2-48 giờ), không thể ép nhanh hơn.
- 4 bước kiểm tra chéo giúp bạn xác nhận cấu hình đã đúng, tách biệt nguyên nhân "hệ thống lỗi" khỏi "DNS chưa kịp cập nhật".
- `curl --resolve` là công cụ debug cực kỳ hữu ích: cho phép test toàn bộ luồng K8s + ALB mà không cần chờ DNS thật phân giải.
- Ở [Bài 63](./bai-63-fix-domain-mismatch-final.md), dù DNS đã phân giải đúng, vẫn còn một "hạt sạn" nhỏ cần khắc phục trước khi hệ thống chạy hoàn hảo.
