# Tổng Hợp: Truy Cập Lưu Trữ Mạng (NFS) trên RHEL 10

> Tài liệu tổng hợp Chương 15 – Accessing Network-attached Storage. Gồm 2 phần: **Mount NFS thủ công/bền vững** và **Automount (autofs)**.

---

## 1. Tổng quan NFS

- **NFS** (Network File System) = giao thức chuẩn để chia sẻ file qua mạng, dùng phổ biến trên Linux/UNIX.
- RHEL 10 dùng **NFSv4.2** mặc định, hỗ trợ cả **NFSv3** và **NFSv4**.

| | NFSv3 | NFSv4 |
|---|---|---|
| Transport | TCP hoặc UDP | **Chỉ TCP** |
| Cơ chế | Dùng RPC (`rpcbind`, port 111) | Không dùng RPC nữa |
| Xem export | `showmount --exports` | Mount root export tree |

**Cài đặt cần thiết:**
```bash
dnf install nfs-utils
```

> 💡 RHEL cũng hỗ trợ mount share Windows qua **SMB/CIFS** (dùng cách tương tự, nhưng option khác — ngoài phạm vi khóa học này).

---

## 2. Truy vấn Export có sẵn trên Server

### NFSv3 — dùng `showmount`
```bash
showmount --exports server
```

### NFSv4 — mount export tree gốc để duyệt
```bash
mount server:/export mountpoint
ls mountpoint      # xem các đường dẫn export có sẵn (chỉ DUYỆT, chưa thực sự mount riêng lẻ)
```
> 💡 Muốn mount 1 export cụ thể: `cd` vào thư mục đó khi đang browse, hoặc dùng đường dẫn đầy đủ với `mount` trực tiếp.
> ⚠️ Export bảo vệ bằng **Kerberos** không cho phép mount/truy cập qua browsing dù thấy được path — cần cấu hình bổ sung (ngoài phạm vi khóa học này).

---

## 3. Mount NFS — 3 phương pháp

| Phương pháp | Bền vững qua reboot? | Use case |
|---|---|---|
| `mount` thủ công | ❌ Không | Test tạm thời, truy cập ngắn hạn |
| `/etc/fstab` | ✅ Có | Cần mount cố định lâu dài |
| `autofs` (automount) | Theo nhu cầu | Mount khi truy cập, tự unmount khi rảnh |

### Mount thủ công
```bash
mkdir mountpoint
mount -t nfs -o rw,sync server:/export mountpoint
```
- `-t nfs`: chỉ định loại FS (nhưng `mount` tự nhận diện khi thấy cú pháp `server:/path`)
- `-o rw,sync`: `rw` = đọc/ghi; `sync` = giao dịch **đồng bộ** — **khuyến nghị cho production** vì đảm bảo transaction hoàn tất hoặc báo lỗi rõ ràng.
- ⚠️ Chỉ **user có quyền (privileged)** mới mount được.
- 💡 Tránh dùng `/mnt` cho mount **dài hạn/bền vững** — chỉ nên dùng tạm thời.

### Mount bền vững qua `/etc/fstab`
```
server:/export  mountpoint  nfs  rw,sync  0 0
```
```bash
mount mountpoint    # đọc thông tin từ fstab để mount
```

> ⚠️ **Nhược điểm của `/etc/fstab` cho NFS:**
> - Timeout tăng lên nếu có sự cố mạng.
> - Timeout cũng tăng khi file system không dùng tới.
> - Có thể gây **lỗi lúc boot** nếu mạng chưa sẵn sàng/không truy cập được.
> → Đây là lý do **autofs** ra đời (xem phần 5).

### Unmount
```bash
umount mountpoint
```
> 💡 `umount` **không xóa entry** trong `/etc/fstab` — file vẫn tự mount lại lúc reboot tiếp theo.

### Xử lý lỗi "device busy" khi unmount
```bash
lsof mountpoint    # xem process nào đang giữ file mở trong mount point
```
→ Đóng ứng dụng/process đó, hoặc `cd` ra khỏi thư mục mount nếu shell đang đứng trong đó, rồi thử `umount` lại.

⚠️ Chỉ trong tình huống **khẩn cấp**, dùng force unmount (có thể **mất dữ liệu chưa ghi**):
```bash
umount -f mountpoint
```

---

## 4. 📝 Bài thực hành: Mount NFS File System

**Mục tiêu:** Mount thủ công `serverb:/shares/public` → unmount → mount lại bền vững qua `/etc/fstab` → xác nhận qua reboot.

1. `mkdir /public`
2. Mount thủ công: `mount -t nfs serverb:/shares/public /public`
3. Kiểm tra nội dung: `ls -l /public`
4. Xem chi tiết mount: `mount | grep public` (thấy `nfs4`, `vers=4.2`, options...)
5. `umount /public`
6. Thêm vào `/etc/fstab`:
   ```
   serverb:/shares/public /public nfs rw,sync 0 0
   ```
