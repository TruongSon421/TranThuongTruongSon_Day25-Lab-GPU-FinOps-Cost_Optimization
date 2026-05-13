# Báo Cáo GPU FinOps & Cost Optimization

**Họ và tên:** Trần Thuờng Trường Sơn
**MSSV:** 2A202600313
**Ngày nộp:** 13/05/2026

---

## 1. Giới Thiệu

### 1.1. Mục tiêu của bài lab
Bài lab GPU FinOps (Financial Operations) & Cost Optimization giúp sinh viên nắm vững các khái niệm và kỹ năng thực hành trong việc quản lý chi phí GPU ở môi trường cloud computing. Sinh viên tiếp cận các công cụ và kỹ thuật để giảm chi phí vận hành mà vẫn đảm bảo hiệu suất tối ưu cho các workload training AI/ML.

### 1.2. Tổng quan về GPU FinOps
GPU FinOps là phương pháp quản lý tài nguyên GPU theo nguyên tắc tính chi phí và tối ưu hoá. Trong bài lab, chúng tôi sử dụng mock cluster với Docker Compose để mô phỏng môi trường cloud thực tế, bao gồm:

- **GPU Cluster Monitoring:** Theo dõi trạng thái GPU nodes theo thời gian thực
- **Cost Tracking:** Phân tích chi phí theo từng workload, node, và giai đoạn
- **Spot Instance Management:** Quản lý Spot instances để tiết kiệm chi phí (tiết kiệm đến 70%)
- **Autoscaling (KEDA-like):** Tự động điều chỉnh số lượng GPU theo nhu cầu
- **Budget Management:** Dự báo chi phí và cấu hình cảnh báo

---

## 2. Phân Tích Từng Phần

### 2.1. Part 1: GPU Cluster Monitoring

Hệ thống cluster bao gồm **5 nodes** với **10 GPUs** thuộc 3 loại khác nhau. Kết quả thu thập được:

**Thông tin các GPU nodes:**

| Node | GPU 0 | GPU 1 |
|------|-------|-------|
| node-00 | T4 - 65.6% util, 11.3/16GB | T4 - 83.9% util, 11.0/16GB |
| node-01 | A100 - 93.8% util, 63.4/80GB | A100 - 64.0% util, 43.0/80GB |
| node-02 | V100 - 91.9% util, 16.8/32GB | V100 - 5.3% util, 0.6/32GB |
| node-03 | T4 - 88.1% util, 13.3/16GB | T4 - 84.1% util, 11.4/16GB |
| node-04 | T4 - 0.0% util, 0.5/16GB (idle) | T4 - 0.0% util, 0.5/16GB (idle) |

**Cluster Metrics Summary:**

- Total GPUs: **10**
- Busy GPUs: **7**
- Idle GPUs: **3**
- Avg Utilization: **57.7%**
- Memory Used: **171.8 GB / 320.0 GB**
- Total Power Draw: **1026 W**
- Node Count: **5**

**Insights:**
- Có 3 GPU đang idle (node-04 hoàn toàn không sử dụng), đây là cơ hội tối ưu hoá chi phí
- GPU A100 trên node-01 có utilization cao nhất (93.8%), cho thấy workload nặng đang chạy
- Tổng công suất tiêu thụ 1026W cho thấy cluster đang hoạt động ở mức trung bình

### 2.2. Part 2: Workload Submission & Cost Tracking

**Kết quả submit 4 workloads:**

```
train-resnet-001: running → node-04, GPU 0
train-bert-002: running → node-02, GPU 1
inference-api-003: running → node-04, GPU 1
train-llm-004: queued → queued
```

- Sau khi submit: Busy GPUs: **10/10**, Utilization: **82.7%**

**Billing Summary:**

| Workload | Type | Cost | Savings |
|----------|------|------|---------|
| train-resnet-001 | ON-DEMAND | $0.0292 | $0.0000 |
| train-bert-002 | ON-DEMAND | $0.6117 | $0.0000 |
| inference-api-003 | SPOT | $0.0035 | $0.0082 |
| train-llm-004 | SPOT | $0.5505 | $1.2845 |
| **Total** | | **$2.5589** | **$2.7078** |

- Budget Used: **2.6%**
- Alert Status: **OK**

**Nhận xét:** Việc sử dụng Spot instances cho inference và LLM training giúp tiết kiệm đáng kể. Tổng savings $2.7078 cho thấy chiến lược hybrid (On-demand + Spot) đã phát huy hiệu quả.

### 2.3. Part 3: Spot Instance Management

