# Báo cáo Tiến độ Checkpoint 1 (PC-1) — Track B: "Ước lượng trước khi tính đúng"

> **Dự án:** Vườn Toán Lớp 3 (Bài kiểm tra Hackathon BKT-2 / Track B)  
> **Vai trò:** Nhóm phát triển (Track B)  
> **Nguyên tắc:** Tuân thủ quy định ẩn danh — Không ghi họ tên, không ghi MSSV, chỉ sử dụng vai trò / alias dự án.

---

## 1. Mục tiêu Checkpoint PC-1
Tập trung vào **Phân tích yêu cầu và Thiết kế giải pháp mở rộng** theo đúng phương châm đánh giá của môn học: *"Ưu tiên PHÂN TÍCH & THIẾT KẾ hơn sản phẩm MVP hoàn chỉnh"*. 

Checkpoint PC-1 giải quyết các nội dung then chốt:
1. Phân tích kiến trúc hiện tại từ tài liệu đặc tả chuẩn (`docs/architecture-reference.md`).
2. Xác định điểm cần mở rộng trong hệ thống sẵn có để bổ sung dạng bài "Ước lượng trước khi tính đúng" cho phép nhân/chia lớp 3.
3. Thiết kế kiến trúc giải pháp (mở rộng schema, định dạng dữ liệu, thuật toán chấm điểm tất định, luồng tương tác UX).
4. Thiết lập môi trường chạy local ổn định và chuẩn bị kế hoạch hoàn thiện các tiêu chí nghiệm thu (B-01 đến B-12).

---

## 2. Phân tích hiện trạng hệ thống (Hiện có)

Theo tài liệu kiến trúc chuẩn `docs/architecture-reference.md`:
* **Kiến trúc:** Node.js (Express) + SQLite (đồng bộ) + Frontend tĩnh (HTML/CSS/JS thuần, gọi API qua `fetch()`).
* **Hạn chế của mô hình hiện tại đối với bài toán ước lượng:**
  1. *Schema câu hỏi (`questions`):* Hiện chỉ hỗ trợ 4 loại (`multiple-choice`, `input`, `compare`, `paint`). Không có kiểu câu hỏi nào biểu diễn một khoảng giá trị (range/interval).
  2. *Cơ chế chấm điểm (`/api/submissions`):* Đang sử dụng so sánh chuỗi đẳng thức tuyệt đối:
     ```javascript
     const ok = given === String(q.correct_answer).trim().toLowerCase();
     ```
     Cơ chế này không thể đánh giá tính "hợp lý" của một giá trị ước lượng hay kiểm tra xem một số có nằm trong khoảng cho phép hay không.
  3. *Luồng giao diện (`exercise.js`):* Mỗi câu hỏi chỉ cho phép nhập một đáp án duy nhất, chưa hỗ trợ quy trình 2 bước: học sinh đưa ra ước lượng trước khi thực hiện phép tính chính xác.

---

## 3. Thiết kế giải pháp mở rộng (Design Decisions)

### 3.1. Thiết kế Schema & Dữ liệu câu hỏi (Data Modeling)
* **Thêm loại câu hỏi mới:** Mở rộng trường `type` với giá trị `'estimate'`.
* **Cấu trúc trường `correct_answer`:** Thay vì thêm cột mới vào bảng `questions` (gây rủi ro xáo trộn schema và migration), trường `correct_answer` của loại `estimate` sẽ lưu trữ một chuỗi JSON chuẩn hóa:
  ```json
  {
    "range": [40, 50],
    "exact": 42,
    "tolerance_rule": "round_ten"
  }
  ```
  - `range`: Khoảng giá trị ước lượng được coi là hợp lý `[min, max]`.
  - `exact`: Kết quả tính toán chính xác tuyệt đối.
  - `tolerance_rule`: Quy tắc xác định khoảng (ví dụ: làm tròn hàng chục gần nhất).

### 3.2. Thuật toán chấm điểm tất định trên Server (Scoring Engine)
* **Quy tắc ước lượng (Được mã hóa bằng code tất định, không dùng LLM):**
  - Đối với phép nhân $a \times b$ (hoặc phép chia $a : b$): Xác định khoảng ước lượng hợp lý bằng cách làm tròn một thừa số/số bị chia về hàng chục gần nhất để tính nhẩm nhanh.
  - Ví dụ: $29 \times 4$ $\rightarrow$ làm tròn $30 \times 4 = 120$, khoảng hợp lý là $[100, 130]$.
* **Cấu trúc dữ liệu bài làm gửi lên từ Client (`answers[q.id]`):**
  ```json
  {
    "estimate": 110,
    "exact": 116
  }
  ```
