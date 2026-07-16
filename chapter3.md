# Tổng Hợp: Lập Lịch Tác Vụ Người Dùng trên RHEL 10

> Tài liệu tổng hợp Chương 3 – Scheduling User Tasks. Gồm 2 phần: **`at` (job chạy 1 lần trong tương lai)** và **`crontab` (job lặp lại định kỳ, cấp user)**.

---

## 1. Tổng quan chung

| Công cụ | Dùng khi nào |
|---|---|
| **`at`** | Chạy **1 lần duy nhất** vào 1 thời điểm cụ thể trong tương lai |
| **`crontab`** (user) | Chạy **lặp lại định kỳ** theo lịch (phút/giờ/ngày...) |

**Use case điển hình:**
- `at`: user muốn chạy tác vụ bảo trì dài vào lúc nửa đêm; admin đặt job "tự rollback cấu hình firewall sau 10 phút" để phòng hờ, rồi hủy job đó nếu cấu hình mới chạy tốt.
- `crontab`: backup định kỳ, gửi báo cáo hàng ngày, dọn dẹp log theo lịch.

---

## 2. Lệnh `at` — Job Chạy 1 Lần Trong Tương Lai

### Cơ chế
- Package `at` cung cấp daemon **`atd`** (bật mặc định) + lệnh `at`, `atq`.
- Mọi user đều dùng được `at` để đưa job vào hàng đợi.
- Có **hàng đợi (queue)** từ `a-z` và `A-Z` — **chữ cao hơn = ưu tiên cao hơn**.

### Tạo job

**Cách 1: Nhập lệnh trực tiếp (tương tác)**
```bash
at TIMESPEC
# Nhập lệnh, kết thúc bằng Ctrl+D trên dòng trống
```

**Cách 2: Đọc từ file/script**
```bash
at now +5min < myfile
```

### Cú pháp thời gian (`TIMESPEC`)
Chấp nhận ngôn ngữ tự nhiên: `02:00pm`, `15:59`, `midnight`, `teatime` (= 16:00), kèm ngày/số ngày tùy chọn.

| Ví dụ | Ý nghĩa |
|---|---|
| `now +5min` | 5 phút nữa |
| `teatime tomorrow` | 16:00 ngày mai |
| `noon +4 days` | 12:00 trưa, 4 ngày sau |
| `5pm august 3 2025` | Giờ + ngày cụ thể |

> 💡 Quy tắc khi thiếu 1 phần: có **ngày** không có **giờ** → giờ mặc định = giờ hiện tại. Có **giờ** không có **ngày** → job chạy vào **lần khớp giờ đó tiếp theo** (hôm nay nếu chưa qua giờ đó, hoặc ngày mai nếu đã qua).

### Xem/Quản lý job đang chờ

```bash
atq              # hoặc: at -l
```
Output mẫu:
```
28  Mon May 19 05:13:00 2025 a user
```
- `28` = số job
- Ngày giờ chạy
- `a` = queue
- `user` = chủ sở hữu

> ⚠️ User thường chỉ xem/quản lý được **job của chính mình**; **root xem/quản lý được của mọi user**.

### Xem nội dung lệnh sẽ chạy
```bash
at -c JOBNUMBER
```

### Xóa job
```bash
atrm JOBNUMBER
```

### Chỉ định queue cụ thể
```bash
at -q g teatime      # đưa job vào queue g, chạy lúc teatime
```

---

## 3. 📝 Bài thực hành: Lập lịch Job Tương lai (`at`)

**Mục tiêu:** Tạo, kiểm tra, và xóa job dùng `at`.

1. Tạo job chạy sau 2 phút, ghi output vào file:
   ```bash
   echo "date >> /home/student/myjob.txt" | at now +2min
   ```
