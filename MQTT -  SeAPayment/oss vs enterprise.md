
---

## ⚔️ So sánh chiến lược: EMQX OSS vs EMQX Enterprise

| Tiêu chí                     | **EMQX OSS**                                          | **EMQX Enterprise**                                         |
| ---------------------------- | ----------------------------------------------------- | ----------------------------------------------------------- |
| **Database backend**         | `mnesia` – embedded, Erlang-based                     | External DBs: PostgreSQL, Redis, MongoDB, MySQL             |
| **Cơ chế lưu dữ liệu**       | Dữ liệu ghi trực tiếp vào **local disk** (trong pod)  | Dữ liệu ghi vào DB **bên ngoài**, tách rời hệ thống compute |
| **Replication**              | Peer-to-peer, **full mesh** giữa các node             | Ghi vào DB → Node nào cũng access được, không cần full mesh |
| **Session/ACL/User storage** | Trên mnesia                                           | Redis / PostgreSQL (tuỳ cấu hình)                           |
| **Số node tối đa**           | **7 nodes** *(giới hạn kỹ thuật của mnesia cluster)*  | Không giới hạn (scale theo DB backend)                      |
| **Khả năng khôi phục node**  | Phụ thuộc **disk + hostname** để khôi phục đúng node  | Node mới có thể join lại miễn là DB vẫn còn                 |
| **Retained message & rule**  | Trên mnesia                                           | Lưu vào PostgreSQL                                          |
| **High Availability (HA)**   | Dễ bị ảnh hưởng nếu mất 1 node hoặc volume            | Đảm bảo HA thật sự qua DB replicate, cluster DB             |
| **Rolling Update**           | Có thể gây mất mát session / message nếu không sticky | Session lưu Redis → dễ rolling update hơn                   |

---

## 🔍 Giải thích chi tiết

### 1. **Mnesia – OSS Edition**

* Là **cơ sở dữ liệu phân tán nội bộ**, viết bằng Erlang.
* Mỗi node EMQX sẽ lưu dữ liệu trên đĩa cục bộ.
* Cluster mnesia hoạt động theo kiểu **full mesh** – node nào cũng kết nối trực tiếp với node còn lại → giới hạn **7 nodes** (vượt quá sẽ rối loạn đồng bộ).
* Nếu **hostname hoặc file dữ liệu bị lệch**, node không thể join lại đúng cluster → rất nhạy cảm với mất disk hoặc mount sai PVC.

⛔ Vì vậy, OSS **phụ thuộc mạnh vào trạng thái từng pod**, không phù hợp cho kiến trúc elastic, autoscale hoặc multi-AZ phức tạp.

---

### 2. **Enterprise – Kiến trúc tách rời compute/data**

* Lưu dữ liệu vào các **DB ngoài như PostgreSQL, Redis, MongoDB**:

  * ACL, user → Redis
  * Rule engine, message retained → PostgreSQL
  * Event, action → MongoDB
* Mỗi EMQX node chỉ xử lý logic, còn trạng thái và dữ liệu được **tách biệt ra ngoài** → scale tốt hơn, dễ HA hơn.

✅ Ưu thế:

* Node chết → spin lại là xong
* Mất 1 zone → chỉ cần DB còn là hệ thống vẫn sống
* Rolling update nhẹ nhàng, không mất session

---

## 🎯 Kết luận

| Kịch bản                      | EMQX OSS                  | EMQX Enterprise                   |
| ----------------------------- | ------------------------- | --------------------------------- |
| Small cluster ≤ 3 nodes       | ✅ Ok                      | ❌ Không cần thiết                 |
| Yêu cầu scale > 7 nodes       | ❌ Không thể               | ✅ Hợp lý                          |
| Session/User persistence mạnh | ❌ Không phù hợp           | ✅ Redis/Postgres đáp ứng tốt      |
| K8s dynamic environment       | ❌ Rủi ro cao nếu PVC fail | ✅ DB outside giúp recover dễ dàng |
| Cần HA và recovery thực sự    | ❌ Dựa vào disk + hostname | ✅ Chỉ cần DB còn                  |

