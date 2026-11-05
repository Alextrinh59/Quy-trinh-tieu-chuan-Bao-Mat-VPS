# QUY TRÌNH TIÊU CHUẨN BẢO MẬT MÁY CHỦ VPS

> **Phiên bản:** 2.1  
> **Người yêu cầu & chủ sở hữu ý tưởng:** Anh **Tony-Consulting Director** — yêu cầu xây dựng bộ quy chuẩn bảo mật VPS, và toàn bộ ý tưởng cốt lõi của quy trình này do anh Tony chủ trì, cung cấp thông qua video: https://youtu.be/SZX1mZiJWPA.
> **Nguồn nội dung:** Toàn bộ nội dung kỹ thuật trong tài liệu này được **bóc tách và hệ thống hóa từ video**, bao gồm quy trình bảo mật của *levelsio.dev* (mục *"Securing a new VPS"*) và video thực nghiệm của ByteGrad (*"I tried levelsio VPS setup"*) - **không phải do người soạn thảo tài liệu tự sáng tạo**; phần chỉnh sửa chỉ mang tính thích nghi với môi trường thực tế của công ty (LXC/ZFS, VPS nội bộ).  
> **Phạm vi áp dụng:** Máy chủ ảo (VPS) — dạng KVM hoặc LXC container — chạy hệ điều hành Ubuntu 22.04 LTS / 24.04 LTS hoặc Debian 12.  
> **Định dạng tài liệu:** Được chia thành từng giai đoạn (Phase) độc lập để ánh xạ trực tiếp 1:1 vào script tự động hóa (mỗi Phase tương ứng với một phân đoạn trong script).  
> **Lịch sử thay đổi:** Phiên bản 2.1 — loại bỏ các bước không tương thích với môi trường LXC/ZFS (bước tạo swap file), bổ sung các lưu ý vận hành đặc thù cho container và cho mô hình triển khai nhiều website khách hàng trên cùng một VPS.

---
## 1. Tổng quan

### 1.1. Mục đích
Tài liệu này xác định một quy trình chuẩn hóa, có tính lặp lại (reproducible) nhằm đưa một máy chủ VPS mới khởi tạo (vốn đang mở toàn bộ cổng kết nối và sử dụng mật khẩu `root` mặc định) sang trạng thái **bảo mật, ổn định và sẵn sàng vận hành** trước khi triển khai bất kỳ ứng dụng thực tế nào.

### 1.2. Nguyên lý cốt lõi
Một máy chủ VPS vừa khởi tạo trên mạng Internet sẽ bị các mạng lưới bot tự động dò quét và tấn công vét cạn (brute-force) chỉ trong vòng **vài phút**. Do đó, **thứ tự thực hiện các bước quan trọng hơn bản thân từng bước riêng lẻ**:

> 1. **Cấu hình SSH Key thành công** $\rightarrow$ Mới được tắt tính năng đăng nhập bằng mật khẩu.  
> 2. **Mở cổng SSH trên tường lửa (Firewall)** $\rightarrow$ Mới được kích hoạt tường lửa.  
> 3. **Xác nhận tài khoản mới đăng nhập và dùng `sudo` thành công** $\rightarrow$ Mới được khóa tài khoản `root` cũ.  
> 4. **Mọi dữ liệu sinh ra trên máy chủ (nhật ký, bộ nhớ tạm, bản sao lưu cục bộ) đều phải nằm trong tầm kiểm soát về dung lượng** $\rightarrow$ Không để bất kỳ thành phần nào làm cạn kiệt ổ cứng mà không có cảnh báo.

### 1.3. Các cấp độ bảo mật (Security Levels)

<p align="center">
  <img width="1200" height="630" alt="image" src="https://github.com/user-attachments/assets/18c0f49c-d4dc-4c41-b94b-d1f9d1563a6a" />
</p>


### 1.4. Quy tắc an toàn bắt buộc (Áp dụng cho toàn bộ quy trình)

<p align="center">
  <img width="1200" height="630" alt="image" src="https://github.com/user-attachments/assets/a113c289-823c-4fc4-91ec-105284d93a3f" />
</p>


1. **Luôn mở sẵn đường truy cập dự phòng:** Mở sẵn giao diện Web Console (noVNC hoặc Serial Console) trên trang quản trị của nhà cung cấp VPS. Đây là con đường duy nhất để cứu máy chủ nếu bạn cấu hình sai làm khóa kết nối SSH. *Đối với VPS dạng LXC, cần xác nhận trước với nhà cung cấp hình thức truy cập dự phòng (web console trong panel hay rescue mode), vì không phải nhà cung cấp nào cũng cung cấp noVNC cho container.*
2. **Duy trì song song 2 phiên kết nối SSH:** Khi thực hiện các thay đổi liên quan đến quyền truy cập hoặc tường lửa, luôn giữ một phiên SSH gốc đang mở để có thể hoàn tác ngay nếu xảy ra sự cố.
3. **Kiểm chứng trên phiên kết nối mới:** Mọi thay đổi về cấu hình SSH hoặc tường lửa đều phải được kiểm tra bằng cách **mở một cửa sổ Terminal mới và đăng nhập thử**, tuyệt đối không đóng phiên kết nối cũ trước khi xác nhận phiên mới hoạt động bình thường.
4. **Kiểm tra sau mỗi thao tác (Verify):** Mỗi bước thực hiện đều đi kèm lệnh kiểm tra. Tuyệt đối không chuyển sang bước tiếp theo nếu kết quả kiểm tra chưa đạt yêu cầu.
5. **Nguyên tắc dừng khẩn cấp (Fail-fast):** Không chạy giai đoạn (Phase) tiếp theo nếu giai đoạn hiện tại báo lỗi hoặc chưa hoàn tất.
6. **Tính bất biến (Idempotency):** Việc chạy lại một giai đoạn đã hoàn tất không được làm hỏng cấu hình hiện có hoặc gây xung đột hệ thống (đây là yêu cầu bắt buộc khi xây dựng script tự động hóa).

### 1.5. Thông tin đầu vào cần chuẩn bị (Input)

| Thông tin | Ví dụ minh họa | Ghi chú chi tiết |
| :--- | :--- | :--- |
| **Địa chỉ IP công khai (Public IP)** | `203.0.113.10` | Lấy từ bảng điều khiển (Control Panel) của nhà cung cấp VPS. |
| **Quyền truy cập `root` ban đầu** | Mật khẩu hoặc SSH Key | Được nhà cung cấp cấp sẵn khi khởi tạo máy chủ. |
| **Khóa SSH chuẩn Ed25519** | `~/.ssh/id_ed25519.pub` | Nằm trên **máy tính cá nhân**. Nếu chưa có, tạo mới bằng lệnh: `ssh-keygen -t ed25519` |
| **Tên tài khoản quản trị (Admin User)** | `admin_ops` | **Tuyệt đối không đặt** các tên phổ biến dễ bị dò quét như: `root`, `ubuntu`, `debian`, `test`. |
| **Tên máy chủ (Hostname)** | `vps-prod-01` | Đặt ngắn gọn, rõ nghĩa để dễ phân biệt khi quản lý nhiều máy chủ. |
| **Múi giờ (Timezone)** | `Asia/Ho_Chi_Minh` | Giúp đồng bộ thời gian ghi nhận nhật ký hệ thống (System Logs) và lịch chạy tự động. |

---

## 2. Quy trình thực hiện chi tiết theo từng giai đoạn (Phase)

### PHASE 0 — CHUẨN BỊ & KIỂM TRA AN TOÀN (Pre-checks)

**Mục đích:** Đảm bảo có sẵn phương án dự phòng khẩn cấp và ghi nhận đầy đủ thông số phần cứng của máy chủ trước khi can thiệp vào hệ thống.

