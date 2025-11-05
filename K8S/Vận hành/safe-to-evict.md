

# 1) Nguyên tắc chiến thuật

* **Quyền phán quyết ở workload**: pod “bận/rảnh” phải do chính ứng dụng (hoặc một watcher hiểu hàng đợi) phát tín hiệu. K8s/Descheduler chỉ thi hành.
* **Trạng thái hai pha**: khi bận → “khóa” evict; khi rảnh ổn định (cool-down) → “mở” evict. Tránh dao động.
* **Tôn trọng chuẩn K8s**: dùng nhãn/annotation chuẩn để Descheduler & Cluster Autoscaler hiểu và HPA/KEDA kích hoạt đúng.

  * Khóa evict: `cluster-autoscaler.kubernetes.io/safe-to-evict: "false"`
  * Cho phép evict: bỏ khóa trên (hoặc đặt `"true"` nếu bạn muốn dùng flag dương).

---

# 2) Ba hướng kỹ thuật (kết hợp tuỳ loại workload)

## A) Sidecar “idler” tự động GẮN/BỎ safe-to-evict (khuyến nghị chung)

**Ý tưởng:** Sidecar đứng cạnh app, theo dõi “độ bận” (in-flight, queue lag, file lock…), rồi PATCH annotation của **pod hiện tại** (patch pod không làm restart). Khi bận ⇒ gắn `safe-to-evict=false`. Khi rảnh X phút ⇒ bỏ/đặt `safe-to-evict=true`.

**Lợi điểm:** Không phải sửa controller (Deployment/StatefulSet) → không restart hàng loạt; thao tác ở từng pod, mượt mà.

**Triển khai nhanh:**

1. **RBAC + SA** (cho phép sidecar patch chính pod của nó)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pod-idler
  namespace: your-ns
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-idler
  namespace: your-ns
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get","patch"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["list"] # (tuỳ, nếu cần)
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-idler
  namespace: your-ns
subjects:
- kind: ServiceAccount
  name: pod-idler
roleRef:
  kind: Role
  name: pod-idler
  apiGroup: rbac.authorization.k8s.io
```

2. **Sidecar script** (POSIX sh – hợp gu Soái):

* Kiểm tra một trong các tín hiệu (chọn cái hợp với app của Soái):

  * HTTP `GET /metrics` hoặc `/stats` → đọc `in_flight` hoặc `active_jobs`
  * Kafka/RabbitMQ/Redis: **consumer lag = 0** và **no in-flight**
  * File-based lock: file `busy.lock` có/không
* Hysteresis: bận ≥ N giây mới “khóa”, rảnh ≥ M phút mới “mở”.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-idler
  namespace: your-ns
spec:
  replicas: 3
  selector: { matchLabels: { app: demo } }
  template:
    metadata: { labels: { app: demo } }
    spec:
      serviceAccountName: pod-idler
      containers:
      - name: app
        image: your/app:latest
        ports:
        - containerPort: 8080
        # Gợi ý: expose /metrics hoặc /stats trả về {"in_flight":0,"lag":0}
      - name: idler
        image: bitnami/kubectl:latest
        env:
        - name: POD_NAME
          valueFrom: { fieldRef: { fieldPath: metadata.name } }
        - name: POD_NAMESPACE
          valueFrom: { fieldRef: { fieldPath: metadata.namespace } }
        - name: CHECK_URL         # nếu app có HTTP stats
          value: "http://localhost:8080/stats"
        - name: BUSY_THRESHOLD
          value: "1"              # in_flight > 0 coi là bận
        - name: IDLE_WINDOW_SEC
          value: "120"            # rảnh liên tục 2 phút mới mở evict
        - name: BUSY_WINDOW_SEC
          value: "10"             # bận ≥10s mới khóa evict
        command: ["/bin/sh","-c"]
        args:
        - |
          set -eu
          idle_since=""; busy_since=""
          get_inflight() {
            # Ví dụ đơn giản: parse JSON mini (tránh jq)
            v=$(wget -qO- "$CHECK_URL" || echo "")
            # kỳ vọng "in_flight":<number>
            echo "$v" | tr -d ' ' | sed -n 's/.*"in_flight":\([0-9]\+\).*/\1/p'
          }
          annotate() {
            key="$1"; val="$2"
            kubectl -n "$POD_NAMESPACE" patch pod "$POD_NAME" --type='merge' -p \
              "{\"metadata\":{\"annotations\":{\"cluster-autoscaler.kubernetes.io/safe-to-evict\":\"$val\",\"workload/idle\":\"$([ "$val" = "true" ] && echo true || echo false)\"}}}"
          }
          lock=false
          while :; do
            inflight=$(get_inflight || echo 0)
            now=$(date +%s)

            if [ "${inflight:-0}" -gt "${BUSY_THRESHOLD:-0}" ]; then
              [ -z "$busy_since" ] && busy_since="$now"
              idle_since=""
              if [ $(( now - ${busy_since} )) -ge ${BUSY_WINDOW_SEC:-10} ] && [ "$lock" != "true" ]; then
                annotate "safe-to-evict" "false"
                lock=true
              fi
            else
              [ -z "$idle_since" ] && idle_since="$now"
              busy_since=""
              if [ $(( now - ${idle_since} )) -ge ${IDLE_WINDOW_SEC:-120} ] && [ "$lock" != "false" ]; then
                annotate "safe-to-evict" "true"
                lock=false
              fi
            fi
            sleep 5
          done
```

