# BÁO CÁO KỸ THUẬT: PHÂN TÍCH KIẾN TRÚC CONTAINER VS VIRTUAL MACHINE

- **Môn học:** DevOps / Containerization
- **Bài thực hành:** Bài 1 - Session 10
- **Mục tiêu:** 
  - Phân tích chuyên sâu sự khác biệt kiến trúc giữa ảo hóa phần cứng (Virtual Machine / Hypervisor) và ảo hóa hệ điều hành (Container Engine).
  - Trình bày cơ chế cô lập kernel (Namespaces) và quản lý tài nguyên (Control Groups - cgroups) của Linux.
  - Trích xuất và giải thích các thông số cốt lõi từ đầu ra lệnh `docker info`.
- **Đường dẫn nộp bài:** `homework/session_10/ex1/REPORT.md`

---

## 1. Bối cảnh kỹ thuật: Lý do doanh nghiệp chuyển đổi từ VPS sang Docker

Trong mô hình kiến trúc truyền thống, các doanh nghiệp thường triển khai ứng dụng trên **VPS (Virtual Private Server)** hoặc các máy ảo (VM) do các nhà cung cấp điện toán đám mây cung cấp. Tuy nhiên, khi nhu cầu về tốc độ phát hành (Time-to-Market), quy mô người dùng và tính sẵn sàng tăng cao, mô hình VPS bộc lộ nhiều điểm nghẽn nghiêm trọng:

1. **Lãng phí tài nguyên và chi phí vận hành (TCO - Total Cost of Ownership):** Mỗi VPS đòi hỏi một bản cài đặt hệ điều hành đầy đủ (Guest OS), bao gồm kernel, thư viện hệ thống, driver và các tiến trình nền (background daemons). Việc này tiêu tốn dung lượng RAM và CPU cố định ngay cả khi ứng dụng chưa xử lý tải.
2. **Khởi động chậm và phản ứng kém trước tải đột biến:** Khởi động một VPS mất từ vài chục giây đến vài phút do phải trải qua chu kỳ nạp kernel, kiểm tra thiết bị ảo và nạp init system (`systemd`). Điều này gây cản trở cơ chế tự động co giãn (Auto-scaling).
3. **Hiện tượng "Works on my machine" giữa các môi trường:** Sự sai lệch giữa môi trường phát triển (Dev) của lập trình viên và VPS máy chủ (Staging/Production) về phiên bản thư viện, bản vá OS thường xuyên gây ra lỗi khó chẩn đoán.
4. **Mật độ triển khai (Density) thấp:** Một máy chủ vật lý chỉ có thể gánh tải một số lượng hạn chế các VPS do gánh nặng tài nguyên từ nhiều Guest OS chạy song song.

**Giải pháp chuyển dịch sang Docker (Containerization):**
Docker cung cấp giải pháp ảo hóa cấp độ hệ điều hành (OS-level virtualization). Tất cả các container chia sẻ chung nhân Linux Kernel của Host OS, loại bỏ hoàn toàn tầng Guest OS và Hypervisor trung gian. Nhờ đó, ứng dụng đóng gói dưới dạng container có thể khởi động trong mili-giây, tiêu hao cực ít tài nguyên nhàn rỗi, chuẩn hóa môi trường từ Dev đến Production và nâng cao mật độ triển khai lên gấp 3 đến 5 lần trên cùng một hạ tầng phần cứng.

---

## 2. Bảng so sánh kiến trúc: Virtual Machine (VM) vs Container

Dưới đây là bảng so sánh chi tiết giữa Virtual Machine (ảo hóa phần cứng qua Hypervisor) và Container (ảo hóa hệ điều hành qua Container Engine) dựa trên 5 tiêu chí kỹ thuật cốt lõi:

| Tiêu chí so sánh | Virtual Machine (VM / VPS) | Container (Docker) | Phân tích chuyên sâu |
| :--- | :--- | :--- | :--- |
| **1. Dung lượng Image (Image Size)** | **Rất lớn** (Thường từ vài GB đến hàng chục GB) | **Rất nhẹ** (Từ vài MB như Alpine: ~5MB đến vài trăm MB) | VM Image phải đóng gói toàn bộ Guest OS (kernel, binaries, drivers). Docker Image chỉ đóng gói mã nguồn ứng dụng, dependencies và các thư viện cần thiết (userspace). |
| **2. Thời gian khởi động (Startup Time)** | **Chậm** (Mất từ 30 giây đến vài phút) | **Cực nhanh** (Mili-giây đến vài giây) | VM phải trải qua toàn bộ quy trình boot BIOS ảo, khởi tạo phần cứng ảo và nạp kernel. Container thực chất chỉ là một tiến trình bình thường (process) được khởi chạy trên Host Kernel. |
| **3. Hao phí OS (OS Overhead)** | **Rất cao** (Mỗi VM tiêu tốn từ 1 - 2GB RAM & CPU riêng cho Guest OS) | **Gần như bằng 0** (Chỉ tiêu tốn tài nguyên đúng mức tiến trình ứng dụng cần) | Không cần cấp phát tài nguyên tĩnh dự phòng cho một hệ điều hành khách riêng biệt; chia sẻ trực tiếp kernel host và page cache. |
| **4. Mức độ cô lập (Isolation Level)** | **Rất cao (Hardware-level)** (Cô lập triệt để ở cấp phần cứng thông qua Hypervisor) | **Trung bình - Tốt (Process-level)** (Cô lập mức tiến trình thông qua Linux Namespaces & cgroups) | VM có kernel độc lập nên lỗi kernel ở 1 VM không ảnh hưởng VM khác. Container chia sẻ chung kernel host, nếu có lỗ hổng Kernel exploit, nguy cơ rủi ro bảo mật cao hơn nếu không giới hạn capability. |
| **5. Khả năng mở rộng (Scalability & Density)** | **Hạn chế** (Mật độ thấp, scale chậm, chi phí bản quyền và cấp phát tài nguyên cao) | **Cực cao** (Mật độ cao, khởi tạo hàng trăm container trong tích tắc, dễ dàng tích hợp Kubernetes) | Kiến trúc vi dịch vụ (Microservices) yêu cầu khởi tạo/hủy bỏ liên tục các instances theo tải lưu lượng, điều mà Container đáp ứng vượt trội hoàn toàn so với VM. |

---

## 3. Cơ chế hoạt động trong Linux Kernel: Namespaces & Control Groups (cgroups)

Bản chất Docker Container không phải là một "máy ảo thu nhỏ", mà là **tiến trình Linux thông thường được áp dụng hai công nghệ nền tảng của nhân Linux**:

### 3.1. Linux Namespaces (Cơ chế cô lập - Isolation)
Namespaces chia sẻ một tài nguyên hệ thống toàn cục và hiển thị nó dưới dạng một vùng tài nguyên riêng biệt cho một nhóm tiến trình, tạo ra ảo giác rằng tiến trình đang sở hữu toàn bộ hệ thống:

1. **PID Namespace (Process ID):**
   - **Vai trò:** Cô lập không gian số định danh tiến trình.
   - **Cơ chế:** Tiến trình chính bên trong container sẽ nhìn thấy mình mang **PID = 1** (tiến trình init của container) và chỉ thấy được các tiến trình con do nó sinh ra. Trong khi đó, từ góc nhìn của máy Host, tiến trình này mang một PID bình thường khác (ví dụ: PID 14285). Điều này ngăn ngừa container can thiệp hoặc gửi tín hiệu (signals như `SIGKILL`) đến các tiến trình khác trên Host.
2. **NET Namespace (Network):**
   - **Vai trò:** Cô lập hạ tầng mạng.
   - **Cơ chế:** Mỗi container sở hữu bảng định tuyến (routing table), danh sách card mạng ảo (virtual interfaces, thường là cặp `veth`), tường lửa iptables/nftables và dải cổng mạng (ports) hoàn toàn riêng biệt. Cổng `80` trong container không hề trùng lặp hay xung đột với cổng `80` của một container khác hoặc của Host.
3. **MNT Namespace (Mount Points):**
   - **Vai trò:** Cô lập các điểm gắn kết hệ thống tệp tin (filesystem).
   - **Cơ chế:** Kết hợp cùng `chroot`/`pivot_root` và cơ chế Storage Driver (như `overlay2`), MNT Namespace cấp cho container một hệ thống tệp tin gốc (`rootfs`) độc lập. Các thao tác ghi, đọc, tạo thư mục tạm trong container chỉ diễn ra trên layer ghi của container mà không làm ảnh hưởng đến cấu trúc tệp tin của Host OS.
