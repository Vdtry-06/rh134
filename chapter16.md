# Tổng Hợp: Cài Đặt Red Hat Enterprise Linux 10

> Tài liệu tổng hợp Chương 16 – Installing Red Hat Enterprise Linux. Gồm 2 phần: **Cài đặt tương tác (Anaconda GUI)** và **Tự động hóa cài đặt bằng Kickstart**.

---

## 1. Tổng quan Media Cài đặt

| Loại media | Đặc điểm |
|---|---|
| **Binary DVD (ISO đầy đủ)** | Chứa Anaconda + repo **BaseOS** + **AppStream** → cài được **không cần mạng** |
| **Boot ISO (nhỏ gọn)** | Chỉ chứa Anaconda, cần **mạng** để tải gói từ HTTP/FTP/NFS |
| **Source code** | Mã nguồn RHEL, dùng để build/phát triển phần mềm dựa trên nền RHEL |

**Kiến trúc hỗ trợ RHEL 10:** x86-64-v3 (AMD/Intel), ARMv8.0-A, POWER9 (IBM), z14 (IBM Z 64-bit).

**RHEL Image Builder:** Công cụ tạo **custom image** tùy biến để deploy lên cloud/VM — dùng qua `composer-cli` hoặc web console.

### Yêu cầu tối thiểu

- **Đĩa:** tối thiểu **10 GiB**
- **RAM** (tùy loại cài đặt):

| Loại cài đặt | x86-64-v3 / ARMv8.0-A / z14 | POWER9 |
|---|---|---|
| Local media (USB/DVD) | 1.5 GiB | 3 GiB |
| NFS network | 1.5 GiB | 3 GiB |
| HTTP/HTTPS/FTP network | 3 GiB | 4 GiB |

---

## 2. Cài đặt Tương tác bằng Anaconda (GUI)

### 2 phương thức cài đặt
- **Manual (thủ công):** Anaconda hỏi từng bước, người dùng tự chọn.
- **Automated (Kickstart):** Dùng file cấu hình → không cần tương tác (xem phần 3).

### Màn hình INSTALLATION SUMMARY — các mục cần cấu hình

| Mục | Nội dung |
|---|---|
| **Keyboard** | Layout bàn phím |
| **Language Support** | Ngôn ngữ bổ sung |
| **Time & Date** | Vùng/timezone, NTP hoặc set thủ công |
| **Connect to Red Hat** | Đăng ký subscription, chọn system purpose |
| **Installation Source** | Nguồn gói cài đặt (DVD hoặc mạng) |
| **Software Selection** | Base environment (vd `Minimal Install` = chỉ gói thiết yếu) |
| **Installation Destination** | Chọn đĩa cài; mặc định dùng **LVM tự động**; có thể Custom (vẫn cho auto-tạo partition) |
| **KDUMP** | Bật/tắt tính năng thu thập crash dump khi kernel crash |
| **Network & Host Name** | Cấu hình mạng, hostname |
| **Root Account** | ⚠️ Từ RHEL 10, **root account bị disable mặc định** — phải bật thủ công + đặt password |
| **User Creation** | Nếu root bị khóa → **bắt buộc** tạo user thường có quyền admin (thuộc nhóm `wheel`, dùng `sudo`) |

> ⚠️ **Điểm quan trọng RHEL 10:** Root account **mặc định bị tắt**. Nếu không bật root, **bắt buộc** phải tạo 1 user non-root có quyền sudo (thuộc `wheel`) trong bước User Creation.
> SSH root login mặc định **không cho phép password-based auth** (có thể bật lại trong Anaconda nếu cần).

### Cài qua Remote Desktop Protocol (RDP)
- RHEL 10 hỗ trợ điều khiển cài đặt từ xa qua **RDP** (thay thế VNC đã deprecated, hiệu năng tốt hơn).
- Kích hoạt bằng tham số kernel: `inst.rdp`
- Xác thực: `inst.rdp.username`, `inst.rdp.password` (nếu không cung cấp, Anaconda sẽ hỏi tương tác).
- Port mặc định: **5900 (TCP)**. Hỗ trợ **nhiều RDP client kết nối cùng lúc** (collaborative).

### Troubleshooting trong lúc cài đặt

