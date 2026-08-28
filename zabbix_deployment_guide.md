# Chi tiết các bước triển khai Zabbix

## 1. Cài đặt Docker và các thành phần cốt lõi

### Bước 1: Cập nhật index các gói phần mềm
```bash
sudo apt update && sudo apt upgrade -y
```

### Bước 2: Cài đặt các dependencies cần thiết
```bash
sudo apt install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release \
    apt-transport-https
```

### Bước 3: Cấu hình repository của Docker
Tạo thư mục chứa keyrings nếu chưa có:
```bash
sudo mkdir -p /etc/apt/keyrings
sudo chmod 0755 /etc/apt/keyrings
```

Tải và lưu GPG key của Docker an toàn:
```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

Thêm Docker repository vào APT sources:
```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Bước 4: Cài đặt Docker
Cập nhật lại apt cache để nhận repo mới và tiến hành cài đặt:
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Bước 5: Cấu hình bảo mật Docker Daemon (Quan trọng)
Mặc định Docker daemon khá "thoải mái". Nếu đang cài Docker mới hoàn toàn, bạn cần thực hiện bước này để siết chặt lại.
```bash
sudo mkdir -p /etc/docker
sudo nano /etc/docker/daemon.json
```
Dán nội dung dưới vào file `daemon.json` và lưu lại:
```json
{
  "icc": false,
  "no-new-privileges": true,
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "3"
  },
  "live-restore": true,
  "userland-proxy": false
}
```
> **Giải thích các thông số bảo mật:**
> - `"icc": false`: Ngăn chặn các container trong cùng default bridge network tự do nói chuyện với nhau. Phải link explicitly qua custom network.
> - `"no-new-privileges": true`: Ngăn chặn tiến trình trong container tự ý leo thang đặc quyền (ví dụ dùng `su` hay `sudo`).
> - `"log-opts"`: Ngăn tình trạng log của container phình to làm tràn ổ cứng (chỉ giữ tối đa 3 file, mỗi file 50MB).
> - `"live-restore": true`: Cho phép container tiếp tục chạy khi Docker daemon tạm thời bị restart/mất kết nối, trong các điều kiện được Docker hỗ trợ
> - `"userland-proxy": false`: Tắt proxy không cần thiết, giảm bề mặt tấn công. Sử dụng iptables thuần túy để route port.

### Bước 6: Khởi động lại và phân quyền
```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```
Cấu hình tự động khởi động Docker và containerd cùng hệ điều hành:
```bash
sudo systemctl enable docker
sudo systemctl enable containerd
```

### Bước 7: Kiểm tra hệ thống
Xác minh phiên bản và trạng thái hoạt động:
```bash
# Kiểm tra version Docker Engine
sudo docker version

# Kiểm tra version Docker Compose Plugin
sudo docker compose version

# Chạy thử container an toàn để test
sudo docker run --rm hello-world
```

---

## 2. Chi tiết các bước cài đặt Zabbix

### Bước 1: Thiết lập timezone hệ thống (chuẩn giờ Việt Nam)
```bash
sudo timedatectl set-timezone Asia/Ho_Chi_Minh
```
Cài đặt và cấu hình Chrony để đồng bộ thời gian:
```bash
sudo apt install -y chrony
sudo systemctl enable chrony && sudo systemctl start chrony
```

### Bước 2: Xây dựng cấu trúc thư mục và bảo mật tham số
Tạo thư mục gốc cho Zabbix:
```bash
sudo mkdir -p /opt/zabbix
cd /opt/zabbix
```
Tạo các thư mục persistent data:
```bash
sudo mkdir -p ./zbx_env/usr/lib/zabbix/alertscripts
sudo mkdir -p ./zbx_env/usr/lib/zabbix/externalscripts
sudo mkdir -p ./zbx_env/var/lib/zabbix/snmptraps
```
Phân quyền chặt chẽ (Tránh lỗi *Permission Denied* khi Docker mount):
```bash
sudo chown -R 1997:1997 ./zbx_env/usr/lib/zabbix
```
Tạo file môi trường để ẩn credentials:
```bash
sudo nano .env
```
Dán Nội dung file `.env`:
```env
# Database credentials
POSTGRES_USER=zabbix
POSTGRES_PASSWORD=THAY_BANG_MAT_KHAU_RAT_MANH
POSTGRES_DB=zabbix

# System settings
TIMEZONE=Asia/Ho_Chi_Minh
```
*(Sau đó lưu lại file bằng phím tắt `Ctrl + O` -> Nhấn `Enter` -> `Ctrl + X` để thoát)*
#Sau khi thoát nano, mới chạy:
# File /opt/zabbix/.env chứa các thông tin cực kỳ nhạy cảm là tài khoản và mật khẩu Database → Nên chuyển quyền root và bảo mật bằng 2 dòng lệnh dưới:
```bash
sudo chmod 600 /opt/zabbix/.env
sudo chown root:root /opt/zabbix/.env
```

# Sau khi chạy 2 lệnh trên cần kiểm tra lại bằng lệnh dưới để đảm bảo file có dạng -rw------- 1 root root ...
```bash
ls -l /opt/zabbix/.env
```



