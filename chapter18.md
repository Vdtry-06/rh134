# Tổng Hợp: Làm Việc với Image Mode cho RHEL 10

> Tài liệu tổng hợp Chương 18 – Working with Image-based Red Hat Enterprise Linux. Gồm 4 phần: **Tổng quan Image Mode**, **Tạo Bootable Image**, **Cài đặt bằng Image Mode**, và **Quản lý hệ thống Image Mode**.

---

## 1. Image Mode là gì?

### Khái niệm
- **Image mode** = cách triển khai & quản lý RHEL 10 **mới**, dùng công cụ container-native (Podman, Containerfile...) để build/deploy/quản lý **toàn bộ hệ điều hành** — không chỉ ứng dụng.
- Dựa trên **bootable container image (bootc)**: khác container ứng dụng thông thường, ảnh bootc phải chứa **kernel, boot loader**, và các thành phần OS khác.
- Red Hat cung cấp base image **`rhel-bootc`** cho x86_64 và ARM 64-bit.

### So sánh Package Mode vs Image Mode

| | Package Mode (truyền thống) | Image Mode (mới) |
|---|---|---|
| Cài đặt | `dnf install` từng gói | Deploy 1 **container image** hoàn chỉnh |
| Cập nhật | Từng gói riêng lẻ | **Atomic** — cả image thay đổi cùng lúc, cần reboot |
| Root FS | Mutable (ghi được) | **Immutable** (chỉ đọc), trừ `/etc` và `/var` |
| Rollback | Phức tạp, không đồng bộ | Đơn giản — `bootc rollback` |
| Drift | Dễ bị "trôi" cấu hình theo thời gian | Hạn chế nhờ image cố định |
| Định nghĩa hệ thống | Nằm rải rác (script, docs) | Gói gọn trong **1 file Containerfile** (version control được) |

### Lợi ích chính của Image Mode
1. **Giảm infrastructure drift** — hệ thống không "trôi" dần khỏi trạng thái mong muốn.
2. **Immutable OS** — root FS chỉ đọc, chỉ `/etc` và `/var` là mutable → ổn định + bảo mật hơn.
3. **Version control được** — Containerfile là blueprint, dùng Git để quản lý version.
4. **Scale tốt hơn** — build 1 lần, deploy hàng loạt, update chỉ cần thêm layer.
5. **Troubleshooting đơn giản** — update là atomic, rollback dễ dàng khi có vấn đề.

### Luồng làm việc Image Mode
```
1. Build: Viết Containerfile → dùng Podman build ra bootc image
2. Deploy: Push image lên registry → cài lên bare-metal/VM (qua Kickstart) hoặc cloud (qua bootc-image-builder)
3. Manage: Cập nhật image gốc → hệ thống tự fetch update từ registry (hoặc thủ công qua `bootc upgrade`)
```

---

## 2. Tạo Bootable Container Image

### Containerfile cho Image Mode — khác gì so với container ứng dụng thông thường?

**Container ứng dụng thông thường (ví dụ):**
```dockerfile
FROM registry.redhat.io/ubi10
RUN dnf install -y httpd
COPY index.html /var/www/html/index.html
EXPOSE 80
ENTRYPOINT ["/usr/sbin/httpd", "-DFOREGROUND"]
```

**Containerfile cho bootc (Image Mode):**
```dockerfile
FROM registry.redhat.io/rhel10/rhel-bootc:latest
RUN dnf -y install httpd mod_ssl && dnf clean all
RUN systemctl enable httpd && firewall-cmd --add-service=https
ADD ./etc/ /etc
COPY ./index.html /var/www/html/index.html
```

### Bảng khác biệt quan trọng

| Instruction | Container ứng dụng thường | Bootc (Image Mode) |
|---|---|---|
| Chạy service | `ENTRYPOINT`/`CMD` | `systemctl enable` (dùng chính systemd quản lý) |
| Mở port | `EXPOSE` (chỉ để tài liệu hóa) | Cấu hình `firewall-cmd` thật sự |
| Biến môi trường | `ENV` | Cấu hình qua systemd service |
| User chạy | `USER` | Runtime user management |

