# devops-engineer

Repo thực hành cá nhân theo lộ trình **DevOps nâng cao** — 17 Unit · 139 bài học · ~872 giờ · 40 tuần.

Dành cho người đã có nền tảng DevOps (Docker, kubectl cơ bản, CI/CD, Linux cơ bản), muốn lên mức chuyên sâu: vận hành được hệ thống production thật, không chỉ chạy tutorial.

Roadmap đầy đủ (workbook Excel có tick tiến độ, checkpoint, tài nguyên tham khảo): xem file `Lo_Trinh_DevOps_Advanced_Workmap.xlsx` được chia sẻ riêng.

## Cấu trúc lộ trình

| Unit | Chủ đề | Tuần |
|---|---|---|
| U01 | Vagrant & Lab Infrastructure | 1-2 |
| U02 | Ansible | 3-4 |
| U03 | Kubespray Cluster | 5-6 |
| U04 | Kafka Cluster | 7-8 |
| U05 | K8s Resource & Scheduling | 9-10 |
| U06 | K8s Autoscaling | 11-12 |
| U07 | K8s Networking | 13-15 |
| U08 | K8s Storage | 16-17 |
| U09 | K8s Security | 18-19 |
| U10 | Helm & GitOps | 20-21 |
| U11 | Terraform chuyên sâu | 22-24 |
| U12 | AWS chuyên sâu | 25-28 |
| U13 | Linux chuyên sâu | 29-31 |
| U14 | Observability | 32-33 |
| U15 | CI/CD & Supply Chain | 34-35 |
| U16 | SRE & Platform Engineering | 36-38 |
| U17 | Capstone & Chứng chỉ | 39-40 |

## Cấu trúc thư mục mỗi bài học

```
UNN_Ten_Unit/
  README.md              — mục tiêu & danh sách bài học của unit
  NN-topic-slug/
    LESSON.md             — lý thuyết + bài tập + quiz + tiêu chí hoàn thành
    SUBMISSION.md          — bài nộp: điền câu trả lời & kết quả vào đây
    lab/                    — nơi đặt Vagrantfile / playbook / manifest thực hành
```

## Cách học

1. Đọc `LESSON.md` của bài tiếp theo.
2. Làm bài tập thực hành trong `lab/`.
3. Điền `SUBMISSION.md` — trả lời quiz bằng lời của bạn, dán kết quả thực tế.
4. Đối chiếu "Tiêu chí hoàn thành" trong `LESSON.md`.
5. Commit + push, tick ☑ ở sheet Unit tương ứng trong workbook Excel.

## Yêu cầu hạ tầng

- 16GB RAM khuyến nghị cho lab 5 node (K8s HA); có thể giảm còn 3 node nếu máy yếu — xem ghi chú trong `LESSON.md` của U01.
- Vagrant + provider (VirtualBox hoặc libvirt/KVM).
- Từ U11 (AWS): cần tài khoản AWS Free Tier + Budget alert.

## Nguyên tắc

- **Lab thật, không chỉ đọc lý thuyết.** Mỗi bài để lại artifact chạy được.
- **Đo được, không chỉ "cảm giác ổn".** Mọi deliverable đi kèm số liệu cụ thể.
- **Phá rồi sửa.** Chủ động gây sự cố trong lab rồi tự khắc phục.
