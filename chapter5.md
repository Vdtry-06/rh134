# Tổng Hợp: Phân Tích và Lưu Trữ Log trên RHEL 10

> Tài liệu tổng hợp Chương 5 – Analyzing and Storing Logs. Gồm 3 phần: **Kiến trúc log hệ thống (Rsyslog + Journal)**, **Journal bền vững (persistent)**, và **Đồng bộ thời gian (NTP/Chrony)**.

---

## 1. Kiến trúc Log Hệ thống

### Hai thành phần cốt lõi

| Thành phần | Vai trò |
|---|---|
| **systemd-journald** | Trung tâm thu thập log: kernel, quá trình boot, stdout/stderr của daemon, sự kiện syslog. Ghi vào **journal nhị phân, có cấu trúc, có index**. Mặc định **không bền vững** (mất khi reboot). |
| **rsyslog** | Đọc log từ journal (qua module `imjournal`, **không đọc trực tiếp `/dev/log`**), phân loại rồi ghi ra file text trong `/var/log/` — **bền vững qua reboot**. |

> 💡 Ghi nhớ luồng: `Chương trình → /dev/log (socket) → systemd-journald → journal nhị phân → rsyslog đọc & ghi ra /var/log/*`

### Các file log quan trọng trong `/var/log/`

| File | Nội dung |
|---|---|
| `/var/log/messages` | Hầu hết log syslog (trừ auth, mail, cron, debug) |
| `/var/log/secure` | Sự kiện bảo mật, xác thực |
| `/var/log/maillog` | Log mail server |
| `/var/log/cron` | Log job định kỳ |
| `/var/log/boot.log` | Log khởi động (từ Plymouth + facility `local7`) |

**Use case:** Đây là bước đầu tiên khi troubleshoot — biết log nào chứa loại thông tin gì để tra cứu nhanh.

---

## 2. Rsyslog — Diễn giải & Quản lý Sự kiện Syslog

### Facility (nguồn gốc) và Priority (mức độ nghiêm trọng)

**Facility phổ biến:** `kern`, `user`, `mail`, `daemon`, `auth`, `cron`, `authpriv`, `local0-7` (tùy chỉnh)

**Priority (từ cao → thấp):** `emerg` > `alert` > `crit` > `err` > `warning` > `notice` > `info` > `debug`

> Khi chỉ định priority trong rule, **tất cả message ở mức đó hoặc cao hơn** đều bị bắt (không phải chỉ đúng mức đó).

### Cấu trúc rule trong `/etc/rsyslog.conf` / `/etc/rsyslog.d/*.conf`
```
facility.priority    action(thường là đường dẫn file)
```
Ví dụ:
```
authpriv.*      /var/log/secure          # mọi priority của authpriv
authpriv.warning /var/log/secure         # warning trở lên
*.emerg         :omusrmsg:*              # gửi tới terminal mọi user đang login
*.info;mail.none;authpriv.none;cron.none  /var/log/messages   # loại trừ 1 số facility
```
- `*` = wildcard (mọi facility/priority)
- `none` = loại trừ facility đó khỏi rule

### Xoay vòng log (`logrotate`)
- Được kích hoạt bởi **systemd timer**, chạy `logrotate` hằng ngày.
- File cũ đổi tên kèm ngày (vd `messages-20250620`), file mới tạo lại.
- Mặc định xoay **hằng tuần**, giữ khoảng **4 tuần** rồi xóa.

### Cấu trúc 1 dòng log
```
Mar 20 20:11:48 localhost sshd[1433]: Failed password for student ...
[Thời gian]      [Host]   [Chương trình[PID]]: [Nội dung]
```

### Lệnh hữu ích

| Mục đích | Lệnh |
|---|---|
| Theo dõi log real-time | `tail -f /var/log/secure` |
| Gửi log thủ công để test | `logger -p local7.notice "message"` |
| Restart rsyslog sau khi sửa config | `systemctl restart rsyslog` |

### 📝 Bài thực hành: Cấu hình ghi log mức debug
**Mục tiêu:** Mọi log ưu tiên `debug` trở lên → ghi vào `/var/log/messages-debug`

1. Tạo `/etc/rsyslog.d/debug.conf`:
   ```
   *.debug /var/log/messages-debug
   ```
2. `systemctl restart rsyslog`
3. Test: `logger -p user.debug "Debug Message Test"`
4. Kiểm tra: `tail /var/log/messages-debug` → thấy dòng "Debug Message Test"

---

## 3. Tìm và Diễn giải Journal (journalctl)

### Đặc điểm journal
- Lưu tại `/run/log/journal` (RAM, **mất khi reboot** — trừ khi cấu hình bền vững, xem phần 4).
- Có nhiều field bổ sung (facility, priority, PID, UID, unit...) mà file text syslog không có.

