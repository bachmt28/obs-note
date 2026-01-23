# **Google API**
> Link tham khảo: https://docs.cloud.google.com/kubernetes-engine/distributed-cloud/bare-metal/docs/installing/proxy

| Link                                  | Diễn giải                                               |
| ------------------------------------- | ------------------------------------------------------- |
| `*.gcr.io`                            | Kéo image container từ Artifact Registry.               |
| `accounts.google.com`                 | Xác thực OpenID, cung cấp public key xác minh token.    |
| `binaryauthorization.googleapis.com`  | Kiểm soát cho phép/từ chối image khi chạy trên cluster. |
| `cloudresourcemanager.googleapis.com` | Lấy metadata dự án Google Cloud kết nối với cluster.    |
| `compute.googleapis.com`              | Xác minh khu vực logging/monitoring.                    |
| `connectgateway.googleapis.com`       | Cho Google Support truy cập chẩn đoán read-only.        |
| `dl.google.com`                       | Tải và cài Google Cloud SDK.                            |
| `gkeconnect.googleapis.com`           | Giao tiếp hai chiều giữa cluster và Google Cloud.       |
| `gkehub.googleapis.com`               | Tạo và quản lý tài nguyên “fleet”.                      |
| `gkeonprem.googleapis.com`            | Quản lý vòng đời cluster on-prem.                       |
| `gkeonprem.mtls.googleapis.com`       | Như trên, dùng mTLS bảo mật cao hơn.                    |
| `iam.googleapis.com`                  | Tạo service account dùng cho xác thực API.              |
| `iamcredentials.googleapis.com`       | Điều khiển truy cập và audit telemetry.                 |
| `kubernetesmetadata.googleapis.com`   | Gửi metadata Kubernetes lên Google Cloud.               |
| `logging.googleapis.com`              | Ghi log và quản lý logging.                             |
| `monitoring.googleapis.com`           | Quản lý dữ liệu và cấu hình Cloud Monitoring.           |
| `oauth2.googleapis.com`               | Xác thực OAuth token.                                   |
| `opsconfigmonitoring.googleapis.com`  | Thu thập metadata Kubernetes để bổ sung metric.         |
| `releases.hashicorp.com`              | Tải Terraform client phục vụ quản trị hạ tầng.          |
| `securetoken.googleapis.com`          | Lấy refresh token phục vụ workload identity.            |
| `servicecontrol.googleapis.com`       | Ghi dữ liệu audit vào Cloud Audit Logs.                 |
| `serviceusage.googleapis.com`         | Kích hoạt và kiểm tra dịch vụ/API GCP.                  |
| `stackdriver.googleapis.com`          | Quản lý metadata observability/Stackdriver.             |
| `storage.googleapis.com`              | Object storage (bucket, artifact…).                     |
| `sts.googleapis.com`                  | Dịch vụ đổi token truy cập tạm thời.                    |
| `www.googleapis.com`                  | Xác thực token dịch vụ.                                 |

---
# **Lib Repo**

| Link                                                     | Diễn giải                                 |
| -------------------------------------------------------- | ----------------------------------------- |
| packages.confluent.io                                    | Kho thư viện Confluent Kafka.             |
| download.docker.com                                      | Kho Docker Engine và runtime.             |
| quay.io                                                  | Registry image OCI.                       |
| api.nuget.org/v3/index.json                              | Nguồn NuGet cho .NET.                     |
| [http://registry.npmjs.org/](http://registry.npmjs.org/) | Kho package NPM công cộng.                |
| plugins.gradle.org/m2/                                   | Kho plugins Gradle dạng Maven.            |
| updates.jenkins.io/                                      | Metadata plugin Jenkins.                  |
| pypi.org/                                                | Kho package Python.                       |
| repo.jenkins-ci.org/                                     | Kho Maven Jenkins.                        |
| repo1.maven.org/maven2/                                  | Maven Central chính thức.                 |
| packages.gitlab.com/                                     | Repo gói GitLab.                          |
| rhc.sonatype.com                                         | Metadata Sonatype/Nexus.                  |
| files.pythonhosted.org                                   | Mirror link thư viện Wheel/Source Python. |
| westeurope.cloudflare.jenkins.io                         | Mirror Jenkins EU.                        |
| nxrm.telemetry.sonatype.com                              | Telemetry Sonatype.                       |
| cdn.download.sonatype.com                                | CDN Sonatype.                             |
| links.sonatype.com                                       | Metadata/docs redirector.                 |
| pkg.jenkins.io                                           | Repo deb/rpm Jenkins.                     |

---
# **Helm Chart**

| Link                                                       | Diễn giải                         |
| ---------------------------------------------------------- | --------------------------------- |
| ansible-community.github.io/awx-operator-helm/             | Helm chart AWX Operator.          |
| charts.jenkins.io/                                         | Helm chart Jenkins.               |
| charts.jetstack.io                                         | Helm chart Cert-Manager.          |
| charts.ververica.com                                       | Helm chart Ververica (Flink).     |
| emberstack.github.io/helm-charts                           | Helm charts tiện ích/dịch vụ.     |
| fluent.github.io/helm-charts                               | Helm chart Fluent Bit/Fluentd.    |
| helm.elastic.co                                            | Helm chart Elastic stack.         |
| kubernetes-sigs.github.io/nfs-subdir-external-provisioner/ | Helm chart NFS provisioner.       |
| nextcloud.github.io/helm/                                  | Helm chart Nextcloud.             |
| prometheus-community.github.io/helm-charts                 | Helm chart Prometheus stack.      |
| qdrant.github.io/qdrant-helm                               | Helm chart Qdrant vector DB.      |
| releases.rancher.com/server-charts/latest                  | Rancher chart (latest).           |
| releases.rancher.com/server-charts/stable                  | Rancher chart (stable).           |
| repos.emqx.io/charts                                       | Helm chart EMQX broker.           |
| trinodb.github.io/charts                                   | Helm chart Trino SQL.             |
| victoriametrics.github.io/helm-charts/                     | Helm chart VictoriaMetrics.       |
| registry-1.docker.io                                       | Registry DockerHub.               |
| repo.maven.apache.org                                      | Maven Central (chart dependency). |
| registry.ververica.com                                     | Registry Ververica OCI.           |
