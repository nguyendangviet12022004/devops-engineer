# Bài nộp — U01-01 Vagrant cơ bản: Vagrantfile, box, provider

## Ngày bắt đầu / hoàn thành

- Bắt đầu:
- Hoàn thành:
- Số giờ thực tế đã bỏ ra:

## Provider đã chọn

- Provider: (VirtualBox / libvirt)
- Lý do chọn:

## BT1 — Tính phù du của VM

- Đã cài nginx, `destroy` rồi `up` lại, nginx còn hay mất?
- Quan sát của bạn:

## BT2 — So sánh thời gian `vagrant up`

| Provider | Thời gian `vagrant up` (giây) |
|---|---|
| VirtualBox | |
| libvirt | |

Ghi chú (nếu chỉ test được 1 provider, giải thích vì sao):

## BT3 — So sánh 3 box

| Box | User SSH mặc định | Package manager | Sudo không cần password? |
|---|---|---|---|
| generic/ubuntu2204 | | | |
| generic/rocky9 | | | |
| generic/debian12 | | | |

## Deliverable — `lab/01-basic/`

- Link/đường dẫn: `U01_Vagrant_Lab/01-vagrant-basics/lab/01-basic/`
- `vagrant up` chạy sạch từ máy trắng: (Có/Không)
- SSH vào được: (Có/Không)

## Trả lời 5 câu hỏi tự kiểm tra

> Viết bằng lời của bạn — không copy từ LESSON.md. Đây là phần quan trọng nhất để xác nhận bạn thực sự hiểu.

**Câu 1: Box khác image Docker ở điểm nào?**

(trả lời)

**Câu 2: `halt` vs `suspend` vs `destroy`?**

(trả lời)

**Câu 3: Vagrantfile nên commit vào git không, còn `.vagrant/`?**

(trả lời)

**Câu 4: Vì sao cần khoá phiên bản box?**

(trả lời)

**Câu 5: `vagrant up` lần 2 có chạy lại provisioner không?**

(trả lời)

## Tự đánh giá

- Mức độ tự tin (1-5):
- Điều còn chưa rõ:
