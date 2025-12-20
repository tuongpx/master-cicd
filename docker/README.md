# Cài đặt Docker Chuẩn Production (Production Grade)

⚠️ Nguyên tắc cốt lõi cho Production:
- Cài từ Official Repo của Docker (tránh gói docker.io cũ của Ubuntu).
- Tuyệt đối không thêm user vào group docker (Rủi ro bảo mật root).
- Cấu hình Log Rotation để tránh tràn ổ cứng server.

## Phần 1: Chuẩn bị hệ thống
Trước tiên, cần xóa sạch các phiên bản cũ (nếu có) để tránh xung đột dependencies.

```bash
# 1. Gỡ bỏ các phiên bản cũ 

```bash
sudo dnf remove -y docker \
  docker-client \
  docker-client-latest \
  docker-common \
  docker-latest \
  docker-latest-logrotate \
  docker-logrotate \
  docker-engine
```

# 2. Cài đặt dependency cần thiết

```bash
sudo dnf install -y dnf-plugins-core ca-certificates curl gnupg
```
## Phần 2: Thiết lập Official Repository

Chúng ta sẽ cấu hình để tải Docker trực tiếp từ nguồn chính chủ.

```bash
sudo dnf config-manager \
  --add-repo \
  https://download.docker.com/linux/rhel/docker-ce.repo
```

Update lại hệ thống

```bash
sudo yum update
```
## Phần 3: Cài đặt Docker Engine

```bash
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
## Phần 4: Cấu hình Production (Quan trọng)

### 4.1. Cấu hình Log Rotation (Chống tràn ổ cứng)

Mặc định Docker không giới hạn dung lượng log container. Ta cần sửa file `daemon.json`:

`
# 1. Mở file cấu hình
sudo vimvim /etc/docker/daemon.json

# 2. Paste nội dung sau vào:
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  },
  "storage-driver": "overlay2",
  "exec-opts": ["native.cgroupdriver=systemd"],
  "live-restore": true
}
`
