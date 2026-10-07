# Báo Cáo Bài Tập 3: Thiết lập Cơ sở dữ liệu và Tự cấu hình dịch vụ Systemd cho Spring Boot

## 1. Thông Tin Chung
- **Học phần:** IT209 - Quản trị Hệ thống Linux & Dịch vụ Mạng
- **Bài tập:** Session 07 - Bài 3 (Thiết lập Cơ sở dữ liệu và Tự cấu hình dịch vụ Systemd cho Spring Boot)
- **Đường dẫn nộp bài:** `homework/session_07/ex3/` (hoặc thư mục `SS7-B3/`)

---

## 2. Mục Tiêu & Yêu Cầu

### Mục tiêu
- Khởi tạo và bảo mật cơ sở dữ liệu MySQL: tạo database, user chuyên biệt và phân quyền hạn tối thiểu.
- Tự thiết kế và cấu hình dịch vụ Systemd (`spring-app.service`) để quản lý vòng đời ứng dụng Spring Boot.
- Triển khai nguyên tắc bảo mật: chạy ứng dụng dưới quyền người dùng giới hạn không có quyền tương tác shell (`spring-runner`, non-login shell).

### Ràng buộc kỹ thuật
1. **MySQL Database:**
   - Database name: `springboot_db`
   - User: `spring-admin`
   - Password: `SpringSecure@123`
   - Toàn quyền (ALL PRIVILEGES) trên `springboot_db.*`.
2. **Linux User:**
   - User hệ thống: `spring-runner` (no-login shell: `/usr/sbin/nologin` hoặc `/bin/false`).
3. **Systemd Service (`/etc/systemd/system/spring-app.service`):**
   - User thực thi: `spring-runner`
   - Lệnh khởi chạy: `/usr/bin/java -jar /opt/spring-app/app.jar`
   - Cơ chế tự động khôi phục (Auto-restart): `Restart=on-failure`, `RestartSec=10` (khởi động lại sau 10 giây nếu bị lỗi).
   - Lắng nghe tại cổng: `8082`.

---

## 3. Các Bước Thực Hiện Chi Tiết

### Bước 1: Cấu hình Cơ sở dữ liệu MySQL
Đăng nhập vào MySQL với quyền root:
```bash
sudo mysql -u root -p
```

Thực thi các lệnh SQL tạo Database, User và phân quyền:
```sql
-- 1. Tạo database
CREATE DATABASE springboot_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- 2. Tạo user spring-admin với mật khẩu an toàn
CREATE USER 'spring-admin'@'localhost' IDENTIFIED BY 'SpringSecure@123';

-- 3. Cấp toàn quyền trên database springboot_db
GRANT ALL PRIVILEGES ON springboot_db.* TO 'spring-admin'@'localhost';

-- 4. Áp dụng thay đổi quyền hạn
FLUSH PRIVILEGES;

-- 5. Thoát MySQL
EXIT;
```

Kiểm tra đăng nhập bằng tài khoản mới tạo:
```bash
mysql -u spring-admin -p'SpringSecure@123' -e "SHOW DATABASES;"
```

---

### Bước 2: Tạo User hệ thống `spring-runner` và chuẩn bị thư mục ứng dụng

```bash
# 1. Tạo user hệ thống không có login shell và không cấp home directory
sudo useradd -r -s /usr/sbin/nologin spring-runner

# 2. Tạo thư mục chứa ứng dụng Spring Boot
sudo mkdir -p /opt/spring-app

# 3. Sao chép tệp app.jar vào thư mục /opt/spring-app/ (giả sử file app.jar đã có)
# sudo cp /path/to/app.jar /opt/spring-app/app.jar

# 4. Phân quyền sở hữu toàn bộ thư mục cho user spring-runner
sudo chown -R spring-runner:spring-runner /opt/spring-app
sudo chmod 750 /opt/spring-app
sudo chmod 640 /opt/spring-app/app.jar
```

---

### Bước 3: Tạo tệp cấu hình dịch vụ Systemd (`spring-app.service`)

Tạo file dịch vụ tại `/etc/systemd/system/spring-app.service`:
```bash
sudo nano /etc/systemd/system/spring-app.service
```

Nội dung tệp cấu hình:
```ini
[Unit]
Description=Spring Boot Application Service
After=network.target mysql.service

[Service]
Type=simple
User=spring-runner
Group=spring-runner
WorkingDirectory=/opt/spring-app
ExecStart=/usr/bin/java -jar /opt/spring-app/app.jar
Restart=on-failure
RestartSec=10
StandardOutput=journal
StandardError=journal
SuccessExitStatus=143

[Install]
WantedBy=multi-user.target
```

---

### Bước 4: Nạp lại cấu hình, Kích hoạt và Khởi chạy Dịch vụ

```bash
# 1. Nạp lại daemon của systemd để nhận diện service mới
sudo systemctl daemon-reload

# 2. Kích hoạt service tự động chạy cùng hệ điều hành
sudo systemctl enable spring-app.service

# 3. Khởi chạy service
sudo systemctl start spring-app.service
```

---

## 4. Kiểm Tra & Đánh Giá Kết Quả

### 1. Kiểm tra trạng thái dịch vụ:
```bash
sudo systemctl status spring-app.service
```
**Kết quả mong đợi:** Trạng thái `active (running)`, tiến trình được thực thi bởi user `spring-runner`.

```text
● spring-app.service - Spring Boot Application Service
     Loaded: loaded (/etc/systemd/system/spring-app.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2026-10-07 15:20:00 UTC; 10s ago
   Main PID: 12345 (java)
      Tasks: 28 (limit: 4684)
     Memory: 280.5M
        CPU: 4.120s
     CGroup: /system.slice/spring-app.service
             └─12345 /usr/bin/java -jar /opt/spring-app/app.jar
```

---

### 2. Kiểm tra cổng lắng nghe (Port 8082):
```bash
ss -tlnp | grep 8082
```
**Kết quả mong đợi:**
```text
LISTEN 0      100          0.0.0.0:8082      0.0.0.0:*    users:(("java",pid=12345,fd=14))
```

---

## 5. Hình Ảnh Minh Chứng (Screenshots)

### 5.1. Ảnh chụp cấu hình MySQL (Database & User)
![MySQL Configuration](./images/mysql_setup.png)

---

### 5.2. Ảnh chụp trạng thái dịch vụ `systemctl status spring-app.service`
![Systemd Status](./images/systemd_status.png)

---

### 5.3. Ảnh chụp kiểm tra cổng mạng `ss -tlnp | grep 8082`
![Port Check 8082](./images/port_check.png)

---

## 6. Tổng Kết Các Ưu Điểm Thiết Kế
1. **Bảo mật tiến trình (Process Security):** Ứng dụng chạy bằng tài khoản `spring-runner` không có shell login (`/usr/sbin/nologin`), nếu ứng dụng có lỗ hổng bảo mật (như RCE), kẻ tấn công cũng không thể trực tiếp đăng nhập terminal hoặc thao tác can thiệp tới file của hệ thống.
2. **Tính sẵn sàng cao (High Availability):** Chỉ thị `Restart=on-failure` cùng `RestartSec=10` giúp dịch vụ tự động khôi phục khi gặp crash mà không cần can thiệp thủ công từ quản trị viên.
3. **Quản lý tập trung qua Systemd:** Tự động ghi nhật ký chuẩn thông qua `journalctl -u spring-app.service` và quản lý khởi động cùng hệ thống (`multi-user.target`).
