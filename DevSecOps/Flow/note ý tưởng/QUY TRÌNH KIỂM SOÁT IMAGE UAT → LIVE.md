## (Signing – Enforcement – Forensics)

## 1. Mục tiêu & phạm vi

### 1.1 Mục tiêu
Thiết lập một quy trình CI/CD nhằm:
* Ngăn image **chưa được ANTT phê duyệt** triển khai vào UAT/LIVE
* Đảm bảo **image deploy có chữ ký hợp lệ**
* **Không tin mù dev pipeline**, nhưng vẫn vận hành được trong điều kiện:
  * Dev **có quyền cấu hình UAT CI**
* Khi có hành vi lách quy trình:
  * **Không yêu cầu bắt ngay**
  * Nhưng **bắt buộc truy vết được** (forensic-ready)
### 1.2 Phạm vi áp dụng
* Jenkins (VM-based)
* Docker build
* Nexus Registry (UAT / LIVE)
* Cosign (image signing)
* Kyverno (admission control)
* Fortify + các công cụ ANTT khác
* Event DB (append-only, phục vụ forensic)

---
## 2. Nguyên tắc thiết kế cốt lõi

### 2.1 Những điều **được chấp nhận**
* Dev **có quyền config UAT CI**
* Không có hệ thống nào **an toàn tuyệt đối**
* Luôn tồn tại “điểm chết”
### 2.2 Những điều **không chấp nhận**
* Tin bất kỳ **một nguồn duy nhất**
* Dùng DB hoặc artifact làm **source of truth**
* Cho phép image vào runtime **không có chữ ký**
---
## 3. Các thành phần & vai trò

| Thành phần             | Vai trò                                  | Có được tin tuyệt đối |
| ---------------------- | ---------------------------------------- | --------------------- |
| Jenkins UAT CI         | Build + scan source + phát sinh evidence | ❌                     |
| Fortify                | Scan source code                         | ❌ (cần bind)          |
| Nexus Registry         | Nguồn sự thật về digest                  | ✅                     |
| ANTT Sign Job          | Quyết định ký                            | ✅                     |
| Cosign                 | Ký image                                 | ✅                     |
| Kyverno                | Chặn deploy chưa ký                      | ✅                     |
| Event DB (append-only) | Ghi sự kiện / timeline                   | ❌                     |

---
## 4. Quy trình chi tiết theo 4 bước

## STEP 1 – UAT CI (Dev-owned): Build, Scan, Emit Evidence

### Mục tiêu
* Build image
* Scan source
* Push image lên UAT registry (tag **không cho overwrite**)
* Phát sinh **evidence** cho ANTT kiểm tra chéo
### Thực hiện
1. Dev trigger Jenkins UAT CI
2. Pipeline gửi source lên Fortify
3. Fortify trả về:
   * `scan_job_id`
   * `verdict = PASS`
4. Pipeline build Docker image
5. Push image lên Nexus UAT
6. Nexus trả về **digest**
7. Pipeline ghi **evidence event** vào Event DB (append-only)
### Evidence tối thiểu
* repo
* tag
* digest
* commit SHA
* scan_job_id
* build_url
* timestamp
> ⚠️ Evidence **không được coi là đúng**, chỉ là “lời khai kỹ thuật”.
---
## STEP 2 – ANTT Sign UAT: Verify Evidence → Ký
### Mục tiêu
* ANTT **tự xác minh**
* Không tin input dev gửi
* Chỉ ký khi **mọi thứ nhất quán**
### Thực hiện
1. Nhận request ký (repo, tag, scan_job_id)
2. Resolve digest **trực tiếp từ Nexus**
3. Truy vấn Event DB để lấy evidence UAT CI
4. Verify Fortify:
   * scan_job_id ↔ commit/build
1. Cross-check:
   * digest từ Nexus == digest trong evidence
   * commit trong Fortify == commit trong evidence
### Kết quả
* Nếu **nhất quán** → ký digest bằng Cosign
* Nếu **lệch** → từ chối ký
### Ghi nhận
* Ghi event `UAT_SIGNED` hoặc `UAT_SIGN_REJECTED` vào Event DB

---
## STEP 3 – UAT CD (AM-owned): Deploy + Enforce + Post-scan

### Mục tiêu
* Chỉ cho deploy image **đã ký**
* Thực hiện scan sau deploy
* Quyết định có được promote sang LIVE hay không

### Thực hiện
1. AM trigger Jenkins UAT CD
2. Apply manifest lên K8s UAT
3. Kyverno:
   * Resolve digest
   * Verify chữ ký Cosign
4. Nếu chưa ký → **deny**
5. Nếu ký hợp lệ → **allow**
6. ANTT chạy post-deploy scans
7. ANTT đưa ra verdict promote

### Ghi nhận
* Event deploy allowed/denied
* Event post-scan result

---
## STEP 4 – LIVE CD: Promote → Ký LIVE → Deploy

### Mục tiêu
* LIVE **không dùng lại chữ ký UAT**
* LIVE chỉ nhận image đã qua promote
### Thực hiện
1. ANTT approve promote
2. Promote job:
   * Pull image bằng digest từ UAT
   * Push sang Nexus LIVE
3. Ký lại digest LIVE
4. AM deploy LIVE
5. Kyverno LIVE verify chữ ký
### Ghi nhận
* Event promote
* Event live signed
* Event live deploy allowed/denied

---

## 5. Event DB – Vai trò & nguyên tắc

### 5.1 Nguyên tắc
* Append-only
* Không update
* Không delete
* Không dùng để ra quyết định

### 5.2 DB dùng để làm gì
* Timeline sự kiện
* Ghép bằng chứng
* Forensics
* Truy cứu trách nhiệm

### 5.3 DB **không dùng để**
* Cho phép ký
* Cho phép deploy
* Tin dev

---
## 6. Forensic Scheme – Khi có hành vi lách

### 6.1 Nguyên tắc forensic
* Không cần phát hiện ngay
* Nhưng **không được vô hình**
* Mọi hành vi phải để lại dấu vết ở **nhiều hệ độc lập**

### 6.2 Khi nghi ngờ 1 image
Có thể truy ra:

| Câu hỏi                        | Trả lời được         |
| ------------------------------ | -------------------- |
| Image nào                      | ✅                    |
| Digest nào                     | ✅                    |
| Tag nào                        | ✅                    |
| Ai push                        | ✅ (Nexus)            |
| Ai xin ký                      | ✅ (Jenkins ANTT job) |
| Scan nào được dùng             | ✅                    |
| Có pipeline CI tương ứng không | ✅                    |
| Timeline chính xác             | ✅                    |

### 6.3 Vì sao truy vết được

* Tag UAT **không overwrite**
* Ký bắt buộc qua ANTT job
* Deploy bắt buộc qua Kyverno
* Event DB giữ timeline bất biến

> Lách được ≠ vô hình
> Lách được ≠ chối được

---

## 7. Tổng kết (Executive Summary)

* Hệ thống **không tuyệt đối an toàn**
* Dev có quyền UAT CI → tồn tại ''điểm chết''
* Nhưng:
  * Không có image vào runtime nếu chưa ký
  * Không có ký nếu không cross-check
  * Không có deploy nếu Kyverno không cho
  * Không có lách mà không để lại dấu vết
---
Nếu bạn muốn bước tiếp theo, có 3 hướng rất tự nhiên:
1. Chuẩn hoá **schema Event DB**
2. Viết **forensic checklist** (30 phút tái dựng sự kiện)
3. Viết **policy Kyverno mẫu** cho UAT/LIVE

Bạn chọn hướng nào, tôi làm tiếp đúng hướng đó.
