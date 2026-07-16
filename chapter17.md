# Tổng Hợp: Quản Lý Container với Podman trên RHEL 10

> Tài liệu tổng hợp Chương 17 – Managing Containers with Podman. Gồm 3 phần: **Tổng quan Container**, **Chạy Container với Podman**, và **Tạo & Quản lý Container Image**.

---

## 1. Tổng quan Container

### Container là gì?
- Container = **process được đóng gói** cùng toàn bộ dependency cần thiết để chạy, **tách biệt** khỏi thư viện của host OS.
- Container engine tạo **union file system** bằng cách gộp các **layer** của image — mỗi layer là 1 tập hợp thay đổi (library, binary, config...).
- Layer của image là **immutable**; khi chạy, engine thêm **1 layer ghi được** (writable layer) — layer này **bị xóa khi xóa container** (container mặc định là **ephemeral** — tạm thời).

### Công nghệ nền tảng Linux
- **Namespaces**: cô lập process khỏi nhau và khỏi host.
- **Cgroups**: quản lý tài nguyên (CPU, memory).
- **SELinux + seccomp**: ràng buộc bảo mật.
- Nguồn gốc từ `chroot`, phát triển thành chuẩn **OCI (Open Container Initiative)**.

### Lợi ích & Thách thức

| Lợi ích | Thách thức |
|---|---|
| Khởi động trong vài giây | Data persistence cần cấu hình thêm (volume) |
| Tiết kiệm tài nguyên hơn VM | Networking phức tạp hơn khi nhiều container/host |
| Portable — chạy nhất quán mọi môi trường | Bảo mật cần quản lý cẩn thận (share kernel với host) |
| Cô lập, dễ dự đoán | Rootless container an toàn hơn nhưng hạn chế 1 số quyền hệ thống |

### Image vs Instance
- **Image** = dữ liệu/instruction/library **immutable**, định nghĩa ứng dụng.
- **Instance (container đang chạy)** = phiên bản **thực thi được** của image, có network/disk/runtime cụ thể.
- 1 image → tạo được **nhiều instance độc lập**, kể cả trên nhiều host khác nhau.
- 💡 Tương tự khái niệm **class vs object** trong OOP.

### Container Registry
File cấu hình: `/etc/containers/registries.conf`

| Registry Red Hat | Đặc điểm |
|---|---|
| `registry.redhat.io` | Cần xác thực |
| `registry.access.redhat.com` | **Không cần** xác thực |
| `registry.connect.redhat.com` | Image từ Red Hat Partner Connect |

### So sánh Container vs Virtual Machine

| | VM | Container |
|---|---|---|
| Quản lý bởi | **Hypervisor** (KVM, Xen, VMware, Hyper-V) | **Container engine** (Podman) |
| Mức độ ảo hóa | Toàn bộ môi trường | Chỉ phần liên quan |
| Kích thước | Đo bằng **GB** | Đo bằng **MB** |
| Portable | Thường chỉ chạy trên cùng hypervisor | Bất kỳ engine nào tuân thủ **OCI** |
| Dùng khi nào | Cần OS/kernel khác host, cần phần cứng riêng | Cần triển khai nhanh, nhẹ, scale linh hoạt |

---

## 2. Podman — Công cụ Quản lý Container

### Đặc điểm nổi bật
- **Daemonless** (không cần daemon nền) — khác các tool container khác cần daemon (tạo single point of failure + rủi ro bảo mật do cần quyền cao).
- Cài sẵn mặc định trên RHEL 10.
```bash
podman -v    # kiểm tra version
```
- Có 3 cách tương tác: **CLI**, **RESTful API**, **Podman Desktop** (GUI).

### Đăng nhập Registry
```bash
podman login registry.redhat.io
```
> 💡 Với registry không cần xác thực (vd `registry.access.redhat.com`), chỉ cần nhấn Enter bỏ trống username/password.

