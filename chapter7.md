# Tổng Hợp: Archive File (Nén & Lưu trữ) trên RHEL 10

> Tài liệu tổng hợp Chương 7 – Archiving Files. Trọng tâm: công cụ **tar** để tạo, liệt kê, giải nén archive (có/không nén).

---

## 1. Khái niệm Archive

- **Archive** = 1 file (hoặc thiết bị: băng từ, USB...) chứa gộp nhiều file bên trong.
- Tương tự khái niệm `.zip` trên các hệ điều hành khác.
- **Use case:** Sao lưu cá nhân (backup), gói nhiều file để chuyển qua mạng khi không có `rsync` hoặc khi cần đơn giản hóa việc truyền file.
- Có thể tạo archive **có nén** hoặc **không nén**.

---

## 2. Công cụ `tar`

### 3 hành động chính (bắt buộc chọn 1)

| Option | Ý nghĩa |
|---|---|
| `-c` / `--create` | Tạo archive |
| `-t` / `--list` | Liệt kê nội dung archive |
| `-x` / `--extract` | Giải nén archive |

### Các option chung thường dùng

| Option | Ý nghĩa |
|---|---|
| `-v` / `--verbose` | Hiện tên file đang xử lý |
| `-f` / `--file` | Chỉ định tên file archive (đi kèm sau đó) |
| `-p` / `--preserve-permissions` | Giữ nguyên quyền gốc khi giải nén |
| `--xattrs` | Lưu extended attributes |
| `--selinux` | Lưu SELinux context |
| `--acls` | Lưu POSIX ACL |

> ⚠️ Mặc định, ACL, SELinux context, và extended attributes **không được lưu** trừ khi chỉ định rõ các option trên.

### Các option chọn thuật toán nén

| Option | Thuật toán | Đuôi file |
|---|---|---|
| `-a` / `--auto-compress` | Tự nhận diện theo đuôi file | (tùy đuôi) |
| `-z` / `--gzip` | gzip | `.tar.gz` |
| `-j` / `--bzip2` | bzip2 | `.tar.bz2` |
| `-J` / `--xz` | xz | `.tar.xz` |

**So sánh 3 thuật toán nén:**

| Thuật toán | Đặc điểm |
|---|---|
| **gzip** | Cũ nhất, **nhanh nhất**, phổ biến rộng rãi |
| **bzip2** | Nén **nhỏ hơn** gzip, nhưng ít phổ biến hơn |
| **xz** | Mới nhất, **tỷ lệ nén tốt nhất** trong 3 loại |

> 💡 Dữ liệu đã nén sẵn (ảnh, file RPM...) thường **không nén nhỏ thêm được nữa** dù dùng thuật toán nào.

---

## 3. Tạo Archive

### Không nén
```bash
tar -cf mybackup.tar myapp1.log myapp2.log myapp3.log
```
- Cần quyền **đọc (read)** trên các file được archive.
- Với thư mục (vd `/etc`), user thường sẽ **bị loại trừ** các file/thư mục con mà họ không có quyền đọc — chỉ root mới archive được toàn bộ.

### Có nén
```bash
tar -czf /root/etcbackup.tar.gz /etc      # gzip
tar -cjf /root/logbackup.tar.bz2 /var/log # bzip2
tar -cJf /root/sshconfig.tar.xz /etc/ssh  # xz
```

