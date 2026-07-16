# Tổng Hợp: Quản Lý Lưu Trữ bằng LVM trên RHEL 10

> Tài liệu tổng hợp Chương 11 – Managing Storage with Logical Volume Manager (LVM). Gồm 3 phần: **Tạo Logical Volume**, **Mở rộng Logical Volume**, và **Thay thế/Gỡ bỏ Physical Volume**.

---

## 1. Tổng quan LVM

### Vì sao dùng LVM?
- Tạo lớp **lưu trữ logic** trên lưu trữ vật lý → linh hoạt hơn dùng trực tiếp partition.
- **Resize volume mà không cần dừng ứng dụng hoặc unmount** file system.
- Ẩn đi cấu trúc phần cứng thực tế khỏi phần mềm.

### 3 lớp khái niệm cốt lõi

| Lớp | Ý nghĩa | Đơn vị nhỏ nhất |
|---|---|---|
| **Physical Volume (PV)** | Thiết bị vật lý (partition, cả ổ đĩa, RAID, SAN) được LVM "khởi tạo" để dùng | **Physical Extent (PE)** |
| **Volume Group (VG)** | "Bể chứa" gộp từ 1 hoặc nhiều PV — tương đương 1 ổ đĩa lớn ảo | Gộp các PE từ nhiều PV |
| **Logical Volume (LV)** | Volume thực sự cấp cho ứng dụng/user, tạo từ PE trống trong VG | **Logical Extent (LE)**, mặc định 1 LE ↔ 1 PE |

> ⚠️ 1 PV chỉ thuộc **1 VG duy nhất**. Toàn bộ thiết bị vật lý phải được dùng làm PV (không dùng 1 phần).

### Luồng làm việc (Workflow) LVM
```
1. Chọn thiết bị vật lý → khởi tạo thành Physical Volume (pvcreate)
2. Gộp nhiều PV → tạo Volume Group (vgcreate)
3. Tạo Logical Volume từ không gian trống trong VG (lvcreate)
4. Format LV với file system (mkfs) hoặc dùng làm swap, rồi mount
```

---

## 2. Tạo LVM Storage — Từng bước

### Bước 0: Chuẩn bị thiết bị vật lý (tùy chọn — có thể dùng cả ổ đĩa raw)
```bash
parted /dev/sdb mklabel gpt
parted /dev/sdb mkpart primary 1MiB 769MiB
parted /dev/sdb mkpart secondary 770MiB 1026MiB
parted /dev/sdb set 1 lvm on      # đánh dấu loại partition = Linux LVM
parted /dev/sdb set 2 lvm on
udevadm settle
```

### Bước 1: Tạo Physical Volume
```bash
pvcreate /dev/sdb1 /dev/sdb2      # có thể tạo nhiều PV cùng lúc
```

### Bước 2: Tạo Volume Group
```bash
vgcreate vg01 /dev/sdb1 /dev/sdb2
```

### Bước 3: Tạo Logical Volume
```bash
lvcreate -n lv01 -L 300M vg01     # -L: theo dung lượng (byte/MiB/GiB)
lvcreate -n lv01 -l 32 vg01       # -l: theo số lượng PE (viết thường)
```
> 💡 Nếu size không khớp chính xác bội số PE, LVM sẽ **tự làm tròn lên** PE gần nhất. Lệnh sẽ **lỗi** nếu VG không đủ PE trống.

### Bước 4: Format & Mount
```bash
mkfs -t xfs /dev/vg01/lv01              # hoặc /dev/mapper/vg01-lv01
mkdir /mnt/data
echo "/dev/vg01/lv01 /mnt/data xfs defaults 0 0" >> /etc/fstab
systemctl daemon-reload
mount /mnt/data
```
> 💡 Có thể mount LV theo **tên** hoặc **UUID** — LVM tự parse UUID từ PV nên luôn hoạt động đúng dù VG được đặt bằng tên.

---

## 3. Xem thông tin trạng thái LVM

| Lệnh chi tiết | Lệnh tóm tắt (1 dòng/entity) | Xem thông tin gì |
|---|---|---|
| `pvdisplay [PV]` | `pvs` | Physical Volume |
| `vgdisplay [VG]` | `vgs` | Volume Group |
| `lvdisplay [LV]` | `lvs` | Logical Volume |

**Các trường quan trọng cần chú ý:**

