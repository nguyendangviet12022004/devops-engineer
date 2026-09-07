# Unit 01 — Vagrant & Lab Infrastructure

**Tuần:** 1-2 · **Số bài:** 7 · **Tổng giờ:** 33 · **Độ khó:** Dễ → TB

## Mục tiêu

Dựng được lab nhiều máy ảo bằng code, tái tạo lại trong 1 lệnh — nền móng để thực hành mọi thứ về sau (Ansible, Kubespray, Kafka, K8s...) mà không sợ phá hỏng máy thật.

## Vì sao bắt đầu ở đây

Đây là unit **dễ nhất** trong toàn bộ lộ trình, cố tình đặt đầu tiên để tạo đà và cho kết quả nhìn thấy được ngay trong buổi học đầu. Không có unit này thì không có lab để thực hành 16 unit còn lại.

## Danh sách bài học

| # | Bài | Giờ | Độ khó |
|---|---|---|---|
| 1.1 | Vagrant cơ bản: Vagrantfile, box, provider | 4 | Dễ |
| 1.2 | Mạng trong Vagrant: private network, port forwarding, IP tĩnh | 4 | Dễ |
| 1.3 | Multi-machine Vagrantfile: dựng cụm bằng vòng lặp | 5 | Dễ |
| 1.4 | Provisioning: shell, file và tích hợp Ansible | 5 | Dễ |
| 1.5 | Synced folder, snapshot và tối ưu tài nguyên | 4 | Dễ |
| 1.6 | Đóng gói box tuỳ chỉnh & giới thiệu Packer | 5 | TB |
| 1.7 | DỰ ÁN — Lab 5 node sẵn sàng cho Kubespray | 6 | TB |

## Điều kiện tiên quyết

Không — đây là bài đầu tiên của toàn bộ lộ trình.

## Sản phẩm cuối unit (Milestone M1 — Tuần 2)

Repo lab dựng được 5 máy ảo từ số 0 bằng 1 lệnh (`vagrant destroy -f && vagrant up`), có box tuỳ chỉnh, chạy trong < 20 phút, sẵn sàng để Unit 02 (Ansible) và Unit 03 (Kubespray) dùng làm nền.

## Nếu máy yếu (< 16GB RAM)

Giảm xuống 3 node (1 control-plane + 2 worker) thay vì 5. Vẫn học được ~90% nội dung của toàn lộ trình, chỉ mất phần kiểm chứng HA control-plane (tắt 1 trong 3 CP mà cụm vẫn sống) ở Unit 03. Xem ghi chú cụ thể ở cuối `01-vagrant-basics/LESSON.md`.
