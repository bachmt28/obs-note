OK. Tao viết lại **spec inputs/outputs** cho **2 gate còn lại** theo đúng mức chi tiết như Gate UAT CI Sign: rõ field, rõ “pointer vs verify”, rõ ai ghi evidence, ai query nguồn thật. Không mơ hồ nữa.

Từ đây dùng thống nhất khái niệm:

* **Evidence Store**: append-only DB (record immutable), mỗi record có `record_id`.
* **Gate Lib (ANTT Zone Only)**: shared-lib logic điều phối, gọi job ANTT, trả kết quả cho pipeline.
* **ANTT Gate Job**: Jenkins job trong folder ANTT, có quyền đọc Evidence + query tool APIs + ra quyết định (human-in-the-loop).
* **Decision Record**: append-only record do ANTT job ghi, gắn với một gate.

---

# A) Gate UAT CD – Promote Decision

**(Gate UAT CD Lib – ANTT Zone Only + Jenkins ANTT Gate UAT CD Job)**

## A1) Mục tiêu gate

Trả lời duy nhất:

* **ALLOW_PROMOTE** hoặc **DENY_PROMOTE** cho **image digest đang chạy trên UAT** (runtime truth), dựa trên:

  * pre-deploy scans: Gitleaks + Checkov
  * post-deploy scan: ZAP
  * (option) Kyverno admission allow evidence
  * human judgement của ANTT

---

## A2) Evidence record (UAT CD) – REQUIRED FIELDS

> UAT CD job (AM-owned) ghi **1 record append-only** sau khi chạy xong predeploy+deploy+zap.

### A2.1 Identity pointers

```yaml
uat_cd_evidence:
  record_id: <uuid>

  cd:
    job_name: <UAT_CD_JOB_NAME>
    build_number: <UAT_CD_BUILD_NUMBER>
    build_url: <UAT_CD_BUILD_URL>

  scm:
    deploy_repo_url: <GIT_DEPLOY_REPO_URL>
    deploy_branch: <DEPLOY_BRANCH>
    deploy_commit: <DEPLOY_COMMIT_SHA>
```

### A2.2 Runtime binding (SOURCE OF TRUTH from K8s)

```yaml
  runtime:
    cluster: <UAT_CLUSTER_ID>
    namespace: <UAT_NAMESPACE>
    workload_kind: Deployment
    workload_name: <APP_NAME>
    runtime_image_digest: sha256:<...>
```

### A2.3 Pre-deploy scan pointers (Gitleaks + Checkov)

```yaml
  predeploy_scans:
    gitleaks:
      scan_ref: <GITLEAKS_REF>
      report_ref: <ARTIFACT_OR_STORAGE_REF>
      report_sha256: <sha256>
    checkov:
      scan_ref: <CHECKOV_REF>
      report_ref: <ARTIFACT_OR_STORAGE_REF>
      report_sha256: <sha256>
```

### A2.4 Post-deploy scan pointers (ZAP)

```yaml
  postdeploy_scans:
    zap:
      scan_ref: <ZAP_SCAN_REF>
      targets_ref: <GIT_PATH_OR_ARTIFACT_REF_TO_URL_LIST>
      report_ref: <ARTIFACT_OR_STORAGE_REF>
      report_sha256: <sha256>
```

### A2.5 Timing (anti-fake)

```yaml
  timing:
    deploy_applied_at: <timestamp>
    deploy_ready_at: <timestamp>
    zap_scan_start: <timestamp>
    zap_scan_end: <timestamp>
```

### A2.6 Metadata (audit)

```yaml
  meta:
    created_at: <timestamp>
    created_by_job: <job_name>
    created_by_build: <build_number>
```

**Ghi chú bắt buộc về tính “đủ chặt”:**

* `runtime.runtime_image_digest` phải được lấy từ **K8s UAT** sau khi deploy Ready, không lấy từ tag.
* `report_sha256` là hash file report để chống sửa report.

---

## A3) Gate UAT CD Lib – INPUTS

> UAT CD pipeline gọi lib ANTT zone.

**Inputs**

```yaml
uat_cd_evidence_record_id: <uuid>
```

---

## A4) Jenkins ANTT Gate UAT CD Job – VERIFICATION & REVIEW FLOW

ANTT Gate job nhận:

```yaml
uat_cd_evidence_record_id
```

Sau đó bắt buộc làm:

### Step 1 — Load UAT CD evidence record

* đọc toàn bộ pointers.

### Step 2 — Verify runtime digest (SOURCE OF TRUTH)

Query K8s UAT (API) theo `cluster/namespace/workload`:

* lấy digest thực tế đang chạy `DIGEST_RUNTIME_K8S`

Check:

```
DIGEST_RUNTIME_K8S == runtime_image_digest (trong record)
```

Mismatch → **DENY_PROMOTE** (evidence khai sai/ngữ cảnh sai)

### Step 3 — Verify ZAP timing sanity

Check:

```
deploy_ready_at <= zap_scan_start <= zap_scan_end
```

Fail → **DENY_PROMOTE**

### Step 4 — Verify scan refs are real (SOURCE OF TRUTH)

* Với `gitleaks.scan_ref`:

  * truy ra Jenkins step log / artifact tồn tại
  * (option) verify commit scanned == deploy_commit
* Với `checkov.scan_ref`:

  * tương tự
* Với `zap.scan_ref`:

  * verify report tồn tại, hash khớp

Nếu report không tồn tại / hash mismatch → **DENY_PROMOTE**

### Step 5 — Human review (ANTT)

ANTT đọc:

* ZAP report (runtime risk)
* Checkov findings (IaC posture)
* Gitleaks findings (secrets in deploy repo/values)

ANTT quyết:

* allow promote
* deny promote
* allow with exception note (vẫn allow, nhưng phải ghi justification)

### Step 6 — Append Decision Record (append-only)

ANTT Gate job ghi record quyết định.

---

## A5) Outputs

### A5.1 Runtime outputs (trả về cho UAT CD pipeline)

```yaml
allowed: true | false
decision_record_id: <uuid>
reason: <text>
```

### A5.2 Decision Record (append-only) – REQUIRED FIELDS

```yaml
uat_cd_decision:
  decision_record_id: <uuid>
  gate: uat_cd_promote
  policy_version: <optional_string>

  subject:
    runtime_image_digest: sha256:<...>
    namespace: <...>
    workload: <...>
    deploy_commit: <...>

  evidence_links:
    uat_cd_evidence_record_id: <uuid>

  verdict: ALLOW_PROMOTE | DENY_PROMOTE
  decision_mode: human
  approver: <antt_user_id>
  justification: <text>
  decided_at: <timestamp>
```

---

# B) Gate LIVE CD – Deploy Decision

**(Gate LIVE CD Lib – ANTT Zone Only + Jenkins ANTT Gate LIVE CD Job)**

## B1) Mục tiêu gate

Trả lời:

* **ALLOW_DEPLOY** hoặc **DENY_DEPLOY** cho LIVE deployment, dựa trên:

  * evidence rằng image đã **promote + sign live** hợp lệ
  * pre-deploy scans trên repo/values LIVE: Gitleaks + Checkov
  * (option) verify Kyverno will enforce (policy presence)
  * human judgement (nếu conditional)

**Kyverno LIVE vẫn là hard enforcement lúc apply**; Gate này là “go/no-go” trước khi apply.

---

## B2) Evidence record (LIVE Promote+Sign) – REQUIRED FIELDS

> Jenkins job Promote (ANTT-owned) ghi append-only.

```yaml
live_promote_sign_evidence:
  record_id: <uuid>

  promote:
    job_name: <PROMOTE_JOB_NAME>
    build_number: <PROMOTE_BUILD_NUMBER>
    build_url: <PROMOTE_BUILD_URL>

  source:
    uat_digest: sha256:<...>
    uat_repo: <...>

  destination:
    live_repo: <...>
    live_digest: sha256:<...>

  signing:
    signature_ref: <cosign_ref>
    signer_identity: <key_id_or_cert_subject>

  meta:
    created_at: <timestamp>
```

---

## B3) Evidence record (LIVE CD predeploy) – REQUIRED FIELDS

> Live CD job (AM-owned) ghi append-only **trước deploy**.