| Lệnh | Trường | Ý nghĩa |
|---|---|---|
| `pvdisplay` | `PV Size` | Dung lượng vật lý (gồm cả phần không dùng được) |
| | `PE Size` | Kích thước 1 physical extent |
| | `Free PE` | Số PE trống, có thể dùng để tạo/mở rộng LV |
| `vgdisplay` | `VG Size` | Tổng dung lượng pool khả dụng |
| | `Free PE / Size` | Không gian trống trong VG |
| `lvdisplay` | `LV Size` | Tổng dung lượng LV |
| | `Current LE` | Số logical extent LV đang dùng |

---

## 4. 📝 Bài thực hành: Tạo Logical Volume

**Mục tiêu:** Tạo 2 partition 256MiB, gộp thành VG, tạo LV 400MiB, format XFS, mount bền vững vào `/data`.

1. `parted /dev/sdb mklabel gpt`
2. Tạo 2 partition:
   ```bash
   parted /dev/sdb mkpart first 1MiB 257MiB
   parted /dev/sdb set 1 lvm on
   parted /dev/sdb mkpart second 257MiB 513MiB
   parted /dev/sdb set 2 lvm on
   udevadm settle
   ```
3. Tạo PV: `pvcreate /dev/sdb1 /dev/sdb2`
4. Tạo VG: `vgcreate vg_servera /dev/sdb1 /dev/sdb2`
5. Tạo LV: `lvcreate -n lv_servera -L 400M vg_servera`
6. Format: `mkfs -t xfs /dev/vg_servera/lv_servera`
7. `mkdir /data`, thêm vào `/etc/fstab`:
   ```
   /dev/vg_servera/lv_servera /data xfs defaults 0 0
   ```
8. `systemctl daemon-reload` → `mount /data`
9. Test ghi dữ liệu: `cp -a /etc/*.conf /data` → `ls /data | wc -l` → 30 file
10. Kiểm tra thông tin: `pvdisplay /dev/sdb2`, `vgdisplay vg_servera`, `lvdisplay /dev/vg_servera/lv_servera`, `df -h /data`

---

## 5. Mở rộng Logical Volume (Extend)

### Ưu điểm lớn của LVM: **resize không downtime**

### Bước 1: Mở rộng LV
```bash
lvextend -L +500M /dev/vg01/lv01
```
- Dấu **`+`** phía trước = **cộng thêm** vào size hiện có.
- Không có dấu `+` = đặt **kích thước cuối cùng** (tuyệt đối).
- `-l` = theo số PE, `-L` = theo dung lượng.
- Trước khi extend, kiểm tra `vgdisplay` xem VG còn đủ **Free PE** không.

### Bước 2: Mở rộng File System (khác nhau theo loại FS!)

**XFS:**
```bash
xfs_growfs /mnt/data/    # dùng MOUNT POINT, không phải device name
```
> ⚠️ File system phải **đang mount** trước khi chạy `xfs_growfs`. XFS **chỉ resize lên được**, không thể thu nhỏ.

**ext4:**
```bash
resize2fs /dev/vg01/lv01    # dùng DEVICE NAME, không phải mount point
```
> `resize2fs` hỗ trợ **cả online lẫn offline**, và có thể resize **lên hoặc xuống** — linh hoạt hơn XFS.

### 💡 Rút gọn 2 bước thành 1
```bash
lvextend -r -L +500M /dev/vg01/lv01
```
`-r` tự động chạy luôn bước resize file system tương ứng (XFS hoặc ext4) sau khi extend LV.

### Mở rộng Swap Logical Volume (⚠️ khác quy trình FS thường)
Swap LV **phải offline** mới extend được:
```bash
swapoff -v /dev/vg01/swap        # 1. tắt swap
lvextend -L +300M /dev/vg01/swap  # 2. mở rộng LV
mkswap /dev/vg01/swap             # 3. format lại swap signature
swapon /dev/vg01/swap             # 4. bật lại
```
> ⚠️ Rủi ro: nếu hệ thống đang thiếu RAM khi tắt swap, có thể gặp lỗi **out-of-memory (OOM)**. Giải pháp an toàn hơn: tạo **LV swap mới** với size mong muốn, bật nó lên, rồi tắt/xóa swap cũ sau khi đã chuyển hẳn sang cái mới.

---

## 6. 📝 Bài thực hành: Mở rộng Logical Volume

**Mục tiêu:** Thêm partition thứ 3 (512MiB) vào VG hiện có, mở rộng LV `lv_servera` lên 700MiB, mở rộng file system XFS tương ứng.

