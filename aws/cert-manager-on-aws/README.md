# LAB03: Triển khai Cert Manager trên EKS

## TẠO & XÁC THỰC ACM SSL BẰNG AWS CLI

### 🎯 Mục tiêu:

- Dùng lệnh để xin chứng chỉ Wildcard *.defenselab.info.
- Dùng lệnh để lấy thông tin CNAME (Name & Value).
- Mang sang Cloudflare cấu hình.

### BƯỚC 1: GỬI YÊU CẦU (REQUEST CERTIFICATE)
Chạy lệnh sau để yêu cầu AWS cấp chứng chỉ cho miền của bạn tại Singapore (ap-southeast-1).

```bash
aws acm request-certificate \
  --domain-name "*.defenselab.info" \
  --validation-method DNS \
  --region ap-southeast-1 \
  --output text
```

👉 KẾT QUẢ: Nó sẽ trả về một dòng ARN duy nhất (Ví dụ: arn:aws:acm:ap-southeast-1:241688915712:certificate/xxxx...).
Hãy Copy dòng ARN đó lại ngay.