**Bảng giá Spot hiện tại:**

| GPU Type | On-Demand | Spot Price | Discount | Availability |
|----------|-----------|------------|----------|-------------|
| T4 | $0.35/hr | $0.2773/hr | 20.8% | low |
| A100 | $3.67/hr | $2.7467/hr | 25.2% | medium |
| V100 | $2.48/hr | $1.9620/hr | 20.9% | low |

**Spot Instance Requests:**
- spot-t4-001 (T4): ✅ granted
- spot-t4-002 (T4): ✅ granted
- spot-a100-001 (A100): ✅ granted

**Spot Preemption & Savings Report:**

| Metric | Value |
|--------|-------|
| Spot cost | $0.5015 |
| On-demand equivalent | $1.6715 |
| **Total saved** | **$1.1701 (70.0%)** |

**Phân tích:**
- Discount spot cho A100 cao nhất (25.2%), mang lại tiết kiệm lớn nhất
- Tổng tiết kiệm 70% khi chuyển sang Spot instances
- Cần có chiến lược checkpointing để xử lý preemption

### 2.4. Part 4: Autoscaling (KEDA-like)

**Autoscaling Policy Evaluation:**

```
Initial state: Utilization 82.7% > threshold 70.0%
Action: SCALE_UP
Current utilization: 82.7%
Nodes: 5 → 6
```

**5 Evaluation Cycles:**

| Cycle | Action | Util | Nodes |
|-------|--------|------|-------|
| 1 | no_action | 68.9% | 6→6 |
| 2 | no_action | 68.9% | 6→6 |
| 3 | no_action | 68.9% | 6→6 |
| 4 | no_action | 68.9% | 6→6 |
| 5 | no_action | 68.9% | 6→6 |

**Nhận xét:**
- Sau khi scale up, utilization giảm xuống 68.9% (dưới threshold 70%), nên không cần thêm node
- Hệ thống tự động cân bằng giữa hiệu suất và chi phí
- Cooldown period hoạt động hiệu quả, tránh repeated scaling

### 2.5. Part 5: Cost Analysis & Optimization

**Waste Analysis Report:**

| Metric | Value |
|--------|-------|
| Average Waste | **6.2%** |
| Total Idle Cost | $0.026329 |
| Total Cost | $0.423330 |
| Potential Monthly Save | **$682.45** |
| Severity | **LOW** |

**Cost Optimization Recommendations:**

| Priority | Recommendation | Mô tả | Savings |
|----------|---------------|--------|---------|
| MEDIUM | USE_SPOT | Chuyển fault-tolerant workloads sang spot instances | **65.0%** |
| LOW | SCHEDULING | Lên lịch training không khẩn cấp vào giờ thấp điểm | **20.0%** |

**Insights:**
- Waste ở mức LOW (6.2%), cho thấy cluster được quản lý tốt
- Tiết kiệm tiềm năng $682.45/tháng nếu áp dụng đầy đủ recommendations
- Ưu tiên 1: Chuyển sang Spot instances (dễ implement, savings cao)

### 2.6. Part 6: Visualization

Các biểu đồ visualization giúp nhận diện xu hướng và bất thường:

- **Cost Breakdown Chart:** Phân chia chi phí theo loại (compute, memory, network)
- **Time-series Tracking:** Theo dõi chi phí theo thời gian, phát hiện chi phí tăng đột biến
- **Waste Percentage:** Độ thi waste theo thời gian, giúp đánh giá hiệu quả tối ưu hoá

### 2.7. Part 7: Complete FinOps Workflow

Quy trình FinOps đầy đủ 7 bước:

| Step | Nội dung | Kết quả |
|------|----------|---------|
| 1 | Initial cluster state | GPUs: 12, Util: 68.9%, Idle: 2 |
| 2 | Submitting heavy workloads | Util: 81.7%, Busy: 12/12 |
| 3 | Autoscaler evaluation | Decision: scale_up (81.7% > 70%) |
| 4 | Cost analysis | Cost: $0.043889, Waste: 4.4% |
| 5 | Recommendations | USE_SPOT (~65%), SCHEDULING (~20%) |
| 6 | Applying optimization | Spot savings: $0.0292 (70.0%) |
| 7 | Final billing | Spend: $2.7280, Saved: $2.8302, Budget: 2.7% |

---

## 3. Part 8: Real GPU Workload Training

### 3.1. GPU Detection

```
Real GPU Detected:
  Name:     Tesla T4
  Memory:   15.6 GB
  Type:     T4
  Pricing:  $0.35/hr (on-demand)
  CUDA:     12.8
  pynvml:   available
```

