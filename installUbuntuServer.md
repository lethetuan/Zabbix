1. Chọn Ngôn ngữ tại bước khởi đầu. Tại màn hình khởi động đầu tiên, hệ thống sẽ yêu cầu bạn chọn ngôn ngữ cài đặt. Sử dụng phím mũi tên để chọn English (Tiếng Anh) và nhấn Enter. 
<img width="1289" height="781" alt="image" src="https://github.com/user-attachments/assets/d03dee79-150f-4dae-9f63-c5923fc80b74" />


2. Cập nhật Trình cài đặt: Tại màn hình thông báo bản cập nhật, chọn "Continue without updating" và nhấn Enter để tiếp tục.
<img width="1243" height="777" alt="image" src="https://github.com/user-attachments/assets/89167e4e-1974-4ed8-8d8a-71e9eff7ddcb" />


 3. Cấu hình Bàn phím:Giữ nguyên cấu hình Layout và Variant mặc định là English (US), di chuyển xuống nút Done và nhấn Enter để tiếp tục.
<img width="1251" height="787" alt="image" src="https://github.com/user-attachments/assets/a21f9e6c-622a-4c82-9300-24d0bc5db440" />


 4. Chọn Loại Cài đặt: Lựa chọn phiên bản Ubuntu Server, sau đó di chuyển xuống nút Done và nhấn phím Enter để tiếp tục.
<img width="1255" height="779" alt="image" src="https://github.com/user-attachments/assets/d5faa5fd-21a8-416f-ab6f-f0bfcdabe1ef" />


 5. Mở Tùy chọn Chỉnh sửa IPv4: Tại màn hình cấu hình mạng (Network configuration), di chuyển đến card mạng cần thiết lập, nhấn Enter để mở menu và chọn "Edit IPv4" để tiến hành chỉnh sửa địa chỉ IP cho server.
<img width="1225" height="773" alt="image" src="https://github.com/user-attachments/assets/304ff590-e375-45f9-a21b-7e9daeb7fff4" />


6. Chuyển sang Cài đặt Cấu hình Thủ công: Thay đổi tùy chọn IPv4 Method từ mặc định sang "Manual" và nhấn Enter để có thể cài đặt IP tĩnh cho Server Ubuntu.
<img width="1271" height="781" alt="image" src="https://github.com/user-attachments/assets/a97998bc-713f-4478-8ff4-2391001fe7a0" />



7.  Điền Thông tin IP Tĩnh: Điền các thông tin địa chỉ mạng bao gồm Subnet (10.1.1.0/24), Address (10.1.1.30), Gateway (10.1.1.2) và Name servers (10.1.1.2). Sau khi điền đầy đủ thông tin, di chuyển xuống nút Save sau đó nhấn Enter để lưu lại.
<img width="1089" height="707" alt="image" src="https://github.com/user-attachments/assets/695cafae-7374-476f-b732-7fc71abd3675" />


8.  Hoàn thành cài đặt Địa chỉ IP: Kiểm tra lại thông tin IP tĩnh (static) vừa thiết lập để đảm bảo tính chính xác, sau đó di chuyển xuống nút Done và nhấn Enter để hoàn thành việc cài đặt địa chỉ IP.
<img width="1013" height="787" alt="image" src="https://github.com/user-attachments/assets/0b7029d1-757f-42ed-ab63-875004ebed73" />



9. Bỏ qua Cấu hình Proxy: Nếu hệ thống không yêu cầu cấu hình Proxy để kết nối internet thì có thể bỏ qua phần này, chỉ cần di chuyển xuống nút Done và nhấn Enter.
<img width="1089" height="785" alt="image" src="https://github.com/user-attachments/assets/3e4003bc-c6d5-4f43-8a08-4b36b11ada4a" />



10. Xác nhận Cấu hình Mirror:Để nguyên địa chỉ Ubuntu archive mirror mặc định, di chuyển xuống nút Done này và nhấn Enter để hoàn thành.
<img width="1097" height="781" alt="image" src="https://github.com/user-attachments/assets/a9e65210-24b0-444f-bf76-7c4cce0ce9b5" />


 11.  Tùy chọn Bố cục Lưu trữ (Storage Layout): Tại bước Guided storage configuration, bạn có hai lựa chọn chính: 
- Di chuyển dấu [X] xuống mục Custom storage layout nếu bạn muốn tự tay chia phân vùng ổ cứng theo ý muốn cá nhân. Hoặc để mặc định ở mục Use an entire disk (thiết lập dưới dạng LVM group) để hệ thống tự động chia. Sau khi chọn xong, di chuyển xuống nút Done và nhấn Enter để tiếp tục.
<img width="1165" height="789" alt="image" src="https://github.com/user-attachments/assets/1e07fd2b-efe3-4456-9bf9-c64d513699a5" />


 Vì mình không muốn Tùy chọn Bố cục Lưu trữ nên mình cài đặt cấu hình mặc định bằng cách Di chuyển dấu [X] lên mục Use an entire disk nhé.  
<img width="1047" height="805" alt="image" src="https://github.com/user-attachments/assets/fd24f8ac-41b5-4068-b309-d103c9e2cafd" />


