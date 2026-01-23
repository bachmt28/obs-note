**Phiên bản áp dụng:**
* CDH: `7.3.1.500`
* Cloudera Manager: `7.13.1.500`
* Hệ điều hành: **RHEL 9.6**

---
# 1. Danh sách máy chủ

| Máy chủ                  | IP            | Hệ điều hành | Vai trò            |
| ------------------------ | ------------- | ------------ | ------------------ |
| dc1d-cloudera-gw-util-01 | 172.28.70.200 | RHEL 9.6     | Gateway + Tiện ích |
| dc1d-cloudera-master-01  | 172.28.70.201 | RHEL 9.6     | Master             |
| dc1d-cloudera-worker-01  | 172.28.70.203 | RHEL 9.6     | Worker             |
| dc1d-cloudera-worker-02  | 172.28.70.204 | RHEL 9.6     | Worker             |
| dc1d-cloudera-worker-03  | 172.28.70.205 | RHEL 9.6     | Worker             |

> **Lưu ý:** License SeABank giới hạn 5 node → kiến trúc triển khai tối thiểu.
> Các Service liệt kê ở sơ đồ sẽ active cụ thể sau
![[Pasted image 20251231084157.png]]
---
# 2. Chuẩn bị hệ thống
## 2.1 Yêu cầu dung lượng đĩa

* `/var/lib` : ~100GB (Gateway + Master)
* `/data` : ~500GB (Worker)

---
## 2.2 Chuẩn bị hệ điều hành (TẤT CẢ NODE)
> [Tài liệu tham khảo](https://docs.cloudera.com/cdp-private-cloud-base/7.3.1/cdp-private-cloud-base-installation/topics/cdpdc-before-you-install.html)
### 2.2.1 Mount ISO RHEL và tạo repo nội bộ
```bash
mount /dev/cdrom /mnt/iso
```
```bash
cat > /etc/yum.repos.d/localiso.repo <<EOF
[LocalISO-BaseOS]
name=RHEL 9.6 BaseOS
baseurl=file:///mnt/iso/BaseOS
enabled=1
gpgcheck=0

[LocalISO-AppStream]
name=RHEL 9.6 AppStream
baseurl=file:///mnt/iso/AppStream
enabled=1
gpgcheck=0
EOF
```
### 2.2.2 Cài gói cơ bản
```bash
# Cài đặt Java
dnf install -y java-11-openjdk

# Require python cho postgre db
dnf install -y python3 
dnf install -y python3-devel
dnf install -y python3-pip
dnf install -y postgresql-jdbc
pip3 install psycopg2-binary --verbose --index-url https://gke-dc2-m2lib-dtu.seauat.com.vn/repository/pypi-all/simple/ --trusted-host gke-dc2-m2lib-dtu.seauat.com.vn
```
### 2.2.3 Tối ưu hệ thống
```bash
# Set swap theo recommend
sysctl -w vm.swappiness=10

# Disable huge page
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
```
### 2.2.4 Thiết lập hostname *(bỏ qua nếu đã thực hiện rồi)*
```bash
hostnamectl set-hostname <hostname>
```
### 2.2.5 Cấu hình /etc/hosts
```bash
cat >> /etc/hosts <<EOF
172.28.70.200 dc1d-cloudera-gw-util-01
172.28.70.201 dc1d-cloudera-master-01
172.28.70.203 dc1d-cloudera-worker-01
172.28.70.204 dc1d-cloudera-worker-02
172.28.70.205 dc1d-cloudera-worker-03
EOF
```
### 2.2.6 Thiết lập SSH không mật khẩu
```sh
ssh-keygen -t rsa -b 4096
```
```sh
for h in dc1d-cloudera-gw-util-01 dc1d-cloudera-master-01 dc1d-cloudera-worker-{01,02,03}; do
  ssh-copy-id $h
done
```

### 2.2.7 Tắt firewall và SELinux
```bash
systemctl disable --now firewalld
setenforce 0
```
---
# 2.3 Repository Cloudera bằng Nexus
> **(PHƯƠNG ÁN CACHE LOCAL REPO CHÍNH THỨC)**
## 2.3.1 Tạo repository trên Nexus
- NEXUS_URL: https://gke-dc2-m2lib-dtu.seauat.com.vn
*  Tạo repo:
	* Name: `cloudera-raw`
	* type: `raw / hosted`
## 2.3.2 Khởi tạo nexus repo cho RPM CM và parcel cdh
```bash
# Set env to load
USR=ea1c9412-386e-4b51-af7e-d1e0dc3a3f9c
PSW=434e4b9aa4eb
export https_proxy=https://swg-url-proxy-https-sse.sigproxy.qq.opendns.com

NEXUS_URL=gke-dc2-m2lib-dtu.seauat.com.vn
NEXUS_USR=tung.ns5
NEXUS_PSW=Cdao0713

export no_proxy=$NEXUS_URL
export NO_PROXY=$NEXUS_URL

# CM Download
BASE=/data/tmp/offline/cm7/

mkdir -p $BASE
cd $BASE
wget https://$USR:$PSW@archive.cloudera.com/p/cm7/7.13.1.500/repo-as-tarball/cm7.13.1.500-redhat9.tar.gz
tar xvf cm7.13.1.500-redhat9.tar.gz

# CM Upload
cd $BASE
find . -type f -print0 | while IFS= read -r -d '' f; do
  curl -u "$NEXUS_USR:$NEXUS_PSW" --upload-file "$f" "https://${NEXUS_URL}/repository/cloudera-raw/cm/${f#./}"
done


# CDH7 Download
BASE=/data/tmp/offline
VER=cdh7/7.3.1.500
mkdir -p $BASE/$VER
cd $BASE/$VER
export https_proxy=https://swg-url-proxy-https-sse.sigproxy.qq.opendns.com
wget https://$USR:$PSW@archive.cloudera.com/p/$VER/parcels/{manifest.json,CDH-7.3.1-1.cdh7.3.1.p500.71942734-el9.parcel,CDH-7.3.1-1.cdh7.3.1.p500.71942734-el9.parcel.sha1,CDH-7.3.1-1.cdh7.3.1.p500.71942734-el9.parcel.sha256}

# CDH7 Upload
cd $BASE/$VER
for f in *; do
  curl -u "$NEXUS_USR:$NEXUS_PSW" --upload-file "$f" "https://${NEXUS_URL}/repository/cloudera-raw/$VER/parcels/${f#./}"
done
```
## 2.3.3 Cấu hình YUM repo

```bash
NEXUS_URL=gke-dc2-m2lib-dtu.seauat.com.vn
cat > /etc/yum.repos.d/cloudera.repo <<EOF
[cloudera-cm7.13.1.500]
name=Cloudera CM Nexus Repo cm7.13.1.500
baseurl=https://gke-dc2-m2lib-dtu.seauat.com.vn/repository/cloudera-raw/cm/cm7.13.1.500/
enabled=1
gpgcheck=0

EOF
```
---
# 2.4 Repository Cloudera bằng httpd
> **(DEPRECATED – PHƯƠNG ÁN DỰ PHÒNG)**
> ⚠️ **CHỈ DÙNG KHI NEXUS KHÔNG KHẢ DỤNG** . Xem mục 2.3
## 2.4.1 Cài httpd
```bash
yum install -y httpd
```
## 2.4.2 Cấu hình MIME cho parcel
```bash
vi /etc/httpd/conf/httpd.conf
```
Thêm:
```apache
AddType application/x-gzip .gz .tgz .parcel
```
## 2.4.3 Khởi động httpd

```bash
systemctl enable --now httpd
mkdir -p /var/www/html/cloudera
chmod -R 755 /var/www/html/cloudera
```
## 2.4.4 Tải CDH parcels
```bash
cd /var/www/html/cloudera
wget https://archive.cloudera.com/p/cdh7/7.3.1.500/parcels/manifest.json
wget https://archive.cloudera.com/p/cdh7/7.3.1.500/parcels/CDH-7.3.1-1.cdh7.3.1.p500.71942734-el9.parcel
wget https://archive.cloudera.com/p/cdh7/7.3.1.500/parcels/CDH-7.3.1-1.cdh7.3.1.p500.71942734-el9.parcel.sha1
wget https://archive.cloudera.com/p/cdh7/7.3.1.500/parcels/CDH-7.3.1-1.cdh7.3.1.p500.71942734-el9.parcel.sha256
```
## 2.4.5 Tải Cloudera Manager RPM
```bash
wget -r -np -nd -A "*.rpm" \
https://archive.cloudera.com/p/cm7/7.13.1.500/redhat9/yum/RPMS/x86_64/
```
## 2.4.6 Tạo repo yum
```bash
createrepo /var/www/html/cloudera
```
## 2.4.7 Cấu hình repo trên các node
```bash
cat > /etc/yum.repos.d/cloudera-httpd.repo <<EOF
[cloudera-httpd]
name=Cloudera HTTPD Repo (Fallback)
baseurl=http://dc1d-cloudera-gw-util-01/cloudera
enabled=1
gpgcheck=0
EOF
```
---
# 3. Chuẩn bị Database (PostgreSQL)
> [Tài liệu tham khảo](https://docs.cloudera.com/cdp-private-cloud-base/7.3.1/cdp-private-cloud-base-installation/topics/cdpdc-configuring-starting-postgresql-server.html)
> dc1u-pgdb-01.seauat.com.vn
## Danh sách database cụ thể cần tạo
### DANH SÁCH DATABASE POSTGRES – METADATA CỤM CLOUDERA

| Services                           | Databases (postgres) | User           | Dùng cho lần upgrade này | Pasword      | Note                                                                                                                                                        |
| ---------------------------------- | -------------------- | -------------- | :----------------------- | ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Cloudera Manager Server            | scm                  | scm            | ✔️                       | SB#26DagaDSg | Chứa tất cả thông tin về các dịch vụ bạn đã cấu hình và phân công vai trò cho chúng, toàn bộ lịch sử cấu hình, lệnh, người dùng và các tiến trình đang chạy |
| Report Manager                     | rman                 | rman           | ✔️                       | Rman#2026    | Theo dõi mức độ sử dụng ổ đĩa và các hoạt động xử lý theo thời gian                                                                                         |
| Hue                                | hue                  | hue            | ✖️                       |              | Chứa thông tin tài khoản người dùng, các yêu cầu công việc và các truy vấn Hive                                                                             |
| Hive Metastore Server              | metastore            | hive           | ✔️                       | Hive#2022    | Chứa dữ liệu Hive (Hive Service)                                                                                                                            |
| Sentry Server                      | sentry               | sentry         | ✔️                       |              | Chứa dữ liệu phân quyền                                                                                                                                     |
| YARN Queue Manager                 | yarn_qm              | yarn           | ✔️                       |              | Chứa dữ liệu từ Yarn (Yarn Service)                                                                                                                         |
| Ranger                             | ranger               | rangeradmin    | ✖️                       |              | Chứa thông tin quản trị người dùng Ranger, nhóm và chính sách truy cập                                                                                      |
| Ranger KMS                         | rangerkms            | rangerkms      | ✖️                       |              | Lưu trữ các khóa được mã hóa                                                                                                                                |
| Oozie                              | oozie                | oozie          | ✖️                       |              | Chứa dữ liệu về quy trình làm việc, điều phối viên và gói của Oozie                                                                                         |
| Schema Registry                    | schemaregistry       | schemaregistry | ✖️                       |              | Chứa các lược đồ và dữ liệu cấu trúc, tất cả các phiên bản và nhánh                                                                                         |
| Streams Messaging Manager          | smm                  | smm            | ✖️                       |              | Chứa siêu dữ liệu Kafka, lưu trữ các chỉ số và định nghĩa cảnh báo                                                                                          |
| Cloudera Navigator Audit Server    | nav                  | nav            | ✖️                       |              | Chứa thông tin audit (CDH 5,6)                                                                                                                              |
| Cloudera Navigator Metadata Server | navms                | navms          | ✖️                       |              | Bao gồm dữ liệu về quyền, chính sách và báo cáo audit (CDH 5,6)                                                                                             |
| Sqoop                              | sqoop                | sqoop          | ✖️                       |              | Bao gồm các thông tin như trình kết nối, trình điều khiển, liên kết và công việc (CDH 5,6)                                                                  |
|                                    |                      |                |                          |              |                                                                                                                                                             |
## SQL khởi tạo db *(thực hiện đối với từng db)*
```sql
CREATE ROLE scm LOGIN PASSWORD '***';
CREATE DATABASE scm OWNER scm ENCODING 'UTF8';
ALTER DATABASE scm SET standard_conforming_strings TO 'off';
```

---
# 4. Cài đặt Cloudera Manager
## 4.1 Cài Server *(thực hiện trên node `gateway`)*
```bash
dnf install -y cloudera-manager-server
```
- NOTE:
> Trên RHEL 9.x, khi cài đặt Cloudera Manager 7.13.x có thể xuất hiện lỗi cài đặt package chkconfig do xung đột /etc/init.d.
> Đây là dependency legacy không còn sử dụng trên systemd. Lỗi này KHÔNG ảnh hưởng đến việc vận hành Cloudera Manager và có thể bỏ qua an toàn.

## 4.2 Cài Agent *(TẤT CẢ NODE)*

```bash
gateway_host=dc1d-cloudera-gw-util-01
dnf install -y cloudera-manager-agent
sed -i "s/server_host=localhost/server_host=${gateway_host}/" /etc/cloudera-scm-agent/config.ini

# dnf install -y cloudera-manager-daemons
```
## 4.3 Khởi tạo schema
1. Khởi tạo:
```bash
cd /opt/cloudera/cm/schema
./scm_prepare_database.sh postgresql -h dc1u-pgdb-01.seauat.com.vn scm scm
	Enter SCM password: # <--- Nhập password đã được dba cung cấp
```
2. Verify:
```sh
cat /etc/cloudera-scm-server/db.properties
```
- Kết quả kỳ vọng cần có: 

> com.cloudera.cmf.db.type=postgresql
> com.cloudera.cmf.db.host=dc1u-pgdb-01.seauat.com.vn
> com.cloudera.cmf.db.name=scm
> com.cloudera.cmf.db.user=scm
> com.cloudera.cmf.db.setupType=EXTERNAL
> com.cloudera.cmf.db.password=***********

## 4.4 Khởi động dịch vụ
- Gateway:
```bash
systemctl start cloudera-scm-server
systemctl start cloudera-scm-agent
```
	- check:
```sh
systemctl status cloudera-scm-server
systemctl status cloudera-scm-agent
```
- Node còn lại
```sh
systemctl start cloudera-scm-agent
systemctl status cloudera-scm-agent
```
---
# 5. Khởi tạo Cluster
1. Truy cập UI Cloudera Manager
2. Kích hoạt license
3. Tạo cluster
4. Gán role theo thiết kế

---
# 6. Init các service qua UI
## Init base add cluster 
![[Pasted image 20260109110853.png]]
![[Pasted image 20260109110910.png]]
![[Pasted image 20260109110955.png]]
![[Pasted image 20260109111638.png]]

### Init Service 
- Add Cluster - Configuration  -> CustomService -> ZooKeeper
![[Pasted image 20260109111830.png]]
- Set dababase rman
-![[Pasted image 20260109112026.png]]
- Enter Required Parameters Skip
- Nếu không có gì thay đổi, giữ như mặc định. Chú ý phân vùng /var/lib/zookeeper đảm bảo đủ dung lượng 
![[Pasted image 20260109112504.png]]
- Init zookeepper lỗi do thiếu java home 
	- set qua host configuration và restart lại 
	- như hình:
	- ![[Pasted image 20260109112838.png]]
	- Restart zookeper instnace và đợi 1 lúc cho zoo cannary
### Add Service HDFS
![[Pasted image 20260109114246.png]]![[Pasted image 20260109114320.png]]
![[Pasted image 20260109114409.png]]- Nếu không quy hoạch phân vùng dfs riêng thì để mặc định như hình . Chú ý đảm bảo dung lượng
![[Pasted image 20260109114528.png]]
- Finish
### Add Service YARN
![[Pasted image 20260109115553.png]]- default
![[Pasted image 20260109115616.png]]![[Pasted image 20260109115757.png]]

### Add Service HIVE
![[Pasted image 20260109120156.png]]
![[Pasted image 20260109120245.png]]
![[Pasted image 20260109120424.png]]
### Add Service ICEBERGE
Tự có sau khi cài HIVE
Nhớ start lên 
### Add Service SPARK 3
![[Pasted image 20260109121328.png]]
default
![[Pasted image 20260109121351.png]]
![[Pasted image 20260109121516.png]]
- Start
### Add Service IMPALA
![[Pasted image 20260109121802.png]]![[Pasted image 20260109121907.png]]
![[Pasted image 20260109122035.png]]![[Pasted image 20260109123706.png]]