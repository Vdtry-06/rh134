# Tổng Hợp: Chương 19 – Lab Tổng Ôn (Comprehensive Review)

> Tài liệu tổng hợp 3 bài Lab tổng hợp kiến thức toàn khóa RH134, mô phỏng tình huống thực tế tại công ty **Quasar Technologies** và **Supernova**. Mỗi lab kết hợp nhiều kỹ năng đã học ở các chương trước — đây là dịp ôn tập tổng thể.

---

## Lab 1: Sửa Lỗi Boot & Bảo Trì Server

> Kết hợp kiến thức: **Chương 12 (Boot process)** + **Chương 3 (Cron user)**

### Tình huống
Máy `serverb` gặp sự cố sau khi maintenance:
1. **Sự cố 1:** Boot bị stuck do lỗi mount partition (script `rhcsa-break1` giả lập).
2. **Sự cố 2:** Default target bị đổi thành `graphical` thay vì `multi-user` (script `rhcsa-break2` giả lập).
3. **Yêu cầu mới:** Lên lịch backup định kỳ thư mục home của user `student`.

### Quy trình xử lý Sự cố 1 — Lỗi mount partition lúc boot

1. Reboot `serverb`, hệ thống tự vào **emergency mode** (vì lỗi mount)
2. Login emergency mode (password `redhat`)
3. Remount root FS sang read-write:
   ```bash
   mount -o remount,rw /
   ```
4. Thử mount tất cả entry còn lại trong fstab để xác định dòng nào lỗi:
   ```bash
   mount -a
   ```
   → Báo lỗi cụ thể (vd mount point không tồn tại, sai UUID...)
5. Sửa `/etc/fstab`: xóa/comment dòng bị lỗi
6. `systemctl daemon-reload`
7. `reboot` → xác nhận boot thành công

> 💡 Đây chính là kỹ năng từ **Chương 12 phần "Repairing Damaged File Systems at Boot Time"**.

### Quy trình xử lý Sự cố 2 — Sai Default Target

1. Sau khi máy reboot lần 2 (do `rhcsa-break2`), kiểm tra target hiện tại:
   ```bash
   systemctl get-default
   ```
2. Đặt lại đúng target (tiết kiệm tài nguyên CPU/RAM — không cần GUI cho server):
   ```bash
   systemctl set-default multi-user.target
   ```
3. `reboot` để xác nhận bền vững

### Yêu cầu 3 — Lên lịch Backup Định kỳ

**Yêu cầu cụ thể:** Chạy script backup **mỗi giờ**, trong khung **19h-21h**, **mỗi ngày trừ Thứ 7 & Chủ Nhật**.

1. Cấp quyền thực thi cho script:
   ```bash
   chmod +x ~/RH134/labs/compreview-review1/backup-home.sh
   ```
2. Mở crontab: `crontab -e`
3. Thêm dòng:
   ```
   0 19-21 * * Mon-Fri /home/student/RH134/labs/compreview-review1/backup-home.sh
   ```
   - `0`: phút 0 (đúng đầu giờ)
   - `19-21`: chạy vào 19h, 20h, 21h (mỗi giờ 1 lần)
   - `Mon-Fri`: chỉ các ngày trong tuần, loại trừ cuối tuần

> 💡 Đây là kỹ năng từ **Chương 3 phần "Scheduling Recurring User Jobs"** — chú ý range giờ và range thứ kết hợp.

4. Reboot lại `serverb`, chờ boot hoàn tất trước khi chấm điểm.

---

## Lab 2: Cấu Hình & Quản Lý Bảo Mật Server

> Kết hợp kiến thức: **SSH key-based auth** + **SELinux mode/Boolean** + **Autofs/NFS** + **Firewalld** + **SELinux Port Labeling**

### Yêu cầu 1 — SSH Key-based Authentication (passwordless)

**Mục tiêu:** `student@serverb` SSH vào `servera` không cần password.

