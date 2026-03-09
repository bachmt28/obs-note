- [ ] Lấy cert 
- [ ] Chuẩn hoá cert 
- [ ]  `vCenter.caCertPath` , sửa file mới trong file cấu hình
- [ ] Chạy cập nhật user trước : gkectl update cluster --config USER_CLUSTER_CONFIG --kubeconfig ADMIN_CLUSTER_KUBECONFIG
- [ ] Verify work ok gkectl diagnose cluster --kubeconfig ADMIN_CLUSTER_KUBECONFIG \
  --cluster-name USER_CLUSTER_NAME
- [ ] Tương tự với admin cluster: 
gkectl update admin --config ADMIN_CLUSTER_CONFIG --kubeconfig ADMIN_CLUSTER_KUBECONFIG
gkectl diagnose cluster --kubeconfig ADMIN_CLUSTER_KUBECONFIG



```sh
helm install pushgateway prometheus-community/prometheus-pushgateway \
  -n sb-kube-pg-stack \
  --set serviceMonitor.enabled=true \
  --set serviceMonitor.namespace=sb-kube-pg-stack \
  --set serviceMonitor.additionalLabels.release=kube-prometheus-stack-81-1770113856




./cpu_only.sh > /tmp/cpu.prom

./cpu_only.sh | curl --data-binary @- http://pushgateway:9091/metrics/job/kubectl_top
```



lưu ý các workload
clusterapi-controllers
vsphere-csi
vsphere-metrics

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kubectl-top-pusher
  namespace: sb-kube-pg-stack
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kubectl-top-pusher
rules:
  - apiGroups: ["metrics.k8s.io"]
    resources: ["pods"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubectl-top-pusher
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kubectl-top-pusher
subjects:
  - kind: ServiceAccount
    name: kubectl-top-pusher
    namespace: sb-kube-pg-stack
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: kubectl-top-pusher-scripts
  namespace: sb-kube-pg-stack
data:
  run.sh: |
    #!/bin/sh
    set -eu

    PUSHGW_URL="${PUSHGW_URL:-http://pushgateway-prometheus-pushgateway.sb-kube-pg-stack.svc:9091}"
    JOB="${JOB:-kubectl_top}"
    INSTANCE="${INSTANCE:-incluster}"
    INTERVAL_SECONDS="${INTERVAL_SECONDS:-60}"

    # Optional: filter namespaces (e.g. "sb-"). Leave empty to collect all.
    NS_PREFIX="${NS_PREFIX:-}"

    while true; do
      start_ts="$(date +%s)"

      # Delete existing grouping key to avoid stale metrics when pods disappear
      curl -sS -X DELETE "${PUSHGW_URL}/metrics/job/${JOB}/instance/${INSTANCE}" >/dev/null || true

      # Collect + push CPU cores per container
      if [ -n "$NS_PREFIX" ]; then
        kubectl top pod -A --containers --no-headers | awk -v pref="$NS_PREFIX" '
        BEGIN { print "# TYPE k8s_top_container_cpu_cores gauge" }
        $1 ~ "^"pref {
          ns=$1; pod=$2; ctn=$3; cpu=$4;
          if (cpu ~ /m$/) { sub(/m$/,"",cpu); cpu_cores=(cpu+0)/1000.0; }
          else { cpu_cores=(cpu+0); }
          printf "k8s_top_container_cpu_cores{namespace=\"%s\",pod=\"%s\",container=\"%s\"} %.6f\n", ns, pod, ctn, cpu_cores;
        }' | curl -sS --fail --data-binary @- \
            "${PUSHGW_URL}/metrics/job/${JOB}/instance/${INSTANCE}"
      else
        kubectl top pod -A --containers --no-headers | awk '
        BEGIN { print "# TYPE k8s_top_container_cpu_cores gauge" }
        {
          ns=$1; pod=$2; ctn=$3; cpu=$4;
          if (cpu ~ /m$/) { sub(/m$/,"",cpu); cpu_cores=(cpu+0)/1000.0; }
          else { cpu_cores=(cpu+0); }
          printf "k8s_top_container_cpu_cores{namespace=\"%s\",pod=\"%s\",container=\"%s\"} %.6f\n", ns, pod, ctn, cpu_cores;
        }' | curl -sS --fail --data-binary @- \
            "${PUSHGW_URL}/metrics/job/${JOB}/instance/${INSTANCE}"
      fi

      end_ts="$(date +%s)"
      elapsed=$((end_ts - start_ts))

      # Sleep to maintain fixed interval
      if [ "$elapsed" -lt "$INTERVAL_SECONDS" ]; then
        sleep $((INTERVAL_SECONDS - elapsed))
      else
        # If collection took longer than interval, sleep a little to avoid tight loop
        sleep 1
      fi
    done
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubectl-top-pusher
  namespace: sb-kube-pg-stack
  labels:
    app: kubectl-top-pusher
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kubectl-top-pusher
  template:
    metadata:
      labels:
        app: kubectl-top-pusher
    spec:
      serviceAccountName: kubectl-top-pusher
      restartPolicy: Always
      containers:
        - name: pusher
          image: bitnami/kubectl:latest
          imagePullPolicy: IfNotPresent
          command: ["/bin/sh","-c"]
          args:
            - |
              chmod +x /scripts/run.sh && /scripts/run.sh
          env:
            - name: PUSHGW_URL
              value: "http://pushgateway-prometheus-pushgateway.sb-kube-pg-stack.svc:9091"
            - name: JOB
              value: "kubectl_top"
            - name: INSTANCE
              value: "incluster"
            - name: INTERVAL_SECONDS
              value: "30"
            # Optional: collect only namespaces starting with sb-
            # - name: NS_PREFIX
            #   value: "sb-"
          volumeMounts:
            - name: scripts
              mountPath: /scripts
      volumes:
        - name: scripts
          configMap:
            name: kubectl-top-pusher-scripts
            defaultMode: 0755

```