### Các tùy chọn `journalctl` cốt lõi

| Tùy chọn | Ý nghĩa |
|---|---|
| `journalctl` | Xem toàn bộ log (từ cũ → mới) |
| `-n 5` | 5 dòng log mới nhất |
| `-f` | Theo dõi real-time (giống `tail -f`) |
| `-p err` | Log mức `err` trở lên |
| `-u sshd.service` | Log của 1 unit cụ thể |
| `--since "2025-06-01 20:30"` | Từ mốc thời gian |
| `--until "2025-06-04 10:00"` | Đến mốc thời gian |
| `--since today` / `--since "-1 hour"` | Mốc tương đối |
| `-o verbose` | Xem đầy đủ mọi field của mỗi entry |
| `_PID=1234` / `_UID=81` / `_SYSTEMD_UNIT=sshd.service` | Lọc theo field cụ thể (có thể kết hợp nhiều field) |

**Use case:** Đây là công cụ debug chính khi cần tìm nguyên nhân sự cố — thu hẹp phạm vi tìm kiếm càng chi tiết càng tốt (theo thời gian + theo unit/PID/UID) để tránh "ngập" trong log.

### 📝 Bài thực hành: Tìm log journal
Thực hiện lần lượt các truy vấn:
- `journalctl _PID=1` — log của tiến trình systemd (PID 1)
- `journalctl _UID=81` — log theo UID cụ thể
- `journalctl -p warning` — log warning trở lên
- `journalctl --since "-10min"` — log 10 phút gần nhất
- `journalctl --since 9:00:00 _SYSTEMD_UNIT="sshd.service"` — kết hợp thời gian + unit

---

## 4. Cấu hình Journal Bền vững (Persistent)

### Vấn đề
Mặc định journal ở `/run/log/journal` (tmpfs) → **mất sạch sau reboot**.

### Cách cấu hình bền vững

| Bước | Lệnh |
|---|---|
| 1. Tạo thư mục lưu trữ bền vững | `mkdir /var/log/journal` |
| 2. Đẩy dữ liệu journal hiện tại vào đó | `journalctl --flush` |
| 3. Kiểm tra | `ls /var/log/journal` → thấy thư mục hex + `system.journal`, `user-*.journal` |

> Cơ chế: `Storage=auto` (mặc định) — nếu `/var/log/journal` **tồn tại** thì tự động bật lưu bền vững; nếu không, dùng volatile ở `/run/log/journal`.

### Xem log theo lần boot

| Lệnh | Ý nghĩa |
|---|---|
| `journalctl --list-boots` | Liệt kê các lần boot đã ghi nhận |
| `journalctl -b` | Log của lần boot hiện tại |
| `journalctl -b -1` | Log của lần boot **trước đó** (hữu ích khi debug crash) |
| `journalctl -b 1` | Log của lần boot đầu tiên được ghi nhận |

### Giới hạn dung lượng & xoay vòng journal
- Mặc định: journal **không vượt quá 10%** dung lượng filesystem, luôn chừa lại **≥15%** trống. Xoay vòng hàng tháng.
- Cấu hình tại `/etc/systemd/journald.conf` (⚠️ từ RHEL 10, file này **không có sẵn** — phải copy từ `/usr/lib/systemd/journald.conf` sang `/etc/systemd/` rồi chỉnh sửa, **không sửa file gốc**).

| Tham số | Ý nghĩa |
|---|---|
| `SystemMaxUse` / `RuntimeMaxUse` | Dung lượng tối đa journal được dùng (mặc định 10%, cap 4GB) |
| `SystemMaxFileSize` / `RuntimeMaxFileSize` | Kích thước tối đa 1 file trước khi xoay (mặc định 1/8 MaxUse, cap 128M/4GB) |
| `SystemKeepFree` / `RuntimeKeepFree` | Dung lượng tối thiểu phải để trống (mặc định 15%, cap 4GB) |

- `System*` → áp dụng cho journal bền vững (`/var/log/journal`)
- `Runtime*` → áp dụng cho journal tạm thời (`/run/log/journal`)
- Sau khi sửa: `systemctl restart systemd-journald`

### 📝 Bài thực hành: Cấu hình journal bền vững
1. Xác nhận `/var/log/journal` **chưa tồn tại**
2. `mkdir /var/log/journal`
3. `journalctl --flush`
4. `systemctl reboot` — khởi động lại máy
5. Sau khi máy lên lại: kiểm tra `/var/log/journal/<hex>/` có `system.journal`, `user-1000.journal` → xác nhận log đã được giữ lại qua reboot

---

## 5. Đồng bộ Thời gian (NTP / Chrony)

### Vì sao quan trọng?
Timestamp chính xác là **bắt buộc** để đối chiếu log giữa nhiều máy khi troubleshoot.

### Lệnh `timedatectl` — quản lý giờ & múi giờ

