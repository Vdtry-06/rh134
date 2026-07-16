# Tổng Hợp: Lập Lịch Tác Vụ Hệ Thống trên RHEL 10

> Tài liệu tổng hợp Chương 4 – Scheduling System Tasks. Gồm 3 công cụ chính: **Systemd Timer**, **Systemd-tmpfiles**, và **Cron**.

---

## 1. Tổng quan chung

| Công cụ | Dùng để làm gì | Thay thế cho |
|---|---|---|
| **Systemd Timer** | Chạy service theo lịch (định kỳ hoặc theo mốc thời gian) | Cron (từ RHEL 10 trở đi) |
| **systemd-tmpfiles** | Tự động tạo/xóa/thiết lập quyền cho file & thư mục tạm | Script dọn dẹp thủ công |
| **Cron / Anacron** | Chạy lệnh theo lịch cố định (phút/giờ/ngày...) | Vẫn dùng song song, đặc biệt cho job người dùng tự định nghĩa |

**Use case chung:** Tự động hóa các tác vụ bảo trì hệ thống lặp lại — dọn rác, thu thập số liệu, sao lưu, xoay vòng log, kiểm tra dịch vụ — mà không cần can thiệp thủ công.

---

## 2. Systemd Timer Unit

### Khái niệm
- File cấu hình kết thúc bằng `.timer`, thường **kích hoạt một service cùng tên**.
- Từ RHEL 10, systemd timer **thay thế cron** cho phần lớn tác vụ hệ thống định kỳ.
- Log sự kiện được ghi vào journal → dễ debug hơn cron truyền thống.

### Các lệnh cốt lõi

| Mục đích | Lệnh |
|---|---|
| Liệt kê timer đang chạy | `systemctl list-units -t timer` |
| Liệt kê tất cả timer đã cài (kể cả disabled) | `systemctl list-unit-files -t timer` |
| Bắt đầu ngay | `systemctl start TÊN.timer` |
| Dừng | `systemctl stop TÊN.timer` |
| Bật khi boot | `systemctl enable TÊN.timer` |
| Bật + chạy ngay | `systemctl enable --now TÊN.timer` |
| Tắt khi boot + dừng ngay | `systemctl disable --now TÊN.timer` |
| Xem trạng thái | `systemctl status TÊN.timer` |
| Nạp lại cấu hình sau khi sửa | `systemctl daemon-reload` |

### Cấu trúc file `.timer` (ví dụ `sysstat-collect.timer`)
```ini
[Unit]
Description=Run system activity accounting tool every 10 minutes

[Timer]
OnCalendar=*:00/10

[Install]
WantedBy=sysstat.service
```
- `OnCalendar=*:00/10` → chạy mỗi 10 phút.
- `OnUnitActiveSec=15min` → chạy 15 phút **sau lần chạy service gần nhất** (lịch tương đối, không phải lịch cố định).

### ⚠️ Quy tắc quan trọng khi chỉnh sửa
- **KHÔNG BAO GIỜ** sửa trực tiếp file trong `/usr/lib/systemd/system/` (sẽ bị package ghi đè khi update).
- Cách đúng: **copy** file từ `/usr/lib/systemd/system/` → `/etc/systemd/system/`, sửa file copy đó.
- File trong `/etc/systemd/system/` **luôn được ưu tiên** hơn file trùng tên trong `/usr/lib/systemd/system/`.
- Sau khi sửa: bắt buộc chạy `systemctl daemon-reload` rồi `systemctl enable --now`.

### 📝 Bài thực hành: Quản lý sysstat-collect.timer
**Mục tiêu:** Đổi tần suất thu thập số liệu hệ thống từ 10 phút → 2 phút.

