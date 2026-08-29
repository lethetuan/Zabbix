# Cẩm nang Quản trị Hệ thống: Cấu hình IP Tĩnh trên Ubuntu Server từ A-Z 

Việc gán IP tĩnh là bước chân đầu tiên và quan trọng nhất khi dựng một máy chủ. Trên Ubuntu Server hiện đại (từ 18.04 trở đi), công cụ chính thống được Google/Canonical áp dụng là **Netplan**.

Dưới đây là quy trình chuẩn chỉnh, an toàn và tối ưu nhất để cấu hình IP tĩnh mà không sợ bị "lockout" khỏi server từ xa.

---

## 1. Nguyên tắc vàng trước khi thao tác
* **Không bao giờ dùng lệnh `netplan apply` ngay lập tức** nếu bạn đang thao tác qua SSH từ xa mà không có cơ hội test. Hãy luôn dùng `netplan try`.
* **Cẩn trọng với khoảng trắng (Whitespace):** YAML cực kỳ nhạy cảm với khoảng trắng. Tuyệt đối dùng phím **Space**, không dùng phím **Tab**.

---

## Bước 1: Xác định tên giao diện mạng (Network Interface)
Truy cập vào server và chạy lệnh để liệt kê các interface đang hoạt động:

```bash
ip a
```

Hoặc dùng lệnh hiện đại hơn để kiểm tra tên card mạng (interface):

```bash
networkctl status
```

**Mục tiêu:** Tìm tên card mạng (ví dụ: `ens33`, `enp3s0`, hoặc `eth0`). Tránh nhầm lẫn với card loopback `lo`.

---

## Bước 2: Kiểm tra thư mục cấu hình Netplan
Các file cấu hình mạng nằm trong thư mục `/etc/netplan/`. Hãy liệt kê xem bạn đang có file nào:

```bash
ls -la /etc/netplan/
```

Thường file sẽ có tên dạng `01-netcfg.yaml`, `50-cloud-init.yaml` hoặc `installer-config.yaml`.

---

## Bước 3: Soạn thảo file cấu hình Netplan
Ví dụ khi liệt kê có 1 file 50-cloud-init.yaml. Hãy mở file cấu hình hiện tại bằng trình soạn thảo (`nano`) bằng câu lệnh:

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

*(Lưu ý: Thay `50-cloud-init.yaml` bằng tên file thực tế trên server của bạn).*

Thay đổi hoặc bổ sung cấu hình theo mẫu chuẩn sau. Chú ý sửa lại tên card mạng, IP, Subnet, Gateway và DNS cho phù hợp với hạ tầng mạng của bạn:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:                  # THAY BẰNG TÊN CARD MẠNG CỦA BẠN (Ví dụ: enp3s0)
      dhcp4: no
      addresses:
        - 192.168.1.100/24  # IP tĩnh và Subnet Mask của bạn
      routes:
        - to: default
          via: 192.168.1.1  # Địa chỉ Gateway của Router/Switch
      nameservers:
        addresses:
          - 8.8.8.8         # DNS chính (Google)
          - 1.1.1.1         # DNS dự phòng (Cloudflare)
```

**Lưu ý kỹ thuật:**
* `renderer: networkd`: Bắt buộc để systemd-networkd quản lý mạng.
* `dhcp4: no`: Tắt nhận IP tự động.
* Cấu trúc `routes` thay thế cho cú pháp `gateway4` cũ (vốn đã bị deprecated từ các bản Netplan gần đây).

Lưu file lại: Nhấn `Ctrl + O`, sau đó `Enter` để ghi, và `Ctrl + X` để thoát.

---

## Bước 4: Kiểm tra và áp dụng an toàn bằng `netplan try`
Đây là mẹo sinh tồn của admin kinh nghiệm. Lệnh này sẽ test cấu hình mới trong vòng 120 giây. Nếu mạng đứt, cấu hình sẽ tự động phục hồi về trạng thái cũ.

```bash
sudo netplan try
```

* Nếu kết nối vẫn ổn định và bạn hài lòng, hãy nhấn **`Enter`** để xác nhận lưu vĩnh viễn.
* Nếu mất kết nối, kệ nó trong 120 giây, server sẽ tự rollback lại mạng cũ, giúp bạn không bị mất quyền điều khiển từ xa.

Nếu bạn đứng trực tiếp tại máy chủ và tự tin 100% với cú pháp, có thể dùng:
```bash
sudo netplan apply
```

---

## Kiểm tra thành quả
Sau khi áp dụng thành công, hãy kiểm tra lại thông tin mạng:

```bash
ip a
```
Và kiểm tra khả năng kết nối Internet (Ping ra ngoài):

```bash
ping -c 4 8.8.8.8
ping -c 4 google.com
```

---
*Chúc hệ thống của bạn luôn uptime 99.99%!*
