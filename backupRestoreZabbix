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