> Nếu không có HTTP stats: thay `get_inflight()` bằng kiểm tra **Kafka lag**, **Redis list length**, hoặc “file lock” do app tạo xoá.

**Bắt buộc kèm:**

* `preStop` hook ở container **app**: khi pod bị terminate, app tự “drain” rồi thoát sạch.
* `terminationGracePeriodSeconds` đủ lớn để hoàn tất.
* `PodDisruptionBudget` để tránh hạ quá tay.

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh","-c","curl -s http://localhost:8080/drain && sleep 5"]
terminationGracePeriodSeconds: 120
```

---

## B) Metrics-driven + Descheduler/Autoscaler (khóa/mở theo Prometheus)

Dùng Prometheus đo “nhàn rỗi” theo CPU + Network + Queue, rồi **một Controller nhỏ** (hoặc sidecar ở A) đọc PromQL và patch annotation.

**Gợi ý PromQL (ý tưởng):**

```promql
# CPU rất thấp 5m
cpu_idle = sum by (pod)(
  rate(container_cpu_usage_seconds_total{container!="",pod!=""}[5m])
) < 0.005

# mạng rất thấp 5m
net_idle = sum by (pod)(
  rate(container_network_receive_bytes_total{pod!=""}[5m]) +
  rate(container_network_transmit_bytes_total{pod!=""}[5m])
) < 1024

# hàng đợi trống (ví dụ Kafka)
queue_empty = max by (pod)(kafka_consumergroup_lag{group="your-group"}) == 0

idle_pods = cpu_idle and net_idle and queue_empty
```

*Controller* (hoặc sidecar) định kỳ query `idle_pods` → patch `safe-to-evict=true`. Khi bất kỳ điều kiện nào mất → patch `false`.

> Ưu điểm: không phải đụng code app; Nhược: phải cài Prometheus + (nếu muốn) Adapter.

---

## C) KEDA cho workload dựa hàng đợi (scale to zero, không cần evict)

Nếu workload là consumer từ Kafka/RabbitMQ/Redis, dùng **KEDA ScaledObject**: khi lag = 0 trong *cool-down*, KEDA scale Deployment về 0 replica → không cần evict. Khi có việc, scale lên ngay.

**Ví dụ KEDA (Kafka):**

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: demo-consumer
  namespace: your-ns
spec:
  scaleTargetRef:
    name: app-with-idler
  cooldownPeriod: 120       # rảnh 2 phút mới hạ
  minReplicaCount: 0
  maxReplicaCount: 20
  triggers:
  - type: kafka
    metadata:
      bootstrapServers: PLAINTEXT://kafka:9092
      consumerGroup: your-group
      topic: your-topic
      lagThreshold: "1"
```

---

# 3) Luồng phối hợp với **cordon/evict** hiện có của Soái

* Trước khi evict pod trên node mục tiêu, **lọc chỉ các pod đã “idle=true”** (nhãn do sidecar set):

