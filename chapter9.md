# Tổng Hợp: Tối Ưu Hiệu Năng Hệ Thống trên RHEL 10

> Tài liệu tổng hợp Chương 9 – Tuning System Performance. Gồm 2 phần: **TuneD (tuning profile)** và **Điều chỉnh lịch trình tiến trình (nice/renice)**.

---

## 1. TuneD — Công cụ Tinh chỉnh Hiệu năng Hệ thống

### Khái niệm
- **TuneD** là daemon chạy nền, tự động áp dụng **profile tinh chỉnh** (tuning profile) phù hợp với loại workload (server hiệu năng cao, laptop tiết kiệm pin...).
- 2 chế độ hoạt động:

| Chế độ | Đặc điểm |
|---|---|
| **Static tuning** (mặc định) | Áp dụng cấu hình **1 lần** khi service khởi động / đổi profile → ổn định, dễ đoán |
| **Dynamic tuning** (tắt mặc định) | Liên tục **giám sát** hoạt động hệ thống (CPU, I/O, mạng...) và **tự điều chỉnh** theo thời gian thực qua *monitor plug-ins* + *tuning plug-ins* |

### Bật dynamic tuning
Sửa file `/etc/tuned/tuned-main.conf`:
```ini
dynamic_tuning = 1     # 0 = tắt (mặc định), 1 = bật
update_interval = 10   # tần suất cập nhật (giây)
```

### Cài đặt & bật dịch vụ
```bash
dnf install tuned
systemctl enable --now tuned
```

### Các profile có sẵn (RHEL 10)

| Profile | Mục đích |
|---|---|
| `balanced` | Cân bằng, không chuyên biệt (mặc định phổ biến) |
| `balanced-battery` | Cân bằng, thiên về tiết kiệm pin |
| `powersave` | Tiết kiệm điện tối đa |
| `desktop` | Tối ưu cho máy bàn |
| `throughput-performance` | Hiệu năng tốt cho nhiều loại server phổ biến |
| `latency-performance` | Ưu tiên độ trễ thấp, đổi lấy tiêu thụ điện cao hơn |
| `network-latency` | Như trên, tập trung vào mạng độ trễ thấp |
| `network-throughput` | Tối ưu băng thông mạng (CPU cũ hoặc mạng ≥40G) |
| `virtual-guest` | Tối ưu chạy **bên trong** máy ảo |
| `virtual-host` | Tối ưu máy chủ **chạy KVM guest** |
| `hpc-compute` | Tính toán hiệu năng cao (HPC) |
| `aws` | Tối ưu cho instance AWS EC2 |
| `accelerator-performance` | Tăng hiệu năng dựa trên accelerator |
| `optimize-serial-console` | Tối ưu khi dùng serial console |
| `intel-sst` | Cấu hình Intel Speed Select |

### Cấu trúc file profile (`tuned.conf`)
Lưu tại `/usr/lib/tuned/profiles/<tên-profile>/tuned.conf`. Ví dụ `virtual-guest`:
```ini
[main]
summary=Optimize for running inside a virtual guest
include=throughput-performance   # kế thừa từ profile khác

[vm]
dirty_ratio = 30

[sysctl]
vm.swappiness = 30
```
- `include=` → cơ chế **kế thừa profile** (profile con dùng lại toàn bộ setting của profile cha, rồi override phần cần thiết).
- `[sysctl]` → chỉnh tham số kernel qua tuning plug-in tương ứng (vd `vm.swappiness`).

### ⚠️ Quy tắc chỉnh sửa profile
- **KHÔNG** sửa trực tiếp file trong `/usr/lib/tuned/profiles/` (profile hệ thống/factory).
- Muốn tùy chỉnh: **copy** thư mục profile từ `/usr/lib/tuned/profiles/` → `/etc/tuned/profiles/`, rồi sửa bản copy.
- File trong `/etc/tuned/profiles/` **luôn được ưu tiên** hơn file trùng tên trong `/usr/lib/tuned/profiles/`.

### Các lệnh `tuned-adm` cốt lõi

| Lệnh | Ý nghĩa |
|---|---|
| `tuned-adm active` | Xem profile đang active |
| `tuned-adm list` | Liệt kê tất cả profile có sẵn |
| `tuned-adm profile_info TÊN` | Xem chi tiết 1 profile (bỏ trống = xem profile hiện tại) |
| `tuned-adm profile TÊN` | Chuyển sang profile khác (**bền vững**, giữ qua reboot) |
| `tuned-adm recommend` | Gợi ý profile phù hợp nhất dựa trên đặc điểm hệ thống |
| `tuned-adm verify` | Kiểm tra setting hệ thống hiện tại có khớp với profile active không |
| `tuned-adm off` | Tắt hoàn toàn TuneD, không áp dụng tinh chỉnh nào |