| Bước | Nội dung công việc | Lệnh thực thi / Thao tác | Cách kiểm tra & Kết quả kỳ vọng |
| :--- | :--- | :--- | :--- |
| **0.1** | Mở giao diện Web Console của nhà cung cấp VPS | Thao tác trực tiếp trên trình duyệt web tại trang quản trị VPS. | Màn hình Console hiển thị dấu nhắc đăng nhập (TTY) hoạt động bình thường. |
| **0.2** | Khởi tạo cặp khóa SSH Ed25519 trên máy tính cá nhân (nếu chưa có) | `ssh-keygen -t ed25519 -C "vps-admin"` | Chạy lệnh `ls ~/.ssh/id_ed25519.pub` thấy tệp khóa công khai tồn tại. |
| **0.3** | Mở đồng thời **2 phiên kết nối SSH** vào máy chủ VPS | Chạy lệnh `ssh root@<IP>` trên 2 cửa sổ Terminal riêng biệt. | Cả 2 cửa sổ Terminal đều đăng nhập thành công vào quyền `root`. |
| **0.4** | Kiểm tra thông tin mạng, phiên bản hệ điều hành và thông số phần cứng | `hostname; ip -4 addr show \| grep inet; cat /etc/os-release \| head -2; free -h; nproc; df -h /; systemd-detect-virt` | Hiển thị đúng địa chỉ IP công khai, đúng phiên bản Ubuntu 24.04 / Debian 12 và đầy đủ thông số phần cứng (RAM, CPU, dung lượng ổ đĩa, **kiểu ảo hóa**) để ghi nhận vào hồ sơ quản trị máy chủ. |
| **0.5** | Xác nhận mức độ sử dụng ổ đĩa ban đầu để lập đường base-line giám sát | `df -h /` | Ghi nhận tỷ lệ phần trăm sử dụng hiện tại (trên máy chủ mới thường dưới $10\%$) làm chuẩn so sánh cho các đợt bảo trì định kỳ sau này. |

<p align="center">
    <img width="1466" height="920" alt="image" src="https://github.com/user-attachments/assets/93e0f06b-e28e-4279-bd4d-128799abc059" />
</p>


**Lưu ý rủi ro:**
* Nếu chưa mở được Web Console dự phòng và mới chỉ có 1 phiên kết nối SSH: **Dừng lại ngay**, mở đầy đủ trước khi tiếp tục.
* Nếu các thông số phần cứng thực tế không khớp với những gì nhà cung cấp công bố (dung lượng ổ đĩa, RAM, số vCPU): Liên hệ bộ phận hỗ trợ kỹ thuật của nhà cung cấp kiểm tra trước khi triển khai ứng dụng.
* Nếu kết quả `systemd-detect-virt` trả về `lxc` (máy chủ là container): ghi nhận vào hồ sơ máy chủ, vì một số bước trong quy trình sẽ có hành vi khác — cụ thể là **Bước 1.5** (không thể tạo swap file) và **Bước 3.6** (tường lửa phụ thuộc quyền của container).
* Nếu sử dụng Cloudflare Proxy cho tên miền sau này: Các quy tắc chặn theo IP của UFW và Fail2ban vẫn hoạt động bình thường đối với cổng SSH trực tiếp, nhưng cần lưu ý kiến trúc mạng để tránh cấu hình chặn nhầm dải IP của Cloudflare đối với lưu lượng Web.

---

### PHASE 1 — THIẾT LẬP HỆ THỐNG NỀN TẢNG (Base System)

**Mục đích:** Cập nhật toàn bộ bản vá hệ điều hành, cài đặt các gói công cụ nền tảng, chuẩn hóa cấu hình hệ thống cơ bản và thiết lập các cơ chế bảo vệ, kiểm soát tài nguyên hệ thống.

| Bước | Nội dung công việc | Lệnh thực thi | Cách kiểm tra & Kết quả kỳ vọng |
| :--- | :--- | :--- | :--- |
| **1.1** | Cập nhật danh sách và nâng cấp toàn bộ gói phần mềm hiện có | `sudo apt update && sudo apt full-upgrade -y` |  <img width="640" height="270" alt="image" src="https://github.com/user-attachments/assets/43d1503e-ba4a-471c-9f1e-5c38df4b70d7" />|
| **1.2** | Cài đặt bộ công cụ bảo mật và quản trị cơ bản | `sudo apt install -y ufw fail2ban unattended-upgrades curl ca-certificates vim sudo logrotate` | Chạy lệnh `dpkg -l ufw fail2ban` xác nhận các gói đã ở trạng thái `ii` (đã cài đặt). <img width="1248" height="250" alt="image" src="https://github.com/user-attachments/assets/efcf1a9a-01ce-4819-97f8-980bf5a108be" /> |
| **1.3** | Thiết lập tên định danh máy chủ (Hostname) | `sudo hostnamectl set-hostname <HOSTNAME>` | Chạy lệnh `hostname` trả về đúng tên mới vừa đặt. |
| **1.4** | Thiết lập múi giờ chuẩn cho hệ thống | `sudo timedatectl set-timezone <TZ>` | Chạy lệnh `timedatectl` hiển thị đúng múi giờ mong muốn. <img width="898" height="281" alt="image" src="https://github.com/user-attachments/assets/55c9afa2-01fb-4143-bdf4-5a6d303250bd" /> |
| **1.5** | Kiểm tra khả năng dùng bộ nhớ ảo (Swap) và ghi nhận baseline dung lượng RAM *(Lưu ý: trên môi trường LXC chạy hệ thống tệp ZFS của công ty, bước tạo swap file **không thể thực hiện** vì kernel từ chối nhận swap file tạo trên ZFS. Vì vậy quy trình thay chuỗi lệnh tạo swap bằng việc kiểm tra và ghi nhận dung lượng RAM thực tế; nếu sau này có nhu cầu bắt buộc cần Swap, yêu cầu nhà cung cấp cấp sẵn swap partition từ phía host)* | `free -h` *(ghi nhận tổng RAM; kiểm tra xem nhà cung cấp có cấp sẵn Swap nào không)* | Dung lượng RAM thực tế được ghi nhận vào hồ sơ máy chủ. Kết quả không có Swap là **trạng thái bình thường** của môi trường này; việc theo dõi sức khỏe RAM được thực hiện định kỳ tại **Mục 4.2, hạng mục 5**. |
| **1.6** | Khởi tạo vùng nhớ tạm (tmpfs) $1\text{ GB}$ cho thư mục `/tmp` *(Giúp các tiến trình ghi dữ liệu tạm không làm đầy ổ cứng vật lý; khi máy khởi động lại, toàn bộ dữ liệu tạm tự động được dọn sạch)* | `sudo systemctl set-property tmp.mount SizeMax=1G && sudo mount -o remount,size=1G tmp` | Chạy lệnh `df -h /tmp` hiển thị dung lượng $1\text{ GB}$ trên hệ thống tập tin tmpfs. |
| **1.7** | Ràng buộc dung lượng tối đa cho thư mục bộ nhớ đệm của hệ thống | Kiểm tra và chỉnh sửa tệp `/etc/tmpfiles.d/` nếu cần, đảm bảo chính sách dọn dẹp bộ nhớ đệm mặc định của hệ thống được giữ nguyên (thực hiện mỗi ngày, giữ lại tối đa 30 ngày). | Chạy lệnh `systemd-tmpfiles --clean --dry-run` không phát hiện bất kỳ mục nào vượt quá ngưỡng dung lượng cho phép. |
| **1.8** | Ràng buộc độ dài nhật ký của dịch vụ systemd *(Nhật ký hệ thống là một trong những nguyên nhân phổ biến nhất khiến ổ đĩa đầy dần theo thời gian, đặc biệt trên các máy chủ có dung lượng lưu trữ hạn chế)* | Dùng lệnh `sudo tee /etc/systemd/journald.conf.d/limit.conf` để ghi:<br>`[Journal]`<br>`SystemMaxUse=500M`<br>`SystemMaxFileSize=50M`<br>Sau đó chạy `sudo systemctl restart systemd-journald` | Chạy lệnh `journalctl --disk-usage` cho thấy dung lượng chiếm dụng hiện đã được giới hạn quanh mức $500\text{ MB}$ hoặc thấp hơn. <img width="1019" height="384" alt="image" src="https://github.com/user-attachments/assets/c19a9a5b-278b-4ad3-a7e5-d299829342e2" />|
| **1.9** | Khởi động lại máy chủ nếu có bản cập nhật nhân hệ điều hành (Kernel) | `if [ -f /var/run/reboot-required ]; then sudo reboot; fi` | Máy chủ khởi động lại thành công, kết nối lại được qua SSH và toàn bộ cấu hình ở các bước 1.6, 1.7, 1.8 được áp dụng đầy đủ. *(Trên VPS dạng LXC, tệp `reboot-required` gần như không xuất hiện vì container dùng chung kernel với máy host — lệnh này tự động bỏ qua trong trường hợp đó.)* <img width="1140" height="249" alt="image" src="https://github.com/user-attachments/assets/4bd34186-01ea-4f9b-b26d-de24afb8908d" /> |

