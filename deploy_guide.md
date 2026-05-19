# Hướng dẫn triển khai Vaultwarden bằng Docker & Docker Compose

Tài liệu này hướng dẫn cách triển khai Vaultwarden (giải pháp thay thế nhẹ cho Bitwarden server) bằng Docker và Docker Compose, đặc biệt tối ưu cho việc đưa lên môi trường public như **AWS EC2** và cấu hình **HTTPS bảo mật**.

---

## PHẦN 1: CẤU HÌNH TRÊN AWS EC2 (FLOATING IP & FIREWALL)

Để người dùng có thể truy cập Vaultwarden từ Internet thông qua IP tĩnh (AWS gọi là **Elastic IP**) hoặc Domain của bạn, bạn cần thực hiện 2 bước cấu hình trên AWS Management Console:

### 1. Gán Elastic IP (Floating IP) cho EC2
Mặc định, khi bạn restart EC2, IP Public của nó có thể bị thay đổi. Chúng ta cần gán một Elastic IP cố định:
1. Truy cập vào **AWS Console** -> **EC2 Dashboard**.
2. Ở thanh menu bên trái, tìm mục **Network & Security** -> Chọn **Elastic IPs**.
3. Click **Allocate Elastic IP address** -> bấm **Allocate** để nhận 1 IP tĩnh.
4. Chọn IP vừa tạo -> Click **Actions** -> Chọn **Associate Elastic IP address**.
5. Chọn **Instance** (máy ảo EC2 của bạn) và click **Associate**.
> *Bây giờ EC2 của bạn đã có một IP Public cố định không bị thay đổi khi restart.*

### 2. Mở cổng trên Security Group (AWS Firewall)
Bạn cần mở các cổng cần thiết để cho phép truy cập từ bên ngoài:
1. Tại giao diện **EC2 Instances**, chọn máy ảo của bạn -> chọn tab **Security** phía dưới -> Click vào **Security Group** đang áp dụng cho instance.
2. Bấm **Edit inbound rules** (Chỉnh sửa quy tắc chiều vào).
3. Thêm các quy tắc sau:
   - **HTTP**: Port `80` | Source: `0.0.0.0/0` (Cho phép Caddy xác thực SSL tự động).
   - **HTTPS**: Port `443` | Source: `0.0.0.0/0` (Truy cập an toàn mã hóa).
   - *(Tùy chọn)* **Custom TCP**: Port `8080` | Source: `0.0.0.0/0` (Chỉ mở nếu bạn muốn truy cập trực tiếp qua HTTP không mã hóa - **không khuyến khích**).
4. Bấm **Save rules**.

---

## PHẦN 2: TẠI SAO PHẢI CÓ HTTPS?
> [!IMPORTANT]
> Bitwarden Extension trên trình duyệt và các App Mobile **bắt buộc yêu cầu HTTPS (SSL)** để có thể hoạt động (đăng ký, đăng nhập, đồng bộ). 
> Nếu bạn truy cập qua HTTP thông thường (`http://<IP>:8080`), các chức năng bảo mật WebCrypto của trình duyệt sẽ bị khóa và bạn **không thể đăng nhập** được.

Để giải quyết vấn đề này đơn giản nhất, tài liệu này hướng dẫn bạn sử dụng **Caddy Server** làm Reverse Proxy để tự động lấy và gia hạn chứng chỉ SSL (HTTPS) Let's Encrypt miễn phí.

---

## PHẦN 3: CÁC BƯỚC TRIỂN KHAI TRÊN EC2

### Bước 1: Trỏ tên miền (Domain) về Elastic IP
Nếu bạn có tên miền riêng (VD: `vault.yourdomain.com`), hãy truy cập vào trang quản lý DNS của tên miền đó và thêm 1 bản ghi:
- **Type**: `A`
- **Name**: `vault` (hoặc tên miền phụ bạn muốn dùng)
- **Value**: Điền **Elastic IP** của AWS EC2.

---

### Bước 2: Cập nhật cấu hình File

Chúng ta sẽ sử dụng cấu hình Docker Compose tích hợp sẵn **Caddy** để tự động chạy HTTPS.

#### 1. File `docker-compose.yml`
Hãy cập nhật file `docker-compose.yml` của bạn như sau:

```yaml
version: '3.8'

services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: always
    environment:
      - WEBSOCKET_ENABLED=true
    volumes:
      - ./vw-data:/data

  caddy:
    image: caddy:2
    container_name: caddy
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - ./caddy-data:/data
      - ./caddy-config:/config
    depends_on:
      - vaultwarden
```

#### 2. Tạo file `Caddyfile` (Cấu hình Proxy & SSL tự động)
Tạo một file mới tên là `Caddyfile` nằm cùng cấp thư mục với `docker-compose.yml`:

```caddy
# Thay thế vault.yourdomain.com bằng tên miền thực tế của bạn đã trỏ về IP của EC2.
# Caddy sẽ tự động đăng ký SSL Let's Encrypt cho tên miền này.
vault.yourdomain.com {
    # Proxy các request thông thường tới Vaultwarden
    reverse_proxy vaultwarden:80
}
```

> *Lưu ý: Nếu chưa có tên miền và muốn test qua IP trước (chỉ chạy HTTP), bạn có thể thay thế dòng tên miền bằng `:80` trong Caddyfile.*

#### 3. Chuẩn bị file `.env`
Tạo file cấu hình môi trường:
```bash
cp .env_default_example .env
```
*(Bạn có thể cấu hình thêm các tính năng gửi Mail SMTP, tắt đăng ký tài khoản mới tự do trong file này).*

---

### Bước 3: Khởi chạy hệ thống

Chạy lệnh để kéo các docker image về và chạy ngầm (background):
```bash
docker-compose up -d
```

### Bước 4: Kiểm tra trạng thái
Kiểm tra xem cả 2 container `vaultwarden` và `caddy` đã chạy thành công hay chưa:
```bash
docker-compose ps
```

Nếu có lỗi liên quan đến việc cấp phát SSL hoặc kết nối, bạn hãy xem logs của Caddy:
```bash
docker-compose logs -f caddy
```

---

## PHẦN 4: HƯỚNG DẪN SAU KHI CÀI ĐẶT
1. Truy cập vào đường dẫn tên miền của bạn (Ví dụ: `https://vault.yourdomain.com`)
2. Đăng ký tài khoản quản trị đầu tiên của bạn.
3. **Quan trọng:** Sau khi bạn đã đăng ký xong các tài khoản cần thiết cho cá nhân/đội ngũ, hãy mở file `.env` sửa dòng `SIGNUPS_ALLOWED=false` để khóa chức năng đăng ký, tránh người lạ đăng ký tài khoản trên server của bạn. Sau đó khởi động lại bằng lệnh:
   ```bash
   docker-compose up -d
   ```
4. Tải Bitwarden Extension trên Chrome/Firefox hoặc App Mobile, chọn mục **Self-Hosted** (Tự lưu trữ) và nhập địa chỉ `https://vault.yourdomain.com` để kết nối và sử dụng.
