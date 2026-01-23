Ngoài **Gitleaks**, còn nhiều công cụ khác (self-hosted lẫn dịch vụ cloud) chuyên dùng để **phát hiện secrets rò rỉ trong Git hoặc mã nguồn**. Dưới đây là bảng tổng hợp chi tiết:

---
## 🧩 So sánh các công cụ phát hiện secrets

| Tên công cụ                 | Loại                | Ưu điểm chính                                                                                   | Nhược điểm / Lưu ý                                                       | Chi phí                                |
| --------------------------- | ------------------- | ----------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------ | -------------------------------------- |
| **Gitleaks**                | Self-hosted         | - Mở nguồn, miễn phí- Dễ tích hợp CI/CD- Rất nhanh                                              | - Precision thấp (~46%)- False positive nhiều                            | ✅ Miễn phí                             |
| **TruffleHog**              | Self-hosted + Cloud | - Tìm được secrets cả trong mã hóa base64- Có thể quét cả API history                           | - Tốc độ chậm hơn- Cấu hình phức tạp                                     | 🔀 Cả miễn phí và trả phí              |
| **GitGuardian**             | SaaS (Cloud)        | - UI đẹp, alert theo thời gian thực- Tích hợp GitHub/GitLab/Azure dễ dàng- Có dashboard quản lý | - Dữ liệu đẩy lên cloud (yếu tố bảo mật)- Trả phí với repo private       | 💰 Trả phí (miễn phí giới hạn cho OSS) |
| **Detect Secrets (Yelp)**   | Self-hosted         | - Modular, có plugin custom- Precision khá cao khi tuning tốt                                   | - Setup cần nhiều effort- Không có sẵn rule tốt như Gitleaks             | ✅ Miễn phí                             |
| **Talisman (ThoughtWorks)** | Pre-commit tool     | - Ngăn ngay từ local- Nhẹ, dễ tích hợp                                                          | - Không quét được lịch sử Git- Rule hạn chế                              | ✅ Miễn phí                             |
| **AWS CodeGuru Reviewer**   | SaaS (AWS)          | - Kết hợp phân tích bảo mật và code quality- Gợi ý tự động                                      | - Chỉ dùng được trên repo CodeCommit hoặc GitHub- Trả phí theo line scan | 💰 Trả phí                             |

---
## 🔬 Đánh giá tổng quan

### 🎯 Nếu muốn **chạy nội bộ (self-hosted)**:
- **Tốt nhất để bắt đầu**: `Gitleaks`
- **Chuyên sâu hơn, tránh false-positive**: `Detect Secrets` (nhưng cần tuning kỹ)
- **Tìm base64 / các trick ẩn giấu**: `TruffleHog`
### ☁️ Nếu muốn **dịch vụ cloud – giám sát liên tục, cảnh báo UI đẹp**:
- **Tốt nhất**: `GitGuardian` – dùng rộng rãi trong ngành, có dashboard tốt, alert real-time.
- **Dành cho môi trường AWS**: `CodeGuru Reviewer` – tích hợp sâu, nhưng giá cao hơn.
---
## 🏆 Tóm tắt: Ai nên dùng gì?

| Trường hợp                       | Nên chọn gì                        |
| -------------------------------- | ---------------------------------- |
| Dự án open-source                | Gitleaks / GitGuardian (free tier) |
| Công ty cần CI/CD scan           | Gitleaks / TruffleHog              |
| Enterprise cần dashboard         | GitGuardian (Cloud SaaS)           |
| Dự án nội bộ bảo mật nghiêm ngặt | Detect Secrets (self-hosted)       |
| Quét code AWS tích hợp           | AWS CodeGuru                       |

---