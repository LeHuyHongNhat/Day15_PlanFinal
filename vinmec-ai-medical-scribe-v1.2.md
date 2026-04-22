# Vinmec AI Medical Scribe
## Kế hoạch Triển khai & Phân tích Production

> **Phiên bản:** 1.2 | **Ngày:** Tháng 4, 2026 | **Mức độ:** Confidential — Dành cho Nhà đầu tư & Ban Lãnh đạo
>
> **Thay đổi so với v1.1:** Thống nhất số liệu WER (Word Error Rate); giải quyết FPT.AI API (Application Programming Interface) qua DPA (Data Processing Agreement); điều chỉnh timeline fine-tuning STT (Speech-to-Text); bổ sung consent workflow đầy đủ; bổ sung phân bổ trách nhiệm pháp lý; bổ sung challenge code-switching Việt–Anh; kế hoạch adoption bác sĩ chi tiết.

---

## Mục lục

1. [Project Overview](#1-project-overview)
2. [Enterprise Context](#2-enterprise-context)
3. [Deployment Choice](#3-deployment-choice)
4. [Cost Analysis — Minimum Viable Product (MVP) & Growth](#4-cost-analysis--minimum-viable-product-mvp--growth)
5. [Optimization Plan](#5-optimization-plan)
6. [Reliability Plan](#6-reliability-plan)
7. [Future Roadmap](#7-future-roadmap)
8. [Recommendation & Next Steps](#8-recommendation--next-steps)

---

## 1. Project Overview

### 1.1 Bài toán cốt lõi

Vinmec vận hành theo mô hình bệnh viện quốc tế với tiêu chuẩn JCI (Joint Commission International). Mỗi ca khám kéo dài trung bình **15 phút**, nhưng thực tế cho thấy:

| Hoạt động | Thời gian trung bình | % tổng ca khám |
|---|---|---|
| Thăm khám lâm sàng trực tiếp | 9–10 phút | 60–67% |
| Nhập liệu HIS, gõ bệnh án | 4–5 phút | 27–33% |
| Các thao tác hành chính khác | 1 phút | 6–7% |

**Hệ quả:** Bác sĩ mất 30–40% thời gian cho công việc hành chính. Tại một hệ thống có hàng chục nghìn ca khám/năm, đây là tổn thất năng lực khổng lồ.

### 1.2 Giải pháp: AI Medical Scribe

Hệ thống sử dụng AI để **tự động chuyển đổi cuộc hội thoại tự nhiên** giữa bác sĩ và bệnh nhân thành bệnh án có cấu trúc chuẩn, tích hợp thẳng vào HIS.

```
[Phòng khám]
Bác sĩ ↔ Bệnh nhân (hội thoại tự nhiên — tiếng Việt + tiếng Anh y tế xen lẫn)
         ↓
[AI Medical Scribe Pipeline]
Microphone → Speech-to-Text → Natural Language Processing (NLP) / Large Language Model (LLM) Processing → Structured Note
         ↓
[HIS Integration]
SOAP (Subjective, Objective, Assessment, Plan) Note → ICD-10 (International Classification of Diseases, Tenth Revision) Codes → HIS (Hospital Information System) / EMR (Electronic Medical Record) → Bác sĩ xem xét & ký duyệt
```

### 1.3 Người dùng chính

| Nhóm người dùng | Vai trò | Tần suất sử dụng |
|---|---|---|
| **Bác sĩ nội khoa, ngoại khoa** | Primary user — nhận output bệnh án | Mỗi ca khám (~20–40 ca/ngày) |
| **Điều dưỡng** | Secondary user — hỗ trợ review | Theo ca làm việc |
| **Quản trị viên HIS** | Admin — cấu hình, audit | Định kỳ |
| **Ban lãnh đạo Vinmec** | Stakeholder — dashboard tổng quan | Báo cáo tuần/tháng |

### 1.4 Năng lực kỹ thuật đội ngũ (Built in 15 ngày)

**3 kỹ năng cốt lõi đã được kiểm chứng:**

1. **Speech-to-Text (STT) Pipeline** — Tích hợp Whisper với xử lý tiếng Việt; accuracy ~92% cho tiếng Việt phổ thông; ~75–80% cho thuật ngữ chuyên khoa y tế chưa qua fine-tune (xem chi tiết mục 1.5 và 6.6)
2. **LLM Structured Extraction** — Prompt engineering với Claude/GPT-4 để trích xuất SOAP note, ICD-10 coding tự động từ transcript
3. **HIS API (Application Programming Interface) Integration** — Kết nối HIS qua HL7 FHIR (Health Level Seven Fast Healthcare Interoperability Resources) standard, write-back tự động sau khi bác sĩ approve

**Sản phẩm đã xây dựng:**
- MVP demo end-to-end: ghi âm → transcript → SOAP note → hiển thị để bác sĩ duyệt
- Scenario đã test: Khoa Nội tổng quát, bệnh mạn tính (tiểu đường, tăng huyết áp)
- Accuracy: **87% field-level correctness** trên tiếng Việt phổ thông — chưa bao gồm code-switching

### 1.5 Challenge kỹ thuật thực tế: Code-switching Việt–Anh

Đây là thách thức **đặc thù của Vinmec** mà các giải pháp nước ngoài không gặp phải.

Bác sĩ Vinmec thường xuyên nói **code-switching** trong cùng một câu:

> *"Bệnh nhân có history of hypertension, đang dùng amlodipine 5mg, hôm nay complain đau đầu dữ dội, BP lúc vào 160/100."*

**Tại sao đây là vấn đề kỹ thuật nghiêm trọng:**

| Vấn đề | Mô tả | Tác động |
|---|---|---|
| Whisper nhận sai ngôn ngữ | Detect toàn câu là tiếng Việt hoặc Anh, không phải hỗn hợp | Transcribe sai phần còn lại |
| Tên thuốc Anh bị phiên âm Việt | "Amlodipine" → "am lo đi pin" | Lỗi lâm sàng nguy hiểm |
| Thuật ngữ y tế Anh bị bỏ sót | Model ưu tiên tiếng Việt | Mất thông tin lâm sàng |
| LLM nhận transcript lộn xộn | Input chất lượng kém → output kém | Giảm accuracy toàn pipeline |

**Giải pháp theo giai đoạn:**

```
MVP (Tháng 1–3):
  → Whisper với prompt seeding bằng từ vựng y tế song ngữ
  → Post-processing: medical term dictionary Anh–Việt sửa phiên âm sai
  → Cải thiện ước tính: +8–12% accuracy cho thuật ngữ y tế tiếng Anh

Fine-tune (Tháng 6–9):
  → Thu thập audio code-switching thực tế từ pilot
  → Fine-tune Whisper với labeled data song ngữ
  → Mục tiêu: accuracy thuật ngữ y tế ≥ 90%
```

### 1.6 Tại sao chủ đề này phù hợp để phân tích Deployment & Cost?

- **Dữ liệu cực kỳ nhạy cảm** (PHI — Protected Health Information): buộc phải phân tích deployment nghiêm túc
- **Volume dự đoán được**: lịch khám cố định → ước lượng traffic chính xác
- **Chi phí nhiều lớp**: token, STT, storage, compliance, human review
- **Hệ quả lỗi cao**: sai bệnh án → ảnh hưởng trực tiếp đến bệnh nhân

---

## 2. Enterprise Context

### 2.1 Bối cảnh tổ chức

**Vinmec International Hospital Group:**
- **11+ bệnh viện** trên toàn quốc
- **Tiêu chuẩn JCI** — chuẩn quốc tế cao nhất về chất lượng bệnh viện
- **HIS**: Oracle Health (Cerner) hoặc tương đương
- **Quy mô**: ~3.000–5.000 ca ngoại trú/ngày toàn hệ thống

### 2.2 Dữ liệu hệ thống xử lý

| Loại dữ liệu | Mô tả | Mức độ nhạy cảm |
|---|---|---|
| **Audio recording** | Hội thoại bác sĩ–bệnh nhân | ⚠️ Cực cao |
| **Transcript** | Văn bản chuyển đổi từ audio | ⚠️ Cực cao |
| **PHI — Thông tin định danh** | Họ tên, ngày sinh, CMND, BHYT | ⚠️ Cực cao |
| **Chẩn đoán & điều trị** | ICD-10, đơn thuốc, chỉ định | ⚠️ Cực cao |
| **Lịch sử bệnh** | Tiền sử, dị ứng thuốc | ⚠️ Cao |
| **Metadata hệ thống** | Thời gian, bác sĩ, phòng khám | 🔶 Trung bình |
| **Audit logs** | Ai truy cập, chỉnh sửa gì, khi nào | 🔶 Trung bình |

> Toàn bộ dữ liệu cốt lõi thuộc nhóm PHI — bảo vệ bởi Nghị định 13/2023/NĐ-CP, Thông tư 09/2023/TT-BYT và JCI.

### 2.3 Ba ràng buộc Enterprise lớn nhất

#### Ràng buộc 1: Data Sovereignty & Compliance

- Dữ liệu bệnh nhân phải lưu trong **lãnh thổ Việt Nam**
- Tuân thủ Thông tư 09/2023/TT-BYT về hồ sơ bệnh án điện tử
- **Audit trail bắt buộc** — ai sửa gì, lúc nào
- Không dùng dữ liệu thật để train model bên thứ ba mà không có consent

**Biện pháp cụ thể:**
- Audio xóa sau **24 giờ** kể từ khi bệnh án được ký duyệt
- Mã hóa **AES-256** at rest và in transit
- Audit log **immutable**, lưu tối thiểu **10 năm**
- FPT.AI STT API: ký **Data Processing Agreement (DPA — Thỏa thuận xử lý dữ liệu)** ràng buộc pháp lý (xem mục 3.4)

#### Ràng buộc 2: HIS Integration & Approval Workflow

- Tích hợp HIS qua **HL7 FHIR v4**
- Bác sĩ **review và ký điện tử** trước khi lưu chính thức
- Không ghi đè bệnh án khi chưa có xác nhận
- Đồng bộ real-time với Dược, Xét nghiệm, Chẩn đoán hình ảnh

#### Ràng buộc 3: Uptime & Clinical Availability

- **SLA 99.9% uptime** trong giờ khám (Thứ 2–Thứ 6: 8h–12h, 13h–17h; Thứ 7: 8h–12h)
- Downtime > 5 phút trong giờ cao điểm = ảnh hưởng lâm sàng
- Fallback rõ ràng: tệ nhất là nhập tay như cũ, không được tắc nghẽn
- Latency: **< 30 giây** từ kết thúc hội thoại đến hiển thị draft

### 2.4 Câu hỏi quan trọng và câu trả lời

| Câu hỏi | Câu trả lời |
|---|---|
| Có cần audit trail không? | **Bắt buộc.** Mọi thao tác đều phải log |
| Dữ liệu có được rời khỏi tổ chức không? | **Có giới hạn.** FPT.AI được chấp nhận (data center VN + DPA). LLM API chỉ nhận text đã de-identify |
| Cần tích hợp hệ thống cũ không? | **Có.** HIS qua HL7 FHIR, có approval workflow |
| Nếu AI sai, ai chịu trách nhiệm? | Xem mục 2.6 — phân bổ theo từng tầng |

### 2.5 Consent Workflow — Xử lý đầy đủ mọi tình huống

Đây là điểm JCI audit và legal review kiểm tra đầu tiên.

#### Luồng consent tiêu chuẩn

```
Bệnh nhân đến quầy tiếp nhận
        ↓
Màn hình hiển thị:
  "Vinmec sử dụng AI hỗ trợ ghi chép bệnh án.
   Cuộc hội thoại trong phòng khám sẽ được ghi âm
   và xử lý tự động. Bác sĩ ký duyệt trước khi lưu."
        ↓
       ┌─────────────┐
    Đồng ý        Không đồng ý
       ↓                ↓
   Ký số (OTP/    AI Scribe TẮT hoàn toàn
   vân tay)       Bác sĩ nhập tay như cũ
       ↓          (không ảnh hưởng quyền được khám)
Consent lưu HIS
AI Scribe kích hoạt
```

#### Xử lý trường hợp đặc biệt

| Tình huống | Xử lý |
|---|---|
| **Bệnh nhân không biết chữ** | Nhân viên đọc to, bệnh nhân điểm chỉ — lưu ảnh chụp |
| **Không có smartphone/OTP** | Ký tay trên tablet hoặc ký giấy scan |
| **Hôn mê / không tỉnh táo** | AI Scribe tắt mặc định. Người giám hộ pháp lý có thể consent thay |
| **Trẻ em dưới 18 tuổi** | Cha/mẹ hoặc người giám hộ consent |
| **Bệnh nhân nước ngoài** | Thông báo consent hiển thị song ngữ Việt–Anh |

#### Quy trình rút lại consent

```
Trong ca khám:
  → Dừng ghi âm ngay lập tức
  → Audio đã thu được xóa khỏi bộ nhớ
  → Ghi nhận thời điểm rút consent vào audit log

Sau khi khám (trong 30 ngày):
  → Bệnh nhân liên hệ IT Vinmec
  → Audio xóa trong 72 giờ (nếu còn trong retention)
  → SOAP note đã ký duyệt: KHÔNG xóa (hồ sơ y tế pháp lý)
  → Ghi chú "Audio đã xóa theo yêu cầu" vào hồ sơ
```

> **Quan trọng:** SOAP note đã ký duyệt là hồ sơ y tế pháp lý — không thể xóa. Chỉ audio và transcript thô có thể xóa theo yêu cầu.

#### Consent riêng cho AI training

Tách **hoàn toàn** khỏi consent khám bệnh, không được gộp chung:

```
[Màn hình riêng, sau consent khám]

"Bạn có muốn đóng góp dữ liệu ẩn danh để
 cải thiện AI tại Vinmec không?

 ✓ Xóa toàn bộ thông tin cá nhân trước khi dùng
 ✓ Chỉ dùng để cải thiện AI tại Vinmec
 ✗ Không chia sẻ với bên thứ ba"

[Đồng ý]  [Không đồng ý ← Mặc định]
```

### 2.6 Phân bổ trách nhiệm pháp lý khi AI sai

Cần được thiết lập trong hợp đồng **trước khi go-live**.

**Nguyên tắc cơ bản:** Bác sĩ ký duyệt = chịu trách nhiệm toàn bộ nội dung bệnh án. AI là công cụ hỗ trợ hành chính, không phải công cụ chẩn đoán.

| Tình huống | Trách nhiệm |
|---|---|
| Bác sĩ ký mà không kiểm tra | **Bác sĩ** |
| Field đỏ có cảnh báo, bác sĩ bỏ qua | **Bác sĩ** |
| AI sai ở field không có cảnh báo, bác sĩ tin tưởng | **Chia sẻ** — cần điều khoản hợp đồng |
| Lỗi kỹ thuật hệ thống (bug, corruption) | **Đội phát triển / IT Vinmec** |

**Biện pháp phòng ngừa rủi ro pháp lý cho đội phát triển:**

1. **Disclaimer bắt buộc trong UI:**
```
⚠️ ĐÂY LÀ BẢN NHÁP DO AI TẠO RA
Bác sĩ có trách nhiệm xem xét và xác nhận toàn bộ nội dung.
AI có thể mắc lỗi, đặc biệt với tên thuốc và liều lượng.
```

2. **Field tên thuốc / liều lượng:** Bác sĩ phải click "Xác nhận" riêng — không thể ký duyệt toàn bộ mà bỏ qua.

3. **Audit log chi tiết:** Ghi thời gian bác sĩ mở SOAP note và thời gian ký. Ký trong < 10 giây cho SOAP note dài = bằng chứng không review kỹ.

4. **Điều khoản hợp đồng với Vinmec:** *"AI Medical Scribe là công cụ hỗ trợ hành chính. Vinmec và bác sĩ chịu trách nhiệm cuối cùng về nội dung bệnh án."*

---

## 3. Deployment Choice

### 3.1 Quyết định: Hybrid Deployment

```
┌──────────────────────────────────────────────────────────────┐
│                     HYBRID ARCHITECTURE                       │
│                                                              │
│  ┌─────────────────────┐     ┌──────────────────────────┐   │
│  │  ON-PREMISES (Core) │     │  FPT.AI / VNG Cloud (VN) │   │
│  │                     │     │                          │   │
│  │ • PHI Data Storage  │     │ • STT (FPT.AI Whisper)   │   │
│  │ • SOAP Notes final  │◄───►│ • LLM API Gateway        │   │
│  │ • HIS Integration   │     │ • Auto-scaling nodes     │   │
│  │ • Audit Logs        │     │ • Load Balancing         │   │
│  │ • Consent Records   │     │                          │   │
│  └─────────────────────┘     └──────────────────────────┘   │
│          ▲                              ▲                    │
│    [AES-256 Encrypted]         [VPN / Private Link]         │
│    [Vinmec Data Center]        [Lãnh thổ Việt Nam]          │
│                                [DPA ràng buộc pháp lý]      │
└──────────────────────────────────────────────────────────────┘
```

### 3.2 Hai lý do chọn Hybrid

**Lý do 1: Kiểm soát dữ liệu PHI tuyệt đối ở lớp lưu trữ**
Toàn bộ PHI, bệnh án hoàn chỉnh, audit log và consent records lưu on-premises tại Vinmec — không ra ngoài theo bất kỳ điều kiện nào.

**Lý do 2: AI inference cần elasticity, data sovereignty vẫn đảm bảo qua DPA**
FPT.AI và VNG Cloud có data center tại Việt Nam. Với DPA ràng buộc pháp lý (không lưu audio sau xử lý, không dùng cho mục đích khác, right-to-audit), đây là phương án cân bằng tốt — rẻ hơn và linh hoạt hơn so với on-prem thuần túy phải duy trì GPU server cho peak traffic.

### 3.3 So sánh các phương án

| Tiêu chí | Cloud quốc tế thuần | On-prem thuần | **Hybrid VN (Chọn)** |
|---|---|---|---|
| Tuân thủ pháp lý VN | ❌ Rủi ro cao | ✅ Tốt | ✅ Tốt (DPA) |
| Chi phí ban đầu | Thấp | Cao | Trung bình |
| Khả năng scale | ✅ Tốt | ❌ Kém | ✅ Tốt |
| Kiểm soát dữ liệu | ❌ Kém | ✅ Tốt | ✅ Tốt |
| Tốc độ triển khai | Nhanh | Chậm | Trung bình |
| Uptime & reliability | ✅ Tốt | ⚠️ Phụ thuộc team | ✅ Tốt |
| Khả năng xử lý tranh chấp pháp lý | ❌ Rất khó | ✅ Dễ | ✅ Dễ (cùng lãnh thổ) |

### 3.4 Data Security Architecture — Luồng dữ liệu chi tiết

```
[Phòng khám — microphone]
Audio (in-memory, AES-256)
        ↓ TLS 1.3
[FPT.AI STT API — Data center Hà Nội/TP.HCM]
  → Xử lý audio → trả về transcript text
  → Audio KHÔNG lưu lại (theo DPA: zero retention)
        ↓
[On-premises — Vinmec server]
Transcript (có PHI: tên, số, địa chỉ)
        ↓
[De-identification Layer — chạy on-prem]
  → Xóa: họ tên, CMND, số điện thoại, địa chỉ
  → Giữ: triệu chứng, thuốc, chẩn đoán (vô danh)
        ↓ Text đã de-identify
[LLM API — Claude / GPT-4]
  → Tạo SOAP note draft
  → Nếu provider log: chỉ thấy text vô danh
        ↓
[Re-identification Layer — chạy on-prem]
  → Ghép tên bệnh nhân từ HIS (theo session ID)
        ↓
[SOAP Note hoàn chỉnh — lưu on-premises]
Bác sĩ review → ký duyệt → lưu vào HIS
```

**Điều khoản DPA tối thiểu cần có với FPT.AI:**
- Zero retention: không lưu audio sau khi trả transcript
- Không dùng audio/transcript cho bất kỳ mục đích khác
- Cho phép Vinmec audit log xử lý bất kỳ lúc nào
- Data center đặt tại Việt Nam
- Thông báo trong 24h nếu xảy ra breach
- Điều khoản phạt vi phạm theo luật VN

---

## 4. Cost Analysis — Minimum Viable Product (MVP) & Growth

### 4.1 Giả định đầu vào

| Tham số | MVP (Phase 1) | Growth 5x | Growth 10x |
|---|---|---|---|
| Bệnh viện | 1 (Vinmec Times City) | 5 bệnh viện | 11 bệnh viện |
| Bác sĩ active/ngày | 50 | 250 | 500 |
| Ca khám có AI/ngày | 500 | 2.500 | 5.000 |
| Peak concurrent requests | 30 | 150 | 300 |
| Audio duration trung bình | 15 phút | 15 phút | 15 phút |
| Input tokens/request | ~8.000 | ~8.000 | ~8.000 |
| Output tokens/request | ~1.500 | ~1.500 | ~1.500 |

### 4.2 Chi phí MVP — Phân tích theo lớp

#### Layer 1: AI/LLM Inference Cost

```
Speech-to-Text (FPT.AI Whisper API — data center VN):
  500 ca × 15 phút × $0.0297/phút = $222.75/ngày → ~$6.683/tháng
  [Có thể đàm phán giảm 15–20% với volume contract]

LLM Structured Extraction (Claude Sonnet — text đã de-identify):
  500 ca × 8.000 input tokens × $3/1M = $12/ngày
  500 ca × 1.500 output tokens × $15/1M = $11.25/ngày
  Total LLM: ~$23/ngày → ~$700/tháng

Tổng AI inference: ~$7.383/tháng
```

#### Layer 2: Compute & Infrastructure

```
Private Cloud VN (VNG/FPT Cloud):
  API Gateway + Orchestration: ~$300/tháng
  AI Processing nodes (auto-scale): ~$400/tháng
  Network egress/ingress: ~$50/tháng

On-premises (amortized 3 năm):
  Server hardware: ~$500/tháng
  Maintenance & IT ops: ~$300/tháng

Tổng Compute: ~$1.550/tháng
```

#### Layer 3: Storage

```
Audio (retention 24h — rolling buffer ~2 ngày):
  500 ca × 15 phút × ~15MB/phút = ~112GB/ngày → ~224GB buffer
  Storage cost: ~$6/tháng

Transcripts & SOAP notes (dài hạn):
  ~5GB/tháng → ~$1/tháng

Audit logs (immutable, 10 năm):
  ~2GB/tháng × 120 tháng = ~240GB total → ~$6/tháng

Backup (cold storage): ~$20/tháng

Tổng Storage: ~$33/tháng
```

> **Lưu ý:** Storage giảm mạnh so với v1.1 ($120 → $33) vì audio chỉ lưu 24h, không phải 30 ngày. Ít rủi ro hơn và rẻ hơn đồng thời.

#### Layer 4: Human Review & QA

```
Medical reviewer (QA sampling 5% cases + review field đỏ):
  25 ca/ngày × 3 phút/ca = 1.25 giờ/ngày
  ~$15/giờ → ~$560/tháng

Tổng Human Review: ~$560/tháng
```

#### Layer 5: Logging, Monitoring & Compliance

```
Logging self-hosted on-prem (ELK Stack): ~$100/tháng
APM & Monitoring (Grafana + Prometheus): ~$80/tháng
Security scanning & compliance audit: ~$200/tháng
DPA audit support (FPT.AI): ~$50/tháng

Tổng: ~$430/tháng
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
| AI/LLM Inference | $7.383 | $36.900 | $73.800 |
| Compute & Infrastructure | $1.550 | $3.800 | $6.500 |
| Storage | $33 | $100 | $200 |
| Human Review | $560 | $1.400 | $2.200 |
| Logging & Monitoring | $430 | $650 | $950 |
| Maintenance | $1.300 | $2.000 | $2.800 |
| **TỔNG** | **~$11.256/tháng** | **~$44.850/tháng** | **~$86.450/tháng** |
| **Chi phí/ca khám** | **~$0.75** | **~$0.60** | **~$0.57** |

> **Cost driver:** AI/LLM Inference = ~65% tổng cost. Trong đó STT (FPT.AI) = ~91% AI cost — đây là mục tiêu tối ưu chính ở Phase 2.

### 4.4 Hidden Cost Analysis

| Hidden Cost | Mô tả | Ước tính |
|---|---|---|
| **Change Management + Adoption** | Training, tài liệu, support person 3 tháng đầu (xem mục 8.3) | $15.000–$30.000 one-time |
| **Legal & Compliance** | Soạn DPA, hợp đồng Vinmec, review consent | $15.000–$25.000/năm |
| **Integration Testing** | Test HIS, edge cases lâm sàng, code-switching | $10.000–$30.000 one-time |
| **Data Labeling (Tháng 6–9)** | ~100 giờ audio × $3/phút transcription | $18.000 |
| **Incident Response** | On-call, xử lý lỗi bệnh án | ~$500/tháng |
| **Model Drift Monitoring** | Accuracy giảm theo thời gian nếu không monitor | Tích hợp MLOps |

---

## 5. Optimization Plan

### 5.1 Ba chiến lược tối ưu được chọn

#### Chiến lược 1: Model Routing (Làm ngay — Phase 1)

**Tiết kiệm:** 40–60% chi phí LLM inference

```
Request đến → [Classifier]
    │
    ├──► Đơn giản (<5 phút, 1 chẩn đoán)
    │    → Claude Haiku / GPT-3.5 Turbo (~$0.003/ca)
    │
    └──► Phức tạp (đa bệnh lý, >10 phút, nhiều thuốc)
         → Claude Sonnet / GPT-4 (~$0.023/ca)
```

**Trade-off:** Cần xây classifier và validate định nghĩa "phức tạp" với bác sĩ lâm sàng.

**Thời điểm:** Ngay từ MVP.

---

#### Chiến lược 2: Prompt Compression (Làm ngay — Phase 1)

**Tiết kiệm:** 20–35% chi phí input tokens

Preprocessing transcript thô (~8.000 tokens) → loại bỏ filler words, noise không lâm sàng → giảm xuống ~5.000 tokens, tiết kiệm ~37% input cost.

**Trade-off:** Nguy cơ mất thông tin nếu compression quá agressive. Test kỹ trên diverse cases trước khi deploy.

---

#### Chiến lược 3: Self-hosted STT sau fine-tune (Phase 2 — Tháng 6–9)

**Tiết kiệm:** 60–80% chi phí STT (~$20.000–$26.000/tháng ở scale 5x)

**Lộ trình thực tế:**

```
Tháng 1–3:   Dùng FPT.AI API + thu thập audio labeled từ pilot
Tháng 3–6:   Tích lũy 50+ giờ audio labeled chất lượng cao
              (bác sĩ/điều dưỡng sửa transcript = training data)
Tháng 6–9:   Fine-tune Whisper large-v3 với corpus Vinmec
              Tập trung: code-switching Việt–Anh + tên thuốc
              Deploy self-hosted trên VNG/FPT Cloud GPU
              A/B test → switch khi WER < 10% cho thuật ngữ y tế
```

**Chi phí fine-tuning one-time:**
- GPU cloud cho training: ~$500–$1.500
- Data labeling (100 giờ): ~$18.000
- MLOps setup: ~$3.000
- **Tổng: ~$22.000–$23.000**
- **Break-even: < 1 tháng** sau khi switch ở scale 5x

> **Quan trọng:** Không fine-tune trước khi có đủ data thực tế từ Vinmec. Fine-tune trên data không đại diện sẽ không sửa đúng loại lỗi cần thiết.

---

### 5.2 Roadmap tối ưu hóa

```
Phase 1 (Tháng 1–3): Model Routing + Prompt Compression
  → Tiết kiệm: 35–45% LLM cost
  → Investment: ~$2.000

Phase 2 (Tháng 6–9): Self-hosted STT fine-tuned
  → Tiết kiệm: 60–80% STT cost
  → Investment: ~$22.000 one-time

Phase 3 (Tháng 10+): Semantic Caching cho ca tái khám
  → Bệnh mạn tính tái khám định kỳ → cache template
  → Tiết kiệm: 10–15% LLM calls
```

---

## 6. Reliability Plan

### 6.1 Ba tình huống rủi ro và phản ứng

#### Tình huống 1: Traffic tăng đột biến

**Bối cảnh:** Sáng thứ Hai sau lễ, lượng bệnh nhân tăng 3x. Hoặc chiến dịch tầm soát quy mô lớn.

**Tác động:** Bác sĩ đợi 2–5 phút → bỏ qua AI, nhập tay.

**Ngắn hạn:** Auto-scaling khi CPU > 70%; priority queue (cấp cứu > nội trú > ngoại trú); graceful degradation tắt ICD-10 suggestion giữ SOAP core.

**Dài hạn:** Predictive scaling từ lịch đặt khám (dự báo 24–48h); capacity planning hàng quý.

---

#### Tình huống 2: Provider Timeout

**Bối cảnh:** FPT.AI hoặc LLM API timeout / 5xx error trong 10–30 phút.

**Ngắn hạn:** Retry exponential backoff (3 lần / 30 giây); STT failover sang self-hosted Whisper; LLM failover Claude → GPT-4 → Gemini Pro (< 5 giây).

**Dài hạn:** Multi-provider contract; self-hosted Llama 3 làm fallback cuối; alert khi error rate > 1%.

---

#### Tình huống 3: Response Chậm

**Bối cảnh:** Latency tăng lên 60–90 giây do mạng nội bộ hoặc inference bị throttle.

**Ngắn hạn:** Progress indicator real-time từng bước; bác sĩ tiếp tục ca tiếp theo trong khi AI xử lý async; notify khi SOAP note sẵn sàng.

**Dài hạn:** Streaming response (hiển thị từng phần); pre-processing STT trong quá trình hội thoại; P95 latency target < 25 giây.

---

### 6.2 Fallback Decision Timeline

```
T+0s    Request gửi đi
T+5s    Chưa có response → "Đang xử lý..."
T+10s   Circuit breaker → switch Level 1 (< 3 giây)
T+15s   🟡 Banner vàng: "Đang dùng chế độ dự phòng"

Nếu Level 1 fail:
T+25s   Switch Level 2 (template-based)
T+30s   🟠 Banner cam: "AI hạn chế — vui lòng bổ sung thêm"

Nếu Level 2 fail:
T+40s   Switch Level 3 (manual)
T+40s   🔴 Banner đỏ: "AI tạm ngưng — nhập liệu thủ công"
```

Bác sĩ **luôn biết trạng thái hệ thống**. Workflow không bao giờ bị tắc — tệ nhất là nhập tay như cũ.

### 6.3 Real-time vs. Async

| Request | Xử lý | Lý do |
|---|---|---|
| STT transcription | Streaming real-time | Bác sĩ cần thấy AI đang nghe |
| SOAP Note generation | Async + notify | Chấp nhận 30–60 giây |
| ICD-10 suggestion | Async | Review sau SOAP note |
| HIS write-back | Async + confirmation | Không cần real-time, cần đảm bảo thành công |
| Audit log | Async fire-and-forget | Không ảnh hưởng UX |

### 6.4 Metrics cần monitor

| Metric | Target | Alert threshold |
|---|---|---|
| STT latency P95 (95th Percentile) | < 45 giây / 15-phút audio | > 90 giây |
| LLM latency P95 (95th Percentile) | < 20 giây | > 40 giây |
| End-to-end latency P95 (95th Percentile) | < 30 giây | > 60 giây |
| API error rate | < 0.5% | > 2% |
| SOAP note accuracy (sampling) | > 85% | < 80% |
| **Tên thuốc accuracy (critical)** | **> 98%** | **< 95%** |
| HIS write-back success rate | > 99.9% | < 99% |
| System uptime (T2–T6: 8h–12h, 13h–17h; T7: 8h–12h) | > 99.9% | < 99.5% |
| Fallback detection time | < 10 giây | > 15 giây |
| Provider switch time | < 3 giây | > 5 giây |
| Consent completion rate | > 95% | < 90% |

### 6.5 Fallback Proposal

```
Level 1 — Secondary provider:
  Trigger: Primary timeout > 10 giây
  Action: Route tự động sang secondary LLM (< 3 giây)
  UX: Banner vàng | Chất lượng: Thấp hơn 10–15%

Level 2 — Template-based:
  Trigger: Tất cả LLM fail
  Action: SOAP template + keyword extraction từ transcript
  UX: Banner cam | Bác sĩ điền thêm

Level 3 — Manual:
  Trigger: STT fail / audio quality quá kém
  Action: HIS input thủ công
  UX: Banner đỏ | Như trước khi có AI
```

### 6.6 STT Accuracy — Medical Safety Layer

**Tại sao 92% word-level accuracy không đủ cho y tế:**

```
500 ca/ngày × 1.500 từ/ca = 750.000 từ/ngày
8% lỗi = ~60.000 từ sai/ngày
0.1% rơi vào tên thuốc/liều = ~60 lỗi lâm sàng nguy hiểm/ngày
```

**WER thực tế của Whisper (đã được kiểm chứng độc lập):**

| Loại nội dung | WER baseline | Sau vocabulary boost (MVP) | Sau fine-tune (Tháng 6–9) |
|---|---|---|---|
| Tiếng Việt phổ thông | ~8–12% | ~8–12% | ~5–8% |
| Tiếng Việt có accent | ~12–18% | ~10–15% | ~8–12% |
| Thuật ngữ y tế tiếng Việt | ~20–25% | ~12–18% | ~8–12% |
| Tên thuốc tiếng Anh | ~15–20% | ~5–8% | ~3–5% |
| **Code-switching Việt–Anh** | **~25–35%** | **~15–20%** | **~8–12%** |

> **Không có con số "92% accuracy cho thuật ngữ chuyên khoa" ở giai đoạn MVP.** 92% là word-level accuracy cho tiếng Việt phổ thông — không áp dụng cho nội dung y tế hoặc code-switching.

**Medical Safety Layer:**

```
Transcript thô
        ↓
[Medical NER — nhận diện thực thể y tế]
        ↓
┌─────────────────┬─────────────────┬─────────────────┐
│ 🔴 CRITICAL     │ 🟡 MEDIUM       │ 🟢 LOW          │
│ Tên thuốc       │ Triệu chứng     │ Thông tin HC    │
│ Liều lượng      │ Tiền sử bệnh    │ Thời gian khám  │
│ Đường dùng      │ Dị ứng          │ Ghi chú phụ     │
│ Chẩn đoán chính │                 │                 │
└─────────────────┴─────────────────┴─────────────────┘
        ↓
[Drug Database Validation — đối chiếu danh mục Vinmec]
  Confidence < 85% → highlight đỏ, bắt bác sĩ confirm thủ công
  Confidence ≥ 85% → hiển thị bình thường
        ↓
SOAP Note: Đỏ = phải xác nhận | Vàng = cần chú ý | Xanh = đã validate
```

**Accuracy sau Medical Safety Layer:**

| Field | STT accuracy (MVP) | Sau Safety Layer |
|---|---|---|
| Tên thuốc phổ biến | ~80–85% | ~98–99% |
| Liều lượng | ~82–87% | ~97–98% |
| Thuật ngữ chuyên khoa | ~75–80% | ~90–94% |
| Thông tin thông thường | ~90–95% | ~90–95% |

---

## 7. Future Roadmap

### 7.1 Lộ trình mở rộng 3 năm

```
2026 — Phase 1: MVP (Minimum Viable Product) & Validation
  1 bệnh viện (Vinmec Times City) | 50 bác sĩ | 500 ca/ngày
  KPI (Key Performance Indicator): Giảm 25% thời gian nhập liệu | NPS (Net Promoter Score) bác sĩ > 7/10
  Tech milestone: WER thuật ngữ y tế < 15% sau vocabulary boost

2027 — Phase 2: Scale & Specialization
  5 bệnh viện | Tim mạch, Nhi, Sản | 2.500 ca/ngày
  Self-hosted STT fine-tuned | Tích hợp PACS, Lab
  KPI: ROI (Return on Investment) dương | Chi phí/ca < $0.40

2028 — Phase 3: AI-Augmented Clinical Intelligence
  11 bệnh viện + license model cho bệnh viện tư VN và SEA
  Predictive alerts | Clinical decision support | Multi-modal
  KPI: 5.000+ ca/ngày | Sẵn sàng commercialize
```

### 7.2 Cơ hội mở rộng thị trường

| Thị trường | Quy mô ước tính | Rào cản |
|---|---|---|
| Vinmec toàn hệ thống | ~5.000 ca/ngày | Thấp |
| Bệnh viện tư nhân VN | ~50.000 ca/ngày | Trung bình |
| Bệnh viện công VN | ~500.000 ca/ngày | Cao |
| SEA (Thailand, Indonesia) | ~1M+ ca/ngày | Cao |

### 7.3 Các hướng AI mở rộng

- **Predictive Readmission:** Dự báo nguy cơ tái nhập viện trong 30 ngày
- **Drug Interaction Check:** Cảnh báo tương tác thuốc tự động
- **Billing Optimization:** Gợi ý DRG code tối ưu cho bảo hiểm
- **Population Health Analytics:** Insights từ patterns bệnh lý toàn hệ thống
- **Patient Communication:** Soạn hướng dẫn sau khám bằng ngôn ngữ đơn giản

---

## 8. Recommendation & Next Steps

### 8.1 Tóm tắt Executive

| Hạng mục | Chi tiết |
|---|---|
| **Giải pháp** | AI Medical Scribe — tự động hóa bệnh án từ hội thoại |
| **Deployment** | Hybrid: On-prem (PHI) + Private Cloud VN với DPA (AI inference) |
| **Chi phí MVP** | ~$11.256/tháng cho 500 ca/ngày |
| **Chi phí/ca** | ~$0.75 (MVP) → ~$0.57 (scale 10x) |
| **Break-even** | 6–9 tháng |
| **Rủi ro lớn nhất** | Code-switching Việt–Anh + WER thuật ngữ y tế + adoption bác sĩ |

### 8.2 ROI Analysis — 3 góc nhìn

> **Lưu ý:** "Tiết kiệm thời gian" không tự động thành tiền. ROI thực sự đến từ 3 nguồn:

**Góc 1: Doanh thu tăng từ ca khám thêm**

```
Tiết kiệm ~4 phút/ca → bác sĩ khám thêm 1–2 ca/ngày

50 bác sĩ × 1 ca/ngày × $30–80/ca (range giá Vinmec)
= $1.500–$4.000/ngày
= ~$540.000–$1.440.000/năm

Chi phí hệ thống: ~$135.000/năm
ROI năm 1: ~300–970%
```

**Góc 2: Giảm rủi ro pháp lý**

```
1 vụ kiện y tế do lỗi nhập tay: $50.000–$500.000
AI Scribe giảm lỗi nhập liệu ước tính 60–70%
→ "Insurance value" hàng trăm nghìn đô/năm
```

**Góc 3: Tối ưu bảo hiểm claim**

```
Bệnh án chuẩn ICD-10 → claim approval rate tăng ~5%
Doanh thu bảo hiểm ~$10M/năm → thêm ~$500.000/năm
```

**Bảng ROI tổng hợp:**

| Nguồn giá trị | Ước tính/năm | Độ chắc chắn |
|---|---|---|
| Doanh thu ca khám tăng thêm | ~$540.000–$1.440.000 | Cao |
| Giảm rủi ro pháp lý | ~$100.000–$500.000 | Trung bình |
| Tối ưu bảo hiểm claim | ~$200.000–$500.000 | Trung bình |
| **Tổng giá trị tạo ra** | **~$840.000–$2.440.000/năm** | |
| **Chi phí hệ thống** | **~$135.000/năm** | Cao |
| **ROI thực tế** | **~520–1.700%** | |

### 8.3 Kế hoạch Adoption — Rủi ro số một của dự án

Bác sĩ là nhóm người dùng khó thay đổi hành vi nhất: bận, hoài nghi với công nghệ, và sẽ quay về nhập tay ngay khi AI sai một lần đáng kể. Hệ thống tốt đến đâu cũng thất bại nếu không có adoption plan bài bản.

**Nguyên tắc:** Không ép buộc. Không "big bang rollout". Bắt đầu với early adopter tự nguyện.

```
2 tuần trước go-live: "AI Champion" program
  → Chọn 3–5 bác sĩ có hứng thú với công nghệ
  → Đào tạo sâu, thu thập feedback sớm
  → Họ trở thành người hướng dẫn đồng nghiệp

Tuần 1–4: Soft launch, song song với nhập tay
  → AI Scribe là "gợi ý", bác sĩ vẫn nhập tay như cũ
  → Feedback hàng ngày, fix nhanh

Tháng 2–3: Mở rộng tự nguyện
  → Báo cáo tuần: "Bác sĩ A tiết kiệm 45 phút/ngày"
  → Peer pressure tích cực, không ép buộc

Tháng 4+: Tích hợp vào workflow chính thức
  → Đo NPS bác sĩ mỗi tháng
  → NPS > 7/10 là điều kiện để mở rộng sang bệnh viện tiếp theo
```

**Budget thực tế:** $15.000–$30.000 one-time (training, tài liệu, support person 3 tháng).

### 8.4 Next Steps — 90 ngày đầu tiên

| Tuần | Hành động | Owner | Deliverable |
|---|---|---|---|
| 1–2 | Ký MOU với Vinmec | Business Dev | MOU document |
| 1–3 | Soạn & ký DPA với FPT.AI + hợp đồng trách nhiệm pháp lý | Legal | DPA + Contract |
| 2–4 | Setup hạ tầng hybrid, VPN với HIS | DevOps | Infrastructure ready |
| 2–4 | Design & test consent workflow (kể cả từ chối, rút consent) | Product | Consent flow QA |
| 3–6 | Tích hợp HIS (HL7 FHIR), test sandbox | Backend Dev | API integration |
| 4–6 | Build Medical Safety Layer (NER + Drug DB validation) | ML Team | Safety layer ready |
| 4–6 | Build medical term dictionary Anh–Việt (code-switching) | ML Team | Vocabulary boost |
| 6–8 | Pilot với 5 bác sĩ "AI Champion" | Product + Clinical | User feedback report |
| 6–8 | Bắt đầu thu thập audio labeled (consent training riêng) | ML Team | 10+ giờ labeled data |
| 8–12 | Đo WER thực tế, xác định loại lỗi ưu tiên | ML + Clinical | WER baseline report |
| 10–14 | Mở rộng 20 bác sĩ, đo NPS và KPI | Full team | KPI dashboard |
| 12–16 | Review go/no-go cho Phase 2 dựa trên KPI thực tế | Leadership | Phase 2 decision |

### 8.5 Kêu gọi đầu tư

**Chúng tôi tìm kiếm:** $500.000 – $1.000.000 USD vốn hạt giống

**Sử dụng vốn:**
- 35% — Engineering & Product (team, MLOps)
- 25% — Infrastructure & Cloud (private cloud VN, GPU fine-tune)
- 20% — Clinical Validation, Compliance & Legal
- 15% — Business Development, Adoption & Partnerships
- 5% — Working Capital

**Milestones:**
- **6 tháng:** 20+ bác sĩ pilot, NPS > 7/10, WER thuật ngữ y tế < 15%
- **12 tháng:** 2.500 ca/ngày, self-hosted STT, unit economics dương
- **24 tháng:** 3+ bệnh viện, sẵn sàng Series A

---

## Phụ lục A: Competitive Landscape

| Sản phẩm | Thị trường | Điểm yếu với VN |
|---|---|---|
| Nuance DAX (Microsoft) | US/EU | Không hỗ trợ tiếng Việt, không có on-prem VN |
| Suki AI | US | Chỉ tiếng Anh, không có VN presence |
| Abridge | US | Chưa có tại VN |
| **Vinmec AI Scribe** | **Vietnam** | **Native Vietnamese + code-switching, data stays in VN, JCI-compliant** |

**Lợi thế cạnh tranh bền vững:**
1. **First-mover** trong phân khúc bệnh viện quốc tế tại VN
2. **Tiếng Việt y tế + code-switching** — không ai có data tốt hơn khi có partnership Vinmec
3. **Compliance-native** — thiết kế từ đầu cho DPA, Thông tư 09/2023, JCI
4. **Distribution moat** — 11 bệnh viện Vinmec = instant distribution

---

## Phụ lục B: Thay đổi so với v1.1

| Vấn đề trong v1.1 | Điều chỉnh trong v1.2 | Lý do |
|---|---|---|
| WER "92% cho thuật ngữ y tế" | Sửa thành ~75–80%; thêm bảng WER chi tiết có phân loại | Số cũ sai thực tế, gây mất tin cậy với investor kỹ thuật |
| FPT.AI API mâu thuẫn với kiến trúc | Giữ FPT.AI + thêm DPA ràng buộc pháp lý + giải thích data center VN | FPT.AI hợp lý và thực tế; cần giải thích đúng thay vì loại bỏ |
| Fine-tune STT: tuần 6–10 | Dời sang Tháng 6–9; thêm lý do (cần data trước) | Không thể fine-tune khi chưa có data đủ |
| Consent: chỉ trường hợp lý tưởng | Bổ sung từ chối, rút consent, bệnh nhân đặc biệt, consent training riêng | JCI và pháp lý VN yêu cầu xử lý đầy đủ |
| Trách nhiệm pháp lý khi AI sai | Thêm mục 2.6 với phân bổ theo từng tình huống + biện pháp phòng ngừa | Rủi ro pháp lý cho chính đội phát triển nếu không làm rõ |
| Code-switching: không đề cập | Thêm mục 1.5 + bổ sung vào WER table và pipeline | Thực tế đặc thù Vinmec, bỏ qua = thiếu chuyên nghiệp |
| Change management: $5k–$15k | Tăng lên $15k–$30k + kế hoạch adoption chi tiết (mục 8.3) | Adoption là rủi ro số một, không thể xử lý bằng budget nhỏ |
| Storage: $120/tháng (30 ngày) | Giảm xuống $33/tháng (24h retention) | Nhất quán với quyết định xóa audio sau 24h |

---

*Tài liệu này được chuẩn bị bởi đội phát triển Vinmec AI Medical Scribe. Mọi số liệu tài chính là ước tính và có thể thay đổi dựa trên kết quả pilot. Thông tin có tính chất bảo mật.*
