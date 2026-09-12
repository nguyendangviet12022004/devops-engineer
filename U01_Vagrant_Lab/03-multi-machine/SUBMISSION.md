# Bài nộp — U01-03 Multi-machine Vagrantfile: dựng cụm bằng vòng lặp

## Ngày bắt đầu / hoàn thành

- Bắt đầu:
- Hoàn thành:
- Số giờ thực tế:

## BT-A — Sinh cụm bằng vòng lặp

- Link `Vagrantfile` ban đầu (3 node):
- Đổi `NUM_NODES` 3 → 5, log xác nhận chỉ tạo thêm node4/node5:

```
(dán log ở đây)
```

- Kết quả ping node1 → node2, node1 → node5:

```
(dán output ping ở đây)
```

## BT-B — Cụm bất đối xứng (1 master + N worker)

- Link `Vagrantfile` dùng mảng `NODES`:
- Log `vagrant up` xác nhận provisioning rẽ nhánh đúng theo role:

```
(dán log ở đây)
```

- Lệnh dùng để chỉ reload riêng `master`:

```
(lệnh ở đây)
```

## BT-C — Thao tác chọn lọc trên cụm

- Lệnh destroy riêng `worker3`:
- `vagrant status` sau khi destroy (dán output):

```
(dán ở đây)
```

- `vagrant up` lại — log xác nhận chỉ tạo lại worker3:

```
(dán ở đây)
```

- ID cụm lấy từ `vagrant global-status`:

## Deliverable — `lab/04-multi-machine/`

- Đường dẫn: `U01_Vagrant_Lab/03-multi-machine/lab/04-multi-machine/`
- Số worker mặc định:
- IP range đang dùng:
- `vagrant destroy -f && vagrant up` dựng lại từ 0 thành công: (Có/Không)
- Toàn bộ node ping được nhau: (Có/Không)

## Trả lời 5 câu hỏi tự kiểm tra

> Viết bằng lời của bạn — không copy từ LESSON.md.

**Câu 1: Vì sao Vagrantfile dùng được vòng lặp mà YAML thì không?**

(trả lời)

**Câu 2: Tăng NUM_NODES từ 3 lên 5 có làm mất dữ liệu node cũ không?**

(trả lời)

**Câu 3: Thứ tự định nghĩa trong mảng NODES ảnh hưởng gì tới thứ tự `vagrant up`?**

(trả lời)

**Câu 4: `vagrant destroy worker3 -f` khác `vagrant destroy -f` thế nào?**

(trả lời)

**Câu 5: Vì sao cấu trúc NODES giống inventory của Ansible?**

(trả lời)

## Tự đánh giá

- Mức độ tự tin (1-5):
- Điều còn chưa rõ:
