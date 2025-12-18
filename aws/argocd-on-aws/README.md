# LAB 02: Triển Khai ArgoCD Trên AWS

## PHẦN 1: CÀI ĐẶT ARGOCD

Trước hết, ta cần cài ArgoCD vào cụm EKS mới tạo.

### Bước 1: Cài đặt (Install)
Mở Terminal (đã trỏ vào cluster tuongpx-lab-cluster), chạy lần lượt:

```bash
# 1. Tạo phòng riêng (Namespace) cho ArgoCD để cách ly với ứng dụng khác
kubectl create namespace argocd

# 2. Cài đặt ArgoCD từ manifest chính hãng (Bản Stable)
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```