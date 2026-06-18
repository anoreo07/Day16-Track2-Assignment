# Báo Cáo Lab 16 — Cloud AI Environment Setup (AWS)

**Phương án:** Dự phòng CPU + LightGBM (Phần 7)

**Lý do:** Tài khoản AWS Free Tier không xin được quota GPU, chuyển sang CPU instance.

---

## 1. Kiến trúc triển khai

- **VPC:** `10.0.0.0/16` với 2 Public Subnet + 2 Private Subnet
- **Bastion Host:** `t3.micro` (Public IP: `32.196.231.47`)
- **CPU Node:** `t3.micro` (Private IP: `10.0.10.249`)
- **NAT Gateway:** Cho phép Private Subnet truy cập internet
- **ALB:** Application Load Balancer (HTTP → port 8000)
- **Security Groups:** Bastion SG, GPU Node SG, ALB SG
- **IAM Role:** `ai-inference-role` + Instance Profile

## 2. Benchmark LightGBM — Credit Card Fraud Detection

Dataset: **Credit Card Fraud Detection** (284,807 giao dịch, 30 features)

### Kết quả

| Metric | Giá trị |
|---|---|
| Load data | 14.14s |
| Train time | 3.81s |
| Best iteration | 1 |
| AUC-ROC | 0.9224 |
| Accuracy | 0.9991 |
| F1-Score | 0.7573 |
| Precision | 0.7222 |
| Recall | 0.7959 |
| Inference latency (1 row) | 0.0128s |
| Inference throughput (1000 rows) | 0.0128s |

<!-- 2. Chụp ảnh terminal output, đặt tên screenshot_terminal.png, xoá dòng comment bên dưới để hiển thị -->
<!-- ![Terminal Output](screenshot_terminal.png) -->

## 3. File benchmark_result.json

```json
{
  "load_time_s": 14.14,
  "train_time_s": 3.81,
  "best_iteration": 1,
  "auc_roc": 0.9224,
  "accuracy": 0.9991,
  "f1_score": 0.7573,
  "precision": 0.7222,
  "recall": 0.7959,
  "inference_latency_1row_s": 0.012844,
  "inference_throughput_1000rows_s": 0.0128
}
```

## 4. AWS Billing — Giải thích $0

<!-- 3. Chụp ảnh AWS Billing sau 1 giờ, đặt tên screenshot_billing.png, xoá dòng comment bên dưới để hiển thị -->
<!-- ![AWS Billing](screenshot_billing.png) -->

**Tại sao billing = $0 dù đang có tài nguyên chạy?**

- **EC2 (2× t3.micro):** AWS Free Tier cấp 750h/tháng cho `t3.micro`/`t2.micro`. Hai instance chạy đồng thời = ~1460h/tháng, nhưng billing có **độ trễ (delay) vài giờ** mới cập nhật. Khi đủ 750h (khoảng nửa tháng), phí EC2 sẽ bắt đầu xuất hiện.
- **NAT Gateway (~$0.045/h) và ALB (~$0.008/h):** **Không thuộc Free Tier**. Phí sẽ hiện sau 1-2 giờ khi AWS hoàn tất chu kỳ billing (thường cập nhật muộn 2-6h).
- **Free Tier 12 tháng:** Tài khoản mới được 750h EC2/month + 5GB S3 + 750h RDS… trong 12 tháng. Các dịch vụ ngoài Free Tier (NAT, ALB) vẫn tính phí dù billing chưa hiện ngay.
- **Kết luận:** Billing $0 là tạm thời do AWS chưa kịp cập nhật. Sau 2-6 giờ, NAT Gateway và ALB sẽ xuất hiện trong billing. **Cần chạy `terraform destroy` ngay sau khi nộp bài để tránh phát sinh chi phí.**

## 5. Mã nguồn Terraform

Thư mục `terraform/` đã chỉnh sửa:
- Đổi instance type GPU node từ `g4dn.xlarge` → `t3.micro`
- Đổi AMI Bastion sang hardcoded `ami-03d84abcde942cf8c`
- Giữ lại Deep Learning AMI cho CPU node (`data.aws_ami.deep_learning`)

## 6. Báo cáo ngắn

LightGBM training trên dataset Credit Card Fraud Detection hoàn tất trong **3.81 giây** với **AUC-ROC = 0.9224**, cho thấy mô hình có khả năng phân loại tốt mặc dù dataset mất cân bằng nghiêm trọng (chỉ ~0.17% giao dịch gian lận). F1-Score đạt **0.7573** với Precision 0.7222 và Recall 0.7959 — cân bằng tốt giữa việc phát hiện fraud và tránh false positive.

Thời gian load data **14.14s** và inference throughput 1000 rows chỉ **0.0128s**, chứng tỏ instance `t3.micro` đáp ứng tốt cho bài toán CPU ML với dataset kích thước trung bình.

Lý do phải dùng CPU thay GPU: Tài khoản AWS Free Tier mới bị khóa quota GPU (Running On-Demand G and VT instances = 0 vCPU) và yêu cầu tăng quota không được duyệt kịp trong thời gian lab. Phương án CPU với `t3.micro` là giải pháp thay thế khả thi (và miễn phí trong Free Tier 750h/tháng) để hoàn thành bài lab với quy trình IaC → Training → Inference → Billing check tương đương.
