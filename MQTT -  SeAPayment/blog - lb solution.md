# Tóm tắt
- https://www.emqx.com/en/blog/mqtt-broker-clustering-part-1-load-balancing
- https://www.emqx.com/en/blog/mqtt-broker-clustering-part-2-sticky-session-load-balancing
- https://www.emqx.com/en/blog/mqtt-broker-clustering-part-3-challenges-and-solutions-of-emqx-horizontal-scalability
---

## 📘 1. **Load balancing – MQTT Broker Clustering Part 1**

- **MQTT là gì & mô hình pub/sub**: MQTT là một protocol nhẹ, hoạt động trên TCP/TLS, dùng cho kết nối thiết bị IoT – client subscribe chủ đề, broker phân phối message. ([www.emqx.com](https://www.emqx.com/en/blog/mqtt-broker-clustering-part-1-load-balancing?utm_source=chatgpt.com "Load balancing - MQTT broker clustering part 1 | EMQ - EMQX"))
    
- **Thách thức scale**: Mặc dù EMQX có thể chịu tải đến ~2 triệu kết nối trên 1 node, để phục vụ hàng triệu thiết bị thì cần cluster.
    
- **Load balancing ở tầng TCP**: Cân bằng kết nối (eg. AWS NLB, HAProxy, NGINX) được thực hiện ở tầng transport/IP, mới chủ yếu focus đến 2021. ([docs.emqx.com](https://docs.emqx.com/en/emqx/latest/design/clustering.html?utm_source=chatgpt.com "Design for EMQX Clustering"), [www.emqx.com](https://www.emqx.com/en/blog/mqtt-broker-clustering-part-1-load-balancing?utm_source=chatgpt.com "Load balancing - MQTT broker clustering part 1 | EMQ - EMQX"))
    
- **Load balancing ở tầng ứng dụng**: Bắt đầu từ HAProxy 2.4 và NGINX Plus hỗ trợ parsing protocol MQTT để điều hướng thông minh hơn (ví dụ phân tích client identifier), nâng cao khả năng sticky và session routing.
    

---

## 📘 2. **Sticky-session Load Balancing – MQTT Broker Clustering Part 2**

- **Khái niệm session MQTT**: Khi dùng “Clean‑Session=false”, broker cần giữ state (các topic đã subscribe, QoS 1/2 message chưa nhận) ngay cả khi client disconnect. Khi reconnect, client sẽ nhận lại các message còn tồn. ([www.emqx.com](https://www.emqx.com/en/blog/mqtt-broker-clustering-part-2-sticky-session-load-balancing?utm_source=chatgpt.com "Sticky session load balancing - MQTT broker clustering part 2 - EMQX"))
    
- **Vấn đề session takeover**: Khi client reconnect và được load balancer đưa đến node khác, broker mới phải thực hiện `session takeover`, tức migrate toàn bộ session và mọi message chờ tới node đó. Tốn tài nguyên, nhất là khi volume lớn.
    
- **Sticky sessions**: Giải pháp là sử dụng load balancer (HAProxy >=2.4) để “dính session”—định tuyến client dựa trên client_id, đặt trong `stick-table` để mỗi client luôn về đúng node ban đầu, tránh việc migrate session.
    
- **Ví dụ config HAProxy**: Bật `stick-table` key theo `mqtt_field_value(connect,client_identifier)` – giữ sticky cho session MQTT. ([docs.emqx.com](https://docs.emqx.com/en/emqx/latest/deploy/cluster/lb-haproxy.html?utm_source=chatgpt.com "Load Balance EMQX Cluster with HAProxy"))
    

---

## 📘 3. **Challenges & Solutions – Horizontal Scalability (Part 3)**

- **Backend database Mnesia**: EMQX dùng Erlang Mnesia để lưu session, topic, route… là một distributed, transactional, NoSQL DB nhúng chạy cùng broker. ([www.emqx.com](https://www.emqx.com/en/blog/mqtt-broker-clustering-part-3-challenges-and-solutions-of-emqx-horizontal-scalability?utm_source=chatgpt.com "Challenges and Solutions of EMQX horizontal scalability - EMQX"))
    
- **Giới hạn của Mnesia**: Vì cơ chế mesh và transaction coordination, càng nhiều nodes thì overhead càng tăng, ảnh hưởng hiệu năng và dễ gặp tình trạng split-brain. ([www.emqx.com](https://www.emqx.com/en/blog/reaching-100m-mqtt-connections-with-emqx-5-0?utm_source=chatgpt.com "Reaching 100M MQTT connections with EMQX 5.0 | EMQ"))
    
- **Giải pháp Mria + RLOG DB (EMQX ≥5.0)**:
    
    - **Mria**: phân node thành `core` (viết) và `replicant` (đọc).
        
    - **RLOG (replication log)**: replicant duy trì replica read-only, mọi writes tập trung vào core, giảm overhead coordination và split-brain. ([www.emqx.com](https://www.emqx.com/en/blog/mqtt-broker-clustering-part-3-challenges-and-solutions-of-emqx-horizontal-scalability?utm_source=chatgpt.com "Challenges and Solutions of EMQX horizontal scalability - EMQX"), [www.emqx.com](https://www.emqx.com/en/blog/reaching-100m-mqtt-connections-with-emqx-5-0?utm_source=chatgpt.com "Reaching 100M MQTT connections with EMQX 5.0 | EMQ"))
        
- **Kết quả benchmark**: Cluster 23 node (3 core + 20 replicant) có thể phục vụ 100 triệu kết nối đồng thời, throughput >1 triệu msg/s, replicant nodes đạt ~97% CPU. So sánh với Mnesia thì RLOG cho hiệu năng kết nối/con người vượt trội. ([www.emqx.com](https://www.emqx.com/en/blog/reaching-100m-mqtt-connections-with-emqx-5-0?utm_source=chatgpt.com "Reaching 100M MQTT connections with EMQX 5.0 | EMQ"))
    

---

### 🔗 Tóm tắt

1. Dùng **load balancer TCP/Application** để phân phối traffic đến cluster broker.
    
2. Sử dụng **sticky session** để tránh migration session gây overhead khi client reconnect.
    
3. Giải quyết bottleneck DB replication dùng **combinations Mria + RLOG**, tách read/write và phân biệt replication để scale ngang cực lớn, đạt kết 100 triệu device.
    

---
# Chi tiết 
## Part 1
---

## 1. Giới thiệu về MQTT

- MQTT là một giao thức **Publish/Subscribe** tiêu chuẩn của OASIS, được thiết kế **nhẹ nhàng**, phù hợp cho IoT – dùng **TCP/TLS** ở tầng transport. ([www.emqx.com](https://www.emqx.com/en/blog/mqtt-broker-clustering-part-1-load-balancing?utm_source=chatgpt.com "Load balancing - MQTT broker clustering part 1 | EMQ - EMQX"))
    
- Giống như HTTP, nhưng thay vì request/response, MQTT sử dụng mô hình **publish/subscribe**, nơi:
    
    - **Publisher** gửi dữ liệu cho broker trên một topic cụ thể.
        
    - **Broker** phân phối dữ liệu đó cho mọi **Subscriber** đã đăng ký topic.  
        Ví dụ: cảm biến nhiệt gửi “27.5°C” đến broker; ứng dụng smart-home nhận tin qua subscribe. ([www.emqx.com](https://www.emqx.com/en/blog/mqtt-broker-clustering-part-1-load-balancing?utm_source=chatgpt.com "Load balancing - MQTT broker clustering part 1 | EMQ - EMQX"))
        

---

## 2. Thách thức khi scale

- Một node EMQX có thể hỗ trợ tới **2 triệu kết nối** – như Raspberry Pi cũng có thể dùng cho Smart Home. ([www.emqx.com](https://www.emqx.com/en/blog/mqtt-broker-clustering-part-1-load-balancing?utm_source=chatgpt.com "Load balancing - MQTT broker clustering part 1 | EMQ - EMQX"))
    
- Với quy mô lớn hơn – như hàng triệu xe, đèn đường,… – nhu cầu vượt xa khả năng của 1 node → cần **cluster** (nhiều node phối hợp).
    
- Nhưng cluster đem lại các vấn đề:
    
    - **Broker discovery**: client cần biết broker cần kết nối.
        
    - **Session takeover**: khi reconnect client có session chưa clean, bị chuyển giữa các node.
        
    - **Routing table toàn cục**: các node phải share dữ liệu về topic/subscribers. ([www.emqx.com](https://www.emqx.com/en/blog/mqtt-broker-clustering-part-1-load-balancing?utm_source=chatgpt.com "Load balancing - MQTT broker clustering part 1 | EMQ - EMQX"))
        

---

## 3. Vai trò của Load Balancer

Load Balancer (LB) đóng vai trò **trung gian thông minh** để:

1. **Giấu endpoint thật** của các broker – client chỉ cần connect tới LB.
    
2. **Chia tải đều** giữa các broker – bằng round-robin, hash, random, hoặc sticky.
    
3. **Terminating TLS** – LB xử lý TLS giúp giảm tải cho broker.
    
4. **Phân phối thông minh** – LB có thể nhận biết MQTT protocol và phân tích gói CONNECT để xử lý sticky sessions. ([docs.emqx.com](https://docs.emqx.com/en/emqx/latest/deploy/cluster/lb.html?utm_source=chatgpt.com "Configure Load Balancer | EMQX Docs"), [www.emqx.com](https://www.emqx.com/en/blog/mqtt-broker-clustering-part-1-load-balancing?utm_source=chatgpt.com "Load balancing - MQTT broker clustering part 1 | EMQ - EMQX"))
    

LB có thể hoạt động ở hai tầng:

- **Transport-layer (TCP)**: như AWS NLB, HAProxy (mode TCP), NGINX.
    
- **Application-layer** (có hiểu MQTT): HAProxy ≥2.4, NGINX Plus. ([www.emqx.com](https://www.emqx.com/en/blog/mqtt-broker-clustering-part-1-load-balancing?utm_source=chatgpt.com "Load balancing - MQTT broker clustering part 1 | EMQ - EMQX"))
    

---

## 4. Cân bằng tải ở tầng Transport

- Các sản phẩm hiện nay chủ yếu hoạt động ở tầng **TCP**, không nhìn sâu vào MQTT, ví dụ: AWS NLB, NGINX, HAProxy.
    
- Cách hoạt động: LB nhận kết nối TCP và chuyển tiếp đến backend broker dựa theo thuật toán như round-robin, leastconn, etc.
    

---

## 5. Cân bằng tại tầng Application với MQTT-aware LB

- **HAProxy 2.4+** và **NGINX Plus** có khả năng phân tích gói MQTT:
    
    - Đọc được client identifier từ CONNECT packet.
        
    - Cho phép áp dụng **sticky sessions** – định tuyến cùng một client đến cùng 1 broker, giúp tránh session takeover.
        
    - Cung cấp thêm giá trị về **bảo mật** (reject connections không hợp lệ). ([docs.emqx.com](https://docs.emqx.com/en/emqx/latest/deploy/cluster/lb-haproxy.html?utm_source=chatgpt.com "Load Balance EMQX Cluster with HAProxy"))
        

Điều này sẽ được khám phá sâu hơn trong phần Part 2.

---

## 6. Tóm tắt mô hình tổng quát

```
 Client
   │
   ▼
 Load Balancer (TCP & TLS termination, MQTT parsing)
   ├─ Round-robin, weighted, sticky...
   └─ Dispatch CONNECT packets to:
       ├─ MQTT Broker Node 1
       ├─ MQTT Broker Node 2
       └─ MQTT Broker Node 3
          ↑
          └── Part of EMQX Cluster sharing routing/session data
```

- Client chỉ cần connect đến LB – dễ quản lý và scale.
    
- LB phân tải và xử lý TLS, MQTT parsing.
    
- Broker chỉ chuyên xử lý message – phân phối dữ liệu giữa nodes.
    

---

### 🚀 Kết luận:

- **Load Balancer** là thành phần không thể thiếu khi triển khai cluster broker MQTT.
    
- Có thể dùng ở tầng **transport** (TCP level), hoặc tầng **application** (aware MQTT).
    
- **HEProxy 2.4+** và **NGINX Plus** hỗ trợ cân bằng ứng dụng MQTT – mở đường cho sticky-session, session stability.
    
- Part 2 và Part 3 sẽ lần lượt đi sâu vào **sticky sessions** và **phân tán state/database** (Mnesia → Mria + RLOG).
    

---
## Part 2
Dưới đây là phần II — **Sticky‐session Load Balancing** (Part 2) of EMQX blog series, giải thích chi tiết, theo từng bước:

---

## 1. 🧱 MQTT Session – Phiên đăng nhập được bảo lưu

- Khi client sử dụng `Clean‐Session=false`, broker sẽ **giữ lại session state**, bao gồm:
    
    - Danh sách topic đã subscribe
        
    - Message QoS 1/2 chưa gửi đến client
        
- Mục tiêu: Nếu client disconnect rồi reconnect, thì broker sẽ **tự động gửi lại các message còn tồn trong session đó** ([www.emqx.com](https://www.emqx.com/en/blog/mqtt-broker-clustering-part-2-sticky-session-load-balancing?utm_source=chatgpt.com "Sticky session load balancing - MQTT broker clustering part 2 - EMQX")).
    

### Thử tượng:

Client tự động mất kết nối vì mạng → reconnect → muốn tiếp tục nhận các message đã bị bỏ lỡ → cần session vẫn còn nguyên.

---

## 2. ⚠️ Session Takeover – Khi session bị chuyển nhầm node

Trong **cluster EMQX**, client kết nối qua Load Balancer (LB). Nếu LB chuyển hướng lần sau đến node khác:

- Node mới sẽ phải nhận session từ node cũ → thực hiện **session takeover** (“chuyển session”)
    
- Session takeover gây **tốn tài nguyên** và **độ trễ**, đặc biệt khi message backlog lớn ([www.emqx.com](https://www.emqx.com/en/blog/mqtt-broker-clustering-part-2-sticky-session-load-balancing?utm_source=chatgpt.com "Sticky session load balancing - MQTT broker clustering part 2 - EMQX")).
    

---

## 3. 🧲 Sticky Session – Giải cứu nhờ “dính session”

- **Load balancer phải hiểu được MQTT** (phân tích gói CONNECT)
    
- Extract **client identifier** (hoặc user name)
    
- Sử dụng cơ chế “stick table” (bảng dính) để định tuyến client luôn đến đúng node ban đầu, tránh session takeover ([SegmentFault](https://segmentfault.com/a/1190000040728521/en?utm_source=chatgpt.com "物联网- Sticky Session Load Balancing-MQTT Broker cluster ..."), [www.emqx.com](https://www.emqx.com/en/blog/mqtt-broker-clustering-part-2-sticky-session-load-balancing?utm_source=chatgpt.com "Sticky session load balancing - MQTT broker clustering part 2 - EMQX")).
    

> Bảng này có thể là **hashing dựa vào client_id** hoặc **mapping table** nếu cluster kích thước cố định.

---

## 4. ⚙️ Demo: Cấu hình HAProxy 2.4 + EMQX 4.3 cluster

### 4.1 Tạo Docker network

```
docker network create test.net
```

### 4.2 Khởi chạy 2 EMQX node với PROXY_PROTOCOL

```bash
docker run -d --name n1.test.net --net test.net \
  -e EMQX_NODE_NAME=emqx@n1.test.net \
  -e EMQX_LISTENER__TCP__EXTERNAL__PROXY_PROTOCOL=on \
  emqx/emqx:4.3.7
# và node2 tương tự
```

- `PROXY_PROTOCOL` giúp broker biết IP thật của client qua LB ([www.emqx.com](https://www.emqx.com/en/blog/mqtt-broker-clustering-part-2-sticky-session-load-balancing?utm_source=chatgpt.com "Sticky session load balancing - MQTT broker clustering part 2 - EMQX")).
    

### 4.3 Join cluster

```bash
docker exec -it n2.test.net emqx_ctl cluster join emqx@n1.test.net
```

→ Hiện thấy 2 node chạy chung đàm ([www.emqx.com](https://www.emqx.com/en/blog/mqtt-broker-clustering-part-2-sticky-session-load-balancing?utm_source=chatgpt.com "Sticky session load balancing - MQTT broker clustering part 2 - EMQX")).

---

### 4.4 Cài đặt HAProxy với Sticky Table

config mẫu (`haproxy.cfg`):

```text
backend emqx_tcp_back
  mode tcp
  stick-table type string len 32 size 100k expire 30m
  stick on req.payload(0,0),mqtt_field_value(connect,client_identifier)
  server emqx1 n1.test.net:1883 check-send-proxy send-proxy-v2
  server emqx2 n2.test.net:1883 check-send-proxy send-proxy-v2
```

- **stick-table** chứa mapping `client_id` → `node`
    
- HAProxy 2.4 đọc trường client_identifier trong MQTT CONNECT, áp sticky routing ([www.emqx.com](https://www.emqx.com/en/blog/mqtt-broker-clustering-part-2-sticky-session-load-balancing?utm_source=chatgpt.com "Sticky session load balancing - MQTT broker clustering part 2 - EMQX"), [SegmentFault](https://segmentfault.com/a/1190000040728521/en?utm_source=chatgpt.com "物联网- Sticky Session Load Balancing-MQTT Broker cluster ...")).
    

---

### 4.5 Test bằng Mosquitto clients

Subscriber:

```bash
mosquitto_sub -h proxy.test.net -t 't/#' -I subscriber1
```

Publisher:

```bash
mosquitto_pub -h proxy.test.net -t 't/xyz' -m 'hello'
```

→ Subscriber nhận được message `hello` nếu routing hoạt động đúng ([www.emqx.com](https://www.emqx.com/en/blog/mqtt-broker-clustering-part-2-sticky-session-load-balancing?utm_source=chatgpt.com "Sticky session load balancing - MQTT broker clustering part 2 - EMQX")).

---

## 5. 🔍 Kiểm tra Sticky-table trong HAProxy

Chạy lệnh truy vấn bảng sticky:

```
show table emqx_tcp_back | socat stdio tcp4-connect:127.0.0.1:9999
```

Output ví dụ:

```
key=subscriber1 ... server_id=2 server_key=emqx2
```

→ `subscriber1` đang dính vào `emqx2` ([www.emqx.com](https://www.emqx.com/en/blog/mqtt-broker-clustering-part-2-sticky-session-load-balancing?utm_source=chatgpt.com "Sticky session load balancing - MQTT broker clustering part 2 - EMQX")).

---

## ✅ Kết luận phần 2

- **MQTT session** cho phép broker giữ lại trạng thái client khi disconnect.
    
- **Session takeover** trong cluster gây tốn kém khi LB route sai node.
    
- **Sticky session** giải quyết bằng cách dán client đến cùng 1 node qua LB.
    
- Cấu hình thực tế với HAProxy 2.4 + MQTT-aware parsing để triển khai sticky-session.
    

---
## Part 3
Dưới đây là phần III—**Challenges and Solutions of EMQX Horizontal Scalability** (Part 3), với giải thích từng bước một theo nội dung blog của EMQX:

---

## 1️⃣ EMQX 4.x và Mnesia – Cơ sở dữ liệu nhúng

- **Mnesia** là DB NoSQL nhúng, chạy trong cùng tiến trình EMQX, hỗ trợ ACID transactional, replication theo kiểu full-mesh giữa các node Erlang/EMQX ([www.emqx.com](https://www.emqx.com/en/blog/mqtt-broker-clustering-part-3-challenges-and-solutions-of-emqx-horizontal-scalability?utm_source=chatgpt.com "Challenges and Solutions of EMQX horizontal scalability - EMQX")).
    
- Ưu điểm:
    
    - Đọc ghi rất nhanh, giống đọc biến local.
        
    - Hỗ trợ phân tán, fault–tolerance dựa trên mesh.
        
    - Hỗ trợ transaction và đảm bảo ACID.
        
    - Tích hợp chặt với EMQX mà không cần ORM ([SegmentFault](https://segmentfault.com/a/1190000040891953/en?utm_source=chatgpt.com "物联网- Challenges and countermeasures related to the horizontal ..."), [www.emqx.com](https://www.emqx.com/en/blog/mqtt-broker-clustering-part-3-challenges-and-solutions-of-emqx-horizontal-scalability?utm_source=chatgpt.com "Challenges and Solutions of EMQX horizontal scalability - EMQX")).
        
- Hạn chế:
    
    - Với nhiều node, full-mesh replication trở nên tốn tài nguyên (đường link ∝ N²), độ trễ tăng & dễ gặp split-brain.
        
    - Khó mở rộng cluster vượt quá ~7 node sao cho ổn định ([www.emqx.com](https://www.emqx.com/en/blog/mqtt-broker-clustering-part-3-challenges-and-solutions-of-emqx-horizontal-scalability?utm_source=chatgpt.com "Challenges and Solutions of EMQX horizontal scalability - EMQX")).
        

---

## 2️⃣ Giải pháp Mria + RLOG (từ EMQX 5.0)

### 🧠 Mria – Mở rộng Mnesia theo mô hình Core‑Replicant

- **Core nodes**: vẫn hoạt động như node Mnesia truyền thống—được phép viết dữ liệu, tự nhiên “đắt giá”.
    
- **Replicant nodes**: đọc-only, chỉ replicate dữ liệu từ Core theo kiểu star topology. Mỗi replicant kết nối với 1 core, không tham gia ghi hay transaction ([www.emqx.com](https://www.emqx.com/en/blog/mqtt-broker-clustering-part-3-challenges-and-solutions-of-emqx-horizontal-scalability?utm_source=chatgpt.com "Challenges and Solutions of EMQX horizontal scalability - EMQX")).
    
- Ưu điểm:
    
    - Scale ngang dễ dàng, latency viết không bị ảnh hưởng khi thêm replicant.
        
    - Replicant có thể autoscaling: thêm/xóa đều minh bạch với hệ thống.
        
    - Tránh split-brain, cải thiện độ ổn định cluster lớn.
        

### 📚 RLOG – Phân mảnh logs replication

- Mỗi nhóm data (table) được ghi log dưới dạng một **RLOG Shard**.
    
- Replicant nhận log theo từng shard—cung cấp replication song song, không đồng bộ ([www.emqx.com](https://www.emqx.com/en/blog/mqtt-broker-clustering-part-3-challenges-and-solutions-of-emqx-horizontal-scalability?utm_source=chatgpt.com "Challenges and Solutions of EMQX horizontal scalability - EMQX"), [www.emqx.com](https://www.emqx.com/en/blog/how-emqx-5-0-achieves-100-million-mqtt-connections?utm_source=chatgpt.com "How EMQX under the new architecture of Mria + RLOG achieves ...")).
    

---

## 3️⃣ Triển khai Pratical trong EMQX 5.0

- Mặc định, các node đều là Core. Để chuyển thành Replicant, cấu hình **`EMQX_NODE__DB_ROLE=replicant`** hoặc chỉnh `node.db_role` trong `emqx.conf` ([SegmentFault](https://segmentfault.com/a/1190000040891953/en?utm_source=chatgpt.com "物联网- Challenges and countermeasures related to the horizontal ...")).
    
- Ưu khuyến nghị:
    
    - Cấu hình tối thiểu: **3 Core + N Replicants**.
        
    - Cụm nhỏ (≤3 node): toàn Core vẫn ổn.
        
    - Cụm lớn (≥10 node): phân chia rõ Core & Replicant để scale hiệu quả.
        

### 🔧 Xử lý sự cố và giám sát

- Replicant tự động chuyển sang core khác nếu core collapse—khách hàng không bị mất kết nối nhưng có thể nhận thông tin stale.
    
- Core crash: điều khiển replicant tự reconnect.
    
- Kết nối đến replicant sẽ bị ngắt nếu replicant shutdown, nhưng hệ thống vẫn an toàn.
    
- Giám sát qua Prometheus metrics (`emqx_mria_*`, `lag`, `bootstrap_time`, v.v.) và console command `mria_rlog:status()` ([www.emqx.com](https://www.emqx.com/en/blog/how-emqx-5-0-achieves-100-million-mqtt-connections?utm_source=chatgpt.com "How EMQX under the new architecture of Mria + RLOG achieves ...")).
    

---

## ✅ Kết quả và Hiệu quả

- EMQX 5.0 ứng dụng Mria + RLOG cluster đã đạt thử nghiệm:
    
    - 23 node cluster (3 core + 20 replicant).
        
    - 100 triệu kết nối MQTT đồng thời.
        
    - Throughput >1 triệu message/s.
        
    - Sử dụng ~97% CPU replicant, chứng tỏ hiệu quả scale ngang ([www.emqx.com](https://www.emqx.com/en/blog/how-emqx-5-0-achieves-100-million-mqtt-connections?utm_source=chatgpt.com "How EMQX under the new architecture of Mria + RLOG achieves ...")).
        
- Mô hình mới giải quyết triệt để vấn đề overhead replication của Mnesia full-mesh, hỗ trợ cluster lớn & autoscale.
    

---

## 📋 Tổng kết Part 3

|Vấn đề|Cách giải quyết|Lợi ích|
|---|---|---|
|Mnesia full-mesh không scale|Mria (Core + Replicant)|Giảm replication overhead, hỗ trợ scale >20 node|
|Ghi replication đồng bộ hạn chế|RLOG Shards|Cho phép phân shard, replica phi đồng bộ|
|Quản lý cluster lớn phức tạp|Cấu hình trực quan, autoscale|Dễ vận hành, resilient|
