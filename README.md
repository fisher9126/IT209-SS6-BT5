# Bài 5: Thiết kế và Cấu hình dịch vụ Systemd Service cho ứng dụng Java

> Lưu ý: Systemd chỉ có trên Linux. Bài này thực hiện trên VPS Ubuntu Server (SSH vào), KHÔNG chạy trên macOS.

## 1. Mục tiêu
- Đóng gói ứng dụng (Spring Boot) thành dịch vụ Systemd để OS tự quản lý vòng đời.
- Chạy dưới tài khoản giới hạn `java-runner` (non-root) để tăng bảo mật.
- Tự động khởi động lại sau 10 giây khi dịch vụ sập bất thường (on-failure).

## 2. Tạo tài khoản hệ thống không có shell đăng nhập
```bash
sudo useradd -r -s /usr/sbin/nologin java-runner
```
- `-r`: tài khoản hệ thống (system account).
- `-s /usr/sbin/nologin`: không cho đăng nhập shell trực tiếp -> an toàn.

## 3. Tệp cấu hình dịch vụ `/etc/systemd/system/java-app.service`
```ini
[Unit]
Description=Java Application Service (Spring Boot mo phong)
After=network.target

[Service]
# Mo phong app Java bang mot tien trinh chay dai han.
# Khi dung JAR that, thay bang: ExecStart=/usr/bin/java -jar /opt/app/app.jar
ExecStart=/usr/bin/sleep 1000
User=java-runner
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Giải thích các block:
- **[Unit]**: mô tả dịch vụ; `After=network.target` -> chạy sau khi mạng sẵn sàng.
- **[Service]**:
  - `ExecStart`: lệnh khởi chạy (ở đây mô phỏng bằng `sleep 1000`; thực tế là `java -jar ...`).
  - `User=java-runner`: chạy dưới tài khoản giới hạn.
  - `Restart=on-failure` + `RestartSec=10`: tự restart sau 10 giây nếu sập bất thường.
- **[Install]**: `WantedBy=multi-user.target` -> kích hoạt khi boot vào chế độ nhiều người dùng.

## 4. Nạp cấu hình và khởi chạy dịch vụ
```bash
sudo systemctl daemon-reload
sudo systemctl start java-app.service
sudo systemctl status java-app.service
```
(Tuỳ chọn - bật tự chạy khi boot:)
```bash
sudo systemctl enable java-app.service
```

## 5. Kiểm tra log
```bash
journalctl -u java-app.service
```

## 6. Kết quả thực tế

### systemctl status java-app.service
```
● java-app.service - Java Application Service (Spring Boot mo phong)
     Loaded: loaded (/etc/systemd/system/java-app.service; disabled; vendor preset: enabled)
     Active: active (running) since Tue 2026-10-06 08:11:57 +07; 64ms ago
   Main PID: 71967 (sleep)
      Tasks: 1 (limit: 2216)
     Memory: 432.0K
        CPU: 2ms
     CGroup: /system.slice/java-app.service
             └─71967 /usr/bin/sleep 1000

Oct 06 08:11:57 fisher-9126 systemd[1]: Started Java Application Service (Spring Boot mo phong).
```

### journalctl -u java-app.service --no-pager
```
Oct 06 08:11:57 fisher-9126 systemd[1]: Started Java Application Service (Spring Boot mo phong).
```

Nhận xét: dịch vụ ở trạng thái `active (running)`, Main PID 71967 chạy tiến trình mô phỏng,
được quản lý bởi systemd dưới cấu hình User=java-runner.

## 7. Kiểm tra chính sách tự restart (tuỳ chọn)
Giết tiến trình để xem systemd tự dựng lại sau 10 giây:
```bash
sudo systemctl kill -s SIGKILL java-app.service
sleep 12
sudo systemctl status java-app.service
```
-> Dịch vụ phải tự `active (running)` trở lại nhờ `Restart=on-failure`.
