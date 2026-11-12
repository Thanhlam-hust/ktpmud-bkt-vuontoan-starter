# 🌱 Vườn Toán Lớp 3 — Track B: "Ước lượng trước khi tính đúng"

> **Báo cáo Bài kiểm tra Hackathon (BKT-2 / Track B) môn Kỹ thuật Phần mềm Ứng dụng (KTPMUD)**  
> **Sinh viên thực hiện:** Bùi Thanh Lâm  
> **Phạm vi:** Mở rộng dạng bài luyện tập "Ước lượng kết quả trước khi tính đúng" trên nền tảng Vườn Toán Lớp 3, tuân thủ các quy định về dữ liệu synthetic và bảo mật thông tin (không PII).

---

## 1. Hướng dẫn chạy dự án (Node.js thuần)

Dự án được tối ưu hóa để chạy trực tiếp trên môi trường **Node.js (18+)**, bao gồm cơ chế tự động tương thích với SQLite tích hợp sẵn (`node:sqlite` trên Node.js 22.5+/24+) mà **không cần cài đặt Docker** hay các công cụ biên dịch C++ (Visual Studio Build Tools).

### Các bước khởi động:
```bash
# 1. Cài đặt thư viện (chỉ cần chạy lần đầu)
npm install

# 2. Khởi động server
npm start
```

