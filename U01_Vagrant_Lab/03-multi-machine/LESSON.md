# U01-03 — Multi-machine Vagrantfile: dựng cụm bằng vòng lặp

> **Unit:** U01 — Vagrant & Lab Infrastructure
> **Tuần:** 2 · **Giờ dự kiến:** 5 · **Độ khó:** Dễ
> **Tài liệu tham khảo:** [Vagrant Docs — Multi-Machine](https://developer.hashicorp.com/vagrant/docs/multi-machine) · Ruby `Array#each`/`times` cơ bản

---

## 1. Mục tiêu bài học

Sau bài này bạn phải:

1. Hiểu vì sao viết tay N block `config.vm.define` không mở rộng được, và vòng lặp Ruby giải quyết vấn đề đó thế nào.
2. Dùng được vòng lặp để sinh ra N máy có tên, hostname, IP tăng dần tự động.
3. Tách được **provisioning khác nhau theo loại node** (ví dụ: control-plane khác worker) trong cùng 1 vòng lặp.
4. Biết cách **lên/xuống từng máy riêng lẻ** hoặc cả cụm, và các lệnh quản lý cụm thường dùng.
5. Có 1 `Vagrantfile` sinh cụm N node (mặc định 5, cấu hình được) — **đây chính là lab dùng cho toàn bộ Unit 02 (Ansible) và Unit 03 (Kubespray) về sau.**

---

## 2. Vấn đề: viết tay không mở rộng được

### 2.1. Lý thuyết

Nếu bạn viết multi-machine Vagrantfile theo cách "chép tay" từng block như bài 1.2:

```ruby
Vagrant.configure("2") do |config|
  config.vm.define "worker1" do |w|
    w.vm.box = "generic/ubuntu2204"
    w.vm.hostname = "worker1"
    w.vm.network "private_network", ip: "192.168.50.11"
  end

  config.vm.define "worker2" do |w|
    w.vm.box = "generic/ubuntu2204"
    w.vm.hostname = "worker2"
    w.vm.network "private_network", ip: "192.168.50.12"
  end

  # ... lặp lại y hệt cho worker3, worker4, worker5 ...
end
```

Vấn đề:
- Muốn có 5 worker → chép-dán 5 lần, đổi tay 2 chỗ (số thứ tự + IP) mỗi lần → **dễ gõ nhầm IP trùng nhau**.
- Muốn đổi từ 5 lên 10 node → sửa tay thêm 5 block nữa.
- Muốn đổi RAM cho tất cả worker → phải sửa N chỗ giống hệt nhau.

`Vagrantfile` là **file Ruby thật sự** (không phải YAML/JSON tĩnh), nên bạn dùng được toàn bộ cú pháp Ruby cơ bản: biến, vòng lặp, mảng, string interpolation. Đây là điểm khác biệt lớn giữa Vagrant và các công cụ cấu hình khai báo thuần (YAML) — bạn **generate cấu hình bằng code**.

### 2.2. Vòng lặp cơ bản: `times` và tên/IP theo index

```ruby
NUM_WORKERS = 3
BASE_IP = "192.168.50."   # .10 = master, .11, .12, .13 = worker1..3

Vagrant.configure("2") do |config|
  (1..NUM_WORKERS).each do |i|
    config.vm.define "worker#{i}" do |worker|
      worker.vm.box = "generic/ubuntu2204"
      worker.vm.box_version = "4.3.12"
      worker.vm.hostname = "worker#{i}"
      worker.vm.network "private_network", ip: "#{BASE_IP}#{10 + i}"

      worker.vm.provider "libvirt" do |lv|
        lv.memory = 2048
        lv.cpus = 2
      end
    end
  end
end
```

**Giải thích từng phần:**
- `NUM_WORKERS = 3` — hằng số đặt ở đầu file, đổi 1 chỗ này là đổi số lượng máy toàn bộ.
- `(1..NUM_WORKERS).each do |i| ... end` — vòng lặp Ruby chạy từ 1 tới `NUM_WORKERS`, biến `i` là số thứ tự hiện tại.
- `"worker#{i}"` — string interpolation của Ruby (giống f-string Python `f"worker{i}"`), sinh ra `worker1`, `worker2`, `worker3`.
- `"#{BASE_IP}#{10 + i}"` — ghép chuỗi + phép tính, sinh IP `192.168.50.11`, `.12`, `.13`.
- Toàn bộ block `config.vm.define` giờ nằm **bên trong vòng lặp** — mỗi lần lặp tạo ra 1 máy ảo mới với tên/IP khác nhau.

> **So sánh với vòng lặp trong lập trình quen thuộc:** đây chính xác là kiểu `for i in range(1, N+1)` mà bạn hay dùng trong Python để sinh script lặp lại N lần — chỉ khác là ở đây, mỗi vòng lặp không in ra màn hình mà **định nghĩa thêm 1 máy ảo** vào cấu hình cụm.

### 2.3. Bài tập A — Sinh cụm 3 node bằng vòng lặp

**BT-A1:** Viết `Vagrantfile` (đặt tại `lab/04-multi-machine/`) dùng vòng lặp sinh ra 3 máy `node1`, `node2`, `node3` với IP lần lượt `192.168.60.11`, `.12`, `.13`. Chạy `vagrant up`, xác nhận cả 3 máy lên (`vagrant status` phải thấy 3 dòng "running").

**BT-A2:** Đổi `NUM_WORKERS` từ 3 thành 5, chạy lại `vagrant up`. Xác nhận Vagrant chỉ **tạo thêm 2 máy mới** (node4, node5) mà không đụng tới 3 máy cũ đang chạy — quan sát log để thấy rõ điều này.

**BT-A3:** Từ `node1`, ping cả `node2` và `node5` để xác nhận toàn bộ cụm nằm chung 1 mạng private.

<details>
<summary><b>💡 Lời giải BT-A</b></summary>

**Vagrantfile hoàn chỉnh:**

```ruby
NUM_NODES = 5   # đổi số này để scale cụm

Vagrant.configure("2") do |config|
  (1..NUM_NODES).each do |i|
    config.vm.define "node#{i}" do |node|
      node.vm.box = "generic/ubuntu2204"
      node.vm.box_version = "4.3.12"
      node.vm.hostname = "node#{i}"
      node.vm.network "private_network", ip: "192.168.60.#{10 + i}"

      node.vm.provider "libvirt" do |lv|
        lv.memory = 1024
        lv.cpus = 1
      end
    end
  end
end
```

**Chạy & kiểm tra:**
```bash
vagrant up
vagrant status
# node1  running (libvirt)
# node2  running (libvirt)
# node3  running (libvirt)
```

**Tăng lên 5 node — chỉ sửa `NUM_NODES = 5` rồi:**
```bash
vagrant up
# Log sẽ chỉ thấy "Bringing machine 'node4' up..." và "'node5' up..."
# node1/2/3 hiện dòng "==> node1: Machine already provisioned" hoặc bị bỏ qua hoàn toàn vì đã running
```

**Test ping toàn cụm:**
```bash
vagrant ssh node1
ping -c 2 192.168.60.12   # node2
ping -c 2 192.168.60.15   # node5
exit
```

Cả 2 lệnh ping phải reply — xác nhận toàn bộ 5 node nằm chung mạng `192.168.60.0/24`.

</details>

---

## 3. Provisioning khác nhau theo loại node

### 3.1. Lý thuyết

Một cụm thật (ví dụ chuẩn bị cho Kubespray ở Unit 03) hiếm khi có N máy **giống hệt nhau** — thường có 1-3 node đóng vai trò **control-plane/master** và các node còn lại là **worker**. Bạn cần:

- Định nghĩa **cấu trúc cụm** bằng dữ liệu (không phải chỉ 1 con số N), ví dụ 1 mảng Ruby.
- Trong vòng lặp, **rẽ nhánh** theo loại node để chạy provisioner khác nhau, hoặc cấp tài nguyên khác nhau (control-plane thường cần nhiều RAM hơn worker nhỏ).

```ruby
NODES = [
  { name: "master1", ip: "192.168.60.10", role: "control-plane", mem: 2048, cpu: 2 },
  { name: "worker1", ip: "192.168.60.11", role: "worker",        mem: 1024, cpu: 1 },
  { name: "worker2", ip: "192.168.60.12", role: "worker",        mem: 1024, cpu: 1 },
]

Vagrant.configure("2") do |config|
  NODES.each do |n|
    config.vm.define n[:name] do |node|
      node.vm.box = "generic/ubuntu2204"
      node.vm.box_version = "4.3.12"
      node.vm.hostname = n[:name]
      node.vm.network "private_network", ip: n[:ip]

      node.vm.provider "libvirt" do |lv|
        lv.memory = n[:mem]
        lv.cpus = n[:cpu]
      end

      # Rẽ nhánh provisioning theo role
      if n[:role] == "control-plane"
        node.vm.provision "shell", inline: <<-SHELL
          echo ">> Provisioning CONTROL-PLANE: #{n[:name]}"
          apt update
          apt install -y curl conntrack
        SHELL
      else
        node.vm.provision "shell", inline: <<-SHELL
          echo ">> Provisioning WORKER: #{n[:name]}"
          apt update
          apt install -y curl
        SHELL
      end
    end
  end
end
```

**Giải thích:**
- `NODES` là 1 **mảng các Hash** (giống list of dict trong Python) — mỗi phần tử mô tả đầy đủ 1 máy: tên, IP, vai trò, tài nguyên.
- `NODES.each do |n| ... end` — lặp qua từng phần tử mảng, `n` là Hash hiện tại, truy cập bằng `n[:name]`, `n[:ip]`...
- Đây là bước đệm quan trọng: **cấu trúc dữ liệu này gần giống hệt inventory mà Ansible/Kubespray dùng** (Unit 02, Unit 03) — bạn đang làm quen dần với tư duy "mô tả cụm bằng dữ liệu có cấu trúc" thay vì viết tay từng máy.

### 3.2. Bài tập B — Cụm bất đối xứng (1 master + N worker)

**BT-B1:** Sửa Vagrantfile ở BT-A thành cụm bất đối xứng: 1 `master` (IP `.10`, 2GB RAM) + 3 `worker` (IP `.11`-`.13`, 1GB RAM mỗi máy), dùng cấu trúc mảng `NODES` như trên.

**BT-B2:** Thêm provisioning rẽ nhánh: master in ra dòng `"Tôi là control-plane"`, worker in ra dòng `"Tôi là worker, chờ lệnh từ control-plane"`. Chạy `vagrant up`, xác nhận log của từng máy in đúng dòng tương ứng với vai trò.

**BT-B3:** Chỉ khởi động lại 1 mình `master` (không đụng tới worker): tìm đúng lệnh Vagrant cho việc này.

<details>
<summary><b>💡 Lời giải BT-B</b></summary>

**Vagrantfile:**

```ruby
NODES = [
  { name: "master",  ip: "192.168.60.10", role: "control-plane", mem: 2048, cpu: 2 },
  { name: "worker1", ip: "192.168.60.11", role: "worker",        mem: 1024, cpu: 1 },
  { name: "worker2", ip: "192.168.60.12", role: "worker",        mem: 1024, cpu: 1 },
  { name: "worker3", ip: "192.168.60.13", role: "worker",        mem: 1024, cpu: 1 },
]

Vagrant.configure("2") do |config|
  NODES.each do |n|
    config.vm.define n[:name] do |node|
      node.vm.box = "generic/ubuntu2204"
      node.vm.box_version = "4.3.12"
      node.vm.hostname = n[:name]
      node.vm.network "private_network", ip: n[:ip]

      node.vm.provider "libvirt" do |lv|
        lv.memory = n[:mem]
        lv.cpus = n[:cpu]
      end

      if n[:role] == "control-plane"
        node.vm.provision "shell", inline: "echo 'Tôi là control-plane'"
      else
        node.vm.provision "shell", inline: "echo 'Tôi là worker, chờ lệnh từ control-plane'"
      end
    end
  end
end
```

**Chạy & quan sát log:**
```bash
vagrant up
# ==> master: Tôi là control-plane
# ==> worker1: Tôi là worker, chờ lệnh từ control-plane
# ==> worker2: Tôi là worker, chờ lệnh từ control-plane
# ==> worker3: Tôi là worker, chờ lệnh từ control-plane
```

**BT-B3 — chỉ restart 1 máy:**
```bash
vagrant reload master
# Chỉ tắt/bật lại "master", 3 worker không bị ảnh hưởng
```

Tương tự: `vagrant halt master`, `vagrant up master`, `vagrant destroy master -f` đều chỉ tác động đúng 1 máy được chỉ tên.

</details>

---

## 4. Quản lý cụm nhiều máy: lệnh thường dùng

### 4.1. Lý thuyết

Khi đã có nhiều máy trong 1 `Vagrantfile`, các lệnh Vagrant đều nhận thêm **tên máy** (tùy chọn) làm tham số cuối:

```bash
vagrant up                # lên TOÀN BỘ máy trong Vagrantfile
vagrant up worker1          # chỉ lên máy "worker1"

vagrant halt               # tắt toàn bộ
vagrant halt worker1         # chỉ tắt worker1

vagrant destroy -f            # xoá toàn bộ (không hỏi xác nhận)
vagrant destroy worker1 -f      # chỉ xoá worker1

vagrant ssh worker1            # SSH vào đúng máy worker1 (bắt buộc chỉ tên khi có ≥2 máy)

vagrant status               # trạng thái toàn bộ máy trong Vagrantfile hiện tại
vagrant global-status           # trạng thái MỌI Vagrantfile trên máy (tất cả thư mục)
```

**Thứ tự khởi động quan trọng khi có phụ thuộc:** Vagrant lên máy theo **đúng thứ tự định nghĩa trong file** (trên xuống dưới). Nếu `worker` cần `master` đã sẵn sàng trước (ví dụ để join cluster), hãy định nghĩa `master` **trước** trong mảng `NODES`/vòng lặp — đây chính là lý do ở BT-B, `master` luôn nằm ở vị trí đầu mảng.

### 4.2. Bài tập C — Thao tác chọn lọc trên cụm

**BT-C1:** Từ cụm 4 máy ở BT-B (đang chạy), destroy riêng `worker3`, xác nhận bằng `vagrant status` rằng chỉ còn 3 máy "running"/"not created".

**BT-C2:** Chạy `vagrant up` lại — quan sát Vagrant chỉ tạo lại đúng `worker3` bị xoá, không đụng 3 máy còn lại.

**BT-C3:** Dùng `vagrant global-status` để tìm ra ID cụm hiện tại, sau đó thử `vagrant destroy <id>` bằng ID đó thay vì tên máy (tính năng ít người biết — hữu ích khi đứng ở thư mục khác không phải thư mục chứa Vagrantfile).

<details>
<summary><b>💡 Lời giải BT-C</b></summary>

```bash
# BT-C1
vagrant destroy worker3 -f
vagrant status
# master   running
# worker1  running
# worker2  running
# worker3  not created

# BT-C2
vagrant up
# ==> master: Machine already provisioned... (bị bỏ qua vì đã chạy)
# ==> worker1: Machine already provisioned...
# ==> worker2: Machine already provisioned...
# Bringing machine 'worker3' up... (chỉ máy này thực sự được tạo mới)

# BT-C3
vagrant global-status
# id       name     provider state   directory
# a1b2c3d  master   libvirt  running /home/.../04-multi-machine
# ...

vagrant destroy a1b2c3d
# Có thể destroy bằng ID toàn cục, không cần đứng đúng thư mục Vagrantfile
```

**Điểm quan trọng rút ra:** Vagrant theo dõi trạng thái từng máy **độc lập** (lưu trong `.vagrant/machines/<tên>/`), nên các thao tác chọn lọc theo tên hoàn toàn an toàn — không sợ ảnh hưởng dây chuyền tới máy khác trong cùng cụm.

</details>

---

## 5. Dự án cuối bài — Deliverable

Thư mục `lab/04-multi-machine/` trong repo này phải có:

- `Vagrantfile` dùng **mảng `NODES`** (không phải số lượng cứng) để định nghĩa cụm: 1 control-plane + N worker (N ≥ 3), IP tự sinh theo pattern nhất quán.
- Provisioning rẽ nhánh theo `role` (control-plane vs worker in ra thông điệp khác nhau, hoặc cài package khác nhau).
- `README.md` ghi rõ: cách đổi số lượng worker (chỉ sửa 1 chỗ), cách destroy/up riêng 1 máy, IP range đang dùng.
- Xác nhận: `vagrant destroy -f && vagrant up` dựng lại **toàn bộ cụm từ số 0** thành công, mọi node ping được nhau qua private network.

> **Đây chính là nền tảng lab dùng xuyên suốt Unit 02 (Ansible) và Unit 03 (Kubespray)** — hãy đặt tên biến/cấu trúc gọn gàng ngay từ bây giờ vì bạn sẽ mở rộng thêm (không viết lại từ đầu) ở 2 unit tiếp theo.

---

## 6. Câu hỏi tự kiểm tra

> Viết câu trả lời bằng lời của bạn trước, sau đó mới mở phần gợi ý.

1. Vì sao `Vagrantfile` dùng được vòng lặp Ruby trong khi 1 file YAML (như `docker-compose.yml`) thì không?
2. Nếu bạn tăng `NUM_NODES` từ 3 lên 5 rồi `vagrant up`, những máy cũ (node1-3) có bị restart/mất dữ liệu không?
3. Thứ tự định nghĩa máy trong mảng `NODES`/vòng lặp có ảnh hưởng gì tới thứ tự `vagrant up`?
4. `vagrant destroy worker3 -f` và `vagrant destroy -f` (không chỉ tên) khác nhau ở điểm nào? Trường hợp nào bạn muốn dùng lệnh không chỉ tên?
5. Vì sao cấu trúc dữ liệu kiểu `NODES = [{name:, ip:, role:}, ...]` lại "gần giống" với inventory của Ansible mà bạn sẽ học ở Unit 02?

<details>
<summary><b>Gợi ý trả lời</b></summary>

**1.** YAML là **định dạng dữ liệu tĩnh** (data serialization) — không có khái niệm biến, vòng lặp, hàm; nó chỉ mô tả cấu trúc dữ liệu để chương trình khác (Docker Compose engine) đọc và diễn giải. `Vagrantfile` thực chất là **1 file mã nguồn Ruby thật sự** được Vagrant `load`/`eval` — nên mọi cú pháp Ruby hợp lệ (biến, vòng lặp, mảng, hàm, điều kiện) đều chạy được, vì bản chất nó sinh ra 1 cấu trúc cấu hình Ruby object rồi Vagrant đọc object đó.

**2.** Không. Vagrant lưu trạng thái từng máy độc lập trong `.vagrant/machines/<tên>/`. Khi bạn tăng N và chạy lại `vagrant up`, Vagrant duyệt qua từng máy trong vòng lặp: với `node1-3` đã tồn tại và đang chạy, nó chỉ in "Machine already provisioned" rồi bỏ qua (trừ khi bạn thêm `--provision`); chỉ `node4`, `node5` (chưa từng tồn tại) mới thực sự được tạo mới. Dữ liệu trong node1-3 hoàn toàn nguyên vẹn.

**3.** Có — Vagrant khởi động các máy **theo đúng thứ tự xuất hiện trong file** (tương đương thứ tự vòng lặp duyệt qua mảng). Nếu worker cần master đã init sẵn (ví dụ cluster join token), bạn phải đặt master lên đầu mảng `NODES` để nó luôn được `up` trước. Vagrant không tự dò dependency như Terraform — thứ tự hoàn toàn do bạn kiểm soát bằng cách sắp xếp mảng/vòng lặp.

**4.** `vagrant destroy worker3 -f` chỉ xoá đúng máy tên `worker3`, các máy khác trong Vagrantfile không bị ảnh hưởng. `vagrant destroy -f` (không chỉ tên) xoá **toàn bộ** máy được định nghĩa trong Vagrantfile hiện tại. Dùng lệnh không chỉ tên khi bạn muốn dọn sạch hoàn toàn cụm để dựng lại từ đầu (ví dụ trước khi thử nghiệm nguy hiểm, hoặc trước khi đổi hẳn cấu trúc `NODES`).

**5.** Ansible inventory về bản chất cũng là 1 danh sách host có gắn kèm **biến/nhóm** (ví dụ nhóm `control_plane` và `workers`, mỗi host có `ansible_host` = IP). Cấu trúc `NODES = [{name:, ip:, role:}, ...]` bạn viết ở đây chính là bạn đang tự tay mô phỏng lại ý tưởng "mô tả tập hợp máy bằng dữ liệu có cấu trúc, gắn role/group cho từng máy" — nền tảng tư duy y hệt khi bạn viết `inventory.ini`/`inventory.yml` cho Ansible ở Unit 02.

</details>

---

## 7. Tổng kết & bước tiếp theo

Bạn đã có:
- ✅ Hiểu và dùng được vòng lặp Ruby để sinh cụm N máy thay vì chép tay.
- ✅ Cấu trúc `NODES` dạng mảng Hash — mô tả cụm bất đối xứng (control-plane + worker) bằng dữ liệu.
- ✅ Thành thạo thao tác chọn lọc trên cụm nhiều máy (up/halt/destroy theo tên, theo ID toàn cục).
- ✅ 1 Vagrantfile tái sử dụng được — nền cho Unit 02 (Ansible) và Unit 03 (Kubespray).

**Bài tiếp theo:** U01-04 — Provisioning: shell, file và tích hợp Ansible.

---

## 8. Ghi điểm (tự chấm)

| Tiêu chí | Điểm |
|---|---|
| BT-A — vòng lặp sinh cụm, scale từ 3 lên 5 node đúng cách | 2 |
| BT-B — cụm bất đối xứng (master + worker), provisioning rẽ nhánh đúng | 3 |
| BT-C — thao tác chọn lọc (destroy/up theo tên, theo ID) thành thạo | 2 |
| Deliverable — `lab/04-multi-machine/` dùng mảng `NODES`, README đầy đủ | 2 |
| Trả lời đúng ≥4/5 câu hỏi tự kiểm tra | 1 |
| **Tổng** | **10** |

Đạt ≥8/10 → tick ☑ ở `PROGRESS.md` và sheet workbook. Dưới 8 → xem lại phần sai, sửa và làm lại trước khi sang bài 1.4.
