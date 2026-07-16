
# Tổng Hợp: Regular Expressions (Regex) trong Quản Trị Hệ Thống RHEL 10

> Tài liệu tổng hợp Chương 2 – Using Regular Expressions for Practical Applications. Trọng tâm: cú pháp regex và cách dùng với `grep`.

---

## 1. Regex là gì?

- **Regular Expression** = cơ chế **tìm kiếm theo mẫu (pattern matching)** để lọc nội dung mong muốn.
- Dùng được với: `grep`, `less`, `vim`, và nhiều ngôn ngữ lập trình (Python, C, Rust...) — cú pháp có thể khác nhau đôi chút.

**Use case:** Lọc log file, tìm dòng cấu hình cụ thể, kiểm tra dữ liệu theo định dạng — công cụ nền tảng cho hầu hết công việc admin hệ thống hằng ngày.

---

## 2. Regex đơn giản nhất: Khớp chính xác chuỗi

Tìm `cat` trong file:
```
cat
dog
concatenate
dogma
category
educated
boondoggle
vindication
chilidog
```
→ Khớp: `cat`, `con[cat]enate`, `[cat]egory`, edu[cat]ed, vindi[cat]ion — vì regex khớp **bất kỳ đâu** trong dòng (đầu/giữa/cuối), không cần cả từ.

---

## 3. Line Anchor — Ký hiệu neo vị trí dòng

| Ký hiệu | Ý nghĩa |
|---|---|
| `^` | Khớp **đầu dòng** |
| `$` | Khớp **cuối dòng** |
| `^text$` | Khớp dòng **chỉ chứa đúng** `text`, không có gì khác |

Ví dụ với file trên:
- `^cat` → khớp `cat`, `category` (bắt đầu bằng cat)
- `cat$` → chỉ khớp dòng `cat` (kết thúc bằng cat)
- `dog$` → khớp `dog`, `chilidog`
- `^cat$` → chỉ khớp dòng **chính xác** là `cat`

---

## 4. Basic vs Extended Regular Expression (BRE vs ERE)

### Khác biệt cốt lõi
Với các ký tự đặc biệt `| + ? ( ) { }`:

| | Basic (BRE) | Extended (ERE) |
|---|---|---|
| Ký tự đặc biệt cần | `\` phía trước mới có nghĩa đặc biệt | **Mặc định** có nghĩa đặc biệt (không cần `\`) |
| Ví dụ: dấu `+` | `\+` = 1 hoặc nhiều lần | `+` = 1 hoặc nhiều lần |

### Lệnh nào dùng loại nào?

| Lệnh | Loại mặc định |
|---|---|
| `grep`, `sed`, `vim` | **Basic** (BRE) |
| `grep -E`, `sed -E` | **Extended** (ERE) |
| `less` | Extended (ERE) |

> 💡 Cách nhớ: muốn dùng cú pháp ERE hiện đại, gọn hơn (không cần backslash) → thêm `-E`.

---

## 5. Bảng Cú pháp Regex Đầy đủ

### Wildcard & Multiplier

| Basic | Extended | Ý nghĩa |
|---|---|---|
| `.` | `.` | Khớp **1 ký tự bất kỳ** |
| — | `?` | Ký tự/nhóm trước đó **tùy chọn** (0 hoặc 1 lần) |
| `*` | `*` | Ký tự/nhóm trước đó lặp **0 lần trở lên** |
| — | `+` | Ký tự/nhóm trước đó lặp **1 lần trở lên** |
| `\{n\}` | `{n}` | Lặp đúng **n lần** |
| `\{n,\}` | `{n,}` | Lặp **≥ n lần** |
| `\{,m\}` | `{,m}` | Lặp **≤ m lần** |
| `\{n,m\}` | `{n,m}` | Lặp **từ n đến m lần** |

### Ví dụ minh họa
```
c.t         → khớp cat, cot, cut, c$t... (1 ký tự bất kỳ giữa c và t)
c[aou]t     → chỉ khớp cat, cot, cut (giới hạn trong tập a/o/u)
c[aou]*t    → khớp coat, coot (0 hoặc nhiều ký tự trong tập)
c.*t        → khớp cat, coat, culvert, ct (0+ ký tự bất kỳ)
c.{2}t      → khớp chính xác 2 ký tự bất kỳ giữa c và t (vd covert, convert... nếu đủ 2 ký tự)
```

> 💡 **Phân biệt với shell globbing:** Regex và file globbing (dùng khi gõ `*` để chọn nhiều file trên dòng lệnh) **trông giống** nhưng **quy tắc khác nhau** — đừng nhầm lẫn 2 khái niệm.

### Character Class (lớp ký tự) — POSIX

| Ký hiệu | Ý nghĩa |
|---|---|
| `[:alnum:]` | Chữ + số (`[0-9A-Za-z]`) |
| `[:alpha:]` | Chữ cái (`[A-Za-z]`) |
| `[:digit:]` | Số (`0-9`) |
| `[:lower:]` | Chữ thường |
| `[:upper:]` | Chữ hoa |
| `[:space:]` | Khoảng trắng (space, tab, newline...) |
| `[:blank:]` | Chỉ space và tab |
| `[:punct:]` | Ký tự dấu câu |
| `[:cntrl:]` | Ký tự điều khiển |
| `[:print:]` | Ký tự in được |
| `[:graph:]` | Ký tự hiển thị được (không gồm space) |
| `[:xdigit:]` | Ký tự hex (0-9, A-F, a-f) |

### Word Boundary & Shortcut (GNU extension)

| Ký hiệu | Ý nghĩa |
|---|---|
| `\b` | Ranh giới từ (đầu/cuối từ) |
| `\B` | KHÔNG phải ranh giới từ |
| `\<` | Đầu từ |
| `\>` | Cuối từ |
| `\w` | Ký tự thuộc từ — tương đương `[_[:alnum:]]` |
| `\W` | Ký tự KHÔNG thuộc từ |
| `\s` | Khoảng trắng — tương đương `[[:space:]]` |
| `\S` | KHÔNG phải khoảng trắng |

---

## 6. Dùng Regex với `grep`

### Cú pháp cơ bản
```bash
grep 'REGEX' filename
```
> 💡 **Luôn dùng dấu nháy đơn** `'...'` bọc regex để tránh shell tự diễn giải các ký tự đặc biệt (`$`, `*`, `{}`...) trước khi `grep` nhận được.

### Dùng với pipe
```bash
ps aux | grep chrony
```

### Các option quan trọng của `grep`

| Option | Ý nghĩa |
|---|---|
| `-i` | Không phân biệt hoa/thường |
| `-v` | **Đảo ngược** — chỉ hiện dòng **KHÔNG** khớp |
| `-r` | Tìm **đệ quy** trong thư mục |
| `-A NUMBER` | Hiện thêm N dòng **sau** dòng khớp |
| `-B NUMBER` | Hiện thêm N dòng **trước** dòng khớp |
| `-e` | Cho phép chỉ định **nhiều regex** cùng lúc (nối bằng OR logic) |
| `-E` | Dùng cú pháp **Extended** regex |

### Ví dụ thực tế

**Tìm không phân biệt hoa thường:**
```bash
grep -i serverroot /etc/httpd/conf/httpd.conf
```

**Loại trừ dòng khớp (`-v`) — xem file bỏ qua comment:**
```bash
grep -v -i server /etc/hosts
grep -v '^[#;]' /etc/systemd/system/.../rsyslog.service
```
→ Loại bỏ dòng bắt đầu bằng `#` hoặc `;` (comment).

**Tìm nhiều pattern cùng lúc (`-e` nhiều lần = OR):**
```bash
grep -e 'pam_unix' -e 'user root' -e 'Accepted publickey' /var/log/secure | less
```

**Xem context trước/sau dòng khớp:**
```bash
systemctl status httpd | grep Active -B 2 -A 3
```

