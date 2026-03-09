

---

### 1. Báo cáo và Chứng minh (Reporting)

* **Velero:** Không có tính năng tạo báo cáo. Nếu sếp hoặc kiểm toán hỏi: *"Cho tôi xem danh sách các bản backup thành công trong 30 ngày qua"*, bạn sẽ phải ngồi viết script để lọc từ lệnh `velero backup get`, sau đó xuất ra file CSV hoặc Excel.
* **Kasten K10:** Có sẵn mục **Reporting**. Nó tự động tạo ra các biểu đồ, báo cáo PDF/CSV định kỳ gửi qua Email. Nó cho biết rõ tỉ lệ thành công (Success Rate), dung lượng tiêu thụ, và quan trọng nhất là **SLA Compliance** (Có bản backup nào bị trễ so với lịch trình không).

### 2. Nhật ký kiểm toán (Audit Logs)

* **Velero:** Nhật ký nằm rải rác trong Log của Pod hoặc ghi lại sơ sài trong Custom Resource (CRD). Muốn xem "Ai là người đã bấm lệnh Restore bản backup này?", bạn phải đi tra ngược lại `kube-apiserver` audit logs – cực kỳ cực nhọc.
* **Kasten K10:** Có một hệ thống **Audit Log tập trung** ngay trên Dashboard. Nó ghi rõ: User A, vào lúc mấy giờ, đã thực hiện hành động gì (tạo policy, xóa backup, restore...). Kiểm toán viên chỉ cần nhìn vào đó là xong.

### 3. Phân quyền tự phục vụ (Self-Service & RBAC)

* **Velero:** Quyền hạn thường là "Tất cả hoặc không có gì". Rất khó để cấu hình cho một team Developer chỉ được phép Restore ứng dụng của chính họ mà không được quyền xóa bản backup của team khác. Thường thì chỉ Admin Cluster mới nắm Velero.
* **Kasten K10:** Tích hợp sâu với Active Directory, OIDC. Bạn có thể phân quyền: *"Team App-A chỉ thấy và chỉ được restore các bản backup của App-A"*. Điều này đáp ứng tiêu chuẩn **"Privilege of least access"** (Quyền hạn tối thiểu) trong bảo mật.

### 4. Chống Ransomware (Immutability)

* **Velero:** Hỗ trợ "Object Lock" (WORM) trên S3, nhưng bạn phải tự cấu hình dưới tầng Storage (như AWS S3 hoặc MinIO) và cấu hình thủ công trong Velero. Nếu cấu hình sai, bản backup vẫn có thể bị xóa.
* **Kasten K10:** Có tính năng **Immutable Backups** hiển thị ngay trên giao diện. Nó tự động bắt tay với Storage để khóa bản backup, đảm bảo ngay cả Admin có quyền cao nhất cũng không thể xóa bản backup đó trong một khoảng thời gian quy định. Đây là yêu cầu bắt buộc của nhiều chứng chỉ bảo mật.

### 5. Kiểm tra định kỳ (Success Validation)

* **Velero:** Bạn không biết bản backup có thực sự chạy được hay không cho đến khi bạn tự tay Restore thử.
* **Kasten K10:** Có tính năng tự động chạy một Job "Restore Test". Nó sẽ dựng một môi trường tạm, restore bản backup vào đó để kiểm tra xem ứng dụng có lên xanh hay không, sau đó mới xác nhận bản backup đó là **"Hợp lệ"**.

---

### Tóm lại bằng bảng so sánh Compliance

| Tính năng | Velero | Kasten K10 |
| --- | --- | --- |
| **Báo cáo PDF/CSV** | ❌ Không có (Phải tự code) | ✅ Có sẵn, tự động gửi Email |
| **Giao diện Dashboard** | ❌ Không (Chủ yếu là CLI) | ✅ Có, cực kỳ chi tiết |
| **SLA Monitoring** | ❌ Không | ✅ Theo dõi RPO/RTO thực tế |
| **Audit Logs cho User** | ⚠️ Khó truy vết (Phải xem K8s logs) | ✅ Có khu vực riêng, chi tiết từng User |
| **Tự động Test Restore** | ❌ Không | ✅ Có (Automated Verification) |

---

### Flow tư duy điều chỉnh lại cho rõ hơn:

Nếu bạn làm cho một **Dự án cá nhân, Lab hoặc Startup nhỏ**:

> **Velero** là quá đủ. Bạn tự tin với kỹ năng CLI và có thể tự viết vài script để lấy dữ liệu khi cần.

Nếu bạn làm cho **Ngân hàng, Bảo hiểm hoặc Công ty Global**:

> **Kasten K10** là bắt buộc. Vì khi có sự cố, hoặc khi kiểm toán nhảy vào, bạn không thể đưa cho họ một màn hình đen xì toàn chữ (CLI) được. Bạn cần những con số, biểu đồ và bằng chứng thép rằng dữ liệu luôn an toàn.
