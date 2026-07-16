# Tổng Hợp: Quản Lý Lưu Trữ Cơ Bản trên RHEL 10

> Tài liệu tổng hợp Chương 10 – Managing Basic Storage. Gồm 2 phần: **Phân vùng & File System (parted, mkfs, mount, fstab)** và **Swap Space (mkswap, swapon)**.

---

## 1. Phân vùng đĩa (Disk Partitioning)

### Vì sao cần phân vùng?
- Giới hạn dung lượng cho app/user cụ thể
- Tách file hệ điều hành khỏi file người dùng
- Tạo vùng riêng cho swap
- Giới hạn dung lượng để tăng tốc backup/diagnostic

### 2 sơ đồ phân vùng chính

| Sơ đồ | Dùng cho | Giới hạn |
|---|---|---|
| **MBR** (Master Boot Record) | Firmware BIOS | Tối đa 4 partition chính (primary); dùng extended+logical → tối đa 15 trên Linux; dung lượng đĩa tối đa **2 TiB** |
| **GPT** (GUID Partition Table) | Firmware UEFI | Tối đa **128 partition**; dung lượng tối đa **8 ZiB**; có checksum chống lỗi + bản backup dự phòng ở cuối đĩa |

> 💡 GPT là chuẩn hiện đại, thay thế MBR do MBR bị giới hạn 2TiB — hạn chế phổ biến với ổ đĩa lớn ngày nay.

---

## 2. Công cụ `parted` — Quản lý Phân vùng

### Xem bảng phân vùng
```bash
parted /dev/sda print
```

### Đổi đơn vị hiển thị
```bash
parted /dev/sda unit s print     # xem theo sector
```
Các đơn vị hỗ trợ: `s` (sector), `B` (byte), `MiB/GiB/TiB` (lũy thừa 2), `MB/GB/TB` (lũy thừa 10).

### Chế độ tương tác vs không tương tác
```bash
# Tương tác
parted /dev/sdb
(parted) mkpart ...

# Không tương tác (1 dòng lệnh)
parted /dev/sdb mkpart primary xfs 2048s 1000MB
```

### Bước 1: Ghi disk label (chọn sơ đồ phân vùng)
```bash
parted /dev/sdb mklabel msdos    # MBR
parted /dev/sdb mklabel gpt      # GPT
```
> ⚠️ **`mklabel` xóa sạch bảng phân vùng hiện có** — chỉ dùng khi chắc chắn không cần dữ liệu cũ.

### Bước 2: Tạo partition (`mkpart`)

**MBR:**
```
(parted) mkpart
Partition type? primary/extended? primary
File system type? [ext2]? xfs
Start? 2048s
End? 1000MB
```
- Cần chọn `primary` hoặc `extended` (extended dùng khi cần >4 partition — chứa nhiều logical partition bên trong).
- `FS-TYPE` (vd `xfs`) chỉ là **nhãn loại partition**, KHÔNG tạo file system thật — cần chạy `mkfs` riêng ở bước sau.

**GPT:**
```
(parted) mkpart
Partition name? []? userdata
File system type? [ext2]? xfs
Start? 2048s
End? 1000MB
```
- GPT yêu cầu đặt **tên** cho partition (không có khái niệm primary/extended).

> 💡 Start sector nên là bội số của **2048** để đảm bảo alignment tối ưu hiệu năng.

### Sau khi tạo partition
```bash
udevadm settle    # chờ hệ thống nhận diện & tạo device file trong /dev
```

### Xóa partition
```bash
parted /dev/sdb print     # xác định số thứ tự partition cần xóa
parted /dev/sdb rm 1      # xóa partition số 1
```

---

## 3. Tạo File System

```bash
mkfs.xfs /dev/sdb1     # định dạng XFS (khuyến nghị mặc định của RHEL)
mkfs.ext4 /dev/sdb1    # định dạng ext4
```

---

## 4. Mount File System

### Mount thủ công (tạm thời, mất khi reboot)
```bash
mount /dev/sdb1 /mnt
mount | grep sdb1     # kiểm tra (không thêm dấu / cuối tên partition)
```

### Mount bền vững qua `/etc/fstab`

**Cấu trúc 6 cột (cách nhau bằng khoảng trắng):**
```
UUID=xxxx-xxxx   /mount/point   xfs   defaults   0   0
```

