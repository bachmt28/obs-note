
---

## 1) Nhóm option **chung** (dùng cho cả `conn` & `pub`)

| Option                                    | Ý nghĩa                                                                                                                                                                                          |
| ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `-h, --host`                              | Host của broker (có thể là HAProxy, EMQX svc, LB…). Hỗ trợ danh sách host, phân tách dấu phẩy.                                                                                                   |
| `-p, --port`                              | Port kết nối. Mặc định 1883.                                                                                                                                                                     |
| `-V, --version`                           | Phiên bản MQTT: `3`, `4` (MQTT 3.1.1), `5` (MQTT v5 – default).                                                                                                                                  |
| `-c, --count`                             | **Số client** tối đa sẽ tạo.                                                                                                                                                                     |
| `-R, --connrate`                          | **Tốc độ tạo kết nối / giây**. Nếu `0`, sẽ dùng `-i`.                                                                                                                                            |
| `-i, --interval`                          | **Khoảng cách giữa các lần tạo kết nối** (ms). Dùng khi `--connrate=0`.                                                                                                                          |
| `--prefix`                                | **Tiền tố client_id**. Nếu KHÔNG set, tool tự sinh: `$HOST_bench_(pub                                                                                                                            |
| `--shortids`                              | Client ID ngắn, chỉ là số thứ tự (hoặc prefix + số). Dễ đọc/log, **nhưng dễ trùng nếu chạy nhiều bench song song**.                                                                              |
| `-n, --startnumber`                       | **Số bắt đầu** cho sequence của client_id. Dùng khi chạy nhiều máy để tránh trùng ID.                                                                                                            |
| `--num-retry-connect`                     | Số lần retry khi kết nối thất bại ban đầu (trước khi bỏ cuộc).                                                                                                                                   |
| `--reconnect`                             | **Số lần retry reconnect** sau khi đã kết nối thành công mà bị rớt. `0` = tắt.                                                                                                                   |
| `-S, --ssl`                               | Bật TLS/SSL. Dùng khi **client trực tiếp bắt tay TLS với cổng TLS thực sự**. (Trong case terminate ở HAProxy và port đó là TLS, có thể cần / không cần tùy cách HAProxy handle – bạn đã verify.) |
| `--ssl-version`                           | Ép version TLS (`tlsv1.2`, `tlsv1.3`, …).                                                                                                                                                        |
| `--cacertfile`, `--certfile`, `--keyfile` | Dùng mutual TLS / verify CA.                                                                                                                                                                     |
| `--ws`                                    | Kết nối qua **WebSocket**.                                                                                                                                                                       |
| `--quic`                                  | Dùng QUIC (nếu broker hỗ trợ).                                                                                                                                                                   |
| `-u, --username`, `-P, --password`        | Auth basic. Hỗ trợ biến `%i`, `%rand_N`.                                                                                                                                                         |
| `-k, --keepalive`                         | Keepalive (s). Mặc định `300`.                                                                                                                                                                   |
| `-C, --clean`                             | Clean session (true/false).                                                                                                                                                                      |
| `-x, --session-expiry`                    | MQTT v5 Session-Expiry (giây) cho session persistent.                                                                                                                                            |
| `-l, --lowmem`                            | Low memory mode (đỡ RAM, ăn CPU).                                                                                                                                                                |
| `--force-major-gc-interval`               | (ms) ép major GC định kỳ – chỉ có ý nghĩa khi bật `--lowmem`.                                                                                                                                    |
| `-Q, --qoe` / `--qoelog`                  | Bật tracking QoE & ghi log để post-process.                                                                                                                                                      |
| `--prometheus`, `--restapi`               | Expose metrics cho Prometheus (thường kèm `--restapi <port>`).                                                                                                                                   |
| `--log_to`                                | `console` (default) hoặc `null` (yên lặng).                                                                                                                                                      |

---

## 2) Option **riêng của `pub`** (gửi message)

|Option|Ý nghĩa|
|---|---|
|`-I, --interval_of_msg`|**Chu kỳ publish mỗi client** (ms).|
|`-t, --topic`|Topic để publish. Hỗ trợ biến: `%u` (username), `%c` (client_id), `%i` (client seq), `%s` (msg seq), `%rand_N`.|
|`-s, --size`|**Payload size** (bytes) nếu không dùng `-m`.|
|`-m, --message`|Payload cụ thể, có thể là literal hoặc template: `template://path/to/file`. Cho phép biến `%TIMESTAMP%`, `%UNIQUE%`, `%RANDOM%`, …|
|`--topics-payload`|JSON định nghĩa nhiều topic/payload.|
|`-q, --qos`|QoS 0/1/2.|
|`-r, --retain`|Gửi retain message.|
|`-L, --limit`|**Giới hạn số message / client** (0 = vô hạn).|
|`-F, --inflight`|Số in-flight tối đa cho QoS 1/2 (0 = infinity).|
|`-w, --wait-before-publishing`|Đợi **tất cả publisher** (ít nhất thử connect) rồi mới bắt đầu publish.|
|`--max-random-wait`, `--min-random-wait`|Random delay trước khi publish (uniform).|
|`--retry-interval`|Thời gian retry nếu QoS1/2 không nhận được ack (0 = không retry).|
|`--payload-hdrs`|Thêm header vào payload: `cnt64` (counter 64-bit), `ts` (timestamp), ...|

---

## 3) **Phân biệt 3 loại “interval/rate” dễ nhầm**

