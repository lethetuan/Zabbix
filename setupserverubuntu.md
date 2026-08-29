#Cẩm nang Quản trị Hệ thống Linux: 
# Phần 1: Cấu hình IP Tĩnh trên Ubuntu Server từ A-Z 

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
  GNU nano 8.7.1                                   00-installer-config.yaml
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens33:
      addresses:
      - 10.1.1.30/24
      dhcp6: false
      match:
        macaddress: 00:0c:29:98:84:94
      nameservers:
        addresses:
        - 10.1.1.2
        search: []
      routes:
      - to: default
        via: 10.1.1.2
      set-name: ens33
  version: 2
```

**Lưu ý kỹ thuật:**
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

## Phần 2: Chi tiết cấu hình SSH trên Ubuntu Server

### Bước 1: Cập nhật hệ thống
Luôn đảm bảo hệ thống được cập nhật trước khi cài đặt phần mềm mới:

```bash
sudo apt update
sudo apt upgrade -y
```

### Bước 2: Cài đặt OpenSSH Server

```bash
sudo apt install openssh-server -y
```

### Bước 3: Kích hoạt SSH tự động khởi động cùng hệ thống

```bash
sudo systemctl enable --now ssh
```

Kiểm tra trạng thái SSH xem đã hoạt động (running) chưa:

```bash
sudo systemctl status ssh
```

![SSH Status](https://github.com/user-attachments/assets/01b9616c-1d23-4f2a-a4ef-8424708f01e8)

### Bước 4: Cấu hình Tường lửa (UFW)
Mở cổng trên Tường lửa (UFW) cho phép lưu lượng truy cập SSH qua cổng mặc định (cổng 22):

```bash
sudo ufw allow ssh
sudo ufw enable
```

![UFW Config](https://github.com/user-attachments/assets/989e0911-bcc9-478f-a80c-56949a6a5694)

### Bước 5: Tạo và sao chép SSH Key (Thực hiện trên máy tính Admin - Windows)

**1. Tạo SSH Key trên máy tính Windows của Admin:**
Mở Command Prompt (cmd) hoặc PowerShell và chạy lệnh:

```bash
ssh-keygen -t ed25519 -C "ten_may_tinh_cua_ban"
```
*(Ghi chú: `-C "ten_may_tinh_cua_ban"` chỉ để note lại tên máy tính giúp bạn dễ quản lý).*

**2. Sao chép Public Key lên Server Linux:**
Di chuyển đến thư mục chứa key:

```cmd
cd C:\Users\username\.ssh
```
*(Thay thế `username` bằng username máy tính của bạn).*

Chạy lệnh sau để đẩy key lên server *(Khuyên dùng Command Prompt - cmd để tránh lỗi encoding của PowerShell)*:

```cmd
type id_ed25519.pub | ssh tuan@10.1.1.30 "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

*Lưu ý: Nếu bạn sử dụng PowerShell, lệnh pipe `|` đôi khi sẽ đổi bảng mã sang UTF-16 gây lỗi định dạng key trên Linux. Hãy dùng **Command Prompt (cmd)** cho bước này để an toàn nhất, hoặc dùng lệnh `ssh-copy-id` nếu máy bạn có hỗ trợ.*

**Minh họa lệnh dùng Command Prompt (cmd):**
![CMD Push Key](https://github.com/user-attachments/assets/1afa38a1-0718-42ba-9a74-2386943e5d7c)

**Minh họa lệnh dùng PowerShell:**
![PowerShell Push Key](https://github.com/user-attachments/assets/9bf99a63-7ce6-4190-bf15-152fc3ec9001)

![Key generation process](https://github.com/user-attachments/assets/20b354c0-ce4c-4a06-8af3-a4de525d47bd)

### Bước 6: Đăng nhập SSH bằng Key
Bây giờ bạn có thể đăng nhập vào server mà không cần nhập mật khẩu thông qua lệnh:

```bash
ssh username@dia_chi_ip_server
```

![SSH Login](https://github.com/user-attachments/assets/3ba4fa4d-03ab-4c9a-a6ef-2f9bd40b09c2)

Lưu trữ lại key đăng nhập thành công:

![SSH Key Saved](https://github.com/user-attachments/assets/b2cd2fe5-9e1b-42c7-ad8e-c1cda56baaf0)

### Bước 7: (Tùy chọn/Nâng cao) Vô hiệu hóa đăng nhập bằng mật khẩu
Sau khi bạn đã chắc chắn có thể đăng nhập thành công bằng SSH Key, bạn nên tắt tính năng đăng nhập bằng mật khẩu để ngăn chặn hoàn toàn các cuộc tấn công dò quét mật khẩu (brute-force):

1. Mở file cấu hình sshd:
   ```bash
   sudo nano /etc/ssh/sshd_config
   ```
2. Tìm dòng `#PasswordAuthentication yes` hoặc `PasswordAuthentication yes` và sửa lại thành:
   ```text
   PasswordAuthentication no
   ```
3. Lưu file (`Ctrl + O`, `Enter`, `Ctrl + X`) và khởi động lại dịch vụ SSH:
   ```bash
   sudo systemctl restart ssh
   ```
