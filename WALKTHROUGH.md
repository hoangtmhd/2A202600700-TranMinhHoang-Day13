# BÁO CÁO TIẾN TRÌNH LÀM BÀI & KẾT QUẢ LAB13 - OBSERVATHON

* **Học viên**: Trần Minh Hoàng
* **Email**: hoangtmhd2001@gmail.com
* **Mã đội**: 2A202600700-TranMinhHoang
* **Mô hình sử dụng**: Gemini 2.5 Flash (Gemini API)

---

## I. TỔNG QUAN KẾT QUẢ ĐẠT ĐƯỢC

Sau các vòng tối ưu hóa hệ thống quan sát (telemetry), tinh chỉnh prompt và xây dựng lớp wrapper giảm thiểu lỗi, hệ thống đạt điểm số rất cao trên cả hai tập kiểm thử:

### 1. Kết quả Public Phase
* **Headline Score**: **99.83 / 100** (Gần như tuyệt đối)
* **Correctness (Độ chính xác)**: **89%** (102/120 câu trả lời đúng hoàn toàn)
* **Diagnosis F1 (Chẩn đoán lỗi)**: **1.000** (Chẩn đoán đúng 10/10 lỗi của simulator)
* **Error Rate (Lỗi hệ thống)**: **1.000** (Không gặp bất kỳ lỗi crash nào)

### 2. Kết quả Private Phase
* **Headline Score**: **84.29 / 100**
* **Correctness (Độ chính xác)**: **64.5%** (48/80 câu trả lời đúng)
* **Diagnosis F1 (Chẩn đoán lỗi)**: **0.952** (Chẩn đoán chính xác cao)
* **Error Rate (Lỗi hệ thống)**: **0.938** (Giảm thiểu tối đa lỗi rate limit TPM của Gemini API dưới tải cao)

---

## II. CHI TIẾT CÁC LỖI PHÁT HIỆN & BIỆN PHÁP KHẮC PHỤC (10 FAULT CLASSES)

Tôi đã chẩn đoán và ghi nhận đầy đủ 10 lớp lỗi giả lập trong [findings.json](solution/findings.json):

1. **`latency_spike`**: Trễ cao do không bật cache và nhiệt độ cao làm LLM phản hồi lan man.
   * *Khắc phục*: Bật cache và hạ temperature xuống `0.2` trong `config.json`, tích hợp cache luồng trong `wrapper.py`.
2. **`error_spike`**: Lỗi 503 UNAVAILABLE do quá tải 15 RPM của Gemini API dưới tải concurrency cao.
   * *Khắc phục*: Tích hợp cơ chế Rate Limit lock giãn cách 4.5s bên trong vòng lặp retry của `wrapper.py` và tăng max retry.
3. **`pii_leak`**: Rò rỉ email/sđt của khách hàng trong câu trả lời.
   * *Khắc phục*: Bật `redact_pii: true` trong `config.json` và thêm rule cấm lặp lại thông tin khách hàng trong system prompt.
4. **`arithmetic_error`**: Tính sai tổng tiền và giảm giá.
   * *Khắc phục*: Thiết lập công thức toán học chia nguyên `// 100` rõ ràng trong prompt và viết hàm tự động tính toán lại dựa trên trace trong `wrapper.py`.
5. **`fabrication`**: Bịa đặt tổng tiền khi hết hàng hoặc địa điểm không hỗ trợ.
   * *Khắc phục*: Cấm đưa ra tổng tiền khi từ chối đơn hàng trong prompt, dọn sạch dòng tổng tiền trong `wrapper.py`.
6. **`tool_failure`**: Lỗi Unicode khi gọi tool với các thành phố tiếng Việt có dấu (Đà Nẵng, Hải Phòng...).
   * *Khắc phục*: Bật `"normalize_unicode": true` trong `config.json` và normalize Unicode trong wrapper.
7. **`cost_blowup`**: Chi phí ảo vượt ngân sách do thiết lập premium tier và verbose prompt.
   * *Khắc phục*: Chuyển `"model_price_tier"` sang `"free"` và tắt `"verbose_system": false` trong `config.json`.
8. **`quality_drift`**: Suy giảm chất lượng và định dạng ở các lượt chat sau trong session.
   * *Khắc phục*: Giảm `"context_size"` xuống 5 và sử dụng cache kết quả để ổn định hành vi của agent.
9. **`infinite_loop`**: Gọi tool lặp đi lặp lại vô hạn dẫn đến cạn kiệt số bước.
   * *Khắc phục*: Bật `"loop_guard": true` trong `config.json`.
