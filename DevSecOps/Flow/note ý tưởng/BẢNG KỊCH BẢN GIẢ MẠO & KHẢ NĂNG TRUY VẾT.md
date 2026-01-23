> Hỏi đáp rủi - ro
## Legend – các nguồn đối chiếu

* **Nexus**: registry (digest, push time, account)
* **Fortify**: scan source (scan_job_id, commit)
* **Event DB**: timeline evidence (append-only)
* **ANTT Sign Job**: log ký
* **Kyverno**: admission log
* **Runtime**: image digest đang chạy

---
## Kịch bản 1 – Dev build image ngoài pipeline CI chuẩn
### Mô tả
Dev:
* build image local / pipeline khác
* push vào UAT registry với **tag mới hợp lệ**
* gửi request ký như bình thường
### Có lách được không?
👉 **Có thể qua được nếu làm rất khéo** (điểm trạo môn đã chấp nhận)

### Truy vết bằng cách nào?

| Đối chiếu | Dấu hiệu không khớp                                          |
| --------- | ------------------------------------------------------------ |
| Event DB  | Có `UAT_IMAGE_PUSH` nhưng **không có pipeline CI tương ứng** |
| Jenkins   | Không tồn tại build CI chuẩn tạo ra digest đó                |
| Fortify   | scan_job_id **không khớp commit/build** của CI chuẩn         |
| Timeline  | Thời điểm push image **sau scan**, hoặc lệch logic           |
### Kết luận họp
> Không chặn tuyệt đối, **nhưng không thể vô hình**.
> Có timeline, có digest, có người chịu trách nhiệm.

---
## Kịch bản 2 – Dev sửa Jenkinsfile để emit evidence giả
### Mô tả
Dev:
* chỉnh UAT CI
* ghi evidence “đẹp”
* scan pass thật
* image build ngoài ý đồ ban đầu

### Có lách được không?
👉 **Có thể**
### Truy vết bằng cách nào?

| Đối chiếu | Dấu hiệu                                             |
| --------- | ---------------------------------------------------- |
| Jenkins   | Diff Jenkinsfile / config tại thời điểm build        |
| Event DB  | Evidence tồn tại nhưng **logic pipeline bất thường** |
| ANTT Sign | Log cho thấy evidence đúng hình thức nhưng           |
| Runtime   | Image behavior / hash không giống baseline           |

### Kết luận họp
> Đây là **lạm quyền CI**, không phải lỗi hệ thống.
> Hệ thống **ghi lại đầy đủ dấu vết để truy cứu trách nhiệm**.
---
## Kịch bản 3 – Dev dùng scan PASS của commit khác
### Mô tả
Dev:
* dùng scan_job_id PASS
* nhưng scan của commit A
* image build từ commit B
### Có lách được không?
👉 **KHÔNG**
### Bị chặn ở đâu?

| Bước          | Lý do                                 |
| ------------- | ------------------------------------- |
| ANTT Sign UAT | Fortify scan_job_id ↔ commit mismatch |
| Event DB      | Evidence commit ≠ scan commit         |
### Kết luận họp
> Case này **bị chặn trước khi ký**, không vào được runtime.

---
## Kịch bản 4 – Dev cố ký giả / lạm dụng credential ký
### Mô tả
Dev:
* cố chạy cosign ngoài ANTT job
* hoặc chiếm credential ký
### Có lách được không?
👉 **Rất khó**, gần như **chỉ khi compromise hạ tầng**
### Truy vết bằng cách nào?

| Đối chiếu | Dấu hiệu                       |
| --------- | ------------------------------ |
| Cosign    | Signature metadata (key, time) |
| Jenkins   | Job nào chạy ký                |
| Agent     | Agent nào thực thi             |
| Event DB  | Không có `UAT_SIGNED` hợp lệ   |
### Kết luận họp
> Đây là **incident hạ tầng**, không phải lỗ hổng quy trình.
> Có đầy đủ log để điều tra.
---

## Kịch bản 5 – Dev push đè tag UAT
### Mô tả
Dev:
* push lại tag cũ để đổi digest
### Có lách được không?
👉 **KHÔNG** (đã cấu hình **không cho overwrite tag**)
### Kết luận họp
> Case này **bị triệt tiêu hoàn toàn** bằng registry policy.

---
## Kịch bản 6 – Dev deploy image chưa ký
### Mô tả
Dev / AM:
* deploy trực tiếp image chưa ký
### Có lách được không?
👉 **KHÔNG**
### Bị chặn ở đâu?

| Bước     | Lý do                       |
| -------- | --------------------------- |
| Kyverno  | Signature missing / invalid |
| Event DB | Có `DEPLOY_DENIED`          |
### Kết luận họp
> Chặn realtime, không cần forensic.
---
## Kịch bản 7 – AM promote sai image sang LIVE
### Mô tả
Dev:
* cố promote image UAT chưa qua post-scan
* hoặc sai digest
### Có lách được không?
👉 **KHÔNG**
### Bị chặn ở đâu?

| Bước         | Lý do                    |
| ------------ | ------------------------ |
| ANTT Gate    | Không có verdict promote |
| Promote job  | Digest không match       |
| Kyverno LIVE | Signature không hợp lệ   |

---
## Kịch bản 8 – Dev cố “qua mặt” nhưng muốn không để lại dấu vết
### Mô tả
Dev cố:
* làm đúng logic
* nhưng không để lại bằng chứng
### Có làm được không?
👉 **KHÔNG**
### Vì sao?
Muốn image vào runtime, **bắt buộc đi qua**:
* Nexus (digest)
* ANTT Sign job
* Kyverno
* Event DB
> Không có đường “tắt vô hình”.

---
# TÓM TẮT 
> * Không có hệ thống nào chặn 100% khi dev có quyền CI
> * Nhưng hệ này đảm bảo:
>
>   * ❌ Không deploy nếu chưa ký
>   * ❌ Không ký nếu dữ liệu không nhất quán
>   * ✅ Khi lách xảy ra: **truy vết được đầy đủ**
> * Gian lận **không thể không truy vết**

---
