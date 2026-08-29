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

# 1. ĐỌC BIẾN TỪ FILE .ENV CỦA DỰ ÁN
if [ -f "$ZABBIX_DIR/.env" ]; then
    # Lệnh set -a giúp export tự động các biến trong .env ra môi trường
    set -a
    source "$ZABBIX_DIR/.env"
    set +a
else
    echo "LỖI: Không tìm thấy file $ZABBIX_DIR/.env. Dừng backup!"
    exit 1
fi

# 2. GÁN BIẾN (Lấy giá trị từ .env)
DB_CONTAINER="zabbix-postgres" # Tên container vẫn cố định theo docker-compose
DB_USER="$POSTGRES_USER"
DB_NAME="$POSTGRES_DB"
KEEP_DAYS=7

echo "=== Bắt đầu Backup Zabbix ($DATE) ==="
echo "Đang dùng Database: $DB_NAME | User: $DB_USER"

# 3. Backup Database
docker exec $DB_CONTAINER pg_dump -U $DB_USER --format=custom $DB_NAME > $BACKUP_DIR/zabbix_db_$DATE.dump

# 4. Backup thư mục cấu hình 
tar -czvf $BACKUP_DIR/zabbix_config_$DATE.tar.gz -C /opt zabbix

# 5. Gom lại thành 1 file duy nhất
tar -czvf $BACKUP_DIR/ZABBIX_FULL_BACKUP_$DATE.tar.gz -C $BACKUP_DIR zabbix_db_$DATE.dump zabbix_config_$DATE.tar.gz

# 6. Xóa các file trung gian
rm -f $BACKUP_DIR/zabbix_db_$DATE.dump $BACKUP_DIR/zabbix_config_$DATE.tar.gz

# 7. Xóa backup cũ
find $BACKUP_DIR -name "ZABBIX_FULL_BACKUP_*.tar.gz" -type f -mtime +$KEEP_DAYS -exec rm -f {} \;

echo "=== Backup Hoàn Tất ==="

echo "Đang copy sang thư mục Share trên Windows Server..."
cp $BACKUP_DIR/ZABBIX_FULL_BACKUP_$DATE.tar.gz /mnt/windows_backup/

# Tùy chọn: Tự động xóa các file backup CŨ HƠN 14 ngày trên Windows Server để đỡ đầy ổ
find /mnt/windows_backup -name "ZABBIX_FULL_BACKUP_*.tar.gz" -type f -mtime +14 -exec rm -f {} \;
```

**Bước 3: Phân quyền và đặt lịch chạy tự động**

```bash
sudo chmod +x /opt/zabbix_backup.sh
sudo crontab -e
```
# Thêm dòng sau vào file crontab để tự động 02:00 sẽ server chạy backup

```bash
0 2 * * * /opt/zabbix_backup.sh >> /var/log/zabbix_backup.log 2>&1
```
#Lệnh kiểm tra cronjob tự động backup chạy vào 2h sáng mỗi ngày!
```bash
sudo crontab -l
```
**Bước 4: Tạo và share thư mục lưu file backup trên Window server**

#1. Chuẩn bị trên Windows Server

#Tạo một thư mục trên Windows Server (Ví dụ: D:\Zabbix_Backup).

#Click chuột phải vào thư mục -> Properties -> Tab Sharing -> Advanced Sharing.

#Tích chọn Share this folder. Chú ý tên ở ô Share name (ví dụ mặc định là Zabbix_Backup).

#Bấm nút Permissions, cấp quyền Full Control cho tài khoản Windows mà bạn định dùng.


#2. Cài đặt công cụ và tạo thư mục ảo trên Ubuntu Server
```bash
sudo apt update && sudo apt install -y cifs-utils
sudo mkdir -p /mnt/windows_backup
```
#3. Tạo file lưu thông tin đăng nhập an toàn
```bash
sudo nano /root/.smb_creds
```

#Dán nội dung sau vào và thay bằng tài khoản của Windows Server:
```bash
username=TAI_KHOAN_WINDOWS
password=MAT_KHAU_WINDOWS
domain=WORKGROUP
```
(Lưu ý: Nếu Windows Server của bạn nằm trong hệ thống Domain Controller (AD), hãy thay chữ WORKGROUP bằng tên Domain của bạn.)
#Lưu file lại và phân quyền tuyệt đối bảo mật:
```bash
sudo chmod 600 /root/.smb_creds
```
#4. Cấu hình tự động kết nối ổ đĩa (Auto-Mount). Mở file cấu hình ổ đĩa của Ubuntu Server:
```bash
sudo nano /etc/fstab
```
#Kéo xuống DƯỚI CÙNG của file, thêm dòng sau. Nhớ thay đổi 192.168.1.10 thành IP thực tế của Windows Server và Zabbix_Backup thành tên thư mục share:

```bash
//192.168.1.10/Zabbix_Backup /mnt/windows_backup cifs credentials=/root/.smb_creds,iocharset=utf8,file_mode=0777,dir_mode=0777,noperm,vers=3.0 0 0
```
#(Tham số vers=3.0 để ép Ubuntu dùng chuẩn SMB phiên bản 3.0 an toàn và tương thích tốt nhất với Windows Server đời mới). Lưu file lại và chạy lệnh sau để kết nối ngay lập tức:

```bash
sudo systemctl daemon-reload
sudo mount -a
```
#Để chắc chắn ổ đĩa Windows đã được kết nối thành công, bạn gõ lệnh kiểm tra nếu thấy có dòng chữ như hình là được.


```bash
df -h | grep windows_backup
```
<img width="752" height="42" alt="image" src="https://github.com/user-attachments/assets/18b85056-3475-4b0b-9c1c-8666f8efa041" />

#Ghi nhớ 4 thông tin: IP của Windows Server (VD: 192.168.1.10), Tên Share (Zabbix_Backup), Tài khoản Windows, Mật khẩu Windows.

#Lưu lại. Kết quả: Từ nay, sau khi tiến trình sao lưu lúc 2h sáng trên Ubuntu hoàn tất, file nén .tar.gz sẽ tự động "bay" thẳng sang ổ D:\Zabbix_Backup trên con Windows Server của bạn!

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