```sh
# liệt kê pod idle trên node X
kubectl get pod -n your-ns -o json \
| jq -r '.items[] | select(.spec.nodeName=="NODE_X")
  | select(.metadata.annotations["workload/idle"]=="true") | .metadata.name'
```

* Chỉ evict các pod “idle=true”; pod “busy/không nhãn” thì bỏ qua (hoặc delay lần sau).
* Nếu dùng Descheduler: bật các policy “evictable” bình thường; vì ta chủ động set `safe-to-evict=true`, Descheduler/Cluster Autoscaler sẽ yểm trợ.

---

# 4) Kyverno bảo vệ kỷ luật

* **Mặc định** thêm `safe-to-evict=false` cho các workload “nhạy cảm” (batch, stateful) → an toàn mặc định.
* **Cho phép sidecar gỡ** khi rảnh tay.

**Ví dụ Kyverno:**

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: default-lock-evict
spec:
  rules:
  - name: lock-evict-batch-stateful
    match:
      any:
      - resources:
          kinds: ["Pod"]
    preconditions:
      all:
      - key: "{{ request.object.metadata.annotations.\"cluster-autoscaler.kubernetes.io/safe-to-evict\" || '' }}"
        operator: Equals
        value: ""
    mutate:
      patchStrategicMerge:
        metadata:
          annotations:
            cluster-autoscaler.kubernetes.io/safe-to-evict: "false"