**Lưu ý tương thích hệ điều hành:** Các câu lệnh ở giai đoạn này hoạt động hoàn toàn giống nhau trên cả Ubuntu 24.04 và Debian 12.

**Lưu ý cho môi trường LXC (container):** Các bước 1.6, 1.7, 1.8 vẫn thực hiện bình thường trong container vì systemd và journald chạy độc lập bên trong. Khi chạy `sudo sysctl --system` ở Bước 1.8, hàng loạt dòng cảnh báo `ignoring: Read-only file system` là **bình thường** — vùng `/proc/sys` của container bị khóa ghi bởi máy host (chỉ `/proc/sys/net` ghi được), và các cảnh báo này không ảnh hưởng đến tiến độ quy trình.

---

### PHASE 2 — QUẢN LÝ TÀI KHOẢN & SIẾT CHẶT BẢO MẬT SSH (Users & SSH Hardening)
> ⚠️ **Cảnh báo:** Đây là giai đoạn nhạy cảm nhất trong toàn bộ quy trình. Quản trị viên bắt buộc phải thực hiện tuần tự từng bước, không bỏ qua các bước kiểm tra trung gian.

**Mục đích:** Vô hiệu hóa hoàn toàn quyền đăng nhập trực tiếp vào tài khoản `root` và tắt phương thức xác thực bằng mật khẩu. Sau giai đoạn này, cách duy nhất để truy cập máy chủ là sử dụng **SSH Key thông qua tài khoản quản trị viên (Admin User)**.

| Bước | Nội dung công việc | Lệnh thực thi / Thao tác | Cách kiểm tra & Kết quả kỳ vọng |
| :--- | :--- | :--- | :--- |
| **2.1** | Tạo tài khoản quản trị viên mới và cấp quyền `sudo` | **Ubuntu:** `sudo adduser <USER> && sudo usermod -aG sudo <USER>`<br>**Debian:** `sudo useradd -m -s /bin/bash <USER> && sudo usermod -aG sudo <USER>` | Chạy lệnh `id <USER>` thấy tài khoản nằm trong nhóm `sudo`. |
| **2.2** | Thiết lập mật khẩu mạnh cho tài khoản mới (dùng khi xác thực lệnh `sudo`) | `sudo passwd <USER>` *(nhập mật khẩu trực tiếp trên màn hình)* | Chạy lệnh `sudo -l -U <USER>` xác nhận tài khoản có quyền thực thi lệnh quản trị. |
| **2.3** | Sao chép khóa công khai (SSH Public Key) vào tài khoản quản trị mới | Từ máy cá nhân chạy: `ssh-copy-id <USER>@<IP>`<br>*(Hoặc dán thủ công nội dung khóa vào tệp `/home/<USER>/.ssh/authorized_keys`, sau đó phân quyền `chmod 700 ~/.ssh` và `chmod 600 ~/.ssh/authorized_keys`)* | **Mở một cửa sổ Terminal mới** trên máy cá nhân và chạy `ssh <USER>@<IP>`. Hệ thống phải cho phép đăng nhập thẳng bằng SSH Key mà không hỏi mật khẩu. |
| **2.4** | ✅ **Chốt kiểm tra bắt buộc số 1:** Xác nhận quyền quản trị trên phiên kết nối mới | Tại cửa sổ Terminal vừa đăng nhập bằng tài khoản `<USER>`, chạy lệnh:<br>`sudo -v && sudo whoami` | Hệ thống yêu cầu nhập mật khẩu của `<USER>` và in ra kết quả là `root`. |
| **2.5** | Tạo tệp cấu hình siết chặt bảo mật dịch vụ SSH | Dùng lệnh `sudo tee /etc/ssh/sshd_config.d/00-hardening.conf` để ghi các thiết lập sau:<br>`PermitRootLogin no`<br>`PasswordAuthentication no`<br>`KbdInteractiveAuthentication no`<br>`X11Forwarding no`<br>`AllowUsers <USER>` | Kiểm tra nội dung tệp bằng lệnh `cat /etc/ssh/sshd_config.d/00-hardening.conf`. |
| **2.6** | Kiểm tra cú pháp cấu hình SSH trước khi áp dụng | `sudo sshd -t` | Câu lệnh không trả về bất kỳ thông báo lỗi nào. |
| **2.7** | Khởi động lại dịch vụ SSH để áp dụng cấu hình mới | **Ubuntu 24.04:** `sudo systemctl restart ssh`<br>**Ubuntu 22.04 / Debian 12:** `sudo systemctl restart sshd` | Chạy lệnh `systemctl is-active ssh` (hoặc `sshd`) trả về trạng thái `active`. *(Lưu ý: Các phiên SSH đang mở vẫn sẽ được giữ nguyên).* |
| **2.8** | ✅ **Chốt kiểm tra bắt buộc số 2:** Xác nhận cấu hình chặn `root` đã có hiệu lực | Mở thêm một cửa sổ Terminal trên máy cá nhân: thử đăng nhập lại bằng `ssh <USER>@<IP>` và thử đăng nhập bằng `ssh root@<IP>`. | Tài khoản `<USER>` vào được bình thường; tài khoản `root` bị từ chối ngay lập tức với thông báo `Permission denied`. |
| **2.9** | Khóa tài khoản `root` để vô hiệu hóa hoàn toàn khả năng đăng nhập cục bộ bằng mật khẩu *(Khuyến nghị)* | `sudo passwd -l root` | Tài khoản `root` không còn khả năng xác thực bằng mật khẩu. |

<p align="center">
    <img width="1902" height="1019" alt="image" src="https://github.com/user-attachments/assets/659eab43-3b84-442d-9074-6d456a2f784a" />
</p>

**Lưu ý quan trọng theo từng bản phân phối (Distro):**
* **Đối với Ubuntu 24.04:** Dịch vụ SSH sử dụng cơ chế *socket activation* và tên dịch vụ là `ssh` (không phải `sshd`). Ngoài ra, các bản cài đặt đám mây (Cloud Image) của Ubuntu thường có sẵn tệp cấu hình phụ trong thư mục `/etc/ssh/sshd_config.d/` (ví dụ `50-cloud-init.conf`) bật sẵn xác thực mật khẩu. Việc đặt tên tệp là `00-hardening.conf` giúp các quy tắc bảo mật của chúng ta được đọc đầu tiên và được ưu tiên áp dụng (do OpenSSH áp dụng giá trị của chỉ thị xuất hiện đầu tiên).
* **Đối với Ubuntu 22.04 và Debian 12:** Tên dịch vụ quản lý bởi systemd là `sshd` (chưa sử dụng cơ chế socket activation). Cơ chế thư mục cấu hình phụ `sshd_config.d/` hoạt động tương tự như Ubuntu 24.04.

**Hướng xử lý khẩn cấp nếu tự khóa kết nối SSH:** Truy cập vào giao diện Web Console đã mở ở **Bước 0.1**, chỉnh sửa lại tệp `/etc/ssh/sshd_config.d/00-hardening.conf` hoặc kiểm tra quyền thư mục `.ssh`, sau đó khởi động lại dịch vụ SSH.

---

### PHASE 3 — TƯỜNG LỬA CẤP MÁY CHỦ (UFW Firewall) — Lớp phòng vệ thứ hai

**Mục đích:** Việc chỉ cho phép thiết bị trong mạng nội bộ công ty (PC, laptop, điện thoại dùng mạng của công ty) truy cập VPS là **đã được hạ tầng đảm bảo sẵn** (server Proxmox nội bộ, VPS không có IP công khai) — đây là **lớp phòng vệ thứ nhất, không cần chúng ta thiết lập**. Giai đoạn này thiết lập **lớp phòng vệ thứ hai ngay trên máy chủ** bằng UFW, theo chính sách: **từ chối mọi kết nối chiều đến, chỉ cho phép đúng dải mạng nội bộ công ty, và cho phép toàn bộ kết nối chiều đi ra Internet** (phục vụ cập nhật phần mềm, Tailscale, Cloudflare Tunnel). Lớp UFW đảm bảo rằng ngay cả khi lớp mạng phát sinh thay đổi hoặc sơ hở (cấu hình Proxmox được chỉnh lại, mở thêm phân đoạn mạng, thiết bị mạng bị xâm nhập), máy chủ vẫn tự duy trì chính sách chặn đúng và không phụ thuộc vào bất kỳ lớp nào khác.