| Phím tắt | Nội dung |
|---|---|
| `Ctrl+Alt+F1` | Vào **tmux terminal** (đa cửa sổ, text-based) |
| `Ctrl+Alt+F6` | Quay lại giao diện **GUI Anaconda** |
| `Ctrl+B 1` | (trong tmux) Trang thông tin chính của quá trình cài |
| `Ctrl+B 2` | Root shell — log cài đặt lưu ở `/tmp` |
| `Ctrl+B 3` | Nội dung `/tmp/anaconda.log` |
| `Ctrl+B 4` | Nội dung `/tmp/storage.log` |
| `Ctrl+B 5` | Nội dung `/tmp/program.log` |
| `Ctrl+B 6` | Nội dung `/tmp/packaging.log` |

> 💡 `Ctrl+Alt+F2` đến `F5` cũng cho root shell (tương thích ngược với bản RHEL cũ hơn).

### 📝 Bài thực hành: Cài đặt RHEL Tương tác
**Mục tiêu:** Cài RHEL 10 qua PXE boot, cấu hình mạng tĩnh, custom partition, tạo user + root.

Các bước chính:
1. Boot qua PXE → chọn ngôn ngữ
2. **Time & Date:** chọn Region/City, cấu hình NTP server riêng (`classroom.example.com`)
3. **Network & Host Name:** đặt hostname, cấu hình **IP tĩnh** (address/netmask/gateway/DNS) — quan trọng vì PXE mặc định cấp DHCP
4. **Installation Source:** xác nhận nguồn mạng đúng
5. **Installation Destination → Custom → Standard Partition:** để installer tự tạo scheme mặc định dựa theo dung lượng đĩa + RAM
6. **Software Selection:** chọn `Minimal Install`
7. **Root Account:** bật root, đặt password (dù cảnh báo password yếu vẫn chấp nhận được khi click Done 2 lần)
8. **User Creation:** tạo user `student`, giữ quyền admin (mặc định check sẵn)
9. Begin Installation → theo dõi qua tmux (`Ctrl+Alt+F1`, `Ctrl+B 2-6`) xem log
10. Reboot → chọn **Boot from local disk** (vì máy mặc định luôn PXE boot)
11. Kiểm tra sau khi cài:
    - `su - -c id` → xác nhận root hoạt động
    - `sudo id` → xác nhận user thường có quyền sudo
    - `lsblk` → xem lại scheme partition
    - `ip addr show`, `hostname`, `cat /etc/resolv.conf`, `ip route show` → xác nhận cấu hình mạng đúng như đã nhập

---

## 3. Tự động hóa Cài đặt bằng Kickstart

### Khái niệm
Kickstart = **1 file text** trả lời sẵn mọi câu hỏi cài đặt (partition, network, package, user...) → cài đặt **hoàn toàn tự động, không cần tương tác**. Tương tự "answer file" của Windows.

### Cấu trúc file Kickstart
```
# Comment bắt đầu bằng #
[Command section - các lệnh cấu hình]

%packages
... danh sách gói ...
%end

%pre
... script chạy TRƯỚC khi phân vùng ...
%end

%post
... script chạy SAU khi cài xong ...
%end
```

> 💡 Có thể có **nhiều section cùng loại** (vd 2 `%post`), chạy theo thứ tự xuất hiện.

### `%packages` — Chọn gói cài đặt
```
%packages
@^graphical-server-environment   # environment group (@^ = nhóm các package group)
vim*                             # package theo tên/wildcard
-hyperv*                         # dấu - = LOẠI TRỪ gói/group đó
%end
```
> ⚠️ Gói bị loại trừ (`-`) vẫn có thể được cài nếu là **dependency bắt buộc** của gói khác.

### `%pre` vs `%post` — khác biệt quan trọng

| | `%pre` | `%post` |
|---|---|---|
| Chạy khi nào | **Trước** khi phân vùng đĩa | **Sau** khi cài đặt xong |
| Môi trường | **Ngoài chroot** (ít lệnh khả dụng hơn) | **Trong chroot** (đầy đủ công cụ hệ thống mới) |
| Dùng để | Khởi tạo storage/network cần cho phần còn lại | Tùy biến hệ thống sau cài (thêm file, cấu hình...) |

> ⚠️ Copy script/RPM từ installation media **không hoạt động** trong `%post` (vì đang trong chroot) — việc này phải làm ở `%pre`.

### Các lệnh Kickstart quan trọng theo nhóm

**Nguồn cài đặt:**
```
url --url="http://classroom.example.com/rhel10.0/x86_64/dvd/"
repo --name="appstream" --baseurl=http://.../AppStream/
text                              # ép cài dạng text mode
```

