# Tổng Hợp: Kiểm Soát & Khắc Phục Sự Cố Quá Trình Boot trên RHEL 10

> Tài liệu tổng hợp Chương 12 – Controlling and Troubleshooting the Boot Process. Gồm 3 phần: **Boot Loader (GRUB2)**, **Systemd Target**, và **Sửa File System hỏng lúc boot**.

---

## 1. Tổng quan quá trình Boot

```
Firmware (UEFI/BIOS) → Boot Loader (GRUB2) → Kernel + initramfs 
→ systemd (PID 1) → pivot root → chọn target → login
```

| Giai đoạn | Việc xảy ra |
|---|---|
| **Firmware** | UEFI (phổ biến hiện nay) hoặc BIOS (máy trước 2020) nạp boot loader từ đĩa |
| **GRUB2** | Boot loader mặc định của RHEL 10; hiển thị menu chọn kernel để boot |
| **Kernel + initramfs** | Kernel nạp vào RAM cùng initramfs (chứa driver, script khởi tạo, systemd) |
| **systemd (PID 1)** | `/sbin/init` (symlink tới systemd) chạy trong initramfs → mount root FS vào `/sysroot` → **pivot root** sang FS thật trên đĩa → systemd tự chạy lại bằng bản cài trên đĩa |
| **Target** | systemd tìm target mặc định, khởi động các unit cần thiết → hoàn tất boot (login screen) |

> 💡 Dùng `lsinitrd` để kiểm tra/trích xuất nội dung file initramfs.

---

## 2. GRUB2 — Boot Loader

### GRUB2 trên UEFI vs BIOS

| | UEFI | BIOS |
|---|---|---|
| Đọc gì | NVRAM → biết đĩa/EFI app nào để nạp | Boot sector đặc biệt chứa `boot.img` |
| Vị trí | EFI system partition, thường mount ở `/boot/efi` | Vùng MBR chưa phân vùng (trước partition đầu tiên) |
| File chính | GRUB2 sinh file EFI application | `boot.img` → nạp `core.img` từ MBR |

### Menu GRUB2 lúc boot
- Nhấn **`Esc`** (hoặc phím bất kỳ trừ Enter) để dừng đếm ngược khi menu hiện ra.
- Nhấn **`E`** để mở **editor** — sửa tạm thời cho **1 lần boot đó** (không lưu vĩnh viễn).
- `Ctrl+X` hoặc `F10` → boot với cấu hình đã sửa.

> ⚠️ Sửa trong GRUB2 editor **chỉ áp dụng 1 lần**. Muốn sửa vĩnh viễn phải dùng lệnh `grubby` từ hệ thống đã boot.

---

## 3. Quản lý GRUB2 bằng `grubby`

### Xem thông tin 1 entry
```bash
grubby --info 1     # 1 = index, bắt đầu từ 0
```
Các trường quan trọng:

| Trường | Ý nghĩa |
|---|---|
| `index` | Số thứ tự entry trong boot menu |
| `kernel` | Đường dẫn tới file kernel |
| `args` | Danh sách kernel command-line arguments |
| `root` | Block device chứa kernel + initramfs |
| `initrd` | Đường dẫn tới ảnh initramfs |
| `title` | Tên hiển thị trong menu GRUB2 |
| `id` | ID định danh duy nhất |

### Xem/đặt index kernel mặc định
```bash
grubby --default-index              # xem index hiện tại
grubby --set-default-index 0        # đặt kernel index 0 làm mặc định
grubby --set-default /path/kernel   # đặt theo đường dẫn kernel
```

### Thêm/xóa kernel command-line argument (bền vững)
```bash
grubby --update-kernel /boot/vmlinuz-X --args="rhgb quiet"          # thêm
grubby --update-kernel /boot/vmlinuz-X --remove-args="rhgb quiet"   # xóa
```

### 📝 Bài thực hành: Quản lý Boot Loader
**Mục tiêu:** Đổi kernel mặc định, thêm/xóa kernel argument, xác nhận qua reboot.

1. Reboot máy, mở menu GRUB2 (`Esc`), nhấn `E` xem editor (không sửa gì), `Ctrl+X` boot bình thường
2. Login, xem index mặc định: `grubby --default-index` → `1`
3. Xem chi tiết: `grubby --info 1`
4. Đổi mặc định sang index 0: `grubby --set-default-index 0`
5. Thêm argument `rhgb quiet` vào kernel mới: `grubby --update-kernel /boot/vmlinuz-... --args="rhgb quiet"`
6. `systemctl reboot` → xác nhận thông báo boot bị ẩn bớt (do `quiet`)
7. Đổi lại về kernel cũ: `grubby --set-default 1`
8. Xóa argument đã thêm: `grubby --update-kernel /boot/vmlinuz-... --remove-args="rhgb quiet"`
9. `systemctl reboot` → xác nhận boot lại bình thường

---

## 4. Systemd Target — Chọn "trạng thái đích" khi boot

### Khái niệm
**Target** = tập hợp các unit systemd cần khởi động để đạt 1 trạng thái hệ thống mong muốn.