| Bước | Nội dung công việc | Lệnh thực thi | Cách kiểm tra & Kết quả kỳ vọng |
| :--- | :--- | :--- | :--- |
| **3.1** | Chặn mặc định toàn bộ kết nối chiều đến | `sudo ufw default deny incoming` | Hiển thị thông báo cập nhật chính sách incoming thành công. |
| **3.2** | Cho phép mặc định toàn bộ kết nối chiều đi ra Internet | `sudo ufw default allow outgoing` | Hiển thị thông báo cập nhật chính sách outgoing thành công. |
| **3.3** | Cho phép **toàn bộ lưu lượng đến từ dải mạng nội bộ công ty** — bao gồm cổng SSH (22) và mọi cổng ứng dụng phục vụ phát triển | `sudo ufw allow from 192.168.1.0/24` *(xác nhận dải mạng chính xác với bộ phận mạng trước khi chạy; nếu công ty có nhiều phân đoạn thì liệt kê từng dải bằng nhiều lệnh `allow from` — **tuyệt đối không dùng `anywhere`**)* | Quy tắc cho phép dải mạng công ty được ghi nhận. |
| **3.4** | Cho phép lưu lượng qua giao diện Tailscale *(chạy khi triển khai Phase 5)* | `sudo ufw allow in on tailscale0` | Quy tắc cho phép giao diện `tailscale0` được ghi nhận. |
| **3.5** | **Không mở bất kỳ cổng Web inbound nào** — website khách hàng được công khai ra Internet bằng Cloudflare Tunnel (Phase 7), chỉ dùng kết nối chiều đi | *(Không có lệnh — chủ động bỏ qua bước này)* | `ufw status` không hiển thị quy tắc 80/443. |
| **3.6** | Kích hoạt tường lửa UFW | `sudo ufw --force enable` | Thông báo `Firewall is active and enabled on system startup`. |
| **3.7** | ✅ Kiểm tra UFW đang hoạt động và quy tắc đúng | `sudo ufw status verbose`, đồng thời từ thiết bị trong mạng công ty chạy `ssh <USER>@<IP>` | `Status: active` với duy nhất các quy tắc của dải mạng công ty; SSH từ máy công ty hoạt động bình thường. |



<p align="center">
    <img width="1200" height="630" alt="image" src="https://github.com/user-attachments/assets/82fcc25a-4008-426e-9482-b078be8ffae4" />
    <img width="983" height="739" alt="image" src="https://github.com/user-attachments/assets/951fa2b7-fbaf-4fd2-a8d5-fdeb3e08c4b4" />
</p>

**Lưu ý kiến trúc bảo mật (đặc thù VPS nội bộ):**
* **Kiến trúc hai lớp phòng vệ:** Lớp 1 = hạ tầng mạng công ty (chặn sẵn truy cập từ bên ngoài, do công ty thiết lập sẵn) + Lớp 2 = UFW trên máy chủ (giai đoạn này). Hai lớp bổ trợ cho nhau — **không được tắt hoặc nới lỏng UFW** với lý do "lớp mạng đã chặn rồi", vì bất kỳ thay đổi nào ở lớp mạng sẽ để máy chủ trơ trụi.
* **Ranh giới tin cậy là mạng nội bộ công ty:** một thiết bị trong mạng công ty bị xâm nhập sẽ có khả năng tiếp cận VPS. Vì vậy các lớp bảo vệ SSH (chỉ dùng key, `AllowUsers`) và Fail2ban ở Phase 2 & 4 **vẫn là bắt buộc** — không được bỏ qua với lý do "máy nằm trong mạng nội bộ".
* **Không nới rộng dải IP** trong Bước 3.3 vượt quá dải mạng thực tế của công ty; đặc biệt **không thêm** dải Wi-Fi khách (guest Wi-Fi) vào danh sách cho phép. Khi công ty mở rộng thêm phân đoạn mạng, đánh giá lại việc bổ sung — mỗi dải thêm vào là một bề mặt mở thêm.
* **Biến thể khắt khe hơn** (nếu muốn giới hạn cả cổng): thay Bước 3.3 bằng `sudo ufw allow from 192.168.1.0/24 to any port 22,3000,8080 proto tcp` (liệt kê đúng các cổng đang dùng) — kiểm soát chặt hơn, đổi lại phải cập nhật quy tắc mỗi khi thêm dịch vụ mới.
* ⚠️ **Cảnh báo rủi ro cao nhất:** Quên chạy **Bước 3.3** trước **Bước 3.6** sẽ làm mất toàn bộ kết nối từ mạng công ty ngay lập tức. Giữ phiên `root` đang mở sẵn và xác nhận với quản trị viên server Proxmox về việc truy cập **console (noVNC)** trong Proxmox Web UI làm đường lui.
* **Đối với VPS chạy trên LXC (container):** Tường lửa UFW/nftables chỉ hoạt động khi container được cấp quyền (capability) quản lý netfilter. Ngay sau **Bước 3.6**, nếu lệnh `ufw --force enable` báo lỗi liên quan đến nftables/iptables hoặc `ufw status` không phản hồi đúng, hãy **tạm dừng quy trình** và yêu cầu quản trị viên server Proxmox cấp quyền tương ứng cho container — tuyệt đối không bỏ qua bước này để tiếp tục.

---

### PHASE 4 — CƠ CHẾ PHÒNG VỆ & CẬP NHẬT TỰ ĐỘNG (Automated Protection)

**Mục đích:** Giúp máy chủ có khả năng tự động phát hiện, khóa địa chỉ IP của kẻ tấn công dò mật khẩu và tự động cài đặt các bản vá lỗ hổng bảo mật mới nhất.

#### 4A. Thiết lập Fail2ban (Tự động chặn IP tấn công Brute-force SSH)

| Bước | Nội dung công việc | Lệnh thực thi | Cách kiểm tra & Kết quả kỳ vọng |
| :--- | :--- | :--- | :--- |
| **4A.1** | Tạo tệp cấu hình tùy chỉnh cho Fail2ban kết hợp với tường lửa UFW | Dùng lệnh `sudo tee /etc/fail2ban/jail.local` để ghi nội dung:<br>`[DEFAULT]`<br>`banaction = ufw`<br><br>`[sshd]`<br>`enabled = true`<br>`maxretry = 5` | Kiểm tra tệp `/etc/fail2ban/jail.local` đã tồn tại và đúng nội dung. |
| **4A.2** | Kích hoạt dịch vụ Fail2ban khởi động cùng hệ thống | `sudo systemctl enable --now fail2ban` | Chạy lệnh `systemctl is-active fail2ban` trả về `active`. |
| **4A.3** | ✅ Kiểm tra trạng thái giám sát đối với dịch vụ SSH | `sudo fail2ban-client status sshd` | Hiển thị bảng trạng thái `Status for the jail: sshd` cùng số lượng IP đang bị chặn. |

<p align="center">
<img width="1012" height="578" alt="image" src="https://github.com/user-attachments/assets/3338fefa-80c7-49dd-9bf4-bdda1be6f646" />
<img width="1460" height="489" alt="image" src="https://github.com/user-attachments/assets/2b3c4034-2145-4bbf-8363-2726a01b8249" />
</p>

**Lưu ý theo từng bản phân phối:**
* Trên cả **Ubuntu 24.04** (mặc định dùng `nftables`) và **Debian 12** (mặc định dùng `iptables-multiport`), việc khai báo tường minh `banaction = ufw` giúp đồng bộ hoàn toàn các lệnh chặn IP của Fail2ban vào danh sách quản lý của tường lửa UFW, giúp việc tra cứu và gỡ chặn dễ dàng hơn.

#### 4B. Thiết lập Unattended-Upgrades (Tự động cài đặt bản vá bảo mật)

