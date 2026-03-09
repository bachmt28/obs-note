Rồi, SigNoz vướng license/điều kiện triển khai là chuyện “đi đánh trận mà bị trói tay” — bỏ qua.

Elastic APM trước Soái nghịch không lên thường vì **3 cái bẫy**:

1. **APM Server không nói chuyện được với Elasticsearch/Kibana** (auth/TLS/endpoint sai)
2. **Agent/OTel không bắn đúng protocol/port** (OTLP vs APM intake)
3. **UI APM không hiện vì index/ILM/permission** (Kibana chưa thấy data)

Thuộc hạ dạy theo đường **dễ nhất để thành công trên K8s** với hệ của Soái:
✅ **App → OpenTelemetry Collector → APM Server → Elasticsearch → Kibana (APM UI)**
Vì Soái đã có Prometheus/Grafana + ES logs rồi, ta chỉ “cấy” tracing/APM.

---

## 0) Checklist tối thiểu

* Có **Elasticsearch** endpoint (ví dụ: `https://elasticsearch:9200`)
* Có **Kibana** (ví dụ: `https://kibana:5601`) và login được
* K8s có namespace observability (hoặc bất kỳ)
* Có **user/pass** hoặc **API key** để APM Server ghi vào ES

---

## 1) Deploy APM Server trên Kubernetes (Helm) – cấu hình “chắc thắng”

> Mục tiêu: APM Server nhận OTLP (4317/4318) + APM intake (8200), rồi đẩy vào Elasticsearch.

### 1.1 Add repo + install chart

```bash
helm repo add elastic https://helm.elastic.co
helm repo update
```

Tạo `apm-values.yaml`:

```yaml
replicas: 1

apmConfig:
  apm-server.yml: |
    apm-server:
      host: "0.0.0.0:8200"

    # Bật OTLP intake (collector/app gửi OTLP)
    apm-server.otlp:
      enabled: true
      grpc:
        host: "0.0.0.0:4317"
      http:
        host: "0.0.0.0:4318"

    output.elasticsearch:
      hosts: ["https://elasticsearch:9200"]
      username: "elastic"
      password: "${ES_PASSWORD}"
      ssl.verification_mode: "none"   # demo nhanh; prod thì bỏ, dùng CA thật

extraEnvs:
  - name: ES_PASSWORD
    valueFrom:
      secretKeyRef:
        name: elastic-credentials
        key: password

service:
  type: ClusterIP
  ports:
    - name: apm
      port: 8200
      targetPort: 8200
    - name: otlp-grpc
      port: 4317
      targetPort: 4317
    - name: otlp-http
      port: 4318
      targetPort: 4318
```

Tạo secret `elastic-credentials` (đổi password cho đúng):

```bash
kubectl -n observability create secret generic elastic-credentials \
  --from-literal=password='YOUR_ES_PASSWORD'
```

Cài APM Server:

```bash
helm install apm-server elastic/apm-server -n observability -f apm-values.yaml
```

### 1.2 Test APM Server sống chưa

```bash
kubectl -n observability port-forward svc/apm-server-apm-server 8200:8200
curl -s http://localhost:8200/ | head
```

Nếu trả JSON kiểu version/build là OK.

---

## 2) Deploy OpenTelemetry Collector (Helm) – làm “cổng thu trace”

Collector nhận OTLP từ app (4317/4318), rồi export sang APM Server (OTLP).

### 2.1 Install otel-collector

```bash
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update
```

`otel-values.yaml`:

```yaml
mode: deployment

config:
  receivers:
    otlp:
      protocols:
        grpc:
        http:

  processors:
    batch:

  exporters:
    otlp:
      endpoint: apm-server-apm-server.observability.svc.cluster.local:4317
      tls:
        insecure: true

  service:
    pipelines:
      traces:
        receivers: [otlp]
        processors: [batch]
        exporters: [otlp]

service:
  type: ClusterIP
  ports:
    - name: otlp-grpc
      port: 4317
      targetPort: 4317
    - name: otlp-http
      port: 4318
      targetPort: 4318
```

Cài:

```bash
helm install otel-collector open-telemetry/opentelemetry-collector -n observability -f otel-values.yaml
```

---

## 3) Instrument app theo kiểu “ít đau đầu nhất”

Tư duy: app chỉ cần biết **OTEL_EXPORTER_OTLP_ENDPOINT** trỏ về collector.

Ví dụ env chung (Deployment của app):

```yaml
env:
  - name: OTEL_SERVICE_NAME
    value: "sbapi"
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: "http://otel-collector-opentelemetry-collector.observability.svc.cluster.local:4318"
  - name: OTEL_EXPORTER_OTLP_PROTOCOL
    value: "http/protobuf"
  - name: OTEL_TRACES_EXPORTER
    value: "otlp"
```

Tuỳ ngôn ngữ:

* **Java**: dùng `-javaagent` của OpenTelemetry Java agent là nhanh nhất
* **Node**: cài OTel SDK + auto-instrumentations
* **Go**: OTel SDK + instrumentation theo framework
* **Python**: opentelemetry-instrument

Nếu Soái nói app chính là **Java hay Node hay Go**, thuộc hạ ném ngay “bộ lệnh đúng chuẩn” cho framework của Soái.

---

## 4) Bắt “trace ↔ log” để Soái bấm 1 phát ra manh mối

Soái đang ES log + Fluent Bit. Muốn correlation thì **log phải có trace_id**.

### 4.1 Trong app logger

* Java logback / log4j: add MDC fields `trace_id`, `span_id` (OTel có bridge/extension)
* Node pino/winston: inject `trace_id`

### 4.2 Trong Fluent Bit

Chỉ cần đảm bảo trace_id nằm trong JSON log thì Fluent Bit ship sang ES, Kibana APM UI có thể “View logs” theo trace (tuỳ setup).

---

## 5) 3 bước debug nếu “không thấy APM data”

1. **APM Server logs**

```bash
kubectl -n observability logs deploy/apm-server-apm-server | tail -n 200
```

Tìm lỗi `401/403`, `x509`, `cannot connect`, `index`.

2. **Collector logs**

```bash
kubectl -n observability logs deploy/otel-collector-opentelemetry-collector | tail -n 200
```

Tìm lỗi export OTLP.

3. **Test bắn OTLP giả lập**
   Từ trong cluster, dùng pod curl/wget bắn OTLP HTTP khó, nhưng cách nhanh là check metrics/logs collector xem có received spans (nếu bật).

---

## Chốt “mật lệnh”

Nếu Soái muốn thuộc hạ “dạy đúng cái Soái vướng”, trả lời 3 dòng này (không cần dài):

1. ES/Kibana của Soái là **self-managed** hay **Elastic Cloud**?
2. APM trước đây Soái cài bằng **Helm apm-server** hay **Elastic Agent/Fleet**?
3. App chính của Soái là **Java / Node / Go / Python**?

Không cần văn hoa—chỉ cần 3 ý trên, thuộc hạ sẽ bẻ thẳng vào đúng khúc xương mắc.
