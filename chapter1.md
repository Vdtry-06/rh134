# Tổng Hợp: Shell Scripting & Command Line trên RHEL 10

> Tài liệu tổng hợp Chương 1 – Shell Scripting and the Command Line. Gồm 3 phần: **Biến & Môi trường Shell**, **Viết Bash Script cơ bản**, và **Vòng lặp & Điều kiện**.

---

## 1. Shell Variables — Biến Shell

### Khái niệm
- Biến shell **chỉ tồn tại trong 1 phiên shell** — mở 2 terminal = 2 bộ biến độc lập.
- Cú pháp gán: `VARIABLENAME=value` — **KHÔNG có khoảng trắng** quanh dấu `=`.
- Tên biến: chữ hoa/thường, số, dấu `_`.

```bash
COUNT=40
first_name=John
full_name='John Smith'      # cần quote vì có khoảng trắng
_ID=Training
```
> 💡 Quy ước: biến do shell tự set thường **VIẾT HOA**; biến tự đặt nên dùng **chữ thường** để tránh trùng tên.

### Xem tất cả biến hiện tại
```bash
set | less
```

### 3 kiểu dữ liệu: String, Number, Array

**String:**
```bash
bare=This\ string\ escapes\ all\ spaces.      # escape từng space bằng \
double_quote="Expand ${variable} here"         # nháy kép → CÓ expand biến/lệnh
single_quote='Literal $ char, no expand'       # nháy đơn → KHÔNG expand gì cả
```

**Number (tính toán số học):**
```bash
five=5
ten=10
fifteen=$((five + ten))    # Bash arithmetic expansion
```

**Array (mảng — index bắt đầu từ 0):**
```bash
colors=('red' 'green' 'blue')
echo "${colors[2]}"        # blue
echo "${colors[@]}"        # red green blue (tất cả phần tử)
```

---

## 2. Variable Expansion — Lấy giá trị biến

```bash
COUNT=40
echo COUNT      # → in chữ "COUNT" (không có $ = không expand)
echo $COUNT     # → 40
echo "Count: $COUNT"     # nháy kép → expand được
echo 'Count: $COUNT'     # nháy đơn → KHÔNG expand, in nguyên văn
```

### ⚠️ Khi nào BẮT BUỘC dùng dấu ngoặc `{}`
Khi có ký tự bám ngay sau tên biến (dễ gây nhầm với tên biến khác):
```bash
echo "Repeat $COUNTx"       # SAI: Bash tìm biến "COUNTx" (không tồn tại) → in rỗng
echo "Repeat ${COUNT}x"     # ĐÚNG: rõ ràng biến là COUNT, "x" chỉ là ký tự thường → "Repeat 40x"
```
> 💡 `{}` còn dùng để truy cập phần tử mảng: `${colors[2]}`.

---

## 3. Cấu hình Shell bằng Shell Variables

| Biến | Tác dụng |
|---|---|
| `HISTFILE` | File lưu lịch sử lệnh (mặc định `~/.bash_history`) |
| `HISTFILESIZE` | Số lệnh tối đa lưu trong file lịch sử |
| `HISTTIMEFORMAT` | Định dạng timestamp cho mỗi lệnh trong `history` (không có sẵn mặc định) |
| `PS1` | Định dạng **command prompt** |

```bash
HISTTIMEFORMAT="%F %T "     # bật timestamp cho history
```

### Tùy biến prompt (`PS1`)
```bash
PS1="[\u@\h \W]\$ "
```
| Ký hiệu | Ý nghĩa |
|---|---|
| `\u` | Username hiện tại |
| `\h` | Hostname (tới dấu `.` đầu tiên) |
| `\W` | Tên thư mục hiện tại (basename), `$HOME` viết tắt `~` |
| `\$` | `#` nếu là root (UID 0), ngược lại `$` |

> 💡 Red Hat khuyến nghị prompt kết thúc bằng **1 khoảng trắng** để dễ phân biệt với lệnh gõ vào.

---

## 4. Environment Variables (Biến Môi trường)

### Khác biệt với Shell Variable
| | Shell variable thường | Environment variable |
|---|---|---|
| Phạm vi | Chỉ shell hiện tại dùng được | Shell **VÀ** mọi chương trình chạy từ shell đó đều dùng được |
| Cách tạo | `VAR=value` | `export VAR=value` (hoặc `export VAR` sau khi gán) |

```bash
EDITOR=vim
export EDITOR          # export riêng
# HOẶC làm 1 bước:
export EDITOR=vim
```

### Xem toàn bộ environment variable
```bash
env
```

### Các biến môi trường quan trọng