| Bước | Nội dung công việc | Lệnh thực thi | Cách kiểm tra & Kết quả kỳ vọng |
| :--- | :--- | :--- | :--- |
| **4B.1** | Kích hoạt chế độ tự động cập nhật (chạy tự động không cần tương tác màn hình) | `echo 'unattended-upgrades unattended-upgrades/enable_auto_updates boolean true' \| sudo debconf-set-selections && sudo dpkg-reconfigure -f noninteractive unattended-upgrades` | Tệp `/etc/apt/apt.conf.d/20auto-upgrades` được tạo tự động với các giá trị bằng `"1"`. |
| **4B.2** | Kiểm tra nguồn cập nhật bảo mật được phép chạy tự động | `grep -A5 'Allowed-Origins' /etc/apt/apt.conf.d/50unattended-upgrades` | Xác nhận có dòng cấu hình chứa `security` (chỉ tự động cập nhật các bản vá bảo mật, không tự ý nâng cấp toàn bộ phần mềm để tránh lỗi tương thích). |

---

### PHASE 5 — THIẾT LẬP MẠNG NỘI BỘ AN TOÀN VỚI TAILSCALE (Private Mesh VPN)

**Mục đích:** Kết nối máy chủ VPS vào mạng riêng ảo mã hóa ngang hàng (Peer-to-Peer VPN). Nhờ đó, quản trị viên có thể truy cập VPS an toàn từ bất kỳ đâu (kể cả khi dùng Wi-Fi công cộng) mà không cần phơi bày các cổng quản trị ra ngoài Internet.

| Bước | Nội dung công việc | Lệnh thực thi | Cách kiểm tra & Kết quả kỳ vọng |
| :--- | :--- | :--- | :--- |
| **5.1** | Cài đặt phần mềm Tailscale từ kho chính thức | `curl -fsSL https://tailscale.com/install.sh \| sh` | Chạy lệnh `tailscale version` hiển thị phiên bản đã cài đặt. |
| **5.2** | Kết nối máy chủ vào mạng Tailnet và bật tính năng Tailscale SSH | **Thủ công:** `sudo tailscale up --ssh` *(mở đường dẫn hiện ra trên trình duyệt để xác thực)*<br>**Tự động (Script):** `sudo tailscale up --ssh --authkey=<TS_AUTHKEY>` | Chạy lệnh `tailscale status` thấy máy chủ hiển thị trạng thái trực tuyến. |
| **5.3** | ✅ Kiểm tra kết nối từ máy tính cá nhân (đã cài đặt và đăng nhập cùng mạng Tailscale) | Trên máy tính cá nhân chạy:<br>`tailscale ping <IP-Tailscale-của-VPS>`<br>và `ssh <USER>@<IP-Tailscale-của-VPS>` | Lệnh ping phản hồi thành công và đăng nhập SSH mượt mà qua địa chỉ IP nội bộ của Tailscale (`100.x.y.z`). |

<p align="center">


<img width="1477" height="711" alt="image" src="https://github.com/user-attachments/assets/deed80a3-18e8-4c1e-afa9-a64534298c57" />
Các câu lệnh cài đặt TailScale

<img width="954" height="463" alt="image" src="https://github.com/user-attachments/assets/2bc40f71-abac-4deb-acc3-63bf989ccd88" />
Truy cập VPS thành công qua Tail Scale trên powershell

</p>

**⚠️ Biến thể LXC — chế độ Userspace Networking (bắt buộc khi container không có thiết bị TUN):**

Khi VPS chạy dưới dạng **LXC container** (vd trên Proxmox), `tailscaled` ở chế độ chuẩn cần thiết bị mạng ảo `/dev/net/tun` để tạo card mạng `tailscale0`. Nếu container chưa được host cấp thiết bị này, dịch vụ sẽ khởi động rồi tự chết với thông báo `CreateTUN("tailscale0") failed; /dev/net/tun does not exist` và `systemctl is-active tailscaled` cho thấy trạng thái `deactivating` lặp lại. Trước tiên kiểm tra bằng lệnh `ls -l /dev/net/tun`. Có hai hướng xử lý:

* **Hướng 1 (ưu tiên nếu có thể làm chủ host):** yêu cầu quản trị viên host Proxmox thêm 2 dòng sau vào mục *Additional config* của container rồi restart container — khi đó chạy chế độ chuẩn như bảng trên:
  ```
  lxc.cgroup2.devices.allow: c 10:200 rwm
  lxc.mount.entry: /dev/net/tun dev/net/tun none bind,optional,create=file
  ```
* **Hướng 2 (không cần can thiệp host — đã áp dụng thành công trên VPS46):** chạy `tailscaled` ở **chế độ Userspace Networking**, dùng "TUN giả" ở tầng tiến trình nên không cần `/dev/net/tun`. Ghi các cờ khởi động vào file cấu hình dịch vụ rồi khởi động lại:
  ```bash
  sudo sed -i 's/^FLAGS=.*/FLAGS="--tun=userspace-networking --socks5-server=localhost:1055 --outbound-http-proxy-listen=localhost:1055"/' /etc/default/tailscaled
  sudo systemctl reset-failed tailscaled   # bắt buộc nếu dịch vụ đã bị chặn sau nhiều lần khởi động liên tiếp
  sudo systemctl restart tailscaled
  ```
  Sau đó tiếp tục **Bước 5.2 và 5.3** như bình thường. **Hạn chế của chế độ này:** không tạo ra giao diện `tailscale0` (do đó **không chạy Bước 3.4** của Phase 3), máy chủ không làm được *exit node*, và các ứng dụng khác trên máy chủ muốn giao tiếp ra Tailnet phải định tuyến qua proxy SOCKS5 tại `localhost:1055`. Với mục đích truy cập quản trị từ xa (SSH + Tailscale SSH), chế độ này đáp ứng đầy đủ.

### PHASE 6 — CÀI ĐẶT CÔNG CỤ LÀM VIỆC & PHÁT TRIỂN (Dev Tools: tmux & Claude Code)

**Mục đích:** Thiết lập môi trường làm việc bền bỉ trên máy chủ (giữ nguyên phiên làm việc ngay cả khi rớt mạng giữa chừng với `tmux`) và tích hợp sẵn công cụ hỗ trợ lập trình AI trên dòng lệnh (`Claude Code`).

| Bước | Nội dung công việc | Lệnh thực thi | Cách kiểm tra & Kết quả kỳ vọng |
| :--- | :--- | :--- | :--- |
| **6.1** | Cài đặt trình quản lý phiên làm việc `tmux` | `sudo apt install -y tmux` | Chạy lệnh `tmux -V` hiển thị phiên bản. |
| **6.2** | Tạo tệp cấu hình `tmux` tối ưu cho thao tác cuộn chuột và lưu lịch sử dài | Ghi vào tệp `~/.tmux.conf` hai dòng cấu hình:<br>`set -g mouse on`<br>`set -g history-limit 50000` | Chạy thử `tmux new -s main` để mở phiên làm việc mới thành công. |
| **6.3** | Cài đặt công cụ Claude Code bằng trình cài đặt gốc (không phụ thuộc Node.js) | `curl -fsSL https://claude.ai/install.sh \| bash` | Chạy lệnh `claude --version` trả về thông tin phiên bản. |
| **6.4** | Đăng nhập và xác thực tài khoản Claude Code *(thao tác tương tác trực tiếp)* | Chạy lệnh `claude` và làm theo hướng dẫn trên màn hình để liên kết tài khoản Anthropic. | Khởi động lại lệnh `claude` vào thẳng giao diện làm việc mà không yêu cầu đăng nhập lại. |

**Gợi ý sử dụng phần mềm phía máy trạm (Client):** Nếu bạn sử dụng ứng dụng **Termius** trên máy tính hoặc điện thoại để quản lý máy chủ, hãy lưu cấu hình kết nối bằng **SSH Key kết hợp với địa chỉ IP của Tailscale** thay vì IP công khai để đạt độ bảo mật và ổn định cao nhất.

**Lưu ý về dung lượng lưu trữ:** Cả hai công cụ ở giai đoạn này đều chỉ chiếm một dung lượng ổ đĩa rất nhỏ (chưa tới $500\text{ MB}$), nên không cần chuẩn bị thêm không gian lưu trữ riêng.

---

### PHASE 7 — CÔNG KHAI DỊCH VỤ WEB AN TOÀN (Cloudflare Tunnel)