* **Cơ chế tính điểm 2 giai đoạn:**
  - **Tiêu chí ước lượng hợp lý (40% điểm câu hỏi):**
    $estimateValid = (min \le given.estimate \le max)$
  - **Tiêu chí tính toán chính xác (60% điểm câu hỏi):**
    $exactCorrect = (given.exact == exact)$
  - Điểm câu hỏi được tính tỷ lệ theo 2 phần, đảm bảo khuyến khích học sinh vừa có khả năng ước lượng vừa tính toán chính xác.
* **Bảo mật đề thi (B-04):** Endpoint `GET /api/question-sets/:id/questions` lọc bỏ hoàn toàn thuộc tính `correct_answer` trước khi trả về client.

### 3.3. Thiết kế luồng tương tác người dùng (UX & Giao diện)
* **Luồng 2 bước tuần tự:**
  1. *Bước 1 (Ước lượng):* Giao diện hiển thị đề bài và ô nhập "Ước lượng của em". Ô tính kết quả chính xác tạm thời bị khóa.
  2. *Bước 2 (Tính đúng):* Sau khi học sinh nhập ước lượng và bấm xác nhận (hoặc nhấn Tab/Enter), ô nhập kết quả chính xác được mở khóa để học sinh tính toán chi tiết.
* **Phản hồi sư phạm phân tách rõ ràng (B-06):**
  - Hiển thị nhận xét riêng cho phần ước lượng (ví dụ: *"Ước lượng rất hợp lý!"* hoặc *"Ước lượng hơi xa, hãy chú ý làm tròn số"*).
  - Hiển thị nhận xét riêng cho kết quả chính xác.
* **Khả năng tiếp cận bằng bàn phím (B-12):** Học sinh có thể dùng hoàn toàn phím Tab để di chuyển giữa các ô nhập liệu và phím Enter để nộp/chuyển bước.

---

## 4. Bảng đối chiếu tiêu chí nghiệm thu (Acceptance Criteria Status)

| ID | Tiêu chí nghiệm thu | Trạng thái PC-1 | Giải pháp kỹ thuật |
|---|---|---|---|
| **B-01** | Truy cập từ luồng học hiện có | Đã thiết kế | Tích hợp vào `exercise.html` và `lesson.html` |
| **B-02** | Tối thiểu 12 fixtures synthetic | Đã thiết kế | Soạn thảo 12 câu hỏi phép nhân/chia lớp 3 vào `seed.json` |
| **B-03** | Quy tắc chấm ước lượng tất định | Đã thiết kế | Thuật toán so sánh khoảng $[min, max]$ thuần code JS |
| **B-04** | API không trả đáp án đúng trước | Đã đối chiếu | Endpoint `/api/question-sets/:id/questions` giữ nguyên cơ chế bảo mật |
| **B-05** | Server chấm & lưu kết quả bền vững | Đã đối chiếu | Ghi nhận vào bảng `attempts` và `progress` SQLite |
| **B-06** | Feedback phân biệt ước lượng vs chính xác | Đã thiết kế | Trả về object chi tiết gồm 2 trạng thái đánh giá riêng |
| **B-07** | Không sửa auth/session/tài khoản mẫu | Tuân thủ 100% | Giữ nguyên vẹn `server/auth.js` |
| **B-08** | Test tự động normal/boundary/invalid | Đang chuẩn bị | Lập kế hoạch các ca test cho `test/estimate.test.js` |
| **B-09** | Bổ sung lệnh `npm test` | Đang chuẩn bị | Thêm test script vào `package.json` |
| **B-10** | CI chạy test trên Pull Request | Đang chuẩn bị | Thiết lập file `.github/workflows/ci.yml` |
| **B-11** | Không có PII/secret | Tuân thủ 100% | Toàn bộ dữ liệu tài khoản và báo cáo là synthetic/ẩn danh |
| **B-12** | Hỗ trợ bàn phím & thông báo lỗi rõ | Đã thiết kế | Thiết kế form nhập liệu 2 bước hỗ trợ phím Tab/Enter |

---

## 5. Kế hoạch hoàn thiện tiếp theo (Next Steps)
1. **Hoàn thiện Fixtures:** Cập nhật 12 câu hỏi `type: "estimate"` vào `data/seed.json`.
2. **Cập nhật Backend:** Bổ sung hàm chấm `type === 'estimate'` tại endpoint `POST /api/submissions` trong `server/api.js`.
3. **Cập nhật Frontend:** Bổ sung hàm render giao diện `type === 'estimate'` trong `public/assets/js/pages/exercise.js`.
4. **Viết Test & CI:** Bổ sung file test tự động `test/estimate.test.js`, thêm lệnh `npm test` và tạo file GitHub Actions workflow `.github/workflows/ci.yml`.

