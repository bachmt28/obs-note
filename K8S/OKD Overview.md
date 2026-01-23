# **BÁO CÁO ĐÁNH GIÁ: OKD TRONG NGỮ CẢNH KUBERNETES & SO SÁNH VỚI STACK HIỆN TẠI**

## **1. Tổng quan về OKD trong hệ sinh thái Kubernetes**

OKD là bản phân phối Kubernetes theo mô hình “distro”, tương tự cách các bản phân phối Linux được xây dựng từ nhân Linux.

OKD cung cấp một nền tảng **PaaS all-in-one** trên Kubernetes, bao gồm:

* Kubernetes control plane (được đóng gói lại)
* CRI-O (container runtime)
* OVN-Kubernetes hoặc OpenShift SDN (CNI)
* Ingress Operator (HAProxy)
* Container Registry tích hợp
* OAuth/SSO tích hợp
* Monitoring stack tự động (Prometheus, Alertmanager, Grafana)
* Quản trị OS/node qua MachineConfig Operator (MCO)
* Quản trị version toàn cluster qua ClusterVersion Operator (CVO)

OKD về bản chất là **upstream của OpenShift**, được Red Hat dùng làm nền để xây dựng OpenShift thương mại.

### Các thành phần CI/CD “gốc” của OKD

* BuildConfig
* Build
* ImageStream
* Source-to-Image (S2I)
* DeploymentConfig

Đây là hệ thống CI/CD đời cũ do Red Hat tự phát triển, tồn tại trước Tekton/Argo.

---

## **2. Quan hệ giữa OKD và Tekton/Argo**

Tekton và Argo **không phải** là lõi của OKD.

* **OpenShift Pipelines** = Tekton được đóng gói lại bằng Operator
* **OpenShift GitOps** = Argo CD được đóng gói lại bằng Operator

Cả hai đều là **addon**, không phải phần nền tảng của OKD.

Trong khi đó, CI/CD nguyên bản của OKD (BuildConfig, S2I, DC, ImageStream) là **đồ tự phát triển**, không dựa trên Tekton hay Argo.

---

## **3. Định hướng chiến lược của Red Hat**

* OKD tiếp tục được duy trì như upstream của OpenShift nhưng **không phải sản phẩm chủ lực**.
* Red Hat đầu tư mạnh vào:

  * Tekton (OpenShift Pipelines)
  * Argo CD (OpenShift GitOps)
  * Operators ecosystem
  * KubeVirt (OpenShift Virtualization)
  * OpenShift AI / Open Data Hub
  * Security Hardened Kubernetes

CI/CD gốc của OKD (BuildConfig, ImageStream, S2I) đã được xem là **legacy** và không còn định hướng phát triển mạnh.

---

## **4. Đánh giá tổng quan: dùng OKD chỉ như nền tảng K8s có nên không?**

Nếu chỉ sử dụng OKD trong vai trò:

* Nền tảng Kubernetes
* Lifecycle node OS
* Operator lifecycle

… mà không dùng các đặc sản như BuildConfig, ImageStream, DeploymentConfig, Route, S2I, Registry tích hợp, Console dev…

→ Lợi thế của OKD giảm đi rất nhiều.

Trong trường hợp này, OKD **không mang lại ưu thế quá lớn** so với tổ hợp:

* Rancher (multi-cluster management)
* Kubernetes thuần (RKE/RKE2/kubeadm)
* Helm (package deployment)
* Jenkins/Tekton (CI pipeline)
* Argo (CD/GitOps) (Nếu áp dụng)
* Nexus/Harbor (registry)

Stack hiện tại có tính linh hoạt cao hơn, dễ mở rộng, dễ tuning và không bị vendor lock-in.

---

## **5. Bảng so sánh OKD với stack hiện tại**

Bảng dưới đây đánh giá theo các khía cạnh: vận hành, maintain, upgrade, kiến trúc, CI/CD, registry, networking, security, multi-cluster, cost và chiến lược dài hạn.

