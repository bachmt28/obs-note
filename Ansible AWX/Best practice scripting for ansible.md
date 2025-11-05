 **“Tiêu chuẩn viết shell / PowerShell script”** – dùng để **làm chuẩn chung** cho toàn bộ team, nhằm:

- Đồng bộ cách viết giữa người và máy
    
- Tối ưu cho tự động hoá (Ansible, Jenkins, cron)
    
- Dễ bảo trì – dễ debug – dễ tái sử dụng
    
- Giảm lỗi – giảm trùng – giảm drama
    

---

# 🧭 **TIÊU CHUẨN VIẾT SHELL / POWERSHELL SCRIPT**

> 📌 **Mục tiêu cốt lõi:**  
> Viết script _dễ gọi_, _dễ log_, _dễ kiểm tra_, _khó sai_ – dành cho môi trường **production, CI/CD, Ansible call**, và vận hành thủ công.

---

## 🧱 1. **TỔNG QUAN KIẾN TRÚC SCRIPT**

|Thành phần|Mô tả|
|---|---|
|`CONFIG`|Khai báo biến, đường dẫn, tên process… ở đầu file|
|`FLAGS`|Hỗ trợ `--quiet`, `--dry-run`, `--help`, `--logfile`…|
|`MAIN LOGIC`|Luôn chia rõ `check`, `stop`, `start`, `report`|
|`OUTPUT`|Hỗ trợ `stdout` rõ ràng (người đọc) và `--quiet` cho máy|
|`EXIT CODE`|Phân loại 0/1/2 chuẩn DevOps|
|`LOGGING`|Có thể ghi vào file, nhưng không bao giờ in rác khi quiet|

---

## ⚙️ 2. **CÁC TIÊU CHUẨN CỤ THỂ**

### ✅ **2.1. Hỗ trợ flag `--quiet`**

- Mặc định: `echo` đầy đủ, nhiều log cho người xem
    
- Khi `--quiet`: **in dưới dạng `key=value`**, chỉ dành cho Ansible/CI dùng `register`
    

> ✨ Tại sao? Để tích hợp dễ, không cần `awk | grep | sed` rối rắm.

---

### ✅ **2.2. Trả về `exit code` rõ ràng**

|Mã|Ý nghĩa|
|---|---|
|`0`|✅ Thành công (hoặc không cần hành động)|
|`1`|⚠️ Có vấn đề nhẹ (đã fix, hoặc không ảnh hưởng)|
|`2`|❌ Lỗi nghiêm trọng (Ansible/CI nên fail)|
|`3+`|Dành riêng cho lỗi option, flag, logic khác|

> 📍 Dùng để `failed_when: rc > 1` trong Ansible hoặc fail Jenkins stage

---

### ✅ **2.3. Không hardcode `echo`**

- Không `echo` lung tung giữa dòng
    
- Dùng hàm `log()` để redirect đồng thời ra file nếu cần
    
- Dùng `QUIET=1` để chặn toàn bộ `echo`
    

---

### ✅ **2.4. Kiểm tra điều kiện kỹ càng**

- Không `cd` mà không kiểm tra `|| exit`
    
- Không `kill` khi PID rỗng
    
- Không `rm -rf` nếu biến không được xác định rõ
    

---

### ✅ **2.5. Cấu trúc output `--quiet` dạng:**

```sh
status=stopped
method=kill
pid=12345
error=none
```

- Tránh dùng JSON vì phải escape, nhưng nếu muốn chuẩn hơn có thể dùng:
    

```sh
echo "{\"status\": \"stopped\", \"pid\": \"$PID\"}"
```

---

### ✅ **2.6. Có thể `source` hoặc `exec`**

- Không dùng `exit` lung tung khi script có khả năng bị `source` trong script khác
    
- Nên return về `exit_code` rõ ràng cho script gọi xử lý
    

---

### ✅ **2.7. Tên file & biến rõ ràng**

|Thành phần|Tên ví dụ tốt|
|---|---|
|Script|`stop_application.sh`, `check_vip.sh`|
|Biến|`VIP_IP`, `LIST_W4_PID`, `BASE_DIR`, `QUIET`|

> ❌ Không đặt kiểu `a=1`, `b=stop`, hoặc `x=$(...)`

---

### ✅ **2.8. PowerShell dùng tương tự**

|Tiêu chuẩn Bash|Tương đương PowerShell|
|---|---|
|`--quiet`|`-Quiet`|
|`exit 0/1/2`|`exit 0/1/2`|
|`key=value`|`Write-Output "key=value"`|
|Logging|`Out-File -Append`|

---

## 💼 3. **GỢI Ý FORMAT SCRIPT CHUẨN**

```sh
#!/bin/sh

# === CONFIG ===
BASE_DIR="/path/to/folder"
PROCESS_NAME="abc"

# === FLAGS ===
QUIET=0
DRY_RUN=0

for arg in "$@"; do
  case "$arg" in
    --quiet) QUIET=1 ;;
    --dry-run) DRY_RUN=1 ;;
    --*) echo "error=unknown_flag:$arg" >&2; exit 3 ;;
  esac
done

# === LOG FUNCTION ===
log() {
  [ "$QUIET" -ne 1 ] && echo "$@"
}

# === MAIN LOGIC ===
PID=$(pgrep -f "$PROCESS_NAME")

if [ -n "$PID" ]; then
  if [ "$DRY_RUN" -eq 1 ]; then
    log "[DRY-RUN] Sẽ kill $PID"
  else
    kill "$PID"
  fi
  STATUS="killed"
else
  STATUS="already_stopped"
fi

# === OUTPUT ===
if [ "$QUIET" -eq 1 ]; then
  echo "status=$STATUS"
  echo "pid=$PID"
else
  log "✅ Trạng thái: $STATUS | PID: $PID"
fi

exit 0
```

---

## 🏁 4. **SUMMARY**

|Quy tắc|Tinh thần|
|---|---|
|Output có người và máy|`--quiet` là ranh giới|
|Exit code là dấu hiệu sống chết|Phản ánh trạng thái xử lý|
|Không script kiểu “chơi vui”|Vì production không cho thử lần hai|
|Càng rõ ràng – càng dễ tích hợp|Từ cron đến Ansible đến CI/CD|
|Người khác sẽ debug thay bạn|Hãy viết như thể 3h sáng họ phải đọc|

---