---
👉 [https://www.emqx.com/en/blog/emqx-open-source-vs-enterprise](https://www.emqx.com/en/blog/emqx-open-source-vs-enterprise)

---

# 🔍 So sánh EMQX Open Source và EMQX Enterprise

Bài viết gốc nêu bật sự khác biệt giữa bản **EMQX Open Source** (miễn phí) và **EMQX Enterprise** (thương mại). Dưới đây là bản dịch và tổng hợp các ý chính để Soái tiện ra quyết sách.

---

## 1. Tầm nhìn và đối tượng sử dụng

| Phiên bản       | Mục tiêu chính                                                                  |
| --------------- | ------------------------------------------------------------------------------- |
| **Open Source** | Dành cho cộng đồng, startup, dự án vừa và nhỏ. Tập trung vào tốc độ phát triển. |
| **Enterprise**  | Dành cho doanh nghiệp quy mô lớn, yêu cầu về hiệu năng, HA, bảo mật và hỗ trợ.  |

---

## 2. Khả năng mở rộng (Scalability)

| Yếu tố           | EMQX Open Source                           | EMQX Enterprise                              |
| ---------------- | ------------------------------------------ | -------------------------------------------- |
| Mô hình cluster  | **Full mesh** – mỗi node kết nối trực tiếp | **Core + Replicant** – kiến trúc phân tầng   |
| Giới hạn số node | **Tối đa 7 nodes**                         | **Không giới hạn** (đã test trên 100+ nodes) |
| Hạ tầng cloud    | Khó triển khai multi-AZ, autoscaling       | Triển khai tốt trên cloud, hạ tầng phân tán  |

> ⚠️ *Mnesia trong bản OSS là nguyên nhân chính khiến kiến trúc full mesh không scale tốt, rất dễ “tắc nghẽn liên kết”.*

---

## 3. Cơ sở dữ liệu lưu trữ

| Thành phần                   | EMQX Open Source | EMQX Enterprise       |
| ---------------------------- | ---------------- | --------------------- |
| ACL, Authentication, Session | **Mnesia**       | **Redis, PostgreSQL** |
| Retained Messages, Rules     | Mnesia           | PostgreSQL            |
| Action/Event Logging         | Không hỗ trợ     | MongoDB / Kafka       |
| Quản lý bằng SQL             | ❌ Không          | ✅ Có (via PostgreSQL) |

---

## 4. Bảo mật và kiểm soát truy cập

| Yếu tố                         | EMQX Open Source | EMQX Enterprise |
| ------------------------------ | ---------------- | --------------- |
| Quản lý user qua dashboard     | Có               | Có              |
| RBAC nâng cao theo vai trò     | ❌ Không          | ✅ Có            |
| Hỗ trợ SSO (LDAP, OIDC, AD)    | ❌ Giới hạn       | ✅ Đầy đủ        |
| Audit log (theo dõi hoạt động) | ❌ Không          | ✅ Có            |

---

## 5. Tính năng doanh nghiệp

| Tính năng                          | OSS | Enterprise |
| ---------------------------------- | --- | ---------- |
| Quản lý qua giao diện web          | ✅   | ✅          |
| Visual Rule Engine (Luồng dữ liệu) | ❌   | ✅          |
| Quản lý cluster nâng cao           | ❌   | ✅          |
| Dashboard phân quyền đa tầng       | ❌   | ✅          |
| Backup/Restore nâng cao            | ❌   | ✅          |
| Hỗ trợ kỹ thuật 24/7 từ EMQ        | ❌   | ✅          |

---

## 6. Tính sẵn sàng cao (High Availability – HA)

| Yếu tố                     | EMQX OSS                | EMQX Enterprise                      |
| -------------------------- | ----------------------- | ------------------------------------ |
| Dữ liệu phụ thuộc hostname | ✅ Phụ thuộc             | ❌ Không phụ thuộc                    |
| Nếu pod chết               | Cần đúng PVC + hostname | Spin lại pod mới là xong             |
| Cluster tolerate node fail | Kém                     | Tốt (nhờ external DB và replication) |

---

## 7. Logging, Monitoring, Integration

| Tính năng                 | OSS        | Enterprise |
| ------------------------- | ---------- | ---------- |
| Prometheus Exporter       | ✅ Có       | ✅ Có       |
| Kafka Export              | ❌ Không    | ✅ Có       |
| Webhook, HTTP integration | ⚠️ Hạn chế | ✅ Mạnh mẽ  |
| Tracing, Audit Log        | ❌ Không    | ✅ Đầy đủ   |

---

## 8. Giấy phép & Chi phí

| Yếu tố    | OSS        | Enterprise                              |
| --------- | ---------- | --------------------------------------- |
| Giấy phép | Apache 2.0 | Thương mại, theo điều khoản EMQX        |
| Chi phí   | ✅ Miễn phí | 💰 Tính phí theo số node hoặc tính năng |

---

## 🎯 Kết luận: Khi nào nên chọn bản nào?

| Trường hợp sử dụng                                | Nên chọn bản |
| ------------------------------------------------- | ------------ |
| Dự án nhỏ, POC, lab nghiên cứu                    | Open Source  |
| Kiến trúc đơn giản ≤ 7 node, ít yêu cầu khôi phục | Open Source  |
| Cần đảm bảo HA thực sự, dữ liệu không mất         | Enterprise   |
| Có sẵn hạ tầng Redis/Postgres, cần scale mạnh     | Enterprise   |
| Cần giám sát & quản lý chính sách phức tạp        | Enterprise   |
| Cần hỗ trợ kỹ thuật và uptime cam kết             | Enterprise   |

---