| #   | Tiêu chí                    | OKD                                     | Stack hiện tại: Rancher + Jenkins + Helm + Nexus + K8s thuần                |
| --- | --------------------------- | --------------------------------------- | --------------------------------------------------------------------------- |
| 1   | Mô hình tổng thể            | Distro + PaaS all-in-one                | Kiến trúc composable, mô-đun do tổ chức định nghĩa                          |
| 2   | Triết lý                    | Opinionated, đóng gói sẵn               | Full customizable                                                           |
| 3   | Control plane               | Operator-driven, tối ưu sẵn             | Kubernetes thuần, tự do cấu hình                                            |
| 4   | Multi-cluster               | Có nhưng không phải thế mạnh            | Rancher là giải pháp chuyên trị multi-cluster                               |
| 5   | Distro lock-in              | Cao, phụ thuộc OKD/OpenShift            | Không lock-in                                                               |
| 6   | Cài đặt                     | Nặng, chuẩn hoá cao                     | Nhẹ hơn, nhiều lựa chọn                                                     |
| 7   | OS lifecycle                | MachineConfig Operator                  | Tự quản qua Ansible/image                                                   |
| 8   | Cluster upgrade             | CVO tự động, một lệnh                   | Tự chủ upgrade, linh hoạt                                                   |
| 9   | OS upgrade                  | Có pipeline chính thống                 | Tự xây pipeline                                                             |
| 10  | CNI                         | OVN/SDN mặc định                        | Calico/Cilium, linh hoạt hơn                                                |
| 11  | Ingress                     | HAProxy operator + Route                | Nginx/Traefik/Istio do đội tự chọn                                          |
| 12  | Registry                    | Tích hợp                                | Nexus/Harbor độc lập                                                        |
| 13  | External registry           | Hạn chế hơn (ưu tiên registry tích hợp) | Tối ưu hoá hoàn toàn                                                        |
| 14  | CI native                   | BuildConfig/S2I/IS (legacy)             | Jenkins/Tekton hiện đại                                                     |
| 15  | CD native                   | DeploymentConfig                        | Helm + Argo                                                                 |
| 16  | Tekton                      | Addon (OpenShift Pipelines)             | Tekton thuần (tùy chọn)                                                     |
| 17  | Argo                        | Addon (OpenShift GitOps)                | Argo thuần                                                                  |
| 18  | Jenkins                     | Hỗ trợ nhưng không phải triết lý gốc    | Thành phần cốt lõi                                                          |
| 19  | Helm                        | Hỗ trợ nhưng không phải workflow gốc    | Trụ cột chính                                                               |
| 20  | Policy                      | SCC + RBAC chặt                         | Kyverno + RBAC theo tổ chức                                                 |
| 21  | Security                    | SELinux enforcing mạnh                  | Tối ưu theo nhu cầu                                                         |
| 22  | Identity                    | OAuth/SSO tích hợp                      | Keycloak/SSO/LDAP linh hoạt                                                 |
| 23  | Multi-tenant                | Project-based                           | Namespace + Rancher Projects                                                |
| 24  | Observability               | Tích hợp sẵn                            | Elastic/Prom/Grafana theo thiết kế riêng                                    |
| 25  | Backup                      | Gắn với cơ chế OKD                      | Dựa vào Velero + storage Tổ chức                                            |
| 26  | Operator lifecycle          | Mạnh, OperatorHub lớn                   | Quản lý bằng Argo/Helm                                                      |
| 27  | Cloud/on-prem               | Tối ưu cho on-prem enterprise           | Hoàn toàn trung lập                                                         |
| 28  | Debug                       | Nhiều lớp operator gây phức tạp         | Debug trực tiếp K8s thuần                                                   |
| 29  | Customization               | Thấp, khó phá luật                      | Rất cao                                                                     |
| 30  | Performance overhead        | Lớn hơn, nhiều controller               | Gọn nhẹ hơn                                                                 |
| 31  | Resource tuning             | Ít linh hoạt                            | Tự tuning                                                                   |
| 32  | Dev console                 | Có                                      | Terminal ngay trên rancher, nhưng không mạnh bằng và ít bao quát hơn        |
| 33  | App DX                      | Hướng PaaS                              | Hướng CNCF workflow                                                         |
| 34  | Storage tích hợp            | Có                                      | Tùy SC                                                                      |
| 35  | Network security            | Nhiều ràng buộc                         | Policy theo thiết kế                                                        |
| 36  | Upgrade risk                | Nếu operator lỗi → recovery khó         | Upgrade từng phần, dễ rollback                                              |
| 37  | Fit CI/CD hiện tại          | Không phù hợp BuildConfig/IS            | Khớp 100% Jenkins/Helm/Tekton/Argo                                          |
| 38  | Fit workload (EMQX/Elastic) | Chạy được nhưng hay vướng SCC           | Dễ dàng tunning hơn                                                         |
| 39  | Chi phí                     | Miễn phí nhưng hướng OpenShift          | Không license                                                               |
| 40  | Tính lâu dài                | Phù hợp nếu định lên OpenShift          | Phù hợp với hướng CNCF mở, với mong muốn tích hợp linh hoạt các phần mềm mở |

---

## **6. Sum**

