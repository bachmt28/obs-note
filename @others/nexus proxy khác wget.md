OK, đính chính này **rất quan trọng**, và bạn nói đúng một nửa — nên tôi sẽ **bóc đến tận lõi**, không né.

> **wget được chạy ngay trên OS của server Nexus**,
> **vẫn qua proxy**,
> **vẫn tải 8.5GB OK**,
> **nhưng Nexus proxy thì treo**
> → nếu chỉ nói “user vs server” là **chưa đủ thuyết phục**.

Dưới đây là **lý do THỰC SỰ**, mang tính **kỹ thuật thuần**, không phải suy diễn policy chung chung.

---

## 1. Cùng máy ≠ cùng stack mạng ≠ cùng hành vi

Dù **cùng OS**, nhưng:

| wget                   | Nexus                      |
| ---------------------- | -------------------------- |
| Native binary          | JVM application            |
| OpenSSL / GnuTLS       | Java TLS (JSSE)            |
| Trực tiếp socket       | Apache HttpClient / Jetty  |
| Stream thuần           | Có buffering, retry, cache |
| Không cần giữ metadata | Phải ghi cache + checksum  |

👉 **Proxy không chỉ nhìn IP**, nó nhìn **hành vi kết nối**.

---

## 2. Điểm khác biệt mấu chốt: **HTTP CONNECT & streaming pattern**

### wget (bạn đang chạy)

* Dùng **HTTP CONNECT**
* Sau CONNECT → **TLS end-to-end**
* Proxy **chỉ relay byte stream**
* Không “hiểu” nội dung bên trong
* **Không giữ state phức tạp**

=> Với proxy, wget giống như:

> “một client tải file lớn bình thường”

---

### Nexus proxy (RAW/YUM proxy)

* Dùng **HTTP client cấp cao**
* Có thể:

  * resolve metadata
  * mở connection
  * giữ connection lâu
  * ghi file xuống blob store
  * tính checksum
* Proxy thấy:

  * **long-lived CONNECT**
  * **server pulling artifact**
  * **pattern giống mirror/cache**

=> Với proxy, Nexus giống:

> “một repository đang mirror content”

Đây là **phân loại hành vi**, không phải user hay server.

---

## 3. Điểm chí mạng: **proxy xử lý khác CONNECT vs non-CONNECT**

Đa số SWG / enterprise proxy có logic như sau:

### Với CONNECT “thuần”

* Cho qua
* Không DLP sâu
* Không giữ content lâu
* Ít state

### Với HTTP client “thông minh” phía server

* Có thể:

  * inspect header
  * delay response
  * enforce size/time
  * giữ stream để scan

👉 **wget = CONNECT thuần**
👉 **Nexus = CONNECT + HTTP semantics**

Kết quả bạn thấy:

* TLS handshake OK
* GET gửi OK
* nhưng **proxy không trả response header**
  → proxy **giữ stream để xử lý**, Nexus chờ, client chờ → treo

---

## 4. JVM + large streaming = điểm yếu kinh điển

Nexus chạy trên JVM, thường dùng:

* Apache HttpClient
* Async streaming
* Chunked transfer

Trong môi trường có SWG:

* Chunked + long stream
* JVM backpressure
* Proxy buffering

→ **dễ rơi vào trạng thái “không chết nhưng không chạy”**

wget thì:

* đọc → ghi → đọc → ghi
* không buffer phức tạp
* không checksum trong lúc tải

---

## 5. Bằng chứng gián tiếp rất mạnh từ log của bạn

Log curl qua Nexus cho thấy:

* TLS OK
* GET OK
* **KHÔNG CÓ RESPONSE HEADER**

Nếu là auth / block:

* proxy đã trả 401 / 403 / 407

Nhưng không.
→ proxy **đã chấp nhận request**, nhưng **chưa release response**.

Đây **không phải lỗi auth**, mà là **flow-control / inspection / buffering issue**.

wget không gặp vì:

* proxy coi đó là stream đơn giản
* không giữ content để xử lý

---

## 6. Tóm lại — câu trả lời chính xác nhất

> **wget và Nexus proxy dù chạy trên cùng một OS, cùng đi qua proxy, nhưng KHÔNG tạo ra cùng một kiểu kết nối.**

* wget:

  * CONNECT thuần
  * streaming thẳng
  * proxy gần như mù nội dung
* Nexus:

  * server-side HTTP client
  * giữ state
  * mirror-like behavior
  * kích hoạt cơ chế xử lý khác trong proxy

👉 **Proxy không chặn bạn**,
👉 **Proxy xử lý Nexus theo một “class traffic” khác**.

---

## 7. Kết luận cuối cùng (không còn vòng vo)

* Bạn **đã debug đúng**
* Nhận định:

  > “đấm = cơm rồi up hosted”
  > là **quyết định đúng kỹ thuật**, không phải workaround.

Trong enterprise có SWG:

* Artifact nhỏ → proxy OK
* Artifact lớn → **host nội bộ là chuẩn**

Nếu bạn muốn, tôi có thể:

* Viết **1 đoạn giải trình kỹ thuật ngắn (1 trang)** để đưa cho security/network
* Đưa **best practice Nexus + SWG** theo kinh nghiệm thực tế
* Gợi ý **tuning Nexus/Java** (dù khả năng fix triệt để là thấp)

Bạn hỏi đúng chỗ khó nhất của câu chuyện này.
