# RH134 — Red Hat System Administration II (RHEL 10)

> Tài liệu tổng hợp toàn bộ nội dung khóa học **RH134** trên nền **Red Hat Enterprise Linux 10**.  
> Mỗi chương bao gồm: khái niệm cốt lõi, bảng tóm tắt lệnh, ví dụ thực tế, bài thực hành và các quy tắc "vàng" cần nhớ.

---

## 📚 Mục lục

| Chương | Tiêu đề | File |
|--------|---------|------|
| 1 | Shell Scripting & Command Line | [chapter1.md](chapter1.md) |
| 2 | Regular Expressions (Regex) | [chapter2.md](chapter2.md) |
| 3 | Lập Lịch Tác Vụ Người Dùng (`at`, `crontab`) | [chapter3.md](chapter3.md) |
| 4 | Lập Lịch Tác Vụ Hệ Thống (Systemd Timer, tmpfiles, Cron) | [chapter4.md](chapter4.md) |
| 5 | Phân Tích và Lưu Trữ Log (Rsyslog, Journal, NTP) | [chapter5.md](chapter5.md) |
| 6 | Quản Lý Bảo Mật với SELinux | [chapter6.md](chapter6.md) |
| 7 | Archive File (tar, gzip, bzip2, xz) | [chapter7.md](chapter7.md) |
| 8 | Truyền File Giữa Các Hệ Thống (sftp, scp, rsync) | [chapter8.md](chapter8.md) |
| 9 | Tối Ưu Hiệu Năng Hệ Thống (TuneD, nice/renice) | [chapter9.md](chapter9.md) |
| 10 | Quản Lý Lưu Trữ Cơ Bản (parted, mkfs, mount, swap) | [chapter10.md](chapter10.md) |
| 11 | Quản Lý Lưu Trữ bằng LVM | [chapter11.md](chapter11.md) |
| 12 | Kiểm Soát & Khắc Phục Sự Cố Quá Trình Boot | [chapter12.md](chapter12.md) |
| 13 | Khôi Phục Quyền Superuser & Reset Mật Khẩu Root | [chapter13.md](chapter13.md) |
| 14 | Quản Lý Bảo Mật Mạng (Firewalld, SELinux Port Labeling) | [chapter14.md](chapter14.md) |
| 15 | Truy Cập Lưu Trữ Mạng (NFS, Autofs) | [chapter15.md](chapter15.md) |
| 16 | Cài Đặt Red Hat Enterprise Linux 10 (Anaconda, Kickstart) | [chapter16.md](chapter16.md) |
| 17 | Quản Lý Container với Podman | [chapter17.md](chapter17.md) |
| 18 | Làm Việc với Image Mode cho RHEL 10 | [chapter18.md](chapter18.md) |
| 19 | Lab Tổng Ôn (Comprehensive Review) | [chapter19.md](chapter19.md) |

---

## 🗂️ Nội dung tóm tắt từng chương

### Chương 1 — Shell Scripting & Command Line
Biến shell và môi trường (`export`, `PATH`, `PS1`); file startup (`~/.bashrc`, `~/.bash_profile`); viết Bash script cơ bản (shebang, positional parameters, quoting); vòng lặp `for`, `while`, `until`; điều kiện `if/elif/else`; toán tử kiểm tra số, chuỗi, file; exit code và `$?`.

### Chương 2 — Regular Expressions (Regex)
Pattern matching với regex; anchor (`^`, `$`); wildcard (`.`) và multiplier (`*`, `+`, `?`, `{n,m}`); BRE vs ERE; character class POSIX (`[:alnum:]`, `[:digit:]`...); word boundary (`\b`, `\w`); dùng `grep` với các option `-i`, `-v`, `-r`, `-e`, `-E`, `-A`, `-B`; tìm kiếm trong `less`/`vim`.

### Chương 3 — Lập Lịch Tác Vụ Người Dùng
**`at`**: tạo job chạy 1 lần (`atq`, `atrm`, `at -c`); cú pháp thời gian linh hoạt (`now +5min`, `teatime`, `noon +4 days`).  
**`crontab`**: cấu trúc 5 trường thời gian, range, bước nhảy `*/x`; lệnh `crontab -l/-e/-r`; biến môi trường `SHELL`, `MAILTO`.

