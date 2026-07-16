# Tổng Hợp: Truyền File Giữa Các Hệ Thống trên RHEL 10

> Tài liệu tổng hợp Chương 8 – Transferring Files. Gồm 3 công cụ: **SFTP** (tương tác), **SCP** (một dòng lệnh), và **rsync** (đồng bộ thông minh).

---

## 1. Tổng quan chung

| Công cụ | Đặc điểm | Dùng khi nào |
|---|---|---|
| **sftp** | Phiên tương tác, giống FTP nhưng bảo mật qua SSH | Cần duyệt/thao tác nhiều file qua lại như 1 phiên làm việc |
| **scp** | Copy 1 dòng lệnh, không tương tác | Copy nhanh 1 lần, không cần duyệt |
| **rsync** | Đồng bộ thông minh, chỉ truyền **phần thay đổi** | Đồng bộ lặp lại nhiều lần, dữ liệu lớn, cần hiệu quả |

Tất cả đều dùng **SSH** làm nền tảng bảo mật (xác thực + mã hóa dữ liệu).

**Cú pháp vị trí từ xa chung:** `user@host:path` (phần `user@` có thể bỏ qua, mặc định dùng user hiện tại của máy local).

---

## 2. SFTP (Secure File Transfer Program)

### Mở phiên tương tác
```bash
sftp remoteuser@remotehost
```
→ Vào chế độ `sftp>` — giống 1 shell nhỏ hoạt động trên hệ thống từ xa.

### Các lệnh trong phiên sftp

| Lệnh | Ý nghĩa |
|---|---|
| `ls`, `cd`, `mkdir`, `rmdir`, `pwd` | Thao tác trên **hệ thống từ xa** |
| `lpwd`, `lcd` | Thao tác trên **hệ thống local** (thêm chữ `l` phía trước) |
| `put file` | **Tải lên** (local → remote) |
| `put -r dir` | Tải lên **cả thư mục** (đệ quy) |
| `get file` | **Tải xuống** (remote → local) |
| `get -r dir` | Tải xuống **cả thư mục** (đệ quy) |
| `help` | Xem danh sách lệnh hỗ trợ |
| `exit` / `bye` | Thoát phiên |

### Copy 1 dòng lệnh (không cần phiên tương tác)
```bash
sftp remoteuser@remotehost:/home/remoteuser/remotefile
```
> ⚠️ Cách này **chỉ tải xuống được (`get`)**, KHÔNG hỗ trợ tải lên (`put`) theo kiểu 1 dòng lệnh.

### 📝 Bài thực hành: Truyền file bằng SFTP
**Mục tiêu:** Copy thư mục `/etc/ssh` từ `serverb` về `servera` bằng cả sftp và scp để so sánh.

**Phần SFTP:**
1. `mkdir ~/serverbackup1` trên `servera`
2. Mở phiên: `sftp root@serverb`
3. Đổi thư mục local đích: `lcd /home/student/serverbackup1/`
4. Tải xuống đệ quy: `get -r /etc/ssh`
5. `exit`
6. Kiểm tra bằng `ls -lR ~/serverbackup1`

---

## 3. SCP (Secure Copy)

### Cú pháp cơ bản
```bash
# Local → Remote
scp file1 file2 remoteuser@remotehost:/path/dich

# Remote → Local
scp remoteuser@remotehost:/path/file /path/local
```

### Copy đệ quy cả thư mục
```bash
scp -r root@serverb:/etc/ssh ~/serverbackup2
```

### ⚠️ Lưu ý quan trọng (RHEL 10)
- Từ RHEL 10, `scp` **thực chất chạy qua giao thức SFTP** bên dưới (không còn dùng giao thức SCP cũ theo mặc định).
- Giao thức SCP cũ (`legacy SCP`) có **lỗ hổng code injection** (CVE-2020-15778) → **Red Hat khuyến cáo không dùng**.
- Muốn ép dùng giao thức SCP cũ: thêm cờ `-O` (không khuyến khích).
- Nếu file `/etc/ssh/disable_scp` tồn tại → giao thức `scp` bị **vô hiệu hóa hoàn toàn** (kể cả `-O`), nhưng `sftp` vẫn hoạt động bình thường.

### 📝 Bài thực hành: Truyền file bằng SCP (tiếp nối bài trên)
```bash
mkdir ~/serverbackup2
scp -r root@serverb:/etc/ssh ~/serverbackup2
```
→ Kiểm tra bằng `ls -lR ~/serverbackup2`, kết quả tương đương cách dùng sftp.

---

## 4. Rsync — Đồng bộ thông minh

### Ưu điểm nổi bật
- Chỉ truyền **phần dữ liệu thay đổi** giữa 2 lần đồng bộ → lần đầu mất thời gian như copy thường, nhưng các lần sau **nhanh hơn rất nhiều**.
- **Use case lý tưởng:** sao lưu log định kỳ, đồng bộ mã nguồn, backup tăng dần (incremental backup).

### Cú pháp cơ bản
```bash
rsync -av NGUỒN ĐÍCH
```
- **Nguồn/đích** có thể là: local↔remote, remote↔local, hoặc local↔local (2 thư mục trên cùng máy).
- Định dạng remote giống sftp/scp: `user@host:path`

### Các option quan trọng

