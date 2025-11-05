
---
## 🧩 Kyverno là gì?

**Kyverno** là một **policy engine dành riêng cho Kubernetes**, được thiết kế để:

- **Quản lý cấu hình tài nguyên** bằng YAML, không cần học Rego như OPA
    
- Tích hợp sâu vào **luồng admission của Kubernetes**
    
- Hỗ trợ cả **mutate – validate – generate**
    
- Phù hợp cho cả **DevOps lẫn Platform teams**
    

---

## 🎯 Mục tiêu của Kyverno

| Mục tiêu                  | Diễn giải dễ hiểu                                 |
| ------------------------- | ------------------------------------------------- |
| 🧠 **Đơn giản**           | Viết policy bằng YAML, không cần ngôn ngữ riêng   |
| 📦 **Tích hợp sâu**       | Là 1 admission controller chạy ngay trong cluster |
| 🛠️ **Hành vi giống K8s** | Policy áp dụng như cách K8s xử lý tài nguyên      |
| 🚀 **Tự động hoá policy** | Có thể generate các object như RBAC, ConfigMap... |
| 🧪 **Có CLI để test**     | Dễ thử nghiệm local trước khi deploy              |

---

## 🛠️ Kyverno có thể làm gì?

### 1. ✅ Validate – Kiểm tra tài nguyên có đúng chuẩn không?

Ví dụ:

- Pod phải có label `app`
    
- Deployment phải có resource limits
    
- Không cho tạo Service type `NodePort` trong môi trường production
    

### 2. ✨ Mutate – Tự động chỉnh sửa tài nguyên cho đúng chuẩn

Ví dụ:

- Thêm label `team: devops` vào tất cả Pod
    
- Set default `imagePullPolicy: IfNotPresent`
    
- Force annotation “managed-by: kyverno”
    

### 3. 🧬 Generate – Sinh tài nguyên phụ trợ khi tạo resource chính

Ví dụ:

- Khi tạo Namespace mới, tự sinh:
    
    - NetworkPolicy
        
    - RoleBinding
        
    - LimitRange
        

---

## 🧱 Kiến trúc hoạt động

Kyverno cài trong cluster và hoạt động theo kiểu:

- Webhook chặn request tới API Server
    
- Policy được định nghĩa trong YAML
    
- Có thể **enforce**, **audit**, hoặc **mutate inline**
    

Nó cũng hỗ trợ **background scanning** để kiểm tra các resource đã tồn tại (không chỉ mới tạo).

---

## 🤝 Kyverno & GitOps

Kyverno rất hợp với GitOps vì:

- Policy là YAML → dễ version control
    
- Có thể apply policy vào cả các manifest nằm trong repo
    
- Hỗ trợ CLI (`kyverno apply`) để test trong CI pipeline
    

---

## 🔐 DevSecOps & Compliance

Kyverno giúp enforce các **security control**, ví dụ:

|Policy|Mục đích|
|---|---|
|Cấm container chạy với root|Bảo vệ security|
|Bắt buộc có resource limit|Ngăn overcommit|
|Gắn annotation để tracking|Hỗ trợ audit/compliance|

---

## 📚 Tài liệu & công cụ

- 🌐 Trang chủ: [https://kyverno.io](https://kyverno.io/)
    
- 🧪 Try online: [Playground](https://kyverno.io/playground/)
    
- 🛠️ GitHub repo: [https://github.com/kyverno/kyverno](https://github.com/kyverno/kyverno)
    
- 📦 Helm Chart: `kyverno/kyverno`
    
- 📄 Policy Library: [https://kyverno.io/policies](https://kyverno.io/policies)
    

---

## 🧠 Tổng kết

| Tính năng           | Kyverno có? | Ghi chú thêm |
| ------------------- | ----------- | ------------ |
| ✅ Validate Policy   | ✔️          | YAML based   |
| ✅ Mutate Resource   | ✔️          | auto patch   |
| ✅ Generate Resource | ✔️          | event-driven |
| ✅ CLI Testing       | ✔️          | Dev friendly |
| ✅ GitOps Compatible | ✔️          | YAML native  |
| ✅ Background Scan   | ✔️          | Compliance   |

---