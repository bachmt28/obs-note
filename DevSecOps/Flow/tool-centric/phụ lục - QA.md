# FAQ – Giải pháp DevSecOps Ký Image, Kyverno, Evidence Store

## Nhóm 1: Câu hỏi nền tảng – “Tại sao phải làm phức tạp”

### Q1. Image đã có digest rồi, sao còn phải ký cho rườm rà?
**A:**
Digest chỉ đảm bảo **toàn vẹn nội dung**, không đảm bảo **được phép sử dụng**.
* Digest trả lời: *image này là cái gì*
* Chữ ký trả lời: *ai cho phép image này được deploy*
Không ký → hệ thống **không phân biệt được**:
* image build đúng pipeline
* image build ngoài pipeline
* image bị push đè tag
Ký + Kyverno biến “digest kỹ thuật” thành **artifact đã được chấp thuận**.
---
### Q2. Nếu dev không làm bậy thì cần gì ký?
**A:**
Hệ thống an toàn **không dựa trên giả định con người luôn đúng**.
* Hôm nay dev tốt
* Ngày mai dev mới
* Ngày kia bị lộ token
Thiết kế này **không chống dev**, mà **chống sai sót và rủi ro hệ thống**.
---
### Q3. Ký có ngăn được hết rủi ro không?

**A:**
Không. Và không hệ thống nào làm được.
Giải pháp này:
* **ngăn deploy artifact chưa được phép**
* **truy vết được khi có sự cố**
Mục tiêu là **kiểm soát + trách nhiệm**, không phải “miễn nhiễm tuyệt đối”.
---
## Nhóm 2: Câu hỏi về Jenkins & Dev có quyền cấu hình
### Q4. Dev có quyền config Jenkins UAT CI, vậy có lách được không?
**A:**
Dev **có thể build**, nhưng **không thể ký**.
* Private key ký nằm trong **ANTT zone**
* Dev không có quyền chạy job ký
* Kyverno chỉ cho deploy image đã ký
Dev có thể cố lách, nhưng:
* không deploy được
* hoặc deploy để lại đầy đủ dấu vết
---
### Q5. Nếu dev build image ngoài pipeline rồi push thẳng lên registry thì sao?
**A:**
Image đó:
* không có chữ ký
* không qua ANTT approve
→ Kyverno **chặn ngay tại admission**, không chạy được.
---
### Q6. Nếu dev push đè tag sau khi đã được ký thì sao?
**A:**
* Chữ ký gắn với **digest**
* Push đè tag → digest mới → **chữ ký không khớp**
Kyverno verify theo digest → reject.
---
## Nhóm 3: Câu hỏi về ANTT & con người
### Q7. ANTT ký nhầm thì sao?
**A:**
Có thể xảy ra, và thiết kế này **không phủ nhận điều đó**.
Khác biệt là:
* ký nhầm **có evidence**
* có người ký
* có bối cảnh ký
* truy vết được
Không ký → **không biết ai chịu trách nhiệm**.
---
### Q8. Tại sao không auto-approve bằng policy cho nhanh?
**A:**
Vì thực tế:
* không phải vulnerability nào cũng có impact như nhau
* nhiều finding phụ thuộc context hệ thống
ANTT cần **quyền đánh giá con người**, không phải bị ép bởi rule cứng.
Policy chỉ dùng cho:
* enforce kỹ thuật (image signed)
* không dùng để thay thế phán đoán chuyên môn.
---
### Q9. Automation verify chưa làm được thì có sao không?
**A:**
Không làm sụp thiết kế.
* Core giá trị nằm ở:
  * tách zone
  * ký
  * kyverno
  * append-only evidence
* Automation chỉ giúp:
  * giảm việc cho ANTT
  * giảm sai sót
Ước lượng: automation ~10% tổng giá trị, không phải nền móng.
---
## Nhóm 4: Câu hỏi về Evidence Store & DB

### Q10. Sao không ghi DB đầy đủ hết cho tiện?
**A:**
Vì:
* DB là **evidence**, không phải **data lake**
* ghi thừa:
  * tăng phức tạp
  * khó maintain
  * dễ sai lệch
