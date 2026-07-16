# Tổng Hợp: Quản Lý Bảo Mật Mạng trên RHEL 10

> Tài liệu tổng hợp Chương 14 – Managing Network Security. Gồm 2 phần: **Firewalld (quản lý tường lửa)** và **SELinux Port Labeling**.

---

## 1. Kiến trúc Firewall trên Linux

- Kernel dùng framework **netfilter** để lọc gói tin, NAT, port translation.
- **nftables** = framework phân loại gói tin xây trên netfilter, là **lõi firewall** của RHEL 10 (thay thế `iptables`/`ip6tables` cũ).
- **firewalld** = dịch vụ quản lý firewall **động**, là front-end khuyến nghị cho nftables.

> 💡 Có thể convert config `iptables` cũ sang `nftables` bằng `iptables-translate` / `ip6tables-translate`.

---

## 2. Firewalld — Khái niệm Zone

### Zone là gì?
- **Zone** = nhóm phân loại traffic mạng, mỗi zone có **danh sách port/service** riêng được cho phép/từ chối.
- Traffic được gán vào zone dựa theo: **source IP** của gói tin, hoặc **network interface** nhận gói tin.

### Thứ tự firewalld xác định zone cho 1 gói tin
```
1. Source IP có gán zone cụ thể? → dùng zone đó
2. Không có → Interface nhận gói có gán zone? → dùng zone đó
3. Không có → dùng ZONE MẶC ĐỊNH (default zone)
```
> 💡 "Default zone" không phải zone riêng biệt — là 1 trong các zone có sẵn được **chỉ định làm mặc định**. Ban đầu là **`public`**. Interface `lo` (loopback) mặc định gán vào zone **`trusted`**.

### Bảng các Zone dựng sẵn (từ ít mở → nhiều mở)

| Zone | Cấu hình mặc định |
|---|---|
| `drop` | **Drop hết** traffic đến (kể cả không phản hồi ICMP error) |
| `block` | **Từ chối** hết traffic đến (trừ traffic liên quan tới outgoing) |
| `dmz` | Từ chối trừ khi liên quan outgoing hoặc là `ssh` |
| `external` | Như `dmz`, cộng thêm: **masquerade** IPv4 traffic đi qua (NAT) |
| `public` | Từ chối trừ `ssh`, `dhcpv6-client` — **zone mặc định cho interface mới** |
| `work` | Từ chối trừ `ssh`, `ipp-client`, `dhcpv6-client` |
| `internal` | Giống `home` ban đầu |
| `home` | Từ chối trừ `ssh`, `mdns`, `ipp-client`, `samba-client`, `dhcpv6-client` |
| `trusted` | **Cho phép TẤT CẢ** traffic đến |

> 💡 Mọi zone đều **luôn cho phép**: traffic đến liên quan tới session mà hệ thống **tự khởi tạo** (outgoing) + toàn bộ **outgoing traffic**.

**Use case NetworkManager tự động đổi zone:** Máy laptop di chuyển giữa mạng nhà/công ty/quán café — NetworkManager có thể tự set zone tương ứng, giúp SSH mở ở nhà nhưng đóng ở mạng công cộng.

### Predefined Services — cấu hình sẵn cho dịch vụ phổ biến
Thay vì tự tra port, dùng tên service có sẵn:

| Service | Cấu hình |
|---|---|
| `ssh` | 22/tcp |
| `dhcpv6-client` | 546/udp (mạng fe80::/64) |
| `ipp-client` | 631/udp |
| `samba-client` | 137/udp, 138/udp |
| `mdns` | 5353/udp (multicast) |
| `cockpit` | 9090/tcp |

```bash
firewall-cmd --get-services    # liệt kê TẤT CẢ service có sẵn
```

---

## 3. Quản lý Firewalld — Command Line

### Nguyên tắc runtime vs permanent

| | Runtime | Permanent |
|---|---|---|
| Áp dụng | Ngay lập tức | Chỉ lưu, chưa áp dụng |
| Mất khi reboot? | **Có** (nếu không đi kèm `--permanent`) | Không |
| Kích hoạt | Tự động | Cần `firewall-cmd --reload` |

> ⚠️ Mặc định `firewall-cmd` chỉ sửa **runtime**. Muốn bền vững phải thêm `--permanent`, rồi `--reload` để áp dụng ngay (không cần đợi reboot).

### Các lệnh `firewall-cmd` quan trọng

