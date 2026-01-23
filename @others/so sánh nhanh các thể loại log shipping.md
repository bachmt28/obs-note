
---

## 🧭 1. Tổng quan 

| Tiêu chí                                       | **rsyslog**                                    | **Fluent Bit**                             | **Fluentd**             | **Logstash**              | **Vector**                      | **OTel Collector**                          |
| ---------------------------------------------- | ---------------------------------------------- | ------------------------------------------ | ----------------------- | ------------------------- | ------------------------------- | ------------------------------------------- |
| **Ngôn ngữ**                                   | C                                              | C                                          | Ruby                    | JRuby (Java)              | Rust                            | Go                                          |
| **Footprint**                                  | Rất nhẹ                                        | Rất nhẹ                                    | Trung bình              | Nặng                      | Nhẹ                             | Trung bình                                  |
| **Hiệu năng / Throughput**                     | Cao, ổn định                                   | Cao                                        | Vừa                     | Vừa                       | Rất cao                         | Cao                                         |
| **Tài nguyên CPU/RAM**                         | 1–5% / ~50 MB                                  | 1–5% / ~70 MB                              | 5–10% / 150 MB          | 10–25% / 500 MB           | 3–7% / 100 MB                   | 5–8% / 150 MB                               |
| **Khả năng parse JSON, regex, transform**      | Có (nhưng hạn chế, cú pháp khó)                | Tốt                                        | Rất tốt                 | Rất mạnh                  | Rất tốt                         | Tốt                                         |
| **Module/plugin ecosystem**                    | Cổ điển, nhiều (imfile, omfwd, mmjsonparse...) | Nhiều, hiện đại                            | Rất nhiều               | Rất nhiều                 | Nhiều (TOML-based)              | Chuẩn OpenTelemetry                         |
| **Hỗ trợ output (ES, Kafka, Loki, QRadar, …)** | Hạn chế, chủ yếu syslog/TCP                    | Rộng (ES, Loki, Kafka, S3...)              | Rất rộng                | Rất rộng                  | Rất rộng                        | Chuẩn OTel exporter                         |
| **Quản lý cấu hình**                           | File tĩnh, phức tạp (RainerScript)             | TOML/YAML, dễ đọc                          | Ruby DSL                | Grok/pipeline khó đọc     | TOML rõ ràng                    | YAML (chuẩn OTel)                           |
| **HA & Buffering**                             | Có queue disk-assisted                         | Có mem/disk buffer                         | Có, nhưng nặng          | Có (persistent queue)     | Có, tối ưu                      | Có, theo exporter                           |
| **Dễ debug**                                   | Trung bình                                     | Dễ                                         | Dễ                      | Khó                       | Dễ                              | Dễ                                          |
| **TLS / Auth / Retry**                         | Có                                             | Có                                         | Có                      | Có                        | Có                              | Có                                          |
| **Kịch bản phù hợp**                           | Forward log nhẹ, syslog truyền thống           | Lightweight agent cho container, IoT, edge | Log pipeline trung bình | Trung tâm xử lý (log hub) | Hạ tầng hiện đại, observability | Chuẩn hóa telemetry (logs, traces, metrics) |

---

## ⚙️ 2. Nhìn dưới lăng kính chiến thuật

### 🪶 **rsyslog – “lão tướng giữ cổng”**

* Ưu điểm:

  * Cực kỳ ổn định, tiêu tốn tài nguyên cực thấp.
  * Chạy trong mọi distro Linux, tích hợp sẵn từ kernel tới syslog().
  * Lý tưởng cho **airgap** hoặc **máy forward tuyến đầu (log shipper → SIEM)**.
* Nhược điểm:

  * Ngôn ngữ cấu hình RainerScript cổ, khó đọc.
  * Thiếu native output như Elasticsearch, Kafka, Loki (cần plugin ngoài).
  * Không mạnh khi cần enrich, parse JSON sâu.

> 💬 **Vai trò:** tường thành thủ thành – chuyển log thô đi nhanh, chắc, không rơi gói.

---

### ⚡ **Fluent Bit – “trinh sát tốc chiến”**

* Viết bằng C, siêu nhẹ, sinh ra cho **container / edge / Kubernetes node**.
* Cấu hình YAML/TOML dễ đọc, có hàng trăm plugin input/output.
* Dễ cắm vào Elasticsearch, Loki, Kafka, S3.
* Có filter, parser JSON, key-value mạnh hơn rsyslog nhiều.

> 💬 **Vai trò:** log shipper nhẹ trong K8s, edge, hoặc agent unified observability.

---

### 🔥 **Fluentd – “sư phụ của Fluent Bit”**

* Mạnh, dẻo, modular — nhưng nặng (Ruby).
* Dành cho log aggregation trung tâm, transform pipeline.
* Dễ tắc nghẽn khi throughput cao (nhiều Ruby thread, GC delay).

> 💬 **Vai trò:** trung tâm hợp nhất log, chứ không nên cài ở node.

---

### 🧱 **Logstash – “pháo đài của ELK”**

* Cực mạnh trong parse và enrich log (grok, mutate, translate, …).
* Nhưng footprint rất lớn (Java).
* Dùng tốt khi kết hợp với Elasticsearch + Kibana, nhưng nặng nếu chỉ forward.

> 💬 **Vai trò:** pipeline trung tâm trong hệ thống ELK, không nên đặt ở edge.

---

### 🦾 **Vector (by Datadog)** – “hậu duệ cơ giới hóa”

* Viết bằng Rust → hiệu năng cực cao, an toàn, multi-core tốt.
* Có thể thay thế cả rsyslog và Fluent Bit.
* Config rõ ràng, dễ quản lý, có built-in metrics & health.
* Dần trở thành log shipper hiện đại phổ biến nhất ngoài OTel Collector.

> 💬 **Vai trò:** successor hiện đại, cân bằng giữa hiệu năng và khả năng transform.

---

### 🌐 **OpenTelemetry Collector – “bộ chỉ huy liên quân”**

* Chuẩn hóa telemetry 3 loại: logs, metrics, traces.
* Hơi nặng hơn Fluent Bit nhưng cực kỳ linh hoạt.
* Tương lai dài hạn của observability stack.

> 💬 **Vai trò:** collector thống nhất cho observability hybrid.

---

## 🧩 3. Kết luận chiến lược

| Môi trường / Mục tiêu                                      | Giải pháp khuyên dùng                |
| ---------------------------------------------------------- | ------------------------------------ |
| GitLab server airgap, forward về SIEM (QRadar, syslog TCP) | **rsyslog** ✅                        |
| Cluster K8s, cần gom log app/container, đẩy Loki/ES        | **Fluent Bit**                       |
| Trung tâm aggregation, cần enrich log                      | **Fluentd** hoặc **Logstash**        |
| Tập trung pipeline thống nhất, có metric + trace           | **OpenTelemetry Collector**          |
| Thay thế rsyslog hiện đại, lightweight, native JSON        | **Vector** (tùy khả năng triển khai) |

---