```bash
# Trên serverb, tạo key pair (không đặt passphrase)
ssh-keygen -t rsa    # Enter, Enter, Enter (bỏ trống passphrase)

# Gửi public key sang servera
ssh-copy-id student@servera

# Test
ssh student@servera    # phải vào được không hỏi password
```

### Yêu cầu 2 — Debug bằng cách nới lỏng SELinux

**Mục tiêu:** Kiểm tra quyền thư mục `/user-homes/production5`, rồi tạm chuyển SELinux sang **Permissive** (bền vững).

```bash
ls -ld /user-homes/production5    # kiểm tra permission

# Sửa /etc/selinux/config
SELINUX=permissive

# Reboot để áp dụng
systemctl reboot

# Xác nhận
getenforce    # → Permissive
```
> 💡 Từ **Chương 6 phần "Operating SELinux"**.

### Yêu cầu 3 — Autofs mount NFS Home Directory

**Mục tiêu:** `serverb` tự mount `servera:/user-homes/production5` vào `/localhome/production5` khi cần.

```bash
dnf install autofs nfs-utils

# Cấu hình direct map trong master map hoặc file riêng
echo "/localhome/production5  -rw,sync  servera.lab.example.com:/user-homes/production5" \
    > /etc/auto.production5
echo "/-  /etc/auto.production5" >> /etc/auto.master

systemctl enable --now autofs
```
> 💡 Từ **Chương 15 phần "Automounting Storage Devices"**.

### Yêu cầu 4 — SELinux Boolean cho NFS Home Directory qua SSH

**Vấn đề:** User `production5` login SSH bằng key, nhưng home directory nằm trên NFS mount → SSH/PAM cần quyền đặc biệt để truy cập home dir qua NFS.

1. Trên `servera`, tạo SSH key cho `production5`, gửi public key sang `serverb`
2. Thử login bằng key → **THẤT BẠI** (SELinux chặn NFS home dir)
3. Tìm và bật Boolean đúng trên `serverb`:
   ```bash
   getsebool -a | grep nfs
   setsebool -P use_nfs_home_dirs on
   ```
4. Thử lại → **thành công**

> 💡 Từ **Chương 6 phần "Tuning the SELinux Policy by Adjusting Booleans"** — đúng dạng bài `httpd_enable_homedirs` nhưng áp dụng cho ngữ cảnh NFS home dir qua SSH.

### Yêu cầu 5 — Chặn kết nối từ 1 địa chỉ IP cụ thể

**Mục tiêu:** `serverb` chặn hết traffic từ `servera` (172.25.250.10).

```bash
firewall-cmd --permanent --zone=block --add-source=172.25.250.10/32
firewall-cmd --reload
```
> 💡 Dùng zone `block` (từ chối tất cả trừ traffic liên quan outgoing) — từ **Chương 14 phần "Managing Server Firewalls"**.

### Yêu cầu 6 — Apache lắng nghe port nonstandard (30080) — Kết hợp SELinux + Firewall

**Vấn đề kép giống bài học trong Chương 14:**

1. Restart httpd → thất bại
2. `systemctl status httpd` / `journalctl -xeu httpd` → lỗi bind port
3. Chẩn đoán SELinux:
   ```bash
   sealert -a /var/log/audit/audit.log
   ```
   → Gợi ý dùng `semanage port`
4. Gán label đúng:
   ```bash
   semanage port -a -t http_port_t -p tcp 30080
   systemctl restart httpd
   ```
5. Mở firewall:
   ```bash
   firewall-cmd --permanent --add-port=30080/tcp
   firewall-cmd --reload
   ```
> 💡 Đúng bài học "2 lớp bảo mật độc lập" từ **Chương 14 phần "SELinux Port Labeling"** — cả SELinux lẫn firewall đều cần đúng.

---

## Lab 3: Chạy Container

> Kết hợp kiến thức: **Chương 17 (Podman: image, container, registry)**

### Yêu cầu 1 — Tạo user quản lý container
```bash
useradd podmgr
passwd podmgr    # đặt redhat

# Đăng nhập registry để verify
podman login registry.lab.example.com:5000    # user: student / pass: redhat
```