### Chương 4 — Lập Lịch Tác Vụ Hệ Thống
**Systemd Timer**: `OnCalendar`, `OnUnitActiveSec`; cách sửa đúng (copy sang `/etc/systemd/system/`, `daemon-reload`, `enable --now`).  
**systemd-tmpfiles**: các type `d`, `D`, `q`, `Z`, `L`; `--create`, `--clean`; file trong `/etc/tmpfiles.d/`.  
**Cron hệ thống**: `/etc/cron.d/` (có thêm cột user); `/etc/cron.hourly/daily/weekly/monthly/`; **Anacron** cho máy có thể tắt.

### Chương 5 — Phân Tích và Lưu Trữ Log
Luồng log: `journald` → `rsyslog` → `/var/log/`; facility & priority trong rsyslog; `logrotate`; **journalctl** với `-n`, `-f`, `-p`, `-u`, `--since`, `--until`, `-o verbose`, field filter; cấu hình journal bền vững (`mkdir /var/log/journal`); `journalctl --list-boots`, `-b -1`; đồng bộ NTP với `chronyd` (`timedatectl`, `chronyc sources`).

### Chương 6 — Quản Lý Bảo Mật với SELinux
DAC vs MAC; context (`httpd_t`, `httpd_sys_content_t`, `http_port_t`); mode Enforcing/Permissive/Disabled; `getenforce`/`setenforce`; `/etc/selinux/config`.  
File context: `cp` vs `mv`; `chcon` (tạm), `semanage fcontext -a` + `restorecon -Rv` (bền vững); cú pháp `(/.*)?`.  
Booleans: `getsebool`, `setsebool -P`.  
Troubleshooting: `sealert -l UUID`, `sealert -a /var/log/audit/audit.log`, `ausearch -m AVC`.

### Chương 7 — Archive File
`tar` với `-c` (tạo), `-t` (liệt kê), `-x` (giải nén); `-f`, `-v`, `-p`; nén: `-z` (gzip), `-j` (bzip2), `-J` (xz); bảo toàn ACL/SELinux với `--acls`, `--selinux`, `--xattrs`; so sánh dung lượng 3 thuật toán; `gzip -l`, `xz -l`.

### Chương 8 — Truyền File Giữa Các Hệ Thống
**sftp**: phiên tương tác (`put`, `get`, `lpwd`, `lcd`); dạng 1 dòng (chỉ download).  
**scp**: copy đệ quy `-r`; lưu ý RHEL 10 dùng giao thức SFTP bên dưới, tránh `-O`.  
**rsync**: `rsync -av`; `-n` (dry-run); `-a` (archive mode = `-rlptgo -D`); `-H` (hard link), `-A` (ACL), `-X` (SELinux); lưu ý dấu `/` cuối đường dẫn nguồn.

### Chương 9 — Tối Ưu Hiệu Năng Hệ Thống
**TuneD**: static vs dynamic tuning; các profile (`balanced`, `throughput-performance`, `latency-performance`, `virtual-guest`...); `tuned-adm active/list/profile/recommend/verify`; cách sửa profile (copy sang `/etc/tuned/profiles/`).  
**Process scheduling**: SCHED_NORMAL (EEVDF từ RHEL 10), real-time (FIFO, RR); nice value (-20 đến +19, mặc định 0); `nice -n N`, `renice -n N PID`; xem bằng `top` (cột PR, NI) và `ps`.

### Chương 10 — Quản Lý Lưu Trữ Cơ Bản
**Phân vùng**: MBR (tối đa 2TiB, 4 primary) vs GPT (128 partition, 8ZiB); `parted` (`mklabel`, `mkpart`, `rm`); `udevadm settle`.  
**File system**: `mkfs.xfs`, `mkfs.ext4`; mount tạm (`mount`) và bền vững (`/etc/fstab`): 6 cột, dùng UUID; `lsblk --fs`; `findmnt --verify`.  
**Swap**: `mkswap`, `swapon`/`swapoff`; priority; bảng khuyến nghị kích thước swap theo RAM.