**GPU Metrics Diagnostic:**

| Metric | Value |
|--------|-------|
| GPU Util | 0% (idle before training) |
| Memory Total | 16,106 MB (~16 GB) |
| Memory Used | 472 MB |
| Power | 10.4W |
| Temperature | 46°C |

### 3.2. FP32 vs Mixed Precision (AMP) Comparison

Chúng tôi thực hiện training model ResNet-18 trên CIFAR-10 để so sánh FP32 và AMP:

**FP32 Training (Baseline):**

| Metric | Value |
|--------|-------|
| Epoch 1 | Loss: 1.9566, Acc: 30.4%, Time: 40.3s |
| Epoch 2 | Loss: 1.3788, Acc: 49.4%, Time: 42.4s |
| Epoch 3 | Loss: 1.0700, Acc: 61.8%, Time: 45.0s |
| Total Time | **127.7s** |
| Peak Memory | **0.82 GB** |
| Avg GPU Util | **95.5%** |
| Avg Power | **66.6W** |
| Avg Temperature | **71.3°C** |
| Max GPU Util | **99.0%** |
| Estimated Cost | **$0.012412** |

**AMP Training (Mixed Precision):**

| Metric | Value |
|--------|-------|
| Epoch 1 | Loss: 1.9696, Acc: 29.1%, Time: 19.5s |
| Epoch 2 | Loss: 1.3989, Acc: 48.5%, Time: 19.4s |
| Epoch 3 | Loss: 1.1015, Acc: 60.5%, Time: 19.6s |
| Total Time | **58.5s** |
| Peak Memory | **0.60 GB** |
| Avg GPU Util | **91.3%** |
| Avg Power | **65.0W** |
| Avg Temperature | **76.6°C** |
| Max GPU Util | **94.0%** |
| Estimated Cost | **$0.005687** |

**So sánh chi tiết:**

| Metric | FP32 | AMP | Cải thiện |
|--------|------|-----|-----------|
| Total Time | 127.7s | 58.5s | **2.18x nhanh hơn** |
| Peak Memory | 0.82 GB | 0.60 GB | **0.22 GB tiết kiệm** |
| Cost (USD) | $0.012412 | $0.005687 | **$0.006725 tiết kiệm** |
| Cost Saving | - | - | **54.2%** |
| Final Accuracy | 61.8% | 60.5% | -1.3% |

**Extrapolated Savings at Scale:**

| Timeframe | FP32 Cost | AMP Cost | Tiết kiệm |
|-----------|-----------|----------|-----------|
| 1 day | $8.40 | $3.85 | **$4.55** |
| 1 week | $58.80 | $26.94 | **$31.86** |
| 1 month | $252.00 | $115.47 | **$136.53** |

**Phân tích:**
- Mixed Precision (AMP) giảm **54.2% chi phí** chỉ trong 3 epochs training
- Training time giảm **2.18 lần** nhờ tăng throughput
- Memory giảm 0.22 GB, cho phép sử dụng batch size lớn hơn
- Độ chính xác giảm không đáng kể (-1.3%), có thể chấp nhận được
- Với training 1 tháng, tiết kiệm được **$136.53**

### 3.3. GPU Cost Reporting

**Reporting to FinOps Gateway:**

```
FP32 workload reported:
  Cost: $0.012400 | Rate: $0.3500/hr

AMP workload reported (as spot):
  Cost: $0.001700 | Saved: $0.004000
```

**Final Dashboard (Mock + Real GPU):**

| Metric | Value |
|--------|-------|
| Total Platform Cost | $2.7280 |
| Total Savings | $2.8302 |
| Budget Utilization | 2.7% |
| Alert | OK |

---

## 4. Part 8.5: Advanced GPU Cost Optimization

### 4.1. Multi-GPU Scaling Efficiency

Phân tích chi phí khi sử dụng nhiều GPU (base: 2 giờ training trên 1 GPU A100 @ $3.67/hr):

| GPU Count | Speedup | Time (h) | Total Cost | Cost/Perf | Efficiency |
|-----------|---------|----------|-----------|-----------|-----------|
| 1 | 1.00x | 2.000 | $7.34 | $7.34 | 100.0% |
| 2 | 1.80x | 1.111 | $8.16 | $4.53 | 90.0% |
| 4 | 3.40x | 0.588 | $8.64 | $2.54 | 85.0% |
| 8 | 6.00x | 0.333 | $9.79 | $1.63 | **75.0%** ← OPTIMAL |

