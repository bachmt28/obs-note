
---
## 1. Monitor tại tầng HAProxy

Đây là tiền tuyến, phải nắm sát tình hình giao tranh:
* **Bật stats socket** (ngoài stats web GUI):
  Cho phép collect qua script/Prometheus exporter.
  ```txt
  stats socket /var/run/haproxy.sock mode 600 level admin
  ```
* **Expose metrics dạng Prometheus** (tích hợp native từ HAProxy 2.x):
  ```txt
  frontend stats
      bind *:8404
      http-request use-service prometheus-exporter if { path /metrics }
  ```
* **Theo dõi key metrics**:
  * `session rate` (Cur, Max) → phát hiện spike.
  * `bytes in/out` → trend traffic.
  * `errors Conn/Resp/Redis` → dấu hiệu connection reset hoặc backend fail.
  * `Chk` và `Dwn` → số lần backend rớt/timeout.
  * `Status` (UP/DOWN/MAINT) → tình trạng node.
* **Cảnh báo real-time** qua Prometheus + Alertmanager hoặc push sang Webex/Slack. Ví dụ:
  * Alert khi `backend DOWN > 0`.
  * Alert khi `queue > 0` liên tục >30s.
  * Alert khi `session rate > ngưỡng baseline`.

---
## 2. Monitor tại tầng EMQX
HAProxy chỉ cho thấy “giao tranh ở cổng thành”. Phải vào tận trong thành (EMQX) để biết quân lính bên trong thế nào:
* **Enable Prometheus exporter trong EMQX** (`emqx_prometheus` plugin).
* Key metrics cần bắt:
  * `connections.count` / `connections.max`.
  * `messages.in`, `messages.out`.
  * `topics.count`, `subscriptions.count`.
  * Latency metrics (delivery, routing).
* **Health check nội bộ**: EMQX cluster join/leave, leader election log.
* **Cảnh báo**:
  * Khi số connection tiệm cận limit.
  * Khi node cluster drop >1 lần/phút.
  * Khi message drop hoặc retry tăng cao.

---
## 3. Monitor hạ tầng & tích hợp

Không thể bỏ qua “đường tiếp vận”:
* **Node metrics**: CPU, RAM, Ephemeral storage, Network (đặc biệt là TCP port 1883/8883).
* **K8s layer**: Pod restarts, HPA scaling events, pod distribution.
* **Log pipeline**: Fluentd/Vector ship log của HAProxy + EMQX về Loki/Elastic để query sự cố.
* **End-to-end probes**:
  * Script hoặc blackbox exporter → test publish/subscribe roundtrip qua HAProxy → EMQX.
  * Cảnh báo khi latency vượt ngưỡng, hoặc message không về đúng topic.

---
## 4. Best practice tổng hợp
* **Triển khai cặp monitor song song**: Prometheus (metrics) + Loki (logs).
* **Chia dashboard**:
  * HAProxy view (traffic, backend health).
  * EMQX view (connection, message, cluster state).
  * Infra view (K8s nodes, pods).
* **Alert layering**:
  * L1: Cổng thành (HAProxy backend down).
  * L2: Nội thành (EMQX cluster fail).
  * L3: Hậu cần (K8s node pressure, network lỗi).
* **Chaos test định kỳ**: simulate backend EMQX pod die, confirm HAProxy failover đúng, alert trigger đủ.
---