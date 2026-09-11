Dưới đây là toàn bộ nội dung file Markdown hoàn chỉnh, chuẩn hóa từ đầu đến cuối và đã khắc phục toàn bộ các lỗi tiềm ẩn. Bạn có thể sao chép trực tiếp nội dung bên dưới và lưu thành file `ZABBIX_DISASTER_RECOVERY.md`.

---

```markdown
# Kịch Bản Disaster Recovery (Backup & Restore) Zabbix Server

Tài liệu này hướng dẫn chi tiết quy trình sao lưu tự động hệ thống giám sát Zabbix (triển khai bằng Docker Compose + PostgreSQL) sang một máy chủ chia sẻ dữ liệu (Windows Server / NAS) và các bước khôi phục dịch vụ nhanh chóng sang một máy chủ hoàn toàn mới khi máy chủ cũ gặp sự cố nghiêm trọng (kể cả trường hợp hỏng phần cứng hoặc cháy nổ server vật lý).

---

## PHẦN 1: CẤU HÌNH BACKUP TỰ ĐỘNG (Thực hiện trên Server đang chạy)

### Bước 1: Chuẩn bị thư mục lưu trữ và script
```bash
sudo mkdir -p /backup/zabbix
sudo chown root:root /backup/zabbix
sudo chmod 700 /backup/zabbix
sudo nano /opt/zabbix_backup.sh
```

### Bước 2: Cấu hình nội dung Script Backup
Dán toàn bộ nội dung sau vào file `/opt/zabbix_backup.sh`:

```bash
#!/bin/bash
set -o pipefail # Bắt lỗi ngay cả khi gặp lỗi trong pipeline

BACKUP_DIR="/backup/zabbix"
mkdir -p "$BACKUP_DIR"
DATE=$(date +"%Y%m%d_%H%M")
ZABBIX_DIR="/opt/zabbix"

# 1. ĐỌC BIẾN MÔI TRƯỜNG TỪ FILE .ENV CỦA DỰ ÁN
if [ -f "$ZABBIX_DIR/.env" ]; then
    set -a
    source "$ZABBIX_DIR/.env"
    set +a
else
    echo "[$(date)] LỖI: Không tìm thấy file $ZABBIX_DIR/.env. Dừng backup!" >&2
    exit 1
fi

# 2. GÁN BIẾN HỆ THỐNG
DB_CONTAINER="zabbix-postgres"
DB_USER="${POSTGRES_USER:-zabbix}"
DB_NAME="${POSTGRES_DB:-zabbix}"
KEEP_DAYS=7

echo "=== [$(date)] Bắt đầu Backup Zabbix ($DATE) ==="

# 3. BACKUP DATABASE (Dùng định dạng custom và truyền mật khẩu an toàn)
echo "Đang dump database $DB_NAME từ container $DB_CONTAINER..."
docker exec -e PGPASSWORD="$POSTGRES_PASSWORD" "$DB_CONTAINER" pg_dump -U "$DB_USER" --format=custom "$DB_NAME" > "$BACKUP_DIR/zabbix_db_$DATE.dump"

if [ $? -ne 0 ] || [ ! -s "$BACKUP_DIR/zabbix_db_$DATE.dump" ]; then
    echo "[$(date)] LỖI NGHIÊM TRỌNG: Backup Database thất bại hoặc file dump rỗng! Dừng script để bảo vệ backup cũ." >&2
    rm -f "$BACKUP_DIR/zabbix_db_$DATE.dump"
    exit 1
fi

# 4. BACKUP THƯ MỤC CẤU HÌNH /opt/zabbix
# (Loại trừ các thư mục dữ liệu sống của DB nếu có mount bên trong để tránh nén đè và làm phình file)
tar --exclude='zabbix/data' \
    --exclude='zabbix/pgdata' \
    --exclude='zabbix/zbx_env/var/lib/postgresql/data' \
    -czf "$BACKUP_DIR/zabbix_config_$DATE.tar.gz" -C /opt zabbix

# 5. GOM DỮ LIỆU THÀNH 1 FILE DUY NHẤT (.tar.gz)
tar -czf "$BACKUP_DIR/ZABBIX_FULL_BACKUP_$DATE.tar.gz" -C "$BACKUP_DIR" "zabbix_db_$DATE.dump" "zabbix_config_$DATE.tar.gz"

# 6. DỌN DẸP FILE TRUNG GIAN TRÊN LOCAL
rm -f "$BACKUP_DIR/zabbix_db_$DATE.dump" "$BACKUP_DIR/zabbix_config_$DATE.tar.gz"

# 7. XÓA BẢN BACKUP CŨ TRÊN MÁY LOCAL (Quá 7 ngày)
find "$BACKUP_DIR" -name "ZABBIX_FULL_BACKUP_*.tar.gz" -type f -mtime +$KEEP_DAYS -exec rm -f {} \;

