# BÁO CÁO THIẾT KẾ GIẢI PHÁP & THỰC THI TỔNG HỢP: KỸ THUẬT PROMPTING NÂNG CAO (LOGIC & CHUỖI)

**Học phần:** Kỹ nghệ Prompting Doanh nghiệp (Enterprise Prompt Engineering)  
**Chủ đề Session 04:** Kỹ thuật Prompting Nâng cao (Chain-of-Thought, Prompt Chaining & Xử lý đứt gãy ngữ cảnh)  
**Đơn vị áp dụng:** Tập đoàn Công nghệ & Logistics FastDelivery  

---

## 1. Bối cảnh & Mục tiêu Bài toán Thực tế

### 1.1. Bối cảnh nghiệp vụ
Tại Tập đoàn Logistics **FastDelivery**, bộ phận CSKH và Vận hành mỗi ngày phải tiếp nhận hàng nghìn yêu cầu khiếu nại, yêu cầu bồi thường đơn hàng giao chậm hoặc hư hỏng. Quá trình xử lý thủ công hoặc dùng các câu lệnh đơn lẻ (*Single Prompt*) gặp phải các vấn đề nghiêm trọng:
1. **Quá tải thông tin (Mega Prompt Failure):** Khi dồn tất cả yêu cầu (phân loại, tính toán toán học, tra cứu chính sách, soạn email) vào một prompt duy nhất, AI thường xuyên bỏ sót các biến số phụ phí hoặc tính toán sai lệch.
2. **Lỗi lan truyền dữ liệu (Cascading Errors):** Dữ liệu đầu vào từ khách hàng thường xuyên chứa các dị biệt (sai định dạng, số âm, thiếu mã vận đơn) làm hệ thống tính toán sai lệch hoặc bị crash.
3. **Đứt gãy phản hồi (Truncated Response):** Khi sinh các báo cáo phân tích chi tiết dài, AI thường xuyên bị dừng ngang chừng do chạm giới hạn *Max Output Tokens*, gây đứt quãng quy trình tự động hóa.

### 1.2. Mục tiêu giải pháp
Xây dựng một hệ thống tự động hóa hoàn chỉnh dựa trên kiến trúc **Prompt Chaining (Dây chuyền đa trạm)**, kết hợp kỹ thuật **Chain-of-Thought (CoT)** để lập luận logic, cơ chế **Validation Gates** chống dữ liệu dị biệt, và chiến lược **Continue Prompting** chống đứt gãy ngữ cảnh.

---

## 2. Thiết kế Kiến trúc Giải pháp (I/O & Pipeline)

### 2.1. Sơ đồ Luồng Thực thi Khép kín (Pipeline Architecture)
Hệ thống được chia thành 3 trạm độc lập kết nối thông qua cơ chế truyền dữ liệu cấu trúc sạch (**Data Handoff** dưới dạng JSON):

```text
[Raw Input từ Khách hàng]
         │
         ▼
┌────────────────────────────────────────────────────────┐
│ TRẠM 1: LÀM SẠCH DỮ LIỆU & BẮT LỖI DỊ BIỆT (Sanitizer)   │
│ - Trích xuất trường thông tin cốt lõi                 │
│ - Kiểm tra tính hợp lệ (Validation Gate: is_valid)     │
└────────────────────────────────────────────────────────┘
         │
         ▼ (Data Handoff: Strict JSON)
┌────────────────────────────────────────────────────────┐
│ TRẠM 2: LẬP LUẬN CHUỖI TƯ DUY COT & TÍNH TOÁN (Auditor) │
│ - Kích hoạt CoT (Let's think step by step)             │
│ - Áp dụng chính sách đền bù tài chính đa bước          │
└────────────────────────────────────────────────────────┘
         │
         ▼ (Data Handoff: Structured Payout JSON)
┌────────────────────────────────────────────────────────┐
│ TRẠM 3: XUẤT BÁO CÁO & CHỐNG ĐỨT GÃY NGỮ CẢNH (Reporter)│
│ - Chuyển đổi thành Bảng Markdown / Email chuẩn         │
│ - Tích hợp cơ chế Continue Prompting phòng thủ ngắt    │
└────────────────────────────────────────────────────────┘
```

---

## 3. Hệ thống Prompt Chi tiết cho Từng Trạm

