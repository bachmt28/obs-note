
---

## 🧱 Các loại Policy trong Kyverno

Kyverno cung cấp **nhiều loại chính sách (policy)**, mỗi loại phục vụ một mục tiêu cụ thể trong hệ sinh thái Kubernetes. Dưới đây là toàn bộ các loại policy được chính thức hỗ trợ:

---

### 🟩 1. `ClusterPolicy` vs `Policy`

|Loại policy|Áp dụng lên namespace?|Phạm vi hoạt động|Dùng khi nào?|
|---|---|---|---|
|`Policy`|✅ Có – namespace scoped|Chỉ trong namespace đó|Rule riêng cho từng team|
|`ClusterPolicy`|❌ Không|Áp dụng toàn cluster|Chính sách chung, bắt buộc|

📌 **Cấu trúc giống nhau**, chỉ khác ở **scope**.

Ví dụ:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy  # hoặc Policy
metadata:
  name: force-limits
spec:
  rules:
    ...
```

---

### 🟦 2. `CleanupPolicy` – Chính sách dọn rác tài nguyên

> Tự động xoá các tài nguyên không còn cần thiết (TTL-based GC)

📌 Áp dụng cho:

- CronJobs cũ
    
- PVC không còn dùng
    
- Resources có label/annotation đặc biệt
    

Ví dụ:

```yaml
apiVersion: kyverno.io/v2alpha1
kind: CleanupPolicy
metadata:
  name: cleanup-old-jobs
spec:
  match:
    any:
      - resources:
          kinds: ["Job"]
  conditions:
    all:
      - key: "{{request.object.status.completionTime}}"
        operator: GreaterThanOrEquals
        value: "{{ time_now_minus_1d }}"
```

⏱️ Hỗ trợ `time_now_minus_X` để tính TTL tự động.

📎 Tài liệu chi tiết: [Cleanup Policy](https://kyverno.io/docs/policy-types/cleanup-policy/)

---

### 🟥 3. `ValidatingAdmissionPolicy` – Tương tự OPA Gatekeeper

> Chính sách xác thực resource trước khi được apply

📌 Có thể dùng để:

- Chặn Pod chạy image `latest`
    
- Bắt buộc có `resource limit`
    
- Không cho tạo `hostPath`
    

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: deny-latest-tag
spec:
  validationFailureAction: enforce
  rules:
    - name: check-tag
      match:
        resources:
          kinds: ["Pod"]
      validate:
        message: "Image tag latest không được phép"
        pattern:
          spec:
            containers:
              - image: "!*:latest"
```

📎 Tài liệu chi tiết: [Validating Policy](https://kyverno.io/docs/policy-types/validating-policy/)

---

### 🟨 4. `ImageVerify` – Xác thực image bằng signature

> Đảm bảo container image đến từ nguồn tin cậy (cosign, keyless, sigstore...)

📌 Cấu hình:

- Sử dụng public key hoặc ký hiệu chứng thực
    
- Có thể tích hợp với **Cosign**, **Rekor**, **keyless sig**
    

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-signed-image
spec:
  validationFailureAction: enforce
  rules:
    - name: verify-signature
      match:
        resources:
          kinds: ["Pod"]
      verifyImages:
        - image: "ghcr.io/myorg/*"
          key: |
            -----BEGIN PUBLIC KEY-----
            ...
            -----END PUBLIC KEY-----
```

📎 Tài liệu chi tiết: [Image Validating Policy](https://kyverno.io/docs/policy-types/image-validating-policy/)

---

### 🔁 Tổng hợp nhanh

|Policy Type|Mục đích chính|Đặc điểm nổi bật|
|---|---|---|
|`Policy`|Áp dụng scoped namespace|Cho team cụ thể|
|`ClusterPolicy`|Áp dụng toàn cluster|Best practice cho mọi cluster|
|`CleanupPolicy`|Dọn tài nguyên không dùng|TTL, tự động xoá|
|`Validating Policy`|Kiểm tra cấu hình hợp lệ trước khi apply|Bắt lỗi config, enforce rule|
|`ImageVerify`|Kiểm tra chữ ký container image|Ngăn deploy image không tin cậy|

---