**Phân vùng & Storage:**
```
clearpart --all --drives=vda,vdb          # xóa hết partition trước khi tạo mới
part /home --fstype=xfs --size=4096 --maxsize=8192 --grow
autopart                                  # tự tạo root+swap+boot (và /home nếu đĩa ≥50GB)
ignoredisk --drives=sdc                   # bỏ qua đĩa này, không đụng vào
bootloader --location=mbr --boot-drive=sda  # BẮT BUỘC
zerombr                                   # khởi tạo đĩa chưa format nhận diện được
```

**LVM (volgroup/logvol):**
```
part pv.01 --size=8192
volgroup myvg pv.01
logvol / --vgname=myvg --fstype=xfs --size=2048 --name=rootvol --grow
logvol /var --vgname=myvg --fstype=xfs --size=4096 --name=varvol
```

**Mạng:**
```
network --device=enp1s0 --bootproto=dhcp
firewall --enabled --service=ssh,http
```

**Ngôn ngữ/Vùng/Bảo mật (⚠️ BẮT BUỘC: `lang`, `keyboard`, `timezone`):**
```
lang en_US
keyboard --vckeymap=us
timezone --utc Europe/Amsterdam
timesource --ntp-server classroom.example.com
selinux --enforcing
```

**Người dùng & mật khẩu:**
```
rootpw --lock                              # khóa root (RHEL 10: mặc định optional vì root đã disable sẵn)
rootpw --plaintext redhat                  # hoặc đặt password rõ
rootpw --iscrypted $6$...                  # hoặc password đã mã hóa
group --name=admins --gid=10001
user --name=jdoe --gecos="John Doe" --groups=admins
```
> ⚠️ RHEL 10: nếu giữ root **locked**, **bắt buộc** tạo user thuộc nhóm `wheel` để có quyền quản trị.

**Khác:**
```
services --disabled=rsyslog --enabled=NetworkManager,firewalld
firstboot --disabled          # tắt GNOME Initial Setup lần đầu boot
logging --host=loghost.example.com
reboot / poweroff / halt      # hành động cuối cùng (mặc định: halt = chờ nhấn phím)
```

### 📄 Ví dụ file Kickstart hoàn chỉnh (rút gọn)
```
# version=RHEL10
graphical
keyboard --vckeymap=us --xlayouts='us'
lang en_US.UTF-8
timezone America/New_York --utc
network --bootproto=dhcp --device=enp1s0 --ipv6=auto --activate
network --hostname=serverb.lab.example.com
url --url="http://content.example.com/rhel10.0/x86_64/dvd/"

zerombr
clearpart --all --initlabel
autopart
ignoredisk --only-use=vda

rootpw --iscrypted --allow-ssh $y$...
user --groups=wheel --name=student --password=$y$... --iscrypted --gecos="student"

%packages
@^minimal-environment
vim-enhanced
-hyperv*
%end

%addon com_redhat_kdump --enable --reserve-mb='auto'
%end

%post
echo "This system was deployed using Kickstart on $(date)" > /etc/motd
%end
```

---

## 4. Quy trình Cài đặt Kickstart (Workflow)

```
1. Tạo file Kickstart
2. Publish file (HTTP/FTP/NFS/USB/CD/local disk)
3. Boot Anaconda, trỏ tới file Kickstart bằng inst.ks=
```

### Bước 1: Tạo file Kickstart

**2 cách:**
- **Kickstart Generator** (web): `https://access.redhat.com/labs/kickstartconfig` — giao diện hỏi-đáp, tự sinh file.
- **Tự viết bằng text editor** — nên bắt đầu từ file `/root/anaconda-ks.cfg` (mọi lần cài đặt đều tự sinh file này ghi lại cấu hình đã dùng) thay vì viết từ đầu.

### Kiểm tra cú pháp: `ksvalidator`
```bash
ksvalidator /tmp/anaconda-ks.cfg
```
- Kiểm tra keyword/option đúng cú pháp.
- **KHÔNG** kiểm tra: URL có hợp lệ không, tên package/group có tồn tại không, nội dung `%pre`/`%post` có đúng không.

### So sánh cú pháp giữa các phiên bản: `ksverdiff`
```bash
ksverdiff -f RHEL9 -t RHEL10
```
→ Liệt kê lệnh bị **xóa**, **deprecated**, **thêm mới** giữa 2 phiên bản.

> 💡 Cả `ksvalidator` và `ksverdiff` nằm trong package **`pykickstart`**.

### Bước 2: Publish file Kickstart
| Nơi lưu | Use case |
|---|---|
| Server HTTP/FTP/NFS | Môi trường enterprise, dùng chung nhiều máy |
| USB/CD-ROM | Thuận tiện hơn khi không có server sẵn |
| Local disk | Rebuild nhanh 1 máy đơn lẻ |