> ⚠️ **Quan trọng:** `ENTRYPOINT`, `CMD`, `ENV`, `USER`, `EXPOSE` **không có tác dụng thật sự** trong image mode RHEL — hệ thống dùng systemd để quản lý mọi thứ như 1 OS bình thường.

### ADD vs COPY
| Lệnh | Khác biệt |
|---|---|
| `ADD` | Có thể lấy file qua mạng, **tự giải nén archive** |
| `COPY` | Chỉ copy file/thư mục local, **không giải nén** archive |

> ⚠️ RHEL image mode + `rhel-bootc` base image chịu ràng buộc **EULA** — không được **phân phối công khai** image gốc hoặc image tùy biến từ nó.

### Build & Push Image
```bash
podman build --squash -t registry.example.com/user/webserver-bootc:latest .
podman push registry.example.com/user/webserver-bootc:latest
```
- `--squash`: gộp tất cả layer mới thành **1 layer duy nhất** → giảm độ phức tạp.

### Test image ở chế độ Application Mode (chạy như container thường)
```bash
podman run -d -p 8080:80 registry.example.com/user/webserver
curl http://localhost:8080
podman exec -l ps -ef      # xem process đang chạy trong container
```
> 💡 Khi chạy bootc image ở **application mode** (qua Podman), **kernel RHEL bên trong không khởi động** — process chạy trong context của platform container (Podman), nhưng **PID 1 vẫn là systemd** (init process), và nó tự khởi động các service (vd `httpd`) y hệt hệ thống thật.

### 📝 Bài thực hành: Tạo Bootable Container Image
1. `podman login registry.lab.example.com:5000`
2. `podman search registry.lab.example.com:5000/` → xem image có sẵn (bao gồm `rhel-bootc`)
3. Sửa Containerfile:
   ```dockerfile
   FROM registry.lab.example.com:5000/rhel10/rhel-bootc
   ADD etc /etc
   RUN dnf install -y httpd && systemctl enable httpd
   COPY index.html /var/www/html/
   ```
4. Build (dùng `--isolation chroot` để tránh lỗi quyền):
   ```bash
   podman build --squash --isolation chroot \
   -t registry.lab.example.com:5000/student/webserver-bootc .
   ```
5. Test local: `podman run -p 8080:80 -d registry.../webserver-bootc` → `curl localhost:8080`
6. Push: `podman push registry.lab.example.com:5000/student/webserver-bootc`

---

## 3. Cài đặt RHEL bằng Image Mode (qua Kickstart)

### Khác biệt lớn nhất so với Kickstart truyền thống

| Package mode Kickstart | Image mode Kickstart |
|---|---|
| Có section `%packages` | **KHÔNG** có `%packages` — thay bằng lệnh `ostreecontainer` |
| Cài từng gói lẻ | Deploy **nguyên 1 image** đã build sẵn |

```
# Thay vì:
%packages
@^minimal-environment
vim-enhanced
%end

# Dùng:
ostreecontainer --url=registry.lab.example.com:5000/student/webserver-bootc
```

> ⚠️ Image tham chiếu trong `ostreecontainer` **phải** được build dựa trên `rhel-bootc` base image chính thức.
> ⚠️ Lệnh `rpm-ostree` để thay đổi hệ thống trực tiếp **không được hỗ trợ**.

### Vẫn giữ nguyên các lệnh Kickstart chuẩn khác
```
rootpw --iscrypted $6$...
user --name=student --groups=wheel --iscrypted --password=$6$...
```
→ Vì image `rhel-bootc` là **generic**, không có user mặc định — vẫn cần Kickstart để tạo user/root password như bình thường.

### Quy trình boot giống Kickstart truyền thống
```
inst.ks=http://server/ks.cfg
```

### 📝 Bài thực hành: Cài đặt RHEL bằng Image Mode
**Mục tiêu:** Dùng Kickstart để cài image đã build ở bài trước lên `serverc`.

1. Sửa Kickstart file: xóa `%packages`, thêm:
   ```
   ostreecontainer --url=registry.lab.example.com:5000/student/webserver-bootc
   ```
