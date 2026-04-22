# Vinmec AI Medical Scribe
## Kế hoạch Triển khai & Phân tích Production

> **Phiên bản:** 1.0 | **Ngày:** Tháng 4, 2026 | **Mức độ:** Confidential — Dành cho Nhà đầu tư & Ban Lãnh đạo

---

## Mục lục

1. [Project Overview](#1-project-overview)
2. [Enterprise Context](#2-enterprise-context)
3. [Deployment Choice](#3-deployment-choice)
4. [Cost Analysis — MVP & Growth](#4-cost-analysis--mvp--growth)
5. [Optimization Plan](#5-optimization-plan)
6. [Reliability Plan](#6-reliability-plan)
7. [Future Roadmap](#7-future-roadmap)
8. [Recommendation & Next Steps](#8-recommendation--next-steps)

---

## 1. Project Overview

### 1.1 Bài toán cốt lõi

Vinmec vận hành theo mô hình bệnh viện quốc tế với tiêu chuẩn JCI (Joint Commission International). Mỗi ca khám kéo dài trung bình **30 phút**, nhưng thực tế cho thấy:

| Hoạt động | Thời gian trung bình | % tổng ca khám |
|---|---|---|
| Thăm khám lâm sàng trực tiếp | 18–20 phút | 60–65% |
| Nhập liệu HIS, gõ bệnh án | 8–10 phút | 28–33% |
| Các thao tác hành chính khác | 2–3 phút | 7–10% |

**Hệ quả:** Bác sĩ mất 30–40% thời gian cho công việc hành chính thay vì tập trung vào bệnh nhân. Tại một hệ thống có hàng chục nghìn ca khám/năm, đây là tổn thất năng lực khổng lồ.

### 1.2 Giải pháp: AI Medical Scribe

Hệ thống sử dụng AI để **tự động chuyển đổi cuộc hội thoại tự nhiên** giữa bác sĩ và bệnh nhân thành bệnh án có cấu trúc chuẩn, tích hợp thẳng vào HIS.

```
[Phòng khám]
Bác sĩ ↔ Bệnh nhân (hội thoại tự nhiên)
         ↓
[AI Medical Scribe Pipeline]
Microphone → Speech-to-Text → NLP/LLM Processing → Structured Note
         ↓
[HIS Integration]
SOAP Note → ICD-10 Codes → HIS/EMR → Bác sĩ xem xét & ký duyệt
```

### 1.3 Người dùng chính

| Nhóm người dùng | Vai trò | Tần suất sử dụng |
|---|---|---|
| **Bác sĩ nội khoa, ngoại khoa** | Primary user — nhận output bệnh án | Mỗi ca khám (~15–30 ca/ngày) |
| **Điều dưỡng** | Secondary user — hỗ trợ review | Theo ca làm việc |
| **Quản trị viên HIS** | Admin — cấu hình, audit | Định kỳ |
| **Ban lãnh đạo Vinmec** | Stakeholder — dashboard tổng quan | Báo cáo tuần/tháng |

### 1.4 Năng lực kỹ thuật đội ngũ (Built in 15 ngày)

**3 kỹ năng cốt lõi đã được kiểm chứng:**

1. **Speech-to-Text Pipeline** — Tích hợp Whisper (OpenAI) với xử lý tiếng Việt y tế, độ chính xác ~92% trên thuật ngữ chuyên khoa
2. **LLM Structured Extraction** — Prompt engineering với Claude/GPT-4 để trích xuất SOAP note, ICD-10 coding tự động từ transcript
3. **HIS API Integration** — Kết nối với hệ thống HIS thông qua HL7 FHIR standard, write-back tự động sau khi bác sĩ approve

**Sản phẩm/Agent đã xây dựng:**
- MVP demo chạy end-to-end: ghi âm → transcript → SOAP note → hiển thị để bác sĩ duyệt
- Scenario đã test: Khoa Nội tổng quát, ca khám bệnh mạn tính (tiểu đường, tăng huyết áp)
- Accuracy rate trên test set nội bộ: **87% field-level correctness** (Chief Complaint, HPI, Assessment, Plan)

### 1.5 Tại sao chủ đề này phù hợp để phân tích Deployment & Cost?

- **Dữ liệu cực kỳ nhạy cảm** (PHI — Protected Health Information): buộc phải phân tích deployment nghiêm túc
- **Volume dự đoán được**: Vinmec có lịch khám cố định → ước lượng traffic chính xác
- **Chi phí phức tạp, nhiều lớp**: không chỉ token API mà còn STT, storage, compliance, human review
- **Hệ quả của lỗi cao**: sai bệnh án → ảnh hưởng trực tiếp đến bệnh nhân → tạo áp lực thiết kế reliability nghiêm túc

---

## 2. Enterprise Context

### 2.1 Bối cảnh tổ chức

**Vinmec International Hospital Group** là tổ hợp bệnh viện tư nhân lớn nhất Việt Nam với:
- **11+ bệnh viện** trải dài từ Hà Nội, TP.HCM đến các tỉnh thành
- **Tiêu chuẩn JCI** — chuẩn quốc tế cao nhất về quản lý chất lượng bệnh viện
- **HIS hiện tại**: Oracle Health (Cerner) hoặc hệ thống tương đương — đã có sẵn cơ sở hạ tầng CNTT
- **Quy mô**: ~3.000–5.000 ca khám ngoại trú/ngày toàn hệ thống

### 2.2 Dữ liệu hệ thống xử lý

| Loại dữ liệu | Mô tả | Mức độ nhạy cảm |
|---|---|---|
| **Audio recording** | Toàn bộ cuộc hội thoại bác sĩ–bệnh nhân | ⚠️ Cực cao |
| **Transcript** | Văn bản chuyển đổi từ audio | ⚠️ Cực cao |
| **PHI — Thông tin định danh** | Họ tên, ngày sinh, CMND, số thẻ BHYT | ⚠️ Cực cao |
| **Chẩn đoán & điều trị** | ICD-10, đơn thuốc, chỉ định xét nghiệm | ⚠️ Cực cao |
| **Lịch sử bệnh** | Tiền sử bệnh, dị ứng thuốc | ⚠️ Cao |
| **Metadata hệ thống** | Thời gian, bác sĩ phụ trách, phòng khám | 🔶 Trung bình |
| **Audit logs** | Ai truy cập, chỉnh sửa gì, khi nào | 🔶 Trung bình |

> **Kết luận về độ nhạy cảm:** Toàn bộ dữ liệu cốt lõi của hệ thống thuộc nhóm **PHI (Protected Health Information)** — được bảo vệ bởi Luật An toàn thông tin mạng Việt Nam, Thông tư 09/2023/TT-BYT và tiêu chuẩn JCI. **Dữ liệu KHÔNG được phép rời khỏi hạ tầng của Vinmec** nếu không có cơ chế bảo mật đặc biệt.

### 2.3 Ba ràng buộc Enterprise lớn nhất

#### Ràng buộc 1: Data Sovereignty & Compliance
- Dữ liệu bệnh nhân phải lưu trữ **trong lãnh thổ Việt Nam** theo quy định pháp lý
- Cần đáp ứng Thông tư 09/2023/TT-BYT về hồ sơ bệnh án điện tử
- Mọi thay đổi bệnh án phải có **audit trail đầy đủ** (ai sửa gì, lúc nào)
- Không được dùng dữ liệu thật để train model của bên thứ ba

#### Ràng buộc 2: HIS Integration & Approval Workflow
- Phải tích hợp với HIS hiện tại qua chuẩn **HL7 FHIR v4**
- Bác sĩ phải **review và ký điện tử** trước khi bệnh án được lưu chính thức
- Không được ghi đè bệnh án mà không có xác nhận của bác sĩ phụ trách
- Cần đồng bộ real-time với các module: Dược, Xét nghiệm, Chẩn đoán hình ảnh

#### Ràng buộc 3: Uptime & Clinical Availability
- Hệ thống phải đảm bảo **SLA 99.9% uptime** trong giờ khám (7h–20h hàng ngày)
- Downtime > 5 phút trong giờ cao điểm ảnh hưởng trực tiếp đến workflow lâm sàng
- Cần **fallback mechanism** rõ ràng khi AI không hoạt động (bác sĩ phải nhập tay như cũ — không được tắc nghẽn)
- Latency tối đa chấp nhận được: **< 30 giây** từ kết thúc hội thoại đến hiển thị draft bệnh án

### 2.4 Các câu hỏi quan trọng và câu trả lời

| Câu hỏi | Câu trả lời |
|---|---|
| Có cần audit trail không? | **Bắt buộc.** Mọi thao tác trên bệnh án phải được log đầy đủ |
| Dữ liệu có được rời khỏi tổ chức không? | **Không.** Phải xử lý trong hạ tầng nội bộ hoặc private cloud |
| Cần tích hợp hệ thống cũ không? | **Có.** Tích hợp HIS qua HL7 FHIR, có approval workflow |
| Nếu trả lời sai thì ai bị ảnh hưởng đầu tiên? | **Bệnh nhân** — sai chẩn đoán, sai thuốc, sai liều |

---

## 3. Deployment Choice

### 3.1 Quyết định: Hybrid Deployment

```
┌─────────────────────────────────────────────────────────┐
│                    HYBRID ARCHITECTURE                   │
│                                                         │
│  ┌──────────────────────┐   ┌───────────────────────┐  │
│  │   ON-PREMISES (Core) │   │  PRIVATE CLOUD (Scale) │  │
│  │                      │   │                        │  │
│  │  • PHI Data Storage  │   │  • AI Inference        │  │
│  │  • Audio Files       │◄──►  • STT Processing      │  │
│  │  • HIS Integration   │   │  • LLM API Gateway     │  │
│  │  • Audit Logs        │   │  • Auto-scaling        │  │
│  │  • Final SOAP Notes  │   │  • Load Balancing      │  │
│  └──────────────────────┘   └───────────────────────┘  │
│           ▲                           ▲                 │
│           │                           │                 │
│     [Encrypted]              [VPN/Private Link]         │
│      [In-VN DC]              [VN-based Cloud]          │
└─────────────────────────────────────────────────────────┘
```

### 3.2 Hai lý do chọn Hybrid

**Lý do 1: Bảo mật dữ liệu PHI không thể thỏa hiệp**
Dữ liệu bệnh án, audio recording và PHI phải lưu trữ on-premises tại data center của Vinmec (hoặc co-location trong lãnh thổ VN). Không có cloud provider toàn cầu nào đảm bảo 100% dữ liệu không ra ngoài biên giới theo chuẩn pháp lý VN hiện tại. On-prem cho core data = tuân thủ pháp lý tuyệt đối.

**Lý do 2: AI inference cần elasticity mà on-prem thuần túy không đáp ứng được**
Tải AI inference biến động mạnh (sáng thứ Hai cao điểm vs. tối Chủ nhật thấp điểm). Duy trì GPU server on-prem đủ công suất cho peak traffic sẽ lãng phí 60–70% tài nguyên trong giờ thấp điểm. Private cloud tại VN (VNG Cloud, FPT Cloud) cho phép auto-scaling, chỉ trả tiền theo usage, trong khi vẫn đảm bảo dữ liệu không rời lãnh thổ.

### 3.3 So sánh các phương án

| Tiêu chí | Cloud thuần | On-prem thuần | **Hybrid (Chọn)** |
|---|---|---|---|
| Tuân thủ pháp lý | ⚠️ Rủi ro | ✅ Tốt | ✅ Tốt |
| Chi phí ban đầu (CapEx) | Thấp | Cao | Trung bình |
| Khả năng scale | ✅ Tốt | ❌ Kém | ✅ Tốt |
| Kiểm soát dữ liệu | ❌ Kém | ✅ Tốt | ✅ Tốt |
| Tốc độ triển khai | Nhanh | Chậm | Trung bình |
| Uptime & reliability | ✅ Tốt | ⚠️ Phụ thuộc team | ✅ Tốt |

---

## 4. Cost Analysis — MVP & Growth

### 4.1 Giả định đầu vào

| Tham số | MVP (Phase 1) | Growth 5x | Growth 10x |
|---|---|---|---|
| Bệnh viện | 1 (Vinmec Times City) | 5 bệnh viện | 11 bệnh viện |
| Bác sĩ active/ngày | 50 | 250 | 500 |
| Ca khám có AI/ngày | 500 | 2.500 | 5.000 |
| Peak concurrent requests | 30 | 150 | 300 |
| Audio duration trung bình | 15 phút | 15 phút | 15 phút |
| Input tokens/request (transcript + context) | ~8.000 | ~8.000 | ~8.000 |
| Output tokens/request (SOAP note) | ~1.500 | ~1.500 | ~1.500 |

### 4.2 Chi phí MVP — Phân tích theo lớp

#### Layer 1: AI/LLM Inference Cost
```
Speech-to-Text (Whisper API hoặc tương đương):
  500 ca × 15 phút × $0.006/phút = $45/ngày → ~$1.350/tháng

LLM Structured Extraction (Claude Sonnet):
  500 ca × 8.000 input tokens × $3/1M tokens = $12/ngày
  500 ca × 1.500 output tokens × $15/1M tokens = $11.25/ngày
  Total LLM: ~$23/ngày → ~$700/tháng

Tổng AI inference: ~$2.050/tháng
```

#### Layer 2: Compute & Infrastructure
```
Private Cloud (VNG/FPT Cloud):
  API Gateway + Orchestration server: ~$300/tháng
  AI Processing nodes (auto-scale): ~$400/tháng
  Network egress/ingress: ~$50/tháng

On-premises (amortized):
  Server hardware (3-year depreciation): ~$500/tháng
  Maintenance & IT ops: ~$300/tháng

Tổng Compute: ~$1.550/tháng
```

#### Layer 3: Storage
```
Audio files: 500 ca × 15 phút × ~15MB/phút = ~112GB/ngày → ~3.4TB/tháng
  Storage cost (hot): ~$0.025/GB → ~$85/tháng
  
Transcripts & SOAP notes: ~5GB/tháng → $1/tháng
Audit logs: ~2GB/tháng → $0.5/tháng
Backup (cold storage): ~$30/tháng

Tổng Storage: ~$120/tháng
```

#### Layer 4: Human Review & QA
```
Medical reviewer (part-time, QA sampling 5% cases):
  25 ca/ngày × 3 phút/ca = 1.25 giờ/ngày
  Cost: ~$15/giờ → ~$560/tháng (bao gồm overhead)

Tổng Human Review: ~$560/tháng
```

#### Layer 5: Logging, Monitoring & Compliance
```
Logging platform (ELK Stack hoặc cloud): ~$150/tháng
APM & Monitoring (Datadog/Grafana): ~$100/tháng
Security scanning & compliance audit: ~$200/tháng

Tổng Logging/Monitoring: ~$450/tháng
```

#### Layer 6: Maintenance & Development
```
DevOps/MLOps engineer (0.5 FTE): ~$1.000/tháng
Model update & retraining pipeline: ~$200/tháng
Security patches & updates: ~$100/tháng

Tổng Maintenance: ~$1.300/tháng
```

### 4.3 Bảng tổng hợp chi phí

| Layer | MVP/tháng | Growth 5x/tháng | Growth 10x/tháng |
|---|---|---|---|
| AI/LLM Inference | $2.050 | $8.500 | $16.000 |
| Compute & Infrastructure | $1.550 | $3.800 | $6.500 |
| Storage | $120 | $450 | $900 |
| Human Review | $560 | $1.400 | $2.200 |
| Logging & Monitoring | $450 | $700 | $1.000 |
| Maintenance | $1.300 | $2.000 | $2.800 |
| **TỔNG** | **~$6.030/tháng** | **~$16.850/tháng** | **~$29.400/tháng** |
| **Chi phí/ca khám** | **~$0.40** | **~$0.22** | **~$0.20** |

> **Economy of scale rõ ràng:** Chi phí/ca giảm từ $0.40 → $0.20 khi scale 10x, nhờ fixed costs (maintenance, infra base) được chia đều.

### 4.4 Cost Driver & Hidden Cost Analysis

**Cost Driver lớn nhất:** AI/LLM Inference (34% tổng cost ở MVP) — đặc biệt phần Speech-to-Text tăng tuyến tính theo số ca khám.

**Hidden costs dễ bị bỏ qua:**

| Hidden Cost | Mô tả | Ước tính |
|---|---|---|
| **Change Management** | Training bác sĩ, điều dưỡng dùng hệ thống | $5.000–$15.000 one-time |
| **Legal & Compliance Review** | Luật sư, chuyên gia y tế review quy trình | $8.000–$20.000/năm |
| **Integration Testing** | Test kết nối HIS, edge cases lâm sàng | $10.000–$30.000 one-time |
| **Data Labeling** | Tạo ground truth cho fine-tuning model VN | $5.000–$20.000 |
| **Incident Response** | On-call support, xử lý lỗi bệnh án | ~$500/tháng |
| **Model Drift Monitoring** | Accuracy giảm theo thời gian nếu không monitor | Tích hợp vào MLOps |

**Khu vực ước lượng lạc quan nhất:** STT accuracy với tiếng Việt có accent địa phương và thuật ngữ chuyên khoa. Trong thực tế production, error rate có thể cao hơn 2–3x so với demo → cần human review nhiều hơn dự kiến.

---

## 5. Optimization Plan

### 5.1 Ba chiến lược tối ưu được chọn

#### Chiến lược 1: Model Routing (Làm ngay — Phase 1)

**Tiết kiệm:** 40–60% chi phí LLM inference

**Cách hoạt động:**
```
Request đến
    │
    ▼
[Classifier] ──► Hội thoại đơn giản, ngắn (<5 phút)
    │              → Dùng model nhỏ: Claude Haiku / GPT-3.5
    │              Cost: ~$0.003/ca
    │
    └──────────► Ca phức tạp, đa bệnh lý, >10 phút
                 → Dùng model mạnh: Claude Sonnet / GPT-4
                 Cost: ~$0.023/ca
```

**Lợi ích:** Giảm 40–60% chi phí LLM mà không ảnh hưởng đến chất lượng (vì ~60% ca khám là routine/đơn giản).

**Trade-off:** Cần xây dựng và maintain classifier. Cần định nghĩa rõ "phức tạp" là gì trong ngữ cảnh lâm sàng.

**Thời điểm áp dụng:** Ngay từ MVP — ROI dương sau 1–2 tháng.

---

#### Chiến lược 2: Prompt Compression (Làm ngay — Phase 1)

**Tiết kiệm:** 20–35% chi phí input tokens

**Cách hoạt động:**
- Thay vì gửi toàn bộ transcript thô (8.000 tokens), dùng bước preprocessing:
  - Loại bỏ filler words, lặp lại, noise
  - Summarize phần không liên quan lâm sàng
  - Kết quả: giảm xuống ~5.000 tokens mà vẫn giữ đủ thông tin lâm sàng

**Lợi ích:** Giảm trực tiếp chi phí input tokens, đồng thời giảm latency.

**Trade-off:** Có nguy cơ mất thông tin lâm sàng quan trọng nếu compression quá mạnh. Cần test kỹ trên diverse cases.

**Thời điểm áp dụng:** Phase 1, sau khi đã có baseline accuracy đủ tốt.

---

#### Chiến lược 3: Self-hosted STT Model (Phase 2 — khi volume đủ lớn)

**Tiết kiệm:** 60–80% chi phí Speech-to-Text

**Cách hoạt động:**
- Tại MVP: dùng Whisper API ($0.006/phút)
- Khi đạt 2.500+ ca/ngày: deploy Whisper Large v3 self-hosted trên private cloud VN
- Fine-tune với corpus tiếng Việt y tế của Vinmec → accuracy tốt hơn + chi phí thấp hơn

**Lợi ích:** STT là cost driver lớn nhất (33% tổng AI cost). Self-hosted = tiết kiệm $7.000–$10.000/tháng ở scale 5x.

**Trade-off:** Cần upfront investment ($15.000–$25.000 cho GPU infra + setup). Break-even ~2–3 tháng sau khi scale. Cần MLOps team maintain.

**Thời điểm áp dụng:** Khi tổng volume > 60.000 phút audio/tháng (tương đương ~4.000 ca/ngày).

---

### 5.2 Roadmap tối ưu hóa

```
Phase 1 (Tháng 1–3): Model Routing + Prompt Compression
  → Tiết kiệm ước tính: 35–45% tổng AI inference cost
  → Investment: ~$2.000 development

Phase 2 (Tháng 4–9): Self-hosted STT khi đủ volume
  → Tiết kiệm ước tính: thêm 25–30% tổng cost
  → Investment: ~$20.000 one-time

Phase 3 (Tháng 10+): Semantic Caching cho common phrases
  → Áp dụng cho các template bệnh án lặp lại (check-up định kỳ, tái khám)
  → Tiết kiệm ước tính: 10–15% LLM calls
```

---

## 6. Reliability Plan

### 6.1 Ba tình huống rủi ro và phản ứng

#### Tình huống 1: Traffic tăng đột biến

**Bối cảnh:** Sáng thứ Hai sau ngày nghỉ lễ, lượng bệnh nhân tăng 3x bình thường. Hoặc Vinmec mở chiến dịch khám tầm soát quy mô lớn.

**Tác động tới user:** Queue xử lý dài, bác sĩ đợi 2–5 phút thay vì 30 giây → workflow bị gián đoạn → bác sĩ bỏ qua AI, nhập tay.

**Phản ứng ngắn hạn:**
- Auto-scaling cloud inference nodes kích hoạt tự động khi CPU > 70%
- Priority queue: ca cấp cứu > nội trú > ngoại trú
- Degrade gracefully: tắt feature phụ (ICD-10 suggestion), giữ core SOAP note generation

**Giải pháp dài hạn:**
- Predictive scaling dựa trên lịch đặt khám (có thể dự báo 24–48h trước)
- Circuit breaker với threshold rõ ràng
- Capacity planning hàng quý dựa trên growth trend

---

#### Tình huống 2: Provider Timeout (LLM API lỗi)

**Bối cảnh:** Claude/GPT-4 API trả về timeout hoặc 5xx error trong 10–30 phút.

**Tác động tới user:** Không có draft SOAP note → bác sĩ phải nhập toàn bộ tay → tương đương mất hoàn toàn giá trị AI.

**Phản ứng ngắn hạn:**
- **Retry policy:** Exponential backoff, tối đa 3 lần trong 30 giây
- **Provider failover:** Primary (Claude) → Secondary (GPT-4) → Tertiary (Gemini Pro) — tự động trong < 5 giây
- **Async fallback:** Gửi transcript vào queue, xử lý sau khi provider phục hồi, notify bác sĩ qua in-app notification

**Giải pháp dài hạn:**
- Duy trì multi-provider contract (không vendor lock-in)
- Self-hosted fallback model (Llama 3 fine-tuned) cho basic SOAP note
- SLA monitoring với alert khi error rate > 1%

---

#### Tình huống 3: Response Chậm / Degraded Performance

**Bối cảnh:** Latency tăng từ 20 giây lên 60–90 giây do mạng nội bộ, STT processing chậm, hoặc model inference bị throttle.

**Tác động tới user:** Bác sĩ đã kết thúc khám nhưng SOAP note chưa xong → phải đợi → phá vỡ flow làm việc.

**Phản ứng ngắn hạn:**
- Hiển thị progress indicator real-time (STT xong rồi, đang phân tích...)
- Cho phép bác sĩ **tiếp tục khám bệnh nhân tiếp theo** trong khi AI xử lý async
- Notify khi SOAP note sẵn sàng (push notification trên HIS dashboard)

**Giải pháp dài hạn:**
- Streaming response: hiển thị SOAP note từng phần khi LLM generate
- Pre-processing STT ngay trong quá trình hội thoại (không đợi đến cuối)
- P95 latency SLA: < 25 giây cho 95% requests

---

### 6.2 Phân loại request: Real-time vs. Async

| Request type | Xử lý | Lý do |
|---|---|---|
| STT transcription | **Streaming real-time** | Cần hiển thị ngay để bác sĩ biết AI đang nghe |
| SOAP Note draft generation | **Async + notify** | Được 30–60 giây, bác sĩ có thể thao tác khác |
| ICD-10 code suggestion | **Async** | Bác sĩ review sau khi xem SOAP note |
| HIS write-back sau approve | **Async + confirmation** | Không cần real-time, cần đảm bảo thành công |
| Audit log ghi nhận | **Async fire-and-forget** | Không ảnh hưởng UX |

### 6.3 Metrics cần monitor

| Metric | Target | Alert threshold |
|---|---|---|
| STT latency P95 | < 45 giây cho 15-phút audio | > 90 giây |
| LLM response latency P95 | < 20 giây | > 40 giây |
| End-to-end latency P95 | < 30 giây | > 60 giây |
| API error rate | < 0.5% | > 2% |
| SOAP note accuracy (sampling) | > 85% | < 80% |
| HIS write-back success rate | > 99.9% | < 99% |
| System uptime (7h–20h) | > 99.9% | < 99.5% |

### 6.4 Fallback Proposal tổng thể

```
Level 1 — Degraded AI (model yếu hơn):
  Dùng khi: Primary model timeout > 5 giây
  Action: Route sang secondary LLM provider
  UX impact: Chất lượng có thể thấp hơn 10–15%, thông báo cho bác sĩ

Level 2 — Template-based (rule-based):
  Dùng khi: Tất cả LLM providers đều lỗi
  Action: Điền template SOAP với thông tin từ STT (keyword extraction)
  UX impact: Bác sĩ phải điền nhiều hơn, nhưng không mất toàn bộ context

Level 3 — Manual (tắt AI hoàn toàn):
  Dùng khi: STT lỗi hoặc audio quality quá kém
  Action: Chuyển sang HIS input thủ công như hiện tại
  UX impact: Trở về trạng thái trước khi có AI — không tệ hơn hiện tại
```

---

## 7. Future Roadmap

### 7.1 Lộ trình mở rộng 3 năm

```
2025 — Phase 1: MVP & Validation
  ✓ Triển khai tại 1 bệnh viện (Vinmec Times City)
  ✓ Nội khoa, Ngoại trú tổng quát
  ✓ 50 bác sĩ, 500 ca/ngày
  ✓ KPI: Giảm 25% thời gian nhập liệu, NPS bác sĩ > 7/10

2026 — Phase 2: Scale & Specialization
  → Mở rộng 5 bệnh viện toàn hệ thống Vinmec
  → Thêm chuyên khoa: Tim mạch, Nhi, Sản
  → Fine-tune model với data Vinmec (domain-specific)
  → Tích hợp PACS (Chẩn đoán hình ảnh), Lab results
  → KPI: 2.500 ca/ngày, ROI dương

2027 — Phase 3: AI-Augmented Clinical Intelligence
  → Predictive alerts: phát hiện bất thường trong pattern khám
  → Clinical decision support: gợi ý chẩn đoán dựa trên evidence
  → Multi-modal: kết hợp transcript + ảnh + kết quả xét nghiệm
  → License model: bán cho các bệnh viện tư nhân khác tại VN/SEA
  → KPI: 5.000+ ca/ngày, sẵn sàng commercialize
```

### 7.2 Cơ hội mở rộng thị trường

| Thị trường | Quy mô ước tính | Rào cản xâm nhập |
|---|---|---|
| Vinmec toàn hệ thống | ~5.000 ca/ngày | Thấp (đã có quan hệ) |
| Bệnh viện tư nhân VN | ~50.000 ca/ngày | Trung bình |
| Bệnh viện công VN | ~500.000 ca/ngày | Cao (thủ tục, ngân sách) |
| SEA (Thailand, Indonesia) | ~1M+ ca/ngày | Cao (ngôn ngữ, pháp lý) |

### 7.3 Các hướng AI mở rộng trong tương lai

- **Predictive Readmission:** AI dự báo nguy cơ tái nhập viện trong 30 ngày
- **Drug Interaction Check:** Tự động cảnh báo tương tác thuốc trong đơn
- **Billing Optimization:** Tự động gợi ý DRG code tối ưu cho bảo hiểm
- **Population Health Analytics:** Aggregate insights từ patterns bệnh lý toàn hệ thống
- **Patient Communication:** AI soạn thảo hướng dẫn sau khám cho bệnh nhân

---

## 8. Recommendation & Next Steps

### 8.1 Tóm tắt Executive

| Hạng mục | Chi tiết |
|---|---|
| **Giải pháp** | AI Medical Scribe — tự động hóa bệnh án từ hội thoại |
| **Deployment** | Hybrid: On-prem (data) + Private Cloud VN (AI inference) |
| **Chi phí MVP** | ~$6.000/tháng cho 500 ca/ngày |
| **Chi phí/ca** | $0.40 (MVP) → $0.20 (scale 10x) |
| **ROI ước tính** | Break-even trong 6–9 tháng |
| **Tiết kiệm** | 25–40% thời gian nhập liệu/bác sĩ → $X,XXX/bác sĩ/năm |
| **Rủi ro lớn nhất** | STT accuracy tiếng Việt y tế + compliance pháp lý |

### 8.2 ROI Analysis cho Nhà đầu tư

**Giả định tính ROI:**
- 50 bác sĩ × 20 ca/ngày × 8 phút tiết kiệm/ca = 8.000 phút/ngày = 133 giờ/ngày
- Giá trị thời gian bác sĩ: ~$30–50/giờ (tại Vinmec international)
- Giá trị tiết kiệm: ~$4.000–6.700/ngày → **~$100.000–$200.000/năm chỉ từ 50 bác sĩ**
- Chi phí hệ thống: ~$72.000/năm (MVP)
- **ROI Year 1:** ~40–180% tùy assumption

**Tại scale 5 bệnh viện (250 bác sĩ):**
- Tiết kiệm: ~$500.000–$1.000.000/năm
- Chi phí hệ thống: ~$200.000/năm
- **ROI: 150–400%**

### 8.3 Next Steps — 90 ngày đầu tiên

| Tuần | Hành động | Owner | Deliverable |
|---|---|---|---|
| 1–2 | Ký MOU với Vinmec, xác nhận pilot scope | Business Dev | MOU document |
| 2–4 | Setup hạ tầng hybrid, VPN với HIS | DevOps | Infrastructure ready |
| 3–6 | Tích hợp HIS (HL7 FHIR), test sandbox | Backend Dev | API integration |
| 4–8 | Thu thập 500 audio samples, build training set | ML Team | Labeled dataset |
| 6–10 | Fine-tune STT cho tiếng Việt y tế | ML Engineer | Improved STT model |
| 8–12 | Pilot với 5 bác sĩ nội khoa | Product | User feedback report |
| 10–14 | Đánh giá accuracy, điều chỉnh prompt | ML + Clinical | Accuracy baseline |
| 12–16 | Mở rộng 20 bác sĩ, đo KPI thực tế | Full team | KPI dashboard |

### 8.4 Kêu gọi đầu tư

**Chúng tôi tìm kiếm:**
- **Vốn hạt giống:** $500.000 – $1.000.000 USD
- **Sử dụng vốn:**
  - 35% — Engineering & Product (team mở rộng)
  - 25% — Infrastructure & Cloud (private cloud VN)
  - 20% — Clinical Validation & Compliance
  - 15% — Business Development & Partnerships
  - 5% — Working Capital

**Cam kết với nhà đầu tư:**
- Milestone 6 tháng: Pilot thành công tại Vinmec với 50+ bác sĩ
- Milestone 12 tháng: 2.500 ca/ngày, unit economics dương
- Milestone 24 tháng: 3+ bệnh viện, sẵn sàng Series A

---

## Phụ lục: Competitive Landscape

| Sản phẩm | Thị trường | Điểm yếu với VN |
|---|---|---|
| Nuance DAX (Microsoft) | US/EU | Không hỗ trợ tiếng Việt, giá rất cao |
| Suki AI | US | Chỉ tiếng Anh, không có VN presence |
| Abridge | US | Chưa có tại VN |
| **Vinmec AI Scribe** | **Vietnam** | **Native Vietnamese, on-premise, Vinmec integration** |

**Lợi thế cạnh tranh bền vững:**
1. **First-mover** trong phân khúc bệnh viện quốc tế tại VN
2. **Tiếng Việt y tế** — không ai có data training tốt hơn đội có partnership với Vinmec
3. **Compliance-native** — thiết kế từ đầu cho quy định VN, không phải retrofit
4. **Distribution moat** — Vinmec có 11 bệnh viện, partnership = instant distribution

---

*Tài liệu này được chuẩn bị bởi đội phát triển Vinmec AI Medical Scribe. Mọi số liệu tài chính là ước tính và có thể thay đổi dựa trên kết quả pilot thực tế. Thông tin trong tài liệu này có tính chất bảo mật.*