### Kiểm tra tham số kernel thực tế
```bash
sysctl vm.swappiness       # xem giá trị hiện tại
```

### Quản lý qua Web Console (Cockpit)
```bash
dnf install cockpit
systemctl enable --now cockpit.socket
firewall-cmd --add-service=cockpit --permanent
firewall-cmd --reload
```
Truy cập: `https://servername:9090` → menu **Overview** → mục **Performance profile** để xem/đổi profile trực quan.

### 📝 Bài thực hành: Đặt Tuning Profile
**Mục tiêu:** Kiểm tra profile đang dùng (`virtual-guest`), rồi chuyển sang `latency-performance` và xác nhận thay đổi tham số kernel thực tế.

1. Kiểm tra tuned đã cài & đang chạy: `dnf list tuned`, `systemctl is-enabled tuned`, `systemctl is-active tuned`
2. `tuned-adm list` → xác nhận profile active hiện tại: `virtual-guest`
3. Xem file cấu hình: `cat /usr/lib/tuned/profiles/virtual-guest/tuned.conf` → `dirty_ratio=30`, `vm.swappiness=30`
4. `tuned-adm verify` → xác nhận setting khớp profile
5. Kiểm tra thực tế: `sysctl vm.dirty_ratio` và `sysctl vm.swappiness` → đều = 30
6. Xem file `latency-performance`: `dirty_ratio=10`, `vm.swappiness=10`
7. Chuyển profile: `sudo tuned-adm profile latency-performance`
8. Xác nhận: `tuned-adm active` → `latency-performance`
9. Kiểm tra lại `sysctl vm.dirty_ratio` và `sysctl vm.swappiness` → đều = 10 (đã áp dụng thành công)

---

## 2. Điều chỉnh Lịch trình Tiến trình (Process Scheduling)

### Khái niệm nền tảng
- CPU đa nhân vẫn có thể **bão hòa** khi số luồng vượt quá khả năng xử lý → cần **scheduler** (bộ lập lịch trong kernel) phân chia thời gian CPU.
- **Preemption**: kernel chủ động ngắt 1 tiến trình đang chạy để nhường CPU cho tiến trình khác → tránh 1 chương trình "chiếm dụng" CPU làm treo máy.

### Chính sách lập lịch (Scheduling Policy)

| Chính sách | Loại | Ghi chú |
|---|---|---|
| `SCHED_NORMAL` (= `SCHED_OTHER`) | Tiến trình thường | Mặc định cho hầu hết ứng dụng |
| `SCHED_FIFO` | Real-time | First-in-first-out |
| `SCHED_RR` | Real-time | Round-robin |

> Từ RHEL 10: thuật toán cho `SCHED_NORMAL` là **EEVDF** (Earliest Eligible Virtual Deadline First), thay thế **CFS** (Completely Fair Scheduler) ở các bản cũ. EEVDF tính "deadline ảo" cho mỗi tiến trình dựa trên thời gian CPU nó "được nợ" + độ ưu tiên → tiến trình có deadline sớm nhất được chạy trước → cải thiện độ trễ & phản hồi.

### Nice Value — công cụ người dùng điều chỉnh ưu tiên

- Chỉ áp dụng cho tiến trình `SCHED_NORMAL` (**không** ảnh hưởng real-time tasks — real-time luôn ưu tiên cao hơn).
- Thang giá trị: **-20** (ưu tiên cao nhất, "ít nhường nhịn nhất") → **+19** (ưu tiên thấp nhất, "nhường nhịn nhiều nhất"). Mặc định = **0**.
- Nice thấp hơn → trọng số cao hơn → deadline ảo sớm hơn → được chạy **thường xuyên hơn**, nhận nhiều CPU hơn.

### Quyền hạn thay đổi nice value

| Ai | Có thể làm gì |
|---|---|
| **Root / privileged user** | Giảm nice value (tăng ưu tiên) của **bất kỳ** tiến trình nào |
| **User thường** | Chỉ được **tăng** nice value (giảm ưu tiên) của **tiến trình của chính mình** — không hạ được xuống thấp hơn, không đổi được tiến trình của user khác |

### Xem độ ưu tiên tiến trình

**`top` (real-time, động):**
```bash
top
```
- Cột `PR` (priority) và `NI` (nice).
- Với tiến trình thường: `PR = 20 + nice_value`
- Với tiến trình real-time: `PR = -1 - real_time_priority` (luôn ra số âm; có thể hiện chữ `rt` = -100, ưu tiên cao nhất)