| Biến | Ý nghĩa |
|---|---|
| `HOME` | Thư mục home, tự động set khi shell khởi động |
| `LANG` | Locale (ngôn ngữ, format ngày/số/tiền tệ) — vd `en_US.UTF-8`, `fr_FR.UTF-8` |
| `PATH` | Danh sách thư mục (cách nhau bởi `:`) để tìm executable |
| `EDITOR` | Trình soạn thảo mặc định cho các chương trình dòng lệnh |

```bash
echo $PATH
export PATH=${PATH}:/home/user/sbin    # thêm thư mục vào PATH
```
> 💡 Khi chạy 1 lệnh, shell tìm executable **theo thứ tự** trong `PATH`, dùng file khớp **đầu tiên** tìm thấy.

---

## 5. File Khởi tạo Shell (Startup Scripts)

| Loại shell | File chạy |
|---|---|
| **Interactive login** (SSH, login trực tiếp) | `/etc/profile` (source `/etc/bashrc`) → `~/.bash_profile` (source `~/.bashrc`) |
| **Interactive non-login** (mở terminal mới trong GUI) | Chỉ `/etc/bashrc` và `~/.bashrc` |
| **Non-interactive** (script chạy nền) | File mà biến `BASH_ENV` chỉ định (mặc định không set) |

> ✅ Muốn setting áp dụng **toàn hệ thống** cho mọi user: tạo file `.sh` trong `/etc/profile.d/` (cần quyền root).
> ✅ Muốn setting chỉ áp dụng **cho riêng mình** khi SSH vào: sửa `~/.bash_profile`.

---

## 6. Bash Aliases

```bash
alias hello='echo "Hello, this is a long string."'
hello    # chạy alias → in ra chuỗi
```
> 💡 Thêm alias vào `~/.bashrc` để có sẵn ở mọi shell tương tác.

### Hủy biến/alias
```bash
unset file1          # xóa biến (unset + unexport)
export -n PS1        # unexport nhưng vẫn giữ biến (không còn là env var)
unalias hello         # xóa alias
```

---

## 7. 📝 Bài thực hành: Thay đổi Shell Environment

1. Sửa `~/.bashrc`, thêm: `PS1='[\u@\h \t \w]$ '` (thêm `\t` hiển thị giờ)
2. `source ~/.bashrc` để áp dụng ngay
3. Tạo biến: `file=examplefile`
4. Dùng biến trong lệnh: `ls -l $file`, rồi `rm $file`
5. Xác nhận đã xóa: `ls -l $file` → lỗi "No such file"
6. Export biến EDITOR: `export EDITOR=vim`, kiểm tra `echo $EDITOR`

---

## 8. Viết Bash Script cơ bản

### Shebang (`#!`)
Dòng đầu tiên chỉ định **interpreter** để chạy script:
```bash
#!/usr/bin/bash
```

### Cấp quyền thực thi & chạy script
```bash
chmod +x script.sh
./script.sh              # chạy trong thư mục hiện tại (cần ./ nếu không có trong PATH)
```
> 💡 Nếu script nằm trong thư mục có trong `$PATH`, có thể gọi trực tiếp bằng tên (không cần `./`). Dùng `which scriptname` để tìm vị trí.

### Positional Parameters (tham số truyền vào script)

| Ký hiệu | Ý nghĩa |
|---|---|
| `$0` | Tên/đường dẫn của chính script |
| `$1`, `$2`... | Tham số thứ 1, 2... |
| `${10}` | Tham số thứ 10 trở lên — **BẮT BUỘC** dùng `{}` |

```bash
echo "Parameter 10 (incorrect): $10"    # SAI: Bash hiểu là "$1" + ký tự "0"
echo "Parameter 10 (correct): ${10}"    # ĐÚNG
```

**Dùng `set --` để đổi lại positional parameters:**
```bash
set -- "$1" "$2" "$4" "$3"    # hoán đổi vị trí $3 và $4, xóa $5 trở đi
```

### Quote & Escape ký tự đặc biệt