| Lệnh | Ý nghĩa |
|---|---|
| `--get-default-zone` | Xem zone mặc định |
| `--set-default-zone=ZONE` | Đặt zone mặc định (sửa cả runtime lẫn permanent) |
| `--get-zones` | Liệt kê tất cả zone có sẵn |
| `--get-active-zones` | Liệt kê zone **đang dùng** (kèm interface/source gán vào) |
| `--add-source=CIDR [--zone=Z]` | Gán toàn bộ traffic từ IP/mạng vào zone |
| `--remove-source=CIDR [--zone=Z]` | Bỏ gán source khỏi zone |
| `--add-interface=IFACE [--zone=Z]` | Gán interface vào zone |
| `--change-interface=IFACE [--zone=Z]` | Đổi zone của interface |
| `--list-all [--zone=Z]` | Xem toàn bộ cấu hình của 1 zone |
| `--list-all-zones` | Xem cấu hình của **mọi** zone |
| `--add-service=SERVICE [--zone=Z]` | Cho phép 1 service |
| `--add-port=PORT/PROTOCOL [--zone=Z]` | Cho phép 1 port/protocol cụ thể |
| `--remove-service` / `--remove-port` | Gỡ bỏ tương ứng |
| `--reload` | Nạp lại cấu hình permanent → runtime |
| `--runtime-to-permanent` | Lưu cấu hình runtime hiện tại thành permanent |

> 💡 Nếu không chỉ định `--zone=`, lệnh áp dụng cho **zone mặc định**.

### Ví dụ thực tế
```bash
firewall-cmd --set-default-zone=dmz
firewall-cmd --permanent --zone=internal --add-source=192.168.0.0/24
firewall-cmd --permanent --zone=internal --add-service=mysql
firewall-cmd --reload
```

> 💡 Nếu cấu hình cơ bản không đủ, có thể dùng **rich-rules** (phức tạp hơn) hoặc **Direct Configuration** (nft syntax thô) — nằm ngoài phạm vi cơ bản.

### Quản lý qua Web Console
`Networking` → `Edit rules and zones` → chọn zone → `Add services` → tick chọn dịch vụ → `Add services`.

---

## 4. 📝 Bài thực hành: Quản lý Firewall Server

**Mục tiêu:** Cài Apache (`httpd`), phát hiện bị chặn bởi firewall, mở port HTTPS.

1. `dnf install httpd mod_ssl`
2. Tạo nội dung test: `echo 'I am servera.' > /var/www/html/index.html`
3. `systemctl enable --now httpd`
4. Từ máy khác: `curl http://servera...` và `curl -k https://servera...` → **cả 2 đều FAIL** (firewall chặn)
5. Kiểm tra firewalld đang chạy: `systemctl status firewalld`
6. Xác nhận default zone: `firewall-cmd --get-default-zone` → `public`
7. Mở HTTPS: `firewall-cmd --permanent --add-service=https`
8. `firewall-cmd --reload`
9. Kiểm tra: `firewall-cmd --permanent --zone=public --list-all` → thấy `https` trong danh sách `services`
10. Test lại: `curl http://servera...` (port 80) → **vẫn fail** (chưa mở service `http`); `curl -k https://servera...` (port 443) → **thành công**, thấy `I am servera.`

---

## 5. SELinux Port Labeling

### Khái niệm
- SELinux không chỉ label **file** và **process** — mà còn label **network port**.
- Mỗi service có **targeted policy** quy định process nào được bind vào port có label nào.
- Ví dụ: SSH policy → port 22/TCP có label `ssh_port_t`; HTTP policy → port 80/443 có label `http_port_t`.

**Use case:** Ngăn 1 service "mạo danh" chiếm port của service hợp lệ khác — nếu process cố bind vào port mà SELinux context không khớp policy → **bị chặn**.

### Xem port label hiện có

```bash
grep gopher /etc/services              # xem port service map từ /etc/services (không phải SELinux)
semanage port -l                        # liệt kê TẤT CẢ port label SELinux
semanage port -l | grep ftp             # lọc theo tên service
semanage port -l | grep -w 70           # lọc theo số port cụ thể
```
> 💡 1 port label có thể xuất hiện nhiều dòng — mỗi dòng cho 1 protocol (tcp/udp) khác nhau.

### Gán/Xóa/Sửa Port Label

| Thao tác | Lệnh |
|---|---|
| **Thêm** port mới vào 1 label có sẵn | `semanage port -a -t TYPE -p tcp\|udp PORT` |
| **Xóa** binding của port | `semanage port -d -t TYPE -p tcp\|udp PORT` |
| **Sửa** (đổi label của port đã gán) | `semanage port -m -t NEW_TYPE -p tcp\|udp PORT` |
| Xem các thay đổi so với default policy | `semanage port -l -C` |