Các bước chính:
1. `dnf install sysstat`
2. Xem các file: `sysstat-collect.timer`, `.service`, `sysstat-rotate.timer`, `sysstat-summary.timer`...
3. Copy: `cp /usr/lib/systemd/system/sysstat-collect.timer /etc/systemd/system/`
4. Sửa `OnCalendar=*:00/10` → `OnCalendar=*:00/2` trong file mới copy
5. `systemctl daemon-reload`
6. `systemctl enable --now sysstat-collect.timer`
7. Kiểm tra kết quả: file dữ liệu xuất hiện trong `/var/log/sa/` sau tối đa 2 phút

---

## 3. Quản lý File Tạm (systemd-tmpfiles)

### Khái niệm
- Công cụ `systemd-tmpfiles` đọc cấu hình từ 3 thư mục (theo thứ tự ưu tiên tăng dần):
  1. `/usr/lib/tmpfiles.d/*.conf` — do package cài đặt, **không sửa**
  2. `/run/tmpfiles.d/*.conf` — cấu hình tạm thời (RAM), daemon tự quản lý
  3. `/etc/tmpfiles.d/*.conf` — **nơi admin nên đặt cấu hình tùy chỉnh**, ưu tiên cao nhất
- Được kích hoạt định kỳ qua timer `systemd-tmpfiles-clean.timer`:
  ```ini
  [Timer]
  OnBootSec=15min       # chạy 15 phút sau khi boot
  OnUnitActiveSec=1d    # rồi lặp lại mỗi 24 giờ
  ```

### Cú pháp file cấu hình (`Type Path Mode UID GID Age Argument`)

| Type | Ý nghĩa |
|---|---|
| `d` | Tạo thư mục nếu chưa có; **không tự xóa nội dung bên trong** |
| `q` | Giống `d` nhưng dùng cho quota-subvolume (thường dùng cho `/tmp`) |
| `D` | Tạo thư mục; nếu đã tồn tại → **xóa hết nội dung bên trong** |
| `Z` | Khôi phục đệ quy: SELinux context, quyền, chủ sở hữu |
| `L` | Tạo symbolic link |

**Ví dụ:**
```
q /tmp 1777 root root 5d
```
→ Đảm bảo `/tmp` tồn tại, quyền 1777, chủ sở hữu root:root, xóa file không dùng >5 ngày.

```
d /run/momentary 0700 root root 30s
```
→ Tạo `/run/momentary`, quyền 0700, xóa file không dùng >30 giây.

### Lệnh thao tác

| Mục đích | Lệnh |
|---|---|
| Tạo file/thư mục theo cấu hình | `systemd-tmpfiles --create FILE.conf` |
| Dọn (xóa) file cũ theo tuổi quy định | `systemd-tmpfiles --clean FILE.conf` |

> ⚠️ Khi test cấu hình mới, chỉ nên chạy từng file `.conf` một để dễ kiểm soát.

### 📝 Bài thực hành: Quản lý file tạm
**Mục tiêu:** Cấu hình dọn `/tmp` sau 5 ngày, và tự tạo thư mục `/run/momentary` tự dọn sau 30 giây.

Các bước chính:
1. Copy `tmp.conf` từ `/usr/lib/tmpfiles.d/` → `/etc/tmpfiles.d/`, sửa tuổi thành `5d`
2. Kiểm chứng: `systemd-tmpfiles --clean /etc/tmpfiles.d/tmp.conf` → exit code `0` là đúng
3. Tạo mới `/etc/tmpfiles.d/momentary.conf` với nội dung `d /run/momentary 0700 root root 30s`
4. `systemd-tmpfiles --create ...` → kiểm tra thư mục được tạo đúng quyền bằng `ls -ld`
5. Tạo file test, `sleep 30`, chạy `--clean` → xác nhận file bị xóa

---

## 4. Cron & Anacron

### Cron hệ thống (khác cron người dùng)
- **Job hệ thống** nên đặt trong `/etc/cron.d/`, KHÔNG dùng lệnh `crontab -e` (đó là cho user).
- File `/etc/crontab` chỉ nên dùng để **tham khảo cú pháp**, không được sửa trực tiếp — để trống job thật sự.