**Nhận xét:**
- Scaling efficiency giảm khi tăng số lượng GPU do communication overhead
- **8 GPUs là điểm tối ưu** về chi phí/hiệu suất (75% efficiency)
- Trên 8 GPUs, chi phí tăng nhanh hơn lợi ích về tốc độ
- Cost per performance giảm từ $7.34 (1 GPU) xuống $1.63 (8 GPUs) = **giảm 78%**

### 4.2. Project Cost Forecasting

Dự báo chi phí cho project training với 4 giai đoạn:

| Phase | GPU | Số lượng | Giờ | Base Cost | Uncertainty |
|-------|-----|----------|-----|-----------|------------|
| Data Preparation | T4 | 1 | 40h | $14.00 | ±15% |
| Model Training | A100 | 4 | 120h | $1,761.60 | ±25% |
| Hyperparameter Tuning | A100 | 8 | 60h | $1,761.60 | ±30% |
| Model Evaluation | T4 | 2 | 20h | $14.00 | ±10% |

**Financial Summary:**

| Metric | Value |
|--------|-------|
| Base Cost (expected) | **$3,551.20** |
| Contingency (20%) | +$710.24 |
| **Total with Contingency** | **$4,261.44** |
| 95% Confidence Interval | [$2,913.09 - $5,609.79] |
| Worst Case (base+3σ) | $5,614.99 |
| Best Case (base-3σ) | $1,487.41 |

**Phân tích:**
- Model Training chiếm ~50% chi phí ($1,761.60)
- Confidence interval rộng (±47%) cho thấy uncertainty cao
- Cần buffer 20% contingency để phòng ngừa rủi ro
- Worst case ($5,614.99) cao hơn 1.5 lần best case

### 4.3. Optimization Strategy Prioritization

Xếp hạng các chiến lược tối ưu hoá (baseline: 4x A100, 100h, FP32, On-demand = $1,468.00):

| # | Strategy | Savings | Effort | Risk | Score | Tiết kiệm ($) |
|---|----------|---------|--------|------|-------|--------------|
| 1 | Switch to Mixed Precision (AMP) | 25% | LOW | LOW | 0.250 | $367.00 |
| 2 | Use Spot Instances | 60% | MEDIUM | HIGH | 0.180 | $880.80 |
| 3 | Optimize Batch Size | 15% | LOW | LOW | 0.150 | $220.20 |
| 4 | Switch to More Efficient GPU Type | 40% | HIGH | MEDIUM | 0.107 | $587.20 |
| 5 | Implement Early Stopping | 20% | MEDIUM | LOW | 0.100 | $293.60 |

**Cumulative Savings Roadmap:**

| Strategy | Cumulative ($) | Total Save % | Remaining Cost |
|----------|---------------|--------------|---------------|
| AMP | $367.00 | 25.0% | $1,101.00 |
| + Spot Instances | $1,027.60 | 70.0% | $440.40 |
| + Batch Size | $1,093.66 | 74.5% | $374.34 |
| + GPU Type | $1,243.40 | 84.7% | $224.60 |
| + Early Stopping | $1,288.32 | **87.8%** | **$179.68** |

**Quick wins:** AMP + Batch Size optimization = **40% savings** với effort thấp

### 4.4. Challenge Exercise: LLM Fine-tuning Cost Optimization

**Scenario:**
- Project: Large Language Model Fine-tuning
- Baseline: 8x A100 trong 200 giờ
- Budget: $5,000
- Deadline: 2 weeks (336h)

**Baseline Cost:** $5,872.00 (vượt budget $872.00 ❌)

**Multi-GPU Analysis với 16 GPUs:**

| GPU Count | Speedup | Time (h) | Total Cost | Efficiency |
|-----------|---------|----------|-----------|-----------|
| 1 | 1.00x | 200.000 | $734.00 | 100.0% |
| 2 | 1.80x | 111.111 | $815.56 | 90.0% |
| 4 | 3.40x | 58.824 | $863.53 | 85.0% |
| 8 | 6.00x | 33.333 | $978.67 | 75.0% |
| 16 | 10.56x | 18.946 | $1,112.54 | **66.0%** ← OPTIMAL |

**Optimization Applied:**
- Chuyển 8x A100 → 16x A100 (optimal GPU count)
- Chuyển FP32 → AMP
- Hybrid: Spot cho batch, On-demand cho critical
- Early stopping + batch size tuning

**Final Cost Forecast:**

