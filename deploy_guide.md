# Hướng dẫn triển khai Vaultwarden bằng Docker

Tài liệu này hướng dẫn cách triển khai Vaultwarden (một giải pháp thay thế nhẹ cho Bitwarden server) bằng Docker và Docker Compose.

## Yêu cầu hệ thống
- Đã cài đặt [Docker](https://docs.docker.com/get-docker/)
- Đã cài đặt [Docker Compose](https://docs.docker.com/compose/install/)

## Các bước triển khai

### Bước 1: Chuẩn bị file cấu hình môi trường
Copy file template `.env_default_example` thành file `.env` chính thức để Docker Compose có thể sử dụng:

```bash
cp .env_default_example .env
```

### Bước 2: Chỉnh sửa file `.env` (Tùy chọn)
Mở file `.env` bằng text editor (như `nano` hoặc `vim`) và điều chỉnh các biến môi trường nếu cần thiết:

- `PORT`: Cổng trên máy host mà Vaultwarden sẽ lắng nghe (mặc định là `8080`).
- `SIGNUPS_ALLOWED`: Đặt thành `true` nếu bạn muốn mở chức năng đăng ký tài khoản mới cho mọi người. (Mặc định đang là `false` để bảo mật).
- `ADMIN_TOKEN`: Khuyến khích tạo một chuỗi ngẫu nhiên an toàn để truy cập vào bảng quản trị của Vaultwarden tại `/admin`. Bạn có thể tạo token bằng lệnh `openssl rand -base64 48`.

### Bước 3: Khởi chạy dịch vụ
Trong cùng thư mục chứa file `docker-compose.yml`, chạy lệnh sau để kéo image về và khởi động container ngầm:

```bash
docker-compose up -d
```
*(Nếu bạn đang sử dụng phiên bản Docker mới, lệnh có thể là `docker compose up -d`)*

### Bước 4: Kiểm tra trạng thái
Kiểm tra xem container `vaultwarden` đã chạy thành công hay chưa:

```bash
docker ps
```
Hoặc xem logs của container để đảm bảo không có lỗi:

```bash
docker-compose logs -f
```

## Các lưu ý quan trọng

1. **Bảo mật và HTTPS:**
   Bitwarden clients (bao gồm cả trình duyệt extension, ứng dụng mobile) **yêu cầu HTTPS** để hoạt động chính xác (ngoại trừ khi bạn truy cập qua `localhost`). Bạn nên đặt Vaultwarden phía sau một Reverse Proxy (như Nginx, Caddy, hoặc Traefik) và cấu hình SSL/TLS (VD: Let's Encrypt).

2. **Dữ liệu (Volumes):**
   Toàn bộ dữ liệu của Vaultwarden (bao gồm database SQLite, các file đính kèm, RSA keys...) sẽ được lưu trong thư mục `vw-data` nằm cùng cấp với file `docker-compose.yml`. **Tuyệt đối không xóa thư mục này** nếu không bạn sẽ mất toàn bộ mật khẩu. Hãy thực hiện backup thư mục này định kỳ.

3. **Cập nhật phiên bản:**
   Để cập nhật Vaultwarden lên phiên bản mới nhất, chạy các lệnh sau:
   ```bash
   docker-compose pull
   docker-compose up -d
   ```

4. **Trang quản trị (Admin Panel):**
   Nếu bạn đã cấu hình `ADMIN_TOKEN` trong file `.env`, bạn có thể truy cập trang quản trị qua đường dẫn:
   `http://<IP-của-bạn>:8080/admin`
