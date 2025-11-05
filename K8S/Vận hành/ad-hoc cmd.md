## xoá pod tại node
```sh
NODE=gke-dc1dtu-sbapiv10-696d8d4f6b-p8ppv
kubectl get pods -A --field-selector spec.nodeName="$NODE" \
  -o custom-columns=NS:.metadata.namespace,NAME:.metadata.name --no-headers | grep 'sb-'| 
while read ns name; do
  [ -n "$ns" ] && [ -n "$name" ] && kubectl delete pod -n "$ns" "$name"
done
```
```sh
for no in `kubectl get no | grep sbapiv10 | grep 'Ready,SchedulingDisabled' | awk -F' ' '{print $1}'`;do 
kubectl get pod -A \
  --field-selector spec.nodeName=$no \
  -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name' \
| awk '/^sb-/{system("kubectl delete pod -n "$1" "$2)}'
done


for ((;;)); do

date

kubectl top pod istio-ingressgateway-backend-56b6fdcf4b-8drgt -n asm-gateway --no-headers

kubectl top pod istio-ingressgateway-backend-56b6fdcf4b-b42xk -n asm-gateway --no-headers

sleep 5

echo '-----'

done
```


```sh
curl --location 'http://dev-asmsbapi.seauat.com.vn/DirectCollectApi/rest/seab/process' \
--header 'Content-Type: application/json' \
--header 'X-IBM-Client-Secret: 29a9b99d9bb9aa6a590fe595b258f230' \
--header 'X-IBM-Client-Id: 7bb07a7d07fdfd7337ee4367e0550402' \
--data '{
"header": {
"reqType": "REQUEST",
"api": "DIRECT_COLLECT_API",
"apiKey": "qmklfoni1ezxlf2ckpygpfx248",
"priority": 1,
"channel": "SEAMOBILE3.0",
"subChannel": "SEANET",
"location": "10.9.12.90",
"context": "PC",
"trusted": "false",
"requestAPI": "t24server",
"requestNode": "10.9.10.14",
"userID": "hunggggggggggg",
"synasyn": "true"
},
"body": {
"command": "GET_TRANSACTION",
"resTopic": "Username_yyyymmdd_timestamp_3",
"transaction": {
"authenType": "getDetailMaDinhDanh-2",
"contractId": "T51V4ZNEZ0"
}
}
}'
```

## labelling
kubectl get ns -o name | grep '^namespace/sb-vhht-istio' | xargs -n1 kubectl label --overwrite=false istio.io/rev-

```yaml
    - name: http_proxy
      value: http://dc2-proxyuat.seauat.com.vn:8080
    - name: https_proxy
      value: http://dc2-proxyuat.seauat.com.vn:8080
    - name: no_proxy
      value: >-
        .seabank.com.vn, .seauat.com.vn, gitlab-internal.seabank.com.vn,
        gke-dc2-m2lib-dtu.seauat.com.vn,
        localhost,gke-dc2-nexus-devtestuat.seauat.com.vn,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16,*.seabank.com.vn,
        *.seauat.com.vn, connectgateway.googleapis.com,199.36.153.8/30
    - name: HTTP_PROXY
      value: http://dc2-proxyuat.seauat.com.vn:8080
    - name: HTTPS_PROXY
      value: http://dc2-proxyuat.seauat.com.vn:8080
    - name: NO_PROXY
      value: >-
        .seabank.com.vn, .seauat.com.vn, gitlab-internal.seabank.com.vn,
        gke-dc2-m2lib-dtu.seauat.com.vn,
        localhost,gke-dc2-nexus-devtestuat.seauat.com.vn,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16,*.seabank.com.vn,
        *.seauat.com.vn, connectgateway.googleapis.com,199.36.153.8/30
```



https://gke-dc2-m2lib-dtu.seauat.com.vn/repository/updates.jenkins.io/
https://gke-dc2-m2lib-dtu.seauat.com.vn/repository/updates.jenkins.io/update-center.json
https://gke-dc2-m2lib-dtu.seauat.com.vn/repository/updates.jenkins.io/current/plugin-versions.json


https://gke-dc2-m2lib-dtu.seauat.com.vn/repository/repo.jenkins-ci.org/
https://gke-dc2-m2lib-dtu.seauat.com.vn/repository/repo.jenkins-ci.org/incrementals


```yaml
- name: JENKINS_UC
  value: https://gke-dc2-m2lib-dtu.seauat.com.vn/repository/updates.jenkins.io/
- name: JENKINS_UC_DOWNLOAD
  value: https://gke-dc2-m2lib-dtu.seauat.com.vn/repository/updates.jenkins.io/download
- name: JENKINS_INCREMENTALS_REPO_MIRROR
  value: https://gke-dc2-m2lib-dtu.seauat.com.vn/repository/repo.jenkins-ci.org/incrementals
- name: JENKINS_PLUGIN_INFO
  value: https://gke-dc2-m2lib-dtu.seauat.com.vn/repository/updates.jenkins.io/current/plugin-versions.json
```

curl -i "https://uat-biapi.seauat.com.vn/dashboard-api/conversations?username=linh.lv7&limit=10&offset=0" \
  -H "Origin: https://uat-data-assist.seauat.com.vn"


```sh
VERSIONS="
17.11.7
18.0.6
18.1.6
18.2.8
18.3.4
18.4.2
18.5.0
"
for ver in $VERSIONS; do
    url="https://packages.gitlab.com/gitlab/gitlab-ce/packages/ol/8/gitlab-ce-${ver}-ce.0.el8.x86_64.rpm/download.rpm"
    outfile="gitlab-ce-${ver}-ce.0.el8.x86_64.rpm"
    echo ">> Downloading GitLab CE ${ver} ..."
    wget $url -O $outfile --verbose
done
```

```sh
docker images --format '{{.Repository}} {{.ID}}' | while read repo id; do
    created=$(docker inspect --format '{{.Created}}' "$id" 2>/dev/null)
    [ -z "$created" ] && continue

    created_ts=$(date -d "$created" +%s)

    if [ "$created_ts" -lt "$cutoff" ]; then
        echo "🔥 Removing old image: $repo ($id)"
        docker rmi -f "$id"
    fi
done
```