12. Kiểm tra Cấu hình Phân vùng. Hệ thống sẽ hiển thị bảng tóm tắt (File system summary) các phân vùng sẽ được tạo. 

Ví dụ trong hình: 

19GB được dùng làm phân vùng hệ điều hành chính (mount vào /) .

2GB được gắn vào /boot, chứa nhân hệ điều hành (kernel). 

Phần free space 19GB là không gian trống chưa được sử dụng, bạn có thể dùng dung lượng này để mở rộng hệ điều hành trong tương lai. Nhấn Enter tại nút Done để đi tiếp.  

<img width="1061" height="799" alt="image" src="https://github.com/user-attachments/assets/f2ee569d-cefd-4166-8abb-a6b5b48f5ff3" />



13.Xác nhận Cảnh báo Mất Dữ liệu: Một bảng cảnh báo Confirm destructive action sẽ xuất hiện, thông báo rằng hành động tiếp theo sẽ định dạng ổ cứng và làm mất toàn bộ dữ liệu cũ. Di chuyển xuống nút Continue và nhấn Enter để đồng ý bắt đầu ghi các phân vùng.
<img width="1139" height="689" alt="image" src="https://github.com/user-attachments/assets/91e7a0bb-fa10-4f05-899d-32845c62cff4" />


 14.Thiết lập Tài khoản Quản trị (Profile Configuration): Bạn cần nhập các thông tin cơ bản để tạo tài khoản đăng nhập vào hệ thống:

- Your name: Tên hiển thị của bạn.
- Your servers name: Tên định danh của máy chủ.
- Pick a username: Tên tài khoản dùng để đăng nhập.
- Choose a password & Confirm: Nhập và xác nhận lại mật khẩu.
- Sau khi điền đủ thông tin, di chuyển xuống nút Done và nhấn Enter.
<img width="1039" height="781" alt="image" src="https://github.com/user-attachments/assets/c4271215-e240-458d-a921-d0bb1c60c794" />



15.Bỏ qua Cài đặt Ubuntu Pro:Màn hình giới thiệu dịch vụ Ubuntu Pro. Bạn di chuyển dấu [X] xuống mục Skip for now (Bỏ qua lúc này), sau đó di chuyển xuống nút Continue để đi tiếp.
<img width="1041" height="803" alt="image" src="https://github.com/user-attachments/assets/e6f8d10a-e67f-48cb-8745-28f9295602f5" />



 16.Cài đặt OpenSSH Server: Tại màn hình SSH Configuration, hãy đảm bảo di chuyển và đánh dấu [X] vào ô Install OpenSSH server. Tính năng này rất quan trọng để bạn có thể dùng các phần mềm kết nối an toàn từ xa. Sau đó di chuyển xuống nút Done và nhấn Enter.
<img width="1035" height="787" alt="image" src="https://github.com/user-attachments/assets/1b8aa61f-eba5-4c9d-967c-a59171a505ad" />



17.Bỏ qua các Gói Phần mềm (Snaps): Màn hình Featured server snaps gợi ý các ứng dụng phổ biến trên server (như Docker, microk8s, powershell,...). Nếu không có nhu cầu sử dụng ngay lúc này, bạn không cần chọn gì cả, chỉ cần di chuyển xuống nút Done và nhấn Enter.
<img width="1057" height="801" alt="image" src="https://github.com/user-attachments/assets/5d06abb0-59cd-4076-9fd5-26794d96aaae" />



 18.Chờ Quá trình Cài đặt: Hệ thống chuyển sang màn hình Installing system và tự động chạy các dòng lệnh để giải nén, cài đặt nhân Linux, cấu hình mạng và tải về bản cập nhật. Bạn chỉ cần chờ đợi cho đến khi hoàn thành.
<img width="1055" height="779" alt="image" src="https://github.com/user-attachments/assets/b89a119e-847d-4490-9b1e-39077d785851" />




19.Hoàn tất và Khởi động lại: Khi góc trên bên trái xuất hiện dòng chữ Installation complete!, quá trình cài đặt đã xong. Di chuyển xuống nút Reboot Now ở cuối màn hình và nhấn Enter để khởi động lại máy chủ. Máy chủ Ubuntu của bạn đã sẵn sàng để đăng nhập và sử dụng!
<img width="1039" height="777" alt="image" src="https://github.com/user-attachments/assets/36aeebae-d798-4935-b7b8-90c1a9ba3293" />

 20. Sau khi khởi động lại chúng ta sẽ tiến hành đăng nhập tài khoản vào Server
<img width="949" height="349" alt="image" src="https://github.com/user-attachments/assets/f0dabec0-17c3-4b11-b329-af706f3cd1bc" />



Chúc mừng bạn đã hoàn tất quá trình cài đặt hệ điều hành Ubuntu Server! Sau khi khởi động lại và đăng nhập thành công bằng tài khoản vừa tạo, máy chủ của bạn đã hoàn toàn sẵn sàng để đưa vào hoạt động.
<img width="1117" height="637" alt="image" src="https://github.com/user-attachments/assets/5adc9eb9-3a1d-406f-8923-2bbe5ddf0842" />