### Bước 3: Trỏ Anaconda tới file Kickstart
Thêm tham số kernel `inst.ks=` ở màn hình boot loader (nhấn `Tab` hoặc `C` để sửa dòng lệnh kernel):
```
inst.ks=http://server/dir/file
inst.ks=ftp://server/dir/file
inst.ks=nfs:server:/dir/file
inst.ks=hd:device:/dir/file
inst.ks=cdrom:device
```
→ Nhấn `Ctrl+X` để bắt đầu boot. Nếu file Kickstart đủ thông tin bắt buộc, Anaconda cài **hoàn toàn tự động không hỏi gì thêm**.

> 💡 Vẫn có thể dùng `Ctrl+Alt+F1`→`F6` để troubleshoot trong lúc cài Kickstart, y hệt cài thủ công.

---

## 5. 📝 Bài thực hành: Tự động hóa Cài đặt bằng Kickstart

**Mục tiêu:** Sửa 1 file Kickstart có sẵn, validate, publish qua HTTP, rồi cài máy `serverc` tự động qua PXE.

1. SSH vào `servera`, sửa file `kickstart.cfg`:
   - Đổi `graphical` → `text`
   - Thêm `@guest-agents` vào `%packages`
   - Thêm `firstboot --disable`
   - Thêm `%post` chạy `mandb` (cập nhật man page index) + ghi ngày cài vào `/etc/issue`
2. Cài `pykickstart`: `sudo dnf install pykickstart`
3. Validate: `ksvalidator ~/RH304/labs/installing-kickstart/kickstart.cfg`
4. Publish file: `sudo cp kickstart.cfg /var/www/html/` (dùng Apache có sẵn trên `servera`)
5. Trên `serverc`: reboot qua PXE, tại menu chọn **Install RHEL 10**, nhấn `Tab`, thêm:
   ```
   inst.ks=http://servera.lab.example.com/kickstart.cfg
   ```
6. Enter → cài đặt **tự động hoàn toàn**, theo dõi qua tmux nếu muốn
7. Sau khi xong, reboot → chọn **Boot from local disk**
8. Login, kiểm tra:
   - Thấy dòng ngày cài đặt trên login prompt (xác nhận `%post` chạy thành công)
   - `nmcli con show ens3 | grep ipv4.method` → `auto` (DHCP, vì Kickstart không set static)
   - `hostnamectl` → hostname = `localhost` (do không set DNS cho IP thuê từ DHCP)
   - `lsblk` → xác nhận dùng **LVM** (do `autopart`)
   - `apropos fstab` → có kết quả → xác nhận `mandb` trong `%post` đã chạy thành công

---

## 6. Bảng so sánh nhanh: Khi nào dùng gì?

| Tình huống | Cách nên dùng |
|---|---|
| Cài 1 máy đơn lẻ, cần tùy chỉnh linh hoạt lúc cài | **Cài tương tác (Anaconda GUI)** |
| Cần triển khai hàng loạt máy giống nhau | **Kickstart** |
| Cần điều khiển cài đặt từ xa | **RDP** (`inst.rdp`) |
| Cần debug lỗi cài đặt | `Ctrl+Alt+F1` → tmux → xem log trong `/tmp/*.log` |
| Cần kiểm tra cú pháp Kickstart trước khi dùng | `ksvalidator` |
| Cần biết khác biệt Kickstart giữa các bản RHEL | `ksverdiff` |

## 7. Các quy tắc "vàng" cần nhớ

1. ⚠️ RHEL 10: **root account mặc định bị disable** — phải bật thủ công hoặc tạo user thuộc `wheel`.
2. ✅ `Minimal Install` = chỉ cài gói thiết yếu — dùng khi cần hệ thống gọn nhẹ.
3. ✅ `%pre` chạy ngoài chroot (trước phân vùng); `%post` chạy trong chroot (sau khi cài xong).
4. ⚠️ Copy file từ installation media chỉ hoạt động trong `%pre`, KHÔNG hoạt động trong `%post`.
5. ✅ Các lệnh Kickstart **bắt buộc**: `lang`, `keyboard`, `timezone`, `bootloader`.
6. ✅ `ksvalidator` chỉ kiểm tra **cú pháp**, không kiểm tra URL/package/script có đúng logic không.
7. ✅ Bắt đầu viết Kickstart từ `/root/anaconda-ks.cfg` (file có sẵn sau mỗi lần cài) thay vì viết từ đầu.
8. ✅ Muốn Anaconda dùng Kickstart: thêm `inst.ks=LOCATION` vào dòng kernel command line lúc boot.