# Nhật ký Sử dụng AI (AI-Use Log)

> Dự án: Vườn Toán Lớp 3 — Track B: "Ước lượng trước khi tính đúng"  
> Thành viên thực hiện: **Bùi Thanh Lâm**  
> Tuân thủ Mục 5.3 — Quy định Chính sách AI-use log môn Kỹ thuật Phần mềm Ứng dụng (KTPMUD).

| Công cụ / Model | Mục đích | Phạm vi | Kiểm chứng | Quyết định của nhóm | Người chịu trách nhiệm |
|---|---|---|---|---|---|
| Gemini 3.8 Flash / Antigravity | Hỗ trợ giải thích và gỡ lỗi `npm install` trên Node.js v24 (xung đột bản build C++ `better-sqlite3` trên Windows) | `package.json`, `server/db.js` | Viết script test bộ adapter `DatabaseSync` từ thư viện `node:sqlite` tích hợp sẵn của Node 24; kiểm tra tạo bảng và nạp seed dữ liệu thành công | **Giữ lại**: Sử dụng bộ adapter tương thích ngược `node:sqlite` để dự án chạy mượt mà trên môi trường Node.js local mà không cần cài Visual Studio C++ Build Tools hay Docker | Bùi Thanh Lâm |
| Gemini 3.8 Flash / Antigravity | Phân tích đặc tả kiến trúc hiện có (`docs/architecture-reference.md`) và gợi ý thiết kế dạng bài "Ước lượng trước khi tính đúng" | `docs/architecture-reference.md`, `README.md` | Đối chiếu cấu trúc bảng `questions`, `question_sets`, luồng chấm chuỗi chính xác trong `/api/submissions` và 12 tiêu chí nghiệm thu B-01 đến B-12 | **Giữ và chỉnh sửa**: Đồng ý thiết kế thêm `type: 'estimate'`, mã hóa khoảng hợp lý `[min, max]` cùng kết quả chính xác `exact`, logic chấm điểm tất định không dùng LLM | Bùi Thanh Lâm |
| Gemini 3.8 Flash / Antigravity | Hỗ trợ chuẩn hóa tài liệu dự án, lược bỏ các phụ thuộc Docker phức tạp, chuyển hướng dẫn sang chạy Node.js thuần | `README.md`, `DEPLOY.md`, `AI_USE.md` | Rà soát toàn bộ tài liệu, kiểm tra tính nhất quán với mã nguồn, đảm bảo không chứa PII (chỉ dùng tên, không dùng MSSV/thông tin cá nhân) | **Chấp thuận**: Cập nhật tài liệu hướng dẫn và lưu vết nhật ký minh bạch theo đúng quy định đề bài | Bùi Thanh Lâm |