Thiết kế chọn:
* **2 bảng core**
* payload JSON mở rộng
* append-only
→ đủ dùng, không over-engineer.
---
### Q11. Dev có quyền ghi evidence, vậy có giả mạo DB không?
**A:**
Dev **chỉ ghi được evidence của chính job mình**.
Nhưng:
* job sau **không tin dữ liệu đó**
* ANTT job luôn:
  * resolve lại digest từ registry
  * verify lại scan từ tool
Evidence không dùng để “tin”, mà dùng để **lần ngược và đối chiếu**.
---
### Q12  Đã cấu hình **UAT registry không cho push đè tag**, vậy ký còn cần thiết không?
**A**: Đúng, **UAT registry đã chặn push đè tag** để giảm rủi ro ở môi trường UAT. Tuy nhiên **ký vẫn cần thiết** vì:
1. **Immutable tag chỉ bảo vệ UAT, không bao trùm toàn chuỗi**
* Promote sang LIVE tạo **digest mới ở LIVE registry**
* Quyết định cho phép deploy LIVE **không thể suy ra chỉ từ việc UAT tag là immutable**
2. **Immutable tag không thay thế được “quyết định con người”**
* Nó nói: *tag không bị ghi đè*
* Nó **không nói**: *image này đã được ANTT chấp thuận dựa trên evidence nào*
3. **Kyverno chỉ hiểu chữ ký, không hiểu chính sách registry**
* Admission control cần một tín hiệu **portable, deterministic**
* Chữ ký theo digest là tín hiệu đó
4. **Forensic & trách nhiệm**
* Khi có sự cố, ký cho biết:
  * ai approve
  * khi nào
  * dựa trên evidence nào
* Immutable tag **không cung cấp thông tin trách nhiệm**
**Kết luận:**
* **UAT immutable tag = biện pháp giảm rủi ro vận hành**
* **Cosign + Kyverno = biện pháp kiểm soát cho phép deploy và truy trách nhiệm**
* Hai thứ **bổ trợ**, không thay thế
> UAT: *immutable tag + ký* → an toàn hơn, ít lách
> LIVE: *ký + kyverno* → bắt buộc, không phụ thuộc chính sách tag
---
## Nhóm 5: Câu hỏi về LIVE & runtime scan
### Q13. Sao LIVE không gate ZAP runtime?
**A:**
Vì vulnerability **trôi theo thời gian**.
* Approve hôm nay
* Mai có CVE mới
* Nếu gate cứng → deploy live thành roulette
LIVE dùng:
* ký image
* predeploy hygiene
* runtime scan thuộc **audit định kỳ**, không block deploy.
---
### Q14. Nếu sau khi deploy LIVE mới phát hiện lỗ hổng thì sao?
**A:**
Thiết kế này không ngăn discovery sau deploy.
Nhưng:
* biết chính xác image nào
* ai ký
* dựa trên evidence nào
→ xử lý theo **incident response**, không mù mờ.
---
## Nhóm 6: Câu hỏi chiến lược & so sánh
### Q15. Giải pháp này có giống Argo hay GitOps không?
**A:**
Giống về **mindset**, không giống về tool.
* Phân quyền rõ:
  * người build
  * người duyệt
  * hệ thống enforce
* Quyết định tách khỏi execution
Dùng Jenkins hay Argo chỉ là phương tiện.
---
### Q16. Có phải đang “vẽ vời quá mức” không?
**A:**
Không.
Vì:
* từng thành phần đều giải quyết **một rủi ro cụ thể**
* không thành phần nào thừa:
  * ký để xác nhận
  * kyverno để enforce
  * evidence để truy vết
Nếu bỏ một mắt xích:
* hệ thống vẫn chạy
* nhưng **không còn kiểm soát đáng tin**
---
### Q17. Nếu không làm theo mô hình này thì rủi ro là gì?
**A:**
Không phải “sập hệ thống”, mà là:
* không biết image từ đâu ra
* không biết ai approve
* không chứng minh được khi audit
* tranh cãi cảm tính khi có sự cố
Giải pháp này đổi “cãi nhau” thành “đối chiếu evidence”.
---
## Câu chốt cuối cho mọi đối tượng

> Đây không phải là giải pháp để làm hệ thống không bao giờ lỗi.
> Đây là giải pháp để khi lỗi xảy ra, mà là để **khi có chuyện thì lôi bằng chứng ra**.