- Mở trình duyệt và truy cập: **[http://localhost:3000](http://localhost:3000)**
- Lần chạy đầu tiên, server tự động tạo cơ sở dữ liệu `data/vuon-toan.db` và nạp 100 tài khoản mẫu cùng toàn bộ dữ liệu bài học từ `data/seed.json`.

### Tài khoản mẫu thử nghiệm (Synthetic):

| Vai trò | Tên đăng nhập | Mật khẩu | Ghi chú |
|---|---|---|---|
| Học sinh | `hocsinh01` … `hocsinh99` | `123456` | 99 tài khoản học sinh giả lập lớp 3A2 |
| Giáo viên | `giaovien01` | `123456` | Đăng nhập vào Góc Giáo Viên |

*Lệnh khôi phục CSDL về mặc định:*
```bash
npm run reset-db
```

---

## 2. Phân tích yêu cầu bài toán (Requirements Analysis — Track B)

### 2.1. Bối cảnh & Mục tiêu nghiệp vụ
Trong chương trình Toán lớp 3, kỹ năng "ước lượng kết quả" (như làm tròn số để ước tính tích hoặc thương trước khi tính toán chi tiết) đóng vai trò then chốt giúp học sinh hình thành tư duy định lượng và phát hiện sai sót tính toán thô.

**Yêu cầu cốt lõi:**
Bổ sung dạng bài luyện tập **"Ước lượng trước khi tính đúng"** áp dụng cho các phép nhân/chia đã có trong chương trình. Học sinh bắt buộc phải nhập một khoảng hoặc giá trị ước lượng hợp lý trước, sau đó mới thực hiện bước tính chính xác và nhận phản hồi phân biệt rõ ràng cho cả hai bước.

### 2.2. Ma trận đối chiếu tiêu chí nghiệm thu (Acceptance Criteria Traceability)

| Mã tiêu chí | Nội dung tiêu chí | Giải pháp đáp ứng trong thiết kế |
|---|---|---|
| **B-01** | Truy cập được từ luồng học hiện có | Tích hợp trực tiếp vào trang làm bài luyện tập `exercise.html` và `lesson.html`, không dùng trang demo tách rời. |
| **B-02** | Tối thiểu 12 câu hỏi/fixtures synthetic tái lập được | Xây dựng bộ đề gồm 12 câu hỏi phép nhân/chia lớp 3 với số liệu chuẩn trong `seed.json`. |
| **B-03** | Quy tắc tạo và chấm tất định, test được | Quy định khoảng ước lượng hợp lý theo quy tắc làm tròn chục/trăm rõ ràng, không dùng LLM. |
| **B-04** | API không trả đáp án đúng trước khi nộp | Endpoint `GET /api/question-sets/:id/questions` lọc bỏ hoàn toàn trường `correct_answer`. |
| **B-05** | Server chấm và lưu kết quả bền vững | Xử lý logic chấm tại `POST /api/submissions`, lưu vào bảng `attempts` và `progress`, không mất khi F5. |
| **B-06** | Feedback phân biệt ước lượng vs tính đúng | Trả về chi tiết: `estimateValid` (ước lượng hợp lý hay không) và `exactCorrect` (tính đúng hay sai). |
| **B-07** | Không sửa auth, session, tài khoản mẫu | Giữ nguyên nguyên vẹn `server/auth.js`, các bảng người dùng và cơ chế đăng nhập. |
| **B-08** | Test tự động normal, boundary, invalid | Bộ kiểm thử tự động kiểm tra các trường hợp cận khoảng, đầu vào không hợp lệ, chấm điểm. |
| **B-09** | Bổ sung lệnh `npm test` | Cấu hình script kiểm thử tự động chạy từ clean checkout. |
| **B-10** | CI chạy test trên Pull Request | File `.github/workflows/ci.yml` kiểm tra build và test tự động. |
| **B-11** | Không chứa PII/secret | 100% dữ liệu danh tính học sinh, giáo viên là synthetic (`hocsinh01`, `giaovien01`). |
| **B-12** | Hỗ trợ điều hướng bàn phím & thông báo lỗi rõ | Form nhập liệu hỗ trợ phím Tab/Enter, phản hồi lỗi dễ hiểu cho học sinh lớp 3. |

---

## 3. Quyết định thiết kế kiến trúc (Architecture & Design Decisions)

### 3.1. Quyết định mở rộng Schema dữ liệu (Data Model Decision)
* **Hiện trạng (`docs/architecture-reference.md`):** Bảng `questions` hiện có 4 loại (`multiple-choice`, `input`, `compare`, `paint`) với cơ chế so sánh chuỗi đẳng thức tuyệt đối (`given === correct_answer`).
* **Giải pháp lựa chọn:** Thêm `type = 'estimate'` vào trường `type` của bảng `questions`.
* **Cấu trúc trường `correct_answer`:** Thay vì tạo cột mới trong CSDL gây xáo trộn schema, định dạng chuỗi JSON được lưu trong cột `correct_answer`:
  ```json
  {
    "range": [40, 50],
    "exact": 42,
    "tolerance_rule": "round_ten"
  }
  ```
  *Lý do lựa chọn:* Đảm bảo tính tương thích ngược hoàn hảo với SQLite hiện tại, không yêu cầu migration phức tạp bảng `questions`.

### 3.2. Thuật toán chấm điểm tất định trên Server (Scoring Logic)
* **Quy tắc ước lượng:**
  - Đối với phép nhân $a \times b$ (ví dụ: $6 \times 7 = 42$): Học sinh ước lượng bằng cách làm tròn một thừa số về hàng chục gần nhất (ví dụ: $6 \times 10 = 60$ hoặc trong khoảng $[35, 50]$). Khoảng chấp nhận được `[min_est, max_est]` được định nghĩa sẵn và tất định.
* **Cơ chế tính điểm 2 giai đoạn (Tổng điểm $P$):**
  - **Điểm ước lượng ($40\% P$):** Nhận trọn điểm khi giá trị học sinh nhập nằm trong đoạn $[min, max]$.
  - **Điểm tính đúng ($60\% P$):** Nhận trọn điểm khi giá trị tính đúng bằng `exact`.
* **Định dạng dữ liệu nộp từ Client (`answers[q.id]`):**
  ```json
  {
    "estimate": 40,
    "exact": 42
  }
  ```

### 3.3. Luồng tương tác người dùng (User Experience Flow)
1. **Bước 1 (Ước lượng):** Học sinh đọc đề (ví dụ: "Ước lượng $29 \times 4$"). Học sinh nhập khoảng dự đoán (ví dụ: 100 - 130).
2. **Bước 2 (Tính toán chính xác):** Sau khi xác nhận ước lượng, ô nhập kết quả chính xác được kích hoạt (học sinh nhập 116).
3. **Bước 3 (Phản hồi sư phạm):** Giao diện phân tách hai lời nhận xét:
   - *Về ước lượng:* "Rất tốt! Ước lượng của em rất gần với thực tế."
   - *Về kết quả:* "Chính xác! $29 \times 4 = 116$."

---

## 4. Cấu trúc thư mục dự án

```text
vuon-toan-lop-3/
├─ public/                    # Giao diện tĩnh (HTML/CSS/JS thuần)
│  ├─ *.html                  # Giao diện học sinh (exercise, lesson, home, leaderboard...)
│  └─ assets/
│     ├─ css/app.css          # CSS nền tảng dùng chung
│     └─ js/
│        ├─ core/             # api.js, app.js
│        ├─ components/       # navbar.js, sidebar.js, footer.js
│        └─ pages/            # Logic các trang: exercise.js, login.js,...
├─ server/
│  ├─ index.js                # Express app entry point
│  ├─ db.js                   # Kết nối SQLite (hỗ trợ better-sqlite3 và node:sqlite)
│  ├─ auth.js                 # Session & xác thực (không sửa đổi theo B-07)
│  └─ api.js                  # API routes và logic chấm bài (/api/submissions)
├─ data/
│  ├─ seed.json               # Dữ liệu gốc: tài khoản, bài học, câu hỏi mẫu
│  └─ vuon-toan.db            # File SQLite CSDL (tự sinh khi khởi chạy)
├─ docs/
│  └─ architecture-reference.md # Đặc tả hệ thống chuẩn từ đề thi
├─ AI_USE.md                  # Nhật ký sử dụng AI (Mục 5.3)
├─ DEPLOY.md                  # Hướng dẫn triển khai
└─ package.json               # Cấu hình dự án và dependencies
```

---

## 5. API chính của hệ thống

| Method | Đường dẫn | Mô tả chức năng |
|---|---|---|
| POST | `/api/auth/login` · `/api/auth/logout` | Đăng nhập cookie httpOnly và đăng xuất |
| GET | `/api/auth/me` | Lấy thông tin phiên đăng nhập người dùng |
| GET | `/api/lessons`, `/api/me/progress` | Danh sách bài học và tiến độ đã lưu của học sinh |
| GET | `/api/question-sets/:id/questions` | Lấy danh sách câu hỏi (**bảo mật, không trả đáp án**) |
| POST | `/api/submissions` | Nộp bài làm; server chấm điểm tất định, lưu vào DB và cộng sao |
| GET | `/api/leaderboard` | Bảng xếp hạng học sinh tính từ điểm và sao thật trong CSDL |
| GET/POST | `/api/teacher/students` | Góc giáo viên: xem danh sách và cấp tài khoản |
