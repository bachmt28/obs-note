---

## 🧠 **Danh sách & phân tích từng Vault Credential Type**

---

### 🔹 **Vault AWS IAM Credential**

- 🧭 **Dùng để**: Xác thực Vault qua AWS IAM (`auth/aws/login`)
    
- 🧰 **Dùng khi**: Jenkins đang chạy trong AWS (EC2/instance profile) hoặc có IAM role
    
- 🔐 Vault sẽ kiểm tra chữ ký AWS request → xác minh identity
    
- ⚠️ Hiếm dùng trong Jenkins vì setup khá phức
    

---

### 🔹 **Vault App Role Credential** ✅ **Phổ biến nhất**

- 🧭 **Dùng để**: Xác thực bằng AppRole (`auth/approle/login`)
    
- 📦 Cần `role_id` + `secret_id`
    
- 🛡 Rất tốt để phân quyền từng Jenkins job
    
- ✅ Best practice trong CI/CD Vault integration
    
- 📑 Phù hợp khi Jenkins chạy **ngoài Kubernetes**
    

---

### 🔹 **Vault Certificate Credential**

- 🧭 **Dùng để**: Xác thực bằng TLS client cert (`auth/cert/login`)
    
- 🧰 Dùng được khi Vault có mount `auth/cert`
    
- 🔐 Cần có cert + key đúng CN + CA
    
- ⚠️ Hiếm dùng trong Jenkins thực tế trừ khi bạn có CA nội bộ riêng
    

---

### 🔹 **Vault GCP Credential**

- 🧭 **Dùng để**: Xác thực bằng GCP Identity (`auth/gcp/login`)
    
- ☁️ Jenkins phải chạy trên GCP hoặc có service account JSON
    
- 📦 Plugin sẽ gửi JWT của GCP lên Vault
    
- ⚠️ Chuyên dụng GCP, hiếm thấy
    

---

### 🔹 **Vault Github Token Credential**

- 🧭 **Dùng để**: Login Vault bằng GitHub personal access token (`auth/github/login`)
    
- ⚠️ Deprecated trong nhiều tổ chức vì hạn chế audit/token control
    
- ❌ Không phù hợp production Jenkins
    

---

### 🔹 **Vault Google Container Registry Login**

- 🧭 Gắn liền với GCP Vault Auth + Docker
    
- 🎯 Rất hiếm dùng trong Jenkins (đa phần xài GCR token riêng)
    

---

### 🔹 **Vault Kubernetes Credential** ✅ **Tối ưu cho Jenkins chạy trong pod**

- 🧭 **Dùng để**: Xác thực bằng JWT từ service account (`auth/kubernetes/login`)
    
- 🐳 Jenkins phải chạy trong Kubernetes
    
- 📦 Plugin sẽ tự lấy JWT file từ pod `/var/run/secrets/...`
    
- ✅ Best practice khi Jenkins chạy bằng Helm chart hoặc ArgoCD
    
- 🔐 Vault validate JWT + namespace + SA name
    

---

### 🔹 **Vault SSH Username with private key Credential**

- 🧭 Dùng để login Vault thông qua SSH (?!)
    
- ⚠️ Rất hiếm dùng – chỉ khi `auth/ssh` được bật và bạn có CA SSH
    

---

### 🔹 **Vault Secret File Credential**

- 📁 Cho phép bạn lấy **file từ Vault** (ví dụ: kubeconfig, pem cert, ...)
    
- ✨ Jenkins sẽ mount file này tạm trong build
    
- ✅ Hữu dụng khi secret là file (not just string)
    

---

### 🔹 **Vault Secret Text Credential**

- 📜 Lấy 1 chuỗi đơn từ Vault → dùng như `env var`
    
- Ví dụ: DB password, Git token, API key
    
- ✅ Phổ biến nhất cho secrets dạng chuỗi đơn giản
    

---

### 🔹 **Vault Token Credential**

- 🧭 Dùng để truyền thẳng token Vault vào Jenkins (token tĩnh)
    
- ⚠️ Rủi ro cao nếu bị lộ token
    
- ❌ Không nên dùng trong prod trừ khi cực kỳ kiểm soát TTL/token
    

---

### 🔹 **Vault Token File Credential**

- 🧭 Giống trên nhưng token lưu dưới dạng file
    
- 📄 Dùng khi bạn mount token qua secret volume / file injection
    

---

### 🔹 **Vault Username-Password Credential**

- 🧭 Dùng cho `auth/userpass` (login bằng username + password)
    
- ⚠️ Thường dành cho con người, không phù hợp CI/CD
    
- ❌ Tránh dùng nếu có thể
    

---

## ✅ Gợi ý lựa chọn theo môi trường Jenkins

|Jenkins chạy ở đâu?|Nên chọn loại Vault Credential|
|---|---|
|**Bare-metal / VM / Docker**|`Vault App Role Credential` ✅|
|**Kubernetes / OpenShift**|`Vault Kubernetes Credential` ✅|
|**Test lab, nhanh gọn**|`Vault Token Credential` (cẩn thận TTL)|
|**Muốn lấy secret dạng file**|`Vault Secret File Credential`|
|**Lấy 1 chuỗi đơn giản**|`Vault Secret Text Credential`|

---

## 📌 Ví dụ dùng AppRole

1. Bên Vault tạo role:
    

```bash
vault write auth/approle/role/jenkins-role \
  token_policies="jenkins-policy" \
  token_ttl=30m \
  token_max_ttl=1h
```

2. Lấy `role_id` và `secret_id`, đưa vào Jenkins credential kiểu `Vault App Role Credential`
    

---