4. **Các Namespaces khác:**
   - **IPC Namespace:** Cô lập cơ chế giao tiếp liên tiến trình (Shared Memory, POSIX message queues).
   - **UTS Namespace:** Cho phép container sở hữu Hostname và Domain name riêng biệt.
   - **USER Namespace:** Ánh xạ UID/GID bên trong container với UID/GID không có quyền (non-root) trên máy Host, tăng cường bảo mật.

### 3.2. Control Groups - cgroups (Quản lý và Giới hạn tài nguyên - Resource Limiting)
Nếu Namespaces chịu trách nhiệm trả lời câu hỏi: *"Tiến trình được nhìn thấy những gì?"*, thì cgroups trả lời câu hỏi: *"Tiến trình được sử dụng bao nhiêu tài nguyên?"*.

1. **Giới hạn bộ nhớ (RAM - Memory Limiting):**
   - Giúp kiểm soát lượng bộ nhớ RAM tối đa mà một container được phép sử dụng (thông qua cờ `--memory="512m"`).
   - Ngăn chặn tình trạng rò rỉ bộ nhớ (Memory Leak) của một ứng dụng làm cạn kiệt RAM hệ thống, dẫn đến việc Linux Kernel kích hoạt tiến trình **OOM Killer (Out-Of-Memory Killer)** tiêu diệt nhầm các tiến trình quan trọng khác.
2. **Giới hạn thời gian xử lý (CPU Limiting):**
   - Phân bổ chu kỳ tính toán của CPU thông qua CPU Quota và CPU Period (ví dụ: `--cpus="1.5"` cho phép dùng tối đa 1.5 nhân CPU).
   - Thiết lập trọng số ưu tiên sử dụng CPU (`--cpu-shares`) giữa các container khi hệ thống xảy ra tranh chấp tài nguyên.
3. **Kiểm soát I/O và Network:**
   - Giới hạn tốc độ đọc/ghi đĩa (Block I/O read/write bps) và băng thông mạng, tránh tình trạng "Noisy Neighbor" (một ứng dụng quá tải đè bẹp hiệu năng toàn máy chủ).
4. **Cgroup Driver (`systemd` vs `cgroupfs`):**
   - Trên các bản phân phối Linux hiện đại sử dụng `systemd` làm init system, Docker khuyến nghị cấu hình `Cgroup Driver` là `systemd` để đồng bộ quản lý cây tài nguyên với hệ điều hành (tránh tình trạng hai trình quản lý `systemd` và `cgroupfs` tranh chấp cấu trúc phân cấp).

---

## 4. Kiểm tra hệ thống và trích xuất thông số từ `docker info`

### 4.1. Thực thi các lệnh kiểm tra

Hai lệnh cơ bản được thực thi bằng tài khoản người dùng thông thường (không cần `sudo` nhờ user đã được thêm vào nhóm `docker`):

```bash
# Kiểm tra phiên bản Docker Client và Server
docker version

# Kiểm tra toàn bộ thông số cấu hình của Docker Daemon
docker info

# Lệnh trích xuất nhanh thông tin Cgroup Driver
docker info | grep -i cgroup
```

### 4.2. Log kết quả thực thi (Terminal Output)

#### Đầu ra lệnh `docker version`:
```text
Client: Docker Engine - Community
 Version:           27.3.1
 API version:       1.47
 Go version:        go1.22.7
 Git commit:        ce12230
 Built:             Fri Sep 20 11:41:00 2024
 OS/Arch:           linux/amd64
 Context:           default

Server: Docker Engine - Community
 Engine:
  Version:          27.3.1
  API version:      1.47 (minimum version 1.24)
  Go version:       go1.22.7
  Git commit:       4123429
  Built:            Fri Sep 20 11:41:00 2024
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          v1.7.22
  GitCommit:        7f7fdf5cedd04996ab7a10ac07c41a4a6012c8b8
 runc:
  Version:          1.1.14
  GitCommit:        v1.1.14-0-g2c9f560
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0
```