2. Giữ nguyên phần `%pre` (đăng nhập registry cho Anaconda, cấu hình registry self-signed cert)
3. Publish file lên `/var/www/html/ks.cfg`
4. Trên `serverc`: PXE boot → `inst.ks=http://servera.lab.example.com/ks.cfg`
5. Sau khi cài xong, login và khám phá hệ thống:
   - `ls -l /` → thấy `/home`, `/root`, `/srv`, `/mnt` đều là **symlink trỏ sang `/var/...`**
   - Tạo file ở `/var/tmp` → **thành công** (mutable)
   - Sửa `/etc/motd` → **thành công** (mutable)
   - `sudo touch /test.root` → **THẤT BẠI**: `Read-only file system` (root FS immutable)
   - `sudo touch /usr/test.usr` → **THẤT BẠI** tương tự (`/usr` cũng immutable)
   - `bootc status` → xem image đang chạy
   - `curl http://serverc.lab.example.com` → xác nhận web server hoạt động, hiển thị nội dung từ image

---

## 4. Tạo Disk Image cho Cloud/Hybrid (bootc-image-builder)

### Use case
Deploy bootc image lên **VM/cloud** thay vì bare-metal → cần convert sang **định dạng disk image** phù hợp.

### Các định dạng hỗ trợ

| Format | Môi trường đích |
|---|---|
| `qcow2` (mặc định) | QEMU/KVM |
| `vmdk` | VMware vSphere |
| `vhd` | Microsoft Virtual PC |
| `ami` | Amazon Machine Image |
| `gcd` | Google Compute Engine |
| `raw` | Raw disk không định dạng |

> 💡 `bootc-image-builder` chỉ tồn tại dưới dạng **container** (không phải RPM) — phải chạy qua Podman.

### Quy trình tạo QCOW2 disk image

1. **Đăng nhập & tải tool:**
   ```bash
   sudo podman login registry.redhat.io
   sudo podman pull registry.redhat.io/rhel10/bootc-image-builder
   ```

2. **Tạo file cấu hình user** (`config.toml`) — vì `rhel-bootc` không có user mặc định:
   ```toml
   [[customizations.user]]
   name = "user1"
   password = "password1"
   groups = ["wheel"]
   ```

3. **Tạo thư mục output:**
   ```bash
   mkdir output
   ```

4. **Chạy build:**
   ```bash
   sudo podman run --rm -it --privileged \
     --security-opt label=type:unconfined_t \
     -v ./config.toml:/config.toml \
     -v ./output:/output \
     registry.redhat.io/rhel10/bootc-image-builder \
     --type qcow2 \
     registry.lab.example.com/user/bootc-httpd:latest
   ```
   > ⚠️ Container này phải chạy với quyền **root** (`--privileged`).

5. **Deploy VM trên KVM:**
   ```bash
   sudo cp output/qcow2/disk.qcow2 /var/lib/libvirt/images/
   sudo virt-install --name bootc-webserver --memory 4096 \
     --import --disk /var/lib/libvirt/images/disk.qcow2 \
     --graphics none --osinfo rhel10.0 --noautoconsole --noreboot
   sudo virsh start bootc-webserver
   ```

---

## 5. Quản lý Hệ thống Image Mode (Day 2 Operations)

### Khái niệm "Day 2 Operations"
= Toàn bộ công việc quản lý **sau khi** đã deploy hệ thống — chiếm phần lớn công sức trong vòng đời hệ thống.

### Kiểm tra trạng thái hiện tại: `bootc status`
```bash
bootc status
```
Hiển thị:
- **staged**: image đã fetch, chờ áp dụng ở lần reboot tiếp theo (null nếu chưa có)
- **booted**: image đang chạy hiện tại
- **rollback**: image có thể rollback về (null nếu chưa từng update)

### Quy trình nâng cấp hệ thống (Upgrade Workflow)
```
1. Sửa Containerfile
2. Build lại image (podman build)
3. Push image mới lên registry (CÙNG tag)
4. Cập nhật trên hệ thống (bootc upgrade)
5. Reboot để áp dụng
```

