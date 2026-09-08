# Bài nộp — U01-02 Mạng trong Vagrant

## Ngày bắt đầu / hoàn thành

- Bắt đầu:
- Hoàn thành:
- Số giờ thực tế:

## BT1 — Port forwarding + nginx

### Vagrantfile
- Đặt ở: `lab/02-port-forwarding/Vagrantfile`
- Config: port forwarding 8080 → 80, cài nginx

### Test từ host
```bash
curl http://localhost:8080
```

Kết quả:
```
(dán output ở đây)
```

Hoạt động? (Có/Không):

---

## BT2 — 2 VM + private network

### Vagrantfile
- Đặt ở: `lab/03-private-network/Vagrantfile`
- Master: 192.168.50.10
- Worker: 192.168.50.11

### Test: ping từ master tới worker
```bash
vagrant ssh master
  ping -c 3 192.168.50.11
```

Output:
```
(dán ở đây)
```

### Test: SSH từ master tới worker
```bash
vagrant ssh master
  ssh vagrant@192.168.50.11
    hostname
    exit
  exit
```

Output:
```
(dán ở đây)
```

---

## BT3 — Khám phá network

SSH vào 1 VM, chạy các lệnh sau:

### `ip addr show`
```
(output ở đây)
```

### `ip route`
```
(output ở đây)
```

### `netstat -tlnp` (hoặc `ss -tlnp`)
```
(output ở đây)
```

### Giải thích
- Các IP nào thấy?
- Có những route gì?
- Có services nào listening?

---

## Trả lời 5 câu hỏi tự kiểm tra

1. Port forwarding hoạt động ở tầng nào?
   > (trả lời)

2. Vì sao private network chỉ cho host + VM khác, mà không cho máy LAN?
   > (trả lời)

3. Để máy khác LAN truy cập VM, dùng loại network nào?
   > (trả lời)

4. Làm sao để chỉ khởi động 1 trong 2 VM?
   > (trả lời)

5. Khác biệt giữa hostname và IP address?
   > (trả lời)

---

## Tự đánh giá

- Mức độ tự tin (1-5):
- Điều chưa rõ:
