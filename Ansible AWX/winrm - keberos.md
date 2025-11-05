## 📌 Mô hình:

```text
[Linux Ansible Controller]
        |
   (Kerberos ticket)
        ↓
[Windows Host đã JOIN DOMAIN]
```

---
## 🎯 Điều kiện bắt buộc

| Thành phần                    | Yêu cầu                                                                                  |
| ----------------------------- | ---------------------------------------------------------------------------------------- |
| 🖥 Windows Target             | Đã **join domain AD**, có WinRM listener HTTP (5985), `Basic=False`, `Unencrypted=False` |
| 🐧 Ansible Controller (Linux) | Đã cài `krb5-user`, `pywinrm[kerberos]`, có thể `kinit`                                  |
| 📡 DNS                        | Resolve được FQDN → `win01.domain.local`                                                 |
| 🕒 Time Sync                  | Cả 2 bên phải đồng bộ NTP hoặc gần giờ nhau (<5 phút)                                    |

---

## 🧱 Triển khai WinRM + Kerberos – Checklist

### 🖥 Trên Windows Target (đã join domain)

- [x] Bật WinRM: `Enable-PSRemoting -Force`
    
- [x] Kiểm tra listener HTTP: `winrm enumerate winrm/config/listener`
    
    - [x] Đảm bảo `Transport = HTTP` và `Port = 5985`
        
- [x] Kiểm tra cấu hình xác thực: `winrm get winrm/config/service/auth`
    
    - [x] `Basic = false`
        
    - [x] `Kerberos = true`
        
    - [x] `AllowUnencrypted = false`
        
- [x] Mở firewall port 5985:
    
    ```powershell
    New-NetFirewallRule -Name "WinRM-HTTP-In" -DisplayName "WinRM HTTP" -Protocol TCP -LocalPort 5985 -Action Allow
    ```
---

### 🐧 Thực hiện binding với domain qua kerberos

#### ⚙️ Cài đặt & chuẩn bị môi trường
- [x] Sử dụng ansible-builder, Bổ sung các lib cần thiết và bindep.txt và build lại EE
```txt
# ─── Kerberos + GSSAPI ───────────────────────────────
krb5-workstation [platform:redhat]     # kinit, klist, runtime tools
krb5-devel        [platform:redhat]     # headers for kerberos build
libkrb5-dev       [platform:dpkg]       # same as krb5-devel on Debian
gss-ntlmssp       [platform:dpkg]       # optional for SPNEGO fallback (Debian only)

# ─── Oracle + system libraries ────────────────────────
libaio1           [platform:dpkg]
libaio            [platform:rpm]
libnsl            [platform:rpm]

# ─── Build tools ──────────────────────────────────────
gcc
python3-devel     [platform:redhat]     # needed for native Python extensions
build-essential   [platform:dpkg]       # Debian equivalent

# ─── Utility tools ────────────────────────────────────
unzip
dos2unix          [platform:rpm]
file              [platform:rpm]
```
#### ⚙️ Cấu hình động thông qa template ở runtime
- [x] Tạo template file `krb5.conf.j2` trong Project
```jinja
[libdefaults]
  default_realm = {{ kerberos_realm }}
  dns_lookup_realm = true
  dns_lookup_kdc = true

[realms]
  {{ kerberos_realm }} = {
    kdc = {{ kdc_hostname }}
    admin_server = {{ kdc_hostname }}
  }

[domain_realm]
  .{{ kerberos_domain }} = {{ kerberos_realm }}
```
- [x] Thêm pretask vào playbook
```yaml
- name: Generate krb5.conf
  become: true
  template:
    src: krb5.conf.j2
    dest: /etc/krb5.conf
    mode: '0644'
```
- [x] Các task verify
```yaml
- name: 🔐 Lấy vé Kerberos
  shell: echo '{{ ansible_password }}' | kinit '{{ ansible_user }}'
  register: kinit_result
  failed_when: "'kinit:' in kinit_result.stderr"
  changed_when: false

- name: 🧾 Kiểm tra vé đã có trong cache
  shell: klist
  register: klist_result
  changed_when: false
  failed_when: "'No credentials cache found' in klist_result.stderr"

- name: ✅ Hiển thị vé Kerberos
  debug:
    var: klist_result.stdout_lines
    
- name: 🔗 Ping host Windows qua Kerberos
  ansible.builtin.win_ping:
  delegate_to: localhost
```
- [x] Sử dụng var global trong cấu hình job template hoặc playbook
```yaml
kerberos_realm: DOMAIN.LOCAL
kdc_hostname: dc01.domain.local
kerberos_domain: domain.local
```
    
---

### 🧾 Cấu hình Ansible Inventory

```ini
[windows]
win01.domain.local

[windows:vars]
ansible_connection=winrm
ansible_winrm_transport=kerberos
ansible_winrm_port=5985
# Optional (nếu bị SPN mismatch):
# ansible_winrm_kerberos_hostname_override=win01.domain.local
```
---
### 🧨 Debug lỗi thường gặp

- [ ] `Server not found in Kerberos database` → SPN mismatch → kiểm tra DNS & FQDN
    
- [ ] `No credentials` → Quên `kinit` hoặc vé hết hạn
    
- [ ] `GSSAPI Error` → domain sai, chênh lệch thời gian, config Kerberos sai
    

---