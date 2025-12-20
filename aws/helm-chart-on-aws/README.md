# Triển Khai Helm Chart Thông qua ArgoCD Kết hợp với Nginx Load Balangcer trên EKS

## 🏗️ PHẦN 1: CÀI ĐẶT NGINX VỚI AWS ACM (NLB SSL TERMINATION)
Đây là bước thay đổi quan trọng nhất. Chúng ta sẽ cấu hình để AWS NLB tự động gán chứng chỉ ACM vào cổng 443.

`Chuẩn bị: Copy cái ARN của chứng chỉ ACM *.tonytechlab.com mà bạn đã có.
(Ví dụ: arn:aws:acm:ap-southeast-1:241688915712:certificate/70c58476-9a59-4bdc-b1df-cca71c88963a)`

Lệnh cài đặt (Helm):

Copy nguyên khối lệnh này vào CMD (Nhớ thay dòng ARN bằng của bạn):

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer \
  --set controller.service.annotations."service.beta.kubernetes.io/aws-load-balancer-type"="nlb" \
  --set controller.service.annotations."service.beta.kubernetes.io/aws-load-balancer-scheme"="internet-facing" \
  --set controller.service.annotations."service.beta.kubernetes.io/aws-load-balancer-ssl-cert"="aarn:aws:acm:ap-southeast-1:130618649638:certificate/c23e55fa-a6b5-4356-909a-30297254c2cb" \
  --set controller.service.annotations."service.beta.kubernetes.io/aws-load-balancer-ssl-ports"="443" \
  --set controller.service.targetPorts.https=http
```

```bash
Giải thích tham số (Tại sao đây là chuẩn AWS?):

    + ...ssl-cert: Gắn chứng chỉ ACM trực tiếp vào NLB.

    + ...ssl-ports: Mở cổng 443 trên NLB.

    + targetPorts.https=http: SSL Offloading. NLB giải mã xong sẽ gửi traffic HTTP (80) vào Nginx. Nginx không cần lo việc giải mã nữa (nhẹ hơn, nhanh hơn).
```