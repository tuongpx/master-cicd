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

- Cài đặt Harbor xem tại [đây](https://github.com/tuongpx/master-cicd/tree/master/docker)
- Cài đặt argoCD xem tại [đây](https://github.com/tuongpx/master-cicd/tree/master/argocd/hands-on-argocd-install)

## Chuẩn bị môi trường

### 1. Tạo Harbor Project
Tại giao diện của Harbor
- Click chọn `NEW PROJECT`
- Điền thông tin của project

![Alt text](./images/harbor-project-create-1.png)

Trong bài lab này, tôi tạo `corejs` project

![Alt text](./images/harbor-project-create.png)

### 2. Chuẩn bị gitlab repository
https://gitlab.defenselab.info/master-cicd/corejs.git

### 3. Chuẩn bị github repository
Github chỉ dùng để lưu trữ helmchart của dự án.

https://github.com/tuongpx/corejs-prod.git

## Thiết lập Jenkins Pipeline

- Chuẩn bị Jenkinsfile thực hiện tự động hóa toàn bộ quy trình CI/CD từ build, push images đến update manifest cho produt lên Github.

```bash
pipeline {
  agent any

  environment {
    APP_NAME   = "corejs"
    DOCKER_TAG = "0.1.${env.BUILD_NUMBER}"

    GIT_REPO = "https://gitlab.defenselab.info/master-cicd/corejs.git"
    GIT_CRED = "gitlab-token"

    GITOPS_REPO = "https://gitlab.defenselab.info/master-cicd/app-manifest.git"

    HARBOR_DOMAIN  = "harbor.defenselab.info"
    HARBOR_PROJECT = "corejs"
    HARBOR_CRED    = "jenkins-harbor"

    FRONTEND_IMAGE = "harbor.defenselab.info/corejs/corejs-frontend"
    BACKEND_IMAGE  = "harbor.defenselab.info/corejs/corejs-backend"
    HARBOR_IMAGE   = "harbor.defenselab.info/corejs/corejs-backend"

    HELM_CHART_DIR = "helmchart"

    AWS_REGION  = "ap-southeast-1"
    ECR_ACCOUNT = "130618649638"
    ECR_REPO    = "tuongpx-backend"
    ECR_IMAGE   = "${ECR_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}"
    AWS_CRED    = "jenkins-aws"

    GITHUB_PROD_REPO = "https://github.com/tuongpx/corejs-prod.git"
    GITHUB_CRED     = "github-token"
  }

  stages {

    stage('Checkout Source') {
      steps {
        git branch: 'master',
            credentialsId: GIT_CRED,
            url: GIT_REPO
      }
    }

    stage('Pre-flight Check') {
      steps {
        sh '''
          docker --version
          helm version
          yq --version
        '''
      }
    }

    stage('Build & Push Docker Images') {
      parallel {
        stage('Frontend Image') {
          steps {
            script {
              docker.withRegistry("https://${HARBOR_DOMAIN}", HARBOR_CRED) {
                sh """
                  docker build -t ${FRONTEND_IMAGE}:${DOCKER_TAG} frontend
                  docker push ${FRONTEND_IMAGE}:${DOCKER_TAG}
                """
              }
            }
          }
        }

        stage('Backend Image') {
          steps {
            script {
              docker.withRegistry("https://${HARBOR_DOMAIN}", HARBOR_CRED) {
                sh """
                  docker build -t ${BACKEND_IMAGE}:${DOCKER_TAG} -f CoreAPI/Dockerfile .
                  docker push ${BACKEND_IMAGE}:${DOCKER_TAG}
                """
              }
            }
          }
        }
      }
    }

    stage('Update Helm values.yaml') {
      steps {
        sh """
          yq -i '
            .frontend.image.repository = "${FRONTEND_IMAGE}" |
            .frontend.image.tag = "${DOCKER_TAG}" |
            .backend.image.repository = "${BACKEND_IMAGE}" |
            .backend.image.tag = "${DOCKER_TAG}"
          ' ${HELM_CHART_DIR}/values.yaml
        """
      }
    }

    stage('Package Helm Chart') {
      steps {
        sh """
          helm lint ${HELM_CHART_DIR}
          helm package ${HELM_CHART_DIR} --version ${DOCKER_TAG}
        """
      }
    }

    stage('Push Helm Chart to Harbor (OCI)') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: HARBOR_CRED,
          usernameVariable: 'HARBOR_USER',
          passwordVariable: 'HARBOR_PASS'
        )]) {
          sh '''
            echo "$HARBOR_PASS" | helm registry login ${HARBOR_DOMAIN} \
              --username "$HARBOR_USER" \
              --password-stdin

            helm push corejs-${DOCKER_TAG}.tgz \
              oci://${HARBOR_DOMAIN}/${HARBOR_PROJECT}/
          '''
        }
      }
    }

    stage('DEV: Update GitLab Manifest (Trigger ArgoCD)') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: GIT_CRED,
          usernameVariable: 'GIT_USER',
          passwordVariable: 'GIT_TOKEN'
        )]) {
          sh """
            rm -rf app-manifest
            git clone https://${GIT_USER}:${GIT_TOKEN}@${GITOPS_REPO.replace('https://','')} app-manifest
            cd app-manifest

            git config user.email "jenkins@defenselab.info"
            git config user.name  "Jenkins Bot"

            sed -i 's|tag:.*|tag: ${DOCKER_TAG}|' helm/values-local.yaml
            git add helm/values-local.yaml
            git commit -m "dev: update image tag ${DOCKER_TAG}" || true
            git push origin main
          """
        }
      }
    }

    stage("QUALITY GATE: QC Approval") {
      steps {
        input message: "QC APPROVAL REQUIRED for version ${DOCKER_TAG}",
              ok: "Approve Release"
      }
    }

    stage("PROMOTE: Push Image to AWS ECR") {
      steps {
        withCredentials([[ $class: 'AmazonWebServicesCredentialsBinding',
                           credentialsId: AWS_CRED ]]) {
          sh '''
            aws ecr get-login-password --region ${AWS_REGION} \
              | docker login --username AWS --password-stdin \
                ${ECR_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com

            docker tag ${HARBOR_IMAGE}:${DOCKER_TAG} ${ECR_IMAGE}:${DOCKER_TAG}
            docker push ${ECR_IMAGE}:${DOCKER_TAG}
          '''
        }
      }
    }

    stage("PROMOTE: Update GitHub Prod Config (Trigger ArgoCD Cloud)") {
      steps {
        withCredentials([
          usernamePassword(
            credentialsId: GITHUB_CRED,
            usernameVariable: 'GH_USER',
            passwordVariable: 'GH_TOKEN'
          ),
          [$class: 'AmazonWebServicesCredentialsBinding',
           credentialsId: AWS_CRED]
        ]) {
          sh """
            rm -rf corejs-prod
            git clone https://${GH_USER}:${GH_TOKEN}@github.com/tuongpx/corejs-prod.git
            cd corejs-prod

            git config user.email "jenkins@defenselab.info"
            git config user.name  "Jenkins Bot"

            AWS_ACCOUNT_ID=\$(aws sts get-caller-identity --query Account --output text)
            ECR_URL="\${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}"

            sed -i "s|repository: .*|repository: \${ECR_URL}|" helm/values-prod.yaml
            sed -i "s|tag:.*|tag: ${DOCKER_TAG}|" helm/values-prod.yaml

            git add helm/values-prod.yaml
            git commit -m "release: promote ${DOCKER_TAG}"
            git push origin master
          """
        }
      }
    }
  }

  post {
    success {
      echo "✅ RELEASE SUCCESS – Version ${DOCKER_TAG}"
    }
    failure {
      echo "❌ PIPELINE FAILED – Please check console log"
    }
    always {
      cleanWs()
    }
  }
}
```
## Kiểm tra kết quả chạy pipeline

Thực hiện build pipeline và theo dõi quá trình thực thi từng stage

![Alt text](./images/pipeline.png)

## Kiểm tra Harbor

Image đã được push lên Harbor với tag version tương ứng build number trong Jenkins.

![Alt text](./images/harbor-helm.png)

## Kiểm tra ECR

Image với tag version tương ứng với build number trong Jenkins đã được push lên.

![Alt text](./images/ecr.png)

## Kiểm tra Github manifest

Kiểm tra file `values-prod.yaml` trên Github đã được update với version tương ứng.

![Alt text](./images/github-manifest.png)

## Cấu hình ArgoCD Onpremise

1. Kết nối Gitlab Repository
- Truy cập ArgoCD UI
- Chọn `Setting` --> `Repositories`
- Add repository : `corejs`
![Alt text](./images/argocd-repo.png)

2. Tạo Application

![Alt text](./images/argocd-app.png)

## Cấu hình ArgoCD AWS

1. Kết nối Gitlab Repository
- Truy cập ArgoCD AWS UI
- Chọn `Setting` --> `Repositories`
- Add repository : `corejs-prod`
- Cấu hình authentication với GitHub Personal Access Token

![Alt text](./images/argocd-aws-repo.png)

2. Tạo Application

![Alt text](./images/argocd-aws-app.png)

✅ Kết quả: Ứng dụng đã được deploy tự động lên cả hai môi trường thông qua GitOps workflow.

# 4. Public ứng dụng ra Internet

## Cấu hình AWS ELB

![Alt text](./images/kubectl-get-ingress.png)

🔑 Lưu lại ELB address để cấu hình DNS trên Cloudflare

## Cấu hình DNS Cloudflare

Tạo bản ghi DNS với các thông số sau:
- Type: `CNAME`
- Name: `corejs`
- Target: `ELB address`

![Alt text](./images/CF-CNAME.png)

Kiểm tra truy cập

![Alt text](./images/corejs-ui.png)

⚠️ Lưu ý: Chỉ public một site (AWS hoặc Local) tại một thời điểm để thực hiện kịch bản DR testing.

# 4. Kịch bản Disaster Recovery

## 1. Giả lập trường hợp hệ thống prod trên AWS bị gián đoạn

Các bước thực hiện
- Truy cập vào ArgoCD AWS
- Chọn Application `corejs-prod`
- Scale replica về 0

Kết quả trả về khi truy cập `corejs.defenselab.info`

![Alt text](./images/corejs-503.png)

## 2. Kích hoạt DR Site (Cloudflare Tunnel Failover)

Các bước thực hiện:
- Truy cập vào `Zero Trust Dashboard` --> `Networks` --> `Connector`
- Chọn Tunnel đang hoạt động `devops-lab` --> `Edit` --> `Published application routes` --> `Added a published application route`
- Điền các thông tin cho route như sau:
  - Subdomain: `corejs`
  - Domain: `defenselab.info`
  - Service: `HTTP`
  - URL: `192.168.80.30:30080` (IP worker node và NodePort của Ingress Controller) 
- Nhấn `Save`

Truy cập lại `corejs.defenselab.info` để kiểm tra
![Alt text](./images/corejs-ui.png)

# 5. Xác nhận ứng dụng đã truy cập được bình thường
✅ Kết quả:
- Website corejs.defenselab.info hoạt động trở lại
- Traffic được phục vụ từ K8S Local (DR Site)
- RTO (Recovery Time Objective) đạt được trong vòng vài phút
- Không mất dữ liệu (RPO = 0)