**Mục đích:** Đưa các ứng dụng Web hoặc API trên máy chủ ra mạng Internet công cộng **mà không cần mở bất kỳ cổng kết nối chiều đến (inbound) nào trên tường lửa**. Toàn bộ lưu lượng truy cập sẽ đi qua hạ tầng chống DDoS và cấp phát chứng chỉ SSL/TLS tự động của Cloudflare, trong khi ứng dụng trên VPS chỉ cần chạy cục bộ tại địa chỉ `127.0.0.1`.

| Bước | Nội dung công việc | Lệnh thực thi | Cách kiểm tra & Kết quả kỳ vọng |
| :--- | :--- | :--- | :--- |
| **7.1** | Thêm kho lưu trữ chính thức của Cloudflare và cài đặt gói `cloudflared` | `curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg \| sudo gpg --dearmor -o /usr/share/keyrings/cloudflare-main.gpg && echo 'deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared any main' \| sudo tee /etc/apt/sources.list.d/cloudflared.list && sudo apt update && sudo apt install -y cloudflared` | Chạy lệnh `cloudflared --version` hiển thị thông tin phiên bản. |
| **7.2** | Khởi tạo đường hầm (Tunnel) kết nối với tài khoản Cloudflare | **Cách 1 (Khuyên dùng cho tự động hóa):** Tạo Tunnel trên bảng điều khiển *Cloudflare Zero Trust*, lấy mã `CF_TUNNEL_TOKEN`.<br>**Cách 2 (Thủ công CLI):** Chạy `cloudflared tunnel login` và `cloudflared tunnel create <TUNNEL-NAME>`. | Đường hầm mới xuất hiện trong danh sách quản lý của Cloudflare. |
| **7.3** | Cấu hình điều hướng tên miền (Ingress Routing) về cổng ứng dụng nội bộ | Thiết lập trỏ tên miền `<HOSTNAME>` về địa chỉ nội bộ `http://127.0.0.1:<PORT>` (trên giao diện Zero Trust Dashboard hoặc qua tệp `/etc/cloudflared/config.yml` kèm lệnh `cloudflared tunnel route dns`). | Bản ghi DNS CNAME của tên miền được tạo tự động trên Cloudflare. |
| **7.4** | Cài đặt `cloudflared` chạy ngầm như một dịch vụ hệ thống (systemd service) | **Nếu dùng Token:** `sudo cloudflared service install <CF_TUNNEL_TOKEN>`<br>**Nếu dùng tệp cấu hình:** `sudo cloudflared service install && sudo systemctl enable --now cloudflared` | Chạy lệnh `systemctl is-active cloudflared` trả về `active`, trạng thái Tunnel trên bảng điều khiển báo `HEALTHY / Connected`. |
| **7.5** | ✅ Kiểm tra truy cập thực tế từ bên ngoài | Mở trình duyệt web truy cập vào địa chỉ `https://<HOSTNAME>`. | Trang web hoặc API phản hồi chính xác và có sẵn chứng chỉ bảo mật HTTPS của Cloudflare. |


<p align="center">
    <img width="1280" height="647" alt="image" src="https://github.com/user-attachments/assets/8fafc8a4-8778-477f-8917-90942b1774a4" />
</p>

**Lưu ý khi triển khai nhiều website khách hàng trên cùng một VPS:** Mỗi website khách hàng là một mục Ingress riêng trong cấu hình đường hầm — ví dụ `ten-mien-khach-A.com` → `http://127.0.0.1:3000`, `ten-mien-khach-B.com` → `http://127.0.0.1:8080` — hoặc tách thành đường hầm riêng cho từng khách hàng nếu yêu cầu cách ly cao hơn. Mỗi dự án khách hàng cần được ghi nhận vào hồ sơ máy chủ theo bảng đối chiếu: **tên miền ↔ cổng nội bộ ↔ thư mục mã nguồn ↔ cơ sở dữ liệu**, để khi bảo trì, xử lý sự cố hoặc sao lưu không nhầm lẫn giữa các dự án.

---

### PHASE 8 — SAO LƯU & PHỤC HỒI DỮ LIỆU TỰ ĐỘNG (Backup & Recovery)

**Mục đích:** Đảm bảo dữ liệu quan trọng trên máy chủ luôn có bản sao lưu mã hóa được lưu trữ tại một dịch vụ đám mây độc lập bên ngoài (Off-site Backup) và **đã được kiểm chứng khả năng khôi phục thực tế**.

**Nguyên tắc lựa chọn phạm vi sao lưu:** Vì dung lượng lưu trữ luôn có hạn và chi phí lưu trữ đám mây cần được kiểm soát, bản sao lưu **không chụp toàn bộ hệ thống** mà chỉ bao gồm (i) các thư mục dữ liệu ứng dụng trọng yếu — với đặc thù VPS này là **thư mục mã nguồn và dữ liệu của từng dự án khách hàng, cơ sở dữ liệu của từng website, cùng các tệp cấu hình ứng dụng** — và (ii) danh sách các gói phần mềm đã cài đặt (đóng vai trò như "công thức dựng lại hệ thống" khi cần khởi tạo lại VPS mới).

| Bước | Nội dung công việc | Lệnh thực thi / Thao tác | Cách kiểm tra & Kết quả kỳ vọng |
| :--- | :--- | :--- | :--- |
| **8.1** | Cài đặt công cụ sao lưu mã hóa `restic` | `sudo apt install -y restic` | Chạy lệnh `restic version` hiển thị phiên bản. |
| **8.2** | Lưu trữ thông tin kết nối kho lưu trữ đám mây (AWS S3 / Backblaze B2 / Cloudflare R2) và khởi tạo kho (Repository) | Tạo tệp `/etc/restic.env` (phân quyền `chmod 600`, chủ sở hữu `root`) chứa biến môi trường `RESTIC_REPOSITORY` và `RESTIC_PASSWORD`. Chạy lệnh khởi tạo: `source /etc/restic.env && restic init` | Chạy lệnh `restic snapshots` kết nối thành công tới kho lưu trữ mà không báo lỗi. |
| **8.3** | Thiết lập kịch bản sao lưu tự động hằng ngày và xoay vòng giữ lại bản sao trong 30 ngày | Tạo script `/usr/local/bin/backup.sh` sao lưu các thư mục trọng yếu đã được xác định (ví dụ `/var/www`, `/etc` và dữ liệu ứng dụng) kèm dịch vụ `restic-backup.service` và bộ hẹn giờ `restic-backup.timer` trong systemd. | Chạy lệnh `systemctl list-timers \| grep restic` thấy lịch chạy tự động đã được kích hoạt. |
| **8.4** | Ghi lại "công thức tái dựng hệ thống" để có thể khôi phục hoàn toàn phần mềm nếu phải khởi tạo lại VPS mới | Chạy lệnh `dpkg --get-selections > /etc/backup-package-list.txt` và đưa tệp này vào phạm vi sao lưu. | Tệp danh sách gói phần mềm được tạo đầy đủ và nằm trong nhóm dữ liệu được sao lưu. |
| **8.5** | ✅ **Kiểm tra khôi phục dữ liệu thực tế** *(Nguyên tắc sống còn: Bản sao lưu chưa từng được khôi phục thử thì chưa được coi là bản sao lưu hợp lệ)* | Chạy lệnh khôi phục thử bản mới nhất ra thư mục tạm:<br>`source /etc/restic.env && restic restore latest --target /tmp/restore-test` | Kiểm tra thư mục `/tmp/restore-test` thấy đầy đủ dữ liệu nguyên vẹn, sau đó xóa thư mục tạm. |

**Quy tắc bảo mật thông tin sao lưu:** Tuyệt đối không đưa mật khẩu `RESTIC_PASSWORD` hay khóa truy cập S3/B2 trực tiếp vào câu lệnh (vì sẽ bị lộ trong lịch sử lệnh và danh sách tiến trình) hoặc đẩy lên kho mã nguồn Git. Mọi thông tin nhạy cảm phải nằm trong tệp `/etc/restic.env` với quyền truy cập `600`.

---

### PHASE 9 — GIÁM SÁT NHẬT KÝ & KIỂM TOÁN BẢO MẬT (Monitoring & Auditing)