| Mục đích | Lệnh |
|---|---|
| Xem trạng thái giờ hệ thống | `timedatectl` |
| Liệt kê danh sách múi giờ | `timedatectl list-timezones` |
| Đổi múi giờ | `timedatectl set-timezone America/Phoenix` |
| Đặt giờ thủ công | `timedatectl set-time 9:00:00` |
| Bật/tắt đồng bộ NTP | `timedatectl set-ntp true\|false` |

> ⚠️ Muốn `set-time` thủ công phải **tắt NTP trước** (`set-ntp false`), nếu không sẽ báo lỗi "Automatic time synchronization is enabled".

- `tzselect`: công cụ hỏi-đáp tương tác để **xác định tên múi giờ đúng** (không tự thay đổi hệ thống, chỉ gợi ý).

### Chronyd — dịch vụ đồng bộ NTP

- Đồng hồ phần cứng (RTC) không chính xác → `chronyd` đồng bộ với NTP server.
- **Stratum**: khoảng cách (số hop) tới nguồn giờ chuẩn — Stratum 0 = đồng hồ tham chiếu, Stratum 1 = server gắn trực tiếp, Stratum 2 = client của stratum 1, v.v.
- Cấu hình tại `/etc/chrony.conf`:
  ```
  pool 2.rhel.pool.ntp.org iburst   # nhiều server từ NTP Pool Project
  server ntp.example.com iburst     # 1 server cụ thể
  ```
  - `iburst`: đo nhanh 4-8 lần để đồng bộ nhanh hơn lúc khởi động.
  - Nên cấu hình **≥3 nguồn** để loại trừ nguồn lỗi và giảm sai số do độ trễ mạng.
- Sau khi sửa: `systemctl restart chronyd`

### Kiểm tra trạng thái đồng bộ

```
chronyc sources -v
```

| Ký hiệu (cột MS) | Ý nghĩa |
|---|---|
| `*` | Nguồn tốt nhất hiện tại, đang dùng để đồng bộ |
| `+` | Cũng đang góp phần đồng bộ |
| `-` | Khả dụng nhưng không được chọn |
| `x` / `~` | Có vẻ sai / dao động quá nhiều |
| `?` | Không dùng được (cần vài lần poll mới ổn định) |

- Dùng `chronyc -n sources` để tắt resolve DNS, hiện IP đầy đủ (không bị cắt bớt IPv6).

### 📝 Bài thực hành: Đồng bộ thời gian
**Kịch bản:** Máy chủ "chuyển" sang Haiti → cần đổi múi giờ + đồng bộ NTP với `classroom.example.com`

1. `tzselect` → chọn Americas → Haiti → xác nhận `America/Port-au-Prince`
2. `timedatectl set-timezone America/Port-au-Prince`
3. Kiểm tra bằng `timedatectl`
4. Sửa `/etc/chrony.conf`, thêm dòng:
   ```
   server classroom.example.com iburst
   ```
5. `timedatectl set-ntp true`
6. Kiểm tra `timedatectl` → `System clock synchronized: yes`
7. Xác nhận nguồn đang dùng: `chronyc sources -v` → thấy `*` bên cạnh `classroom.example.com`

---

## 6. Bảng so sánh nhanh: Khi nào dùng gì?

| Tình huống | Công cụ nên dùng |
|---|---|
| Cần xem log text đơn giản, dễ `grep`/`tail` | File trong `/var/log/` (qua rsyslog) |
| Cần lọc chi tiết theo PID/UID/unit/thời gian, có cấu trúc | `journalctl` |
| Cần log tồn tại qua reboot để debug sự cố quá khứ | Cấu hình journal bền vững (`/var/log/journal`) |
| Cần định tuyến 1 loại message tới file riêng | Rule trong `/etc/rsyslog.d/*.conf` |
| Cần đối chiếu thời gian log giữa nhiều máy | Đồng bộ NTP qua `chronyd` |

## 7. Các quy tắc "vàng" cần nhớ

1. ✅ Rsyslog đọc log từ **journal** (qua `imjournal`), không đọc trực tiếp `/dev/log`.
2. ✅ `journalctl -p PRIORITY` lấy log ở mức đó **và cao hơn**.
3. ✅ Luôn thu hẹp phạm vi `journalctl` (theo `--since`, `-u`, field) để tránh ngập thông tin.
4. ❌ Không sửa `/usr/lib/systemd/journald.conf` trực tiếp → copy sang `/etc/systemd/journald.conf`.
5. ✅ Muốn journal sống sót qua reboot: `mkdir /var/log/journal` + `journalctl --flush`.
6. ✅ Muốn `set-time` thủ công → phải `set-ntp false` trước.
7. ✅ Nên cấu hình ít nhất 3 nguồn NTP để đồng bộ chính xác và có dự phòng.