2. `atq` → xem job đang chờ
3. Theo dõi real-time: `watch atq` (tự refresh mỗi 2 giây) → `Ctrl+C` khi job đã chạy xong (biến mất khỏi queue)
4. `cat myjob.txt` → xác nhận nội dung đúng với thời điểm job chạy
5. Tạo job tương tác vào **queue `g`**, chạy lúc teatime (16:00):
   ```bash
   at -q g teatime
   at> echo "It's teatime" >> /home/student/tea.txt
   at> Ctrl+D
   ```
6. Tạo job tương tự vào **queue `b`**, chạy lúc 16:05
7. `atq` → xem cả 2 job
8. `at -c 2` và `at -c 3` → xem nội dung lệnh mỗi job
9. `atrm 2` → xóa job teatime
10. `atq` → xác nhận chỉ còn job 16:05

---

## 4. Cron — Job Lặp Lại Định Kỳ (cấp User)

### Cơ chế
- Daemon **`crond`** (bật mặc định) đọc file cấu hình **riêng cho từng user**.
- Nếu job **không redirect output**, `crond` sẽ **gửi email** kết quả/lỗi cho chủ job (cần cấu hình mail server/SMTP relay để hoạt động thật sự).

### Quản lý crontab của user

| Lệnh | Ý nghĩa |
|---|---|
| `crontab -l` | Liệt kê job hiện có |
| `crontab -e` | Sửa job (mở editor, mặc định `vim` trừ khi đổi biến `$EDITOR`) |
| `crontab -r` | **Xóa hết** job của user hiện tại |
| `crontab filename` | Thay thế toàn bộ job bằng nội dung từ file |
| `crontab -u USER ...` | (chỉ root) quản lý job cho **user khác** |

> ⚠️ **Không nên** dùng `crontab` với quyền root để tạo job cá nhân — có rủi ro bị khai thác nếu job đó chạy với quyền root. `crontab` **không dùng để quản lý job hệ thống** (đó là việc của `/etc/cron.d/`, xem Chương 4).

### Cấu trúc file crontab

```
# Comment bắt đầu bằng #
NAME=value              # biến môi trường, áp dụng cho các dòng SAU nó

phút giờ ngày-tháng tháng thứ-trong-tuần  lệnh
```

**Biến môi trường phổ biến:** `SHELL` (shell diễn giải dòng lệnh), `MAILTO` (ai nhận email kết quả).

### Cú pháp 5 trường thời gian

| Ký hiệu | Ý nghĩa |
|---|---|
| `*` | Mọi giá trị hợp lệ |
| Số cụ thể | Đúng giá trị đó |
| `x-y` | Khoảng (bao gồm cả x và y) |
| `x,y,z` | Danh sách (có thể kết hợp cả range) |
| `*/x` | Bước nhảy x đơn vị |
| Viết tắt 3 chữ | `Jan`, `Feb`, `Mon`, `Tue`... |

> 💡 Ngày trong tuần: `0`=Chủ nhật, `1`=Thứ 2,... `7` cũng = Chủ nhật.

> ⚠️ **Quy tắc quan trọng cho range:** số/tên đầu phải **≤** số/tên sau. `Tue-Fri` (2-5) hợp lệ; `Fri-Tue` (5-2) **KHÔNG hợp lệ** — phải viết thành 2 range: `Sun-Tue,Fri-Sat` (0-2,5-6).

> 💡 **Cả `ngày-tháng` VÀ `thứ-trong-tuần` khác `*`:** job chạy khi khớp **1 trong 2 điều kiện** (OR logic), không phải AND.

> 💡 Dấu `%` chưa escape trong command field = xuống dòng — nội dung sau `%` được truyền vào **STDIN** cho lệnh.

### Ví dụ minh họa