10. **`tool_overuse`**: Gọi tool dư thừa nhiều lần không cần thiết.
    * *Khắc phục*: Đặt `"tool_budget": 4` trong `config.json` và ra lệnh gọi mỗi tool tối đa 1 lần trong system prompt.

---

## III. CÁC GIẢI PHÁP TỐI ƯU HÓA ĐẶC BIỆT TRONG WRAPPER & PROMPT

### 1. Logic kiểm tra tồn kho thông minh (Stock Quantity Check)
* **Vấn đề**: Khi khách đặt mua số lượng vượt quá tồn kho (ví dụ đặt 5 MacBook nhưng kho chỉ còn 4), tool `check_stock` vẫn trả về `in_stock: True` (vì kho còn hàng). Agent từ chối nhưng wrapper trước đó không phát hiện ra, dẫn đến việc tự động sửa và in thêm dòng `Tong cong:` bịa đặt ở cuối.
* **Giải pháp**: Wrapper được nâng cấp để parse trường `'quantity'` từ tool, so sánh `qty > stock`. Nếu vượt quá tồn kho, wrapper sẽ tự động đặt `in_stock = False` để kích hoạt luồng từ chối sạch sẽ và loại bỏ toàn bộ dòng tiền thừa.

### 2. Tự động Retry khi tool gặp lỗi hệ thống ngẫu nhiên (`tool_error_rate: 0.18`)
* **Vấn đề**: Simulator giả lập lỗi tool ngẫu nhiên với tỷ lệ 18% (ví dụ trả về `'error': 'tool_failure'`). Wrapper trước đó coi đây là lỗi thật và từ chối nhầm các đơn hàng hợp lệ.
* **Giải pháp**: Trong vòng lặp retry của `wrapper.py`, tôi đã bổ sung logic duyệt trace. Nếu phát hiện tool gặp lỗi hệ thống khác lỗi nghiệp vụ thật (`'destination_not_served'`), wrapper sẽ bỏ qua kết quả lỗi và tự động gọi lại agent từ đầu.

### 3. Phòng chống Prompt Injection (Private Phase)
* **Vấn đề**: Các ghi chú đơn hàng của khách hàng chứa các lệnh ẩn cố gắng thay đổi giá hoặc ép buộc agent giao đến nơi không được hỗ trợ.
* **Giải pháp**:
  - Hàm `sanitize_input` trong [wrapper.py](solution/wrapper.py) tự động phát hiện các từ khóa `"GHI CHÚ"`, `"Note"` và cô lập nội dung của chúng thành dạng RAW DATA kèm cảnh báo LLM không được tuân theo các chỉ thị bên trong.
  - System prompt trong [prompt.txt](solution/prompt.txt) được thiết kế chi tiết để khẳng định giá cả chỉ lấy từ kết quả của tool `check_stock`, coi các chỉ thị trong ghi chú là dữ liệu thuần túy không được làm theo.

### 4. Chuẩn hóa định dạng đầu ra
* **Đơn hàng thành công**: Wrapper tự động dọn sạch mọi dòng tổng tiền ngẫu nhiên của LLM và tự động chèn duy nhất một dòng tổng tiền chuẩn ở cuối cùng dạng `Tong cong: <expected_total> VND`, đảm bảo khớp 100% định dạng của Scorer.
* **Đơn hàng bị từ chối**: Wrapper chủ động ghi đè bằng lời từ chối chuẩn hóa, lịch sự và loại bỏ hoàn toàn mọi thông tin số tiền (kể cả khi LLM in ra `Thành tiền:` hoặc `Tạm tính:`).

---

## IV. TRẠNG THÁI NỘP BÀI

Tôi đã thực hiện các lệnh Git để nộp bài chính thức:
* Thiết lập cấu hình git user: `Trần Minh Hoàng` (hoangtmhd2001@gmail.com).
* Add force các file kết quả và điểm số chính thức (`run_output.json`, `score.json`) lên repo do chúng bị `.gitignore` chặn.
* Thực hiện commit và push toàn bộ mã nguồn giải pháp lên repository GitHub thành công.
* Các tệp tin nộp bài bao gồm:
  - [config.json](solution/config.json)
  - [wrapper.py](solution/wrapper.py)
  - [prompt.txt](solution/prompt.txt)
  - [findings.json](solution/findings.json)
  - [run_output.json](run_output.json) (Kết quả Private)
  - [score.json](score.json) (Điểm số Private: 84.29)
