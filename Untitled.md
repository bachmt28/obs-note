

---

#  Guideline: Setup CI/CD theo demo CaaS (Configuration-as-Code)

## 1) Dev cần chuẩn bị gì trước khi chạy

### 1.1 Jenkins prerequisites

* Jenkins agent label: `local-backend` (nếu có thay đổi tuỳ từng team, vhht sẽ cung cấp).
* Jenkins có các tool/CLI:

  * `git`
  * `docker` (build + push)
  * `helm`
  * `kubectl` (nếu chart cần/hoặc debug)
* Jenkins có credentials (Dev chỉ dùng, không tự tạo):

  * `GIT_CRED_ID` (checkout GitLab)
  * Docker registry credential: trong ví dụ là `dev-vhht-docker.seauat.com.vn`, với onprem thì creds trùng với tên của dev repo, còn cloud thì sẽ là `sa-jenkins-to-artifact` hoặc `sa-jenkins-to-artifact-uat`
  * Kubeconfig credentials: **credentialId phải trùng từng dòng trong `KUBE_CONTEXTS`** (vhht đã cung cấp)
  * Prisma creds (nếu bật scan): `prisma-cloud-access-key`, `prisma-cloud-secret-key`

---

## 2) Repo layout: Dev phải chỉnh những file nào

```
CaaS/
├── Jenkinsfile.yaml                # pipeline YAML (đã chuẩn)
├── app/
│   ├── values.yaml                 # (BẮT BUỘC) cấu hình workload
│   └── configMap.yaml              # (OPTIONAL) envFrom + mount file
├── mesh/
│   └── values.yaml                 # (BẮT BUỘC nếu deploy Istio/VS)
└── security/
    └── zapScanList.yaml            # (OPTIONAL) target list cho ZAP
```

**Nguyên tắc:** Dev muốn CI/CD chạy đúng thì **cần** chỉnh `app/values.yaml` (và `mesh/values.yaml` nếu có route), còn lại pipeline + lib xử lý.

---

## 3) File 1: `CaaS/app/values.yaml` (BẮT BUỘC)

Đây là file quan trọng nhất. Library sẽ đọc file này để dựng toàn bộ biến môi trường phục vụ build/deploy.

### 3.1 Các field tối thiểu Dev phải set

* `env`: môi trường (vd: `dev`, `uat`, `prod`)
* `version`: version logic của workload (vd: `main`, `v1`)
* `workload.specs.image.repository`: registry host (vd: `dev-vhht-docker.seauat.com.vn`)
* Port mapping:

  * `workload.specs.ports[].containerPort` (app lắng nghe)
  * `service.ports[].targetPort` phải **trùng** containerPort

Nếu thiếu các field bắt buộc, stage `sanityCicdEnv()` sẽ **fail ngay** (đúng thiết kế để dev không deploy bừa).

### 3.2 Những field Dev có thể để trống (fail-safe của library)

* `appLabel: ""`
  → lib tự lấy theo tên repo (đỡ bắt dev phải đặt).
* `workload.specs.image.name: ""` và `tag: ""`
  → dev để trống là bình thường ở dev/test. Build stage sẽ sinh imageName/tag và deploy stage sẽ inject vào chart.

### 3.3 Các field Dev hay sai (gây crash/không vào được URL)

* `service.targetPort` ≠ `containerPort` → service không route vào container
* `imagePullSecrets` sai → pod ImagePullBackOff
* `workload.kind` không đúng (`Deployment` vs `StatefulSet`)

---

## 4) File 2: `CaaS/app/configMap.yaml` (OPTIONAL)

Dùng khi app cần config env hoặc file.

### 4.1 Env configmap (envFrom)

```yaml
configMap:
  env:
    enabled: true
    data:
      TZ: "Asia/Ho_Chi_Minh"
```

### 4.2 File configmap (mount file)

```yaml
configMap:
  file:
    enabled: true
    data:
      app.js: |
        ...
    mounts:
      - key: app.js
        mountPath: /opt/config/app.js
        readOnly: true
```

**Lưu ý bắt buộc:** `mountPath` phải là full path.

---

## 5) File 3: `CaaS/mesh/values.yaml` (BẮT BUỘC nếu dùng Istio/VirtualService)

Dev chỉ chỉnh file này khi workload cần route qua Istio gateway.

