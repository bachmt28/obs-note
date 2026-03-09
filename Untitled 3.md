##
from
```promql
sum( max by (cluster, namespace, pod, container)(node_namespace_pod_container:container_cpu_usage_seconds_total:sum_rate5m{cluster="$cluster", namespace="$namespace"}) * on(cluster, namespace, pod) group_left(workload, workload_type) namespace_workload_pod:kube_pod_owner:relabel{cluster="$cluster", namespace="$namespace", workload=~"$workload", workload_type=~"$type"} ) by (pod)
```
to
```
sum( max without (job, instance, endpoint, service) ( rate(container_cpu_usage_seconds_total{ cluster="$cluster", namespace="$namespace", container!="POD", container!="" }[5m]) ) * on(cluster, namespace, pod) group_left(workload, workload_type) namespace_workload_pod:kube_pod_owner:relabel{ cluster="$cluster", namespace="$namespace", workload=~"$workload", workload_type=~"$type" } ) by (pod)
```
## `node_namespace_pod_container:container_cpu_usage_seconds_total:sum_rate5m`
from
```
sum by (cluster, namespace, pod, container) (
  rate(container_cpu_usage_seconds_total{job="kubelet", metrics_path="/metrics/cadvisor", image!=""}[5m])
) * on (cluster, namespace, pod) group_left(node) topk by (cluster, namespace, pod) (
  1, max by (cluster, namespace, pod, node) (kube_pod_info{node!=""})
)
```
to 
```
sum by (cluster, namespace, pod, container) (sum without (instance, endpoint, service) (rate(container_cpu_usage_seconds_total{container!="",container!="POD",image!="",job="kubelet",metrics_path="/metrics/cadvisor"}[5m]))) * on (cluster, namespace, pod) group_left (node) topk by (cluster, namespace, pod) (1, max by (cluster, namespace, pod, node) (kube_pod_info{node!=""}))
```


```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  annotations:
    meta.helm.sh/release-name: kube-prometheus-stack-81-1770113856
    meta.helm.sh/release-namespace: sb-kube-pg-stack
  labels:
    app: kube-prometheus-stack
    app.kubernetes.io/instance: kube-prometheus-stack-81-1770113856
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/part-of: kube-prometheus-stack
    app.kubernetes.io/version: 81.4.3
    chart: kube-prometheus-stack-81.4.3
    heritage: Helm
    release: kube-prometheus-stack-81-1770113856
  name: kube-prometheus-stack-81-1-k8s.rules.container-cpu-usage-second
  namespace: sb-kube-pg-stack
spec:
  groups:
    - name: k8s.rules.container_cpu_usage_seconds_total
      rules:
        - expr: >-
			sum by (cluster, namespace, pod, container) (
			  rate(container_cpu_usage_seconds_total{job="kubelet", metrics_path="/metrics/cadvisor", image!=""}[5m])
			) * on (cluster, namespace, pod) group_left(node) topk by (cluster, namespace, pod) (
			  1, max by (cluster, namespace, pod, node) (kube_pod_info{node!=""})
			)
          record: >-
            node_namespace_pod_container:container_cpu_usage_seconds_total:sum_rate5m
        - expr: >-
            sum by (cluster, namespace, pod, container) (
              irate(container_cpu_usage_seconds_total{job="kubelet", metrics_path="/metrics/cadvisor", image!=""}[5m])
            ) * on (cluster, namespace, pod) group_left(node) topk by (cluster,
            namespace, pod) (
              1, max by (cluster, namespace, pod, node) (kube_pod_info{node!=""})
            )
          record: >-
            node_namespace_pod_container:container_cpu_usage_seconds_total:sum_irate

```