| Option | Ý nghĩa |
|---|---|
| `-n` (`--dry-run`) | **Chạy thử**, chỉ hiển thị sẽ làm gì mà **không thực sự thay đổi** gì — nên chạy trước khi đồng bộ thật để tránh mất/ghi đè file quan trọng |
| `-v` (`--verbose`) | Hiện chi tiết tiến trình |
| `-a` (`--archive`) | Chế độ archive — bật hàng loạt option để giữ nguyên đặc tính file |
| `-H` | Giữ hard link (không nằm trong `-a` mặc định vì tốn thời gian) |
| `-A` | Giữ ACL |
| `-X` | Giữ SELinux context |

### `-a` (archive mode) tương đương với:

| Option con | Ý nghĩa |
|---|---|
| `-r` | Đồng bộ đệ quy toàn bộ cây thư mục |
| `-l` | Đồng bộ symbolic link |
| `-p` | Giữ nguyên quyền (permissions) |
| `-t` | Giữ nguyên timestamp |
| `-g` | Giữ nguyên group ownership |
| `-o` | Giữ nguyên owner |
| `-D` | Giữ nguyên device file |

> ⚠️ `-a` **không** giữ hard link — phải thêm `-H` riêng nếu cần.

### Ví dụ minh họa

```bash
# Đồng bộ local → remote
rsync -av /var/log hosta:/tmp

# Đồng bộ remote → local
rsync -av hosta:/var/log /tmp

# Đồng bộ 2 thư mục trên cùng máy (cần quyền root để giữ ownership)
sudo rsync -av /var/log /tmp
```

> 🔑 **Muốn giữ nguyên ownership ở đích → phải là root** (nếu đích là remote, xác thực bằng root; nếu đích là local, chạy `rsync` với quyền root).

### ⚠️ Cực kỳ quan trọng: Dấu `/` ở cuối đường dẫn nguồn

| Cú pháp | Kết quả |
|---|---|
| `rsync -av /var/log hosta:/tmp` (không có `/` cuối) | Copy **cả thư mục `log`** vào `/tmp` → thành `/tmp/log/...` |
| `rsync -av /var/log/ hosta:/tmp` (**có** `/` cuối) | Chỉ copy **nội dung bên trong** `/var/log/` thẳng vào `/tmp/...` (không có thư mục `log` bao ngoài) |

> 💡 Bash tab-completion **tự động thêm `/`** vào cuối tên thư mục — cần chú ý khi gõ tay để tránh sai kết quả.

### 📝 Bài thực hành: Đồng bộ nội dung giữa các hệ thống
**Mục tiêu:** Đồng bộ `/var/log` từ `servera` sang `/home/student/serverlogs` trên `serverb`, minh họa rsync chỉ truyền phần thay đổi ở lần sau.

1. Trên `serverb`: `mkdir ~/serverlogs`
2. Trên `servera` (với quyền root): đồng bộ lần đầu
   ```bash
   rsync -av /var/log student@serverb:/home/student/serverlogs
   ```
   → Toàn bộ file được truyền (lần đầu).
3. Tạo thay đổi nhỏ để test: `logger "Log files synchronized"` (ghi thêm 1 dòng vào `/var/log/messages`)
4. Chạy lại đúng lệnh rsync lần 2:
   ```bash
   rsync -av /var/log student@serverb:/home/student/serverlogs
   ```
   → Lần này **chỉ truyền 2 file thay đổi** (`messages`, `audit.log`) thay vì toàn bộ — thấy rõ rsync tiết kiệm băng thông ra sao.
5. Kiểm tra kết quả trên `serverb`: `tail -n 5 ~/serverlogs/log/messages` → thấy dòng log mới nhất.

---

## 5. Bảng so sánh nhanh: Khi nào dùng gì?

| Tình huống | Công cụ nên dùng |
|---|---|
| Cần duyệt qua lại nhiều file/thư mục từ xa như 1 phiên làm việc | **sftp** |
| Copy nhanh 1-2 file hoặc thư mục, không cần tương tác | **scp** |
| Đồng bộ định kỳ, dữ liệu lớn, muốn tiết kiệm băng thông | **rsync** |
| Backup tăng dần (incremental backup) | **rsync** |
| Chỉ cần tải 1 file remote về, không cần mở phiên | `sftp user@host:path` (dạng 1 dòng, chỉ hỗ trợ `get`) |

## 6. Các quy tắc "vàng" cần nhớ

1. ✅ Trong phiên `sftp`, thêm chữ `l` phía trước lệnh (`lpwd`, `lcd`) để thao tác trên máy **local** thay vì remote.
2. ✅ `sftp` dạng 1 dòng lệnh **chỉ tải xuống được**, không tải lên được.
3. ⚠️ Từ RHEL 10, `scp` chạy qua SFTP; **tránh dùng cờ `-O`** (giao thức SCP cũ có lỗ hổng bảo mật).
4. ✅ Luôn chạy `rsync -n` (dry-run) trước khi đồng bộ thật với dữ liệu quan trọng.
5. ⚠️ Chú ý dấu `/` cuối đường dẫn nguồn trong `rsync` — quyết định có tạo thêm thư mục con ở đích hay không.
6. ✅ Muốn giữ nguyên owner/group ở đích khi `rsync` → phải chạy với quyền **root**.
7. ✅ `-a` (archive mode) không giữ hard link — cần thêm `-H` nếu muốn giữ.