### Chương 11 — Quản Lý Lưu Trữ bằng LVM
3 lớp: PV → VG → LV; `pvcreate`, `vgcreate`, `lvcreate`; `pvdisplay/pvs`, `vgdisplay/vgs`, `lvdisplay/lvs`; mở rộng: `lvextend -L +SIZE`; `xfs_growfs MOUNT_POINT` (online, chỉ tăng), `resize2fs DEVICE` (cả 2 chiều); `lvextend -r` (kết hợp); `pvmove`, `vgreduce`; xóa theo thứ tự LV → VG → PV.

### Chương 12 — Kiểm Soát & Khắc Phục Sự Cố Quá Trình Boot
Luồng boot: Firmware → GRUB2 → Kernel+initramfs → systemd → target; **GRUB2**: menu (`E` editor, `Ctrl+X`); `grubby --info/--set-default-index/--update-kernel --args`.  
**Systemd Target**: `graphical`, `multi-user`, `rescue`, `emergency`; `systemctl isolate/get-default/set-default`; thêm `systemd.unit=TÊN.target` vào GRUB2 editor.  
**Sửa FS hỏng**: emergency shell, `mount -o remount,rw /`, `mount -a`; `xfs_repair`, `fsck.ext4 -p`; option `nofail`.

### Chương 13 — Khôi Phục Quyền Superuser
Reset mật khẩu root khi bị quên; **Cách 1 (khuyến nghị)**: rescue media ISO → `chroot /mnt/sysroot` → `passwd` → `touch /.autorelabel` → `exit` 2 lần; **Cách 2**: `init=/bin/bash` trong GRUB2 editor (xóa `console=` trước) → `mount -o remount,rw /` → `passwd` → `touch /.autorelabel` → `exec /sbin/init`; lý do bắt buộc relabel SELinux sau khi đổi password.

### Chương 14 — Quản Lý Bảo Mật Mạng
**Firewalld**: netfilter → nftables → firewalld; zone (`drop`, `block`, `dmz`, `external`, `public`, `trusted`...); thứ tự gán zone (source → interface → default); runtime vs permanent (`--permanent` + `--reload`); `firewall-cmd --add-service/--add-port/--add-source`.  
**SELinux Port Labeling**: `semanage port -l`, `-a/-d/-m -t TYPE -p tcp PORT`; kết hợp với firewall là 2 lớp độc lập.

### Chương 15 — Truy Cập Lưu Trữ Mạng (NFS)
NFSv3 vs NFSv4; `showmount --exports`; mount thủ công (`-t nfs -o rw,sync`), bền vững (`/etc/fstab`), theo nhu cầu (`autofs`).  
**Autofs**: `dnf install autofs`; master map (`/etc/auto.master`); direct map (`/-`) vs indirect map (base directory); wildcard `*` và `&`; `x-systemd.automount` làm phương án thay thế đơn giản hơn.

### Chương 16 — Cài Đặt Red Hat Enterprise Linux 10
**Cài tương tác (Anaconda GUI)**: Binary DVD vs Boot ISO; yêu cầu tối thiểu; các màn hình cấu hình; root mặc định bị disable trong RHEL 10; RDP thay VNC (`inst.rdp`); troubleshooting qua tmux (`Ctrl+Alt+F1`).  
**Kickstart**: file text tự động hóa cài đặt; cấu trúc (`%packages`, `%pre`, `%post`); các lệnh bắt buộc (`lang`, `keyboard`, `timezone`, `bootloader`); `ksvalidator`, `ksverdiff` (package `pykickstart`); `inst.ks=URL`.

### Chương 17 — Quản Lý Container với Podman
Kiến trúc container (namespaces, cgroups, OCI, union filesystem); Image vs Instance; Container Registry (`registry.redhat.io`, `registry.access.redhat.com`); Podman daemonless; **chạy container** (`podman run`, `-d`, `--name`, `-p`, `--rm`); quản lý vòng đời (`podman ps/stop/kill/rm/restart`); **quản lý image** (`podman search`, `pull`, `images`, `image inspect`, `tag`, `rmi`, `image prune`); **build image** từ Containerfile (`podman build -t`); push lên registry; UBI (Universal Base Image).

