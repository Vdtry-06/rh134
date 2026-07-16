# Tổng Hợp: Khôi Phục Quyền Superuser & Reset Mật Khẩu Root trên RHEL 10

> Tài liệu tổng hợp Chương 13 – Recovering Superuser Access. Trọng tâm: reset mật khẩu root khi bị quên/khóa, bằng 2 cách — **có rescue media** và **không cần rescue media**.

---

## 1. Bối cảnh & Use Case

**Vấn đề:** Quên mật khẩu root, và **không có** user nào khác có quyền sudo full để tự sửa → cần khởi động vào **rescue mode** (môi trường đặc biệt) để reset mật khẩu.

**2 cách tiếp cận:**

| Cách | Đặc điểm | Khi nào dùng |
|---|---|---|
| **Với rescue media** (khuyến nghị) | Boot từ ISO/CD/USB cài đặt riêng | An toàn hơn, được Red Hat khuyến nghị chính thức |
| **Không cần rescue media** | Dùng tham số kernel `init=/bin/bash` | Nhanh hơn nhưng **rủi ro hơn** — cần cẩn thận |

---

## 2. Cách 1: Reset bằng Rescue Media (Khuyến nghị)

### Yêu cầu
- Cần **media cài đặt** giống đúng phiên bản hệ thống (vd RHEL 10 system → phải dùng **RHEL 10 boot ISO**).
- Máy vật lý: CD-ROM/USB. Máy ảo: file ISO.
- Red Hat cung cấp sẵn: **Red Hat Enterprise Linux 10.0 Boot ISO** (bản cài đặt tối thiểu qua mạng).

### Các bước thực hiện

1. **Reboot máy**, chọn boot từ `boot.iso`
2. Trong menu GRUB2 của rescue media: chọn **Troubleshooting → Rescue a Red Hat Enterprise Linux system**
3. Trong menu rescue: nhấn **`1`** (Continue) → Enter → hệ thống mount bình thường
4. Tại shell prompt, **chroot vào root file system thật:**
   ```bash
   chroot /mnt/sysroot
   ```
5. **Đổi mật khẩu root:**
   ```bash
   passwd root
   ```
6. **Đánh dấu relabel SELinux** (xem giải thích quan trọng bên dưới):
   ```bash
   touch /.autorelabel
   ```
7. **Thoát 2 lần** để reboot:
   ```bash
   exit    # thoát chroot
   exit    # thoát rescue mode, reboot hệ thống
   ```
8. Hệ thống tự **relabel SELinux toàn bộ**, rồi **reboot lại lần nữa** tự động.

---

## 3. Cách 2: Reset KHÔNG cần Rescue Media

> ⚠️ Cách này **rủi ro hơn** — cần thực hiện cẩn thận, đúng từng bước.

### Nguyên lý
Truyền tham số kernel `init=/bin/bash` → hệ thống boot thẳng vào shell `bash`, bỏ qua hoàn toàn systemd — không cần password để vào, vì kernel coi bash chính là tiến trình init (PID 1) luôn.

### Các bước thực hiện

1. **Reboot** hệ thống
2. Nhấn **`Esc`** để dừng đếm ngược GRUB2
3. Di chuyển tới entry kernel cần boot, nhấn **`E`** để sửa
4. Di chuyển tới dòng bắt đầu bằng `linux`
5. **Xóa mọi option `console=`** khỏi dòng đó
   > ⚠️ Nếu không xóa `console=`, root prompt có thể hiện lên **sai console** → không truy cập được!
6. Di chuyển tới cuối dòng (`Ctrl+E`), thêm: ` init=/bin/bash`
7. `Ctrl+X` để boot với cấu hình đã sửa
8. Chờ boot xong → vào thẳng `bash-5.2#` prompt
9. **Remount root FS sang read/write** (vì đang ở chế độ read-only):
   ```bash
   mount -o remount,rw /
   ```
10. **Đổi mật khẩu:**
    ```bash
    passwd
    ```
11. **Đánh dấu relabel SELinux:**
    ```bash
    touch /.autorelabel
    ```
12. **Khởi động lại đúng cách** (không dùng `reboot`/`systemctl reboot` vì systemd chưa chạy):
    ```bash
    exec /sbin/init
    ```