### Kiểm tra cấu hình registry hiện tại
```bash
podman info    # xem OS, hardware, registry đã cấu hình, plugin...
```
- File cấu hình: `/etc/containers/registries.conf` (hoặc drop-in `/etc/containers/registries.conf.d`)
- Credential lưu ở: `${XDG_RUNTIME_DIR}/containers/auth.json` (mã hóa base64)

---

## 3. Chạy & Quản lý Container

### Chạy container
```bash
podman run IMAGE COMMAND
```
Ví dụ:
```bash
podman run registry.redhat.io/ubi10 echo 'Hello World!'
```
→ Container tự **dừng** khi command chạy xong (không còn tiến trình nào giữ nó sống).

### Chạy nền (detached mode)
```bash
podman run -d registry.redhat.io/ubi10 sleep infinity
```
- `-d` / `--detach`: chạy nền, không chiếm terminal.

### Liệt kê container
```bash
podman ps           # chỉ container ĐANG chạy
podman ps -a         # TẤT CẢ (kể cả đã dừng)
```
> 💡 Container ID có 2 dạng: **short UUID** (12 ký tự) hoặc **long UUID** (64 ký tự).
> ⚠️ Container **dừng** không đồng nghĩa với bị **xóa** — vẫn tồn tại trong storage cho tới khi `podman rm`.

### Đặt tên container
```bash
podman run --name my_python registry.redhat.io/ubi9/python-312 which python
```
> 💡 Không đặt tên → Podman tự sinh tên ngẫu nhiên — nên đặt tên rõ ràng để dễ quản lý.

### Restart / Stop / Kill / Remove

| Lệnh | Hành vi |
|---|---|
| `podman restart NAME` | Khởi động lại (hoặc start container đã dừng) |
| `podman stop NAME` | Dừng **êm** — gửi `SIGTERM`, đợi cleanup |
| `podman stop --all` | Dừng tất cả container đang chạy |
| `podman kill NAME` | Gửi `SIGKILL` ngay — dừng **cưỡng chế** |
| `podman rm ID` | Xóa container **đã dừng** |
| `podman rm ID --force` | Xóa cả container **đang chạy** (cưỡng chế) |

> 💡 Nếu container không phản hồi `SIGTERM` sau **10 giây** (mặc định khi `stop`), Podman tự gửi `SIGKILL`.

### Tự động xóa khi thoát
```bash
podman run --rm registry.redhat.io/ubi10 echo 'Hello World!'
```
→ Container biến mất ngay sau khi chạy xong, không cần `podman rm` thủ công.

### Expose port ra ngoài (cho web server, database...)
```bash
podman run -p 8080:8080 registry.redhat.io/rhel10/httpd-24:latest
```
- `-p LOCAL_PORT:CONTAINER_PORT` — map port local vào port trong container.
- Kết hợp `-d` để chạy nền không chặn terminal:
  ```bash
  podman run -d -p 8080:8080 registry.redhat.io/rhel10/httpd-24:latest
  ```
```bash
curl 127.0.0.1:8080    # test truy cập
```

---

## 4. 📝 Bài thực hành: Chạy Container với Podman

**Mục tiêu:** Chạy container in ra text, xóa nó; sau đó chạy web server, test, dừng, xóa.

1. `podman login registry.lab.example.com:5000` (user `student` / pass `redhat`)
2. Chạy container in text:
   ```bash
   podman run registry.lab.example.com:5000/ubi10/ubi echo 'Hello Red Hat!'
   ```
3. `podman ps` → không thấy gì (đã dừng); `podman ps -a` → thấy container **Exited**
4. `podman rm CONTAINER_ID` → xóa; xác nhận lại `podman ps -a` → rỗng
5. Chạy web server (foreground trước để xem log):
   ```bash
   podman run --name my_webserver -p 8080:8080 registry.lab.example.com:5000/rhel10/httpd-24
   ```
