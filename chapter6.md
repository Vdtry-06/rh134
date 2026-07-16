# Tổng Hợp: Quản Lý Bảo Mật với SELinux trên RHEL 10

> Tài liệu tổng hợp Chương 6 – Managing Security with SELinux. Gồm 4 phần: **Vận hành SELinux (mode)**, **File Context**, **Booleans**, và **Điều tra & Xử lý sự cố SELinux**.

---

## 1. Kiến trúc & Khái niệm SELinux

### Vấn đề mà SELinux giải quyết
- **File permission (DAC)** chỉ kiểm soát **ai** được đọc/ghi/chạy file — không kiểm soát file được **dùng vào việc gì**.
- Ví dụ: user có quyền ghi vào 1 file dữ liệu có cấu trúc, nhưng chương trình khác (không phải chương trình dự kiến) vẫn có thể mở/sửa file đó → rủi ro hỏng dữ liệu/bảo mật.

### DAC vs MAC

| | DAC (Discretionary Access Control) | MAC (Mandatory Access Control) — SELinux |
|---|---|---|
| Ai kiểm soát | Admin/owner tùy ý set permission | **Policy cố định**, áp dụng cho **mọi user**, không thể bypass tùy tiện |
| Ví dụ | `chmod`, `chown` | SELinux targeted policy |

**Use case kinh điển:** Web server bị khai thác lỗ hổng, hacker chiếm quyền user `apache` → dù có quyền file hệ thống, SELinux **vẫn chặn** truy cập vào `/tmp`, `/var/tmp` hay các thư mục khác nếu policy không cho phép.

### Nguyên tắc cốt lõi: **"Mặc định từ chối"**
- SELinux label mọi resource (process, file, port...) bằng **context**.
- **Không có rule cho phép rõ ràng → hành động bị TỪ CHỐI**.
- Targeted policy dùng **type context** (tên thường kết thúc bằng `_t`).

### Ví dụ policy thực tế

| Resource | Type Context |
|---|---|
| Process Apache | `httpd_t` |
| File/thư mục web (`/var/www/html/`) | `httpd_sys_content_t` |
| File tạm (`/tmp`, `/var/tmp`) | `tmp_t` |
| Port web (80/443) | `http_port_t` |
| Process MariaDB | `mysqld_t` |
| File DB MariaDB | `mysqld_db_t` |

→ Apache (`httpd_t`) được phép truy cập file `httpd_sys_content_t`, nhưng **không** có rule cho phép truy cập `tmp_t` → dù bị compromise, hacker vẫn không đụng được vào `/tmp`.

### Xem context bằng option `-Z`
```bash
ps axZ                  # xem context của mọi process
ps -ZC httpd            # lọc theo tên process
ls -Z /var/www          # xem context của file/thư mục
```

---

## 2. Chế độ hoạt động (Mode) của SELinux

| Mode | Ý nghĩa |
|---|---|
| **Enforcing** | Áp dụng đầy đủ rule — **mặc định và khuyến nghị** |
| **Permissive** | Vẫn load policy, nhưng chỉ **log** vi phạm, không chặn — dùng để test/debug |
| **Disabled** | Tắt hẳn — **không khuyến khích**, và **RHEL không còn hỗ trợ** đặt `SELINUX=disabled` trong config nữa |

> ⚠️ Không dùng chế độ Disabled: file sẽ không được label → sau này muốn bật lại SELinux sẽ rất khó khăn.

### Xem/đổi mode hiện tại (runtime)
```bash
getenforce                          # xem mode hiện tại
setenforce 0                        # chuyển sang Permissive
setenforce 1                        # chuyển sang Enforcing
setenforce Enforcing|Permissive     # cũng chấp nhận tên đầy đủ
```

### Đổi mode qua kernel parameter lúc boot (tạm thời)
```
enforcing=0      # boot vào Permissive
enforcing=1      # boot vào Enforcing
selinux=0        # tắt hẳn SELinux
selinux=1        # bật SELinux
```

### Đặt mode mặc định bền vững: `/etc/selinux/config`
```ini
SELINUX=enforcing      # enforcing | permissive | disabled
SELINUXTYPE=targeted   # targeted | minimum | mls
```
> 💡 File này được đọc lúc **boot**; kernel argument (`selinux=`, `enforcing=`) sẽ **override** giá trị trong file này.

> ⚠️ **Khuyến nghị Red Hat:** Khi chuyển từ Permissive → Enforcing, nên **reboot** để đảm bảo các service khởi động lại đúng trong chế độ bị ràng buộc (confined) ngay từ đầu.