### 📌 Lưu ý quan trọng về đường dẫn
- `tar` **tự động bỏ dấu `/` đầu tiên** khi archive đường dẫn tuyệt đối:
  ```
  tar: Removing leading `/' from member names
  ```
- Lý do: lưu bằng **đường dẫn tương đối** để khi giải nén **không ghi đè** file hệ thống hiện có (an toàn hơn) — có thể giải nén vào thư mục mới tùy ý.

---

## 4. Liệt kê nội dung Archive

```bash
tar -tf /root/etc.tar
tar -tzf /root/etc.tar.gz     # với archive nén, có thể vẫn dùng -tf (tar tự nhận diện nén qua header)
```
- Không bắt buộc phải chỉ rõ loại nén khi liệt kê — `tar` tự đọc từ header của file.

---

## 5. Giải nén Archive

```bash
mkdir /root/etcbackup
cd /root/etcbackup
tar -xf /root/etc.tar
```

**Quy tắc về quyền sở hữu khi giải nén:**
- Root giải nén → **giữ nguyên** user/group gốc.
- User thường giải nén → user đó trở thành **chủ sở hữu mới** của file.

**Quy tắc về permission (quyền truy cập file):**
- Mặc định, `umask` hiện tại sẽ áp dụng lên file vừa giải nén (có thể khác quyền gốc).
- Dùng `-p` để **giữ nguyên quyền gốc** đã lưu trong archive (mặc định **tự động bật** nếu chạy với quyền root).
  ```bash
  tar -xpf /home/user/myscripts.tar
  ```

**Nên giải nén vào thư mục rỗng** để tránh ghi đè file hiện có.

### ⚠️ Lỗi thường gặp: chỉ định sai loại nén
```bash
tar -xzf /root/etcbackup.tar.xz
# → gzip: stdin: not in gzip format
```
→ Thực ra `tar` **có thể tự nhận diện nén** mà không cần chỉ định `-z/-j/-J`; nếu chỉ định sai loại thì sẽ báo lỗi không khớp định dạng.

---

## 6. Công cụ nén độc lập (không tạo archive)

| Lệnh nén | Lệnh giải nén | Ghi chú |
|---|---|---|
| `gzip` | `gunzip` | Chỉ nén **1 file**, không gộp nhiều file |
| `bzip2` | `bunzip2` | Tương tự |
| `xz` | `unxz` | Tương tự |

> Muốn nén **nhiều file/thư mục** thành 1 file → phải dùng `tar` kèm option nén, không dùng các lệnh độc lập này trực tiếp.

### Xem kích thước gốc trước khi giải nén (kiểm tra đủ dung lượng đĩa)
```bash
gzip -l file.tar.gz
xz -l file.tar.xz
```
→ Hiển thị dung lượng nén, dung lượng gốc, và tỷ lệ nén — hữu ích để chắc chắn có đủ ổ đĩa trống trước khi giải nén.

---

## 7. 📝 Bài thực hành: Quản lý Archive Tar Nén

**Mục tiêu:** Tạo archive `/etc` bằng 4 cách (không nén + 3 loại nén), so sánh dung lượng, và xác minh archive hợp lệ bằng cách giải nén thử.

### Các bước chính

1. **Tạo 4 loại archive từ `/etc`:**
   ```bash
   tar -cf  /tmp/etc.tar     /etc   # không nén
   tar -czf /tmp/etc.tar.gz  /etc   # gzip
   tar -cjf /tmp/etc.tar.bz2 /etc   # bzip2
   tar -cJf /tmp/etc.tar.xz  /etc   # xz
   ```

2. **So sánh dung lượng:**
   ```bash
   ls -lh /tmp/etc.tar*
   ```
   Kết quả mẫu:
   | File | Dung lượng |
   |---|---|
   | `etc.tar` (không nén) | 22M |
   | `etc.tar.gz` | 5.5M |
   | `etc.tar.bz2` | 4.7M |
   | `etc.tar.xz` | 4.1M |

   → **xz nén tốt nhất**, tiếp đến bzip2, rồi gzip — đúng như lý thuyết.

3. **Kiểm tra nội dung archive nén:**
   ```bash
   tar -tzf /tmp/etc.tar.gz
   ```

4. **Xác minh archive hợp lệ bằng cách giải nén thử vào thư mục mới:**
   ```bash
   mkdir /backuptest
   cd /backuptest
   tar -xzf /tmp/etc.tar.gz
   ls -l          # → thấy thư mục etc/
   ls -l etc      # → thấy đầy đủ file cấu hình gốc
   ```

---

## 8. Bảng tóm tắt lệnh nhanh

| Việc cần làm | Lệnh |
|---|---|
| Tạo archive không nén | `tar -cf ten.tar file1 file2 ...` |
| Tạo archive gzip | `tar -czf ten.tar.gz thu_muc/` |
| Tạo archive bzip2 | `tar -cjf ten.tar.bz2 thu_muc/` |
| Tạo archive xz | `tar -cJf ten.tar.xz thu_muc/` |
| Liệt kê nội dung | `tar -tf ten.tar` |
| Giải nén (tự nhận diện nén) | `tar -xf ten.tar.gz` |
| Giải nén giữ nguyên quyền gốc | `tar -xpf ten.tar` |
| Xem dung lượng gốc trước khi giải nén | `gzip -l` / `xz -l` |

## 9. Các quy tắc "vàng" cần nhớ

1. ✅ `tar` tự động bỏ dấu `/` đầu của đường dẫn tuyệt đối → lưu dạng tương đối để an toàn khi giải nén.
2. ✅ Luôn giải nén vào **thư mục rỗng** để tránh ghi đè nhầm file quan trọng.
3. ✅ Không cần chỉ định loại nén khi **liệt kê** hoặc **giải nén** — `tar` tự nhận diện qua header; chỉ định sai sẽ gây lỗi.
4. ✅ Muốn giữ ACL/SELinux/extended attributes → phải thêm `--acls`, `--selinux`, `--xattrs` (không mặc định).
5. ✅ `xz` nén tốt nhất nhưng chậm hơn; `gzip` nhanh nhất nhưng nén kém nhất — chọn theo nhu cầu (tốc độ vs. dung lượng).
6. ✅ Dùng `gzip -l` hoặc `xz -l` để kiểm tra đủ dung lượng đĩa trước khi giải nén file lớn.