| Cột | Ý nghĩa |
|---|---|
| 1. Device | Nên dùng **UUID** (ổn định, không đổi kể cả khi tên `/dev/sdX` bị đổi do thứ tự nhận diện đĩa thay đổi) |
| 2. Mount point | Thư mục đích (phải tồn tại sẵn, tạo bằng `mkdir` nếu chưa có) |
| 3. Loại FS | vd `xfs`, `ext4` |
| 4. Options | `defaults` = tập hợp option phổ biến |
| 5. Dump flag | Dùng cho lệnh `dump`, thường để 0 |
| 6. fsck order | XFS → luôn để **0** (không dùng fsck); ext4: root = **1**, các FS khác = **2** |

### Lấy UUID
```bash
lsblk --fs /dev/sdb1
```

### Sau khi sửa `/etc/fstab`
```bash
systemctl daemon-reload
mount /archive          # mount theo entry trong fstab
```

> ⚠️ **Cực kỳ quan trọng:** Sai sót trong `/etc/fstab` có thể khiến máy **không boot được**. Trước khi reboot, hãy kiểm tra bằng cách unmount rồi mount lại theo đúng entry đó, hoặc dùng:
> ```bash
> findmnt --verify
> ```

---

## 5. 📝 Bài thực hành: Tạo & Quản lý File System trên Partition

**Mục tiêu:** Tạo 1 partition MBR 1GB, định dạng XFS, mount bền vững vào `/archive`.

1. `parted /dev/sdb mklabel msdos`
2. Tạo partition (tương tác):
   ```
   parted /dev/sdb
   (parted) mkpart → primary → xfs → Start: 2048s → End: 1001MB
   (parted) quit
   ```
3. `udevadm settle`
4. `mkfs.xfs /dev/sdb1`
5. `mkdir /archive`
6. Lấy UUID: `lsblk --fs /dev/sdb`
7. Thêm vào `/etc/fstab`:
   ```
   UUID=291bb68e-91b9-4522-8566-7cc1d848babd /archive xfs defaults 0 0
   ```
8. `systemctl daemon-reload`
9. `mount /archive`
10. Kiểm tra: `mount | grep /archive`
11. **Reboot** máy → login lại → kiểm tra `mount | grep /archive` vẫn còn → xác nhận mount **bền vững qua reboot**

---

## 6. Swap Space

### Khái niệm
- Swap = vùng đĩa được kernel dùng để **mở rộng RAM** — lưu tạm các trang bộ nhớ (memory page) ít dùng khi RAM đầy.
- **Virtual memory = RAM + Swap**.
- ⚠️ Swap **chậm hơn RAM nhiều** — không phải giải pháp lâu dài thay cho việc thiếu RAM.

### Bảng khuyến nghị kích thước Swap

| RAM | Swap khuyến nghị | Swap nếu cần Hibernate |
|---|---|---|
| ≤ 2 GB | Gấp 2 lần RAM | Gấp 3 lần RAM |
| 2 GB – 8 GB | Bằng RAM | Gấp 2 lần RAM |
| 8 GB – 64 GB | Tối thiểu 4 GB | Gấp 1.5 lần RAM |
| > 64 GB | Tối thiểu 4 GB | **Không khuyến khích** hibernate |

> 💡 Hibernate cần swap ≥ RAM vì toàn bộ nội dung RAM được lưu vào swap trước khi tắt máy.

### Quy trình tạo Swap Space (2 bước)

**Bước 1: Tạo partition kiểu `linux-swap`**
```bash
parted /dev/sdb mkpart swap1 linux-swap 1001MB 1257MB
udevadm settle
```

**Bước 2: Đóng dấu (format) swap signature**
```bash
mkswap /dev/sdb2
```
> Khác với `mkfs`, `mkswap` chỉ ghi 1 block dữ liệu ở đầu device, phần còn lại để kernel tự quản lý (không phải file system truyền thống).

### Kích hoạt / Tắt Swap

| Lệnh | Ý nghĩa |
|---|---|
| `swapon /dev/sdb2` | Kích hoạt (tạm thời) |
| `swapon -a` | Kích hoạt tất cả swap khai báo trong `/etc/fstab` |
| `swapon --show` | Xem danh sách swap đang active |
| `free` | Xem tổng quan RAM + Swap |
| `swapoff /dev/sdb2` | Tắt swap (kernel sẽ cố gắng chuyển page đang dùng sang chỗ khác; nếu không đủ chỗ → lệnh **thất bại**, swap vẫn active) |

> ⚠️ `swapon` **không** làm swap bền vững qua reboot — cần thêm vào `/etc/fstab`.

### Kích hoạt Swap bền vững qua `/etc/fstab`
```
UUID=39e2667a-9458-42fe-9665-c5c854605881   swap   swap   defaults   0 0
```
- Cột mount point dùng giá trị `swap` (thay vì đường dẫn thư mục, vì swap không có mount point thật).
- Cột FS type = `swap`.
- `defaults` bao gồm auto-mount lúc boot.
- Sau khi sửa: `systemctl daemon-reload`