### 📝 Bài thực hành: Vận hành SELinux
1. `getenforce` → `Enforcing`
2. Sửa `/etc/selinux/config`: `SELINUX=permissive`
3. `setenforce 0` (áp dụng ngay, không cần reboot)
4. `getenforce` → `Permissive`
5. Sửa lại config: `SELINUX=enforcing`
6. `setenforce 1`
7. `getenforce` → `Enforcing`
8. `systemctl reboot` → login lại → `getenforce` → xác nhận vẫn `Enforcing` (bền vững qua reboot)

---

## 3. SELinux File Context

### Cơ chế gán label khi tạo file mới
```
1. Tên file khớp rule trong policy? → dùng label đó
2. Không khớp? → kế thừa label của THƯ MỤC CHA
```
→ Mọi file **luôn có label**, dù không có rule cụ thể (nhờ kế thừa).

### Quan trọng: `cp` vs `mv` xử lý label khác nhau!

| Thao tác | Hành vi với SELinux context |
|---|---|
| **`cp`** (copy) | Tạo **inode mới** → context được **gán lại theo vị trí đích** (hoặc kế thừa thư mục cha nếu không có rule) |
| **`mv`** (move, cùng file system) | **Giữ nguyên inode** → context **giữ nguyên như file gốc**, KHÔNG đổi theo đích |

> ⚠️ Đây là nguồn gốc phổ biến nhất của lỗi SELinux: **di chuyển file** vào thư mục web server nhưng label vẫn là label cũ (vd `tmp_t`) → service không truy cập được dù đường dẫn đúng.

Ví dụ minh họa:
```bash
touch /tmp/file1 /tmp/file2     # cả 2 đều có label user_tmp_t (kế thừa từ /tmp)
mv /tmp/file1 /var/www/html/    # GIỮ label cũ user_tmp_t (dù giờ nằm trong /var/www/html)
cp /tmp/file2 /var/www/html/    # NHẬN label mới httpd_sys_content_t (theo đích)
```

> 💡 Muốn giữ nguyên context khi copy: `cp -p` (giữ mọi thuộc tính) hoặc `cp --preserve=context` (chỉ giữ SELinux context).

### 3 công cụ quản lý Context

| Lệnh | Vai trò |
|---|---|
| `chcon` | Đổi context **trực tiếp, tạm thời** trên 1 file — không dùng policy hệ thống |
| `semanage fcontext` | Định nghĩa **policy** cho 1 đường dẫn (persistent) |
| `restorecon` | **Áp dụng** context theo policy đã định nghĩa (relabel) |

> ⚠️ **`chcon` không bền vững thực sự** — nếu sau này ai đó chạy `restorecon`, context bị đặt bằng `chcon` sẽ bị **ghi đè về giá trị mặc định theo policy** (vì không khớp rule).

### Cách làm ĐÚNG (khuyến nghị): `semanage fcontext` + `restorecon`
```bash
# Bước 1: định nghĩa rule cho đường dẫn
semanage fcontext -a -t httpd_sys_content_t '/custom(/.*)?'

# Bước 2: áp dụng rule đó lên file/thư mục thực tế
restorecon -Rv /custom
```
> 💡 Cú pháp `(/.*)?` (gọi vui là "pirate" — trông giống mặt cướp biển có mắt kính che 1 bên + móc câu) nghĩa là: khớp cả thư mục gốc lẫn **mọi file/thư mục con** bên trong, đệ quy.

### Các lệnh `semanage fcontext` quan trọng
```bash
semanage fcontext -l          # liệt kê TẤT CẢ rule (bao gồm mặc định hệ thống)
semanage fcontext -l -C       # chỉ liệt kê rule TÙY CHỈNH (Customized) do người dùng thêm
semanage fcontext -a -t TYPE 'PATH(/.*)?'    # thêm rule mới
semanage fcontext -d -t TYPE 'PATH'          # xóa rule
```

> 💡 Cần cài `policycoreutils` và `policycoreutils-python-utils` để có `restorecon`/`semanage`.

### 📝 Bài thực hành: Kiểm soát SELinux File Context

**Mục tiêu:** Apache dùng document root **không chuẩn** (`/custom` thay vì `/var/www/html`) → bị chặn → sửa bằng `semanage fcontext` + `restorecon`.

1. Tạo `/custom/index.html` với nội dung test
2. Sửa `httpd.conf`: đổi `DocumentRoot` sang `/custom`
3. `systemctl enable --now httpd`
4. Từ máy khác: `curl http://servera/index.html` → **403 Forbidden**
5. So sánh context: `ls -ldZ /custom /var/www/html` → `/custom` có `default_t`, còn `/var/www/html` có `httpd_sys_content_t`
6. Định nghĩa rule: `semanage fcontext -a -t httpd_sys_content_t '/custom(/.*)?'`
7. Áp dụng: `restorecon -Rv /custom`
8. Test lại: `curl http://servera/index.html` → **thành công**

