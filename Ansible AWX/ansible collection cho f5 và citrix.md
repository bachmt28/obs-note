## List module của ansible đối với f5
[https://docs.ansible.com/ansible/latest/collections/f5networks/f5\_modules/index.html](https://docs.ansible.com/ansible/latest/collections/f5networks/f5_modules/index.html)

## Các module có liên quan đến dns
### 1. [`bigip_device_dns`](https://docs.ansible.com/ansible/latest/collections/f5networks/f5_modules/bigip_device_dns_module.html)
> **Cấu hình hệ thống DNS resolver nội bộ của thiết bị F5**.

* Tác động tới: `/sys/dns`
* Cho phép cấu hình:
   * `name_servers` → list IP DNS để thiết bị truy vấn
   * `search` → search domain suffix

* ❌ Không liên quan đến record DNS (A/CNAME).
* ✅ Thường dùng khi F5 cần truy vấn DNS ra ngoài (chứ không phải để phục vụ DNS cho client).

---
### 2. [`bigip_dns_cache_resolver`](https://docs.ansible.com/ansible/latest/collections/f5networks/f5_modules/bigip_dns_cache_resolver_module.html)
> **Tạo cache DNS resolver object trên F5** để dùng trong profile DNS.

* Tác động tới: `ltm dns cache resolver`
* Cho phép F5 thực hiện **DNS caching** khi forwarding query.
* Có thể cấu hình DNS forwarder, TTL, enable/disable.
* ❌ Không liên quan đến DNS record.
* ✅ Phục vụ mục đích cache DNS lookup trong profile forwarding hoặc resolving.

---
### 3. [`bigip_dns_nameserver`](https://docs.ansible.com/ansible/latest/collections/f5networks/f5_modules/bigip_dns_nameserver_module.html)
> **Tạo đối tượng "nameserver" bên trong cấu hình DNS profile**.

* Tác động tới: `ltm dns nameserver`
* Xác định IP nào là DNS server mà profile DNS dùng để forward.
* ❌ Không tạo/đổi record DNS.
* ✅ Dùng để cấu hình đường đi của DNS query nếu bạn setup forwarding chain.

```yaml
nodeSelector:
	app: t24
tolerations:
- key: app
  operator: Equal
  value: non-system
  effect: NoSchedule
```