### Ưu tiên Swap (Priority)
- Kernel dùng swap có **priority cao nhất trước**, hết chỗ mới chuyển sang priority thấp hơn.
- Priority mặc định = **-2**; swap tạo sau có priority thấp hơn swap tạo trước (nếu không set thủ công).
- Cùng priority → kernel ghi luân phiên kiểu round-robin.
- Set priority bằng option `pri=` trong `/etc/fstab`:
  ```
  UUID=aaa...   swap   swap   defaults   0 0
  UUID=bbb...   swap   swap   pri=4      0 0
  UUID=ccc...   swap   swap   pri=10     0 0
  ```
  → Kernel dùng UUID `ccc` (pri=10) trước, rồi `bbb` (pri=4), cuối cùng `aaa` (pri=-2, mặc định).

---

## 7. 📝 Bài thực hành: Quản lý Swap Space

**Mục tiêu:** Tạo partition swap 500MB trên GPT disk đã có sẵn 1 partition, kích hoạt và làm nó bền vững.

1. Kiểm tra disk hiện có: `parted /dev/sdb print` → đã có 1 partition GPT `data` (1000MB)
2. Tạo partition swap ngay sau đó:
   ```bash
   parted /dev/sdb mkpart myswap linux-swap 1001MB 1501MB
   ```
3. `udevadm settle`
4. Format swap: `mkswap /dev/sdb2`
5. **Kiểm chứng chưa active:** `swapon --show` → rỗng
6. Kích hoạt: `swapon /dev/sdb2`
7. Xác nhận: `swapon --show` → thấy `/dev/sdb2 ... 476M ... -2` (priority mặc định)
8. Tắt thử: `swapoff /dev/sdb2` → xác nhận `swapon --show` lại rỗng
9. Lấy UUID: `lsblk --fs /dev/sdb2`
10. Thêm vào `/etc/fstab`:
    ```
    UUID=968e395a-8d1a-4ff0-95aa-71e6671e0ecf  swap  swap  defaults  0 0
    ```
11. `systemctl daemon-reload`
12. Kích hoạt theo fstab: `swapon -a`
13. Xác nhận: `swapon --show`
14. **Reboot** máy → login lại → `swapon --show` vẫn thấy swap active → xác nhận **bền vững qua reboot**

---

## 8. Bảng tóm tắt lệnh nhanh

| Việc cần làm | Lệnh |
|---|---|
| Xem bảng phân vùng | `parted /dev/sdX print` |
| Tạo disk label | `parted /dev/sdX mklabel msdos\|gpt` |
| Tạo partition | `parted /dev/sdX mkpart ...` |
| Xóa partition | `parted /dev/sdX rm SỐ` |
| Chờ device file xuất hiện | `udevadm settle` |
| Định dạng XFS | `mkfs.xfs /dev/sdXn` |
| Mount tạm | `mount /dev/sdXn /mnt` |
| Lấy UUID | `lsblk --fs /dev/sdXn` |
| Nạp lại fstab | `systemctl daemon-reload` |
| Format swap | `mkswap /dev/sdXn` |
| Bật/tắt swap | `swapon` / `swapoff` |
| Xem trạng thái swap | `swapon --show`, `free` |

## 9. Các quy tắc "vàng" cần nhớ

1. ⚠️ `mklabel` xóa sạch bảng phân vùng — cẩn thận với dữ liệu cũ.
2. ✅ FS-type khi `mkpart` chỉ là **nhãn**, chưa tạo file system thật — luôn cần chạy `mkfs.xfs`/`mkfs.ext4` sau đó.
3. ✅ Start sector nên là bội số của 2048 để tối ưu alignment.
4. ✅ Luôn dùng **UUID** thay vì tên `/dev/sdX` trong `/etc/fstab` — tên device có thể đổi giữa các lần boot.
5. ⚠️ Sai `/etc/fstab` có thể làm máy không boot được — kiểm tra kỹ trước khi reboot (`findmnt --verify` hoặc mount thử).
6. ✅ XFS → fsck order = 0; ext4 root = 1, ext4 khác = 2.
7. ✅ `mkswap` khác `mkfs` — không tạo file system, chỉ đóng dấu swap signature.
8. ✅ `swapon`/`mount` thủ công chỉ tạm thời — luôn cần thêm vào `/etc/fstab` để bền vững qua reboot.
9. ✅ Swap priority càng cao → được dùng càng trước; mặc định = -2.