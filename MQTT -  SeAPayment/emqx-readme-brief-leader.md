

---

## 2. Lý do 

Trước đây, đối tác đề xuất triển khai hệ thống EMQX theo mô hình **truyền thống sử dụng máy ảo (VM)**:
3–5 node EMQX chạy độc lập, kết nối qua 2 node HAProxy (dùng Keepalived) để cân bằng tải.

Tuy nhiên, sau quá trình đánh giá và thử nghiệm,  quyết định **tái thiết kế toàn bộ kiến trúc** – chuyển đổi sang **nền tảng Kubernetes**, giảm thiểu lỗi vận hành, tăng khả năng mở rộng và khôi phục.

---

### ✅ Lợi ích khi chuyển sang Kubernetes

| Nội dung                       | Lý do chọn triển khai trên K8s                                   |
| ------------------------------ | ---------------------------------------------------------------- |
| **Duy trì & cập nhật ổn định** | Rolling update không ngắt kết nối, dễ rollback nếu gặp lỗi       |
| **Tách biệt workload rõ ràng** | EMQX, HAProxy, Nginx vận hành độc lập, dễ theo dõi & giám sát    |
| **Khả năng mở rộng linh hoạt** | HPA, autoscaling, resource control native của Kubernetes         |
| **Sticky session sẵn sàng**    | Sử dụng `balance source` tại HAProxy (dù chưa bắt buộc hiện tại) |
| **Backup dễ kiểm soát hơn**    | PVC gắn từng node EMQX, có thể rsync ra ngoài nếu cần            |

---

### ⚠️ Những vấn đề cốt lõi cần giải quyết

1. **EMQX không có cơ chế chia tải nội bộ**
   → Tất cả node đều đồng cấp, không có Leader/Router riêng → **bắt buộc phải có Load Balancer ngoài**.
   → TCP-based protocol như MQTT thì **HAProxy hiệu quả hơn Nginx**.

2. **HAProxy phải đặt *trong cluster K8s***
   → Nếu đặt trên VM ngoài, **sẽ không thể truy cập headless service** – là cơ chế DNS discovery chính của EMQX.
   → Việc thiết lập **2 workload HAProxy (active/passive)** ngay trong namespace EMQX đảm bảo:

   * Giao tiếp nội bộ ổn định
   * Tận dụng probe, autoscaling, logging của K8s
   * Sẵn sàng nhận TLS passthrough hoặc termination theo yêu cầu

3. **Sử dụng bản EMQX Open Source – giới hạn đặc thù**

   * Lưu dữ liệu (ACL, user, retained) bằng **Mnesia** nội bộ → **phụ thuộc vào PVC của từng Pod**
   * **Tối đa chỉ nên 7 node EMQX** do hạn chế của Mnesia cluster
   * Tuy nhiên, **phía đối tác xác nhận có thể chấp nhận mất dữ liệu**, nên giải pháp này vẫn đảm bảo hiệu quả và phù hợp với mục tiêu sản phẩm

---

### 📌 Các yếu tố then chốt đã được giải quyết

| Vấn đề                         | Giải pháp triển khai hiện tại                          |
| ------------------------------ | ------------------------------------------------------ |
| Chia tải MQTT không đồng đều   | HAProxy TCP routing sticky → EMQX nodes                |
| Không failover được khi LB lỗi | HAProxy active/passive workload, tự recover theo probe |
| TLS phân tán, khó kiểm soát    | TLS hợp nhất tại HAProxy (pem key mount từ Secret)     |
| Rủi ro mất dữ liệu khi crash   | PVC mount theo pod, rsync backup thủ công (nếu cần)    |
| Scale thủ công, khó monitor    | Helm chart + config map → giám sát dễ, rollout nhanh   |

