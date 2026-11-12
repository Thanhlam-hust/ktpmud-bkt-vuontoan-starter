# Hướng dẫn Triển khai Vườn Toán Lớp 3 (Node.js thuần)

> Dự án: Vườn Toán Lớp 3  
> Sinh viên thực hiện: Bùi Thanh Lâm  
> Môi trường: Node.js 18+ (hỗ trợ trực tiếp Node 20, 22, 24 với SQLite tích hợp)

---

## 1. Lưu ý quan trọng trước khi đưa lên môi trường mạng

1. **Biến môi trường `SESSION_SECRET`**:
   - Cần thiết lập biến môi trường `SESSION_SECRET` bằng một chuỗi ngẫu nhiên dài để mã hóa và ký cookie phiên đăng nhập an toàn.
2. **Cơ sở dữ liệu bền vững (Persistent Storage)**:
   - File CSDL SQLite được lưu mặc định tại `data/vuon-toan.db`.
   - Nếu triển khai trên các nền tảng đám mây (Render, Railway, VPS), bạn có thể cấu hình biến môi trường `DB_PATH` trỏ tới đường dẫn ổ đĩa gắn ngoài (Persistent Disk / Volume) để đảm bảo điểm số và tài khoản không bị xóa khi dịch vụ khởi động lại:
     ```bash
     DB_PATH=/var/data/vuon-toan.db
     ```
3. **Bảo mật tài khoản**:
   - Đổi mật khẩu tài khoản mẫu `giaovien01` và các tài khoản học sinh sau khi thiết lập xong môi trường production.

---

## 2. Triển khai trên máy chủ Linux / VPS (Node.js thuần)

1. **Cài đặt Node.js**:
   Cài đặt Node.js 18 hoặc mới hơn trên máy chủ Ubuntu/Debian:
   ```bash
   curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
   sudo apt-get install -y nodejs
   ```

2. **Tải mã nguồn và cài đặt dependencies**:
   ```bash
   git clone <url-repo-cua-ban>
   cd ktpmud-bkt-vuontoan-starter
   npm install
   ```

3. **Chạy dịch vụ với PM2 (Quản lý tiến trình nền tự khởi động lại)**:
   ```bash
   sudo npm install -g pm2
   SESSION_SECRET="chuoi-bi-mat-rat-dai-va-an-toan" pm2 start server/index.js --name "vuon-toan"
   pm2 startup
   pm2 save
   ```

4. **Cấu hình Nginx làm Reverse Proxy và cấp HTTPS (khuyên dùng)**:
   - Trỏ port 80/443 của Nginx về `http://localhost:3000`.
   - Sử dụng Certbot để tự động cấp chứng chỉ SSL miễn phí từ Let's Encrypt.

---

## 3. Triển khai trên nền tảng Cloud (Render / Railway)

1. Kết nối kho lưu trữ GitHub của bạn với Render hoặc Railway.
2. Thiết lập cấu hình dịch vụ dạng **Web Service**:
   - **Build Command:** `npm install`
   - **Start Command:** `npm start`
3. Thêm các biến môi trường trong phần **Environment Variables**:
   - `NODE_ENV`: `production`
   - `SESSION_SECRET`: `<chuỗi-ngẫu-nhiên-bảo-mật>`
4. Gắn Persistent Disk và cấu hình đường dẫn qua biến `DB_PATH`.
5. Kiểm tra trạng thái hoạt động của server qua endpoint:
   `https://<domain-cua-ban>/api/health` trả về `{"ok":true}` là hoàn tất.