### 3.1. Trạm 1: Làm sạch Dữ liệu & Bắt lỗi Dị biệt (Sanitizer)
* **Mục tiêu:** Nhận văn bản khiếu nại thô từ khách hàng, trích xuất các thông số kỹ thuật và tạo cờ kiểm định (`is_valid`) để loại bỏ dữ liệu rác/dị biệt, ngăn chặn hiệu ứng domino lỗi lan truyền.
* **Prompt Thực thi:**
```markdown
[System Role]: Bạn là Data Sanitizer chuyên nghiệp tại FastDelivery. Nhiệm vụ của bạn là đọc đoạn văn bản thô khiếu nại trong thẻ <data> và trích xuất thành chuỗi JSON hợp lệ.

[Quy tắc xử lý dị biệt]:
1. Trích xuất các trường: "tracking_id", "delay_days" (số nguyên), "is_damaged" (boolean: true/false).
2. Kiểm tra tính hợp lệ: Nếu thiếu mã vận đơn, định dạng hỏng nặng, hoặc "delay_days" có giá trị âm (< 0), hãy thiết lập "is_valid": false và ghi rõ lý do vào trường "error_reason". Ngược lại, thiết lập "is_valid": true.
3. Tuyệt đối KHÔNG đưa ra lời chào dạo đầu, KHÔNG giải thích lòng vòng. Chỉ xuất duy nhất một khối mã JSON chuẩn trong ```json ... ```.

<data>
{RAW_CUSTOMER_INPUT}
</data>
```

---

### 3.2. Trạm 2: Lập luận CoT & Tính toán Đền bù (Auditor)
* **Mục tiêu:** Nhận tệp JSON sạch từ Trạm 1. Nếu đơn hàng không hợp lệ (`is_valid: false`), hệ thống sẽ bỏ qua an toàn. Nếu hợp lệ, kích hoạt chuỗi tư duy **Chain-of-Thought** để tính toán mức bồi thường minh bạch theo chính sách của công ty.
* **Prompt Thực thi:**
```markdown
[System Role]: Bạn là Senior Compensation Auditor tại FastDelivery. Hãy nhận dữ liệu JSON đầu ra từ trạm trước và tiến hành tính toán mức đền bù tài chính.

[Chính sách đền bù FastDelivery]:
- Phí trễ hạn cơ bản: 10 USD/ngày cho mỗi ngày trễ (chỉ tính khi delay_days > 2).
- Thưởng hư hỏng: Cộng thêm 50 USD nếu is_damaged == true.

[Yêu cầu thực thi]:
1. Kiểm tra cờ "is_valid":
   - Nếu "is_valid" == false, hãy xuất ngay JSON: {"status": "SKIPPED", "payout": 0, "reason": [error_reason]}
   - Nếu "is_valid" == true, hãy áp dụng kỹ thuật Chain-of-Thought (Let's think step by step):
     * Bước 1: Tính tiền trễ hạn.
     * Bước 2: Kiểm tra khoản phụ phí hư hỏng.
     * Bước 3: Tổng hợp toàn bộ thành số tiền bồi thường cuối cùng ("payout").
2. Đưa toàn bộ các bước lập luận vào mảng "cot_steps" và kết quả cuối cùng vào trường "payout".
3. Chỉ xuất duy nhất cấu trúc JSON sạch, không kèm lời thừa.
```

---

### 3.3. Trạm 3: Xuất Báo cáo & Chống Đứt gãy Ngữ Cảnh (Reporter)
* **Mục tiêu:** Tổng hợp kết quả từ Trạm 2 để tạo ra Bảng báo cáo quản trị dạng Markdown sạch sẽ, sẵn sàng tích hợp vào hệ thống ERP hoặc gửi cho Ban Giám đốc. Đồng thời tích hợp cơ chế phòng thủ đứt gãy (`Truncated Response Guard`).
* **Prompt Thực thi:**
```markdown
[System Role]: Bạn là Operations Reporter tại FastDelivery. Nhiệm vụ của bạn là nhận tệp JSON kết quả tính toán đền bù và tổng hợp thành một Bảng Markdown chuyên nghiệp.

[Quy tắc định dạng]:
1. Trình bày dưới dạng Bảng Markdown gồm các cột: | Mã Vận Đơn | Trạng Thái | Chi Tiết Lập Luận (CoT) | Mức Đền Bù ($) |
2. Không viết lời chào đầu, không viết kết luận rườm rà.
3. [Phòng thủ đứt gãy / Truncated Guard]: Nếu phản hồi bị ngắt quãng giữa chừng do chạm giới hạn Max Output Tokens, hệ thống sẽ tự động gọi lệnh: "Hãy tiếp tục viết ngay sau cụm từ: [3-5 từ cuối cùng]". Hãy đảm bảo tuân thủ nghiêm ngặt quy tắc nối tiếp khi nhận được lệnh mớm câu.
```

