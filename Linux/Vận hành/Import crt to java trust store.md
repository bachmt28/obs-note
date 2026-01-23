Java 8/11/17/21 (OpenJDK) luôn có truststore chuẩn:

```
$JAVA_HOME/lib/security/cacerts
```

Ở đây, `$JAVA_HOME` = `/usr/lib/jvm/java-21-openjdk-21.0.3.0.9-1.el8.x86_64`.

---
# 🔎 **Xác minh 

Có thể kiểm tra trực tiếp:

```
ls -l /usr/lib/jvm/java-21-openjdk-21.0.3.0.9-1.el8.x86_64/lib/security/
```

Sẽ thấy:

```
cacerts
blacklisted.certs
blocked.certs
java.policy
java.security
```

Kiểm tra nội dung keystore:

```
keytool -list -keystore /usr/lib/jvm/java-21-openjdk-21.0.3.0.9-1.el8.x86_64/lib/security/cacerts \
  -storepass changeit | head
```

---
# 📌 Import CA vào keystore:

```sh
JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.362.b09-2.el8_7.x86_64
CERT_FILE=/root/sigproxy-dtu.crt
ALIAS_NAME=sigproxy-dtu
keytool -importcert \
  -alias ${ALIAS_NAME} \
  -keystore ${JAVA_HOME}/lib/security/cacerts \
  -storepass changeit \
  -file ${CERT_FILE} \
  -noprompt
  
  
keytool -importcert \
  -alias ${ALIAS_NAME} \
  -storepass changeit \
  -file ${CERT_FILE} 
```

---
# 🧠 Ghi chú quan trọng

1. **Keystore này thuộc về JVM**, không liên quan tới OS truststore.
2. Jenkins luôn dùng**JVM** → nên phải import CA vào **cacerts của JVM**.
3. Nếu Jenkins chạy trong container → phải import vào cacerts bên trong **container image**, không phải host.
---
Chuẩn, hỏi câu này là đúng “điểm huyệt” luôn 👍

---

## 1. Trỏ `-Djavax.net.ssl.trustStore=...` thì chuyện gì xảy ra?

Khi ông set:

```text
-Djavax.net.ssl.trustStore=/opt/jenkins/sigproxy-truststore.jks
-Djavax.net.ssl.trustStorePassword=...
```

thì:

* **Java sẽ bỏ qua `cacerts` mặc định** của nó.
* Từ giờ **toàn bộ TLS trong JVM đó** (Jenkins + plugin) sẽ chỉ dùng **cái truststore ông chỉ định**.

Tức là:

> Có **1 truststore “active”** cho JVM, không phải nhiều cái song song (trừ khi code tự chơi SSLContext riêng).

Nên nếu truststore riêng của ông **chỉ chứa mỗi CA Cisco** → JVM sẽ **không còn trust các CA public mặc định** nữa → đi ra internet, gọi GitHub, plugin update… có thể vỡ mồm.

---

## 2. Vậy làm sao “an toàn” mà vẫn giữ default CA?

Chiến lược an toàn là:

> **Không đụng vào file `cacerts` gốc, mà clone nó ra một bản riêng rồi thêm CA mới vào.**
> Jenkins sẽ dùng bản clone đó: vẫn có toàn bộ CA mặc định + thêm Cisco.

Cụ thể:

```bash
# 1. Clone cacerts gốc
sudo cp \
  /etc/pki/java/cacerts \
  /var/lib/jenkins/jenkins-truststore.jks

# 2. Import thêm Cisco CA vào bản clone
sudo keytool -importcert \
  -alias sigproxy-dtu \
  -file sigproxy-dtu.crt \
  -keystore /var/lib/jenkins/jenkins-truststore.jks \
  -storepass changeit \
  -noprompt
```

Sau đó cho Jenkins chạy với:

`/etc/systemd/system/jenkins.service.d/override.conf

```
[Service]
Environment="JENKINS_JAVA_OPTIONS=-Djavax.net.ssl.trustStore=/var/lib/jenkins/jenkins-truststore.jks -Djavax.net.ssl.trustStorePassword=changeit"

```
Kết quả:

* **Không chạm vào** file `cacerts` gốc (đỡ lo hỏng + update hệ điều hành đè lên).
* Jenkins vẫn **trust toàn bộ CA mặc định** (vì file clone có sẵn).
* Đồng thời **có thêm CA Cisco**.

Đó mới là “chơi an toàn mà vẫn đủ lửa”.

---

## 3. Java có dùng được “nhiều truststore” cùng lúc không?

### Trả lời thẳng:

* Ở level **JVM + system properties**: **KHÔNG**.
  Chỉ có **1 cặp**:

  ```text
  javax.net.ssl.trustStore
  javax.net.ssl.trustStorePassword
  javax.net.ssl.trustStoreType
  ```

* Nếu để **trống** → Java sẽ dùng **default** (`$JAVA_HOME/lib/security/cacerts`).

* Nếu set → **hoàn toàn override** default, không merge tự động.

Muốn chơi kiểu “nhiều truststore” thì phải:

* Viết code Java tạo `SSLContext` riêng,
* Tự tạo `TrustManager` merge từ nhiều `KeyStore`.

Nhưng:

* Jenkins core + plugin đa số **không expose** chỗ đó cho mình can thiệp.
* Thực tế triển khai Jenkins: **luôn coi như chỉ có 1 truststore** → cách chuẩn là **merge CA vào một file** như bước trên.

---

## 4. Tóm tắt mệnh lệnh chiến lược

* **Không trỏ thẳng** đến truststore “trọc” chỉ chứa Cisco → sẽ mất hết root CA khác.
* Làm như sau:

  1. Copy `cacerts` gốc → `/opt/jenkins/jenkins-truststore.jks`
  2. Import thêm mọi CA custom (Cisco, nội bộ, v.v.) vào **file này**.
  3. Cho Jenkins dùng:

     ```text
     -Djavax.net.ssl.trustStore=/opt/jenkins/jenkins-truststore.jks
     -Djavax.net.ssl.trustStorePassword=changeit
     ```

→ Vừa **giữ lại toàn bộ trust mặc định**, vừa **bổ sung CA riêng**, lại không sợ phá JVM system.

Nếu ông muốn, ta có thể soạn luôn một đoạn drop-in systemd override cho `jenkins.service` để set mấy cái `JENKINS_JAVA_OPTIONS` chuẩn chỉnh, copy phát là chạy.
