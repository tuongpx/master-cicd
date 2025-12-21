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
- Cài đặt gitlab, jenkins, dns, npm 
    - Tạo macvlan network cho server cài đặt gitlab, jenkins, dns và npm

    - Tạo `docker-compose.yml`

```bash
secops@devops-lab:/opt/defenselab$ cat docker-compose.yml
# ===================================================================================
# ==        DOCKER-COMPOSE FOR DEVSECOPS LAB (ALL-IN-ONE – IPVLAN L2)              ==
# ==        Domain : defenselab.info                                               ==
# ==        Subnet : 192.168.80.0/24                                               ==
# ==        IP Pool: 192.168.80.100 – 192.168.80.127                               ==
# ===================================================================================

version: "3.8"

services:

  # -------------------------------------------------------------------------------
  # SERVICE 1: TECHNITIUM DNS
  # -------------------------------------------------------------------------------
  technitium-dns:
    image: technitium/dns-server:latest
    container_name: technitium-dns
    hostname: dns.defenselab.info
    restart: unless-stopped
    environment:
      - TZ=Asia/Ho_Chi_Minh
    ports:
      - "5380:5380/tcp"     # DNS Web UI
    volumes:
      - ./technitium-config:/etc/dns
    networks:
      VLAN80:
        ipv4_address: 192.168.80.16


  # -------------------------------------------------------------------------------
  # SERVICE 2: NGINX PROXY MANAGER (REVERSE PROXY + SSL)
  # -------------------------------------------------------------------------------
  nginx-proxy-manager:
    image: jc21/nginx-proxy-manager:latest
    container_name: nginx-proxy-manager
    hostname: npm.defenselab.info
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "81:81"             # Admin UI
    volumes:
      - ./npm/data:/data
      - ./npm/letsencrypt:/etc/letsencrypt
    dns:
      - 192.168.80.16
      - 1.1.1.1
    networks:
      VLAN80:
        ipv4_address: 192.168.80.101


  # -------------------------------------------------------------------------------
  # SERVICE 3: GITLAB EE
  # -------------------------------------------------------------------------------
  gitlab:
    image: gitlab/gitlab-ee:latest
    container_name: gitlab
    hostname: gitlab.defenselab.info
    restart: unless-stopped
    shm_size: "256m"
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://gitlab.defenselab.info'
        gitlab_rails['initial_root_password'] = 'ChangeMe@2025'
    volumes:
      - ./gitlab/config:/etc/gitlab
      - ./gitlab/logs:/var/log/gitlab
      - ./gitlab/data:/var/opt/gitlab
    ports:
      - "22:22"             # SSH (direct, not via NPM)
      - "5050:5050"         # Registry internal
    dns:
      - 192.168.80.16
      - 1.1.1.1
    networks:
      VLAN80:
        ipv4_address: 192.168.80.102


  # -------------------------------------------------------------------------------
  # SERVICE 4: GITLAB RUNNER
  # -------------------------------------------------------------------------------
  gitlab-runner:
    image: gitlab/gitlab-runner:latest
    container_name: gitlab-runner
    restart: unless-stopped
    privileged: true
    depends_on:
      - gitlab
    volumes:
      - ./runner-config:/etc/gitlab-runner
      - /var/run/docker.sock:/var/run/docker.sock   # ⚠️ LAB ONLY
    dns:
      - 192.168.80.16
      - 1.1.1.1
    networks:
      VLAN80:
        ipv4_address: 192.168.80.103


  # -------------------------------------------------------------------------------
  # SERVICE 5: JENKINS (ALL-IN-ONE CONTROLLER)
  # -------------------------------------------------------------------------------
  jenkins:
    build:
      context: ./jenkins
    container_name: jenkins
    hostname: jenkins.defenselab.info
    restart: unless-stopped
    user: root
    privileged: true           # ⚠️ LAB ONLY
    volumes:
      - ./jenkins/jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
      - /project/kube/.kube:/root/.kube              # optional
    dns:
      - 192.168.80.16
      - 1.1.1.1
    networks:
      VLAN80:
        ipv4_address: 192.168.80.104


# -------------------------------------------------------------------------------
# GLOBAL NETWORK CONFIGURATION
# -------------------------------------------------------------------------------
networks:
  VLAN80:
    external: true   # Pre-created Docker IPVLAN L2 network
```

    - Tạo `Dockerfile` cho Jenkins
```bash
# ------------------------------------------------------------------------------
# Jenkins Controller Image with Docker CLI (LAB ONLY)
# ------------------------------------------------------------------------------
FROM jenkins/jenkins:lts-jdk17

USER root

# Install prerequisites
RUN apt-get update && apt-get install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release \
    && rm -rf /var/lib/apt/lists/*

# Add Docker GPG key
RUN install -m 0755 -d /etc/apt/keyrings \
    && curl -fsSL https://download.docker.com/linux/debian/gpg \
       -o /etc/apt/keyrings/docker.asc \
    && chmod a+r /etc/apt/keyrings/docker.asc

# Add Docker repository
RUN echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
https://download.docker.com/linux/debian \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
    | tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker CLI only (NOT Docker Engine)
RUN apt-get update && apt-get install -y docker-ce-cli \
    && rm -rf /var/lib/apt/lists/*

# Install kubectl and Helm
RUN curl -fsSL "https://dl.k8s.io/release/$(curl -fsSL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" \
    -o /usr/local/bin/kubectl \
 && chmod +x /usr/local/bin/kubectl

ENV HELM_VERSION=v3.17.4
RUN curl -L "https://get.helm.sh/helm-${HELM_VERSION}-linux-amd64.tar.gz" | tar -xz \
    && mv linux-amd64/helm /usr/local/bin/helm \
    && chmod +x /usr/local/bin/helm \
    && helm plugin install https://github.com/chartmuseum/helm-push

# Back to jenkins user
USER jenkins

# Final verification check
RUN kubectl version --client && helm version
```

Chạy lệnh `dc up -d --build --force-recreate`

- Cài đặt Harbor xem tại [đây] (https://github.com/tuongpx/master-cicd/blob/master/docker/README.md)
- Cài đặt argoCD xem tại [đây] (https://github.com/tuongpx/master-cicd/tree/master/argocd/hands-on-argocd-install)

## Cấu hình Harbor Repository

Tạo project trên Harbor

![Alt text](./images/harbor-project-create.png)

## Chuẩn bị Gitlab Repository




