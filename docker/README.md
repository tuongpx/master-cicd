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

```bash
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
```
### 4.2. Khởi động Service

```bash
sudo systemctl daemon-reload
sudo systemctl start docker
sudo systemctl enable docker
```
# Phần 5: Bảo mật & Alias Tiện ích

⛔ SECURITY WARNING:
Không chạy lệnh sudo usermod -aG docker $USER trên server Production. Điều này tương đương với việc trao quyền Root không cần mật khẩu cho user đó.

Thay vì add user vào group, ta tạo lệnh tắt (alias) để tự động thêm sudo khi gõ lệnh. An toàn và tiện lợi.
```bash
# Chạy lệnh này để ghi alias vào cuối file .bashrc
echo "alias d='sudo docker'" >> ~/.bashrc
echo "alias dc='sudo docker compose'" >> ~/.bashrc

# Kích hoạt ngay lập tức
source ~/.bashrc
```

💡 Tip Troubleshooting:
Nếu gặp lỗi “Command not found” sau khi tạo alias, hãy kiểm tra file ~/.bashrc xem có bị lỗi dấu nháy (quote) do copy paste không. Hãy dùng lệnh nano ~/.bashrc để sửa lại.

# Phần 6: Kiểm tra kết quả

Sử dụng alias d và dc vừa tạo để kiểm tra phiên bản:

```bash
# Kiểm tra docker
d ps

# Kiểm tra compose
dc version
```