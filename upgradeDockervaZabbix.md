# Quy Trình Bảo Trì & Nâng Cấp Hệ Thống Zabbix Docker

Tài liệu này quy định các bước thao tác chuẩn (SOP) để cập nhật hệ thống Zabbix đang giám sát thiết bị. Quá trình nâng cấp được chia thành 2 phân hệ hoàn toàn độc lập nhằm đảm bảo an toàn tối đa cho dữ liệu.

---

## PHẦN 1: Nâng Cấp Lớp Ứng Dụng (Zabbix Server & Web)
*Thực hiện khi có bản vá lỗi (Minor Patch) từ Zabbix, ví dụ nâng cấp từ bản `7.0.30` lên `8.0.x`.*

###Bước 1: Sao lưu dữ liệu (Bắt buộc)
#Trước khi tác động vào cấu trúc Database, bắt buộc phải chạy kịch bản Backup thủ công để tạo điểm khôi phục an toàn.
```bash
sudo /opt/zabbix_backup.sh
```
###Lưu ý: Chỉ tiếp tục làm Bước 2 khi màn hình hiển thị === Backup Hoàn Tất === và bạn đã xác nhận file nén .tar.gz xuất hiện an toàn trên ổ đĩa Share của Windows Server.

###Bước 2: Chỉnh sửa cấu hình phiên bản. Mở file cấu hình Docker Compose để thay đổi tag phiên bản:
```bash
sudo nano /opt/zabbix/docker-compose.yml
```
#Tìm đến khai báo image của dịch vụ zabbix-server và zabbix-web, thay đổi số phiên bản. Ví dụ: Đổi từ alpine-7.0.30 thành bản mới hơn.
```bash
zabbix-server:
    image: zabbix/zabbix-server-pgsql:alpine-8.0.x
    # ...
  zabbix-web:
    image: zabbix/zabbix-web-nginx-pgsql:alpine-8.0.x
    # ...
```

###Bước 3: Tải mã nguồn mới. Kéo (pull) các image mới về máy chủ trước để giảm thiểu tối đa thời gian gián đoạn (downtime) lúc chuyển đổi:
```bash
cd /opt/zabbix
sudo docker compose pull
```
###Bước 4: Thực thi nâng cấp. Chạy lệnh khởi tạo lại hệ thống. Docker sẽ tự động xóa các container cũ và dựng lên container mới với mã nguồn vừa tải:
```bash
cd /opt/zabbix
sudo docker compose up -d
```
###Bước 5: Kiểm tra tiến trình nâng cấp Database ngầm. Sau khi khởi động lên, Zabbix Server có thể sẽ tự động chạy các lệnh SQL để cấu trúc lại Database cho phù hợp với bản mới. Bắt buộc phải kiểm tra log để đảm bảo không có lỗi:
```bash
sudo docker logs -f zabbix-server
```
#Nhấn Ctrl + C để thoát màn hình xem log khi thấy thông báo Zabbix đã chạy ổn định.

## PHẦN 2: Nâng Cấp Lớp Hạ Tầng (Ubuntu & Docker Engine)
###Thực hiện định kỳ mỗi 1 - 3 tháng để vá lỗ hổng bảo mật cho hệ điều hành và Docker. Không ảnh hưởng đến dữ liệu Zabbix.

###Bước 1: Cập nhật danh sách gói phần mềm
```bash
sudo apt update && sudo apt upgrade -y
```

###Bước 3: Dọn dẹp hệ thống. Xóa bỏ các thư viện cũ không còn sử dụng để giải phóng ổ cứng:
```bash
sudo apt autoremove -y
```

###Bước 4: Khởi động lại máy chủ (Nếu cần). Nếu hệ điều hành cập nhật nhân (Kernel mới), hệ thống sẽ yêu cầu khởi động lại.
```bash
sudo reboot
```
