
---
## 📜 **TỔNG HỢP TOÀN DIỆN – KYVERNO CLUSTERPOLICY**

> Tập trung vào cấu trúc `rules` và tất cả thành phần liên quan như match, mutate, validate, v.v.

---

### I. 🧱 `rules[]` – Tâm pháp trung tâm

Một `ClusterPolicy` (hoặc `Policy`) chứa một hay nhiều `rules`. Mỗi rule có:

|Trường|Vai trò chính|
|---|---|
|`name`|Tên rule (duy nhất trong policy)|
|`match` / `exclude`|Chọn tài nguyên nào áp dụng|
|`validate`|Kiểm tra hợp lệ (nếu có)|
|`mutate`|Biến đổi cấu hình (nếu có)|
|`generate`|Sinh resource khác (nếu có)|
|`verifyImages`|Kiểm tra chữ ký image (nếu có)|
|`preconditions`|Ràng buộc logic nếu – thì|

---

### II. 🎯 `match` / `exclude` – Lọc tài nguyên áp policy

```yaml
match:
  any:
    - resources:
        kinds: ["Pod"]
        namespaces: ["dev", "prod"]
        selector:
          matchLabels:
            app: frontend
```

- `match`: Chỉ định resource **được áp dụng**
    
- `exclude`: Resource **bị loại trừ**
    
- Có thể dùng `any` / `all` để kết hợp điều kiện
    

---

### III. ✅ `validate` – Kiểm tra cấu hình

```yaml
validate:
  message: "Pod phải có resource limit"
  pattern:
    spec:
      containers:
        - resources:
            limits:
              memory: "?*"
```

- `pattern`: So khớp cấu trúc YAML với wildcard (`?*`)
    
- `anyPattern`: Một trong các pattern hợp lệ là pass
    
- Hỗ trợ cả `deny`, `foreach`, `context`, `message`
    

---

### IV. ✨ `mutate` – Sửa cấu hình tự động

```yaml
mutate:
  patchStrategicMerge:
    metadata:
      labels:
        team: devops
```

- `patchStrategicMerge`: Cách sửa mặc định (giống `kubectl patch`)
    
- `overlay`: Biến đổi với wildcard nâng cao
    
- `foreach`: Lặp qua danh sách item để mutate
    

---

### V. 🧬 `generate` – Tự tạo resource kèm theo

```yaml
generate:
  kind: NetworkPolicy
  name: default-deny
  namespace: "{{request.namespace}}"
  data:
    spec:
      ...
```

- Tự động tạo resource nếu chưa có
    
- Có thể sync với nguồn (`synchronize: true`)
    
- Hữu ích khi tạo Namespace, sinh LimitRange, RoleBinding mặc định
    

---

### VI. 🔐 `verifyImages` – Xác thực image bằng signature

```yaml
verifyImages:
  - image: "ghcr.io/org/*"
    key: |-
      -----BEGIN PUBLIC KEY-----
      ...
      -----END PUBLIC KEY-----
```

- Dùng để kiểm tra **chữ ký Cosign**, hoặc keyless (OIDC)
    
- Có thể khai báo public key inline hoặc từ URL
    

---

### VII. 📦 `variables` – Dùng biến trong policy

```yaml
{{ request.object.metadata.name }}
{{ images.containers[].name }}
```

Nguồn biến phổ biến:

- `request.object`: resource đang xét
    
- `request.userInfo`: thông tin user tạo
    
- `serviceAccount`, `namespace`, `configMap`, ...
    

---

### VIII. 🌐 `externalData` – Tra biến từ bên ngoài

```yaml
context:
  - name: userInfo
    apiCall:
      urlPath: "/userinfo?uid={{request.userInfo.username}}"
      jmesPath: "department"
```

- Truy vấn API ngoài để lấy thông tin **gắn vào context**
    
- Dùng khi cần xác thực ngoài cluster (user, RBAC...)
    

---

### IX. ♻️ `autogen` – Tự tạo rule cho controller phụ

```yaml
metadata:
  annotations:
    pod-policies.kyverno.io/autogen-controllers: none
```

- Mặc định Kyverno sẽ **tự generate** rule tương tự cho `Pod` nếu bạn viết cho `Deployment`
    
- Dùng annotation để **tắt** hoặc **tuỳ biến** controller áp dụng
    

---

### X. 🔄 `preconditions` – Điều kiện “nếu thì” trước khi thực thi rule

```yaml
preconditions:
  all:
    - key: "{{request.operation}}"
      operator: Equals
      value: CREATE
```

- Chặn rule trừ khi điều kiện đúng
    
- Dùng để chỉ validate khi tạo mới, hoặc nếu label tồn tại...
    

---

### XI. 🧠 `jmesPath` – Truy xuất nâng cao

> Cho phép viết điều kiện, truy xuất hoặc so khớp sâu trong cấu trúc YAML

```yaml
pattern:
  spec:
    containers:
      - image: "*"
        name: "*"
```

Kết hợp với `foreach` và biến context, cực kỳ mạnh mẽ trong các use-case validate phức tạp.

---

### XII. 🪛 Tips thực chiến (theo docs chính thức)

- ✅ Luôn dùng `foreach` nếu muốn áp rule cho từng container
    
- ⚠️ Cẩn thận với rule validate dạng phủ định – cần rõ ràng
    
- 🔍 Sử dụng `kyverno test` và `kyverno apply` để thử local trước khi apply
    
- 🐛 Có thể bật `audit` trước khi `enforce` để tránh gián đoạn hệ thống
    

---

## 📌 Mẫu khung chuẩn ClusterPolicy

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: enforce-best-practice
spec:
  validationFailureAction: enforce
  background: true
  rules:
    - name: require-resources
      match:
        resources:
          kinds: ["Pod"]
      validate:
        message: "Pod cần có limit"
        foreach:
          - list: "request.object.spec.containers"
            pattern:
              resources:
                limits:
                  memory: "?*"
    - name: add-label
      match:
        resources:
          kinds: ["Pod"]
      mutate:
        patchStrategicMerge:
          metadata:
            labels:
              kyverno: managed
    - name: verify-signed
      match:
        resources:
          kinds: ["Pod"]
      verifyImages:
        - image: "ghcr.io/secureorg/*"
          key: "-----BEGIN PUBLIC KEY-----..."
```

---