* OKD mạnh khi được dùng **đúng triết lý OpenShift**: OS lifecycle, security cứng, operator-driven, dev console, registry tích hợp, CI/CD native.
* Nếu chỉ dùng OKD làm “nền Kubernetes + operator lifecycle”, phần lớn giá trị của OKD **không được tận dụng**.
* Stack hiện tại (Rancher + Jenkins + Helm + Nexus + K8s thuần) mang lại:

  * Tính linh hoạt cao
  * Dễ mở rộng
  * Dễ phối hợp với công cụ DevSecOps hiện đại
  * Không phụ thuộc vendor
  * Phù hợp multi-site và kiến trúc nhiều cluster

Trong bối cảnh kiến trúc hiện tại, OKD không vượt trội đáng kể so với giải pháp hiện có, trừ khi có định hướng tương lai sử dụng OpenShift bản thương mại hoặc cần cơ chế OS lifecycle management tích hợp sâu.

---

1. Trade-off khi dùng **OKD trong môi trường multi-site**

---

## 1. Trade-off của OKD trong môi trường multi-site

### 1.1. Phạm vi và bối cảnh

Giả định các mô hình phổ biến:

* **Mô hình A – Multi-cluster**: mỗi site (DC1, DC2, DR…) là **một OKD cluster độc lập**, chia sẻ CI/CD, registry, GitOps, nhưng **không kéo giãn control plane**.
* **Mô hình B – Stretch cluster**: một OKD cluster duy nhất, master/worker trải đều nhiều site, etcd nằm trên nhiều DC (HA đa site).

Trong thực tế, mô hình B tiềm ẩn Trade-off rất cao; mô hình A an toàn hơn nhưng phức tạp vận hành.

---

### 1.2. Trade-off về topology và HA

#### 1.2.1. Trade-off khi kéo giãn 1 OKD cluster qua nhiều site (stretch cluster)

* **Độ trễ giữa các master/etcd**:
  OKD dựa nhiều vào etcd và operator. Khi latency giữa các master cao, các nguy cơ:

  * Mất quorum, cluster rơi vào trạng thái “không đọc không ghi” hoặc “đọc được nhưng khó update”.
  * Các operator (ClusterVersion, MachineConfig, Ingress, Network…) báo lỗi dây chuyền.

* **Ngắt kết nối mạng liên site (network partition)**:

  * Một site có thể nghĩ mình còn sống, site kia nghĩ ngược lại.
  * etcd có thể rơi vào split-brain hoặc mất khả năng commit mới.
  * Các tiến trình upgrade hoặc apply MachineConfig bị kẹt, cluster rơi vào trạng thái nửa vời.

* **Failover khó kiểm soát**:

  * Không giống mô hình “mỗi site một cluster”, việc mất một site trong stretch cluster có thể làm **sập toàn bộ control plane**, không chỉ site đó.

Kết luận: **stretch OKD cluster đa site là mô hình Trade-off cao**, không phù hợp môi trường cần ổn định.

#### 1.2.2. Trade-off khi dùng nhiều OKD cluster độc lập (multi-cluster)

* **Drift cấu hình**:

  * Mỗi cluster là một “thực thể” với bộ operator, version, policy, SCC riêng.
  * Nguy cơ lệch:

    * Version OKD khác nhau giữa site.
    * Operator khác version.
    * SCC, NetworkPolicy, ResourceQuota, LimitRange… không đồng nhất.
  * Dẫn tới: cùng một manifest, hành vi khác nhau ở từng site.

* **Đồng bộ upgrade khó hơn**:

  * Cần lập kế hoạch upgrade cho từng cluster, canh phiên bản tương thích (CI/CD, registry, runtime, driver).
  * Khi số cluster tăng, cửa sổ bảo trì nhân lên.

---

### 1.3. Trade-off vận hành và upgrade

#### 1.3.1. MachineConfig Operator (MCO) trong môi trường multi-site

* MCO áp chính sách cấu hình OS/nodes theo dạng **declarative**:

  * Thay đổi kernel args, sysctl, kubelet config, ignition…
  * Mỗi thay đổi có thể kéo theo reboot hàng loạt.
* Trong môi trường multi-site:

  * Nếu áp MCO chung cho nhiều cluster mà không test theo từng site, có thể gây **reboot đồng loạt** vào giờ không mong muốn.
  * Nhu cầu “mỗi site mỗi lịch bảo trì” trở nên khó khăn hơn.

Trade-off: **một lỗi cấu hình MCO có thể gây downtime diện rộng** trên nhiều site nếu không phân tách rõ profile.

#### 1.3.2. ClusterVersion Operator (CVO) và upgrade đồng bộ

* CVO chịu trách nhiệm orchestrate upgrade:

  * Nâng version control plane
  * Nâng version các cluster operator
