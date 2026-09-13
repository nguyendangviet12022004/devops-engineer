# U01-04 — Provisioning: shell, file và tích hợp Ansible

> **Unit:** U01 — Vagrant & Lab Infrastructure
> **Tuần:** 2 · **Giờ dự kiến:** 5 · **Độ khó:** Dễ
> **Tài liệu tham khảo:** [Vagrant Docs — Provisioning](https://developer.hashicorp.com/vagrant/docs/provisioning) · [Shell Provisioner](https://developer.hashicorp.com/vagrant/docs/provisioning/shell) · [Ansible Provisioner](https://developer.hashicorp.com/vagrant/docs/provisioning/ansible)

---

## 1. Mục tiêu bài học

Sau bài này bạn phải:

1. Dùng thành thạo **shell provisioner** ở cả 2 dạng: `inline` và `path` (file script riêng).
2. Dùng được **file provisioner** để copy file/thư mục từ host vào VM.
3. Biết cách chạy **nhiều provisioner theo thứ tự**, và chạy chọn lọc bằng `--provision-with`.
4. Hiểu vì sao **shell script tự viết tay thường không idempotent**, và Ansible giải quyết vấn đề đó thế nào.
5. Cấu hình được **Ansible provisioner** (`local`) trong Vagrantfile — cầu nối trực tiếp sang Unit 02.

---

## 2. Shell Provisioner — 2 cách viết

### 2.1. Lý thuyết

Có 2 cách khai báo shell provisioner:

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "generic/ubuntu2204"

  # Cách 1: INLINE — script nằm ngay trong Vagrantfile
  config.vm.provision "shell", inline: <<-SHELL
    apt update
    apt install -y curl git
  SHELL

  # Cách 2: PATH — script nằm ở file riêng
  config.vm.provision "shell", path: "scripts/setup.sh"
end
```

**So sánh:**

| | `inline` | `path` |
|---|---|---|
| Script nằm ở đâu | Ngay trong Vagrantfile (heredoc) | File `.sh` riêng, Vagrant copy vào VM rồi chạy |
| Phù hợp khi | Script ngắn (vài dòng) | Script dài, phức tạp, muốn test riêng bằng `bash setup.sh` trước |
| Version control | Nằm chung 1 file, khó review diff dài | Tách file riêng, diff rõ ràng, tái dùng được ở nơi khác |
| Truyền tham số | `env:` option | `args:` option (giống `$1 $2` khi gọi script) |

**Khai báo có tham số:**

```ruby
config.vm.provision "shell", path: "scripts/setup.sh", args: ["ubuntu", "22.04"]
```

Trong `scripts/setup.sh`, `$1` = `"ubuntu"`, `$2` = `"22.04"` — giống hệt cách truyền argument cho bất kỳ shell script nào.

**Quyền chạy — `privileged`:**

```ruby
config.vm.provision "shell", inline: "whoami", privileged: false
```

Mặc định `privileged: true` — script chạy bằng `root` (Vagrant tự thêm `sudo`). Đặt `privileged: false` khi bạn muốn chạy bằng đúng user SSH mặc định (thường là `vagrant`), ví dụ để test quyền hạn thông thường.

### 2.2. Bài tập A — Inline vs Path, có tham số

**BT-A1:** Viết Vagrantfile tại `lab/05-provisioning/` dùng **inline** để cài `curl` và `git`.

**BT-A2:** Tách phần cài `nginx` ra file riêng `scripts/install-nginx.sh`, dùng `path:` để gọi. Script phải nhận 1 tham số là port nginx sẽ lắng nghe (mặc định port 80), dùng `sed` để sửa file cấu hình nginx theo tham số đó.

**BT-A3:** Chạy `vagrant up`, xác nhận cả 2 provisioner chạy đúng thứ tự khai báo (inline trước, path sau).

### 💡 Lời giải BT-A

**`Vagrantfile`:**

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "generic/ubuntu2204"
  config.vm.box_version = "4.3.12"
  config.vm.hostname = "provision-lab"
  config.vm.network "forwarded_port", guest: 8081, host: 8081

  config.vm.provider "libvirt" do |lv|
    lv.memory = 1024
    lv.cpus = 1
  end

  # BT-A1: inline
  config.vm.provision "shell", inline: <<-SHELL
    echo ">> [1] Cài curl + git (inline)"
    apt update
    apt install -y curl git
  SHELL

  # BT-A2: path + args
  config.vm.provision "shell", path: "scripts/install-nginx.sh", args: ["8081"]
end
```

**`scripts/install-nginx.sh`:**

```bash
#!/bin/bash
set -e
PORT=$1

echo ">> [2] Cài nginx, đổi port sang ${PORT} (path + args)"
apt update
apt install -y nginx

# Đổi "listen 80" thành "listen $PORT" trong default site
sed -i "s/listen 80 default_server;/listen ${PORT} default_server;/" /etc/nginx/sites-available/default
systemctl restart nginx
```

**Chạy & xác nhận thứ tự:**
```bash
vagrant up
# ==> default: >> [1] Cài curl + git (inline)
#     ... (apt output) ...
# ==> default: >> [2] Cài nginx, đổi port sang 8081 (path + args)
#     ... (apt output) ...
```

**Test:**
```bash
curl http://localhost:8081
# → thấy nginx welcome page, xác nhận nginx nghe đúng port 8081
```

**Điểm cần chú ý:** thứ tự log in ra đúng bằng thứ tự khai báo `config.vm.provision` trong Vagrantfile — không phải ngẫu nhiên.

---

## 3. File Provisioner — copy dữ liệu từ host vào VM

### 3.1. Lý thuyết

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "generic/ubuntu2204"

  # Copy 1 file
  config.vm.provision "file", source: "files/app.conf", destination: "/tmp/app.conf"

  # Copy 1 thư mục
  config.vm.provision "file", source: "files/configs/", destination: "/tmp/configs"
end
```

**Điểm quan trọng cần nhớ:**

- File provisioner **luôn copy vào home directory của user SSH trước** (thường `/home/vagrant/`), **không copy thẳng vào thư mục hệ thống** (như `/etc/`) — vì user SSH không có quyền ghi ở đó. Muốn đưa vào `/etc/nginx/`, bạn phải: copy vào `/tmp/` bằng file provisioner, rồi dùng **shell provisioner chạy sau đó** với `sudo mv`/`sudo cp` để chuyển vào đúng chỗ.
- Thứ tự thường dùng: **file provisioner trước** (đưa dữ liệu vào) → **shell provisioner sau** (dùng `sudo` để đặt đúng chỗ + áp dụng).

```ruby
config.vm.provision "file", source: "files/nginx-custom.conf", destination: "/tmp/nginx-custom.conf"

config.vm.provision "shell", inline: <<-SHELL
  sudo mv /tmp/nginx-custom.conf /etc/nginx/sites-available/custom.conf
  sudo ln -sf /etc/nginx/sites-available/custom.conf /etc/nginx/sites-enabled/
  sudo systemctl reload nginx
SHELL
```

### 3.2. Bài tập B — File provisioner + áp dụng bằng shell

**BT-B1:** Tạo file `files/motd-custom.txt` trên host với nội dung tuỳ ý (ví dụ ASCII art hoặc dòng chào mừng). Dùng file provisioner copy vào `/tmp/` trong VM.

**BT-B2:** Thêm shell provisioner chạy **sau** để `sudo mv` file đó vào `/etc/motd` (file hiển thị mỗi khi SSH vào máy Linux).

**BT-B3:** `vagrant ssh` vào VM, xác nhận thấy nội dung MOTD tuỳ chỉnh ngay khi vừa đăng nhập.

### 💡 Lời giải BT-B

**`files/motd-custom.txt`:**
```
========================================
  Lab Provisioning — U01-04
  Máy này được dựng tự động bằng Vagrant
========================================
```

**Thêm vào `Vagrantfile`:**

```ruby
config.vm.provision "file", source: "files/motd-custom.txt", destination: "/tmp/motd-custom.txt"

config.vm.provision "shell", inline: <<-SHELL
  echo ">> [3] Áp dụng MOTD tuỳ chỉnh"
  sudo mv /tmp/motd-custom.txt /etc/motd
SHELL
```

**Chạy & kiểm tra:**
```bash
vagrant up --provision   # ép chạy lại toàn bộ provisioner nếu VM đã tồn tại từ trước

vagrant ssh
# ========================================
#   Lab Provisioning — U01-04
#   Máy này được dựng tự động bằng Vagrant
# ========================================
# vagrant@provision-lab:~$
```

**Vì sao phải copy vào `/tmp/` trước rồi mới `mv`?** Nếu bạn khai báo thẳng `destination: "/etc/motd"` ở file provisioner, Vagrant sẽ báo lỗi permission denied — vì file provisioner chạy bằng user SSH thường (không phải root), và `/etc/` chỉ root mới ghi được. Đây là lý do bắt buộc phải qua bước trung gian `/tmp/` + shell provisioner có `sudo`.

---

## 4. Nhiều provisioner: thứ tự và chạy chọn lọc

### 4.1. Lý thuyết

Khi Vagrantfile có nhiều `config.vm.provision`, mặc định **`vagrant up` chỉ chạy chúng ở lần tạo VM đầu tiên**. Muốn chạy lại có kiểm soát:

```bash
vagrant up --provision              # ép chạy lại TOÀN BỘ provisioner
vagrant provision                  # chạy lại toàn bộ, không restart VM

vagrant provision --provision-with shell    # chỉ chạy provisioner loại "shell"
```

Muốn chọn đúng 1 provisioner cụ thể (khi có nhiều provisioner cùng loại `shell`), đặt tên cho nó bằng cách truyền **type là string đầu tiên**, rồi dùng đúng tên đó:

```ruby
config.vm.provision "install-tools", type: "shell", inline: "apt install -y curl git"
config.vm.provision "install-nginx", type: "shell", path: "scripts/install-nginx.sh"
```

```bash
vagrant provision --provision-with install-nginx   # chỉ chạy đúng block này
```

**Provisioner chạy "luôn luôn"** (kể cả khi VM đã tồn tại, không cần `--provision`):

```ruby
config.vm.provision "shell", inline: "date >> /var/log/last-boot.log", run: "always"
```

`run: "always"` hữu ích cho các tác vụ cần chạy **mỗi lần `vagrant up`** (kể cả khi máy chỉ đang `halt` rồi bật lại), khác với hành vi mặc định "chỉ chạy 1 lần lúc tạo mới".

### 4.2. Bài tập C — Chạy chọn lọc & `run: always`

**BT-C1:** Đặt tên cho 2 provisioner shell ở BT-A (`install-tools` và `install-nginx`). Chạy `vagrant provision --provision-with install-nginx`, xác nhận **chỉ** log của nginx xuất hiện, không thấy log cài `curl git`.

**BT-C2:** Thêm 1 provisioner mới có `run: "always"` để ghi thời điểm boot vào `/var/log/vagrant-boot.log`. `vagrant reload` 2 lần, xác nhận file log có 2 dòng (mỗi lần reload thêm 1 dòng) dù các provisioner khác không chạy lại.

### 💡 Lời giải BT-C

**Vagrantfile (phần provision, đặt tên đầy đủ):**

```ruby
config.vm.provision "install-tools", type: "shell", inline: <<-SHELL
  echo ">> [1] Cài curl + git (inline)"
  apt update
  apt install -y curl git
SHELL

config.vm.provision "install-nginx", type: "shell", path: "scripts/install-nginx.sh", args: ["8081"]

config.vm.provision "log-boot", type: "shell", run: "always", inline: <<-SHELL
  echo "Booted at: $(date)" | sudo tee -a /var/log/vagrant-boot.log
SHELL
```

**BT-C1 — chạy chọn lọc:**
```bash
vagrant provision --provision-with install-nginx
# ==> default: Running provisioner: install-nginx (shell)...
#     default: >> [2] Cài nginx, đổi port sang 8081 (path + args)
# (KHÔNG thấy log "install-tools" — đúng như mong đợi)
```

**BT-C2 — `run: always`:**
```bash
vagrant reload
vagrant reload
vagrant ssh -c "cat /var/log/vagrant-boot.log"
# Booted at: Sun Sep 13 08:00:01 UTC 2026
# Booted at: Sun Sep 13 08:02:15 UTC 2026
```

Cả 2 dòng xuất hiện dù `install-tools` và `install-nginx` **không** chạy lại ở 2 lần reload này — chỉ `log-boot` (có `run: "always"`) chạy mỗi lần.

---

## 5. Vì sao cần Ansible: giới hạn của shell provisioner

### 5.1. Lý thuyết

Shell script tự viết tay có 1 nhược điểm cốt lõi: **không tự nhiên idempotent** (chạy lại nhiều lần cho cùng 1 kết quả, không lỗi, không làm trùng).

Ví dụ shell script **KHÔNG idempotent**:

```bash
# Mỗi lần chạy lại, dòng này được append thêm 1 lần nữa vào file
echo "127.0.0.1 myapp.local" >> /etc/hosts

# useradd sẽ LỖI ở lần chạy thứ 2 vì user đã tồn tại
useradd appuser
```

Chạy script này 3 lần → `/etc/hosts` có 3 dòng giống hệt nhau, và lệnh `useradd` ở lần 2-3 sẽ **exit với lỗi**, làm cả provisioner dừng giữa chừng (Vagrant coi exit code khác 0 là provisioning thất bại).

Muốn script shell idempotent, bạn phải **tự tay** thêm điều kiện kiểm tra cho từng lệnh:

```bash
grep -qxF "127.0.0.1 myapp.local" /etc/hosts || echo "127.0.0.1 myapp.local" >> /etc/hosts
id appuser &>/dev/null || useradd appuser
```

Việc này **có làm được**, nhưng càng script dài càng nhiều lệnh cần bọc điều kiện thủ công → dễ quên, dễ sai, code rối. **Ansible module được thiết kế idempotent ngay từ đầu** — mỗi module (như `user`, `lineinfile`, `apt`) tự kiểm tra trạng thái hiện tại trước khi hành động, không cần bạn tự viết điều kiện.

### 5.2. Ansible Provisioner trong Vagrantfile

Vagrant hỗ trợ gọi Ansible trực tiếp làm provisioner — không cần dựng máy Ansible Control Node riêng, Vagrant tự chạy `ansible-playbook` từ chính máy host của bạn nhắm vào VM:

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "generic/ubuntu2204"
  config.vm.box_version = "4.3.12"
  config.vm.hostname = "ansible-lab"

  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "playbook.yml"
  end
end
```

**`playbook.yml` tối thiểu (tương đương BT-A nhưng viết bằng Ansible):**

```yaml
---
- hosts: all
  become: true
  tasks:
    - name: Cài curl và git
      apt:
        name:
          - curl
          - git
        state: present
        update_cache: true

    - name: Cài nginx
      apt:
        name: nginx
        state: present
```

**Đây gọi là "Ansible local provisioner"** — `ansible-playbook` chạy trên **host** (máy bạn), kết nối SSH tới VM để thực thi. Khác với "Ansible remote/pull provisioner" (`ansible_local`) là Ansible được cài **bên trong** VM và tự chạy nhắm vào chính nó — dùng khi host không có sẵn Ansible (ví dụ CI runner Windows).

> **Yêu cầu:** máy host của bạn phải cài `ansible` (`sudo apt install ansible` hoặc `pip install ansible`) thì `config.vm.provision "ansible"` mới chạy được — Vagrant chỉ gọi ra `ansible-playbook` có sẵn trên host, không tự cài giúp bạn.

### 5.3. Bài tập D — Chuyển từ shell sang Ansible provisioner

**BT-D1:** Cài Ansible trên máy host (`sudo apt install -y ansible` hoặc `pip install ansible`), xác nhận bằng `ansible --version`.

**BT-D2:** Viết lại toàn bộ provisioning ở BT-A (cài `curl`, `git`, `nginx`) bằng 1 playbook Ansible duy nhất, dùng `config.vm.provision "ansible"` thay cho shell provisioner.

**BT-D3:** Chạy `vagrant provision` (không destroy VM) **2 lần liên tiếp**, xác nhận lần thứ 2 Ansible báo `changed=0` cho mọi task — đây chính là bằng chứng trực quan cho tính idempotent mà shell script không có sẵn.

### 💡 Lời giải BT-D

**Cài Ansible trên host:**
```bash
sudo apt update
sudo apt install -y ansible
ansible --version
# ansible [core 2.16.x]
```

**`Vagrantfile`:**
```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "generic/ubuntu2204"
  config.vm.box_version = "4.3.12"
  config.vm.hostname = "ansible-lab"
  config.vm.network "forwarded_port", guest: 80, host: 8082

  config.vm.provider "libvirt" do |lv|
    lv.memory = 1024
    lv.cpus = 1
  end

  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "playbook.yml"
  end
end
```

**`playbook.yml`:**
```yaml
---
- hosts: all
  become: true
  tasks:
    - name: Cài curl và git
      apt:
        name: ["curl", "git"]
        state: present
        update_cache: true

    - name: Cài nginx
      apt:
        name: nginx
        state: present

    - name: Đảm bảo nginx đang chạy
      service:
        name: nginx
        state: started
        enabled: true
```

**Chạy lần 1:**
```bash
vagrant up
# PLAY [all] *********************************************************
# TASK [Cài curl và git] ***********************************************
# changed: [default]
# TASK [Cài nginx] ******************************************************
# changed: [default]
# TASK [Đảm bảo nginx đang chạy] ****************************************
# changed: [default]
# PLAY RECAP *************************************************************
# default: ok=3  changed=3  unreachable=0  failed=0
```

**Chạy lần 2 (không đổi gì):**
```bash
vagrant provision
# TASK [Cài curl và git] ***********************************************
# ok: [default]
# TASK [Cài nginx] ******************************************************
# ok: [default]
# TASK [Đảm bảo nginx đang chạy] ****************************************
# ok: [default]
# PLAY RECAP *************************************************************
# default: ok=3  changed=0  unreachable=0  failed=0
```

**So sánh trực tiếp với shell provisioner (BT-A):** nếu bạn chạy lại script `install-nginx.sh` bằng `vagrant provision --provision-with install-nginx` 2 lần, log **in ra y hệt** cả 2 lần (apt install, sed, restart) — không có cách nào phân biệt "lần này có thay đổi gì không" chỉ bằng cách đọc output, vì bash không tự báo cáo trạng thái thay đổi. Ansible thì có — `changed=0` ở lần 2 là bằng chứng rõ ràng, đo được, không phải "cảm giác chạy ổn".

---

## 6. Dự án cuối bài — Deliverable

Thư mục `lab/05-provisioning/` trong repo này phải có:

- `Vagrantfile` với **ít nhất 3 provisioner được đặt tên**: 1 shell `inline`, 1 shell `path` (script riêng trong `scripts/`), 1 file provisioner (copy file từ `files/`).
- 1 provisioner có `run: "always"` ghi log mỗi lần `vagrant reload`.
- Thư mục `ansible/` chứa `playbook.yml` — bản Ansible tương đương với phần shell (chọn 1 Vagrantfile riêng `lab/05-provisioning/Vagrantfile.ansible` hoặc dùng biến để switch, tuỳ bạn), chạy `vagrant provision` 2 lần liên tiếp cho `changed=0` ở lần 2.
- `README.md` ghi rõ: cách chạy chọn lọc từng provisioner, và bảng so sánh ngắn "khi nào dùng shell, khi nào dùng Ansible" theo trải nghiệm thực tế bạn vừa làm.

---

## 7. Câu hỏi tự kiểm tra

> Viết câu trả lời bằng lời của bạn trước khi đọc gợi ý bên dưới cùng.

1. File provisioner copy thẳng vào `/etc/nginx/` được không? Vì sao, và cách khắc phục là gì?
2. `vagrant up` (không có `--provision`) trên 1 VM đã tồn tại có chạy provisioner không? Còn `vagrant reload`?
3. `run: "always"` khác gì với hành vi mặc định của provisioner?
4. Vì sao `useradd appuser` chạy 2 lần bằng shell provisioner sẽ làm cả quá trình provisioning thất bại?
5. Ansible local provisioner (`config.vm.provision "ansible"`) chạy `ansible-playbook` ở đâu — trên host hay trong VM?

## Gợi ý trả lời

**1.** Không được trực tiếp — file provisioner copy file vào home directory của user SSH (thường `/home/vagrant/`) vì user đó không có quyền ghi vào `/etc/`. Khắc phục: copy vào `/tmp/` trước, sau đó dùng shell provisioner (`privileged: true` mặc định, tức chạy bằng root) để `sudo mv`/`sudo cp` file vào đúng vị trí hệ thống.

**2.** `vagrant up` trên VM đã tồn tại (đang chạy hoặc đã `halt`) **không** tự chạy lại provisioner — Vagrant coi VM đã ở đúng trạng thái. `vagrant reload` (tắt rồi bật lại) **cũng không** tự chạy provisioner theo mặc định — trừ khi bạn dùng `vagrant reload --provision`, hoặc provisioner đó có khai báo `run: "always"` (lúc đó luôn chạy bất kể `reload` có cờ `--provision` hay không).

**3.** Mặc định, mọi provisioner chỉ chạy **đúng 1 lần** — ở lần `vagrant up` đầu tiên tạo VM. Muốn chạy lại phải chủ động ép bằng `--provision`. `run: "always"` đổi hành vi: provisioner đó chạy **mỗi lần** VM khởi động lên (kể cả `vagrant up` khi máy đang `halt`, hay `vagrant reload`), không cần cờ `--provision`.

**4.** Vì `useradd` trả về **exit code khác 0** khi user đã tồn tại (báo lỗi ra `stderr`: "user already exists"). Vagrant coi bất kỳ lệnh nào trong shell provisioner exit khác 0 là **toàn bộ provisioner thất bại** — nó dừng ngay tại dòng đó, các lệnh phía sau trong cùng script **không được chạy tiếp**, và `vagrant up` báo lỗi đỏ dừng luôn.

**5.** Chạy trên **host** — máy bạn (nơi gõ lệnh `vagrant up`) phải có sẵn `ansible-playbook`. Vagrant gọi lệnh đó từ host, `ansible-playbook` dùng SSH (thông qua thông tin Vagrant tự tạo — inventory tạm) để kết nối và thực thi task bên trong VM, tương tự cách bạn tự gõ `ansible-playbook -i <inventory> playbook.yml` nhắm vào VM đó.

---

## 8. Tổng kết & bước tiếp theo

Bạn đã có:
- ✅ Thành thạo shell provisioner (`inline`, `path`, `args`, `privileged`) và file provisioner.
- ✅ Biết chạy chọn lọc provisioner bằng tên, và dùng `run: "always"` cho tác vụ lặp lại mỗi lần boot.
- ✅ Hiểu rõ **vì sao** Ansible tồn tại — không phải "vì mọi người dùng nó" mà vì **idempotency là thuộc tính có sẵn của module**, không phải thứ bạn tự viết tay bằng `if`/`grep -q`.
- ✅ Đã chạy được Ansible provisioner ngay trong Vagrantfile — sẵn sàng cho Unit 02, nơi bạn viết Ansible role đầy đủ (không chỉ 1 playbook phẳng).

**Bài tiếp theo:** U01-05 — Synced folder, snapshot và tối ưu tài nguyên.

---

## 9. Ghi điểm (tự chấm)

| Tiêu chí | Điểm |
|---|---|
| BT-A — inline + path provisioner, đúng thứ tự, có tham số | 2 |
| BT-B — file provisioner + shell áp dụng vào `/etc/`, đúng cơ chế 2 bước | 2 |
| BT-C — chạy chọn lọc theo tên, `run: always` hoạt động đúng | 2 |
| BT-D — Ansible provisioner, chứng minh được `changed=0` ở lần chạy 2 | 3 |
| Trả lời đúng ≥4/5 câu hỏi tự kiểm tra | 1 |
| **Tổng** | **10** |

Đạt ≥8/10 → tick ☑ ở `PROGRESS.md` và sheet workbook. Dưới 8 → xem lại phần sai, sửa và làm lại trước khi sang bài 1.5.
