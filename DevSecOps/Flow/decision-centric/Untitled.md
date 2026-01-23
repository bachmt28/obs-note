# 1) Bài toán & ràng buộc thực tế

## Mục tiêu

* Dev build image ở UAT CI.
* ANTT chỉ cho push/ký/đẩy sang LIVE khi đạt điều kiện.
* UAT/LIVE deploy **chỉ nhận image đã ký** (enforce bằng Kyverno).
* Khi có “lách” xảy ra: **không cần bắt ngay**, nhưng **phải truy vết được**.

## Ràng buộc (điểm chết đã chấp nhận)

* Jenkins chạy trên VM.
* Dev **có quyền config UAT CI** ⇒ pipeline và evidence do CI phát sinh **không thể tin tuyệt đối**.
* Nexus UAT **không cho overwrite tag** (tag UAT là bất biến).

---

# 2) Kiến trúc 4 bước (luồng chuẩn hiện tại)

## STEP 1 — UAT CI (Dev-owned)

**Làm:**

* Scan source (ví dụ Fortify) → lấy `scan_job_id`, verdict.
* Build Docker image.
* Push lên Nexus UAT theo tag **unique** (commit SHA/build number), không reuse.
* Resolve digest từ registry.

**Xuất “evidence build” (append-only):**

* `commit_sha`
* `uat_image_tag`
* `uat_digest` (sha256)
* `scan_job_id` (Fortify)
* `build_url`
* timestamp

> Evidence này là “lời khai kỹ thuật”, **không dùng làm nguồn tin tuyệt đối**, chỉ để cross-check.

---

## STEP 2 — ANTT Sign UAT (ANTT-owned, trust boundary)

**Nguyên tắc:** ANTT job **không tin mù** dữ liệu dev gửi.

**Làm:**

* Nhận request ký (ít nhất: `repo`, `tag`, `scan_job_id`).
* Tự resolve digest **trực tiếp từ Nexus** theo tag.
* Query evidence build (append-only) để lấy commit/digest đã khai.
* Verify Fortify: `scan_job_id` ↔ commit/build.
* Cross-check:

  * digest từ Nexus == digest trong evidence
  * commit từ Fortify == commit trong evidence

**Kết quả:**

* Consistent → cosign sign digest và push signature.
* Mismatch → reject, ghi reason.

---

## STEP 3 — UAT CD (AM-owned) + Post-deploy scans (ANTT)

**Làm:**

* AM deploy lên UAT.
* Kyverno UAT chặn nếu image digest chưa có signature hợp lệ.
* Deploy ok → chạy post-deploy scans (hiện tại ZAP; sau này Prisma/khác).

**Kết quả post-scan** dùng để quyết định promote sang LIVE.

---

## STEP 4 — LIVE Promote + LIVE CD

**Làm:**

* Chỉ promote khi ANTT gate post-scan PASS.
* Promote job pull image theo **digest UAT**, push sang **Nexus LIVE**.
* Ký lại digest ở LIVE (cosign live key/identity).
* AM deploy LIVE.
* Kyverno LIVE verify signature.

---

# 3) Evidence – đang có 2 loại (phải tách bạch)

## 3.1 Evidence build (STEP 1)

Gắn với:

* commit
* image digest (UAT)

Dùng để:

* phục vụ ANTT Sign UAT cross-check và quyết định ký.

## 3.2 Evidence post-deploy (STEP 3) – ví dụ ZAP

ZAP scan **hành vi runtime**, nên evidence phải gắn với:

* `image_digest`
* `runtime context` (namespace/deployment/endpoint)
* `scan profile`
* `policy version`
* `report hash + report ref`

**ZAP evidence tối thiểu (append-only record):**

* subject:
  * `image_digest`
  * `namespace`, `deployment`, `service_url`
* scan meta:

  * `tool=zap`, `zap_version`, `scan_profile`, `duration`
* decision meta:

  * `policy_version`
  * `verdict=PASS|FAIL|CONDITIONAL`
  * `summary` (high/medium/low count)
* trace:

  * `zap_job_id`
  * `report_ref` (link tới kho báo cáo)
  * `report_hash`
  * timestamp

**Quan trọng:**

* Không lưu raw report trong DB; DB chỉ lưu index + hash + ref.
* Post-scan evidence dùng cho **promote**, không dùng để “ký”.

---

# 4) “Nếu nó lách thì sao?” — truy vết dựa vào đâu

## Những điểm “không thể tránh” nếu muốn vào runtime

* Nexus (digest thật + lịch sử push)
* ANTT Sign job (log ký + identity)
* Kyverno (admission allow/deny)
* Evidence DB append-only (timeline và correlation)

## Các kịch bản giả mạo chính & cách phát hiện mismatch (tóm tắt)

