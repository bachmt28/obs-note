# Gate UAT CI – Sign
**(Gate UAT CI Lib – ANTT Zone Only + Jenkins ANTT Sign UAT Job)**
## 1) Nguyên tắc cốt lõi (khóa ngay từ đầu)
* **Dev job chỉ gửi `evidence_record_id`**
* Evidence record là **append-only**
* Evidence record **chỉ chứa POINTER (ID/ref)**, **không chứa “sự thật”**
* **ANTT Sign job KHÔNG TIN evidence**, chỉ dùng để:
  * biết **phải query cái gì**
  * ghi lại **kết quả verify + ký**
---
## 2) Evidence record (UAT CI) – REQUIRED FIELDS

> Evidence này được **Dev UAT CI job ghi 1 lần**, không sửa được.
### 2.1 Identity & lineage pointers

```yaml
evidence_record_id: <uuid>

ci:
  job_name: <CI_JOB_NAME>
  build_number: <CI_BUILD_NUMBER>

image:
  uat_repo: <UAT_DOCKER_REPO>
  uat_image_tag: <UAT_IMAGE_TAG>

scm:
  repo_url: <GIT_REPO_URL>
  branch: <GIT_BRANCH>
```
---
### 2.2 Scan pointers (BẮT BUỘC)
```yaml
scans:
  fortify:
    scan_job_id: <FORTIFY_SCAN_JOB_ID>

  gitleaks:
    scan_ref: <GITLEAKS_SCAN_REF>
```
**Giải thích `gitleaks.scan_ref`:**
* Có thể là:
  * Jenkins build ID của step gitleaks
  * artifact path
  * hoặc report hash
* **Chỉ cần đủ để ANTT Sign job truy ngược ra report/log gốc**
* **Không ghi verdict vào đây**
---
### 2.3 Metadata kỹ thuật (phục vụ audit, KHÔNG dùng để verify)
```yaml
meta:
  created_at: <timestamp>
  created_by_job: <job_name>
  created_by_build: <build_number>
```
---
## 3) Gate UAT CI Lib – INPUTS

> Đây là **shared lib thuộc ANTT zone**, được gọi trong Dev UAT CI job.
### Inputs
```yaml
evidence_record_id: <uuid>
```
❌ **Không truyền**:
* commit sha
* digest
* verdict
* scan result
---
## 4) Jenkins ANTT Sign UAT Job – VERIFICATION FLOW
ANTT Sign job nhận **chỉ**:
```yaml
evidence_record_id
```
Sau đó tự làm **toàn bộ xác thực độc lập**.

---

## 5) Jenkins ANTT Sign UAT Job – VERIFICATION STEPS

### Step 1 – Load evidence record

* Đọc các pointer fields:

  * `ci.job_name`
  * `ci.build_number`
  * `image.uat_repo`
  * `image.uat_image_tag`
  * `scans.fortify.scan_job_id`
  * `scans.gitleaks.scan_ref`

---

### Step 2 – Verify Jenkins build (SOURCE OF TRUTH)

Query Jenkins API:

* job_name + build_number
* Lấy:

  * **commit SHA thực tế**
  * SCM revision
  * build trigger info

→ `COMMIT_SHA_JENKINS`

---

### Step 3 – Verify Fortify binding (SOURCE OF TRUTH)

Query Fortify API bằng `scan_job_id`:

* Lấy:

  * commit / revision mà Fortify scan
  * scan verdict

→ `COMMIT_SHA_FORTIFY`

Check:

```
COMMIT_SHA_FORTIFY == COMMIT_SHA_JENKINS
```

Nếu fail → **DENY**

---

### Step 4 – Verify Gitleaks binding (SOURCE OF TRUTH)

Từ `gitleaks.scan_ref`:

* Truy ra:

  * Jenkins step log / artifact
  * commit SHA được scan
  * scan result

Check:

```
COMMIT_SHA_GITLEAKS == COMMIT_SHA_JENKINS
```

Nếu:

* phát hiện secret thật → **DENY**
* false positive đã được mark (nếu có cơ chế) → OK

---

### Step 5 – Resolve image digest (SOURCE OF TRUTH)

Query Nexus UAT:

* resolve `uat_repo:uat_image_tag`

→ `DIGEST_NEXUS`

Nếu:

* tag không tồn tại
* resolve fail

→ **DENY**

---

### Step 6 – (Khuyến nghị mạnh) Verify image lineage

* Pull image manifest/config theo `DIGEST_NEXUS`
* Đọc labels:

  * commit
  * jenkins job
  * build number

Check:

```
image.commit == COMMIT_SHA_JENKINS
```

Mismatch → **DENY**

---

### Step 7 – Sign

* Cosign sign **DIGEST_NEXUS**
* Push signature

---

## 6) Jenkins ANTT Sign UAT Job – OUTPUTS

### Runtime outputs (return về Gate Lib)

```yaml
result: ALLOW | DENY
signed_digest: sha256:...
signature_ref: <cosign signature ref>
```

---

### Evidence record appended by ANTT Sign job

```yaml
sign_result:
  verdict: ALLOW | DENY
  signed_digest: sha256:...
  signature_ref: <ref>

verification:
  jenkins_commit: <sha>
  fortify_commit: <sha>
  gitleaks_commit: <sha>

verified_at: <timestamp>
verified_by_job: <ANTT_SIGN_JOB_NAME>
verified_by_build: <BUILD_NUMBER>
```

> Record này **append-only**, là **nguồn audit tin cậy**.

---

## 7) Gate UAT CI Lib – OUTPUTS (cho Dev job)

```yaml
allowed: true | false
reason: <text>
signed_digest: <sha256> (if allowed)
signature_ref: <ref> (if allowed)
```

---

## 8) Một câu khóa cứng để đưa vào spec

> **Evidence record chỉ chứa pointer.
> Jenkins ANTT Sign job chỉ tin dữ liệu truy vấn trực tiếp từ Jenkins, Fortify, Gitleaks và Nexus.**

---

## 9) Trạng thái hiện tại

* ✅ Spec **đã đủ field**
* ✅ Gitleaks **đã được đưa vào đúng vai**
* ✅ Không còn “tin lời khai”
* ✅ Code theo được 1–1

Nếu m muốn bước tiếp theo:

* tao có thể viết **schema SQL / JSON cho evidence append-only**
* hoặc **pseudo-code Jenkins ANTT Sign UAT Job**
* hoặc **checklist verify fail cases**

Nói thẳng cái m cần, tao làm đúng cái đó.
