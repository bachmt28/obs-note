# **Kế hoạch (Airgap Upgrade Procedure)**

## **Mục tiêu**
* Nâng cấp GitLab lên phiên bản **latest khả dụng cao nhất** để:
  * Vá toàn bộ các lỗ hổng bảo mật (vulnerability fix).
  * Đảm bảo tương thích và ổn định cho hệ thống CI/CD và tích hợp AD.
## **Fix vuls**
| CVE               | Mô tả                                                                                                                                                                                                                                                                        |
| ----------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **CVE‑2025‑0549** | Bypass flow OAuth thiết bị (Device OAuth flow) – người tấn công có thể khởi tạo form ủy quyền mà ít tương tác người dùng. ([Tenable®](https://www.tenable.com/plugins/nessus/235666?utm_source=chatgpt.com "GitLab 17.3 < 17.9.8 / 17.10 < 17.10.6 / 17.11 < 17.11.2 (CVE")) |
| **CVE‑2025‑1278** | Bỏ qua hạn chế IP nhóm – người tấn công có thể truy vấn tiêu đề issue của dự án bị hạn chế IP. ([Tenable®](https://www.tenable.com/plugins/nessus/235665?utm_source=chatgpt.com "GitLab 12.0 < 17.9.8 / 17.10 < 17.10.6 / 17.11 < 17.11.2 (CVE"))                            |
| **CVE‑2025‑5121** | Trong bản Ultimate EE: người dùng đã xác thực có thể chèn job CI/CD độc hại vào tất cả pipeline tương lai. ([Tenable®](https://www.tenable.com/plugins/nessus/240212?utm_source=chatgpt.com "GitLab 17.11 < 17.11.4 / 18.0 < 18.0.2 (CVE-2025-5121) \| Tenable®"))           |
| **CVE‑2025‑5996** | DoS qua xử lý response HTTP không kiểm soát. ([Tenable®](https://www.tenable.com/plugins/nessus/238313?utm_source=chatgpt.com "GitLab 2.10 < 17.10.7 / 17.11 < 17.11.3 / 18.0 < 18.0.1 (CVE-2..."))                                                                          |
| **CVE‑2025‑6948** | XSS mức cao trong GitLab CE/EE ảnh hưởng bản từ 17.11 trước 17.11.6. ([about.gitlab.com](https://about.gitlab.com/releases/2025/07/09/patch-release-gitlab-18-1-2-released/?utm_source=chatgpt.com "GitLab Patch Release: 18.1.2, 18.0.4, 17.11.6"))                         |
## **Phạm vi**
Áp dụng cho 2 hệ thống GitLab nội bộ:
* **gitlab-internal**: Hiện tại ở phiên bản `17.11.2` trên OS Oracle Linux 8.
* **gitlab-partner**: Hiện tại ở phiên bản `16.10.1` trên OS Oracle Linux 7.
## **Nguyên tắc nâng cấp**

1. **Tuân thủ chuỗi nâng cấp tuần tự (Minor Upgrade Path)** mà GitLab khuyến cáo, không nhảy vượt quá 1 version chính.
   → Ví dụ: `17.11.x → 18.0.x → 18.1.x → ... → 18.5.x`
2. **Thực hiện nâng cấp từng bước**, kiểm tra thành công từng giai đoạn trước khi chuyển bước tiếp.
3. **Sử dụng gói RPM tải thủ công** từ máy có internet, chuyển vào hệ thống airgap qua phương thức nội bộ (USB, SCP, v.v.).
4. **Kiểm tra tính toàn vẹn hệ thống** sau mỗi lần nâng cấp, đặc biệt xác minh:
   * Đăng nhập AD.
   * Hoạt động webhook giữa GitLab ↔ Jenkins.
---
## **I. Nâng cấp `gitlab-internal`**
### 1. Chuẩn bị gói RPM
Trên máy có internet, tải về các bản GitLab CE dành cho **OL8 (Oracle Linux 8)** theo chuỗi nâng cấp khuyến nghị:
```bash
VERSIONS="
17.11.7
18.0.6
18.1.6
18.2.8
18.3.4
18.4.2
18.5.0
"

for ver in $VERSIONS; do
  url="https://packages.gitlab.com/gitlab/gitlab-ce/packages/ol/8/gitlab-ce-${ver}-ce.0.el8.x86_64.rpm/download.rpm"
  outfile="gitlab-ce-${ver}-ce.0.el8.x86_64.rpm"
  echo ">> Downloading GitLab CE ${ver} ..."
  wget $url -O $outfile --verbose
done
```

> ✅ **Lưu ý:** Kiểm tra checksum SHA256 của từng file sau khi tải để tránh lỗi trong môi trường offline.
### 2. Chuyển gói vào máy mục tiêu
Copy toàn bộ `.rpm` vào máy GitLab nội bộ (qua USB hoặc kênh nội bộ).
### 3. Thực hiện nâng cấp tuần tự
```bash
rpm -Uvh gitlab-ce-17.11.7-ce.0.el8.x86_64.rpm && \
rpm -Uvh gitlab-ce-18.0.6-ce.0.el8.x86_64.rpm && \
rpm -Uvh gitlab-ce-18.1.6-ce.0.el8.x86_64.rpm && \
rpm -Uvh gitlab-ce-18.2.8-ce.0.el8.x86_64.rpm && \
rpm -Uvh gitlab-ce-18.3.4-ce.0.el8.x86_64.rpm && \
rpm -Uvh gitlab-ce-18.4.2-ce.0.el8.x86_64.rpm && \
rpm -Uvh gitlab-ce-18.5.0-ce.0.el8.x86_64.rpm
```

> ⚙️ Mỗi bước `rpm -Uvh` sẽ tự động chạy `gitlab-ctl reconfigure` và khởi động lại các service cần thiết.
> Nên đợi hoàn tất trước khi chạy bước tiếp theo.
### 4. Kiểm tra hậu nâng cấp

* Đăng nhập bằng tài khoản **Active Directory** → xác nhận kết nối LDAP/SSO hoạt động.
* Thực hiện một **webhook thử nghiệm** tới Jenkins để đảm bảo token và API còn hợp lệ.
* Kiểm tra lại tình trạng hệ thống:
  ```bash
  sudo gitlab-rake gitlab:check SANITIZE=true
  sudo gitlab-ctl status
  ```
---
## **II. Nâng cấp `gitlab-partner`**
### 1. Chuẩn bị gói RPM
Trên máy có internet, tải các bản GitLab CE cho **OL7 (Oracle Linux 7)**:
```bash
VERSIONS="
16.10.10
16.11.10
17.0.8
17.1.8
17.2.9
17.3.7
17.4.6
17.5.5
17.6.5
17.7.7
17.8.7
17.9.8
17.10.8
17.11.7
18.0.6
18.1.6
18.2.8
18.3.4
18.4.2
18.5.0
"

for ver in $VERSIONS; do
  url="https://packages.gitlab.com/gitlab/gitlab-ce/packages/ol/7/gitlab-ce-${ver}-ce.0.el8.x86_64.rpm/download.rpm"
  outfile="gitlab-ce-${ver}-ce.0.el8.x86_64.rpm"
  echo ">> Downloading GitLab CE ${ver} ..."
  wget $url -O $outfile --verbose
done
```
### 2. Chuyển RPM vào máy mục tiêu

Upload toàn bộ `.rpm` vào máy GitLab partner.
### 3. Thực hiện nâng cấp tuần tự

```bash
rpm -Uvh gitlab-ce-16.10.10-ce.0.el8.x86_64.rpm && \
rpm -Uvh gitlab-ce-16.11.10-ce.0.el8.x86_64.rpm && \
rpm -Uvh gitlab-ce-17.0.8-ce.0.el8.x86_64.rpm && \
rpm -Uvh gitlab-ce-17.1.8-ce.0.el8.x86_64.rpm && \
rpm -Uvh gitlab-ce-17.2.9-ce.0.el8.x86_64.rpm && \
rpm -Uvh gitlab-ce-17.3.7-ce.0.el8.x86_64.rpm && \
rpm -Uvh gitlab-ce-17.4.6-ce.0.el8.x86_64.rpm && \
rpm -Uvh gitlab-ce-17.5.5-ce.0.el8.x86_64.rpm && \
rpm -Uvh gitlab-ce-17.6.5-ce.0.el8.x86_64.rpm && \
rpm -Uvh gitlab-ce-17.7.7-ce.0.el8.x86_64.rpm && \
rpm -Uvh gitlab-ce-17.8.7-ce.0.el8.x86_64.rpm && \
rpm -Uvh gitlab-ce-17.9.8-ce.0.el8.x86_64.rpm && \
rpm -Uvh gitlab-ce-17.10.8-ce.0.el8.x86_64.rpm && \
rpm -Uvh gitlab-ce-17.11.7-ce.0.el8.x86_64.rpm && \
rpm -Uvh gitlab-ce-18.0.6-ce.0.el8.x86_64.rpm && \
rpm -Uvh gitlab-ce-18.1.6-ce.0.el8.x86_64.rpm && \
rpm -Uvh gitlab-ce-18.2.8-ce.0.el8.x86_64.rpm && \
rpm -Uvh gitlab-ce-18.3.5-ce.0.el8.x86_64.rpm && \
rpm -Uvh gitlab-ce-18.4.3-ce.0.el8.x86_64.rpm && \
rpm -Uvh gitlab-ce-18.5.1-ce.0.el8.x86_64.rpm
```
### 4. Kiểm tra hậu nâng cấp

* Đăng nhập bằng tài khoản AD → xác nhận hoạt động bình thường.
* Kiểm tra lại **tổng số dự án** hiển thị đúng.
* Do hệ thống này chỉ phục vụ đối tác trung chuyển → chỉ cần xác nhận kết nối và dự án không mất dữ liệu.
---
# **Sao lưu và phục hồi**

## **I. Phục hồi bằng cách revert từ snapshot của vm**
1. Trước khi nâng cấp thực hiệp snapshot vm 
2. Nếu upgrade thất bại, lập tức khôi phục lại từ snapshot của vm
## **II. Phục hồi bằng phương pháp import lại từ dump database**
1. ***Note: Phương pháp này chỉ phải dùng đến khi revert từ snapshot của vm bị crash***
2. Dữ liệu trước khi nâng cấp đã được dump ra nfs hàng ngày
3. Thực hiện cài đặt lại phiên bản trước khi nâng cấp 
4. Import lại datadump
5. Các bước chi tiết như sau đây: 

### Chuẩn bị
* Sao chép file bí mật (`gitlab-secrets.json` hoặc tương ứng) từ môi trường gốc vào máy mới – vì thiếu sẽ gây mất quyền người dùng 2FA, Runner không đăng nhập được. ([GitLab Docs][1])
### Thực thi khôi phục (ví dụ với gói Omnibus Linux)

* Copy file backup `.tar` vào đúng thư mục `backup_path` đã cấu hình (thường `/var/opt/gitlab/backups`). Set quyền sở hữu cho user `git`. 

  ```bash
  sudo cp <backup-file>.tar /var/opt/gitlab/backups/
  sudo chown git:git /var/opt/gitlab/backups/<backup-file>.tar
  ```
* Dừng các tiến trình kết nối DB (ví dụ `puma`, `sidekiq`), để đảm bảo quá trình restore an toàn:
  ```bash
  sudo gitlab-ctl stop puma
  sudo gitlab-ctl stop sidekiq
  sudo gitlab-ctl status
  ```
- Chạy lệnh khôi phục, chỉ định `BACKUP=<backup-ID>` (không gồm `_gitlab_backup.tar`):
  ```bash
  sudo gitlab-backup restore BACKUP=<backup-ID>
  ```
* Sau khi hoàn tất: chạy `sudo gitlab-ctl start` và kiểm tra hệ thống:

```bash
sudo gitlab-rake gitlab:check SANITIZE=true
sudo gitlab-rake gitlab:doctor:secrets
```