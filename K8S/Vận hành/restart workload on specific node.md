```sh
kubectl get pods -A --field-selector spec.nodeName=$NODE -o jsonpath='{range .items[*]}{.metadata.ownerReferences[0].kind}{" "}{.metadata.ownerReferences[0].name}{" "}{.metadata.namespace}{"\n"}{end}' | grep 'StatefulSet' | sort -u | while read kind name ns; do   kubectl rollout restart sts $name -n $ns; done

kubectl get pods -A --field-selector spec.nodeName=$NODE -o jsonpath='{range .items[*]}{.metadata.ownerReferences[0].kind}{" "}{.metadata.ownerReferences[0].name}{" "}{.metadata.namespace}{"\n"}{end}' | grep 'Deployment' | sort -u | while read kind name ns; do  kubectl rollout restart deploy $name -n $ns; done
```


```sh
kubectl get pods -A --field-selector spec.nodeName=$NODE -o jsonpath='{range .items[*]}{.metadata.ownerReferences[0].kind}{" "}{.metadata.ownerReferences[0].name}{" "}{.metadata.namespace}{"\n"}{end}' | grep 'StatefulSet' | sort -u | while read kind name ns; do   echo sts $name -n $ns; done

kubectl get pods -A --field-selector spec.nodeName=$NODE -o jsonpath='{range .items[*]}{.metadata.ownerReferences[0].kind}{" "}{.metadata.ownerReferences[0].name}{" "}{.metadata.namespace}{"\n"}{end}' | grep 'Deployment' | sort -u | while read kind name ns; do   echo echo deploy $name -n $ns; done
```
## Delete pod tại node cụ thể

evict_node_cordoned.sh
```sh
#!/bin/bash

# Mảng chứa các node pool pattern cần kiểm tra
NODEPOOL_PATTERNS=("gke-dc1dtu-sbapiv4" "gke-dc1dtu-sbapiv5" "gke-dc1dtu-core")
NS_PREFIX="sb-"

# Danh sách node bị cordoned sẽ gom vào đây
CORDONED_NODES=()

echo "📦 Kiểm tra các node pool: ${NODEPOOL_PATTERNS[*]}"

# Vòng lặp để tìm các node bị cordoned theo từng pattern
for pattern in "${NODEPOOL_PATTERNS[@]}"; do
  NODES=$(kubectl get nodes | awk -v p="$pattern" '$1 ~ p && $2 ~ /SchedulingDisabled/ {print $1}')
  if [[ -n "$NODES" ]]; then
    CORDONED_NODES+=($NODES)
  fi
done

if [[ ${#CORDONED_NODES[@]} -eq 0 ]]; then
  echo "✅ Không tìm thấy node nào bị cordoned."
  exit 0
fi

# Xử lý các node bị cordoned
for NODE in "${CORDONED_NODES[@]}"; do
  echo "⚠️  Xử lý node: $NODE"

  kubectl get pods -A --field-selector spec.nodeName="$NODE" --no-headers \
    | awk -v prefix="$NS_PREFIX" '$1 ~ "^"prefix {print $1, $2}' \
    | while read ns pod; do
        echo "🗑️  Xoá pod: $pod (namespace: $ns)"
        kubectl delete pod "$pod" -n "$ns" 
      done
done

echo "🏁 Hoàn tất xử lý các node cordoned."




```

```sh
#!/bin/bash

LABEL_SELECTOR="app=sbapi"  # <-- chỉnh label selector tại đây

echo -e "Node\t\tCPU (cores)\tMemory (Mi)"
echo "---------------------------------------------"

TOTAL_CPU=0
TOTAL_MEM=0

for NODE in $(kubectl get nodes -l "$LABEL_SELECTOR" -o name | cut -d'/' -f2); do
  CPU_SUM=0
  MEM_SUM=0

  PODS=$(kubectl get pods --all-namespaces --field-selector spec.nodeName=$NODE -o json)
  CPU_SUM=$(echo "$PODS" | jq '[.items[].spec.containers[].resources.requests.cpu // "0"] 
    | map(
        if test("m$") then
          (. | sub("m$"; "") | tonumber / 1000)
        else
          (. | sub("n$"; "") | tonumber / 1000000000)
        end
      ) 
    | add')

  MEM_SUM=$(echo "$PODS" | jq '[.items[].spec.containers[].resources.requests.memory // "0"] 
    | map(
        if test("Mi$") then
          (. | sub("Mi$"; "") | tonumber)
        elif test("Gi$") then
          (. | sub("Gi$"; "") | tonumber * 1024)
        elif test("Ki$") then
          (. | sub("Ki$"; "") | tonumber / 1024)
        else
          0
        end
      ) 
    | add')

  printf "%-15s\t%.3f\t\t%.0f\n" "$NODE" "$CPU_SUM" "$MEM_SUM"

  TOTAL_CPU=$(echo "$TOTAL_CPU + $CPU_SUM" | bc)
  TOTAL_MEM=$(echo "$TOTAL_MEM + $MEM_SUM" | bc)
done

echo "---------------------------------------------"
echo -e "Total\t\t$TOTAL_CPU\t\t$TOTAL_MEM"

```