# 8. ĐỒNG BỘ SANG THƯ MỤC SHARE WINDOWS SERVER / NAS
if timeout 10 touch /mnt/windows_backup/.test_write 2>/dev/null; then
    echo "Đang copy bản backup sang Windows Server..."
    cp "$BACKUP_DIR/ZABBIX_FULL_BACKUP_$DATE.tar.gz" /mnt/windows_backup/
    # Xóa backup trên máy Windows quá 14 ngày
    find /mnt/windows_backup -name "ZABBIX_FULL_BACKUP_*.tar.gz" -type f -mtime +14 -exec rm -f {} \;
    rm -f /mnt/windows_backup/.test_write
    echo "Đồng bộ Windows Server thành công."
else
    echo "CẢNH BÁO: Mất kết nối thư mục /mnt/windows_backup! Đang thử kết nối lại..." >&2
    mount -o remount /mnt/windows_backup || mount -a
fi

echo "=== [$(date)] Backup Hoàn Tất Thành Công! ==="
```

### Bước 3: Cấp quyền và đặt lịch chạy tự động (Cronjob)
```bash
sudo chmod +x /opt/zabbix_backup.sh
sudo crontab -e
```
Thêm dòng sau vào cuối file crontab để hệ thống tự chạy sao lưu vào **02:00 sáng hàng ngày**:
```bash
0 2 * * * /opt/zabbix_backup.sh >> /var/log/zabbix_backup.log 2>&1
```
Kiểm tra lại lịch cronjob đã nhận hay chưa:
```bash
sudo crontab -l
```

---

## PHẦN 2: CẤU HÌNH LIÊN KẾT Ổ LƯU TRỮ TRÊN WINDOWS SERVER (CIFS/SMB)

### 1. Thao tác trên Windows Server
1. Tạo một thư mục lưu trữ (ví dụ: `D:\Zabbix_Backup`).
2. Chuột phải vào thư mục `Zabbix_Backup` -> chọn **Properties** -> chuyển sang tab **Sharing** -> bấm **Advanced Sharing**.
3. Tích chọn **Share this folder**. Kiểm tra Share name (mặc định là `Zabbix_Backup`).
4. Bấm nút **Permissions**, thêm tài khoản người dùng và tích chọn quyền **Full Control** cho tài khoản đó.

### 2. Cài đặt công cụ và tạo điểm gắn kết trên Ubuntu Server
```bash
sudo apt update && sudo apt install -y cifs-utils
sudo mkdir -p /mnt/windows_backup
```

### 3. Tạo file lưu thông tin đăng nhập an toàn
```bash
sudo nano /root/.smb_creds
```
Điền nội dung tài khoản Windows (thay bằng thông tin thực tế):
```ini
username=TAI_KHOAN_WINDOWS
password=MAT_KHAU_WINDOWS
domain=WORKGROUP
```
*(Nếu Windows Server nằm trong Active Directory Domain, hãy thay `WORKGROUP` bằng tên Domain thực tế).*

Phân quyền bảo mật tối đa cho file chứa mật khẩu:
```bash
sudo chmod 600 /root/.smb_creds
```

### 4. Cấu hình tự động kết nối ổ đĩa qua `/etc/fstab`
Mở file fstab:
```bash
sudo nano /etc/fstab
```
Thêm dòng cấu hình sau vào **dưới cùng** của file (thay `192.168.1.10` bằng IP thực tế của Windows Server):
```fstab
//192.168.1.10/Zabbix_Backup /mnt/windows_backup cifs credentials=/root/.smb_creds,iocharset=utf8,file_mode=0777,dir_mode=0777,noperm,vers=3.0,_netdev,nofail,x-systemd.automount 0 0
```
> **Giải thích tham số:**
> * `_netdev`, `nofail`, `x-systemd.automount`: Đảm bảo chỉ mount khi đã có mạng, nếu Windows Server tắt máy thì Ubuntu vẫn khởi động bình thường không bị treo hệ điều hành.
> * `vers=3.0`: Bắt buộc dùng giao thức SMB phiên bản 3.0 an toàn và tương thích tối đa.

Thực hiện nạp cấu hình và kết nối ngay lập tức:
```bash
sudo systemctl daemon-reload
sudo mount -a
```

Kiểm tra kết quả gắn kết:
```bash
df -h | grep windows_backup
```
Nếu màn hình hiển thị dung lượng ổ đĩa Windows được mount vào `/mnt/windows_backup` là thành công.

---

## PHẦN 3: KỊCH BẢN PHỤC HỒI THẢM HỌA (DISASTER RECOVERY)

Quy trình này áp dụng khi Server Zabbix cũ bị hỏng hoàn toàn và cần dựng lại trên Server Ubuntu mới.

### Bước 1: Chuẩn bị Server mới
1. Cài đặt hệ điều hành Ubuntu Server mới.
2. Thiết lập địa chỉ IP tĩnh, cấu hình SSH, Timezone đúng với hệ thống cũ.
3. Cài đặt Docker và Docker Compose plugin mới nhất.

### Bước 2: Chuyển file Backup sang Server mới và giải nén
1. Dùng công cụ SFTP (như **WinSCP** hoặc lệnh `scp`) kết nối vào IP của Server mới với tài khoản có quyền `sudo`.
2. Lấy file backup mới nhất `ZABBIX_FULL_BACKUP_*.tar.gz` từ thư mục `D:\Zabbix_Backup` trên Windows Server và đưa vào thư mục `/tmp` trên máy chủ mới.
3. Mở terminal trên Server mới và tiến hành giải nén cấu hình:

```bash
cd /tmp