```bash
# Chạy backup 09:00 ngày 3/2 hàng năm
0 9 3 2 * /usr/local/bin/yearly_backup

# Mỗi 5 phút, từ 9h-16h59, thứ 6, tháng 7 → gửi email "Chime"
*/5 9-16 * Jul 5 echo "Chime"

# Report hàng ngày, Thứ 2-6, lúc 23:58
58 23 * * 1-5 /usr/local/bin/daily_report

# Gửi mail check-in Thứ 2-6 lúc 9h, dùng % để xuống dòng làm nội dung mail
0 9 * * 1-5 mutt -s "Checking in" developer@example.com % Hi there, just checking in.
```

> 💡 Trong ví dụ `*/5 9-16`: range giờ 9-16 nghĩa là **09:00 đến 16:59** — lần chạy cuối là **16:55** (vì 16:55+5=17:00, đã ngoài phạm vi).

---

## 5. 📝 Bài thực hành: Lập lịch Job Lặp lại (Cron User)

**Mục tiêu:** Tạo cron job ghi ngày giờ mỗi 2 phút, chỉ chạy trong khoảng "hôm qua → hôm nay → ngày mai".

1. Xem ngày hiện tại: `date` → ví dụ `Wed`
2. Tính khoảng ngày: `date -d "last day" +%a` → `Tue`; `date -d "next day" +%a` → `Thu`
3. `crontab -e`, thêm dòng:
   ```
   */2 * * * Tue-Thu /usr/bin/date >> /home/student/my_first_cron_job.txt
   ```
4. Lưu → thấy thông báo `crontab: installing new crontab`
5. `crontab -l` → xác nhận nội dung đã lưu đúng
6. Chờ job chạy bằng vòng lặp:
   ```bash
   while ! test -f my_first_cron_job.txt; do sleep 1s; done
   ```
7. `cat my_first_cron_job.txt` → xác nhận nội dung khớp thời điểm job chạy
8. `crontab -r` → xóa hết job
9. `crontab -l` → xác nhận `no crontab for student`

---

## 6. Bảng so sánh nhanh: `at` vs `crontab`

| Tiêu chí | `at` | `crontab` (user) |
|---|---|---|
| Số lần chạy | **1 lần** | **Lặp lại** theo lịch |
| Daemon | `atd` | `crond` |
| Xem job | `atq` / `at -l` | `crontab -l` |
| Sửa/thêm job | `at TIMESPEC` (mỗi lần 1 job mới) | `crontab -e` (sửa toàn bộ file job) |
| Xóa 1 job | `atrm JOBNUMBER` | Sửa lại bằng `crontab -e` (xóa dòng) |
| Xóa tất cả | (xóa từng job bằng `atrm`) | `crontab -r` |
| Có nhiều hàng đợi ưu tiên | ✅ (a-z, A-Z) | ❌ |

## 7. Các quy tắc "vàng" cần nhớ

1. ✅ User thường chỉ quản lý job **của chính mình** với cả `at` và `crontab`; root quản lý được của mọi user.
2. ⚠️ Không nên chạy `crontab` với quyền root để tạo job cá nhân — dùng `/etc/cron.d/` cho job hệ thống thay vào đó (xem Chương 4).
3. ✅ `at` có **hàng đợi ưu tiên** (a-z, A-Z) — chữ cao hơn chạy ưu tiên hơn.
4. ⚠️ Range trong cron **phải tăng dần** (`Tue-Fri` OK, `Fri-Tue` SAI) — dùng danh sách nhiều range nếu cần "vòng qua" cuối tuần.
5. ⚠️ Khi **cả** `ngày-tháng` và `thứ-trong-tuần` khác `*` → job chạy nếu khớp **1 trong 2**, không phải cả 2 (OR, không phải AND).
6. ✅ Job cron không redirect output sẽ được **gửi email** cho chủ job — cần cấu hình mail server để nhận được thật sự.
7. ✅ Dùng `at -c` / xem `crontab -l` để kiểm tra lại nội dung job trước khi tin tưởng nó sẽ chạy đúng.
8. ✅ `crontab filename` (không có `-e`) sẽ **thay thế toàn bộ** job hiện có bằng nội dung file mới — cẩn thận mất job cũ.