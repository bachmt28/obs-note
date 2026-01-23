/etc/rsyslog.d/30-gitlab.conf

```conf

```


Ngon nhưng chưa “chuẩn chỉ” lắm, Đại Nguyên Soái. Cấu hình hiện tại **chạy được**, nhưng tôi khuyến nghị vài chỉnh sửa để ổn định hơn, hợp chuẩn gửi về SIEM (QRadar), và tránh lỗi “lặt vặt”:

## Nhận xét nhanh về bản hiện tại

* `imfile` OK, chỉ định `Facility/Severity` theo từng log → hợp lý.
* Dòng `*.* @@siem.seabank.com.vn:514` → gửi **TCP**; OK nhưng là cú pháp **legacy**. Nên dùng `omfwd` (RainerScript) để:

  * Chọn **format RFC5424** (QRadar đọc tốt).
  * Bật **octet-counted framing** cho TCP ổn định.
  * Cấu hình **action queue** (buffer, retry) để tránh mất log khi mạng chập chờn.
* `Tag` đang có dấu “`:`”. Trong rsyslog, template mặc định đã tự thêm `:` sau programname; đặt `Tag` kèm `:` dễ thành **“double colon”**. → Nên bỏ `:` trong `Tag`.
* Với log có khả năng **multi-line** (Rails `production.log`), cân nhắc `escapeLF="on"` hoặc chế độ đọc gom dòng (nếu cần).
* Nên tăng `PersistStateInterval` để giảm nguy cơ đọc trùng sau restart.

## Bản cấu hình đề xuất (chuẩn RainerScript, khuyến nghị)

```conf
#### Modules
module(load="imuxsock")            # nhận log từ local syslog() nếu có
module(load="imfile")              # đọc log từ file
# module(load="imjournal")         # nếu dùng systemd-journald

#### Templates
# Dùng RFC5424 (RSYSLOG_SyslogProtocol23Format) để QRadar parse tốt
# Có thể dùng mặc định RSYSLOG_SyslogProtocol23Format nên không cần define thêm.

#### Inputs (imfile)
# GitLab audit
input(type="imfile"
      File="/var/log/gitlab/gitlab-rails/audit_json.log"
      Tag="gitlab-audit"
      StateFile="gitlab-audit.state"
      Severity="info"
      Facility="local0"
      PersistStateInterval="100"
      escapeLF="on")

# GitLab authentication
input(type="imfile"
      File="/var/log/gitlab/gitlab-rails/auth_json.log"
      Tag="gitlab-auth"
      StateFile="gitlab-auth.state"
      Severity="info"
      Facility="local1"
      PersistStateInterval="100"
      escapeLF="on")

# Rails production (có thể multi-line)
input(type="imfile"
      File="/var/log/gitlab/gitlab-rails/production.log"
      Tag="gitlab-production"
      StateFile="gitlab-production.state"
      Severity="info"
      Facility="local2"
      PersistStateInterval="100"
      escapeLF="on")

# NGINX access
input(type="imfile"
      File="/var/log/gitlab/nginx/gitlab_access.log"
      Tag="gitlab-nginx-access"
      StateFile="gitlab-nginx-access.state"
      Severity="info"
      Facility="local3"
      PersistStateInterval="100")

# NGINX error
input(type="imfile"
      File="/var/log/gitlab/nginx/gitlab_error.log"
      Tag="gitlab-nginx-error"
      StateFile="gitlab-nginx-error.state"
      Severity="warning"
      Facility="local4"
      PersistStateInterval="100")

#### Action: Forward về QRadar bằng omfwd (TCP + octet-counted + queue)
action(
  type="omfwd"
  target="siem.seabank.com.vn"
  port="514"
  protocol="tcp"
  template="RSYSLOG_SyslogProtocol23Format"
  TCP_Framing="octet-counted"
  KeepAlive="on"

  # Hàng đợi chống mất log
  action.resumeRetryCount="-1"      # retry vô hạn khi đích down
  queue.type="LinkedList"
  queue.size="100000"               # tùy dung lượng
  queue.dequeuebatchsize="1000"
  queue.highwatermark="80000"
  queue.lowwatermark="20000"
  queue.discardmark="95000"
  queue.discardseverity="0"         # chỉ discard khi quá tải nặng
)
```

### Vì sao nên chuyển sang `omfwd`?

* Dễ kiểm soát **format** (RFC5424), **framing** (octet-counted) – QRadar parse chuẩn, ít rủi ro “kẹt dòng”.
* Có **queue** + **retry** → an toàn khi SIEM/ mạng gián đoạn (tránh mất log).
* `*.* @@host:514` vẫn chạy, nhưng thiếu các cơ chế cực kỳ hữu ích trên môi trường prod.

## Tuỳ chọn nâng cao (nếu cần)

* **TLS** đến QRadar (nếu chính sách yêu cầu): `StreamDriver="gtls"`, `StreamDriverMode="1"`, `StreamDriverAuthMode="x509/name"` + certs. (Cần cấu hình CA/cert đôi bên.)
* **Gom multi-line** cho Rails log:

  * Đơn giản nhất: `escapeLF="on"` (đã bật ở trên) → xuống dòng thành `\n` (QRadar vẫn đọc dễ).
  * Nếu muốn gom khối thực sự, cần `readMode="2"` + `startmsg.regex`/`endmsg.regex` (phức tạp hơn, chỉ làm khi thật sự cần).

## Checklist nhanh để “đóng máy”

* Quyền đọc file log cho user chạy rsyslog (`root` thì OK; nếu non-root, cần group/ACL).
* SELinux: đảm bảo context/boolean cho rsyslog đọc các file trong `/var/log/gitlab/...`.
* Mở firewall nội bộ đến `siem.seabank.com.vn:514/tcp`.
* Test:

  ```bash
  logger -p local0.info "rsyslog test to QRadar"
  # và tạo một dòng giả trong audit_json.log để xem có lên SIEM
  echo '{"event":"test_audit","ts":"'"$(date -Is)"'"}' | sudo tee -a /var/log/gitlab/gitlab-rails/audit_json.log
  ```
* Theo dõi `/var/log/rsyslog` (hoặc `journalctl -u rsyslog`) xem có lỗi queue/forward.

---

### Kết luận

* Bản của Ngài **chạy được**; chỉ cần:

  1. Bỏ dấu `:` ở `Tag`,
  2. Chuyển forward sang `omfwd` với RFC5424 + `TCP_Framing=octet-counted` + **queue**,
  3. Bật `escapeLF` cho log có thể multi-line,
  4. Tăng `PersistStateInterval`.

Làm vậy là “chuẩn chiến”, đạn lên nòng, không rơi log dù mạng có “hắt hơi sổ mũi”.