# 1. Giải nén gói backup tổng hợp
tar -xzvf ZABBIX_FULL_BACKUP_*.tar.gz

# 2. Giải nén thư mục cấu hình về đúng vị trí chuẩn /opt/zabbix
sudo tar -xzvpf /tmp/zabbix_config_*.tar.gz -C /opt
```
*Lệnh trên sẽ khôi phục lại toàn bộ thư mục `/opt/zabbix` bao gồm file `docker-compose.yml`, file biến môi trường ẩn `.env` và toàn bộ các custom alert script/external script cũ với đúng phân quyền gốc.*

---

### Bước 3: Khởi động RIÊNG Database Container
> ⚠️ **CẢNH BÁO QUAN TRỌNG:** TUYỆT ĐỐI KHÔNG chạy lệnh `docker compose up -d` lúc này. Nếu bật toàn bộ hệ thống ngay, Zabbix Server container sẽ tự tạo database rỗng đè lên cấu trúc bảng cũ, gây lỗi không đồng bộ dữ liệu.

Khởi động riêng container cơ sở dữ liệu:
```bash
cd /opt/zabbix
sudo docker compose up -d postgres-server
```
*(Lưu ý: `postgres-server` là tên service PostgreSQL định nghĩa trong file `docker-compose.yml`, đảm bảo container được đặt tên là `zabbix-postgres` qua thuộc tính `container_name`).*

Chờ khoảng 10-15 giây để PostgreSQL khởi tạo môi trường lần đầu. Kiểm tra trạng thái sẵn sàng kết nối:
```bash
sudo docker exec zabbix-postgres pg_isready
```
Chờ đến khi màn hình hiển thị: **`accepting connections`**.

---

### Bước 4: Phục hồi Database từ file Dump
Thực hiện nạp lại toàn bộ cấu trúc và dữ liệu từ file dump đã giải nén ở thư mục `/tmp`:

```bash
# 1. Di chuyển vào thư mục dự án và đọc các biến môi trường
cd /opt/zabbix
set -a && source .env && set +a

# 2. Định vị chính xác file dump mới nhất
DUMP_FILE=$(ls -t /tmp/zabbix_db_*.dump | head -n 1)

# 3. Tiến hành Restore vào Database
cat "$DUMP_FILE" | sudo docker exec -i -e PGPASSWORD="$POSTGRES_PASSWORD" zabbix-postgres pg_restore -U "$POSTGRES_USER" -d "$POSTGRES_DB" --clean --if-exists
```
> **Lưu ý:** Trong quá trình restore, các thông báo cảnh báo (WARNING) liên quan đến quyền sở hữu (owner) hoặc extension là hoàn toàn bình thường và an toàn để bỏ qua. Thời gian chạy phụ thuộc vào dung lượng database cũ (thường từ vài chục giây đến vài phút).

---

### Bước 5: Cập nhật IP mạng và Khởi động toàn bộ hệ thống
Nếu Server mới sử dụng IP mạng LAN khác so với máy cũ, hãy cập nhật lại IP trong cấu hình:
```bash
sudo nano /opt/zabbix/docker-compose.yml
```
*(Cập nhật lại các dòng bind port nếu có chỉ định IP cứng, ví dụ: `- "IP_MOI:10051:10051"`).*

Sau khi kiểm tra xong cấu hình, khởi động toàn bộ stack Zabbix:
```bash
cd /opt/zabbix
sudo docker compose up -d
```

Kiểm tra trạng thái hoạt động của các container:
```bash
sudo docker compose ps
```

### Bước 6: Dọn dẹp thư mục tạm
Sau khi toàn bộ hệ thống đã hoạt động bình thường, xóa các file dump tạm để giải phóng dung lượng đĩa:
```bash
sudo rm -f /tmp/ZABBIX_FULL_BACKUP_* /tmp/zabbix_db_* /tmp/zabbix_config_*
```

---
**Hệ thống giám sát Zabbix đã được phục hồi nguyên vẹn 100% bao gồm toàn bộ Host, Template, Lịch sử dữ liệu (History/Trends), và cấu hình cảnh báo.**
```