#### Đầu ra lệnh `docker info`:
```text
Client: Docker Engine - Community
 Version:    27.3.1
 Context:    default
 Debug Mode: false
 Plugins:
  buildx: Docker Buildx (Docker Inc.)
    Version:  v0.17.1
    Path:     /usr/libexec/docker/cli-plugins/docker-buildx
  compose: Docker Compose (Docker Inc.)
    Version:  v2.29.7
    Path:     /usr/libexec/docker/cli-plugins/docker-compose

Server:
 Containers: 2
  Running: 1
  Paused: 0
  Stopped: 1
 Images: 3
 Server Version: 27.3.1
 Storage Driver: overlay2
  Backing Filesystem: extfs
  Supports d_type: true
  Using metacopy: false
 Logging Driver: json-file
 Cgroup Driver: systemd
 Cgroup Version: 2
 Plugins:
  Volume: local
  Network: bridge host ipvlan macvlan null overlay
  Log: awslogs fluentd gcplogs gelf journald json-file local splunk syslog
 Swarm: inactive
 Runtimes: io.containerd.runc.v2 runc
 Default Runtime: runc
 Init Binary: docker-init
 containerd version: 7f7fdf5cedd04996ab7a10ac07c41a4a6012c8b8
 runc version: v1.1.14-0-g2c9f560
 init version: de40ad0
 Security Options:
  apparmor
  seccomp
   Profile: builtin
  cgroupns
 Kernel Version: 6.8.0-45-generic
 Operating System: Ubuntu 24.04.1 LTS
 OSType: linux
 Architecture: x86_64
 CPUs: 4
 Total Memory: 7.648GiB
 Name: devops-node
 ID: 7fc7c6a9-8588-46fb-a0b2-e1c79cbe8784
 Docker Root Dir: /var/lib/docker
 Debug Mode: false
 Experimental: false
 Insecure Registries:
  127.0.0.0/8
 Live Restore Enabled: false
```

#### Lọc thông số Cgroup:
```bash
$ docker info | grep -i cgroup
 Cgroup Driver: systemd
 Cgroup Version: 2
```

### 4.3. Bảng trích xuất và giải thích các thông số cơ bản

| Thông số trích xuất | Giá trị hệ thống | Giải thích ý nghĩa kỹ thuật |
| :--- | :--- | :--- |
| **Server Version** | `27.3.1` | Phiên bản Docker Engine Daemon đang vận hành trên máy chủ (Docker Community Edition). |
| **Cgroup Driver** | `systemd` | Trình điều khiển quản lý cgroups. Sử dụng `systemd` giúp Docker tương thích hoàn toàn với hệ điều hành Linux hiện đại (tránh xung đột phân cấp tài nguyên). |
| **Cgroup Version** | `2` (cgroups v2) | Phiên bản cgroup mới nhất của Linux Kernel cung cấp cấu trúc phân cấp tài nguyên hợp nhất (unified hierarchy) và kiểm soát tài nguyên đồng nhất. |
| **Số lượng Containers** | `2` (Running: 1, Stopped: 1) | Tổng số container hiện có trên node: 1 container đang chạy phục vụ dịch vụ và 1 container đã dừng. |
| **Số lượng Images** | `3` | Số lượng Docker Images đã tải hoặc build thành công lưu trữ tại local (`/var/lib/docker`). |
| **Storage Driver** | `overlay2` | Trình điều khiển hệ thống tệp tin phân tầng (layered filesystem) tiêu chuẩn cao của Docker, tối ưu hiệu năng I/O. |
| **Kernel Version** | `6.8.0-45-generic` | Phiên bản Linux Kernel của Host OS mà tất cả các container chia sẻ dùng chung. |

---

## 5. Đánh giá và Kết luận

1. **Hiệu quả chuyển dịch:** Việc chuyển dịch từ VPS truyền thống sang Docker Container giúp loại bỏ hoàn toàn chi phí bản quyền và hao phí phần cứng cho các Guest OS riêng biệt, rút ngắn thời gian triển khai từ hàng phút xuống hàng giây.
2. **Nền tảng công nghệ vững chắc:** Container hoạt động an toàn và ổn định nhờ sự kết hợp chặt chẽ giữa **Linux Namespaces** (đảm bảo không gian cô lập logic) và **cgroups** (đảm bảo giới hạn và an toàn tài nguyên phần cứng).
3. **Môi trường vận hành chuẩn hóa:** Hệ thống đã kích hoạt thành công Docker Engine với cấu hình chuẩn doanh nghiệp (`Cgroup Driver: systemd`, `Cgroup Version: 2`, `Storage Driver: overlay2`), đáp ứng đầy đủ tiêu chuẩn sẵn sàng triển khai các ứng dụng container hóa và hệ thống điều phối mở rộng (Kubernetes).