1. **Build ngoài pipeline rồi push tag mới**

   * Mismatch: không có lineage CI tương ứng; scan_job_id/commit/timeline không khớp; evidence thiếu/khác.
2. **Dùng scan PASS của commit khác**

   * Bị chặn ở Sign job: Fortify commit mismatch.
3. **Cố deploy chưa ký**

   * Bị chặn ở Kyverno.
4. **Cố promote khi post-scan fail/không có**

   * Gate promote dựa evidence post-scan theo digest; thiếu là fail-close.
5. **Cố ký giả (chiếm key/agent)**

   * Truy vết qua job/agent identity, signature metadata, event trail.

**Kết luận forensic:**

* Không đảm bảo “không ai lách được”, nhưng đảm bảo **lách không vô hình** và **không chối được**.

---

# 5) Vấn đề lớn khi mở rộng thêm tool (Prisma/Checkov/Gitleaks/Trivy…)

Nếu giữ cách “tool-centric”:

* Mỗi tool thêm vào = thêm nhánh logic, thêm schema, thêm quy tắc pass/fail riêng, thêm tranh cãi.
* Hệ sẽ “chết vì mở rộng” (operability & bus-factor).

---

# 6) Mô hình decision-centric (đề xuất mở rộng bền vững)

## Nguyên tắc vàng

* Tool chỉ phát sinh kết quả.
* Evidence phải **chuẩn hoá** theo schema chung.
* **Decision Engine** (scope ANTT) mới ra quyết định theo **policy versioned**.
* Downstream (sign/promote/deploy) chỉ tin **DecisionRecord**, không tin trực tiếp tool output.

## Cấu trúc

* Collectors/Adapters: nhận output thô (Fortify/Checkov/Gitleaks/Trivy/Prisma/ZAP) → chuẩn hoá `ScanResult`.
* Evidence Store: append-only `ScanResult` + report_ref/report_hash.
* Decision Engine: gom `ScanResult` theo subject (commit/digest/runtime) + áp `policy_version` → xuất `DecisionRecord`.
* Gates (Sign UAT, Promote LIVE, Deploy LIVE): tiêu thụ DecisionRecord.

## Rủi ro mới của decision-centric (cần làm chặt)

* Decision Engine trở thành “crown jewel” (nếu compromise → phát hành quyết định sai).
* Evidence poisoning (ai cũng gửi ScanResult) nếu không khóa ingestion.
* Policy drift nếu không version/immutable.

**Ba “đinh” bắt buộc nếu làm decision-centric:**

1. DecisionRecord phải **được ký** và verify ở gate.
2. Evidence ingestion phải **allowlist + identity-bound**; report_ref do collector tin cậy tạo.
3. Policy phải **versioned + immutable + change control**.

---

# 7) Trade-off: đáng hay không đáng?

## Kết luận thực dụng

* Nếu bạn sẽ có **nhiều tool**, nhiều team, cần ban hành quy trình và audit: **đáng**.
* Nếu làm nửa mùa (không có 3 “đinh”): **không đáng** vì tạo thêm điểm chết.

## Bản chất

* Trước: rủi ro rải rác, khó kiểm soát.
* Sau: rủi ro tập trung vào Decision Engine nhưng **dễ harden** hơn và **giảm chi phí mở rộng** mạnh.

---

# 8) Chốt khuyến nghị triển khai theo lộ trình (không phá hệ đang chạy)

1. Giữ nguyên 4 bước hiện tại (sign + kyverno + evidence build + zap evidence).
2. Chuẩn hoá schema chung tối thiểu cho:

   * `ScanResult`
   * `DecisionRecord`
3. Áp decision-centric trước cho **post-deploy scans** (ZAP/Prisma) vì đây là nơi sẽ nở tool nhiều nhất.
4. Sau khi ổn mới mở rộng sang CI scans (Fortify/Checkov/Gitleaks…).

---

## Câu chốt để nói trong họp

* “Hệ hiện tại chặn deploy bằng Kyverno và ký bởi ANTT; chấp nhận dev có quyền CI nên dùng cross-check + evidence append-only để truy vết.”
* “Khi tool scan tăng, nếu làm tool-centric sẽ chết vì vận hành; decision-centric giúp chuẩn hoá evidence và ra quyết định theo policy versioned.”
* “Decision-centric chỉ triển khai nếu kèm 3 kiểm soát bắt buộc: ký decision, khóa ingestion evidence, policy immutable versioned.”

---

Nếu bạn muốn tôi làm nốt để “ban hành được ngay”, tôi sẽ viết tiếp 2 phụ lục:

1. **Schema mẫu** (YAML) cho `ScanResult` và `DecisionRecord`
2. **Policy mẫu** cho 3 gate: `sign-uat`, `promote-live`, `deploy-live`
