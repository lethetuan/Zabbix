# Quy Trình Bảo Trì & Nâng Cấp Hệ Thống Zabbix Docker

Tài liệu này quy định các bước thao tác chuẩn (SOP) để cập nhật hệ thống Zabbix đang giám sát thiết bị. Quá trình nâng cấp được chia thành 2 phân hệ hoàn toàn độc lập nhằm đảm bảo an toàn tối đa cho dữ liệu.

---

## PHẦN 1: Nâng Cấp Lớp Ứng Dụng (Zabbix Server & Web)
*Thực hiện khi có bản vá lỗi (Minor Patch) từ Zabbix, ví dụ nâng cấp từ bản `7.0.30` lên `7.0.35`.*

### Bước 1: Sao lưu dữ liệu (Bắt buộc)
#Trước khi tác động vào cấu trúc Database, bắt buộc phải chạy kịch bản Backup thủ công để tạo điểm khôi phục an toàn.
```bash
sudo /opt/zabbix_backup.sh
```
###Lưu ý: Chỉ tiếp tục làm Bước 2 khi màn hình hiển thị === Backup Hoàn Tất === và bạn đã xác nhận file nén .tar.gz xuất hiện an toàn trên ổ đĩa Share của Windows Server.

### Bước 2: Chỉnh sửa cấu hình phiên bản
Mở file cấu hình Docker Compose để thay đổi tag phiên bản:
