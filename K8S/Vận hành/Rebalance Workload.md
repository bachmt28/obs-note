

1. **Version**:
    
    - Kubernetes / GKE on‑prem (Anthos) phiên bản mấy?
        
    - `kubectl version --short`
        
2. **Scheduler profile** bạn đang dùng có custom không?
    
    - `kubectl -n kube-system get cm kube-scheduler -o yaml` (hoặc đường cấu hình tương đương trên GKE on-prem).
        
3. **Cluster Autoscaler flags** (hoặc cấu hình trong GKE on-prem):
    
    - `--scale-down-utilization-threshold`, `--scale-down-unneeded-time`, `--balance-similar-node-groups`, …
        
4. **Descheduler** đã cài chưa? Version?
    
5. **PDB / Pod annotations**: có nhiều PDB “chặt” quá hoặc thiếu `cluster-autoscaler.kubernetes.io/safe-to-evict=true` không?
    
6. Có đang bật **DefaultPodTopologySpread** (mặc định) hay có **topologySpreadConstraints** trong PodSpec không?
    

Trả lời nhanh các mục trên, mình sẽ đưa manifest/command chính xác cho môi trường của bạn.

---

## 1) Binpack ngay từ lúc **schedule** (đỡ phải “gom rác” về sau)

Nếu bạn **điều khiển được kube-scheduler profiles**, đổi scoring strategy sang **MostAllocated** (binpacking):

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- schedulerName: default-scheduler
  plugins:
    score:
      disabled:
        - name: DefaultPodTopologySpread   # tắt mặc định rải đều
      enabled:
        - name: NodeResourcesFit
  pluginConfig:
  - name: NodeResourcesFit
    args:
      scoringStrategy:
        type: MostAllocated
        resources:
        - name: cpu
          weight: 1
        - name: memory
          weight: 1
```

Kết quả: scheduler sẽ **ưu tiên nhét pod vào node đã đầy tương đối**, giải phóng các node còn lại → CA dễ cắt.

> Nếu bạn không tắt được `DefaultPodTopologySpread`, ít nhất hãy giảm weight của nó hoặc cấu hình lại để nó đừng “rải” quá đều.

---

## 2) **Descheduler – HighNodeUtilization**: hút pod về ít node để CA cắt phần còn lại

Đây là cách **ít đụng chạm control-plane nhất** nếu bạn không custom được scheduler.

Chạy descheduler định kỳ (ví dụ 2–5 phút) với plugin **HighNodeUtilization** để **di chuyển pod khỏi các node dưới ngưỡng**, gom về các node “đủ đầy”:

```yaml
apiVersion: descheduler/v1alpha2
kind: DeschedulerPolicy
profiles:
- name: binpack
  plugins:
    balance:
      enabled:
        - name: HighNodeUtilization
  pluginConfig:
  - name: HighNodeUtilization
    args:
      thresholds:
        cpu: 0
        memory: 0
      targetThresholds:
        cpu: 70
        memory: 70
```

- `thresholds`: node nào **dưới** ngưỡng này coi là underutilized → đẩy pod đi.
    
- `targetThresholds`: cố gắng “đổ” pod sang node khác đến mức ~70% để binpack.
    

> Nhớ: pod phải **evictable** (không có PDB quá ngặt, có annotation `cluster-autoscaler.kubernetes.io/safe-to-evict=true`, không phải DaemonSet, v.v.).

---

## 3) Tinh chỉnh **Cluster Autoscaler**

Ngay cả khi đã binpack, nếu CA cấu hình “hiền quá” nó vẫn không xuống:

- `--scale-down-utilization-threshold=0.5` (hoặc cao hơn, tùy khẩu vị)
    
- `--scale-down-unneeded-time=5m` (hoặc thấp hơn mặc định 10m–20m)
    
- `--scale-down-non-empty-candidates-count` đủ lớn để CA xét nhiều node hơn
    
- `--skip-nodes-with-system-pods=false` (nếu system pods có thể di chuyển)
    
- `--balance-similar-node-groups=false` (đôi khi việc cân bằng nhóm làm chậm scale-down)
    

Và **đừng quên**: các pod stateless nên có `priorityClassName` & `PDB` hợp lý để CA/Descheduler “động” được.

---

## 4) Mẹo bổ sung

- **Tách node pool** cho workload chiến dịch vs workload nền. Khi chiến dịch hạ, node pool chiến dịch bị “cắt nguyên cụm” dễ hơn.
    
- Tránh đặt **PodTopologySpreadConstraints** (hoặc để weight thấp) cho các workload bạn muốn binpack.
    
- Dùng **Overprovisioning pods** (low priority pause pods) để đẩy scheduler/CA “làm việc” theo ý — nhưng đây là trick, dùng khi bạn hiểu rõ trade-off.
    

---

## Quy trình thực thi gọn:

1. Gửi mình các thông tin ở mục **0) Xác minh**.
    
2. Nếu bạn **không sửa được scheduler** → mình gửi ngay manifest Descheduler “HighNodeUtilization” + thông số CA hợp lý cho phiên bản GKE on-prem của bạn.
    
3. Nếu bạn **sửa được scheduler** → mình build cho bạn cấu hình profile “binpack” chuẩn, kèm hướng dẫn rollout an toàn + rollback path.
    
4. Kiểm tra PDB, `safe-to-evict`, priority class.
    
5. Test: chạy 1 chiến dịch giả, thả tải xuống, quan sát `kubectl top nodes` + log CA để đảm bảo node rỗng bị cắt đúng như kỳ vọng.
    