### Chương 18 — Làm Việc với Image Mode cho RHEL 10
Package mode vs Image mode; `rhel-bootc` base image; Containerfile cho bootc (`systemctl enable`, `firewall-cmd` thay `ENTRYPOINT`/`EXPOSE`); `podman build --squash`, `podman push`.  
Kickstart image mode: `ostreecontainer --url=` thay `%packages`.  
**bootc-image-builder**: tạo QCOW2/VMDK/AMI; **Day 2 Operations**: `bootc status`, `bootc upgrade [--apply|--check]`, `bootc rollback`.  
Cấu trúc FS: root immutable (composefs), `/etc` mutable (3-way merge), `/var` mutable (không rollback).

### Chương 19 — Lab Tổng Ôn (Comprehensive Review)
**Lab 1**: Sửa lỗi boot (`/etc/fstab` lỗi → emergency mode); đổi default target; lên lịch cron backup giờ cao điểm.  
**Lab 2**: SSH key-based auth; SELinux Permissive; autofs NFS home directory; SELinux Boolean `use_nfs_home_dirs`; firewalld block source IP; Apache port nonstandard + SELinux port labeling + firewall (2 lớp bảo mật).  
**Lab 3**: Build/push/run container Podman; port mapping; mở firewall cho container port.

---

## 🔑 Các công cụ chính theo chủ đề

| Chủ đề | Công cụ chính |
|--------|--------------|
| Shell & Script | `bash`, `set`, `export`, `alias`, `source` |
| Tìm kiếm văn bản | `grep`, `less`, `vim` (với regex) |
| Lập lịch tác vụ | `at`, `atq`, `atrm`, `crontab`, `systemctl` (timer) |
| Quản lý log | `journalctl`, `rsyslog`, `logger`, `logrotate`, `timedatectl`, `chronyc` |
| Bảo mật SELinux | `getenforce`, `setenforce`, `chcon`, `semanage`, `restorecon`, `getsebool`, `setsebool`, `sealert`, `ausearch` |
| Lưu trữ & nén | `tar`, `gzip`, `bzip2`, `xz` |
| Truyền file | `sftp`, `scp`, `rsync` |
| Hiệu năng | `tuned-adm`, `nice`, `renice`, `top`, `ps` |
| Quản lý disk | `parted`, `mkfs`, `mount`, `umount`, `lsblk`, `findmnt`, `mkswap`, `swapon` |
| LVM | `pvcreate`, `vgcreate`, `lvcreate`, `lvextend`, `xfs_growfs`, `resize2fs`, `pvmove` |
| Boot & Recovery | `grubby`, `systemctl` (target), `xfs_repair`, `fsck.ext4` |
| Firewall | `firewall-cmd`, `nftables` |
| NFS & Autofs | `mount`, `autofs`, `showmount` |
| Cài đặt OS | `anaconda`, `ksvalidator`, `ksverdiff` |
| Image Mode | `podman`, `bootc`, `bootc-image-builder` |

---

## ⚡ Quick Reference — Lệnh hay dùng nhất

```bash
# SELinux
semanage fcontext -a -t httpd_sys_content_t '/custom(/.*)?'
restorecon -Rv /custom
setsebool -P httpd_enable_homedirs on
sealert -a /var/log/audit/audit.log

# Firewall
firewall-cmd --permanent --add-service=https
firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --reload

# LVM
pvcreate /dev/sdb1 && vgcreate vg01 /dev/sdb1
lvcreate -n lv01 -L 300M vg01
lvextend -r -L +500M /dev/vg01/lv01   # resize LV + FS cùng lúc

# Journal
journalctl -p err --since today
journalctl -u sshd.service --since "-1 hour"

# Archive
tar -cJf backup.tar.xz /etc          # tạo archive xz
tar -xf backup.tar.xz -C /restore/   # giải nén vào thư mục khác

# Autofs
echo "/-  /etc/auto.direct" >> /etc/auto.master
echo "/mnt/nfs  -rw,sync  server:/share" > /etc/auto.direct
systemctl enable --now autofs
```

---

*Cập nhật lần cuối: 2026-07-16*