```yaml
live_cd_predeploy_evidence:
  record_id: <uuid>

  cd:
    job_name: <LIVE_CD_JOB_NAME>
    build_number: <LIVE_CD_BUILD_NUMBER>
    build_url: <LIVE_CD_BUILD_URL>

  scm:
    deploy_repo_url: <LIVE_DEPLOY_REPO_URL>
    deploy_branch: <LIVE_DEPLOY_BRANCH>
    deploy_commit: <LIVE_DEPLOY_COMMIT_SHA>

  predeploy_scans:
    gitleaks:
      scan_ref: <GITLEAKS_REF>
      report_ref: <...>
      report_sha256: <sha256>
    checkov:
      scan_ref: <CHECKOV_REF>
      report_ref: <...>
      report_sha256: <sha256>

  deploy_target:
    cluster: <LIVE_CLUSTER_ID>
    namespace: <LIVE_NAMESPACE>
    workload_kind: Deployment
    workload_name: <APP_NAME>
    intended_image_ref: <LIVE_REPO:TAG or DIGEST_POINTER>

  meta:
    created_at: <timestamp>
```

**Ghi chú:** `intended_image_ref` có thể là tag, nhưng gate phải resolve digest thật ở bước verify.

---

## B4) Gate LIVE CD Lib – INPUTS

Live CD pipeline gọi lib ANTT zone với 2 record id:

```yaml
live_promote_sign_evidence_record_id: <uuid>
live_cd_predeploy_evidence_record_id: <uuid>
```

---

## B5) Jenkins ANTT Gate LIVE CD Job – VERIFICATION & REVIEW FLOW

ANTT Gate job nhận 2 record id, bắt buộc làm:

### Step 1 — Load both evidence records

* read pointers.

### Step 2 — Verify Live digest exists (SOURCE OF TRUTH: Nexus LIVE)

Query Nexus LIVE:

* resolve digest existence & metadata for `live_digest`
* verify signature object exists for digest (cosign ref or registry signature storage)

Nếu digest không tồn tại → **DENY_DEPLOY**

### Step 3 — Verify signature validity (SOURCE OF TRUTH: Cosign verify against registry)

* cosign verify digest với expected identity/key
* check signature_ref matches (nếu dùng)

Fail → **DENY_DEPLOY**

### Step 4 — Verify predeploy scans refs are real (SOURCE OF TRUTH)

* verify gitleaks/checkov report tồn tại, hash khớp
* (option) verify they scanned correct deploy_commit

Fail → **DENY_DEPLOY**

### Step 5 — Human review (ANTT)

* review checkov/gitleaks findings
* decide allow/deny

### Step 6 — Append Decision Record

* ghi quyết định allow/deny

---

## B6) Outputs

### B6.1 Runtime outputs (trả về Live CD pipeline)

```yaml
allowed: true | false
decision_record_id: <uuid>
reason: <text>
```

### B6.2 Decision Record (append-only) – REQUIRED FIELDS

```yaml
live_cd_decision:
  decision_record_id: <uuid>
  gate: live_cd_deploy
  policy_version: <optional_string>

  subject:
    live_digest: sha256:<...>
    live_repo: <...>
    live_deploy_commit: <...>
    target_namespace: <...>
    target_workload: <...>

  evidence_links:
    live_promote_sign_evidence_record_id: <uuid>
    live_cd_predeploy_evidence_record_id: <uuid>

  verdict: ALLOW_DEPLOY | DENY_DEPLOY
  decision_mode: human
  approver: <antt_user_id>
  justification: <text>
  decided_at: <timestamp>
```

---

# C) Kyverno audit refs sau deploy (LIVE) – bổ sung rõ

Kyverno audit refs **không phải output của Gate job**, mà là **post-apply evidence** do Live CD pipeline append sau deploy.

## LIVE CD Post-deploy evidence record (optional but recommended)

```yaml
live_cd_postdeploy_evidence:
  record_id: <uuid>
  relates_to_decision_record_id: <uuid>

  kyverno:
    admission_result: ALLOW
    policy_refs:
      - <policy_name_version>
    audit_log_ref: <log_ref_or_query_id>

  deploy:
    applied_at: <timestamp>
    rollout_status: SUCCESS | FAIL
```

---

## Chốt lại đúng ý m

* Mỗi gate **theo phase**.
* Mỗi phase **1 evidence record** do pipeline ghi (append-only).
* Gate job của ANTT **verify bằng cách query source of truth**, không tin “verdict trong evidence”.
* Output là **decision record id + allow/deny + reason**.

Nếu m muốn, tao sẽ cập nhật luôn 2 sequenceDiagram UAT CD và LIVE CD để phản ánh đúng các record id + verify steps như spec này (không còn entity mơ hồ).