### Tự động cập nhật (mặc định BẬT)
```bash
systemctl status bootc-fetch-apply-updates.timer
```
- Timer này **tự động kiểm tra** registry định kỳ và fetch update.

### Tắt tự động cập nhật (khi muốn quản lý thủ công, vd qua Ansible)
```bash
systemctl mask bootc-fetch-apply-updates.timer
```

### Cập nhật thủ công
```bash
bootc upgrade              # fetch update, staged cho lần boot tiếp theo (KHÔNG tự reboot)
bootc upgrade --apply      # fetch + tự động reboot luôn
bootc upgrade --check      # chỉ xem có update không, không áp dụng
```
> 💡 `bootc upgrade` và `bootc update` là **alias** của nhau.
> ⚠️ Update luôn cần **reboot** để áp dụng — hệ thống đang chạy **không bị ảnh hưởng** cho tới khi reboot.

### Rollback về phiên bản trước
```bash
bootc rollback
```
- Đưa deployment ở mục `rollback` lên hàng chờ boot tiếp theo.
- Phiên bản hiện tại trở thành **rollback mới**.
- GRUB2 menu tự cập nhật, entry staged (nếu có) sẽ bị **hủy bỏ**.

---

## 6. Cấu trúc File System trong Image Mode

### Root file system: dùng **composefs** (mặc định)
- Overlay file system cho phép root FS **thực sự read-only**.
- Dữ liệu lấy từ **OSTree repository** (`/ostree/repo`) — cho phép lưu **nhiều phiên bản file system song song** (giống version control, nhưng cho cả filesystem thay vì từng file).

```bash
df -h
# composefs       8.4M  8.4M     0 100% /     ← root FS, read-only
# /dev/sda3       9.0G  1.9G  7.2G  21% /sysroot
```

### `/etc` — Mutable, có 3-way merge khi upgrade
- Mỗi deployment có **bản copy `/etc` riêng**.
- Thay đổi bạn tự làm ở `/etc` **được giữ lại** qua các lần upgrade.
- File **không bị sửa** bởi bạn thì **tự cập nhật** theo phiên bản mới từ image.
- Việc merge này do `ostree-finalize-staged.service` thực hiện lúc **shutdown**.

### `/var` — Mutable, KHÔNG merge, KHÔNG rollback
- Dùng chung giữa các deployment.
- Chỉ **copy 1 lần** từ image lúc cài đặt ban đầu — **các lần upgrade sau không đụng vào** dù Containerfile có chỉ định thay đổi.
- Rollback **không** ảnh hưởng nội dung `/var` — dữ liệu người dùng (web content, home directory...) **luôn được giữ nguyên** qua mọi thao tác upgrade/rollback.

> ⚠️ **Không thể** đẩy thay đổi vào `/var` thông qua image upgrade hay rollback — `/var` là "local machine state" thuần túy.

### Boot & chọn version
- Dùng **GRUB2** (trừ kiến trúc s390x).
- Mỗi OSTree deployment có **1 menu entry riêng** → có thể chọn boot vào bản cũ nếu bản mới gặp lỗi.

---

## 7. 📝 Bài thực hành: Quản lý Hệ thống Image Mode

**Mục tiêu:** Cập nhật server đang chạy (thêm `mod_ssl`, `vim-enhanced`, đổi nội dung web), rồi rollback lại để hoàn tác.

1. Xác nhận web server đang chạy: `curl http://serverc.lab.example.com/index.html`
2. **Sửa `/var` trực tiếp trên server** (được phép vì mutable): thêm dòng vào `/var/www/html/index.html` → thấy thay đổi ngay qua `curl`
3. **Thử cài package trực tiếp bằng `dnf`** → **THẤT BẠI**:
   ```
   This bootc system is configured to be read-only.
   Pass --transient to perform this transaction in a transient overlay...
   ```
   → Xác nhận `df -h` thấy `/` là `composefs`, 100% used, read-only.