| Metric | Value |
|--------|-------|
| Optimized Base Cost | $3,593.66 |
| Contingency (15%) | +$539.05 |
| **Total Forecast** | **$4,132.71** |
| Total Savings | $2,278.34 (**38.8%**) |
| Under Budget? | **YES ✅** |
| Deadline Check | **NO ❌** (533.3h > 336h) |

**Kết luận Challenge:**
- ✅ Vượt budget: tiết kiệm $867.29 so với $5,000
- ❌ Không đáp ứng deadline 2 tuần với 16 GPUs
- Đề xuất: giảm scope hoặc kéo dài deadline

---

## 5. Kết Luận Và Học Hỏi

### 5.1. Kỹ năng FinOps đã học

1. **Cost Awareness:** Hiểu rõ chi phí GPU là yếu tố quan trọng trong AI/ML projects
2. **Optimization Techniques:** Biết cách sử dụng AMP, Spot instances, autoscaling để giảm chi phí
3. **Forecasting:** Có khả năng dự báo chi phí, giúp lập kế hoạch tài chính tốt hơn
4. **Monitoring:** Thiết lập hệ thống monitoring để phát hiện waste sớm

### 5.2. Các chiến lược cost optimization hiệu quả nhất

| Chiến lược | Tiết kiệm trung bình | Effort | Risk |
|------------|---------------------|--------|------|
| Mixed Precision (AMP) | **25-54%** | Thấp | Thấp |
| Spot Instances | **60-70%** | Trung bình | Cao |
| Early Stopping | **15-25%** | Trung bình | Thấp |
| Batch Size Optimization | **10-20%** | Thấp | Thấp |
| GPU Type Selection | **30-70%** | Cao | Trung bình |

### 5.3. Bảng tổng hợp chi phí GPU

| GPU Type | On-Demand | Spot | Discount |
|----------|-----------|------|----------|
| T4 | $0.35/hr | $0.2773/hr | 20.8% |
| A100 | $3.67/hr | $2.7467/hr | 25.2% |
| V100 | $2.48/hr | $1.9620/hr | 20.9% |

### 5.4. Ứng dụng thực tế trong projects

- **Small projects (< 50h):** Dùng T4 on-demand, AMP, early stopping
- **Medium projects (50-200h):** Kết hợp T4 + A100 Spot, autoscaling
- **Large projects (> 200h):** Full FinOps workflow, checkpointing, multi-GPU optimization

### 5.5. Recommendations

1. **Always use Mixed Precision (AMP)** - Gain free performance
2. **Start with smaller GPUs** - Upgrade only when needed
3. **Use Spot for non-critical training** - Save 60-70%
4. **Implement checkpointing** - Protect against preemption
5. **Set up budget alerts** - Prevent cost overruns
6. **Review waste weekly** - Continuous optimization

---

## 6. Hình Ảnh Minh Hoạ

Các hình ảnh minh hoạ được lưu trong thư mục `screenshots/` và `generated_charts/`:
- Cluster monitoring screenshots (Part 1)
- Cost breakdown charts (Part 6)
- FP32 vs AMP comparison (Part 8)
- Multi-GPU scaling efficiency (Part 8.5)
- Advanced FinOps dashboard (Part 8.5)
- Project forecast visualizations (Part 8.5)

---

## 7. Phụ Lục: Bảng Giá Tham Khảo

### GPU Pricing (On-Demand vs Spot)

| GPU | On-Demand ($/hr) | Spot ($/hr) | Savings |
|-----|-----------------|-------------|---------|
| T4 | 0.35 | 0.2773 | 20.8% |
| A100 | 3.67 | 2.7467 | 25.2% |
| V100 | 2.48 | 1.9620 | 20.9% |

### Training Cost Extrapolation (AMP trên T4)

| Timeframe | Cost |
|-----------|------|
| Per epoch (3 epochs = 58.5s) | $0.0057 |
| 1 hour | $0.35 |
| 1 day | $8.40 |
| 1 week | $58.80 |
| 1 month | $252.00 |

---

**Kết luận:** Bài lab đã cung cấp nền tảng kiến thức và kỹ năng thực hành quan trọng về GPU FinOps. Việc áp dụng những kỹ thuật tối ưu hoá chi phí đã học có thể giảm **50-88% chi phí GPU** trong các dự án AI/ML thực tế. Đặc biệt, Mixed Precision (AMP) là chiến lược "quick win" với hiệu quả cao nhất với effort thấp nhất.

---

*Người thực hiện: Trần Thuờng Trường Sơn*
*MSSV: 2A202600313*
*Ngày hoàn thành: 13/05/2026*