* Trade-off trong multi-site:

  * Nếu dùng chiến lược “upgrade đồng thời nhiều cluster”, lỗi trong release hoặc bug operator → **nhiều site cùng gặp sự cố**.
  * Nếu upgrade lệch site (DC1 trước, DC2 sau), CI/CD có thể phải xử lý compatibility giữa phiên bản cũ và mới (API deprecation, CRD version).

---

### 1.4. Trade-off về tích hợp CI/CD và tooling

Trong khi stack hiện tại sử dụng:

* Jenkins / Tekton
* Helm
* Argo CD (nếu áp dụng)
* Nexus/Harbor
* Kyverno / Checkov / Prisma

thì OKD có thêm:

* Route
* SCC
* BuildConfig, ImageStream, DeploymentConfig (legacy)

Trade-off:

* **Divergence kiến trúc**:

  * Nếu một số site dùng K8s thuần + Rancher, một số site dùng OKD, pipeline và manifest có thể phải “rẽ nhánh”:

    * Site OKD dùng Route, SCC, annotation đặc thù
    * Site K8s thuần dùng Ingress, PSP-like, policy khác
  * Việc giữ chung một bộ manifest Helm/GitOps cho tất cả site khó hơn.

* **Coupling vào OKD API**:

  * Nếu tận dụng mạnh ImageStream, DeploymentConfig, Route…:

    * Khó migrate sang K8s thuần hoặc cluster khác không phải OKD.
    * Trong multi-site kiến trúc lai, CI/CD phải chứa logic rẽ nhánh theo loại cluster.

---

### 1.5. Trade-off bảo mật và policy

* **SCC khác biệt giữa cluster**:

  * Mỗi OKD cluster có thể có bộ SCC khác nhau (do history tùy biến).
  * Trong multi-site, cùng một workload có thể:

    * Chạy được ở site A
    * Bị chặn tại site B do SCC khác hoặc chặt hơn

* **Khó kiểm soát ngoại lệ**:

  * Khi số site tăng, số lượng request “mở quyền” (privileged, hostPath, hostNetwork…) tăng theo.
  * Nếu không có chính sách chung, đặc biệt là với multi-cluster, việc audit “ai được phép gì, ở site nào” trở nên khó.

---

### 1.6. Trade-off về vendor lock-in và linh hoạt kiến trúc

* Khi tận dụng tối đa đặc sản OKD (Route, SCC, BuildConfig, ImageStream, DeploymentConfig…):

  * Hệ thống gắn sâu vào OpenShift/OKD ecosystem.
  * Nếu sau này cần:

    * Chuyển sang K8s thuần
    * Đổi sang EKS/GKE/AKS
    * Hoặc quay về mô hình Rancher multi-cluster

  thì chi phí refactor manifest, pipeline, policy là không nhỏ.

* Trong môi trường multi-site, Trade-off này lớn hơn vì:

  * Số lượng workload/tài nguyên nhân lên theo số site.
  * Chi phí “gỡ OKD-đặc thù” không chỉ là 1 lần.

---

### 1.7. Trade-off tổ chức và vận hành thực tế

* Đội vận hành phải học thêm “OKD/OpenShift way”:

  * Route, Project, SCC, BuildConfig, ImageStream
  * Cách đọc log của operator, CVO, MCO
* Trong khi đã quen:

  * Rancher
  * K8s thuần
  * Jenkins + Helm + Argo

Nguy cơ:

* Split kiến thức: một số cụm chạy theo “OpenShift style”, một số cụm theo “K8s thuần style”.
* Tài liệu, quy trình, playbook phải viết hai bộ, tăng khả năng sai lệch.

---

### 1.8. Tóm tắt nhóm Trade-off chính

1. **Topology & HA**: stretch cluster đa site  có nhiều rủi ro, multi-cluster giảm rủi ro nhưng tăng drift và chi phí đồng bộ.
2. **Upgrade & OS lifecycle**: MCO/CVO mạnh nhưng chỉ cần 1 lỗi là ảnh hưởng hàng loạt node/cluster.
3. **CI/CD & tooling**: nguy cơ phân mảnh pipeline giữa OKD và K8s thuần; coupling vào API đặc thù OKD.
4. **Security & policy**: SCC và policy không đồng nhất giữa site gây khó kiểm soát.
5. **Vendor lock-in**: khó quay lại mô hình thuần CNCF/K8s khi đã phụ thuộc nhiều vào tính năng riêng.
6. **Tổ chức & kỹ năng**: tăng độ phức tạp đào tạo và vận hành khi tồn tại song song nhiều “cách dùng Kubernetes”.

---