Các field Dev phải khớp với app:

* `env`, `appLabel`, `org`, `system` → nên đồng bộ với `app/values.yaml`
* `serviceMesh.istio.gateway` + `host` → theo môi trường cấp
* `routing.prefix` → context path của app
* `routing.canaries` → tỷ lệ traffic theo version (tổng ~100)

---

## 6) File 4: `CaaS/security/zapScanList.yaml` (OPTIONAL)

Khai báo target để ZAP scan.

```yaml
webs:
  - name: web1
    url: https://example.com

apis:
  - name: noti
    url: https://example.com/openapi.yaml
```

Nếu pipeline bật ZAP mà file rỗng hoàn toàn → lib ZAP sẽ **fail pipeline** (đúng thiết kế: scan mà không có target thì dừng).

---

# 7) Jenkins parameters: Dev cần set gì khi chạy job

Các parameter trong pipeline:

### Build

* `GITLAB_REPOSITORY_URL`: URL repo (khi chạy tay)
* `GITLAB_REFS`: branch/tag (vd `dev`, `main`)
* `GIT_CRED_ID`: Jenkins credential để checkout GitLab

### Deploy

* `KUBE_CONTEXTS`: mỗi dòng 1 context (vd: `gke-dc1dtu-usrcl`)
* `NAMESPACE`: namespace deploy

### Security/Notify

* `SECURITY_CONTEXT`, `SECURITY_VER`, `SECURITY_TIMEOUT`
* `DEV_WEBEX`

---

# 8) Vì sao pipeline phải gọi các hàm đó (cơ chế vận hành – Dev level)

## 8.1 `common.checkout()`

**Mục đích:** chuẩn hoá thông tin repo/branch để các bước sau dùng thống nhất.

**Nó làm gì:**

* Nếu chạy từ GitLab webhook: lấy `env.gitlabSourceRepoHomepage` và `env.gitlabBranch`
* Nếu chạy tay: lấy `params.GITLAB_REPOSITORY_URL` và `params.GITLAB_REFS`
* Từ repo URL suy ra:

  * `env.gitFullRepoName`
  * `env.gitRepoName` (tên repo cuối)

**Dev quan tâm gì:** nếu build ra `appLabel` lạ, thường do repo/branch input lạ.

---

## 8.2 `git.checkoutAndMerge()`

**Mục đích:** checkout đúng branch/tag với Git credential chuẩn.

**Nó làm gì:**

* Chuẩn hoá refs: nếu không bắt đầu bằng `refs/...` thì tự thêm `origin/<branch>`
* Checkout qua Jenkins GitSCM với `params.GIT_CRED_ID`
* Ghi lại commit: `env.gitRevParseHEAD`

**Dev quan tâm gì:** lỗi checkout = credential/branch/tag sai.

---

## 8.3 `common.sanityCicdEnv(valuesPath: 'CaaS/app')`

**Mục đích:** đây là bước “khóa an toàn” để pipeline chỉ chạy khi values hợp lệ.

**Nó làm gì:**

* Đọc `CaaS/app/values.yaml`
* Bắt buộc có: `env`, `version`, `workload.specs.image.repository`
* Set các biến môi trường dùng xuyên suốt:

  * `env.env`, `env.version`, `env.system`, `env.org`
  * `env.appLabel`: lấy từ values hoặc fail-safe theo repo name
  * `env.repository`: registry để build/push image
* Tạo file `sts-ordinal-per-cluster.yaml` (mặc định ordinals start=0 cho cluster mẫu)

**Dev quan tâm gì:**

* Pipeline fail ở đây = values.yaml thiếu trường bắt buộc hoặc sai path.

---

## 8.4 `common.setBuildMeta()`

**Mục đích:** giúp trace build: ai chạy, branch nào, commit nào.

**Nó làm gì:** set `currentBuild.displayName` và `currentBuild.description`.

**Dev quan tâm gì:** không ảnh hưởng deploy, chỉ giúp nhìn build rõ ràng.

---

## 8.5 `package.dockerBuild()` + `package.dockerPushDev()`

**Mục đích:** build/push image dựa trên values + git refs.

**Nó làm gì:**

* Sinh `env.imageName`, `env.imageTag`, `env.image`
* `docker build -t ${env.image} .`
* Login registry (credential DOCKER_CREDS)
* `docker push ${env.image}`

