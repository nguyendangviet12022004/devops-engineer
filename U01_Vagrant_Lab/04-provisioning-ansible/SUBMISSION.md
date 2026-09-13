# Bài nộp — U01-04 Provisioning: shell, file và tích hợp Ansible

## Ngày bắt đầu / hoàn thành

- Bắt đầu:
- Hoàn thành:
- Số giờ thực tế:

## BT-A — Inline vs Path, có tham số

- `scripts/install-nginx.sh` — link:
- Log `vagrant up` xác nhận đúng thứ tự (inline trước, path sau):

```
(dán log ở đây)
```

- Test `curl http://localhost:8081` — kết quả:

```
(dán ở đây)
```

## BT-B — File provisioner + áp dụng bằng shell

- `files/motd-custom.txt` — link:
- MOTD hiển thị khi SSH vào (dán output):

```
(dán ở đây)
```

## BT-C — Chạy chọn lọc & `run: always`

- Log `vagrant provision --provision-with install-nginx` (chỉ 1 provisioner chạy):

```
(dán ở đây)
```

- Nội dung `/var/log/vagrant-boot.log` sau 2 lần `vagrant reload`:

```
(dán ở đây)
```

## BT-D — Ansible provisioner

- `ansible --version` trên host:

```
(dán ở đây)
```

- Log `vagrant up` lần 1 (PLAY RECAP, changed=?):

```
(dán ở đây)
```

- Log `vagrant provision` lần 2 (PLAY RECAP, changed=0?):

```
(dán ở đây)
```

- Nhận xét: shell provisioner có báo được "changed hay không" như Ansible không? Vì sao?

(trả lời)

## Deliverable — `lab/05-provisioning/`

- Đường dẫn: `U01_Vagrant_Lab/04-provisioning-ansible/lab/05-provisioning/`
- Số provisioner đặt tên:
- Có `run: always` chưa: (Có/Không)
- `ansible/playbook.yml` chạy `changed=0` ở lần 2: (Có/Không)

## Trả lời 5 câu hỏi tự kiểm tra

> Viết bằng lời của bạn — không copy từ LESSON.md.

**Câu 1: File provisioner copy thẳng vào `/etc/nginx/` được không? Vì sao?**

(trả lời)

**Câu 2: `vagrant up` và `vagrant reload` trên VM đã tồn tại có chạy lại provisioner không?**

(trả lời)

**Câu 3: `run: "always"` khác gì hành vi mặc định?**

(trả lời)

**Câu 4: Vì sao `useradd` chạy 2 lần bằng shell sẽ làm provisioning thất bại?**

(trả lời)

**Câu 5: Ansible local provisioner chạy `ansible-playbook` ở đâu?**

(trả lời)

## Tự đánh giá

- Mức độ tự tin (1-5):
- Điều còn chưa rõ:
