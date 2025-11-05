
---
# **Tài Liệu Nâng Cấp Hệ Thống Rancher Cài Đặt Bằng Docker (Single Node)**

**Mục tiêu:**  
Thực hiện nâng cấp Rancher phiên bản mới nhất trong môi trường Single Node cài qua Docker, đảm bảo an toàn dữ liệu và khả năng rollback khi cần thiết.

**Phạm vi áp dụng:**  
Áp dụng cho môi trường Rancher single-node triển khai bằng Docker (không dùng HA mode hoặc Kubernetes cluster).

---

## 1. **Kiểm tra và sao lưu trước khi nâng cấp**

### 1.1 Xác định phiên bản Rancher đang sử dụng

```bash
docker ps | grep rancher/rancher
```

### 1.2 Backup toàn bộ dữ liệu Rancher

Rancher sử dụng volume Docker mặc định là `rancher-data`. Thực hiện sao lưu bằng lệnh sau:

```bash
docker run --rm \
  -v rancher-data:/volume \
  -v $(pwd):/backup \
  alpine \
  tar czvf /backup/rancher-data-backup.tar.gz /volume
```

> 🔒 **Ghi chú:** Đảm bảo thư mục `$(pwd)` có đủ dung lượng và quyền ghi.

---

## 2. **Tạm dừng và gỡ container Rancher cũ**

### 2.1 Dừng container hiện tại:

```bash
docker stop <container_id>
```

### 2.2 Xoá container:

```bash
docker rm <container_id>
```

> Thay `<container_id>` bằng ID hoặc tên container từ lệnh `docker ps`.

---

## 3. **(Tuỳ chọn) Xoá image Rancher cũ**

```bash
docker images | grep rancher/rancher
docker rmi rancher/rancher:<old_version>
```

> ⚠️ Không xoá Docker Volume `rancher-data`.

---

## 4. **Tải về image Rancher mới**

```bash
docker pull rancher/rancher:<new_version>
```

Ví dụ:

```bash
docker pull rancher/rancher:v2.11.3
```

---

## 5. **Khởi động lại container Rancher với phiên bản mới**

```bash
docker run --name rancherv2101 -d --volumes-from rancher-data --restart=unless-stopped   -p 80:80 -p 443:443   -v ~/cert.pem:/etc/rancher/ssl/cert.pem  -v ~/key.pem:/etc/rancher/ssl/key.pem  -e HTTPS_PROXY="http://dc2-proxyuat.seauat.com.vn:8080" -e HTTP_PROXY="http://dc2-proxyuat.seauat.com.vn:8080" -e NO_PROXY="localhost,127.0.0.1,0.0.0.0,10.0.0.0/8,172.16.0.0/12,.svc,.cluster.local,.seabank.com.vn,.seauat.com.vn" --privileged   rancher/rancher:v2.11.3   --no-cacerts
```

---

## 6. **Kiểm tra sau nâng cấp**

- Truy cập trình duyệt vào địa chỉ: `https://<IP hoặc domain Rancher>`
    
- Kiểm tra các chức năng chính:
    
    - Giao diện dashboard
        
    - Các cluster đang quản lý
        
    - Auth, access control, tokens
        
    - Các Apps/Catalogs (nếu có)
        
- Kiểm tra log container:
    
    ```bash
    docker logs -f <container_id>
    ```
    

---

## 7. **Phương án khôi phục (Rollback)**

Trong trường hợp lỗi nghiêm trọng sau nâng cấp:

### 7.1 Xoá volume hiện tại

```bash
docker stop <container_id>
docker rm <container_id>
docker volume rm rancher-data
```

### 7.2 Tạo lại volume và phục hồi từ backup

```bash
docker volume create rancher-data

docker run --rm \
  -v rancher-data:/volume \
  -v $(pwd):/backup \
  alpine \
  tar xzvf /backup/rancher-data-backup.tar.gz -C /
```

### 7.3 Khởi chạy lại Rancher với image cũ:

```bash
docker run -d --restart=unless-stopped \
  -p 80:80 -p 443:443 \
  -v /etc/localtime:/etc/localtime:ro \
  -v rancher-data:/var/lib/rancher \
  rancher/rancher:<old_version>
```

---

## 8. **Kiểm tra & xác nhận sau rollback**

- Truy cập lại Rancher và xác minh toàn bộ dữ liệu đã khôi phục đúng.
    
- Kiểm tra logs đảm bảo không còn lỗi bất thường.
    

---

## 10. **Phụ lục**

### 10.1 Kiểm tra volume hiện có

```bash
docker volume ls
```

### 10.2 Kiểm tra vị trí lưu trữ volume thực tế

```bash
docker volume inspect rancher-data
```

---