7. `systemctl daemon-reload`
8. `mount /public`
9. Kiểm tra nội dung lại
10. **Reboot** máy → login lại → `ls -l /public`, `cat /public/hello` → xác nhận mount **bền vững qua reboot**

---

## 5. Automount (autofs) — Mount theo Nhu cầu

### Vấn đề mà autofs giải quyết
- User thường **không có quyền** dùng `mount` → không tự truy cập được media/NFS chưa mount.
- `/etc/fstab` mount **cố định** suốt thời gian máy chạy — lãng phí tài nguyên nếu ít dùng, và có nhược điểm timeout/lỗi boot đã nêu ở trên.

### Cách hoạt động
- Autofs **chỉ mount khi có người truy cập** vào mount point, và **tự unmount** sau 1 khoảng timeout khi không còn ai dùng.
- Vẫn dùng `mount`/`umount` nội bộ — hành vi khi đang mount **giống hệt** file system mount qua fstab.

### Lợi ích của Automounter
1. **Tiết kiệm tài nguyên**: file system nhàn rỗi/chưa mount gần như không tốn gì.
2. **Bảo vệ khỏi hỏng dữ liệu**: unmount khi không dùng → giảm rủi ro corrupt khi file đang mở lúc có sự cố.
3. **Luôn dùng cấu hình mới nhất**: mount lại mỗi lần cần → không bị "kẹt" với config cũ từ lúc boot như `/etc/fstab`.
4. **Tự chọn kết nối nhanh nhất**: nếu có nhiều server/path dự phòng, autofs có thể chọn cái nhanh nhất mỗi lần mount mới.

### Cài đặt
```bash
dnf install autofs nfs-utils
```

---

## 6. Cấu hình Autofs — Direct Map vs Indirect Map

### File cấu hình chính: Master Map (`/etc/auto.master` hoặc file trong `/etc/auto.master.d/*.autofs`)

Cú pháp:
```
mountpoint     map-file
```

### So sánh Direct vs Indirect Map

| | Direct Map | Indirect Map |
|---|---|---|
| Mount point | **Đường dẫn tuyệt đối cố định** trong master map (`/-`) | Chỉ khai báo **thư mục gốc**, thư mục con tự sinh động |
| Khi nào tồn tại | Autofs tạo/xóa **toàn bộ path** khi cần | Autofs chỉ tạo/xóa **thư mục con** bên dưới base khi cần |
| Cú pháp master map | `/-  /etc/auto.direct` | `/shares  /etc/auto.indirect` |
| Use case | Mount cố định 1 vị trí không đổi | Mount linh hoạt nhiều thư mục con dưới 1 base directory |

### Cấu hình Direct Map
**Master map:**
```
/-  /etc/auto.direct
```
**Map file (`/etc/auto.direct`):**
```
/mnt/docs  -rw,sync  hosta:/shares/docs
```
→ Mount point luôn là **đường dẫn tuyệt đối**.

### Cấu hình Indirect Map
**Master map:**
```
/shares  /etc/auto.indirect
```
**Map file (`/etc/auto.indirect`):**
```
work  -rw,sync  hosta:/shares/work
```
→ Mount point thực tế = `/shares/work` (ghép base + key). Autofs tự tạo/xóa `/shares` và `/shares/work` khi cần.

> 💡 Tên thư mục local **không bắt buộc** phải trùng tên trên server — autofs không ép cấu trúc đặt tên.

### Wildcard trong Map File (mount nhiều thư mục con bằng 1 dòng)
```
*  -rw,sync  hosta:/shares/&
```
- `*` (key) = khớp với bất kỳ tên thư mục con nào được truy cập.
- `&` = tự thay bằng giá trị của `*` tương ứng.
- Ví dụ: truy cập `/shares/work` → tự map sang `hosta:/shares/work`.

### Các option hữu ích khi khai báo map
| Option | Ý nghĩa |
|---|---|
| `-fstype=nfs4` | Chỉ định rõ loại file system |
| `-strict` | Coi lỗi mount là **fatal** (dừng hẳn thay vì bỏ qua) |
| `rw`, `sync` | Giống hệt option mount thông thường |

> ⚠️ Nếu server export ở chế độ **read-only**, client dù yêu cầu `rw` vẫn chỉ nhận được **read-only access**.

### Khởi động dịch vụ Autofs
```bash
systemctl enable --now autofs
```

---

## 7. Phương pháp thay thế: `x-systemd.automount`