Ví dụ:
```bash
semanage port -a -t gopher_port_t -p tcp 71     # cho service gopher nghe thêm ở port 71
semanage port -d -t gopher_port_t -p tcp 71     # xóa binding đó
semanage port -m -t http_port_t -p tcp 71       # đổi port 71 sang label http (thay vì xóa rồi thêm)
```

> ⚠️ **Quan trọng:** Hầu hết service có sẵn **SELinux policy module** định nghĩa port mặc định của chúng — **KHÔNG thể đổi port mặc định** bằng `semanage`. Chỉ dùng `semanage port` để **thêm port mới** (nonstandard) vào 1 label có sẵn, không phải sửa port gốc của service.

### Tài liệu SELinux theo từng service
```bash
dnf -y install selinux-policy-doc
man -k _selinux       # tìm man page dạng "servicename_selinux"
```

---

## 6. 📝 Bài thực hành: SELinux Port Labeling

**Mục tiêu:** Web app chạy port **82/TCP** (nonstandard) bị SELinux chặn → gán label đúng, rồi mở firewall tương ứng.

1. `systemctl restart httpd` → **THẤT BẠI**
2. `systemctl status -l httpd` → thấy lỗi:
   ```
   (13)Permission denied: AH00072: could not bind to address [::]:82
   ```
3. Dùng công cụ phân tích SELinux audit log:
   ```bash
   sealert -a /var/log/audit/audit.log
   ```
   → Kết quả gợi ý:
   ```
   SELinux is preventing /usr/sbin/httpd from name_bind access on tcp_socket port 82.
   Do: semanage port -a -t PORT_TYPE -p tcp 82
   (PORT_TYPE: http_cache_port_t, http_port_t, jboss_..., puppet_port_t...)
   ```
4. Xác định type phù hợp: `semanage port -l | grep http` → thấy `http_port_t` chứa port 80, 81, 443... → đây là type đúng cho web server
5. Gán port 82 vào `http_port_t`:
   ```bash
   semanage port -a -t http_port_t -p tcp 82
   ```
6. `systemctl restart httpd.service` → **thành công**
7. Test local: `curl http://servera.lab.example.com:82` → `Hello` (thành công)
8. Test từ máy khác: `curl http://servera...:82` → **THẤT BẠI** (vì **firewall chưa mở port này**, khác với vấn đề SELinux)
9. Mở port trên firewall:
   ```bash
   firewall-cmd --permanent --add-port=82/tcp
   firewall-cmd --reload
   ```
10. Test lại từ máy khác → **thành công**

> 💡 **Bài học quan trọng:** Đây là ví dụ điển hình về **2 lớp bảo mật độc lập** — SELinux (process có được bind port không) và Firewall (traffic từ mạng có được vào port không). Cần xử lý **cả 2** thì service mới truy cập được từ xa.

---

## 7. Bảng so sánh nhanh: Khi nào dùng gì?

| Tình huống | Công cụ |
|---|---|
| Chặn/mở truy cập theo IP/mạng nguồn | Firewalld **zone + source** |
| Chặn/mở theo service/port cụ thể | Firewalld **service/port** |
| Service chạy port khác chuẩn, bị lỗi "Permission denied" khi bind | **SELinux port labeling** (`semanage port`) |
| Không truy cập được dù server đã lắng nghe đúng port | Kiểm tra **cả SELinux** (`sealert`) **và firewall** (`firewall-cmd --list-all`) |
| Máy laptop cần đổi rule theo mạng đang kết nối | NetworkManager tự set zone |

## 8. Các quy tắc "vàng" cần nhớ

1. ✅ Zone mặc định của firewalld = `public`; `lo` (loopback) luôn ở zone `trusted`.
2. ✅ Chỉ có zone `trusted` cho phép **mọi** traffic mặc định — các zone khác đều có whitelist.
3. ⚠️ `firewall-cmd` không có `--permanent` chỉ sửa **runtime** — mất khi reload/reboot.
4. ✅ Sau khi thêm `--permanent`, luôn cần `firewall-cmd --reload` để áp dụng ngay (không cần chờ reboot).
5. ⚠️ `semanage port` chỉ dùng để thêm port **nonstandard** vào label có sẵn — **không sửa được** port mặc định của service (đã định nghĩa trong policy module).
6. ✅ Dùng `sealert -a /var/log/audit/audit.log` để chẩn đoán nhanh khi nghi ngờ SELinux chặn — kết quả thường gợi ý luôn câu lệnh `semanage` cần chạy.
7. ⚠️ Sau khi sửa SELinux label, **vẫn phải kiểm tra firewall riêng** — 2 lớp bảo mật độc lập, cả 2 đều cần đúng thì service mới truy cập được từ xa.