---

## 4. SELinux Booleans — Bật/Tắt Hành vi Tùy chọn

### Khái niệm
- Developer định nghĩa sẵn các **hành vi tùy chọn** (optional behavior) trong policy → **Boolean** cho phép bật/tắt các hành vi đó mà **không cần viết policy mới**.
- Tài liệu: man page `SERVICENAME_selinux` (package `selinux-policy-doc`).

### Xem danh sách Boolean
```bash
getsebool -a                       # tất cả Boolean + trạng thái hiện tại
getsebool httpd_enable_homedirs    # 1 Boolean cụ thể
```

### Ví dụ điển hình: `httpd_enable_homedirs`
- Cho phép Apache chia sẻ nội dung từ **home directory** của user (vd `~/public_html`) qua web — mặc định **tắt**.

### Bật/Tắt Boolean

| Lệnh | Hiệu lực |
|---|---|
| `setsebool NAME on\|off` | **Tạm thời** — mất khi reboot |
| `setsebool -P NAME on\|off` | **Bền vững** — ghi vào policy file |

> ⚠️ Chỉ **user có quyền** (root) mới set được Boolean.

### Xem chi tiết + so sánh default vs current
```bash
semanage boolean -l | grep httpd_enable_homedirs
# httpd_enable_homedirs   (off, off)   Allow httpd to enable homedirs
#                          ^current ^default

semanage boolean -l -C    # chỉ xem Boolean đã bị TÙY CHỈNH khác default
```

### 📝 Bài thực hành: Tune SELinux Policy bằng Boolean

**Mục tiêu:** Cho phép user `student` publish web content từ `~/public_html`.

1. Sửa `/etc/httpd/conf.d/userdir.conf`: comment `UserDir disabled`, bỏ comment `UserDir public_html`
2. `systemctl enable --now httpd`
3. Tạo `~/public_html/index.html` với nội dung test
4. `chmod 711 /home/student` (cho phép Apache "đi qua" thư mục home để vào `public_html`)
5. Test: `http://servera/~student/index.html` → **lỗi permission** (dù đã sửa Apache config + file permission đúng)
6. Kiểm tra Boolean: `getsebool -a | grep home` → `httpd_enable_homedirs --> off`
7. Bật bền vững: `setsebool -P httpd_enable_homedirs on`
8. Test lại → **thành công**

> 💡 Bài học: Vấn đề này **không phải** do file context (khác với bài trước) — mà là do 1 **hành vi bị tắt mặc định** ở cấp policy (Boolean), cần bật riêng.

---

## 5. Điều tra & Xử lý Sự cố SELinux

### Nguyên tắc tư duy khi troubleshoot
- Đa số trường hợp bị chặn = SELinux đang **làm đúng việc của nó**.
- Lỗi phổ biến nhất: **context sai** trên file mới tạo/copy/move.
- SELinux **không thay thế** file permission/ACL — cả 2 lớp đều phải đúng.
- Ưu tiên kiểm tra man page `_selinux` của service trước khi đổi cấu hình rộng.

### Công cụ giám sát: `setroubleshoot`
- Package: `setroubleshoot-server`
- Khi SELinux từ chối hành động → ghi **AVC (Access Vector Cache) message** vào `/var/log/audit/audit.log`.
- Service `setroubleshoot` theo dõi AVC, gửi tóm tắt kèm **UUID** vào `/var/log/messages`.

### Quy trình chẩn đoán

```bash
# Bước 1: tìm UUID gợi ý trong log
tail /var/log/messages
# → "SELinux is preventing ... run: sealert -l <UUID>"

# Bước 2: xem báo cáo chi tiết
sealert -l <UUID>

# Hoặc xem TẤT CẢ sự kiện cùng lúc
sealert -a /var/log/audit/audit.log
```

### Đọc kết quả `sealert`
- Gợi ý kèm **confidence rating** (độ tin cậy của gợi ý).
- ⚠️ **Gợi ý không phải lúc nào cũng đúng giải pháp cho tình huống của bạn** — vd nếu file đặt **sai vị trí** hoàn toàn, giải pháp đúng là **di chuyển file** rồi restorecon, chứ không phải tạo policy mới cho vị trí sai đó.