- `-i / --interval` (cả `conn` & `pub`): **khoảng cách giữa 2 lần tạo client**
    
- `-R / --connrate` (cả `conn` & `pub`): **tốc độ tạo kết nối / giây** (ưu tiên hơn `-i`)
    
- `-I / --interval_of_msg` (chỉ `pub`): **khoảng cách publish giữa 2 message** của **mỗi client**
    

---

## 4) **Một mớ câu lệnh mẫu** cho các tình huống hay gặp

### 4.1. **Kịch bản: Sticky theo client_id qua HAProxy (TLS terminate ở HAProxy, EMQX 1883)**

```bash
# 1) Connect-only để kiểm tra sticky dính vào 1 pod
emqtt_bench conn \
  -h demo-emqx-haproxy -p 8883 \
  -c 500 -R 100 \
  --prefix stickytest \
  --shortids false

# 2) Publish QoS0, 100 msg/s mỗi client, 1000 clients
emqtt_bench pub \
  -h demo-emqx-haproxy -p 30071 \
  -c 1000 -R 200 \
  -t "load/%c" -q 0 -s 128 \
  -I 10 \
  --prefix q0_sticky
```

> Với terminate TLS ở HAProxy → nếu HAProxy “ăn” TLS chuẩn, có thể **không cần `-S`**. Nếu đi thẳng EMQX 8883 → cần `-S`.

---

### 4.2. **TLS trực tiếp ở EMQX (port 8883)**

```bash
emqtt_bench conn \
  -h demo-emqx -p 8883 \
  -S \
  -c 1000 -R 200 \
  --prefix tls_direct
```

---

### 4.3. **WebSocket / WSS**

```bash
# WS (port 8083)
emqtt_bench pub \
  -h demo-emqx -p 8083 \
  --ws \
  -c 200 -R 50 \
  -t 'ws/%i' -I 100 -s 200

# WSS (port 8084) – terminate tại EMQX
emqtt_bench pub \
  -h demo-emqx -p 8084 \
  --ws -S \
  -c 200 -R 50 \
  -t 'wss/%i' -I 100 -s 200
```

---

### 4.4. **QoS1/2 + inflight tuning**

```bash
# QoS1, inflight = 100, 200 clients, 50 msg/s mỗi client
emqtt_bench pub \
  -h demo-emqx-haproxy -p 30070 \
  -c 200 -R 50 \
  -q 1 -F 100 \
  -I 20 -s 256 \
  --prefix qos1_bench

# QoS2, inflight = infinity (0), giới hạn 10k msg / client
emqtt_bench pub \
  -h demo-emqx-haproxy -p 30070 \
  -c 100 -R 20 \
  -q 2 -F 0 \
  -L 10000 \
  -I 50 -s 512 \
  --prefix qos2_blast
```

---

### 4.5. **Reconnect storm / session expiry (MQTT v5)**

```bash
# Tạo 500 client persistent session, session-expiry = 1h
emqtt_bench conn \
  -h demo-emqx-haproxy -p 30070 \
  -c 500 -R 100 \
  -V 5 -x 3600 \
  --prefix sess_persist

# (Sau đó kill bench rồi chạy lại với cùng prefix + shortids để simulate reconnect)
emqtt_bench conn \
  -h demo-emqx-haproxy -p 30070 \
  -c 500 -R 200 \
  -V 5 -x 3600 \
  --prefix sess_persist --shortids true
```

---

### 4.6. **Stress connect rate (Kubernetes – check HPA, EMQX acceptors)**

```bash
emqtt_bench conn \
  -h demo-emqx-haproxy -p 30070 \
  -c 10000 -R 2000 \
  --prefix flood_conn
```

---

### 4.7. **Expose metrics Prometheus từ chính emqtt_bench**

```bash
emqtt_bench pub \
  -h demo-emqx-haproxy -p 30070 \
  -c 200 -R 50 \
  -t bench/%i -I 100 \
  --prometheus --restapi 0.0.0.0:9091
```

---

## 5) **Vài mẹo nhỏ để test “đúng vấn đề”**

1. **Muốn test sticky theo client_id** → **set `--prefix` (để ID ổn định) và **tăng `len` stick-table đủ dài** (64/128).
    
2. **Không kiểm soát được client_id (random)** → **đừng sticky theo client_id**, cân bằng theo `leastconn` thuần sẽ “thật” hơn.
    
3. **TLS terminate ở proxy** → client **không cần `-S`** (tùy tool), EMQX listener 1883 là OK.
    
4. **TLS ở EMQX** → client **phải `-S`** & gửi thẳng 8883 (hoặc 30071 nếu HAProxy passthrough).
    
5. **Muốn kiểm tra cluster phân phối đều** → `emqx ctl clients list` trên từng pod, đếm số client / prefix.
    
6. **Muốn kiểm chứng HAProxy thực sự sticky** → `show table` qua runtime API (`stats socket`).
    
7. **Muốn đo throughput thực sự** → dùng `pub` với `-I` nhỏ, QoS0 (ít overhead), rồi dùng `emqx ctl metrics` / Dashboard để xem.
    
8. **Muốn thử resilience** (disconnect/reconnect) → dùng `--reconnect`, `--num-retry-connect`, `-x` (session expiry) + kill bớt pod EMQX/Haproxy.
    

---

Cần mình viết sẵn **bộ script chạy – thu log – parse kết quả (client phân bổ theo pod) tự động** không? Hô một tiếng mình quăng luôn, khỏi phải grep tay từng pod nữa. 💥