### Tìm kiếm trong `less`/`vim`
- Nhấn `/` rồi gõ pattern, `Enter` để tìm.
- `n` để tìm tiếp lần khớp kế tiếp.
```
/usb$              # tìm dòng kết thúc bằng "usb"
/[Nn][Tt][Pp]      # tìm NTP không phân biệt hoa thường (kiểu thủ công)
/^ServerAdmin       # tìm dòng bắt đầu bằng ServerAdmin
```

---

## 7. 📝 Bài thực hành: Khớp Text bằng Regex

**Mục tiêu:** Dùng `grep` để tra cứu thông tin hệ thống/cấu hình Apache qua nhiều tình huống thực tế.

1. **Tìm UID/GID của user/group `apache`** trong nhiều file cùng lúc:
   ```bash
   grep apache /etc/passwd /etc/group
   ```
   → Kết quả có tiền tố tên file (vì tìm trong ≥2 file).

2. **Xem trạng thái service kèm context xung quanh:**
   ```bash
   systemctl status httpd | grep Active -B 2 -A 3
   ```

3. **Tìm ServerRoot/DocumentRoot, không phân biệt hoa thường, chỉ ở đầu dòng:**
   ```bash
   grep -i -e ^serverroot -e ^documentroot /etc/httpd/conf/httpd.conf
   ```

4. **Lọc log liên quan httpd, lưu ra file riêng:**
   ```bash
   grep httpd /var/log/messages > /tmp/httpd.log
   cat /tmp/httpd.log
   ```

5. **Xác nhận process đang chạy:**
   ```bash
   ps ax | grep httpd
   ```

6. **Tìm `ServerAdmin` trong file cấu hình bằng `vim`:**
   ```
   vim /etc/httpd/conf/httpd.conf
   /^ServerAdmin
   ```

---

## 8. Bảng so sánh nhanh: Khi nào dùng gì?

| Tình huống | Cách dùng |
|---|---|
| Tìm dòng chứa từ khóa, không quan tâm hoa/thường | `grep -i 'pattern'` |
| Xem file bỏ comment | `grep -v '^[#;]'` |
| Tìm nhiều điều kiện OR | `grep -e 'A' -e 'B' -e 'C'` |
| Xem thêm ngữ cảnh quanh kết quả | `grep 'pattern' -A N -B N` |
| Regex phức tạp (dùng `+`, `?`, `{}` không cần escape) | `grep -E 'pattern'` |
| Tìm chính xác 1 dòng (không có gì khác) | `^pattern$` |
| Tìm trong nhiều file/thư mục cùng lúc | `grep -r 'pattern' /path` |
| Tìm trong `less`/`vim` khi đang xem file | `/pattern` rồi `n` để lặp lại |

## 9. Các quy tắc "vàng" cần nhớ

1. ✅ Luôn bọc regex trong **dấu nháy đơn** `'...'` để tránh shell can thiệp vào ký tự đặc biệt.
2. ✅ `grep`, `sed`, `vim` mặc định dùng **Basic** regex — muốn cú pháp gọn hơn (`+`, `?`, `{}` không cần `\`) → thêm `-E`.
3. ✅ `^` = đầu dòng, `$` = cuối dòng, `^...$` = khớp chính xác toàn bộ dòng.
4. ✅ `.` khớp 1 ký tự bất kỳ; `*` = lặp 0+ lần của ký tự/nhóm **ngay trước nó** — không phải wildcard độc lập.
5. ⚠️ Regex ≠ Shell globbing (file name expansion) — cùng dùng `*` nhưng ý nghĩa và ngữ cảnh khác nhau.
6. ✅ `grep -v` hữu ích để **loại bỏ comment**/nhiễu khi đọc file cấu hình.
7. ✅ Nhiều `-e` trong 1 lệnh `grep` = tìm theo logic **OR** giữa các pattern.
8. ✅ Trong `less`/`vim`, `/` để tìm, `n` để nhảy tới kết quả tiếp theo.