| Cách | Hiệu ứng |
|---|---|
| `\` (backslash) | Escape **1 ký tự** ngay sau nó |
| `'...'` (nháy đơn) | Literal **hoàn toàn** — không expand gì cả (biến, lệnh, glob) |
| `"..."` (nháy kép) | Cho phép **expand biến & command substitution**, nhưng chặn file globbing |

```bash
echo \# not a comment          # → # not a comment
echo '# not a comment #'        # → giữ nguyên literal
echo "hostname is ${var}"       # → expand biến
echo 'Will $var expand?'        # → KHÔNG expand, in nguyên "$var"
```

### Output & Redirect trong Script
```bash
echo "Hello, world"                              # ra STDOUT
echo "ERROR: something wrong" >&2                # ra STDERR (khuyến nghị cho message lỗi)
script.sh 2> error.log                            # redirect STDERR ra file log
```
> 💡 Dùng `echo` để debug script — chèn thêm dòng in giá trị biến giúp theo dõi luồng chạy.

---

## 9. 📝 Bài thực hành: Viết Bash Script Đơn giản

**Mục tiêu:** Tạo script ghi thông tin hệ thống ra file output.

1. Tạo `firstscript.sh`:
   ```bash
   #!/usr/bin/bash
   echo "This is my first bash script" > ~/output.txt
   echo "" >> ~/output.txt
   echo "################################################" >> ~/output.txt
   ```
2. `chmod +x firstscript.sh`
3. `./firstscript.sh` → `cat output.txt` kiểm tra
4. Mở rộng script, thêm phần liệt kê block device (`lsblk`) và dung lượng đĩa (`df -h`) vào cùng file output
5. Chạy lại, xác nhận nội dung đầy đủ trong `output.txt`

---

## 10. Vòng lặp `for`

### Cú pháp
```bash
for VARIABLE in LIST; do
    COMMAND $VARIABLE
done
```

### Các cách tạo LIST

```bash
for HOST in host1 host2 host3; do echo $HOST; done       # liệt kê tay
for HOST in host{1,2,3}; do echo $HOST; done              # brace expansion
for HOST in host{1..3}; do echo $HOST; done                # range expansion
for FILE in file{a..c}; do ls $FILE; done                   # range chữ cái
for PACKAGE in $(rpm -qa | grep kernel); do ... done        # command substitution
for EVEN in $(seq 2 2 10); do echo $EVEN; done              # dùng seq: 2 4 6 8 10
```

**Use case:** Chạy 1 lệnh trên **nhiều host**, backup **nhiều database**, kiểm tra định kỳ trạng thái process.

---

## 11. Exit Code — Mã thoát

- Giá trị **0-255**, biểu thị quá trình kết thúc **thành công (0)** hay **lỗi (khác 0)**.
- Biến đặc biệt `$?` = exit code của **lệnh vừa chạy xong**.

```bash
/bin/true; echo $?     # → 0 (luôn thành công)
/bin/false; echo $?    # → 1 (luôn thất bại)
```

Trong script:
```bash
exit 0     # kết thúc script sớm, trả exit code 0 (thành công)
exit       # không có số → trả exit code của LỆNH CUỐI CÙNG đã chạy trong script
```
> 💡 Dùng `errno -l` để tra danh sách mã lỗi chuẩn.

---

## 12. Kiểm tra Điều kiện (`test` / `[[ ]]`)

### So sánh số

| Toán tử | Ý nghĩa |
|---|---|
| `-eq` | Bằng |
| `-ne` | Khác |
| `-gt` | Lớn hơn |
| `-ge` | Lớn hơn hoặc bằng |
| `-lt` | Nhỏ hơn |
| `-le` | Nhỏ hơn hoặc bằng |

```bash
[[ 1 -eq 1 ]]; echo $?    # 0 (true)
[[ 2 -lt 2 ]]; echo $?    # 1 (false)
```

### So sánh chuỗi

| Toán tử | Ý nghĩa |
|---|---|
| `=` hoặc `==` | Bằng |
| `!=` | Khác |
| `-z "$STRING"` | Chuỗi **rỗng** |
| `-n "$STRING"` | Chuỗi **không rỗng** |

### Kiểm tra file/thư mục
| Toán tử | Ý nghĩa |
|---|---|
| `-f` | Là file thường, tồn tại |
| `-d` | Là thư mục, tồn tại |
| `-r` | User hiện tại có quyền đọc |

> ⚠️ **Khoảng trắng bên trong `[[ ]]` là BẮT BUỘC** — thiếu khoảng trắng sẽ gây lỗi cú pháp.

---

## 13. Vòng lặp `while` và `until`

### `while` — chạy khi điều kiện còn ĐÚNG
```bash
x=1
while [[ $x -le 5 ]]; do
    echo "x is $x"
    x=$((x + 1))
done
```

### `until` — chạy khi điều kiện còn SAI (ngược với `while`)
```bash
x=5
until [[ $x -lt 1 ]]; do
    echo "x is $x"
    x=$((x - 1))
done
```
> 💡 Cả 2 đều **kiểm tra điều kiện TRƯỚC** khi chạy — nếu điều kiện không thỏa ngay từ đầu, vòng lặp **không chạy lần nào**.

---

## 14. Cấu trúc điều kiện `if` / `elif` / `else`

### `if/then`
```bash
if <CONDITION>; then
    <STATEMENT>