**Dev quan tâm gì:**

* Image cuối cùng pipeline build ra là gì (log có `env.image`)
* Nếu push fail: credential hoặc registry/repo.

---

## 8.6 `deploy.helmAppChart()` và `deploy.helmVSChart()`

**Mục đích:** deploy chart theo `KUBE_CONTEXTS` và values.

**Nó làm gì:**

* Với mỗi context trong `KUBE_CONTEXTS`:

  * load kubeconfig từ Jenkins credential file (credentialId = context)
  * `helm template` để render
  * `helm upgrade --install`
* App chart có fail-safe inject image:

  * nếu build đã set `env.imageName/env.imageTag` thì deploy sẽ `--set-string workload.specs.image.name/tag=...`

**Dev quan tâm gì:**

* Release Name và Namespace hiển thị trong log
* Deploy đa cluster: mỗi dòng context chạy 1 lượt.

---

## 8.7 `secZap.run()`

**Mục đích:** scan DAST cơ bản bằng ZAP container theo danh sách target.

**Nó làm gì:**

* Load YAML list
* Preflight curl để skip endpoint unreachable
* Run ZAP container baseline/api scan và output report

**Dev quan tâm gì:** URL phải reachable từ Jenkins agent.

---

## 8.8 `secPrisma.scanAndPublish()`

**Mục đích:** scan image đã build.

**Dev quan tâm gì:** image phải tồn tại local/registry theo scanner config (tuỳ plugin Prisma).

---

# 9) Runbook Dev: chạy lần đầu

1. Clone repo demo.
2. Chỉnh `CaaS/app/values.yaml` (bắt buộc: env/version/repository + port)
3. (Nếu cần Istio) chỉnh `CaaS/mesh/values.yaml` (gateway/host/prefix/canary)
4. (Nếu bật ZAP) chỉnh `CaaS/security/zapScanList.yaml`
5. Jenkins job:

   * set `KUBE_CONTEXTS` đúng (mỗi dòng 1 context)
   * set `NAMESPACE`
   * chạy build

---


ea1c9412-386e-4b51-af7e-d1e0dc3a3f9c
434e4b9aa4eb

wget -v https://ea1c9412-386e-4b51-af7e-d1e0dc3a3f9c:434e4b9aa4eb@archive.cloudera.com/p/cdh7/7.1.7.2000/parcels/CDH-7.1.7-1.cdh7.1.7.p2000.37147774-el8.parcel



wget -v https://gke-dc2-m2lib-dtu.seauat.com.vn/repository/cloudera-cdh7-parcels/7.1.7.2000/parcels/CDH-7.1.7-1.cdh7.1.7.p2000.37147774-el8.parcel


wget -v https://usr:pass@archive.cloudera.com/p/cdh7/7.1.7.2000/parcels/CDH-7.1.7-1.cdh7.1.7.p2000.37147774-el8.parcel
--2025-12-30 11:17:44--  https://usr:*password*@archive.cloudera.com/p/cdh7/7.1.7.2000/parcels/CDH-7.1.7-1.cdh7.1.7.p2000.37147774-el8.parcel
Resolving swg-url-proxy-https-sse.sigproxy.qq.opendns.com (swg-url-proxy-https-sse.sigproxy.qq.opendns.com)... 3.0.39.255
Connecting to swg-url-proxy-https-sse.sigproxy.qq.opendns.com (swg-url-proxy-https-sse.sigproxy.qq.opendns.com)|3.0.39.255|:443... connected.
Proxy request sent, awaiting response... 401 Authentication required
Authentication selected: Basic realm=Secured
Connecting to swg-url-proxy-https-sse.sigproxy.qq.opendns.com (swg-url-proxy-https-sse.sigproxy.qq.opendns.com)|3.0.39.255|:443... connected.
Proxy request sent, awaiting response... 200 OK
Length: 9128093468 (8.5G) [binary/octet-stream]
Saving to: ‘CDH-7.1.7-1.cdh7.1.7.p2000.37147774-el8.parcel.1’

CDH-7.1.7-1.cdh7.1.7.p2000.37147774-el8.parc   0%[                                                                                               ]   5.55M  1.18MB/s    eta 2h 9m  ^C

172.28.199.67