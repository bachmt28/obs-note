## I. Nguyên lý hoạt động của Cosign và Kyverno trong kiến trúc này

## I.1 Cosign thực chất làm gì

### I.1.1 Digest là gì
* Image container thực tế được định danh bằng **digest (sha256)**, không phải tag
* Tag chỉ là alias, có thể:
  * bị push đè
  * bị trỏ sang digest khác
Ví dụ:
```
repo/app:uat-123  --> sha256:aaa
repo/app:uat-123  --> sha256:bbb   (sau khi bị push đè)
```
➡️ **Digest mới là “bản chất” của image**.

---
### I.1.2 Cosign ký cái gì
Cosign **KHÔNG ký tag**
Cosign **KÝ DIGEST**
Cụ thể:
* Input: `sha256:...`
* Output:
  * chữ ký số (signature)
  * metadata (ai ký, khi nào, identity)
Chữ ký này được lưu:
* cùng registry (OCI artifact)
* gắn với digest, **không phụ thuộc tag**
➡️ Nếu tag bị đổi nhưng digest khác:
* chữ ký **không khớp**
* verify fail
---
### I.1.3 Vì sao Cosign key phải nằm trong ANTT zone
* Ai giữ private key → người đó quyết định “image nào được tin”
* Nếu dev có key:
  * mọi kiểm soát trước đó = vô nghĩa
* Vì vậy:
  * Cosign private key **chỉ tồn tại trong ANTT job**
  * Jenkins agent riêng
  * Folder riêng
  * Không reuse cho job dev
➡️ Đây là **trust anchor** của toàn bộ hệ thống.
---
## I.2 Kyverno làm gì với Cosign

### I.2.1 Kyverno không “ký”, chỉ “verify”
* Kyverno chạy trong Kubernetes
* Nó không tạo chữ ký
* Nó chỉ:
  1. Nhận request deploy pod
  2. Lấy image reference
  3. Resolve ra digest
  4. Kiểm tra digest đó **đã có chữ ký hợp lệ hay chưa**

---
### I.2.2 Kyverno verifyImages hoạt động thế nào
Trình tự logic:
1. Pod spec gửi lên API server
2. Kyverno intercept admission request
3. Kyverno:
   * resolve image tag → digest
   * fetch chữ ký từ registry
4. Kyverno dùng:
   * public key
   * hoặc identity trust rule
5. Nếu verify OK → cho deploy
6. Nếu không → reject ngay tại admission
➡️ Pod **không bao giờ chạy** nếu image chưa được ký.

---
### I.2.3 Vì sao Kyverno và Cosign không “giao tiếp trực tiếp”
* Cosign là CLI/tool để:
  * tạo chữ ký
* Kyverno là policy engine để:
  * kiểm tra chữ ký

Chúng **không gọi nhau qua API**.
Mối liên hệ là:
* Cosign tạo chữ ký
* Kyverno đọc chữ ký đó từ registry
➡️ **Registry là điểm trung gian tin cậy**.
---
## I.3 Tại sao phải tách ANTT job và Kyverno ngay từ đầu
* ANTT job:
  * quyết định **ký hay không**
* Kyverno:
  * enforce **đã ký mới deploy**
Hai lớp này:
* độc lập
* không tin nhau
* nhưng bổ sung nhau
➡️ Nếu lỡ:
* ANTT ký nhầm → vẫn có audit
* Dev push bậy → Kyverno chặn
---
## II. Trả lời câu hỏi: “Image đã có digest rồi, sao phải ký cho phức tạp”

## II.1 Digest chỉ nói “cái gì”, không nói “ai cho phép”

Digest trả lời được:
* image này có nội dung gì
Digest **KHÔNG trả lời được**:
* image này có được phép deploy hay không
* ai chịu trách nhiệm cho quyết định đó
➡️ Digest là **dữ liệu**, không phải **quyền hạn**.
---
## II.2 Ký = gắn trách nhiệm con người vào artifact kỹ thuật
Khi ký:
* Có:
  * người ký
  * thời điểm
  * bối cảnh (evidence)
* Có thể audit:
  * ai approve
  * dựa trên scan nào
  * dựa trên evidence nào
➡️ Đây là **decision binding**, không phải checksum.
## II.3 Không ký thì hệ thống không phân biệt được 3 trường hợp
Không có chữ ký, hệ thống **không phân biệt được**:
1. Image build đúng pipeline, đã scan, đã approve
2. Image build ngoài pipeline
3. Image bị push đè tag sau khi approve
Cả 3 đều có digest hợp lệ về mặt kỹ thuật.
➡️ **Chữ ký là “dấu xác nhận hợp pháp”**, không phải hash.

---
## II.4 Ký để giải quyết bài toán “trust at deploy time”

Deploy là lúc:
* rủi ro cao nhất
* hậu quả lớn nhất
Ký + Kyverno đảm bảo:
* deploy time chỉ chấp nhận artifact đã được con người có thẩm quyền chấp thuận
➡️ Không ký → deploy dựa trên **niềm tin mơ hồ**
➡️ Có ký → deploy dựa trên **quyết định có trách nhiệm**


## Giá trị bảo mật thực tế của mô hình ký + kyverno + evidence

## III.1 Điều hệ thống này NGĂN được
* Dev push image ngoài pipeline → không ký → không deploy
* Dev push đè tag → digest khác → chữ ký không khớp
* Bypass CI/CD → không có evidence → không có ký
* Deploy nhầm image → Kyverno chặn
---
## III.2 Điều hệ thống này KHÔNG ngăn tuyệt đối (nhưng truy vết được)
* ANTT ký nhầm
* Zero-day xuất hiện sau khi approve
➡️ Nhưng:
* Có evidence
* Có chain
* Có audit trail
➡️ Không phải “không có lỗi”, mà là **có trách nhiệm và truy vết**.
---
## III.3 Vì sao append-only evidence quan trọng hơn automation
* Automation có thể lỗi
* API có thể đổi
* Tool có thể không tích hợp được
Nhưng:
* Append-only record
* prev_record_id chain
* correlation_id
➡️ Cho phép:
* forensic
* truy trách nhiệm
* giải trình với audit bên ngoài
---
## Sumarize

* Cosign + Kyverno không phải “thừa giấy vẽ voi” -> Đây là cách **chuyển niềm tin từ con người sang hệ thống**
* Digest đảm bảo **tính toàn vẹn** -> Chữ ký đảm bảo **tính hợp pháp**
* Kyverno đảm bảo **thực thi không khoan nhượng**
* Evidence đảm bảo **truy vết sau cùng**

> Nếu bỏ ký, hệ thống chỉ biết “chạy được”.
> Nếu có ký, hệ thống biết “chạy đúng và có người chịu trách nhiệm”.
---
