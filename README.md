# Chi tiết các bước triển khai Zabbix

#Bước 1: Cài đặt Docker

#Cập nhật index các gói phần mềm

sudo apt update && sudo apt upgrade -y

#Cài đặt các dependencies cần thiết

sudo apt install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release \
    apt-transport-https


#Tạo thư mục chứa keyrings nếu chưa có


sudo mkdir -p /etc/apt/keyrings


sudo chmod 0755 /etc/apt/keyrings


#Tải và lưu GPG key của Docker an toàn


curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg


sudo chmod a+r /etc/apt/keyrings/docker.gpg


#Thêm Docker repository vào APT sources


echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

#Cập nhật lại apt cache để nhận repo mới


sudo apt update && sudo apt upgrade -y



#Cài đặt các thành phần cốt lõi của Docker.


sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

#Cấu hình bảo mật Docker Daemon (Quan trọng). Mặc định Docker daemon khá "thoải mái". Nếu đang cài Docker mới hoàn toàn --> Thực hiện bước này. Chúng ta cần tạo file /etc/docker/daemon.json để siết chặt lại.


sudo mkdir -p /etc/docker


sudo nano /etc/docker/daemon.json

Dán nội dung dưới vào file daemon.json và lưu lại.

{
  "icc": false,
  "userns-remap": "default",
  "no-new-privileges": true,
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "3"
  },
  "live-restore": true,
  "userland-proxy": false
}


Giải thích các thông số bảo mật:

"icc": false: Ngăn chặn các container trong cùng default bridge network tự do nói chuyện với nhau (Inter-Container Communication). Phải link explicitly qua custom network.

"no-new-privileges": true: Ngăn chặn tiến trình trong container tự ý leo thang đặc quyền (ví dụ dùng su hay sudo).

"log-opts": Ngăn tình trạng log của container phình to làm tràn ổ cứng (chỉ giữ tối đa 3 file, mỗi file 50MB).

"live-restore": true: Nếu Docker daemon crash hoặc update, các container đang chạy sẽ KHÔNG bị chết theo.

"userland-proxy": false: Tắt proxy không cần thiết, giảm bề mặt tấn công. Sử dụng iptables thuần túy để route port.


#Khởi động lại và phân quyền (Cảnh báo)

sudo systemctl daemon-reload

sudo systemctl restart docker

#Cấu hình tự động khởi động docker và containerd cùng hệ điều hành (trong các trường hợp server bị tắt hoặc bị khởi động lại)

sudo systemctl enable docker

sudo systemctl enable containerd


#Kiểm tra hệ thống: Xác minh phiên bản và trạng thái hoạt động.Bash

#Kiểm tra version Docker Engine

sudo docker version

#Kiểm tra version Docker Compose Plugin

sudo docker compose version

#Chạy thử container an toàn để test

sudo docker run --rm hello-world



# Chi tiết các bước cài đặt Zabbix.

#Bước 1: Thiết lập timezone hệ thống (chuẩn giờ Việt Nam)

sudo timedatectl set-timezone Asia/Ho_Chi_Minh

#cài đặt chrony

sudo apt install -y chrony

#Cấu hình để chrony tự động khởi động cùng hệ điều hành

sudo systemctl enable chrony && sudo systemctl start chrony

#Bước 2: Xây dựng cấu trúc thư mục và bảo mật tham số

#Tạo thư mục gốc cho Zabbix

sudo mkdir -p /opt/zabbix

cd /opt/zabbix

#Tạo các thư mục persistent data


sudo mkdir -p ./zbx_env/usr/lib/zabbix/alertscripts

sudo mkdir -p ./zbx_env/usr/lib/zabbix/externalscripts

sudo mkdir -p ./zbx_env/var/lib/zabbix/snmptraps

#Phân quyền chặt chẽ (Tránh lỗi Permission Denied khi Docker mount)



sudo chown -R 1997:1997 ./zbx_env/usr/lib/zabbix


#Tạo file môi trường để ẩn credentials:

sudo nano .env

#Dán Nội dung file .env: 

#Database credentials

POSTGRES_USER=zabbix

POSTGRES_PASSWORD=MatKhauSieuKho123!@#

POSTGRES_DB=zabbix


#System settings

TIMEZONE=Asia/Ho_Chi_Minh


Sau đó lưu lại file bằng phím tắt Ctrl + O -> Nhấn enter --> Ctrl + X để thoát


#Bước 3: Triển khai file Docker Compose "Chống Đạn" (Bulletproof). Tạo file triển khai:

sudo nano docker-compose.yml

#Dán cấu hình kiến trúc sau vào file.

services:
  postgres-server:
    image: postgres:16-alpine
    container_name: zabbix-postgres
    restart: always
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
    volumes:
      # Named Volume: Giao phó cho Docker quản lý quyền (fix lỗi userns-remap)
      - zabbix-postgres-data:/var/lib/postgresql/data
    networks:
      - zabbix-net

  zabbix-server:
    image: zabbix/zabbix-server-pgsql:alpine-7.0-latest
    container_name: zabbix-server
    restart: always
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
      # Các thư mục này map dạng Read-Only (ro) nên không bị ảnh hưởng bởi lỗi phân quyền
      - ./zbx_env/usr/lib/zabbix/alertscripts:/usr/lib/zabbix/alertscripts:ro
      - ./zbx_env/usr/lib/zabbix/externalscripts:/usr/lib/zabbix/externalscripts:ro
      - ./zbx_env/var/lib/zabbix/snmptraps:/var/lib/zabbix/snmptraps:ro
    cap_add:
      - NET_RAW
      - NET_ADMIN
    depends_on:
      - postgres-server
    networks:
      - zabbix-net

  zabbix-web:
    image: zabbix/zabbix-web-nginx-pgsql:alpine-7.0-latest
    container_name: zabbix-web
    restart: always
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
      - zabbix-server
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

        
#Bước 4: Triển khai và Kiểm tra sự cố

cd /opt/zabbix

sudo docker compose up -d


<img width="1104" height="172" alt="image" src="https://github.com/user-attachments/assets/af05d78f-44b0-4a5a-98cd-fa9491a0a5cd" />


#Kiểm tra tiến trình Import Database (Bắt buộc). Lần đầu tiên chạy, Zabbix Server sẽ tự động bung cấu trúc bảng (schema) vào database PostgreSQL trắng. Quá trình này mất khoảng 30 giây đến 1 phút tùy tốc độ ổ cứng. Hãy soi log để chắc chắn không có lỗi SQL nào xảy ra: 

sudo docker logs -f zabbix-server

Bước 5: Truy cập và dọn dẹp. Mở trình duyệt truy cập: http://<IP_SERVER>

Tài khoản mặc định siêu quan trọng cần đổi ngay:


Username: Admin (Chữ A viết hoa)


Password: zabbix

#Cần lưu ý 3 điểm sau để hệ thống sống sót qua nhiều năm:


Vỡ file log Docker: Mặc định Docker lưu log không giới hạn. Trong cấu hình ở bài trước, tôi đã nhắc bạn set "log-opts": {"max-size": "50m"} trong daemon.json. Nếu chưa làm, hệ thống sớm muộn cũng sẽ Disk Full.


Housekeeping Database: Zabbix sinh ra một lượng dữ liệu Time-series khổng lồ. Hãy vào Administration -> General -> Housekeeping trên giao diện Web, cấu hình xóa dữ liệu History (Trend) cũ đi (ví dụ: giữ History 30 ngày, Trend 365 ngày). Nếu để mặc định, PostgreSQL sẽ phình to chiếm hết ổ cứng.


Lỗi Ping (ICMP) báo giả: Trong file compose trên, tôi đã thêm cap_add: - NET_RAW. Thiếu dòng này, Docker sẽ block gói tin ICMP, khiến Zabbix báo toàn bộ thiết bị mạng bị "DOWN" dù chúng vẫn sống. Bạn không cần lo lỗi này nữa.
