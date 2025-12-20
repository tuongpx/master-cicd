# Hướng dẫn tích hợp Gitlab, Jenkins và Harbor
🎯 Mục tiêu:
Developer push code → Jenkins CI → Harbor → Argo CD → Kubernetes
✅ Không hardcode secret
✅ Least Privilege
✅ GitOps
✅ Audit & Security by Design

## 🧩 KIẾN TRÚC TỔNG THỂ

```bash
Developer
   |
   v
GitLab (SCM + GitOps)
gitlab.defenselab.info
   |
   v
Jenkins (CI)
jenkins.defenselab.info
   |
   |-- Build
   |-- Scan
   |-- Push artifact
   v
Harbor (Artifact Registry + Security)
harbor.defenselab.info
   |
   v
Argo CD (GitOps CD)
argocd.defenselab.info
   |
   v
Kubernetes Cluster (Production)

```

## 🔐 NGUYÊN TẮC ÁP DỤNG SUỐT QUÁ TRÌNH

- ❌ Jenkins KHÔNG deploy

- ❌ Không dùng user cá nhân

- ✅ Robot account / Service account

- ✅ Argo CD chỉ pull Git

- ✅ Kubernetes chỉ pull image

## 🧱 KIẾN TRÚC REPO CHUẨN

### 1️⃣ GitLab – Source of Truth

#### 1.1 App repo
```bash
app-repo/
 ├─ src/
 ├─ Dockerfile
 ├─ chart/
 └─ Jenkinsfile
```

#### 1.2 GitOps repo (deploy)
```bash
gitops-repo/
 ├─ dev/
 ├─ staging/
 └─ prod/
```

### 🔑 QUẢN LÝ TOKEN & CREDENTIAL (TRỌNG TÂM)

#### 2️⃣ GitLab – Token chuẩn production

##### 2.1 Group Access Token (CHO JENKINS – CI)

Truy cập vào Gitlab `Group → Settings → Access Tokens`

```bash
| Thuộc tính | Giá trị                |
| ---------- | ---------------------- |
| Token type | **Group Access Token** |
| Name       | `jenkins-ci`           |
| Role       | Developer              |
| Scope      | ✅ `read_repository`    |
| Scope khác | ❌ Không chọn           |
| Expiration | 90 ngày                |
```
🎯 Mục đích duy nhất: Jenkins clone code

##### 2.2 Deploy Token (CHO ARGO CD)

Tạo tại: GitOps repo
`Repository → Deploy Tokens`

```bash
| Thuộc tính | Giá trị           |
| ---------- | ----------------- |
| Name       | `argocd-readonly` |
| Scope      | `read_repository` |
```
🎯 Argo CD chỉ đọc Git

#### 3️⃣ Jenkins – Credential Vault

KHÔNG lưu token trong file

###### 3.1 GitLab token

- Type: Secret Text

- ID: gitlab-group-token

###### 3.2 Harbor credential (robot)

- Type: Username + Password

- ID: harbor-robot

#### 4️⃣ Harbor – Robot Account (KHÔNG DÙNG ADMIN)

##### 4.1 Robot cho Jenkins (CI)

```bash
| Thuộc tính | Giá trị          |
| ---------- | ---------------- |
| Name       | `ci-jenkins`     |
| Permission | Push, Pull, Scan |
| Scope      | Project          |
| Expire     | 90 ngày          |
```
👉 Jenkins push image & Helm OCI

##### 4.2 Robot cho Kubernetes (Runtime)

```bash
| Name       | Permission |
| ---------- | ---------- |
| `k8s-pull` | Pull       |
```

