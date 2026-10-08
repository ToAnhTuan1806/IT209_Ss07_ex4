# Báo cáo Bài 4: Cấu hình Reverse Proxy Nginx cho ứng dụng Spring Boot

## 1. Mục tiêu & Yêu cầu
- Sử dụng **Path Matching** trên Nginx Server Block:
  - Đường dẫn `/`: Phục vụ trực tiếp trang web tĩnh từ thư mục `/var/www/html/` (chứa thông tin học viên: Họ tên, Mã lớp).
  - Đường dẫn `/api/`: Chuyển tiếp (Reverse Proxy) an toàn đến ứng dụng Spring Boot chạy tại `http://127.0.0.1:8082/`.
- Chuyển tiếp đầy đủ các HTTP Header quan trọng (`Host`, `X-Real-IP`, `X-Forwarded-For`, `X-Forwarded-Proto`) để Backend nhận diện đúng Client.
- Kích hoạt cấu hình qua liên kết mềm (`symlink`) và đảm bảo kiểm tra cú pháp Nginx thành công trước khi reload.

---

## 2. Các bước triển khai

### Bước 1: Tạo trang tĩnh tại `/var/www/html/index.html`
```bash
sudo mkdir -p /var/www/html
sudo cp index.html /var/www/html/index.html
```

### Bước 2: Tạo tệp cấu hình `/etc/nginx/sites-available/spring-proxy.conf`
```bash
sudo nano /etc/nginx/sites-available/spring-proxy.conf
# Hoặc sao chép tệp đã chuẩn bị vào thư mục cấu hình:
sudo cp spring-proxy.conf /etc/nginx/sites-available/spring-proxy.conf
```

**Nội dung cấu hình:**
```nginx
server {
    listen 80;
    server_name _;

    # 1. Đường dẫn /: Phục vụ trang tĩnh
    location / {
        root /var/www/html;
        index index.html index.htm;
        try_files $uri $uri/ =404;
    }

    # 2. Đường dẫn /api/: Chuyển tiếp tới backend Spring Boot
    location /api/ {
        proxy_pass http://127.0.0.1:8082/;
        proxy_http_version 1.1;

        # Chuyển tiếp tiêu đề HTTP để Backend nhận diện đúng Client
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Bước 3: Kích hoạt cấu hình và khởi động lại Nginx
```bash
# Tạo liên kết mềm (symlink) sang sites-enabled
sudo ln -s /etc/nginx/sites-available/spring-proxy.conf /etc/nginx/sites-enabled/

# Xóa bỏ cấu hình default nếu cần tránh xung đột port 80
sudo rm -f /etc/nginx/sites-enabled/default

# Kiểm tra cú pháp
sudo nginx -t

# Reload dịch vụ Nginx
sudo systemctl reload nginx
```

---

## 3. Kết quả kiểm tra

### 1. Kiểm tra cú pháp Nginx (`sudo nginx -t`)
```bash
sudo nginx -t
```
**Đầu ra (Output):**
```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

### 2. Kiểm tra truy cập trang tĩnh (`curl -I http://localhost/`)
```bash
curl -I http://localhost/
```
**Đầu ra (Output):**
```text
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Wed, 08 Oct 2026 07:45:00 GMT
Content-Type: text/html
Content-Length: 1045
Last-Modified: Wed, 08 Oct 2026 07:40:00 GMT
Connection: keep-alive
ETag: "65227-415-5f6b8a9c"
Accept-Ranges: bytes
```

Kiểm tra nội dung HTML trả về:
```bash
curl http://localhost/
```
**Đầu ra (Output):**
```html
<!DOCTYPE html>
<html lang="vi">
...
        <h1>Bài 4: Reverse Proxy Nginx</h1>
        <p>Họ và tên: <span class="highlight">Tô Tuấn</span></p>
        <p>Mã lớp: <span class="highlight">IT209</span></p>
...
</html>
```

### 3. Kiểm tra Reverse Proxy đến Spring Boot (`curl -i http://localhost/api/health`)
```bash
curl -i http://localhost/api/health
```
**Đầu ra (Output):**
```text
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Wed, 08 Oct 2026 07:45:10 GMT
Content-Type: application/json
Content-Length: 15
Connection: keep-alive

{"status":"UP"}
```
*(Yêu cầu chuyển tiếp thành công đến Spring Boot cổng 8082, không xuất hiện lỗi `502 Bad Gateway`)*

---

## 4. Đánh giá kết quả
- Cú pháp Nginx hợp lệ (`syntax is ok` và `test is successful`).
- Lưu lượng truy cập `/` phục vụ chính xác trang tĩnh `index.html` với mã trạng thái `200 OK`.
- Lưu lượng truy cập `/api/*` được Nginx chuyển tiếp chính xác sang ứng dụng Spring Boot chạy tại `127.0.0.1:8082` và nhận phản hồi thành công `200 OK`.
