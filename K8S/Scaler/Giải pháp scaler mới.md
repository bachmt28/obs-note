
**Tiêu đề:** Báo cáo bất cập trong quản lý workload bật/tắt ngoài giờ k8s dev-test & đề xuất giải pháp chuẩn hoá

Dear 

Trong thời gian qua, việc xử lý các yêu cầu bật/tắt workload k8s dev-test ngoài giờ hoạt động gặp nhiều bất cập:

1. **Yêu cầu thiếu thông tin cụ thể**

   * Người đưa yêu cầu (thường là PM) chỉ mô tả chung chung, không cung cấp đủ chi tiết như *tên workload, namespace, thời hạn, lý do*.
   * Có trường hợp thông tin sai lệch, lẫn lộn môi trường (dev/test/uat), gây nhầm lẫn khi thực hiện.
   * Bản thân dev OT phát sinh tự phát, cứ nhè cuối tuần gọi

2. **Ngoại lệ toàn bộ namespace không được quản lý chặt**

   * Nhiều khi có yêu cầu bật toàn bộ namespace, em buộc phải loại namespace đó khỏi danh sách quản lý.
   * Khi hết nhu cầu, việc đưa namespace trở lại đôi khi bị quên, dẫn đến sai lệch quản trị.

3. **Quy trình yêu cầu rời rạc, thiếu lưu vết**

   * Nhiều yêu cầu được gửi ad-hoc qua chat, khó kiểm soát và không có lịch sử truy vết. Khi sếp hỏi 'thế nó làm cái gì', em rất khó trả lời
   * Dù có email, việc tìm lại để đối chiếu rất mất thời gian, dễ bỏ sót hoặc xử lý nhầm.

Những hạn chế này dẫn tới:

* Khó kiểm soát tính đúng đắn và tính hợp lệ của ngoại lệ.
* Rủi ro hệ thống bị bật/tắt không đúng nhu cầu, gây lãng phí tài nguyên hoặc gián đoạn dịch vụ.
* Không có cơ chế chuẩn hoá để tổng hợp, báo cáo và ra quyết định dựa trên dữ liệu.

**Vì vậy, Hiện tại em đã dev xong bộ công cụ “Exception Workload On-time Scaler”:
* Gồm các script python
* Kết hợp cùng pipeline jenkins
**nhằm mục đích:**

* Chuẩn hoá cách đăng ký ngoại lệ (dev gửi payload bắt buộc đủ thông tin: requester, reason, workload, end\_date ≤ 60 ngày).
* Tool tự ghi nhận, thực hiện Dedupe, chuẩn hoá dữ liệu và sinh báo cáo digest minh bạch gửi tự động cho các bên.
* Scaler tự động thực thi UP/DOWN theo khung giờ chuẩn (weekday, weekend, holiday) dựa trên danh sách ngoại lệ hợp lệ.
* Lưu vết đầy đủ trong file, có thể đối chiếu và audit bất kỳ lúc nào.

**Chi tiết mô tả giải pháp, nguyên lý, cơ chế hoạt động em đã viết ở các link sau**:

Anh xem review, đánh giá, bổ sung. Em sẽ book họp, giải thích chi tiết nếu cần.
OK rồi sẽ phổ biến đến các PM và các Dev lead ạ 


**Trân trọng,**