### Các target quan trọng

| Target | Mục đích |
|---|---|
| `graphical.target` | Đa người dùng, có **GUI + CLI** login |
| `multi-user.target` | Đa người dùng, **chỉ CLI** login |
| `rescue.target` | Chế độ single-user để sửa lỗi (đã init phần lớn: logging, mount FS...) |
| `emergency.target` | Chế độ tối giản nhất, dùng khi `rescue.target` cũng không khởi động được được |

> 💡 Target có thể **lồng nhau**: `graphical.target` bao gồm `multi-user.target`, cái này lại phụ thuộc `basic.target`, v.v.

### Xem cây phụ thuộc target
```bash
systemctl list-dependencies graphical.target | grep target
```

### Liệt kê tất cả target
```bash
systemctl list-units --type=target --all
```

### Chuyển target ngay lập tức (không reboot)
```bash
systemctl isolate multi-user.target
```
- `isolate` = dừng mọi service **không** cần cho target đích, khởi động service **cần** mà chưa chạy.
- ⚠️ Chỉ isolate được target có `AllowIsolate=yes` trong file unit (kiểm tra bằng `systemctl cat TÊN.target`).

### Xem/đặt target mặc định (bền vững)
```bash
systemctl get-default              # xem target mặc định
systemctl set-default graphical.target   # đặt target mặc định (tạo symlink /etc/systemd/system/default.target)
```

### Chọn target khác **chỉ cho 1 lần boot** (qua GRUB2)
1. Reboot, tại menu GRUB2 nhấn phím bất kỳ (trừ Enter) để dừng đếm ngược
2. Nhấn `E` để sửa entry
3. Di chuyển tới dòng bắt đầu bằng `linux`
4. Thêm vào cuối dòng: `systemd.unit=rescue.target` (hoặc `emergency.target`)
5. `Ctrl+X` để boot với cấu hình tạm này

### Rescue vs Emergency — khác nhau gì?

| | `rescue.target` | `emergency.target` |
|---|---|---|
| Chờ gì trước khi vào shell | Chờ `sysinit.target` hoàn tất (nhiều dịch vụ đã chạy: logging, mount FS...) | Không chờ gì — tối giản nhất |
| Root FS | Mount **read/write** (đủ điều kiện) | Mount **read-only** — muốn sửa `/etc/fstab` phải remount trước: `mount -o remount,rw /` |
| Dùng khi | Sự cố nhẹ, cần môi trường gần đầy đủ hơn | `rescue.target` cũng không khởi động nổi |

### Tắt / Khởi động lại hệ thống
```bash
systemctl poweroff   # dừng service, unmount FS, tắt máy
systemctl reboot      # dừng service, unmount FS, khởi động lại
systemctl halt        # đưa hệ thống về trạng thái an toàn để TẮT THỦ CÔNG (không tự tắt nguồn)
```
(`poweroff`, `reboot`, `halt` không tiền tố `systemctl` cũng chạy được — là symlink tới lệnh tương đương)

### 📝 Bài thực hành: Khám phá Boot Process & Chọn Target
1. Xác nhận target mặc định: `systemctl get-default` → `graphical.target`
2. Chuyển tạm sang CLI: `sudo systemctl isolate multi-user.target` → màn hình chuyển sang text console
3. Đặt CLI làm mặc định: `systemctl set-default multi-user.target`
4. `systemctl reboot` → xác nhận boot vào text console (không GUI)
5. Đặt lại GUI làm mặc định: `systemctl set-default graphical.target`
6. **Phần 2 — vào rescue target qua GRUB2:**
   - `systemctl reboot`, tại GRUB2 nhấn `Esc` → `E` → thêm `systemd.unit=rescue.target` vào cuối dòng `linux`
   - `Ctrl+X` để boot
   - Login rescue mode (yêu cầu password root)
   - Kiểm tra root FS: `mount` → thấy `rw` (rescue mode cho phép ghi)
   - `Ctrl+D` để tiếp tục boot bình thường vào graphical mode

---

## 5. Sửa File System Hỏng Lúc Boot

### Các lỗi thường gặp khi mount `/etc/fstab` lúc boot

| Lỗi | systemd xử lý thế nào |
|---|---|
| File system bị hỏng (corrupted) | Tự động thử sửa (replay journal với XFS/ext4); nếu không sửa được → vào **emergency shell** |
| Thiết bị lưu trữ/UUID/mount point không tồn tại | Timeout chờ thiết bị → vào **emergency shell** |

### Công cụ kiểm tra & sửa file system

| File system | Lệnh |
|---|---|
| **XFS** | `xfs_repair block-device` |
| **ext4/ext3/ext2** | `fsck.ext4 -p block-device` (hard link tới `e2fsck`; `-p` = tự sửa lỗi nhẹ không cần hỏi) |