**`ps` (snapshot tĩnh):**
```bash
ps -o pid,priority,nice,cls,pcpu,comm -C sha1sum
```
- Cột `CLS` (scheduling class): `TS` = time-sharing (SCHED_NORMAL), `FF`/`RR` = real-time.
- Real-time process không có nice value → cột `NI` hiện dấu `-`.

Sắp xếp theo nice value:
```bash
ps -eo pid,priority,nice,cls,pcpu,comm --sort=-nice | head
```

### Khởi động tiến trình với nice value cụ thể

| Lệnh | Kết quả |
|---|---|
| `command &` | Chạy nền, nice = 0 (kế thừa từ shell) |
| `nice command &` | Chạy nền, nice = **10** (mặc định của lệnh `nice`) |
| `nice -n 15 command &` | Chạy nền, nice = **15** (tự chọn) |

### Đổi nice value của tiến trình đang chạy: `renice`
```bash
renice -n 19 3033        # đổi nice của PID 3033 thành 19
```
> ⚠️ User thường **không hạ được** nice value xuống thấp hơn (báo lỗi `Permission denied`) — chỉ root mới làm được. User thường muốn "phục hồi" ưu tiên gốc phải **kill và chạy lại** tiến trình.

Cũng có thể renice ngay trong `top`: nhấn phím **`R`** → nhập PID → nhập nice value mới.

### 📝 Bài thực hành: Ảnh hưởng đến Lập lịch Tiến trình
**Mục tiêu:** Quan sát cách nice value ảnh hưởng % CPU thực tế của tiến trình.

1. Xem số CPU: `nproc` → ví dụ `2`
2. Tạo tải CPU: `for i in {1..4}; do sha1sum /dev/zero & done` (4 tiến trình chiếm CPU)
3. Xem tiến trình đang chạy: `jobs`, và `ps u -C sha1sum` (xem %CPU mỗi tiến trình ~49% mỗi cái)
4. Dừng hết: `pkill sha1sum`
5. Tạo lại: 3 tiến trình bình thường + 1 tiến trình nice=12:
   ```bash
   for i in {1..3}; do sha1sum /dev/zero & done
   nice -n 12 sha1sum /dev/zero &
   ```
6. Kiểm tra: `ps -o pid,pcpu,nice,comm -C sha1sum` → tiến trình nice=12 nhận %CPU **thấp hơn hẳn** (~5.4% so với ~65% của các tiến trình khác)
7. Tăng ưu tiên tiến trình đó: `sudo renice -n 5 9498` (giảm nice từ 12 → 5)
8. Kiểm tra lại: %CPU của tiến trình đó **tăng lên** (~9.7%) — ưu tiên cao hơn dẫn tới CPU share nhiều hơn
9. Dọn dẹp: `pkill sha1sum`

---

## 3. Bảng so sánh nhanh: Khi nào dùng gì?

| Tình huống | Công cụ nên dùng |
|---|---|
| Cần tối ưu tổng thể hệ thống theo loại workload (server, VM, laptop...) | **TuneD profile** |
| Cần tinh chỉnh **1 tiến trình cụ thể** để ưu tiên/hạ ưu tiên CPU | **nice / renice** |
| Muốn hệ thống tự thích nghi theo tải thực tế theo thời gian | **Dynamic tuning** (TuneD) |
| Cần ổn định, dễ đoán, không đổi theo thời gian | **Static tuning** (mặc định) |
| Muốn quản lý profile qua giao diện web | **Cockpit (Web Console)** |

## 4. Các quy tắc "vàng" cần nhớ

1. ❌ Không sửa file trong `/usr/lib/tuned/profiles/` → copy sang `/etc/tuned/profiles/` rồi sửa.
2. ✅ `tuned-adm profile TÊN` thay đổi **bền vững** (giữ qua reboot).
3. ✅ Dynamic tuning **tắt mặc định** để đảm bảo hiệu năng dễ đoán — chỉ bật khi thực sự cần thích nghi theo tải.
4. ✅ Nice value chỉ ảnh hưởng tiến trình `SCHED_NORMAL`, không vượt qua được tiến trình real-time.
5. ⚠️ User thường chỉ tăng được nice (giảm ưu tiên) của tiến trình **chính mình**, không hạ xuống được.
6. ✅ Công thức nhanh: `PR (top) = 20 + nice` cho tiến trình thường.
7. ✅ Dùng `tuned-adm verify` để xác nhận cấu hình hệ thống thực sự khớp với profile đang khai báo.