### ⚠️ Lưu ý quan trọng: Container không bền vững qua session
> Phải **giữ nguyên phiên SSH** đăng nhập là `podmgr` cho tới khi chấm điểm xong — thoát session sẽ làm container biến mất (vì cấu hình container persistent/systemd nằm ngoài phạm vi khóa học này).

### Yêu cầu 2 — Build Image tùy chỉnh

Tạo Containerfile trong `~/http-dev`:
```dockerfile
FROM registry.lab.example.com:5000/rhel10/httpd-24
RUN echo "Welcome to the Supernova containerized webserver." > /var/www/html/index.html
```

Build & push:
```bash
podman build -t http-server:9.0 ~/http-dev
podman tag http-server:9.0 registry.lab.example.com:5000/podmgr/http-server:9.0
podman push registry.lab.example.com:5000/podmgr/http-server:9.0
```
> 💡 Từ **Chương 17 phần "Creating and Managing Container Images"**.

### Yêu cầu 3 — Chạy Container Detached với Port Mapping

```bash
podman run -d --name http-srv01 \
  -p 30000:8080 \
  registry.lab.example.com:5000/podmgr/http-server:9.0
```
> 💡 `-p LOCAL:CONTAINER` — port 30000 trên máy thật map vào port 8080 trong container. Từ **Chương 17 phần "Running Containers with Podman"**.

### Yêu cầu 4 — Mở Firewall cho Port Mới
```bash
firewall-cmd --permanent --add-port=30000/tcp
firewall-cmd --reload
```
> 💡 Kết hợp lại kiến thức **Chương 14** — dù container đã map port đúng, vẫn cần mở firewall để truy cập từ ngoài.

### Yêu cầu 5 — Xác nhận từ máy khác
```bash
curl http://serverb.lab.example.com:30000
# → Welcome to the Supernova containerized webserver.
```

---

## Bảng Tổng Hợp: Kỹ Năng Nào Thuộc Chương Nào?

| Kỹ năng trong Lab | Chương liên quan |
|---|---|
| Sửa lỗi mount `/etc/fstab` lúc boot (emergency mode) | Chương 12 |
| Đổi default systemd target | Chương 12 |
| Cron job user với range giờ + range ngày trong tuần | Chương 3 |
| SSH key-based authentication | (Kỹ năng nền tảng RHCSA — ssh-keygen/ssh-copy-id) |
| Đổi SELinux mode (Enforcing/Permissive) | Chương 6 |
| Autofs mount NFS | Chương 15 |
| SELinux Boolean (`use_nfs_home_dirs`) | Chương 6 |
| Firewalld zone + block source IP | Chương 14 |
| SELinux port labeling (`semanage port`) kết hợp firewall port | Chương 14 |
| Build/push/run container với Podman | Chương 17 |

## Bài học tổng quát từ 3 Lab này

1. ✅ **Sự cố thực tế thường là chuỗi nhiều lớp** — 1 vấn đề (vd service không chạy) có thể do **nhiều nguyên nhân xếp chồng** (SELinux + firewall; hoặc SELinux + Boolean).
2. ✅ Luôn **kiểm tra log/status trước khi đoán mò** — `systemctl status`, `journalctl`, `sealert` là bước đầu tiên khi troubleshoot.
3. ✅ Kỹ năng **boot troubleshooting** (emergency/rescue mode) là nền tảng để xử lý mọi sự cố nghiêm trọng không login được.
4. ✅ Container trong bài lab **không persistent** qua session SSH nếu không cấu hình thêm (systemd service cho container, quản lý bằng Quadlet...) — nằm ngoài phạm vi RHCSA/RH134 cơ bản.
5. ✅ Luôn nhớ 2 lớp bảo mật độc lập cần đồng bộ: **SELinux** (process có được phép không) và **Firewall** (traffic có được vào không) — quên 1 trong 2 là nguyên nhân phổ biến nhất khiến service "không truy cập được" dù cấu hình tưởng như đã đúng.