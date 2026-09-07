# U01-01 — Vagrant cơ bản: Vagrantfile, box, provider

> **Unit:** U01 — Vagrant & Lab Infrastructure
> **Tuần:** 1 · **Giờ dự kiến:** 4 · **Độ khó:** Dễ
> **Tài liệu tham khảo:** [Vagrant Docs — Getting Started](https://developer.hashicorp.com/vagrant/docs) · [Vagrant Cloud](https://portal.cloud.hashicorp.com/vagrant/discover) · HashiCorp Learn

---

## 1. Mục tiêu bài học

Sau bài này bạn phải:

1. Hiểu Vagrant giải quyết vấn đề gì và vì sao "dựng lab bằng code" tốt hơn "dựng lab bằng chuột".
2. Phân biệt được **provider** (VirtualBox vs libvirt/KVM) và biết chọn cái nào cho máy của mình.
3. Hiểu khái niệm **box** — nơi tìm, cách quản lý phiên bản.
4. Nắm vững **vòng đời của 1 máy ảo Vagrant**: `up` → `ssh` → `halt`/`suspend` → `destroy`.
5. Đọc hiểu và tự viết được 1 `Vagrantfile` tối thiểu.
6. Dựng ra 1 lab tái lập được — người khác clone repo, chạy `vagrant up` là có VM giống hệt bạn.

---

## 2. Lý thuyết

### 2.1. Vagrant giải quyết vấn đề gì

Trước Vagrant, dựng 1 máy ảo để thực hành thường là: mở VirtualBox/VMware bằng GUI, click "New", chọn ISO, cài hệ điều hành bằng tay, cấu hình network bằng tay. Cách này có 3 vấn đề:

- **Không tái lập được.** Máy ảo bạn dựng hôm nay và máy ảo đồng nghiệp dựng hôm sau gần như chắc chắn khác nhau — khác phiên bản OS, khác cấu hình network, khác gói cài sẵn.
- **Không versioning được.** Cấu hình nằm trong đầu bạn hoặc trong ảnh chụp màn hình, không nằm trong git.
- **Chậm khi cần dựng lại.** Máy ảo hỏng → phải làm lại từ đầu bằng tay, mất hàng giờ.

Vagrant giải quyết cả 3: bạn mô tả máy ảo bằng 1 file text (`Vagrantfile`, cú pháp Ruby), và `vagrant up` sẽ dựng đúng y hệt máy đó ở bất kỳ đâu, bất kỳ lúc nào, từ số 0.

> **Vagrant khác gì Terraform?** Cả hai đều là "infrastructure as code", nhưng Vagrant chuyên cho **máy ảo cục bộ** (dev/lab), còn Terraform chuyên cho **hạ tầng cloud** (AWS, GCP...). Bạn sẽ học Terraform ở Unit 11 — lúc đó khái niệm "mô tả hạ tầng bằng code, apply để tạo ra" sẽ rất quen thuộc vì đã làm với Vagrant từ Unit 01.

### 2.2. Provider: VirtualBox vs libvirt/KVM

Vagrant tự nó không chạy máy ảo — nó điều khiển 1 **provider** (trình ảo hoá thật sự) đứng phía sau.

| Provider | Nền tảng | Tốc độ | Khi nào dùng |
|---|---|---|---|
| **VirtualBox** | Windows / macOS / Linux | Chậm hơn trên Linux (dùng kernel module riêng của Oracle) | Mặc định, dễ cài, đa nền tảng — chọn nếu bạn dùng Windows/macOS |
| **libvirt (KVM)** | Chỉ Linux | **Nhanh hơn đáng kể** trên Linux — dùng thẳng KVM của kernel, ít lớp trung gian | Chọn nếu máy bạn chạy Linux (đặc biệt khi có `/dev/kvm`) |

**Vì sao libvirt nhanh hơn trên Linux:** KVM (Kernel-based Virtual Machine) là công nghệ ảo hoá tích hợp sẵn trong kernel Linux — máy ảo chạy gần như trực tiếp trên CPU thật (hardware-assisted virtualization qua VT-x/AMD-V), không qua thêm 1 tầng hypervisor riêng như VirtualBox phải cài thêm.

Kiểm tra máy bạn có hỗ trợ KVM không:

```bash
# Có flag vmx (Intel) hoặc svm (AMD) không
grep -E "vmx|svm" /proc/cpuinfo

# /dev/kvm có tồn tại và dùng được không
kvm-ok   # hoặc: ls -la /dev/kvm
```

**Cài plugin libvirt cho Vagrant** (nếu chọn hướng này):

```bash
sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients virtinst
vagrant plugin install vagrant-libvirt
```

Quyết định của lộ trình này: nếu bạn chạy Linux, **ưu tiên libvirt** — nhanh hơn, tiết kiệm tài nguyên hơn, phù hợp khi bạn cần dựng cụm 3-5 node cho các unit sau. Nếu bạn dùng Windows/macOS, dùng VirtualBox.

### 2.3. Box là gì

**Box** là 1 template máy ảo đã đóng gói sẵn (giống "base image" của Docker, nhưng nặng hơn nhiều vì chứa cả hệ điều hành đầy đủ). Bạn không tự cài OS từ ISO — bạn tải 1 box có sẵn rồi Vagrant clone ra máy ảo mới từ đó.

Tìm box ở [Vagrant Cloud](https://portal.cloud.hashicorp.com/vagrant/discover) — mỗi box có tên dạng `tổ_chức/tên`, ví dụ `generic/ubuntu2204`, `bento/ubuntu-22.04`.

```bash
# Tải 1 box về máy (Vagrant tự tải khi vagrant up nếu chưa có, nhưng có thể tải trước)
vagrant box add generic/ubuntu2204

# Liệt kê box đã tải
vagrant box list

# Box có bản cập nhật không
vagrant box outdated

# Cập nhật box lên bản mới nhất
vagrant box update

# Xoá box không dùng nữa (giải phóng dung lượng — mỗi box thường 300MB-1GB)
vagrant box prune
```

> **Vì sao phải khoá phiên bản box?** Giống lock file của `uv`/`pip` — nếu không khoá version, 2 người chạy `vagrant up` ở 2 thời điểm khác nhau có thể nhận 2 bản box khác nhau (tổ chức maintain box có thể release bản mới bất kỳ lúc nào), dẫn đến môi trường lệch nhau dù cùng 1 `Vagrantfile`. Khoá bằng `config.vm.box_version` trong Vagrantfile.

### 2.4. Vòng đời của 1 máy ảo Vagrant

```
vagrant up          # Tạo (nếu chưa có) và khởi động VM, chạy provisioner lần đầu
vagrant ssh          # SSH vào VM
vagrant halt          # Tắt VM (giữ nguyên đĩa, giải phóng RAM/CPU)
vagrant suspend        # "Đóng băng" VM (lưu cả trạng thái RAM ra đĩa — resume nhanh hơn halt)
vagrant resume         # Khôi phục từ suspend
vagrant reload         # halt + up (dùng khi đổi cấu hình Vagrantfile, ví dụ đổi RAM)
vagrant provision       # Chạy lại provisioner mà không restart VM
vagrant destroy -f       # XOÁ HẲN VM (mất mọi dữ liệu bên trong, đĩa bị xoá)
```

**Lệnh kiểm tra trạng thái:**

```bash
vagrant status          # Trạng thái VM trong thư mục hiện tại
vagrant global-status      # Trạng thái TẤT CẢ VM Vagrant đang quản lý trên máy (mọi thư mục)
```

Điểm quan trọng nhất cần khắc cốt ghi tâm: **`vagrant destroy` xoá sạch mọi thứ bên trong VM.** Đây không phải bug — đây là **tính năng cốt lõi**: máy ảo lab là phù du (ephemeral), mọi cấu hình quan trọng phải nằm trong code (Vagrantfile + provisioner script), không nằm trong trạng thái tay bạn gõ vào VM. Nếu `destroy` rồi `up` lại mà bạn "mất" gì đó quan trọng, nghĩa là bạn đã làm sai quy trình — cấu hình đó lẽ ra phải nằm trong code.

### 2.5. Cấu trúc Vagrantfile tối thiểu

```ruby
Vagrant.configure("2") do |config|
  # Box: nguồn máy ảo, khoá version để tái lập được
  config.vm.box = "generic/ubuntu2204"
  config.vm.box_version = "4.3.12"

  # Hostname bên trong VM (khác với tên "define" nếu multi-machine — xem Bài 1.3)
  config.vm.hostname = "lab-basic"

  # Tài nguyên cấp cho VM (cấu hình cho provider cụ thể)
  config.vm.provider "libvirt" do |lv|
    lv.memory = 2048
    lv.cpus = 2
  end
  config.vm.provider "virtualbox" do |vb|
    vb.memory = 2048
    vb.cpus = 2
  end
end
```

**Giải thích từng phần:**

- `Vagrant.configure("2")` — số `"2"` là phiên bản cú pháp cấu hình Vagrant (V2 API), gần như mọi Vagrantfile hiện đại đều dùng số này.
- `config.vm.box` — bắt buộc phải có, đây là template VM sẽ được clone.
- `config.vm.box_version` — nên luôn khai báo tường minh (xem lý do ở mục 2.3).
- `config.vm.hostname` — đặt tên máy bên trong OS (`hostname` command sẽ trả về giá trị này).
- `config.vm.provider "..." do |x| ... end` — cấu hình riêng cho từng provider; Vagrant chỉ áp dụng block khớp với provider đang chạy.

### 2.6. `vagrant provision` và tính idempotent

Khi bạn chạy `vagrant up` lần đầu, Vagrant chạy toàn bộ provisioner (script cài đặt, cấu hình...). Nhưng `vagrant up` **lần thứ 2** trên VM đã tồn tại **không** tự động chạy lại provisioner — Vagrant giả định VM đã ở đúng trạng thái rồi.

Muốn chạy lại provisioner có chủ đích:

```bash
vagrant up --provision          # up + ép chạy provisioner dù VM đã tồn tại
vagrant provision              # chỉ chạy provisioner, không đổi trạng thái VM
vagrant provision --provision-with shell   # chỉ chạy provisioner tên "shell" (nếu có nhiều loại)
```

Đây là lý do bạn sẽ học Ansible ngay ở Unit 02 — Ansible role viết đúng chuẩn thì **idempotent**: chạy lại bao nhiêu lần cũng ra cùng 1 kết quả, không lỗi, không làm gì thêm nếu đã đúng trạng thái. Shell script tự viết tay thường KHÔNG có tính chất này (chạy `apt install` 2 lần thì vô hại, nhưng `echo "x" >> file` 2 lần thì file bị nhân đôi nội dung).

---

## 3. Thực hành

### 3.1. Cài Vagrant

```bash
# Debian/Ubuntu — cài từ kho chính thức HashiCorp
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install vagrant

# Kiểm tra
vagrant --version
```

### 3.2. Dựng VM đầu tiên

Tạo thư mục `lab/01-basic/` (sẽ dùng làm deliverable của bài này):

```bash
mkdir -p lab/01-basic && cd lab/01-basic
```

Viết `Vagrantfile` (dựa trên mẫu ở mục 2.5), sau đó:

```bash
vagrant up               # Dựng VM lần đầu — theo dõi log để hiểu Vagrant đang làm gì
vagrant status            # Xác nhận trạng thái "running"
vagrant ssh               # SSH vào VM
  # Trong VM: kiểm tra hostname, OS
  hostname
  cat /etc/os-release
  exit
vagrant halt              # Tắt VM
vagrant up                # Bật lại — nhanh hơn lần đầu vì không cần tải box nữa
```

### 3.3. README cho thư mục lab

Mỗi `lab/*` nên có 1 `README.md` ngắn ghi rõ yêu cầu hệ thống và cách chạy — vì đây chính là "tiêu chí hoàn thành" của bài: người khác (hoặc chính bạn 3 tháng sau) clone repo, đọc README, chạy `vagrant up` là ra kết quả giống hệt.

Mẫu tối thiểu:

```markdown
# Lab 01 — Vagrant cơ bản

## Yêu cầu
- Vagrant >= 2.4
- Provider: libvirt (khuyến nghị trên Linux) hoặc VirtualBox
- RAM trống >= 2GB

## Chạy
\`\`\`bash
vagrant up
vagrant ssh
\`\`\`

## Dọn dẹp
\`\`\`bash
vagrant destroy -f
\`\`\`
```

---

## 4. Bài tập

**BT1:** Dựng 1 VM, SSH vào, cài nginx thủ công (`sudo apt install -y nginx`), sau đó `vagrant destroy -f` rồi `vagrant up` lại — xác nhận nginx **biến mất**. Ghi lại quan sát này vào `SUBMISSION.md`: đây là cách trực quan nhất để hiểu tính phù du (ephemeral) của VM lab.

**BT2:** Nếu máy bạn chạy Linux, so sánh thời gian `vagrant up` (từ lúc gõ lệnh tới lúc SSH vào được) giữa provider VirtualBox và libvirt. Dùng `time vagrant up` để đo. Ghi lại số đo cụ thể (giây).

**BT3:** Thử dựng VM với 3 box khác nhau: 1 bản Ubuntu (`generic/ubuntu2204`), 1 bản Rocky/Alma (`generic/rocky9`), 1 bản Debian (`generic/debian12`). Với mỗi box, SSH vào và ghi lại:
- User SSH mặc định là gì (`whoami`)
- Package manager là gì (`apt` / `dnf` / `yum`)
- Có `sudo` không cần password không

### Deliverable (Yêu cầu đầu ra)

Thư mục `lab/01-basic/` trong repo này có:
- `Vagrantfile` dựng 1 VM Ubuntu 22.04
- `vagrant up` chạy sạch từ máy trắng (không cần sửa gì thêm)
- SSH vào được
- `README.md` ghi rõ yêu cầu hệ thống và cách chạy

---

## 5. Câu hỏi tự kiểm tra

> Viết câu trả lời của bạn vào `SUBMISSION.md` **trước**, bằng lời của chính bạn — sau đó mới đối chiếu với phần "Gợi ý trả lời" bên dưới để tự chấm.

1. Box khác image Docker ở điểm nào?
2. `vagrant halt` vs `vagrant suspend` vs `vagrant destroy` khác nhau ở đâu, khi nào dùng cái nào?
3. `Vagrantfile` nên commit vào git không? Còn thư mục `.vagrant/` thì sao? Vì sao?
4. Vì sao cần khoá phiên bản box (`config.vm.box_version`)?
5. `vagrant up` chạy lần thứ 2 trên VM đã tồn tại có tự chạy lại provisioner không? Muốn ép chạy lại thì làm sao?

<details>
<summary><b>Gợi ý trả lời (bấm để mở — chỉ xem SAU khi đã tự trả lời)</b></summary>

**1.** Docker image là snapshot của **filesystem** dùng chung kernel với host (container hoá ở tầng process/namespace) — nhẹ, khởi động trong mili-giây. Vagrant box là template của **cả 1 máy ảo đầy đủ** (kernel riêng, thiết bị ảo riêng) — nặng hơn nhiều (300MB-1GB+), khởi động trong vài giây tới vài chục giây, nhưng cô lập hoàn toàn ở tầng phần cứng ảo hoá, phù hợp khi bạn cần mô phỏng 1 hệ thống Linux đầy đủ (ví dụ để test cluster, kernel module, systemd).

**2.** `halt` tắt VM hoàn toàn — giải phóng RAM/CPU, giữ nguyên đĩa; khởi động lại (`up`) mất thời gian boot OS bình thường. `suspend` lưu toàn bộ trạng thái RAM ra đĩa rồi dừng — `resume` khôi phục gần như tức thì, không cần boot lại, nhưng chiếm dung lượng đĩa bằng đúng RAM đã cấp. `destroy` xoá hẳn đĩa ảo — mất toàn bộ dữ liệu, lần `up` sau clone lại từ box y hệt ban đầu. Dùng `halt` khi nghỉ dài, `suspend` khi tạm dừng ngắn muốn quay lại nhanh, `destroy` khi muốn đảm bảo môi trường sạch 100%.

**3.** `Vagrantfile` **phải** commit — đây chính là "code" định nghĩa hạ tầng, là thứ duy nhất đảm bảo người khác dựng lại được y hệt. Thư mục `.vagrant/` **không** commit — đây là state cục bộ (ID máy ảo trên provider, đường dẫn tuyệt đối trên máy bạn...), hoàn toàn không di chuyển được sang máy khác và tái sinh tự động mỗi lần `vagrant up`. Cho vào `.gitignore`.

**4.** Nếu không khoá version, box có thể được nhà phát hành cập nhật bất kỳ lúc nào. Bạn `vagrant up` hôm nay ra Ubuntu kernel bản X, đồng nghiệp `vagrant up` tuần sau với cùng `Vagrantfile` có thể ra kernel bản Y (nếu box đã update) — 2 môi trường lệch nhau âm thầm, khó debug khi có lỗi "chỉ xảy ra trên máy tôi". Khoá version đảm bảo mọi người, mọi thời điểm, luôn ra đúng 1 baseline.

**5.** Không. Vagrant coi VM đã tồn tại là "đã đúng trạng thái" nên bỏ qua provisioner ở lần `up` thứ 2 trở đi (kể cả khi bạn sửa provisioner script). Muốn ép chạy lại: `vagrant up --provision` (chạy up + ép provision) hoặc `vagrant provision` (chỉ chạy provisioner, VM vẫn đang chạy, không restart). Đây là điểm rất hay bị quên khi debug: sửa script provisioner xong `vagrant up` lại tưởng đã áp dụng, nhưng thực ra chưa chạy gì cả.

</details>

---

## 6. Tổng kết & bước tiếp theo

Bạn đã có:
- ✅ Vagrant cài đặt, hiểu rõ vòng đời VM và khái niệm box/provider.
- ✅ 1 lab tối thiểu (`lab/01-basic/`) — nền cho các bài tiếp theo của Unit 01.
- ✅ Hiểu vì sao VM lab là phù du và mọi cấu hình quan trọng phải nằm trong code.

**Bài tiếp theo:** U01-02 — Mạng trong Vagrant: private network, port forwarding, IP tĩnh.

---

## 7. Ghi điểm (tự chấm)

| Tiêu chí | Điểm tối đa |
|---|---|
| `lab/01-basic/Vagrantfile` chạy sạch từ máy trắng bằng `vagrant up` | 3 |
| SSH vào được, `README.md` đủ để người khác làm theo không cần hỏi thêm | 2 |
| BT1-BT3 làm đủ, có ghi số liệu/quan sát cụ thể (không chỉ "đã làm") | 3 |
| Trả lời đúng ≥4/5 câu hỏi tự kiểm tra (viết ra bằng lời của bạn) | 2 |
| **Tổng** | **10** |

Đạt ≥ 8/10 → tick ☑ trong `PROGRESS.md` và ở sheet `U01_Vagrant_Lab` trong workbook Excel, ghi ngày hoàn thành + giờ thực tế. Dưới 8 → xem lại phần sai, sửa và làm lại.

---

## 8. Ghi chú nếu máy bạn yếu (< 16GB RAM)

Bài này chỉ dựng 1 VM nên không bị ảnh hưởng. Ảnh hưởng bắt đầu từ Bài 1.3 (multi-machine) và rõ nhất ở Bài 1.7 (dự án 5 node) — lúc đó cân nhắc giảm xuống 3 node (1 control-plane + 2 worker) thay vì 5. Xem chi tiết ở `U01_Vagrant_Lab/README.md`.
