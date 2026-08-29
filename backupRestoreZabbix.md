# Kịch Bản Disaster Recovery (Backup & Restore) Zabbix Server

Tài liệu này hướng dẫn cách sao lưu hệ thống Zabbix (chạy trên Docker Compose + PostgreSQL) và khôi phục nhanh chóng sang một máy chủ mới trong trường hợp máy chủ cũ gặp sự cố nghiêm trọng.

---

## 1. Cấu hình Backup Tự Động (Thực hiện trên Server đang chạy)

**Bước 1: Tạo thư mục lưu trữ backup và script**
```bash
sudo mkdir -p /backup/zabbix
sudo chown root:root /backup/zabbix
sudo chmod 700 /backup/zabbix
sudo nano /opt/zabbix_backup.sh
```
**Bước 2: Cấu hình nội dung Script Backup**
```bash
#!/bin/bash
BACKUP_DIR="/backup/zabbix"
DATE=$(date +"%Y%m%d_%H%M")
ZABBIX_DIR="/opt/zabbix"
DB_CONTAINER="zabbix-postgres" 
DB_USER="zabbix"               
DB_NAME="zabbix"               
KEEP_DAYS=7                    

echo "=== Bắt đầu Backup Zabbix ($DATE) ==="

# 1. Backup Database (Dùng pg_dump, không cần stop container)
docker exec -t $DB_CONTAINER pg_dump -U $DB_USER -F c$DB_NAME > $BACKUP_DIR/zabbix_db_$DATE.dump

# 2. Backup thư mục cấu hình (Bao gồm docker-compose.yml, .env, zbx_env)
tar -czvf $BACKUP_DIR/zabbix_config_$DATE.tar.gz -C /opt zabbix

# 3. Gom lại thành 1 file duy nhất để dễ quản lý
tar -czvf $BACKUP_DIR/ZABBIX_FULL_BACKUP_$DATE.tar.gz -C$BACKUP_DIR zabbix_db_$DATE.dump zabbix_config_$DATE.tar.gz

# 4. Xóa các file trung gian
rm -f $BACKUP_DIR/zabbix_db_$DATE.dump $BACKUP_DIR/zabbix_config_$DATE.tar.gz

# 5. Xóa các bản backup cũ hơn KEEP_DAYS
find $BACKUP_DIR -name "ZABBIX_FULL_BACKUP_*.tar.gz" -type f -mtime +$KEEP_DAYS -exec rm -f {} \;

echo "=== Backup Hoàn Tất ==="
```

**Bước 3: Phân quyền và đặt lịch chạy tự động**

```bash
sudo chmod +x /opt/zabbix_backup.sh
sudo crontab -e
```
# Thêm dòng sau vào file crontab:

```bash
0 2 * * * /opt/zabbix_backup.sh >> /var/log/zabbix_backup.log 2>&1
```
# Kịch Bản Disaster Recovery (Backup & Restore) Zabbix Server

#Bước 1: Chuẩn bị Server mới và cài đặt hệ điều hành Ubuntu Server mới.

#Cài đặt Docker, Docker Compose, cấu hình bảo mật Docker (daemon.json) và cài đặt Timezone hệ thống giống với máy chủ cũ. https://github.com/lethetuan/Zabbix/blob/main/zabbix_deployment_guide.md

#Bước 2: Phục hồi cấu hình Zabbix. Đưa file ZABBIX_FULL_BACKUP_xxxx.tar.gz (lấy từ NAS/Cloud) vào thư mục /tmp trên server mới.
```bash
cd /tmp
#Giải nén file tổng
tar -xzvf ZABBIX_FULL_BACKUP_xxxx.tar.gz

#Giải nén thư mục cấu hình về đúng vị trí cũ (/opt/zabbix)
sudo tar -xzvpf zabbix_config_xxxx.tar.gz -C /opt




```

#Lệnh này sẽ khôi phục lại toàn bộ file docker-compose.yml, file ẩn .env và các thư mục script tuỳ chỉnh với đúng phân quyền cũ.

#Bước 3: Bật riêng Database (CHƯA bật toàn bộ hệ thống) ⚠️ KHÔNG chạy lệnh docker compose up -d lúc này để tránh Zabbix Server tự tạo bảng trắng đè lên DB cũ.

```bash
cd /opt/zabbix
sudo docker compose up -d postgres-server
```
#Chờ khoảng 15-20 giây để container PostgreSQL khởi tạo xong, nạp dữ liệu xong xuôi, rồi mới được bật các dịch vụ còn lại..

#Bước 4: Đưa dữ liệu (Restore) vào Database. Dùng file .dump đã giải nén ở /tmp để khôi phục cấu trúc và dữ liệu:
```bash
cat /tmp/zabbix_db_xxxx.dump | sudo docker exec -i zabbix-postgres pg_restore -U zabbix -d zabbix --clean --if-exists
```
#Thời gian chạy tuỳ thuộc vào dung lượng database cũ.
#Bước 5: Chỉnh sửa IP và Khởi động phần còn lại, Nếu Server mới có địa chỉ IP LAN khác máy cũ, hãy cập nhật lại, (Sửa các dòng mapping ports thành IP mới, ví dụ: - "IP_MOI:10051:10051").
```bash
sudo nano /opt/zabbix/docker-compose.yml
```
#Khởi động toàn bộ hệ thống:
```bash
cd /opt/zabbix
sudo docker compose up -d
```
#Hệ thống của bạn lúc này đã được phục hồi hoàn chỉnh cùng với mọi cài đặt, host và lịch sử giám sát cũ!