**Mục đích:** Theo dõi các dấu hiệu đăng nhập bất thường, theo dõi sức khỏe tài nguyên hệ thống (đặc biệt là dung lượng ổ đĩa) và đánh giá định kỳ điểm số an toàn của hệ thống.

| Bước | Nội dung công việc | Lệnh thực thi | Cách kiểm tra & Kết quả kỳ vọng |
| :--- | :--- | :--- | :--- |
| **9.1** | Kiểm tra nhật ký xác thực SSH trong 24 giờ gần nhất | `sudo journalctl -u ssh -u sshd --since "24 hours ago" --no-pager \| tail -n 50` | Đọc và trích xuất được nhật ký truy cập rõ ràng. |
| **9.2** | Thiết lập cảnh báo tự động khi dung lượng ổ đĩa vượt ngưỡng $80\%$ *(Cơ chế phòng thủ quan trọng cho các máy chủ có dung lượng lưu trữ hạn chế)* | Tạo script `/usr/local/bin/disk-alert.sh` kiểm tra định kỳ bằng `df --output=pcent /` và gửi thông báo cảnh báo (qua Telegram/Email) khi vượt quá ngưỡng. Kết hợp dịch vụ `disk-alert.service` và bộ hẹn giờ `disk-alert.timer` chạy mỗi 6 giờ. | Chạy lệnh `systemctl list-timers \| grep disk-alert` xác nhận lịch kiểm tra định kỳ đã được kích hoạt. |
| **9.3** | Cài đặt công cụ kiểm toán bảo mật hệ thống `Lynis` | `sudo apt install -y lynis` | Chạy lệnh `lynis --version` hiển thị phiên bản. |
| **9.4** | Chạy quét đánh giá bảo mật tổng thể toàn hệ thống | `sudo lynis audit system --quick` | Xuất ra báo cáo chi tiết kèm chỉ số *Hardening Index* để theo dõi. |
| **9.5** | Thiết lập giám sát thời gian hoạt động (Uptime Monitoring) từ bên ngoài *(Khuyến nghị)* | Đăng ký giám sát đường dẫn URL công khai trên các nền tảng như UptimeRobot, Better Stack hoặc Cloudflare Notification. | Nhận được thông báo cảnh báo ngay lập tức qua Email/Telegram khi dịch vụ ngừng phản hồi. |

---

## 3. Danh mục nghiệm thu tổng thể (Acceptance Checklist)

Sau khi hoàn tất các giai đoạn theo cấp độ bảo mật mục tiêu (L1, L2 hoặc L3), quản trị viên chạy lần lượt các lệnh kiểm tra dưới đây. **Hệ thống chỉ được nghiệm thu đưa vào sử dụng khi toàn bộ các mục đều khớp với kết quả kỳ vọng.**

| Mã | Hạng mục kiểm tra | Câu lệnh kiểm tra | Kết quả kỳ vọng đạt chuẩn |
| :--- | :--- | :--- | :--- |
| **A1** | Các gói phần mềm hệ thống đã cập nhật mới nhất | `apt list --upgradable` | Danh sách trống (không còn gói nào chờ cập nhật). |
| **A2** | Chặn đăng nhập trực tiếp bằng tài khoản `root` | `sudo sshd -T \| grep -i permitrootlogin` | `permitrootlogin no` |
| **A3** | Vô hiệu hóa hoàn toàn đăng nhập SSH bằng mật khẩu | `sudo sshd -T \| grep -i ^passwordauthentication` | `passwordauthentication no` |
| **A4** | Giới hạn danh sách tài khoản được phép SSH | `sudo sshd -T \| grep -i allowusers` | `allowusers <USER>` |
| **A5** | Tường lửa UFW đang hoạt động đúng quy tắc | `sudo ufw status` | `Status: active` và chỉ hiển thị các cổng được phép mở. |
| **A6** | Dịch vụ chống dò mật khẩu Fail2ban đang bảo vệ SSH | `sudo fail2ban-client status sshd` | Hiển thị bảng trạng thái hoạt động của jail `sshd`. |
| **A7** | Cơ chế tự động cài đặt bản vá bảo mật đã bật | `cat /etc/apt/apt.conf.d/20auto-upgrades` | Chứa dòng `APT::Periodic::Unattended-Upgrade "1";` |
| **A8** | Xác nhận trạng thái Swap theo môi trường *(môi trường LXC/ZFS không hỗ trợ swap file)* | `swapon --show` | Kết quả trống là **đúng như kỳ vọng**, trừ khi nhà cung cấp đã cấp sẵn swap từ host. |
| **A9** | Vùng nhớ tạm `/tmp` đã được giới hạn $1\text{ GB}$ | `df -h /tmp` | Hiển thị tmpfs $1\text{ GB}$ thay vì chiếm dung lượng ổ cứng vật lý. |
| **A10** | Dung lượng nhật ký systemd đã được ràng buộc | `journalctl --disk-usage` | Dung lượng chiếm dụng ở mức thấp (không vượt quá $500\text{ MB}$ sau một thời gian vận hành). |
| **A11** | Kết nối mạng riêng Tailscale hoạt động *(Cấp độ L2+)* | `tailscale status` | Trạng thái trực tuyến (Online) và đã bật tính năng SSH. |
| **A12** | Đường hầm Cloudflare Tunnel thông suốt *(Cấp độ L3)* | `systemctl is-active cloudflared` | Trả về trạng thái `active` (website truy cập được qua HTTPS). |
| **A13** | Lịch trình sao lưu tự động đã kích hoạt *(Cấp độ L3)* | `systemctl list-timers \| grep restic` | Hiển thị thời gian chạy lần kế tiếp của `restic-backup`. |
| **A14** | Kiểm chứng khôi phục bản sao lưu *(Cấp độ L3)* | Thực hiện theo hướng dẫn tại **Bước 8.5** | Dữ liệu khôi phục ra thư mục tạm đầy đủ, không lỗi. |
| **A15** | Rà soát các tiến trình đang mở cổng lắng nghe mạng | `sudo ss -tulpn` | Chỉ xuất hiện các dịch vụ hợp lệ đã biết (`sshd`, `tailscaled`, hoặc ứng dụng nội bộ `127.0.0.1`). |
| **A16** | Giám sát dung lượng ổ đĩa định kỳ đã kích hoạt *(Cấp độ L3)* | `systemctl list-timers \| grep disk-alert` | Hiển thị lịch kiểm tra định kỳ mỗi 6 giờ. |

---

## 4. Quy trình vận hành và bảo trì định kỳ (Runbook)

### 4.1. Công việc bảo trì hằng tuần (Thời lượng: ~5 phút)
```bash
# 1. Cập nhật các gói phần mềm thông thường (bổ sung cho unattended-upgrades vốn chỉ vá lỗi bảo mật)
sudo apt update && sudo apt full-upgrade -y

# 2. Thống kê số lần bị tấn công dò mật khẩu thất bại trong 7 ngày qua
sudo journalctl -u ssh -u sshd --since "7 days ago" --no-pager | grep -i "failed" | wc -l

# 3. Kiểm tra danh sách các địa chỉ IP đã bị Fail2ban khóa
sudo fail2ban-client status sshd

# 4. Kiểm tra dung lượng ổ đĩa và dung lượng bộ nhớ tạm
df -h / /tmp
```

### 4.2. Công việc bảo trì hằng tháng (Thời lượng: ~30 phút)

<p align="center">
    <img width="1200" height="630" alt="image" src="https://github.com/user-attachments/assets/ec184213-3076-40e7-90b3-6d293dba79ec" />
</p>

### 4.3. Quy trình ứng phó sự cố khi nghi ngờ máy chủ bị xâm nhập

<p align="center">
    <img width="1200" height="630" alt="image" src="https://github.com/user-attachments/assets/42340ba4-3fdd-4ca7-9624-08e13e8b3c89" />
</p>


### 4.4. Quy trình ứng phó khi dung lượng ổ đĩa đầy dần bất thường

<p align="center">
    <img width="1200" height="630" alt="image" src="https://github.com/user-attachments/assets/90d98a24-b65c-4d88-8acb-aba43da5441e" />

</p>

## 5. Hướng dẫn xử lý các sự cố thường gặp (Troubleshooting)