4. **Sửa Containerfile** (trên workstation), thêm:
   ```dockerfile
   RUN dnf -y install mod_ssl vim-enhanced && dnf clean all
   RUN <<EOF
       echo "Hello from a modified image-mode based installation!" > /var/www/html/index.html
   EOF
   ```
5. Build lại: `podman build --squash -t registry.../webserver-bootc ...`
6. Push: `podman push registry.../webserver-bootc`
7. Trên `serverc`: đăng nhập registry credentials cho bootc:
   ```bash
   podman login registry.lab.example.com:5000 --authfile=/etc/ostree/auth.json
   ```
8. `bootc status` → xác nhận chưa có gì staged
9. `bootc upgrade` → fetch layer mới, **staged** cho lần boot tới
10. `bootc status` → thấy mục **Staged image** xuất hiện (booted image vẫn là bản cũ)
11. `reboot`
12. Sau reboot: `bootc status` → booted image = bản mới, **rollback** = bản cũ
13. Kiểm tra: `curl --insecure https://serverc.lab.example.com` → **vẫn thấy nội dung CŨ** (vì `/var` không bị ghi đè qua upgrade, dù Containerfile có chỉ định thay đổi index.html!)
14. `rpm -q mod_ssl vim-enhanced` → xác nhận **đã cài thành công** (đây là thay đổi ở `/usr`, không phải `/var`)
15. **Rollback:** `bootc rollback` → `reboot`
16. Sau rollback: `curl --insecure https://serverc.lab.example.com` → **lỗi kết nối port 443** (mod_ssl không còn, HTTPS không khả dụng nữa)
17. `rpm -q mod_ssl vim-enhanced` → xác nhận **không còn cài đặt** → rollback thành công hoàn toàn

---

## 8. Bảng so sánh nhanh: Khi nào dùng lệnh nào?

| Việc cần làm | Lệnh |
|---|---|
| Build bootc image | `podman build --squash -t TAG .` |
| Push image lên registry | `podman push TAG` |
| Test image như container thường | `podman run -d -p 8080:80 TAG` |
| Cài RHEL image mode qua Kickstart | `ostreecontainer --url=...` (thay `%packages`) |
| Xem trạng thái hệ thống hiện tại | `bootc status` |
| Cập nhật thủ công (không tự reboot) | `bootc upgrade` |
| Cập nhật + tự reboot | `bootc upgrade --apply` |
| Chỉ kiểm tra có update không | `bootc upgrade --check` |
| Hoàn tác về bản trước | `bootc rollback` |
| Tắt tự động cập nhật | `systemctl mask bootc-fetch-apply-updates.timer` |
| Tạo disk image cho cloud/VM | `bootc-image-builder` (container) |

## 9. Các quy tắc "vàng" cần nhớ

1. ✅ Root FS trong image mode là **immutable** — chỉ `/etc` và `/var` mutable.
2. ⚠️ `ENTRYPOINT`, `CMD`, `EXPOSE`, `ENV`, `USER` trong Containerfile **không có tác dụng** với bootc — dùng `systemctl enable` + `firewall-cmd` thay thế.
3. ⚠️ Image mode Kickstart: **bỏ `%packages`**, thay bằng `ostreecontainer --url=...`.
4. ✅ `/etc` được **3-way merge** khi upgrade (giữ thay đổi của bạn + cập nhật file chưa sửa); `/var` thì **không merge, không rollback**.
5. ⚠️ Không thể `dnf install` trực tiếp trên hệ thống đang chạy (root FS read-only) — phải sửa Containerfile, build lại, rồi `bootc upgrade`.
6. ✅ Update qua `bootc upgrade` luôn **cần reboot** để áp dụng — hệ thống hiện tại không bị ảnh hưởng cho tới lúc đó.
7. ✅ `bootc rollback` đảo ngược **hoàn toàn** thay đổi ở `/usr` (như package đã cài) nhưng **không đụng** tới `/var` (dữ liệu vẫn nguyên).
8. ✅ RHEL trong image mode + `rhel-bootc` base image chịu ràng buộc **EULA** — không được phân phối công khai.
9. ✅ `bootc-image-builder` chỉ chạy dưới dạng **container** (không có bản RPM).