---

## 4. Kết quả Chạy thử Thực tế (Mô phỏng Hệ thống)

### 4.1. Dữ liệu Đầu vào (Raw Test Cases)
Hệ thống tiến hành kiểm thử đồng thời với 2 trường hợp: đơn hàng chuẩn và đơn hàng dị biệt (bị lỗi dữ liệu đầu vào).
- **Đơn hàng 1 (Chuẩn):** `"Mã đơn FD-889, giao trễ 4 ngày, hàng hóa bị móp méo hư hỏng."`
- **Đơn hàng 2 (Dị biệt / Lỗi):** `"Đơn giao chậm -5 ngày, mã đơn không xác định ###XYZ"`

### 4.2. Luồng Xử lý qua Pipeline & Kết quả Cuối cùng

1. **Trạm 1 (Sanitizer Output - JSON):**
```json
[
  {
    "tracking_id": "FD-889",
    "delay_days": 4,
    "is_damaged": true,
    "is_valid": true,
    "error_reason": null
  },
  {
    "tracking_id": "UNKNOWN",
    "delay_days": -5,
    "is_damaged": false,
    "is_valid": false,
    "error_reason": "Invalid delay_days (negative value) and corrupted tracking ID"
  }
]
```

2. **Trạm 2 (Auditor Output - CoT & Payout JSON):**
Hệ thống áp dụng CoT lập luận cho đơn hợp lệ (`4 ngày trễ * $10 = $40`, cộng thêm `$50` do hư hỏng $\rightarrow$ Tổng `$90`), và bỏ qua an toàn đơn dị biệt nhờ Validation Gate.

3. **Trạm 3 (Reporter Output - Bảng Markdown Cuối cùng):**

| Mã Vận Đơn | Trạng Thái | Chi Tiết Lập Luận (CoT Summary) | Mức Đền Bù ($) |
| :--- | :--- | :--- | :---: |
| **FD-889** | `APPROVED` | 1. Trễ 4 ngày: $4 <math><mo>&gt;</mo></math> 2$ $\rightarrow$ $4 \times \$10 = \$40$.<br>2. Hàng hư hỏng: +$50$.<br>3. Tổng bồi thường: $\$40 + \$50 = \$90$. | **$90** |
| **UNKNOWN** | `SKIPPED` | Bỏ qua tự động do lỗi kiểm định: `Invalid delay_days (negative value) and corrupted tracking ID`. | **$0** |

---

## 5. Đánh giá Ưu - Nhược điểm & Bài học Kinh nghiệm

* **Ưu điểm vượt trội của giải pháp:**
  - **Khả năng chống crash tuyệt đối:** Cờ kiểm định `is_valid` tại Trạm 1 hoạt động như một *Validation Gate* vững chắc, triệt tiêu hoàn toàn hiện tượng lỗi lan truyền (*Cascading Errors*).
  - **Tối ưu hóa chi phí & Độ chính xác:** Áp dụng *Model Routing* (dùng mô hình nhỏ/rẻ cho Trạm 1 và mô hình cao cấp cho Trạm 2, 3) giúp tiết kiệm đến 70% chi phí API.
  - **Minh bạch lộ trình:** Kỹ thuật *Chain-of-Thought* giúp bóc tách từng bước tính toán, hỗ trợ nhân viên kiểm toán dễ dàng kiểm tra đối soát khi cần thiết.
* **Thách thức kỹ thuật:**
  - Đòi hỏi hệ thống điều phối bên ngoài (như script Python/Node.js) để quản lý luồng dữ liệu trung gian giữa các trạm (Data Handoff).