### Cú pháp `/etc/cron.d/*`
```
# .---------------- phút (0-59)
# |  .------------- giờ (0-23)
# |  |  .---------- ngày trong tháng (1-31)
# |  |  |  .------- tháng (1-12)
# |  |  |  |  .---- thứ trong tuần (0-6, CN=0 hoặc 7)
# |  |  |  |  |
# *  *  *  *  * user  lệnh_cần_chạy
```
- Cron hệ thống có thêm **cột user** (khác cron user thường).
- `*/2` = bước nhảy 2 → "mỗi 2 đơn vị" (ví dụ mỗi 2 phút).

### Các thư mục script định kỳ
| Thư mục | Ý nghĩa |
|---|---|
| `/etc/cron.hourly/` | Script chạy mỗi giờ |
| `/etc/cron.daily/` | Script chạy mỗi ngày |
| `/etc/cron.weekly/` | Script chạy mỗi tuần |
| `/etc/cron.monthly/` | Script chạy mỗi tháng |

- Chứa **script thực thi được** (không phải file crontab) → nhớ `chmod +x`.
- Cơ chế: `/etc/cron.d/0hourly` chạy `run-parts /etc/cron.hourly` vào phút 01 mỗi giờ.

### Anacron — bổ sung cho cron
- Đảm bảo job **daily/weekly/monthly vẫn chạy** kể cả khi máy tắt/ngủ đông lúc đến giờ.
- Cấu hình: `/etc/anacrontab`, cú pháp 4 cột: `period_days | delay_phút | job_id | command`
- `START_HOURS_RANGE` giới hạn khung giờ chạy job.
- Cơ chế liên kết: `cron.hourly` → script `0anacron` → `anacron -s` → đọc `/etc/anacrontab` → chạy `cron.daily/weekly/monthly` (có delay ngẫu nhiên tối đa 45 phút để tránh dồn tải).

### 📝 Bài thực hành: Tạo job cron hệ thống
**Mục tiêu:** Ghi log số user đang hoạt động, mỗi 3 phút.

Các bước chính:
1. Tạo `/etc/cron.d/usercount`:
   ```
   SHELL=/bin/bash
   PATH=/sbin:/bin:/usr/sbin:/usr/bin
   MAILTO=root
   */3 * * * * root logger "There are `w -h | wc -l` active users"
   ```
2. Theo dõi log: `tail -f /var/log/messages | grep --line-buffered "There are"`
3. Xác nhận log xuất hiện đều đặn mỗi 3 phút

---

## 5. Bảng so sánh nhanh: Khi nào dùng gì?

| Tình huống | Công cụ nên dùng |
|---|---|
| Cần chạy service định kỳ, có tích hợp journal/log tốt | **Systemd Timer** |
| Cần đảm bảo thư mục/file tạm luôn tồn tại đúng quyền, tự dọn dẹp | **systemd-tmpfiles** |
| Cần lịch chạy đơn giản kiểu "phút/giờ/ngày cố định", dễ viết nhanh | **Cron (`/etc/cron.d/`)** |
| Job daily/weekly/monthly nhưng máy có thể tắt lúc đến giờ | **Anacron** |

## 6. Các quy tắc "vàng" cần nhớ

1. ❌ Không sửa file gốc trong `/usr/lib/systemd/system/` hay `/usr/lib/tmpfiles.d/` → luôn copy sang `/etc/...` rồi sửa.
2. ✅ Sau khi sửa timer unit: luôn `systemctl daemon-reload` rồi `enable --now`.
3. ❌ Không sửa trực tiếp `/etc/crontab` — chỉ dùng để tham khảo cú pháp.
4. ✅ Job cron hệ thống → đặt trong `/etc/cron.d/`, có thêm cột user.
5. ✅ Script trong `cron.hourly/daily/weekly/monthly` phải có quyền thực thi (`chmod +x`).