6. Test qua trình duyệt `http://localhost:8080` → thấy trang mặc định
7. `Ctrl+C` để dừng → `podman rm my_webserver`
8. Chạy lại ở chế độ **detached**:
   ```bash
   podman run -d --name my_webserver -p 8080:8080 registry.lab.example.com:5000/rhel10/httpd-24
   ```
9. `podman ps` → xác nhận đang chạy nền; `curl 127.0.0.1:8080` → xác nhận phản hồi
10. `podman stop my_webserver` → `podman rm my_webserver`
11. `podman ps -a` → xác nhận sạch, không còn container nào

---

## 5. Quản lý Container Image

### Các thao tác Podman hỗ trợ trên image
Pull, list, inspect, copy, tag, save/load, push (share qua registry).

### Tìm kiếm image
```bash
podman search registry.redhat.io/rhel10/     # LƯU Ý: cần dấu / cuối URL
podman search registry.redhat.io/ubi          # tìm Universal Base Image (UBI)
```
> 💡 **UBI (Universal Base Image)**: image nền chuẩn OCI của Red Hat, **miễn phí phân phối lại** — nền tảng tốt để build image ứng dụng riêng.

### Pull image
```bash
podman pull registry.redhat.io/rhel10/rhel-bootc          # mặc định lấy tag "latest"
podman pull registry.redhat.io/rhel10/rhel-bootc:10.0     # chỉ định version cụ thể
```

### Liệt kê image đã lưu local
```bash
podman images
```

### Inspect image (xem chi tiết dạng JSON)
```bash
podman image inspect rhel-bootc:latest
podman image inspect rhel-bootc:latest --format "{{.Config.Labels.name}}"      # lọc field cụ thể
podman image inspect rhel-bootc:latest --format "{{.Created}}"
```
> 💡 Muốn inspect image **remote**, phải `pull` về local trước.

### Gắn tag mới cho image
```bash
podman tag c6222576494f fedora:latest
```

---

## 6. Tạo Image từ Containerfile

### Ví dụ Containerfile
```dockerfile
FROM ubi10/ubi
RUN dnf install -y httpd
COPY index.html /var/www/html/index.html
EXPOSE 80
ENTRYPOINT ["/usr/sbin/httpd", "-DFOREGROUND"]
```

### Build image
```bash
podman build -t my-httpd -f Containerfile
```
- `-t`: đặt tên + tag cho image.

### Chạy container từ image vừa build
```bash
podman run my-httpd
```

### Push image lên registry
```bash
podman image push my-httpd registry.redhat.io/my-httpd
```

### Xóa image

| Lệnh | Ý nghĩa |
|---|---|
| `podman rmi IMAGE_ID` | Xóa 1 image (và các parent image không còn tag/tham chiếu) |
| `podman image prune` | Xóa **dangling images** (không tag, không ai tham chiếu) |
| `podman image prune --all` | Xóa cả image **không được container nào dùng** |

---

## 7. 📝 Bài thực hành: Tạo & Quản lý Container Image

**Mục tiêu:** Build 2 phiên bản image (`1.0` dựa trên ubi8, `1.1` dựa trên ubi10), push lên registry, so sánh, rồi dọn dẹp.

1. `podman login registry.lab.example.com:5000`
2. Tìm image: `podman search registry.lab.example.com:5000/ubi`
3. Tạo `~/my_image/Containerfile`:
   ```dockerfile
   FROM registry.lab.example.com:5000/ubi8/ubi
   CMD echo "This container uses the ubi8/ubi image"
   ```
4. Build: `podman build -t my_image:1.0 /home/student/my_image/.`
5. Kiểm tra: `podman images` → thấy `my_image:1.0`
6. Chạy thử: `podman run my_image:1.0` → in đúng message
7. Xem container đã Exited: `podman ps -a --format "table {{.ID}}\t{{.Image}}\t{{.Status}}"`
8. Xóa container: `podman rm CONTAINER_ID`
9. Push lên registry:
   ```bash
   podman image push localhost/my_image:1.0 registry.lab.example.com:5000/my_image:1.0
   ```