fi
```

### `if/then/else`
```bash
if <CONDITION>; then
    <STATEMENT>
else
    <STATEMENT>
fi
```

### `if/elif/else` (nhiều điều kiện)
```bash
if <CONDITION1>; then
    <STATEMENT>
elif <CONDITION2>; then
    <STATEMENT>
else
    <STATEMENT>
fi
```
> 💡 Bash kiểm tra **theo thứ tự**, dừng lại ở điều kiện **đầu tiên đúng**; nếu không có điều kiện nào đúng → chạy `else`.

### Ví dụ thực tế: chọn client database theo service đang chạy
```bash
systemctl is-active mariadb > /dev/null 2>&1
MARIADB_ACTIVE=$?
sudo systemctl is-active postgresql > /dev/null 2>&1
POSTGRESQL_ACTIVE=$?

if [[ "$MARIADB_ACTIVE" -eq 0 ]]; then
    mysql
elif [[ "$POSTGRESQL_ACTIVE" -eq 0 ]]; then
    psql
else
    sqlite3
fi
```

---

## 15. 📝 Bài thực hành: Vòng lặp & Điều kiện

**Mục tiêu:** Viết script SSH vào nhiều máy lấy hostname, kèm điều kiện phân biệt máy nào là `servera`.

1. Test thủ công: `ssh student@servera hostname`, `ssh student@serverb hostname`
2. Viết vòng lặp trực tiếp ở dòng lệnh:
   ```bash
   for host in servera serverb; do
       ssh student@${host} hostname
   done
   ```
3. Tạo `~/bin/printhostname.sh` (đảm bảo `~/bin` đã có trong `$PATH`):
   ```bash
   #!/usr/bin/bash
   for host in servera serverb; do
       ssh student@${host} hostname
       if [[ "${host}" == "servera" ]]; then
           echo "The host is servera"
       else
           echo "The host is not servera"
       fi
   done
   ```
4. `chmod +x ~/bin/printhostname.sh`
5. Chạy trực tiếp bằng tên (không cần `./` vì đã có trong PATH): `printhostname.sh`
6. Kiểm tra exit code: `echo $?` → `0`

---

## 16. Bảng tóm tắt nhanh

| Việc cần làm | Cú pháp |
|---|---|
| Gán biến shell | `VAR=value` |
| Export thành biến môi trường | `export VAR=value` |
| Lấy giá trị biến | `$VAR` hoặc `${VAR}` |
| Mảng | `arr=(a b c)`, truy cập `${arr[0]}`, tất cả `${arr[@]}` |
| Vòng lặp for | `for X in LIST; do ...; done` |
| Vòng lặp while | `while [[ COND ]]; do ...; done` |
| Vòng lặp until | `until [[ COND ]]; do ...; done` |
| Điều kiện | `if [[ COND ]]; then ...; elif ...; else ...; fi` |
| Kiểm tra file tồn tại | `[[ -f file ]]` |
| Kiểm tra chuỗi rỗng | `[[ -z "$STR" ]]` |
| Exit code lệnh trước | `$?` |
| Positional parameter thứ N (N≥10) | `${10}` (bắt buộc ngoặc) |

## 17. Các quy tắc "vàng" cần nhớ

1. ⚠️ Không có khoảng trắng quanh dấu `=` khi gán biến.
2. ✅ Nháy đơn `'...'` = literal hoàn toàn; nháy kép `"..."` = cho phép expand biến/lệnh.
3. ⚠️ Dùng `${VAR}` (có ngoặc) khi có ký tự bám ngay sau tên biến, tránh Bash hiểu sai tên biến.
4. ✅ Biến thường chỉ dùng được trong shell hiện tại; phải `export` mới truyền được cho chương trình con.
5. ⚠️ Tham số vị trí từ 10 trở lên **bắt buộc** dùng `${10}`, không thì Bash hiểu sai thành `$1` + `"0"`.
6. ✅ Khoảng trắng bên trong `[[ ]]` là **bắt buộc** — thiếu sẽ lỗi cú pháp.
7. ✅ `while`/`until` đều kiểm tra điều kiện **trước** mỗi lần lặp — có thể không chạy lần nào nếu điều kiện ban đầu không thỏa.
8. ✅ `if/elif/else` dừng ở điều kiện đúng **đầu tiên** theo thứ tự viết.
9. ✅ Luôn dùng `#!/usr/bin/bash` ở dòng đầu file script + `chmod +x` để chạy được như lệnh thường.
10. ✅ Nên redirect message lỗi ra STDERR (`>&2`) để tách biệt với output bình thường.