Ví dụ output điển hình:
```
SELinux is preventing /usr/sbin/httpd from getattr access on the file /var/www/html/mypage.

*****  Plugin restorecon (99.5 confidence) suggests  *****
# /sbin/restorecon -v /var/www/html/mypage

*****  Plugin catchall (1.49 confidence) suggests  *****
# ausearch -c 'httpd' --raw | audit2allow -M my-httpd
# semodule -X 300 -i my-httpd.pp
```
→ Chú ý **confidence cao hơn** (99.5% vs 1.49%) — nên ưu tiên giải pháp `restorecon` hơn là tạo policy module riêng.

### Tìm AVC event bằng `ausearch`
```bash
ausearch -m AVC -ts recent     # tìm event AVC gần đây
ausearch -m AVC -ts today      # tìm event AVC hôm nay
```
- `-m`: loại message (AVC)
- `-ts`: mốc thời gian bắt đầu tìm

### Web Console
Menu **SELinux** → xem trạng thái enforcing + danh sách lỗi → click chi tiết → **Apply this solution** để áp dụng gợi ý trực tiếp.

### 📝 Bài thực hành: Điều tra & Xử lý Sự cố SELinux

**Mục tiêu:** `/custom/index.html` (từ bài file context) lại gặp lỗi tương tự khi Apache không truy cập được — luyện quy trình chẩn đoán đầy đủ bằng log.

1. Truy cập `http://servera/index.html` → lỗi permission
2. `less /var/log/messages`, tìm `sealert`, lấy UUID gợi ý
3. Chạy `sealert -l UUID>` → thấy:
   - Source Context: `httpd_t`
   - Target Context: `default_t` (SAI — cần `httpd_sys_content_t`)
   - Gợi ý: `semanage fcontext -a -t FILE_TYPE ...` rồi `restorecon`
4. So sánh với context đúng của `/var/www/html`: `ls -ldZ /var/www/html` → `httpd_sys_content_t`
5. Tra cứu thêm bằng `ausearch -m AVC -ts today` → xác nhận chi tiết event
6. Áp dụng fix:
   ```bash
   semanage fcontext -a -t httpd_sys_content_t '/custom(/.*)?'
   restorecon -Rv /custom
   ```
7. Test lại `http://servera/index.html` → thành công

---

## 6. Bảng so sánh nhanh: Khi nào dùng công cụ nào?

| Tình huống | Công cụ |
|---|---|
| Cần test nhanh, tạm tắt enforcement để debug | `setenforce 0` |
| Cần đổi mode bền vững | Sửa `/etc/selinux/config` |
| File bị sai label do `mv`/tạo ở vị trí lạ | `semanage fcontext -a` + `restorecon` |
| Cần đổi label tạm thời để test nhanh (không nên dùng lâu dài) | `chcon` |
| Service cần bật 1 tính năng tùy chọn (vd chia sẻ home dir) | `getsebool` + `setsebool -P` |
| Không rõ tại sao bị chặn, cần chẩn đoán | `sealert -a /var/log/audit/audit.log` hoặc theo UUID trong `/var/log/messages` |
| Cần tìm chi tiết raw AVC event | `ausearch -m AVC -ts ...` |

## 7. Các quy tắc "vàng" cần nhớ

1. ✅ SELinux mặc định **từ chối mọi thứ** trừ khi có rule cho phép rõ ràng.
2. ⚠️ RHEL 10 **không còn hỗ trợ** `SELINUX=disabled` trong config — Enforcing là chế độ khuyến nghị duy nhất cho production.
3. ⚠️ `mv` **giữ nguyên** SELinux context gốc; `cp` **gán lại** theo context đích — đây là lỗi phổ biến nhất cần nhớ.
4. ✅ Cách đúng để đổi context bền vững: `semanage fcontext -a` (định nghĩa rule) → `restorecon` (áp dụng) — không nên chỉ dùng `chcon` một mình.
5. ✅ Cú pháp `(/.*)?` nghĩa là khớp cả thư mục gốc và mọi nội dung bên trong (đệ quy).
6. ✅ Booleans dùng cho hành vi **tùy chọn đã được developer định nghĩa sẵn** — không phải để tự tạo rule mới.
7. ✅ `setsebool` không có `-P` chỉ tạm thời; luôn thêm `-P` để bền vững qua reboot.
8. ⚠️ Gợi ý của `sealert` có **confidence rating** khác nhau — cần đánh giá xem có phù hợp với tình huống thật (đôi khi giải pháp đúng là sửa vị trí file, không phải đổi policy).
9. ✅ SELinux **không thay thế** file permission/ACL — cả 2 lớp bảo mật đều cần đúng đồng thời.