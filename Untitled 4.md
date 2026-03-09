```sql
sum( max without (job, instance, endpoint, service) ( rate(container_cpu_usage_seconds_total{ cluster="$cluster", namespace="$namespace", container!="POD", container!="" }[5m]) ) * on(cluster, namespace, pod) group_left(workload, workload_type) namespace_workload_pod:kube_pod_owner:relabel{ cluster="$cluster", namespace="$namespace", workload=~"$workload", workload_type=~"$type" } ) by (pod)
```

```sql
sum by (container) (

  max without (job, instance, endpoint, service) (

    node_namespace_pod_container:container_cpu_usage_seconds_total:sum_rate5m{

      cluster="$cluster",

      namespace="$namespace",

      pod="$pod",

      image!="",

      container!="",

      container!="POD"

    }

  )

)
```

```sh
 cd ACS_RREQ_VA22 
 ls
 cd ACS_RREQ_VA24
 ls
 ctkmu d -s1 -nACS_RREQ_VA24
 ctcert i -s1 -lACS_RREQ_VA24 -fACS_RREQ_VA24.pem
 ctkmu d -s1 -nACS_RREQ_VA24
 cd ..
 ls
 cd ACS_SIGN_VA24
 ls
 ctkmu d -s1 -nACS_SIGN_VA24
 ctcert i -s1 -lACS_SIGN_VA24 -fACS_SIGN_VA24.pem
 ctkmu d -s1 -nACS_SIGN_VA24
```