> ⚠️ **Lưu ý quan trọng:**
> - `fsck.xfs` tồn tại trên hệ thống nhưng **chỉ để thỏa điều kiện boot** — thực chất **không làm gì**, luôn exit code 0.
> - Trước khi chạy `xfs_repair` hoặc `e2fsck`, file system **phải được unmount** để tránh mất/hỏng dữ liệu.
> - Quy trình đúng: mount rồi unmount trước để "replay" log file, tránh dữ liệu tồn dư/hỏng.

### Quy trình sửa lỗi khi rơi vào Emergency Shell

1. **Xem trạng thái mount hiện tại:**
   ```bash
   mount
   ```
2. **Nếu root FS đang read-only**, remount sang read-write để sửa được `/etc/fstab`:
   ```bash
   mount -o remount,rw /
   ```
3. **Thử mount lại toàn bộ theo `/etc/fstab`:**
   ```bash
   mount --all    # hoặc mount -a
   ```
   → Lệnh này bỏ qua FS đã mount, chỉ báo lỗi cho entry nào có vấn đề.
4. **Sửa lỗi cụ thể** trong `/etc/fstab` (vd: mount point không tồn tại → tạo bằng `mkdir`; sai UUID/đường dẫn → sửa lại đúng).
5. **Báo cho systemd nạp lại cấu hình mới:**
   ```bash
   systemctl daemon-reload
   ```
6. **Thử mount lại lần nữa để xác nhận hết lỗi:**
   ```bash
   mount --all
   ```
7. **Reboot để kiểm tra boot bình thường:**
   ```bash
   systemctl reboot
   ```

### 💡 Mẹo test nhanh: option `nofail`
Thêm `nofail` vào cột option trong `/etc/fstab` → hệ thống vẫn **boot được** dù mount entry đó thất bại.
> ⚠️ **Không dùng cho file system quan trọng cho production** — ứng dụng có thể khởi động dù thiếu dữ liệu, gây hậu quả nghiêm trọng.

### 📝 Bài thực hành: Sửa File System Hỏng Lúc Boot
**Mục tiêu:** Hệ thống có lỗi trong `/etc/fstab` (mount point `/fakeroot` không tồn tại) khiến boot thất bại → sửa lỗi từ emergency shell.

1. Reboot máy → boot process **không hoàn tất** (do lỗi fstab)
2. Vào GRUB2 (`Esc` → `E`), thêm `systemd.unit=emergency.target` vào dòng `linux`, `Ctrl+X`
3. Login emergency shell (password root)
4. Kiểm tra mount: `mount` → thấy root FS đang **read-only** (`ro`)
5. Remount read-write: `mount -o remount,rw /`
6. Thử mount tất cả: `mount -a` → báo lỗi `/fakeroot: mount point does not exist`
7. Sửa `/etc/fstab`: đổi `/fakeroot` thành `/` (mount point đúng)
8. `systemctl daemon-reload`
9. Xác nhận hết lỗi: `mount -a` (không còn báo lỗi)
10. `systemctl reboot` → xác nhận boot thành công, login được bình thường

---

## 6. Bảng so sánh nhanh: Khi nào dùng gì?

| Tình huống | Công cụ/Cách xử lý |
|---|---|
| Cần đổi kernel mặc định bền vững | `grubby --set-default-index` |
| Cần thêm/xóa kernel argument bền vững | `grubby --update-kernel --args/--remove-args` |
| Cần thử nghiệm boot khác **1 lần duy nhất** | Sửa trong GRUB2 editor (nhấn `E`) |
| Cần đổi chế độ chạy hệ thống (GUI/CLI) ngay | `systemctl isolate` |
| Cần đổi chế độ chạy **bền vững** | `systemctl set-default` |
| Hệ thống không boot được, cần sửa | Boot vào `rescue.target` hoặc `emergency.target` qua GRUB2 |
| Root FS hỏng hoặc lỗi `/etc/fstab` | `xfs_repair` / `fsck.ext4`, sửa `/etc/fstab`, `daemon-reload` |

## 7. Các quy tắc "vàng" cần nhớ

1. ✅ Sửa trong GRUB2 editor (`E`) chỉ áp dụng **1 lần boot** — muốn bền vững phải dùng `grubby`.
2. ✅ `isolate` chỉ hoạt động với target có `AllowIsolate=yes`.
3. ⚠️ `emergency.target` mount root **read-only** — phải `remount,rw` mới sửa được `/etc/fstab`; `rescue.target` thì đã **read-write** sẵn.
4. ⚠️ Luôn **unmount** trước khi chạy `xfs_repair`/`e2fsck` để tránh mất dữ liệu.
5. ✅ `fsck.xfs` không thực sự làm gì — XFS dùng `xfs_repair` để sửa lỗi thật sự.
6. ✅ Sau khi sửa `/etc/fstab`, luôn `systemctl daemon-reload` trước khi `mount -a` lại.
7. ⚠️ Option `nofail` chỉ nên dùng để **test**, không dùng cho FS quan trọng ở production.
8. ✅ Muốn boot vào target khác chỉ 1 lần → thêm `systemd.unit=TÊN.target` vào dòng `linux` trong GRUB2 editor.