Thay vì cài `autofs`, có thể thêm option `x-systemd.automount` trực tiếp vào `/etc/fstab`:
```
server:/export /remote/finance nfs x-systemd.automount 0 0
```
```bash
systemctl daemon-reload
systemctl start remote-finance.automount    # tên unit sinh từ đường dẫn mount
```
> 💡 Đơn giản hơn autofs, nhưng **chỉ hỗ trợ đường dẫn tuyệt đối** (tương tự direct map, không hỗ trợ indirect map).

---

## 8. 📝 Bài thực hành: Automount Storage Devices

**Mục tiêu:** Cấu hình cả direct map và indirect map cho cùng 1 NFS export, quan sát autofs tự mount/unmount theo nhu cầu.

1. `sudo dnf install autofs`
2. Mount thủ công để kiểm tra dữ liệu trước: `sudo mount -t nfs serverb:/shares /home/student/nfs`
3. `grep -r RED /home/student/nfs` → xác nhận có dữ liệu (file test chứa từ "RED")
4. Cấu hình **Master map** (`/etc/auto.master`):
   ```
   /-  /etc/auto.direct
   /home/student/indirect  /etc/auto.indirect
   ```
5. **Direct map** (`/etc/auto.direct`):
   ```
   /home/student/direct  -fstype=nfs,rw,sync  serverb:/shares
   ```
6. **Indirect map** (`/etc/auto.indirect`):
   ```
   *  -fstype=nfs,rw,sync  serverb:/shares/&
   ```
7. `systemctl enable --now autofs`
8. **Kiểm tra Direct Map:**
   - Trước khi truy cập: `mount | grep serverb` → chỉ thấy mount thủ công ban đầu, **chưa có** direct mount
   - Truy cập file: `grep -r RED /home/student/direct` → **lần đầu truy cập kích hoạt mount tự động**
   - Kiểm tra lại: `mount | grep serverb` → giờ thấy thêm `/home/student/direct` đã mount
9. **Kiểm tra Indirect Map:**
   - Trước khi truy cập bất kỳ thư mục con nào: `grep -r RED /home/student/indirect` → **KHÔNG có kết quả** (chưa mount gì cả)
   - Truy cập `west`: `ls /home/student/indirect/west` → autofs **chỉ mount `west`**, chưa mount `south`
   - `grep -r RED /home/student/indirect` → chỉ thấy kết quả trong `west`, **chưa có** `south`
   - Truy cập `south`: `ls /home/student/indirect/south`
   - `grep -r RED /home/student/indirect` → giờ thấy **đầy đủ cả 2** thư mục

> 💡 Bài học cốt lõi: Indirect map autofs **chỉ mount đúng phần được truy cập**, không mount toàn bộ cây thư mục ngay từ đầu — khác hẳn với direct map hay `/etc/fstab` (mount hết 1 lần).

---

## 9. Bảng so sánh nhanh: Khi nào dùng gì?

| Tình huống | Cách nên dùng |
|---|---|
| Test tạm thời 1 export | `mount` thủ công |
| Cần mount cố định, luôn dùng, ít khi đổi | `/etc/fstab` |
| Ít dùng, muốn tiết kiệm tài nguyên + tránh lỗi boot khi mạng chậm | **autofs** |
| Mount point cố định không đổi | **Direct map** (hoặc `/etc/fstab`) |
| Nhiều thư mục con dưới 1 base, không muốn khai báo từng cái | **Indirect map** với wildcard `*`/`&` |
| Chỉ cần đơn giản, mount point tuyệt đối, không muốn cài thêm gói | `x-systemd.automount` trong `/etc/fstab` |

## 10. Các quy tắc "vàng" cần nhớ

1. ✅ NFSv4 chỉ dùng **TCP**, không còn dùng RPC/rpcbind như NFSv3.
2. ✅ Luôn dùng option `sync` cho mount NFS production — đảm bảo transaction toàn vẹn.
3. ⚠️ `/etc/fstab` cho NFS có nhược điểm: tăng timeout, có thể treo lúc boot nếu mạng lỗi — cân nhắc autofs cho các export ít dùng.
4. ✅ `umount` không xóa entry `/etc/fstab` — file sẽ tự mount lại lúc reboot.
5. ⚠️ Lỗi "device busy" khi unmount → dùng `lsof mountpoint` để tìm process đang giữ file, tránh dùng `-f` trừ khi thật cần thiết (có thể mất dữ liệu).
6. ✅ Direct map = mount point cố định (path tuyệt đối trong map file); Indirect map = base directory cố định, thư mục con sinh động theo truy cập.
7. ✅ Indirect map + wildcard (`*` và `&`) giúp mount nhiều thư mục con chỉ với 1 dòng cấu hình.
8. ⚠️ Server export read-only → client luôn chỉ nhận read-only, dù yêu cầu `rw`.
9. ✅ `x-systemd.automount` đơn giản hơn cài `autofs` nhưng chỉ hỗ trợ direct-style mount point.