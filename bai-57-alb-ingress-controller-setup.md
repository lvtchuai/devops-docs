# Bài 57: Cài đặt AWS ALB Ingress Controller lên EKS

## Giới thiệu

Bài này chính là bước "xắn tay áo" vào thực hành — cài đặt "người thợ xây" **ALB Ingress Controller** lên cụm EKS. Đặc biệt, bài học chứa một khái niệm bảo mật cực kỳ quan trọng thường xuyên xuất hiện trong các buổi phỏng vấn DevOps.

Dưới đây là 3 điểm cốt lõi bạn cần nắm vững trong quá trình cài đặt này, sau đó là bộ lệnh thực chiến đầy đủ.

---

## 1. Câu hỏi phỏng vấn kinh điển: Kỹ thuật IAM OIDC Provider

### Bài toán

Ingress Controller (đã học ở [Bài 56](./bai-56-ingress-controller.md)) thực chất chỉ là một cái **Pod** (phần mềm) nằm bên trong K8s. Vậy làm sao cái Pod bé nhỏ này lại có quyền thò tay ra ngoài K8s, ra lệnh cho hệ thống hạ tầng của AWS để tạo một cái **Elastic Load Balancer (ELB)** vật lý?

### Giải pháp

Pod sử dụng **Service Account** (thẻ căn cước nội bộ của K8s — đã học ở [Bài 46](./bai-46-service-account-rbac.md)). Để thẻ nội bộ này có hiệu lực với hệ thống quản lý quyền hạn của AWS bên ngoài, ta cần một cầu nối ủy quyền gọi là **IAM OIDC Provider**.

### Cơ chế

```
Service Account (của Pod)
        │
        ▼  kết nối qua
IAM OIDC Provider
        │
        ▼  đóng giả làm
IAM Role của AWS (đã được cấp quyền tạo ELB)
```

Kỹ thuật này rất tuyệt vời vì nó giúp **giới hạn quyền tối đa** — chỉ cấp quyền cho chính xác cái Pod Controller đó, thay vì cấp quyền bừa bãi cho toàn bộ cụm máy chủ EC2.

---

## 2. Chuẩn bị "Giấy phép" (Permissions Setup)

Trước khi cài đặt phần mềm Ingress Controller, cần làm thủ tục "xin giấy phép" cho nó hoạt động trên AWS:

- **Bước 1:** Tải file `iam_policy.json` do chính AWS viết sẵn (chứa các quyền liên quan đến Load Balancer) và dùng nó tạo ra một **IAM Policy** trên AWS.
- **Bước 2:** Chạy lệnh `eksctl create iamserviceaccount` "thần thánh". Lệnh này xử lý **3 việc cùng lúc**:
  1. Tạo một **IAM Role**.
  2. Gắn Policy trên vào Role đó.
  3. Tự động liên kết Role này với một **Service Account** chuẩn bị tạo trong K8s.

---

## 3. Cài đặt phần mềm bằng Helm

Sau khi đã có "giấy phép", sử dụng **Helm** (Trình quản lý gói — Package Manager của Kubernetes, tương tự như `apt` trên Ubuntu hay `npm` trên NodeJS) để tải và cài đặt ALB Ingress Controller.

Khi chạy lệnh cài đặt Helm, bắt buộc phải chỉ đường dẫn cho phần mềm này bằng cách truyền vào 3 thông số:

| Tham số | Ý nghĩa |
|---|---|
| `vpcId` | ID của mạng VPC |
| `region` | Khu vực (region) của cụm EKS |
| `clusterName` | Tên cụm EKS |

### Nghiệm thu

```bash
kubectl get pods -n kube-system
```

Nếu thấy 2 pods của ALB Controller báo trạng thái `Running` (1/1) và xem log không báo lỗi màu đỏ nào, nghĩa là "người thợ xây" đã vào vị trí và sẵn sàng nhận lệnh!

---

## 4. Hướng dẫn thực chiến từng bước (How to setup ALB add-on)

### Bước 1: Thiết lập OIDC Connector

Khai báo tên cluster:

```bash
export cluster_name=demo-cluster
```

Lấy OIDC ID của cluster:

```bash
oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
```

Kiểm tra xem đã có IAM OIDC provider được cấu hình chưa:

```bash
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4
```

Nếu chưa có, chạy lệnh sau để tạo:

```bash
eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve
```

### Bước 2: Tải và tạo IAM Policy

Tải file IAM policy mẫu do AWS cung cấp:

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
```

Tạo IAM Policy từ file vừa tải:

```bash
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

### Bước 3: Tạo IAM Role gắn với Service Account

```bash
eksctl create iamserviceaccount \
  --cluster=<your-cluster-name> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

### Bước 4: Deploy ALB Controller bằng Helm

Thêm Helm repo của AWS EKS charts:

```bash
helm repo add eks https://aws.github.io/eks-charts
```

Cập nhật repo:

```bash
helm repo update eks
```

Cài đặt ALB Controller:

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=<your-cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=<region> \
  --set vpcId=<your-vpc-id>
```

### Bước 5: Kiểm tra deployment

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

---

## 5. Xử lý lỗi thường gặp: Thiếu quyền `DescribeListenerAttributes`

### Triệu chứng

Khi gõ `kubectl get ingress -n <namespace>` (ví dụ `robot-shop`), bạn có thể không thấy địa chỉ Load Balancer được trả về.

### Nguyên nhân

`AWSLoadBalancerControllerIAMPolicy` thiếu quyền `elasticloadbalancing:DescribeListenerAttributes`.

### Cách kiểm tra

Lấy nội dung policy hiện tại và tìm quyền `elasticloadbalancing:DescribeListenerAttributes`:

```bash
aws iam get-policy-version \
    --policy-arn arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
    --version-id $(aws iam get-policy --policy-arn arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy --query 'Policy.DefaultVersionId' --output text)
```

### Cách khắc phục

**1. Tải policy hiện tại về file:**

```bash
aws iam get-policy-version \
    --policy-arn arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
    --version-id $(aws iam get-policy --policy-arn arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy --query 'Policy.DefaultVersionId' --output text) \
    --query 'PolicyVersion.Document' --output json > policy.json
```

**2. Sửa file `policy.json`, thêm quyền còn thiếu vào mảng `Statement`:**

```json
{
  "Effect": "Allow",
  "Action": "elasticloadbalancing:DescribeListenerAttributes",
  "Resource": "*"
}
```

**3. Tạo phiên bản policy mới và đặt làm mặc định:**

```bash
aws iam create-policy-version \
    --policy-arn arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://policy.json \
    --set-as-default
```

---

## Tổng kết

- **IAM OIDC Provider** là cầu nối cho phép Service Account bên trong K8s "đóng giả" thành một IAM Role của AWS — cơ chế phân quyền an toàn, chi tiết đến từng Pod thay vì cả cụm EC2.
- Quy trình cấp quyền gồm: tạo **IAM Policy** từ file mẫu của AWS → dùng `eksctl create iamserviceaccount` để tạo **IAM Role** + gắn Policy + liên kết Service Account.
- Cài đặt ALB Ingress Controller thông qua **Helm**, với 3 tham số bắt buộc: `vpcId`, `region`, `clusterName`.
- Kiểm tra thành công bằng `kubectl get pods -n kube-system` (2 pods `Running` 1/1).
- Nếu Ingress không trả về địa chỉ Load Balancer, kiểm tra và bổ sung quyền `elasticloadbalancing:DescribeListenerAttributes` vào IAM Policy.