1. Tạo partition 3: `parted /dev/sdb mkpart third 514MiB 1026MiB`
2. `parted /dev/sdb set 3 lvm on`
3. `udevadm settle`
4. Tạo PV: `pvcreate /dev/sdb3`
5. Mở rộng VG: `vgextend vg_servera /dev/sdb3`
6. Mở rộng LV: `lvextend -L 700M /dev/vg_servera/lv_servera` (chỉ định size **tuyệt đối**, không dùng `+`)
7. Mở rộng file system: `xfs_growfs /data`
8. Kiểm tra: `lvdisplay`, `df -h /data` (từ 336M → 636M), `ls /data | wc -l` → vẫn 30 file (dữ liệu **không mất**)

---

## 7. Thay thế Physical Volume & Quản lý LVM

### Mở rộng Volume Group (thêm PV mới)
```bash
pvcreate /dev/sdb3
vgextend vg01 /dev/sdb3
```

### Thu nhỏ Volume Group (di chuyển dữ liệu ra khỏi 1 PV rồi loại bỏ nó)

**Use case:** Thay thế PV nhỏ/sắp hỏng bằng PV khác, **không cần downtime**.

```bash
pvmove -A y /dev/sdb1      # di chuyển toàn bộ data từ /dev/sdb1 sang PV khác còn trống trong cùng VG
vgreduce vg01 /dev/sdb1    # loại /dev/sdb1 khỏi VG sau khi đã trống
```
- `-A y` = tự động backup metadata VG (dùng `vgcfgbackup`) sau khi thay đổi.

> ⚠️ **Luôn backup dữ liệu trước khi `pvmove`** — nếu mất điện đột ngột giữa chừng, VG có thể rơi vào trạng thái **không nhất quán**, gây mất dữ liệu.

### Gỡ bỏ hoàn toàn LVM Component

**Thứ tự bắt buộc: LV → VG → PV** (từ trong ra ngoài)

```bash
umount /mnt/data                        # 1. unmount FS, xóa entry /etc/fstab liên quan
lvremove /dev/vg01/lv01                 # 2. xóa LV (có hỏi xác nhận y/n)
vgremove vg01                           # 3. xóa VG
pvremove /dev/sdb2 /dev/sdb3            # 4. xóa PV (xóa metadata LVM khỏi partition)
```

> ⚠️ **`lvremove`, `vgremove`, `pvremove` là thao tác PHÁ HỦY và KHÔNG THỂ HOÀN TÁC** — chỉ chạy khi chắc chắn 100% không còn cần dữ liệu.

---

## 8. Bảng so sánh nhanh: Khi nào dùng lệnh nào?

| Việc cần làm | Lệnh |
|---|---|
| Đánh dấu 1 partition sẵn sàng làm PV | `pvcreate` |
| Gộp PV thành pool lưu trữ | `vgcreate` |
| Cấp phát dung lượng thành ổ dùng được | `lvcreate` |
| Format ổ logic | `mkfs -t xfs/ext4` |
| Mở rộng LV | `lvextend -L +SIZE` |
| Mở rộng FS trên LV (XFS) | `xfs_growfs MOUNT_POINT` |
| Mở rộng FS trên LV (ext4) | `resize2fs DEVICE` |
| Mở rộng LV + FS cùng lúc | `lvextend -r` |
| Thêm PV mới vào VG có sẵn | `vgextend` |
| Di chuyển data ra khỏi 1 PV | `pvmove` |
| Loại PV khỏi VG | `vgreduce` |
| Xóa LV / VG / PV | `lvremove` / `vgremove` / `pvremove` |
| Xem thông tin chi tiết | `pvdisplay` / `vgdisplay` / `lvdisplay` |
| Xem thông tin tóm tắt | `pvs` / `vgs` / `lvs` |

## 9. Các quy tắc "vàng" cần nhớ

1. ✅ Luồng LVM luôn theo thứ tự: **PV → VG → LV → Format → Mount**.
2. ✅ Một PV chỉ thuộc **1 VG duy nhất**.
3. ⚠️ `xfs_growfs` dùng **mount point**, `resize2fs` dùng **device name** — dễ nhầm lẫn.
4. ⚠️ XFS **chỉ resize lên được**; ext4 resize được **cả 2 chiều**.
5. ⚠️ Mở rộng swap LV **bắt buộc offline** (`swapoff` trước) — khác hoàn toàn quy trình FS thường.
6. ✅ Luôn **backup trước khi `pvmove`** để tránh mất dữ liệu nếu có sự cố giữa chừng.
7. ⚠️ `lvremove`/`vgremove`/`pvremove` **không thể hoàn tác** — luôn unmount và xác nhận kỹ trước khi chạy.
8. ✅ Thứ tự xóa: LV trước, rồi VG, cuối cùng mới PV.
9. ✅ `+` trong `lvextend -L +500M` = cộng thêm; không có `+` = size tuyệt đối cuối cùng.