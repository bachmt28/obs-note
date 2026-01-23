# Ý tưởng triển khai “resume từ node X” trong AWX Workflow

## 1) Thêm **2 biến chung** cho mọi JT trong workflow
- `resume_from_carddrp_seq` – số thứ tự node muốn bắt đầu lại (do người launch chọn).
- `node_carddrp_seq` – số thứ tự (cố định) của **chính** JT/nút đó.
> Quy ước đơn giản: S1=10, S2=20, S3=30, S4=40…

Trong **Workflow Template**:
- Tạo **Survey** (hoặc Variables tại launch) với câu hỏi:
    - `resume_from_carddrp_seq` (default `0`, nghĩa là chạy tất cả).
- Ở **mỗi node (JT)**, đặt `extra_vars`:
    - `node_carddrp_seq: 10` (S1), `node_carddrp_seq: 20` (S2), v.v.
    - (Không cần sửa gì khác ở node).
## 2) Thêm 1 pre_task ngắn vào **đầu mỗi playbook** của các JT
Đặt ngay đầu play (trước các task thật):
```yaml
pre_tasks:
  - name: Short-circuit if this node is before resume point
    ansible.builtin.meta: end_play
    when: resume_from_carddrp_seq | default(0) | int > node_carddrp_seq | int
```
- Nếu `resume_from_carddrp_seq` > `node_carddrp_seq` ⇒ **kết thúc play ngay lập tức với exit 0 (OK)**.
- Nếu `resume_from_carddrp_seq` ≤ `node_carddrp_seq` ⇒ playbook chạy bình thường.
> `meta: end_play` kết thúc _thành công_, **không** fail, nên workflow coi node này là **success** và tiếp tục tới node sau.  
> Không “nuốt lỗi”: node tại điểm resume và sau đó vẫn chạy đúng logic; chỉ những node trước điểm resume là “OK nhanh”.
## 3) (Tuỳ chọn) Giao diện khi launch
Trong **Workflow Survey**, cho user chọn “Resume từ bước nào”:
- None (0) / S1 (10) / S2 (20) / S3 (30) / S4 (40)  
    → Survey map sang `resume_from_carddrp_seq`
---
# Ví dụ áp vào playbook

```yaml
- name: S1 - Check standby sync status
  hosts: "{{ target_hosts | default('myapp_group') }}"
  vars:
    node_carddrp_seq: 10
  pre_tasks:
    - meta: end_play
      when: resume_from_carddrp_seq | default(0) | int > node_carddrp_seq | int
  tasks:
    - import_role:
        name: database
        tasks_from: db_s1_check_standby_sync.yml

- name: Verify Source DB ready for switch over and Execute switch
  hosts: "{{ source_hosts | default('myapp_group') }}"
  vars:
    node_carddrp_seq: 20
  pre_tasks:
    - meta: end_play
      when: resume_from_carddrp_seq | default(0) | int > node_carddrp_seq | int
  tasks:
    - import_role:
        name: database
        tasks_from: db_s2_verify_swover.yml
    - import_role:
        name: database
        tasks_from: db_s3_do_switchover.yml

- name: S4 - Turn on db & Check new prime ready ok
  hosts: "{{ target_hosts | default('myapp_group') }}"
  vars:
    node_carddrp_seq: 40
  pre_tasks:
    - meta: end_play
      when: resume_from_carddrp_seq | default(0) | int > node_carddrp_seq | int
  tasks:
    - import_role:
        name: database
        tasks_from: db_s4_verify_new_prim_ready.yml
```

(Trong Workflow node, set `node_carddrp_seq` tương ứng bằng `extra_vars`; khi launch, nhập `resume_from_carddrp_seq` = 20 nếu muốn “bắt đầu” từ S2/S3 chẳng hạn.)

---
## Thoả mãn được các điều kiện
- **Không đổi status giả**: node trước điểm resume vẫn **success thật** (do end_play), nhưng không chạy nội dung nặng.
- **Không fallback bừa / không nuốt lỗi**: chỉ skip các node trước điểm resume; từ node fail trở đi chạy đúng logic gốc.
- **Không phải chia workflow vụn**: một survey + mỗi JT gán 1 con số là xong.
- **Nhanh**: skip diễn ra ngay đầu play, tiết kiệm thời gian.

```yaml
sq_id:
  planA: 1
  planB: 40
  planC: 15
```
## Trường hợp 1 JT tái sử dụng cho nhiều case khác nhau 
1. Ở **Job Template** (JT),  khai báo **một dict** `sq_id` như  nói (nhớ dùng _spaces_ thay vì tab trong YAML):

```yaml
# JT extra_vars (hoặc defaults trong playbook của JT)
sq_id:
  planA: 1
  planB: 40
  planC: 15
```
2. Ở **Workflow Template** cho “kế hoạch” cụ thể (ví dụ planA),  chỉ cần **truyền 1 biến**:
```yaml
plan_id: "planA"
```
3. Trong **playbook** của JT, lấy ra giá trị theo **key động** và tính `node_seq` (nếu cần “base*1000 + offset”):
```yaml
vars:
  # offset đặt ở node nếu JT dùng nhiều vị trí khác nhau (vd 10/20/30...)
  offset: "{{ offset | default(10) }}"

pre_tasks:
  - name: Resolve node_seq from sq_id and plan_id
    set_fact:
      seq_base: >-
        {{ (sq_id | default({})).get(plan_id, (sq_id.default | default(0))) | int }}
      node_seq: "{{ (seq_base * 1000) + (offset | int) }}"

  - name: Short-circuit if resuming beyond this node
    meta: end_play
    when: (resume_from_seq | default(0) | int) > (node_seq | int)
```
Giải thích nhanh:
- `sq_id[plan_id]` → lấy key động.
- Dùng `.get(plan_id, fallback)` để **không lỗi** nếu thiếu key; fallback có thể là `sq_id.default` hoặc `0`.
- `plan_id` chỉ truyền **một lần từ workflow**, JT tái dùng ở plan khác chỉ đổi `plan_id` (và `offset` nếu vị trí khác).
4. Nếu node **không phải JT** (Inventory Sync/Approval/Sub-workflow), đặt một **Gate JT** trước nó, Gate cũng dùng đúng công thức trên để quyết định **skip hay chạy**.
---
### Ví dụ truyền biến trong Workflow (node JT này dùng cho planA)
- Node extra_vars:

```yaml
plan_id: "planA"
offset: 20
```
- Workflow survey (toàn cục):
```yaml
resume_from_seq: 0   # chọn khi muốn resume
```
→ Lúc chạy:
- `seq_base = sq_id.get('planA') = 1`
- `node_seq = 1*1000 + 20 = 1020`
- Nếu `resume_from_seq > 1020` thì JT này _end_play_ (skip OK); ngược lại chạy bình thường.
---
### Lưu ý nhỏ
- YAML **không dùng tab**: hãy thay tab bằng 2/4 space:
    
    ```yaml
    sq_id:
      planA: 1
      planB: 40
      planC: 15
    ```
- Nếu muốn đặt **mặc định** cho mọi JT khi thiếu key, thêm:
    ```yaml
    sq_id:
      default: 0
      planA: 1
      planB: 40
    ```
    rồi dùng `.get(plan_id, sq_id.default)` như ở trên.

Vậy là mỗi JT chỉ cần **khai 1 dict** (`sq_id`) một lần. Khi JT được dùng ở workflow nào,  **chỉ truyền `plan_id`** (và `offset` nếu cần), là nó tự “bốc” đúng sequence ID của plan đó.

