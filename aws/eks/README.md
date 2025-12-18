# LAB 1: Tạo cụm EKS
## Chuẩn bị:
- Một tài khoản AWS Trial có sẵn 100 credit
- Các thành phần lab
    - I. Tạo IAM và Access Key.
    - II. Cài AWS CLI, EKSCTL, KUBECTL
    - III. Tiến hành cài đặt EKS

## I. Tạo IAM và Access Key
### 1. Tạo IAM
- Bước 1: Tạo IAM để truy cập cli lẫn web console.
    - Sau khi tạo user vào phần search gõ IAM và click chọn IAM

![Alt text](./images/IAM.png)

- Bước 2: Tạo user:
    - Tại IAM bạn click vào Users sau đó bấm Create user

![Alt text](./images/create-user.png)

- Bước 3: Tại hộp thoại tạo user:
    - User name bạn gõ user. lưu ý không gõ ký tự đặc biệt chỉ gõ ký tự thường.
    - Sau đó bạn click vào Provide user access to AWS Management Console: ở đây bạn chọn tick vào để có quyền truy cập console trên web management của AWS
    - Phần Console password bạn chọn Custom password và gõ mật khẩu vào.
    - Lưu ý từ phiên bản 2025 về sau việc tạo AWS access key sẽ không tạo lúc tạo user mà sẽ tạo sau khi tạo user xong.
    - Sau đó bấm Next để qua bước Set permissions.

![Alt text](./images/create-user-2.png)

- Bước 4: Tại đây ta click chọn Attach policy dirrectly và chọn Administrator
```bash
Tại sao AWS lại đưa ra 3 lựa chọn? Khi nào dùng cái nào? Hãy cùng phân tích sơ đồ tư duy dưới đây:

1. Add user to group (Thêm vào nhóm) – Chuẩn mực doanh nghiệp
Đây là lựa chọn Best Practice (Thực hành tốt nhất) cho môi trường thực tế.

Tư duy: Hãy tưởng tượng bạn quản lý một công ty có 50 kỹ sư DevOps. Nếu bạn đi gắn quyền cho từng người một, việc quản lý sẽ trở nên hỗn loạn. Thay vào đó, bạn tạo một “Phòng ban ảo” (Group) tên là DevOps-Team và gắn quyền cho Group này.
Ưu điểm: Khả năng mở rộng (Scalability).
Nhân viên mới vào? Chỉ cần thêm vào Group DevOps-Team là tự động có đủ quyền.
Nhân viên nghỉ việc? Xóa khỏi Group là xong.
2. Copy permissions (Sao chép quyền) – Người thừa kế
Tư duy: Dùng cho tình huống bàn giao công việc. Ví dụ: Nhân viên A nghỉ việc, Nhân viên B vào thay thế đúng vị trí đó.
Hành động: Thay vì phải nhớ A từng có những quyền gì, bạn chọn chức năng này để “Clone” (nhân bản) y hệt quyền của A sang cho B. Nhanh chóng và chính xác.
3. Attach policies directly (Gắn trực tiếp) – Lựa chọn cho Lab/Admin
Đây là lựa chọn chúng ta sử dụng trong bài học này.

Tư duy: Dành cho các trường hợp “cá biệt” hoặc “siêu quyền lực”. Thường dùng cho tài khoản Root, tài khoản Admin tổng, hoặc môi trường Lab/Test nhanh gọn.
Tại sao chọn AdministratorAccess cho bài học này?
Policy này chứa một đoạn mã JSON quyền lực nhất AWS:

{
    "Effect": "Allow",
    "Action": "*",
    "Resource": "*"
}

Ý nghĩa: “Cho phép (Allow) thực hiện mọi hành động (Action: *) trên mọi tài nguyên (Resource: *).”
Lý do sư phạm: Khi triển khai EKS, hệ thống sẽ tự động gọi đến hàng chục dịch vụ khác nhau (VPC, EC2, ELB, IAM Role…). Nếu không cấp quyền Admin, học viên sẽ liên tục gặp lỗi “Access Denied” (Từ chối truy cập), gây gián đoạn quá trình học tập.
```
```bash
⚠️ CẢNH BÁO BẢO MẬT: Trong môi trường Production (Chạy thật), TUYỆT ĐỐI HẠN CHẾ dùng Attach policies directly với quyền AdministratorAccess cho nhân viên thường. Hãy luôn tuân thủ nguyên tắc “Least Privilege” (Đặc quyền tối thiểu – chỉ cấp vừa đủ quyền để làm việc).
```