13. Hệ thống relabel SELinux toàn bộ, rồi tự reboot lại lần nữa.

---

## 4. ⚠️ Vì sao BẮT BUỘC phải `touch /.autorelabel`?

- SELinux **chưa được kích hoạt** trong rescue/emergency mode → mọi file bạn tạo/sửa **không có SELinux context** (nhãn bảo mật).
- Lệnh `passwd` thực chất tạo 1 file **tạm** rồi thay thế file `/etc/shadow` gốc → file `/etc/shadow` mới này **mất label SELinux đúng**.
- Nếu không relabel: hệ thống sau khi boot lại bình thường (SELinux enforcing) sẽ **từ chối truy cập** các file bị sai label → gây lỗi khó hiểu (đăng nhập thất bại, service không chạy được...).
- `touch /.autorelabel` → tạo file cờ hiệu, báo cho hệ thống **relabel toàn bộ SELinux context** ở lần boot tiếp theo (quá trình này có thể mất vài phút, sau đó tự reboot thêm 1 lần).

---

## 5. So sánh 2 phương pháp

| | Rescue Media | `init=/bin/bash` |
|---|---|---|
| Cần gì | File ISO/CD/USB cài đặt | Không cần gì thêm |
| Độ an toàn | Cao hơn, đúng quy trình khuyến nghị | Rủi ro hơn nếu thao tác sai |
| Bước đặc trưng | `chroot /mnt/sysroot` | `mount -o remount,rw /` |
| Khởi động lại | `exit` × 2 | `exec /sbin/init` (không dùng `reboot` bình thường) |
| Điểm chung | Đều cần `passwd`, đều cần `touch /.autorelabel` |

---

## 6. 📝 Bài thực hành: Khôi phục quyền Superuser (không dùng rescue media)

**Mục tiêu:** Reset mật khẩu root về `redhat` bằng phương pháp `init=/bin/bash`.

1. Xác nhận hiện tại **không đăng nhập được** với password `redhat` (login incorrect)
2. Reboot máy (Ctrl+Alt+Del)
3. Tại GRUB2, nhấn `Esc` để dừng đếm ngược
4. Chọn kernel entry, nhấn `E` để sửa
5. Tìm dòng `linux`, **xóa các option `console=`**
6. Di chuyển cuối dòng (`Ctrl+E`), thêm: ` init=/bin/bash`
7. `Ctrl+X` để boot
8. Tại `bash-5.2#`: remount read-write:
   ```bash
   mount -o remount,rw /
   ```
9. Đổi mật khẩu:
   ```bash
   passwd
   # New password: redhat
   # (cảnh báo password ngắn — vẫn chấp nhận)
   # Retype new password: redhat
   ```
10. Đánh dấu relabel: `touch /.autorelabel`
11. Khởi động lại đúng cách: `exec /sbin/init`
12. Chờ hệ thống relabel SELinux + tự reboot
13. Xác nhận đăng nhập lại được: `root` / `redhat` → thành công

---

## 7. Các quy tắc "vàng" cần nhớ

1. ✅ **Rescue media là cách được khuyến nghị** — an toàn hơn `init=/bin/bash`.
2. ⚠️ Rescue media phải **đúng phiên bản** hệ điều hành (RHEL 10 system → RHEL 10 boot ISO).
3. ⚠️ Với `init=/bin/bash`: **luôn xóa `console=`** trước khi thêm tham số, tránh prompt hiện sai console.
4. ✅ Root FS mặc định mount **read-only** trong các chế độ khôi phục → luôn `mount -o remount,rw /` trước khi đổi password.
5. ✅ **Luôn `touch /.autorelabel`** sau khi đổi password bằng `passwd` trong môi trường rescue — nếu quên, hệ thống có thể gặp lỗi SELinux khó hiểu sau khi boot lại bình thường.
6. ⚠️ Khi dùng `init=/bin/bash`: **dùng `exec /sbin/init`** để khởi động lại đúng cách — KHÔNG dùng `reboot` hay `systemctl reboot` (vì systemd chưa chạy, các lệnh đó sẽ không hoạt động).
7. ✅ Với rescue media: `exit` 2 lần (thoát chroot, rồi thoát rescue mode) để hoàn tất và reboot.