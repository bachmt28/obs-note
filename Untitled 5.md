# 🧪 Phần 1 – Troubleshooting (rất hay ra thi)

### Tình huống 1

Một Pod stuck ở trạng thái `Pending`.

`kubectl describe pod` báo:

```
0/3 nodes are available: 
3 Insufficient memory.
```

Câu hỏi:

1. Ngài sẽ kiểm tra gì đầu tiên?
2. Nếu cluster còn tài nguyên nhưng pod vẫn không schedule được, khả năng nguyên nhân là gì?
3. Cách xử lý thực tế?

Trả lời:
1. Kiểm tra tài nguyên của các node được chỉ định để chạy workload, cụ thể là memory, xem available để được assign có đủ không, mem request của các workload khác 
2. Nếu còn tài nguyên thì nguyên nhân tiếp theo có thể dẫn đến là workload đang config sai, node affinity, nên đc assign vào node pool khác không có đủ resource
3. xử lý thực tế, hoặc scale resource, hoặc hiệu chỉnh config của workload 

---

# 🧪 Phần 2 – Networking

### Tình huống 2

Có 2 namespace:

* frontend
* backend

Yêu cầu:

* frontend được gọi backend
* backend không được gọi frontend
* các namespace khác không được gọi backend

Câu hỏi:

Ngài sẽ triển khai NetworkPolicy như thế nào về mặt logic?

Không cần viết YAML hoàn chỉnh, chỉ cần nói rõ:

* Selectors
* Ingress/Egress rule
* Default deny có cần không?

Trả lời:

hiện hệ thống bên t đang áp dụng rule đối với nội bộ thì không tạo fw, vm ko bật fw, nên chưa có áp dụng gì cả 
T đang hiểu network policies của k8s nó tương đương với firewall như trên vm 
Nói chung phần này t chưa hề làm nhiều hay đụng tới, chỉ hiểu thôi 

Ở góc độ bảo mật, t mới thấy 1 case là có thể cần rule deny tới các namespace về mặt hệ thống như kube-system

---

# 🧪 Phần 3 – RBAC

### Tình huống 3

Một service account cần:

* list pod
* get pod
* không được delete
* chỉ trong namespace `prod`

Ngài sẽ tạo những object nào?

(Role? ClusterRole? Binding gì?)


Trả lời:
- Tạo role pod-list-get , resources trỏ vào pods, verbs get watch list, gán namespace prod cụ thể 
- chỉ 1 namespace thì ko cần cluster role
- binding role đã tạo vào 1 service account cụ thể

---

# 🧪 Phần 4 – etcd

### Tình huống 4

Control plane node bị chết.

Cluster có 1 control plane duy nhất.

Câu hỏi:

1. Điều gì sẽ xảy ra?
2. Worker node có chạy workload không?
3. Cách chuẩn bài để backup etcd trước đó là gì?


Trả lời:
1. chỉ 1 node control plane, không có ha tức là kubeapi server sẽ không thể tương tác được nữa , Scheduler & Controller Manager cũng sẽ không hoạt động, khi có yêu cầu mới về lập lịch vào theo dõi trạng thái sẽ không đáp ứng được 
2. workload nào đang chạy rồi thì vẫn chạy sẽ ko bị ảnh hưởng, nhưng nếu chết và muốn schedule lại thì sẽ không được 
3. sử dụng snapshot để backup etcd định kỳ, lưu trữ ra 1 nơi khác như s3, nfs


---

# 🧪 Phần 5 – Storage

### Tình huống 5

Một Pod cần mount PVC nhưng báo lỗi:

```
pod has unbound immediate PersistentVolumeClaims
```

Ngài sẽ kiểm tra theo thứ tự nào?

Trả lời: 
- describe cái pvc đang gặp vấn đề , để tìm kiếm các sự kiện liên quan đến pvc đó
- kiểm tra thông số cấu hình của cả pvc và pv xem có matching về class, access mode không 


---

# 🧪 Phần 6 – Câu hỏi đánh giá tư duy

CKA thi cho phép tra tài liệu chính thức.

Ngài có thường:

* Bookmark đúng doc?
* Biết search nhanh trong docs?
* Thuộc syntax cơ bản của kubectl?

Vì trong thi, tốc độ gõ và search quan trọng hơn “biết sâu”.

---

# 🎯 Cách trả lời

Ngài trả lời từng phần ngắn gọn theo cách thực tế mình làm.

Ta sẽ:

* Chấm mức độ 0–100%
* Chỉ ra lỗ hổng
* Ước lượng thời gian cần ôn để pass CKA

---

Không cần màu mè.

Cứ trả lời như đang ngồi trước terminal.

Ta đang test không phải kiến thức Google,
mà test phản xạ vận hành của ngài.