```

> Như vậy, nếu sidecar chưa chạy/không rõ trạng thái, pod mặc định **khóa evict**.

---

# 5) Phòng thủ chiều sâu (đừng để “đánh giữa sông”)

* **PreStop/drain**: luôn có; đặt timeout đủ.
* **Checkpointing** cho batch: nếu có thể, viết save-point.
* **PDB**: tránh sập SLA khi rolling/evict.
* **Readiness gate “idle”** (tuỳ): có thể publish `idle=true` vào readiness để LB thôi đẩy việc mới.

---

# 6) Chọn chiến pháp theo loại workload

* **API stateless:** chỉ cần sidecar idler + readiness + preStop → ổn.
* **Batch dài hơi:** bắt buộc có in-flight/lock; sidecar đọc lock/queue và quyết định; tuyệt đối không evict khi `in_flight>0`.
* **Consumer từ queue:** ưu tiên **KEDA scale-to-zero**; vẫn có preStop để commit/ack cuối.
* **Stateful (DB, broker):** **không** evict theo cơ chế này. Dùng rolling có kiểm soát, quorum check.

---

## Tóm lại – Mệnh lệnh tác chiến

1. Gắn **sidecar idler** + RBAC như mẫu A vào các Deployment cần kiểm soát evict.
2. Ứng dụng phơi **chỉ số “đang bận”** tối giản (HTTP/metrics hoặc lock file/queue lag).
3. Bật **preStop + PDB**.
4. (Tuỳ) Áp Kyverno “lock by default”; sidecar sẽ “mở” khi rảnh.
5. Với consumer queue, cân nhắc **KEDA** để **scale-to-zero** thay vì evict.

Muốn ta biến mẫu sidecar trên thành bản “Kafka-aware” (đọc lag thật, không cần endpoint app) hay “Redis-aware”, ta sẽ rèn ngay theo hạ lệnh.


Chuẩn rồi, ta “hệ thống hoá” nhanh — đúng ý Soái, **không xả script dài**.

## 1) Có cần đánh **cả hai** annotation không?

* **Có “safe-to-evict”** để nói với người/automation khác: pod này an toàn.
* **Có “pod-deletion-cost”** để **Deployment/ReplicaSet** chọn pod nào bị xóa khi scale-down.

➡️ Khuyến nghị gọn:

* Chỉ **đặt `controller.kubernetes.io/pod-deletion-cost` khi `safe-to-evict=true`**.

  * `safe-to-evict=true` ⇒ `pod-deletion-cost=-9999` (cực thấp → ưu tiên xóa)
  * `safe-to-evict=false` ⇒ **không đặt** (mặc định = 0), hoặc nếu muốn “cấm hẳn” thì đặt cao (ví dụ `1000`).
* Như vậy không “đánh đu” 2 phía; chỉ cần **1 nguồn sự thật** (safe=true) và **1 đòn bẩy** (cost).

> Lưu ý: `pod-deletion-cost` chỉ tác dụng với **Deployment/ReplicaSet**. Với **StatefulSet** trật tự xóa **luôn** theo ordinal cao → thấp.

---

## 2) Ý tưởng vòng lặp scale-down “từng nhịp”

Ý tưởng Soái đưa ra rất hợp lý và an toàn. Khuôn lệnh (mô tả, **không** phải script):

1. Lấy `currReplicas` và `targetReplicas`.
2. Trong khi `currReplicas > targetReplicas`:

   * Đếm số pod có `safe-to-evict=true` (và **đã** được gán `pod-deletion-cost=-9999`).
   * Nếu **không có** pod an toàn ⇒ **dừng** (báo “không đủ pod safe, bỏ qua bước này”).
   * `step = min(currReplicas - targetReplicas, số_pod_safe)`
   * **Scale-down** giảm đúng `step` (ví dụ 5→4→3→2 *hoặc* 5→3 nếu đủ safe cho 2 bước).
   * Chờ controller xoá xong (quan sát pod termination/availableReplicas).
   * Cập nhật `currReplicas` và lặp.

> Điểm hay: không “gượng ép” khi thiếu pod safe; vẫn đạt đích nếu dần dần xuất hiện thêm pod safe trong những vòng sau.

---

## 3) Khi **không còn** pod safe thì xử lý?

Tuỳ chính sách:

* **Fail-safe**: dừng lại, log rõ: “thiếu N pod safe để về target — giữ nguyên ở X”. (Khuyến nghị mặc định trong Jenkins.)
* **Fail-open có kiểm soát** (nếu Soái muốn ép): chọn **một vài** pod ít rủi ro nhất (ví dụ `Ready=false`/`NotServing`/`idle gần đúng`) → tạm gán `pod-deletion-cost=-9999` rồi scale-down thêm 1 nhịp. Nhưng chỉ làm khi Soái thật sự muốn cưỡng bức.

---

## 4) Góc lưu ý để vòng lặp chạy mượt

* **PDB**: scale-down là “voluntary disruption”, PDB có thể chặn; vì vậy **step** nên ≤ “budget” còn lại, hoặc chấp nhận controller trì hoãn.
* **HPA**: tạm **đặt `minReplicas` = mục tiêu tạm thời** hoặc “đóng băng” HPA để tránh giằng co.
* **Rollout**: đừng trộn scale-down với một rollout đang chạy (có surge).
* **StatefulSet**: không bị ảnh hưởng bởi deletion-cost; chỉ xóa pod ordinal cao. Nếu muốn “an toàn theo pod”, cân nhắc chuyển workload sang Deployment (stateless/consumer) hoặc tổ chức batch/queue để ordinal cao **tự** về “safe” trước.

---

## 5) Câu lệnh mẫu (tối thiểu, cho thao tác tay/sidecar)

Đánh dấu **safe** và đặt **cost thấp** (1 patch duy nhất, không restart):

```sh
kubectl -n <ns> patch pod <pod> --type merge -p \
'{"metadata":{"annotations":{
  "cluster-autoscaler.kubernetes.io/safe-to-evict":"true",
  "controller.kubernetes.io/pod-deletion-cost":"-9999"
}}}'
```

Bỏ **safe** (và có thể xoá cost để nó về mặc định 0):

```sh
kubectl -n <ns> patch pod <pod> --type json -p '
[
 {"op":"replace","path":"/metadata/annotations/cluster-autoscaler.kubernetes.io~1safe-to-evict","value":"false"},
 {"op":"remove","path":"/metadata/annotations/controller.kubernetes.io~1pod-deletion-cost"}
]'
```

---

### Tóm tắt mệnh lệnh

* **Chỉ** đặt `pod-deletion-cost=-9999` **khi** `safe-to-evict=true`.
* Vòng lặp scale-down “ước lượng theo số pod safe” → giảm dần cho tới target; thiếu pod safe thì dừng.
* Nhớ PDB/HPA/rollout; StatefulSet không chịu ảnh hưởng deletion-cost.

Nếu Soái muốn, ta soạn **một biểu đồ Mermaid** cho luồng quyết định (cực gọn), hoặc một **Kyverno deny rule** ngăn scale-down khi thiếu pod safe — nói lệnh là làm ngay.