10. Xác nhận: `podman search registry.lab.example.com:5000/my_image`
11. **Sửa Containerfile** để tạo version mới:
    ```dockerfile
    FROM registry.lab.example.com:5000/ubi10/ubi
    CMD echo "This container uses the ubi10/ubi image"
    ```
12. Build tag mới: `podman build -t my_image:1.1 /home/student/my_image/.`
13. `podman images` → xác nhận **2 phiên bản** (1.0 và 1.1) cùng tồn tại local, khác `IMAGE ID`
14. Chạy thử `podman run my_image:1.1` → in message khác
15. Xóa container, push tag `1.1` lên registry
16. Xem tất cả tag trên registry:
    ```bash
    podman search --list-tags registry.lab.example.com:5000/my_image
    ```
    → thấy cả `1.0` và `1.1`
17. So sánh 2 image bằng inspect:
    ```bash
    podman image inspect my_image:1.0    # xem field "Cmd"
    podman image inspect my_image:1.1
    ```
    → xác nhận `Cmd` khác nhau đúng như đã cấu hình
18. Dọn dẹp: `podman rmi my_image:1.0 my_image:1.1`

---

## 8. Bảng tóm tắt lệnh nhanh

| Việc cần làm | Lệnh |
|---|---|
| Đăng nhập registry | `podman login REGISTRY` |
| Xem cấu hình registry | `podman info` |
| Tìm image | `podman search REGISTRY/` |
| Tải image | `podman pull IMAGE:TAG` |
| Liệt kê image local | `podman images` |
| Xem chi tiết image | `podman image inspect IMAGE` |
| Gắn tag mới | `podman tag ID NEWNAME:TAG` |
| Build image từ Containerfile | `podman build -t NAME:TAG -f Containerfile .` |
| Push image lên registry | `podman image push LOCAL REMOTE` |
| Xóa image | `podman rmi ID` |
| Dọn image không dùng | `podman image prune [--all]` |
| Chạy container | `podman run IMAGE [COMMAND]` |
| Chạy nền | `podman run -d IMAGE` |
| Đặt tên | `podman run --name NAME IMAGE` |
| Map port | `podman run -p HOST:CONTAINER IMAGE` |
| Tự xóa sau khi thoát | `podman run --rm IMAGE` |
| Xem container đang chạy | `podman ps` |
| Xem tất cả (kể cả đã dừng) | `podman ps -a` |
| Dừng êm | `podman stop NAME` |
| Dừng cưỡng chế | `podman kill NAME` |
| Restart | `podman restart NAME` |
| Xóa container | `podman rm ID` / `podman rm ID --force` |

## 9. Các quy tắc "vàng" cần nhớ

1. ✅ Container mặc định **ephemeral** — writable layer mất khi container bị xóa; dữ liệu cần giữ lại phải dùng **volume**.
2. ✅ Container dừng ≠ container bị xóa — vẫn cần `podman rm` để dọn dẹp thật sự.
3. ✅ Podman **daemonless** — không có single point of failure như các tool cần daemon.
4. ⚠️ `podman search` cần dấu `/` ở cuối URL registry để hoạt động đúng.
5. ✅ Không chỉ định tag → Podman mặc định dùng `latest`.
6. ✅ Muốn inspect image **remote** → phải `pull` về local trước.
7. ✅ `podman stop` gửi `SIGTERM` (êm, có thời gian cleanup); `podman kill` gửi `SIGKILL` ngay (cưỡng chế).
8. ✅ Dùng `--rm` khi chạy container tạm thời để tự dọn dẹp, tránh phải nhớ `podman rm` sau.
9. ✅ UBI (Universal Base Image) là nền tảng miễn phí, tuân thủ OCI — điểm khởi đầu tốt để build image riêng.