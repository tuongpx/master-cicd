# 🚀 Lab Bài Tập Cuối Khóa
MỤC TIÊU: XÂY DỰNG HỆ THỐNG HYBRID CLOUD DR & GITOPS

# 1. TỔNG QUAN DỰ ÁN
Xây dựng một hệ thống triển khai ứng dụng (CI/CD) đảm bảo tính sẵn sàng cao (High Availability) và khả năng phục hồi sau thảm họa (Disaster Recovery).

## Thách thức chính:
- Hệ thống chính (Primary) chạy trên Cloud (AWS EKS) để phục vụ khách hàng toàn cầu với tốc độ cao.
- Hệ thống dự phòng (DR Site) chạy tại văn phòng (On-Premise) để đề phòng trường hợp Cloud bị sập (Region Outage) hoặc đứt cáp quang biển.
- Yêu cầu đặc biệt: Code nguồn phải được bảo mật nội bộ. Chỉ phiên bản Release mới được đẩy ra Public Cloud.

# 2. YÊU CẦU KỸ THUẬT (REQUIREMENTS)

A. Hạ tầng (Infrastructure Setup)

🏢 On-Premise (DR Site)
- Dựng cụm K8s Local.
- Dựng bộ công cụ Core: Jenkins, GitLab, Harbor.
- Thiết lập Cloudflare Tunnel để:
- Public DR App: dr.tonytechlab.com -> Trỏ về Nginx Ingress Local.
(Optional) Kết nối ArgoCD Cloud về GitLab Local.

☁️ Cloud (Primary Site)
- Dựng AWS EKS (Production).
- Tạo AWS ECR (Registry).
- Tạo GitHub Repo (Public/Private) để chứa Config cho Production.

B. Quy trình CI/CD (Pipeline Workflow)

![Alt text](./images/Lab-CI-CD-Final-scaled.png)

Viết Jenkinsfile thực hiện luồng công việc sau:

1. Giai đoạn Phát triển (Local Phase):
- Dev push code vào GitLab.
- Jenkins build Docker Image -> Push vào Harbor.
- Jenkins update manifest trên GitLab -> ArgoCD Local sync về K8s Local.
- Mục tiêu: Dev và QC test nội bộ tốc độ cao.

2. Giai đoạn Kiểm duyệt (Quality Gate):
- Pipeline dừng lại, chờ QC bấm nút “Approve”.

3. Giai đoạn Release (Promotion Phase):
- Jenkins đẩy Image từ Local lên AWS ECR.
- Jenkins đẩy Config (file values-prod.yaml) lên GitHub.
- ArgoCD Cloud sync từ GitHub -> Deploy lên EKS.
- ArgoCD Local sync từ GitLab -> Deploy lên K8s Local (Namespace prod-dr) để làm Backup nóng.

C. Kịch bản DR (Disaster Recovery Test)
- Truy cập Web chính (app.tonytechlab.com) đang chạy trên AWS.
- Giả lập sự cố: Tắt cụm EKS (hoặc scale về 0).
- Failover: Vào Cloudflare DNS, chuyển traffic trỏ về Tunnel (DR Site).
- Kết quả: Web vẫn hoạt động bình thường (chạy từ máy Local).

# 3. Thực hiện

## Chuẩn bị hạ tầng
- Cụm k8s local
- Cài đặt gitlab, jenkins, dns, npm (xem tại ([đây] (https://github.com/tuongpx/master-cicd/blob/master/docker/README.md)))



## Cấu hình Harbor Repository

Tạo project trên Harbor

![Alt text](./images/harbor-project-create.png)

## Chuẩn bị Gitlab Repository