### Bước 3: Triển khai file Docker Compose "Chống Đạn" (Bulletproof)
Tạo file triển khai:
```bash
sudo nano docker-compose.yml
```
Dán cấu hình kiến trúc sau vào file `docker-compose.yml`:
```yaml
services:
  postgres-server:
    image: postgres:16-alpine
    container_name: zabbix-postgres
    restart: unless-stopped
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
    volumes:
      - zabbix-postgres-data:/var/lib/postgresql/data
    networks:
      - zabbix-net
    # Healthcheck đảm bảo DB sẵn sàng trước khi Zabbix Server kết nối
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s

  zabbix-server:
    image: zabbix/zabbix-server-pgsql:alpine-7.0-latest
    container_name: zabbix-server
    restart: unless-stopped
    ports:
      - "10051:10051"
    environment:
      - DB_SERVER_HOST=postgres-server
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
      - ZBX_CACHESIZE=256M
      - ZBX_STARTPINGERS=10
      - ZBX_STARTSNMPPOLLERS=10
      - ZBX_TIMEOUT=15
    volumes:
      - ./zbx_env/usr/lib/zabbix/alertscripts:/usr/lib/zabbix/alertscripts:ro
      - ./zbx_env/usr/lib/zabbix/externalscripts:/usr/lib/zabbix/externalscripts:ro
    cap_add:
      - NET_RAW
    depends_on:
      postgres-server:
        condition: service_healthy # Chỉ chạy khi Postgres đã pass healthcheck
    networks:
      - zabbix-net

  zabbix-web:
    image: zabbix/zabbix-web-nginx-pgsql:alpine-7.0-latest
    container_name: zabbix-web
    restart: unless-stopped
    ports:
      - "80:8080"
      - "443:8443"
    environment:
      - ZBX_SERVER_HOST=zabbix-server
      - DB_SERVER_HOST=postgres-server
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
      - PHP_TZ=${TIMEZONE}
    depends_on:
      postgres-server:
        condition: service_healthy # Web cũng cần chờ DB sẵn sàng
      zabbix-server:
        condition: service_started
    networks:
      - zabbix-net

networks:
  zabbix-net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.30.0.0/16

volumes:
  zabbix-postgres-data:
```
#Kiểm tra image Zabbix 7.0 trên Docker Registry trước khi triển khai. Nếu pull thành công → OK. 
```bash
# 1. Kiểm tra image Zabbix Server
sudo docker pull zabbix/zabbix-server-pgsql:alpine-7.0-latest

# 2. Kiểm tra image Zabbix Web
sudo docker pull zabbix/zabbix-web-nginx-pgsql:alpine-7.0-latest

# 3. Kiểm tra image PostgreSQL
sudo docker pull postgres:16-alpine
```
### Bước 4: Triển khai và Kiểm tra sự cố
Chạy Docker Compose:
```bash
cd /opt/zabbix
sudo docker compose up -d
```
![image](https://github.com/user-attachments/assets/af05d78f-44b0-4a5a-98cd-fa9491a0a5cd)

**Kiểm tra tiến trình Import Database (Bắt buộc):**
Lần đầu tiên chạy, Zabbix Server sẽ tự động bung cấu trúc bảng (schema) vào database PostgreSQL trắng. Quá trình này mất khoảng 30 giây đến 1 phút tùy tốc độ ổ cứng. Hãy soi log để chắc chắn không có lỗi SQL nào xảy ra:
```bash
sudo docker logs -f zabbix-server
sudo docker logs --tail 100 zabbix-postgres
```
Kiểm tra trạng thái container:
```bash
sudo docker compose ps
sudo docker ps
```

### Bước 5: Truy cập và Cấu hình
Mở trình duyệt truy cập: `http://<IP_SERVER>`

Tài khoản mặc định siêu quan trọng cần đổi ngay:
- **Username:** `Admin` (Chữ A viết hoa)
- **Password:** `zabbix`

---

## 3. Các lưu ý để hệ thống sống sót qua nhiều năm

1. **Vỡ file log Docker:**
   Mặc định Docker lưu log không giới hạn. Trong cấu hình ở bài trước, bạn đã set `"log-opts": {"max-size": "50m"}` trong `daemon.json`. Nếu chưa làm, hệ thống sớm muộn cũng sẽ Disk Full.

2. **Housekeeping Database:**
   Zabbix sinh ra một lượng dữ liệu Time-series khổng lồ. Hãy vào **Administration -> General -> Housekeeping** trên giao diện Web, cấu hình xóa dữ liệu History (Trend) cũ đi (ví dụ: giữ History 30 ngày, Trend 365 ngày). Nếu để mặc định, PostgreSQL sẽ phình to chiếm hết ổ cứng.

3. **Lỗi Ping (ICMP) báo giả:**
   Trong file compose trên, cấu hình đã thêm `cap_add: - NET_RAW`. Trong một số môi trường Docker, tiện ích fping của Zabbix có thể không đủ quyền tạo raw socket ICMP và phát sinh lỗi Operation not permitted. Khi đó cần cấp capability NET_RAW.
