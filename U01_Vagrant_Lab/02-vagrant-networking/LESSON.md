# U01-02 — Mạng trong Vagrant: private network, port forwarding, IP tĩnh

> **Unit:** U01 — Vagrant & Lab Infrastructure
> **Tuần:** 2 · **Giờ dự kiến:** 4 · **Độ khó:** Dễ
> **Tài liệu tham khảo:** [Vagrant Docs — Networking](https://developer.hashicorp.com/vagrant/docs/networking)

---

## 1. Mục tiêu bài học

Sau bài này bạn phải:

1. Hiểu 3 loại network trong Vagrant: **forwarded_port**, **private_network**, **public_network**.
2. Biết cách **gán IP tĩnh** cho VM.
3. Có thể **truy cập VM từ máy khác** trong LAN.
4. Hiểu **port forwarding** hoạt động như thế nào.
5. Dựng lab gồm **2 máy ảo** liên lạc được với nhau qua private network.

---

## 2. Lý thuyết

### 2.1. 3 loại network trong Vagrant

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "generic/ubuntu2204"
  
  # 1️⃣ FORWARDED_PORT — map port từ host → VM
  config.vm.network "forwarded_port", guest: 80, host: 8080
  
  # 2️⃣ PRIVATE_NETWORK — VM có IP riêng (chỉ host + các VM khác)
  config.vm.network "private_network", ip: "192.168.50.10"
  
  # 3️⃣ PUBLIC_NETWORK — VM lấy IP từ router (giống máy thật)
  config.vm.network "public_network", bridge: "eth0"
end
```

### 2.2. Forwarded Port — chi tiết

Port forwarding là **iptables rule**, forward traffic từ port host → port VM.

```
Host: localhost:8080
   ↓ (iptables forward)
VM: localhost:80
```

**Lưu ý:**
- Chỉ hoạt động trên **localhost** — máy khác LAN không thể truy cập `host-ip:8080`
- Nếu muốn máy khác truy cập → dùng **private_network** hoặc **public_network**

### 2.3. Private Network — VM có IP cố định, LAN cục bộ

```ruby
config.vm.network "private_network", ip: "192.168.50.10"
```

**Điều gì xảy ra:**
- Vagrant tạo mạng ảo 192.168.50.0/24
- VM nhận IP cố định 192.168.50.10
- **Host + các VM khác** trong cùng provider **có thể truy cập**
- **Máy khác LAN thật** không thể (firewall của provider chặn)

**Lợi ích:**
- Các VM có thể ping được nhau
- Không phải forward từng port — truy cập trực tiếp qua IP
- Lý tưởng cho lab nhiều máy

### 2.4. Public Network — VM có IP thật

```ruby
config.vm.network "public_network", bridge: "eth0"
```

- VM kết nối **trực tiếp** tới NIC thật của host
- DHCP server LAN cấp IP thật cho VM
- VM trở thành **máy thật trong LAN**

---

## 3. Thực hành

### 3.1. Lab 1: Port forwarding + test từ host

Thư mục: `lab/02-port-forwarding/`

**Vagrantfile:**
```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "generic/ubuntu2204"
  config.vm.box_version = "4.3.12"
  config.vm.hostname = "web-server"
  config.vm.network "forwarded_port", guest: 80, host: 8080
  
  config.vm.provider "libvirt" do |lv|
    lv.memory = 2048
    lv.cpus = 2
  end
  
  config.vm.provision "shell", inline: <<-SHELL
    apt update
    apt install -y nginx
    systemctl start nginx
  SHELL
end
```

**Chạy:**
```bash
vagrant up
```

**Test từ host:**
```bash
curl http://localhost:8080
```

### 3.2. Lab 2: Private network + 2 VM

Thư mục: `lab/03-private-network/`

**Vagrantfile:**
```ruby
Vagrant.configure("2") do |config|
  config.vm.define "master" do |master|
    master.vm.box = "generic/ubuntu2204"
    master.vm.box_version = "4.3.12"
    master.vm.hostname = "master"
    master.vm.network "private_network", ip: "192.168.50.10"
    master.vm.provider "libvirt" do |lv|
      lv.memory = 2048
      lv.cpus = 2
    end
  end
  
  config.vm.define "worker" do |worker|
    worker.vm.box = "generic/ubuntu2204"
    worker.vm.box_version = "4.3.12"
    worker.vm.hostname = "worker"
    worker.vm.network "private_network", ip: "192.168.50.11"
    worker.vm.provider "libvirt" do |lv|
      lv.memory = 2048
      lv.cpus = 2
    end
  end
end
```

**Test liên lạc:**
```bash
vagrant ssh master
  ping -c 3 192.168.50.11   # → phải reply
  ssh vagrant@192.168.50.11  # → phải vào được
  exit
exit
```

---

## 4. Bài tập

**BT1:** Port forwarding + nginx
- Tạo Vagrantfile, cài nginx qua provisioning
- Test: `curl http://localhost:8080`
- Ghi lại kết quả

**BT2:** 2 VM + private network
- Dựng master (192.168.50.10) và worker (192.168.50.11)
- Từ master, ping worker và SSH tới worker
- Ghi lại output

**BT3:** Khám phá network
- SSH vào 1 VM, chạy:
  - `ip addr show` (các IP)
  - `ip route` (routing table)
  - `netstat -tlnp` (services listening)
- Ghi lại output

---

## 5. Câu hỏi tự kiểm tra

1. Port forwarding hoạt động ở tầng nào (hint: iptables)?
2. Vì sao private network chỉ cho host + các VM truy cập?
3. Để máy khác LAN truy cập VM, dùng loại network nào?
4. Làm sao để chỉ khởi động 1 trong 2 VM?
5. Khác biệt giữa hostname và IP address?

---

## 6. Deliverable

Thư mục `lab/` chứa ≥2 Vagrantfile hoạt động:
1. `lab/02-port-forwarding/` — port forwarding
2. `lab/03-private-network/` — 2 VM + private network

Mỗi folder: `Vagrantfile` + `README.md` (hướng dẫn test)

---

## 7. Ghi điểm

| Tiêu chí | Điểm |
|---|---|
| BT1 — port forwarding hoạt động | 2 |
| BT2 — 2 VM liên lạc được | 2 |
| BT3 — khám phá network, ghi output | 2 |
| Trả lời ≥4/5 câu hỏi | 2 |
| Vagrantfile đầu đủ, README rõ ràng | 2 |
| **Tổng** | **10** |

Đạt ≥8/10 → tick ☑ PROGRESS.md