| Triệu chứng gặp phải | Nguyên nhân phổ biến | Cách khắc phục chi tiết |
| :--- | :--- | :--- |
| **Mất hoàn toàn kết nối SSH ngay sau khi bật tường lửa UFW** | Quên chạy **Bước 3.3** (`ufw allow from <dải-mạng-công-ty>`) trước khi kích hoạt tường lửa. | Đăng nhập vào máy chủ qua **console (noVNC) trong Proxmox Web UI** (xin quản trị viên server nội bộ hỗ trợ), chạy lệnh:<br>`sudo ufw allow from 192.168.1.0/24 && sudo ufw reload` |
| **Báo lỗi `Permission denied (publickey)` sau khi tắt xác thực mật khẩu** | Khóa SSH chưa được chép đúng vào tệp `authorized_keys`, sai tên người dùng, hoặc phân quyền thư mục `.ssh` bị sai. | Vào **Web Console**, kiểm tra lại đường dẫn `/home/<USER>/.ssh/authorized_keys`, đảm bảo quyền thư mục `.ssh` là `700`, tệp khóa là `600` và chủ sở hữu thuộc về `<USER>` (`chown -R <USER>:<USER> /home/<USER>/.ssh`). |
| **Đổi cổng SSH trong `sshd_config` nhưng cổng mới không hoạt động** | Trên Ubuntu 24.04, cổng lắng nghe SSH được quản lý bởi `ssh.socket` của systemd chứ không chỉ đọc từ `sshd_config`. | Chỉnh sửa cấu hình cổng trong systemd bằng lệnh `sudo systemctl edit ssh.socket` (khai báo `ListenStream=`), sau đó chạy `sudo systemctl daemon-reload && sudo systemctl restart ssh.socket`. |
| **Không thể kết nối SSH từ mạng hiện tại dù cấu hình đúng** | Địa chỉ IP của bạn đã bị Fail2ban khóa tạm thời do đăng nhập sai quá số lần quy định trước đó. | Đăng nhập qua **Web Console** (hoặc qua mạng 4G/Tailscale khác IP) và chạy lệnh gỡ khóa:<br>`sudo fail2ban-client set sshd unbanip <IP-của-bạn>` |
| **Lệnh `sshd -t` báo lỗi cú pháp sau khi chỉnh sửa cấu hình** | Gõ sai tên từ khóa cấu hình hoặc thiếu tham số trong tệp `00-hardening.conf`. | Mở lại tệp `/etc/ssh/sshd_config.d/00-hardening.conf` để sửa lỗi chính tả. Tuyệt đối không khởi động lại dịch vụ SSH cho đến khi chạy `sudo sshd -t` không còn báo lỗi. |
| **Không kích hoạt được Tailscale trong script tự động** | Thiếu khóa xác thực `TS_AUTHKEY` hoặc khóa đã hết hạn / đã sử dụng. | Truy cập trang quản trị Tailscale Admin Console, tạo một *Auth Key* mới và chạy lại lệnh: `sudo tailscale up --ssh --authkey=<KEY_MỚI>`. |
| **Dịch vụ `tailscaled` khởi động rồi tự chết (trạng thái `deactivating`), log có dòng `/dev/net/tun does not exist`** | Container LXC chưa được host cấp thiết bị TUN — xem chi tiết ở phần *Biến thể LXC* của Phase 5. | Xin quản trị viên host cấp TUN cho container (2 dòng config Proxmox), hoặc chuyển sang chế độ Userspace Networking bằng cách ghi FLAGS `--tun=userspace-networking` vào `/etc/default/tailscaled`, sau đó `sudo systemctl reset-failed tailscaled && sudo systemctl restart tailscaled`. |
| **Cloudflare Tunnel báo trạng thái kết nối thành công nhưng truy cập web báo lỗi 502 / 404** | Khai báo sai tên miền (Hostname) hoặc sai cổng ứng dụng nội bộ (`127.0.0.1:<PORT>`) trong cấu hình Ingress. | Kiểm tra ứng dụng nội bộ đã chạy trên cổng tương ứng chưa (`curl -I http://127.0.0.1:<PORT>`) và cập nhật lại cấu hình điều hướng trong Cloudflare Tunnel. |
| **Lệnh `swapon` báo lỗi "it appears to have holes" khi tạo swap file** | Hệ thống tệp gốc là ZFS (phổ biến trên VPS dạng LXC) — kernel không thể xác minh các khối nền của swap file tạo trên ZFS, do đó từ chối kích hoạt. | Đây là **hạn chế của môi trường, không phải cấu hình sai**. Không cố khắc phục thêm: xóa file swap (`rm /swapfile`), gỡ dòng liên quan khỏi `/etc/fstab` và bỏ qua bước tạo swap — hệ thống vận hành bình thường với RAM thực tế. Nếu bắt buộc cần Swap, yêu cầu nhà cung cấp cấp sẵn swap partition từ phía host. |
| **Dung lượng ổ đĩa tăng vọt bất thường sau vài tuần vận hành** | Nhật ký hệ thống, bộ nhớ tạm hoặc tệp ứng dụng không được ràng buộc dung lượng và không có cơ chế dọn dẹp định kỳ. | Thực hiện quy trình tại **Mục 4.4**: xác định thủ phạm bằng lệnh `du`, dọn dẹp, sau đó kiểm tra lại toàn bộ các ràng buộc dung lượng tại Bước 1.6, 1.7, 1.8 và cơ chế cảnh báo tại Bước 9.2. |

---

## 6. Bảng ánh xạ giai đoạn sang Script tự động hóa (Phase-to-Script Mapping)

Bảng dưới đây quy định cấu trúc tham số dòng lệnh (CLI Flags) và các biến môi trường cần thiết để chuyển hóa toàn bộ quy trình trong tài liệu này thành một kịch bản tự động hóa duy nhất (`vps-harden.sh`):

| Giai đoạn | Tham số chạy Script (Flag) | Khả năng tự động hóa 100% | Các biến môi trường cần truyền vào Script |
| :--- | :--- | :---: | :--- |
| **Phase 0** | `--precheck` *(Kiểm tra & in hướng dẫn)* | Có | *(Không yêu cầu biến đầu vào)* |
| **Phase 1** | `--base` | Có | `HOSTNAME`, `TZ`, `TMP_SIZE` *(mặc định: `1G`)* |
| **Phase 2** | `--ssh` | Có *(trừ bước đặt mật khẩu sudo)* | `ADMIN_USER`, `PUB_KEY` *(chuỗi nội dung khóa công khai)* |
| **Phase 3** | `--firewall` | Có | `LAN_SUBNET` *(mặc định: `192.168.1.0/24` — dải mạng nội bộ công ty, có thể liệt kê nhiều dải)*, `DEV_PORTS` *(tuỳ chọn: danh sách cổng ứng dụng cho phép; bỏ trống = cho phép toàn bộ cổng từ dải mạng)*, `WITH_TAILSCALE` (`yes`/`no`) |
| **Phase 4** | `--protection` | Có | `F2B_MAXRETRY` |
| **Phase 5** | `--tailscale` | Có *(nếu cung cấp `TS_AUTHKEY`)* | `TS_AUTHKEY` |
| **Phase 6** | `--devtools` | Có *(trừ bước đăng nhập tài khoản Claude)* | *(Không yêu cầu biến đầu vào)* |
| **Phase 7** | `--tunnel` | Có *(nếu cung cấp `CF_TUNNEL_TOKEN`)* | `CF_TUNNEL_TOKEN`, `CF_TUNNEL_NAME`, `CF_HOSTNAME`, `ORIGIN_PORT` |
| **Phase 8** | `--backup` | Có | `BACKUP_REPO`, `RESTIC_PASSWORD`, `BACKUP_PATHS` *(mặc định: `/etc` cùng các thư mục dữ liệu ứng dụng, **không bao gồm** toàn bộ hệ thống)* |
| **Phase 9** | `--audit` | Có | `DISK_ALERT_THRESHOLD` *(mặc định: `80` phần trăm)* |
| **Nghiệm thu** | `--verify` | Có | Chạy tự động toàn bộ danh mục kiểm tra tại **Mục 3**. |
| **Tổng thể** | `--all --level <L1\|L2\|L3>` | Có | Thực thi tuần tự từ Phase 1 đến Phase 9 theo cấp độ bảo mật đã chọn. |

---
