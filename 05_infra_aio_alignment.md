# Hạ tầng CDO 5 (bản xét lại) — Host image của đội AI bằng Fargate + Lambda

> Phương án hạ tầng **Lambda + ECS Fargate** cho TF1 Triage Hub, suy ra từ những gì đội AI (AIO)
> bàn giao và từ 3 hợp đồng đã chốt: `ai-api-contract.md`, `deployment-contract.md`, `telemetry-contract.md`,
> đối chiếu thêm bộ tài liệu bàn giao của AIO (`docs/06_cdo_evidence_handoff.md`, `10_datapack_insights_for_cdo.md`, `05_adrs.md`, `07_w11_readiness_checklist.md`).
>
> Bản này thay hướng EKS-native trong `02_infra_design.md` ở phần host AI Engine. Xương sống pipeline
> (SQS FIFO, DynamoDB, S3, Correlator) giữ nguyên — chỉ đổi cách chạy container.
>
> **Đang ở Tuần 2:** contract đã ký, đã qua giai đoạn hỏi AIO. Tài liệu này chỉ còn việc CDO tự làm.

---

## 1. Đội AI (AIO) đã bàn giao những gì

Các thứ **cố định** không được đổi (đã ký). Thiết kế hạ tầng xoay quanh các ràng buộc này.

| Nhóm | Đội AI giao / quy định | Nguồn |
|---|---|---|
| **Image chạy sẵn** | Docker image trên ECR: `589077667575.dkr.ecr.us-east-1.amazonaws.com/tf1-ai-triage-engine:latest`, digest `sha256:db688c5e…` | readiness §Bootstrap |
| **Endpoint sống** | Bootstrap App Runner `https://snpmtcwpys.us-east-1.awsapprunner.com` — `/healthz` + `/v1/triage` đã pass, **test được ngay** | readiness §Bootstrap |
| **Kiểu dịch vụ** | HTTP API trong container, cổng **8080** | deployment §Runtime |
| **Hai endpoint** | `GET /healthz`, `POST /v1/triage` | ai-api §Endpoint |
| **Header bắt buộc** | `X-Tenant-Id` (khớp `tenant_id` body), `X-Correlation-Id` (khớp `correlation_id`), `Authorization` | ai-api §Headers |
| **Xác thực (demo)** | Bearer token, biến `SERVICE_AUTH_TOKEN`, lưu Secrets Manager `tf1/ai-engine/service-auth-token` | deployment §Secrets |
| **Biến môi trường** | `APP_ENV`, `AWS_REGION=us-east-1`, `AIOPS_LOG_LEVEL`, `AIOPS_INVESTIGATION_MODE`; chế độ skeleton `AI_MODE=rules` (không gọi LLM) | deployment §Config, engine-spec §2 |
| **Scaling** | min **2**, max **6**; CPU **70%** (hoặc 100 req/phút/pod); nguội tăng 60s, giảm 300s | deployment §Scaling |
| **Mạng** | Chỉ private subnet; vào qua internal ALB; chỉ dịch vụ tích hợp CDO gọi cổng 8080 | deployment §Networking |
| **Health check** | `/healthz` → `200 {"status":"ok"}`, 30s, khỏe sau 2, hỏng sau 3 | deployment §Health |
| **Canary** | 10% → 50% → 100%, mỗi bước 5 phút; hủy nếu 5xx>1%, p99>2s/5 phút, thiếu `audit_id` | deployment §Rollout |
| **Rollback** | Quay bản trước, **RTO < 60 giây** | deployment §Rollback |
| **SLA API** | p99 **<2s**, sẵn sàng **≥99.5%**, **60 req/phút/tenant** (vượt → `429`), payload **≤512 KB** | ai-api §SLA |
| **Bốn trạng thái** | `DIAGNOSED`, `INVESTIGATE`, `INSUFFICIENT_CONTEXT`, `UNSAFE_SUGGESTION_BLOCKED` | ai-api §Status |
| **An toàn** | AI **không tự sửa lỗi**; chỉ gợi ý để người duyệt | ai-api §Safety |
| **Slack/Jira** | AI trả **dữ liệu thô** (`ticket_payload`, evidence, confidence, gợi ý người nhận); **CDO tự render Slack + tạo Jira**; gán người phải có người xác nhận | ai-api §Jira/Slack |
| **Nhận alert** | **Push-based**: CDO phát hiện rồi gọi AI; AI không poll | telemetry §Delivery |
| **Chống trùng** | CDO cấp `correlation_id`; AI idempotent theo `correlation_id` | telemetry §Delivery |
| **Datapack sẵn** | 9 case (3 scenario × 3): `adapted/*.request.json` (body `/v1/triage` đầy đủ) + 9 `evidence-bundle.json`; đã validate 9/9 HTTP 200 | handoff §Assets, insights §E2E |

**Hai điểm rút ra:**

1. **AIO giao một image, không giao mã nguồn.** Việc CDO là *chạy image cho tốt*, không viết logic RCA.
2. **Runtime do CDO chọn — và chính AIO chọn Fargate.** Contract ghi tên "EKS" nhưng doc giải pháp của AIO (`02_solution_design`, `03_ai_engine_spec`) ghi rõ *"Dockerized FastAPI on ECS/Fargate"*, *"Deployment target: ECS Fargate behind internal ALB"*, và bootstrap đang chạy App Runner. → Chọn **Fargate là khớp với AIO**, không phải đi ngược.

---

## 2. Suy ra hạ tầng từ ràng buộc

Đọc theo kiểu *"AIO yêu cầu X → CDO cần Y"*.

| AIO yêu cầu | Suy ra CDO cần | Dịch vụ AWS |
|---|---|---|
| Giao **image** HTTP API cổng 8080 | Nơi **chạy container** sẵn, không tự dựng cluster | **ECS Fargate** |
| p99 < 2s, tối thiểu 2 bản chạy | Container **luôn ấm**, không cold-start | Fargate `desired = 2` (không Lambda thuần cho engine) |
| Token từ Secrets Manager | Tạo secret + tiêm vào container | **Secrets Manager** + task role |
| Cần `APP_ENV`, `AWS_REGION`... | Khai báo env lúc chạy | Task definition (env) / SSM |
| Private subnet, internal ALB, cổng 8080 | Container subnet riêng sau ALB nội bộ, chặn nguồn gọi | VPC private subnet + **Internal ALB** + Security Group |
| Health `/healthz` 30s | ALB health-check đúng path | ALB Target Group `/healthz` |
| Scaling 2→6 CPU 70% | Bật auto-scaling | **Application Auto Scaling** cho ECS |
| Nhận alert **push** | Bộ phát hiện alert + cửa tiếp nhận | Demo app → **CloudWatch Alarm** → **SNS** → **Ingest Lambda** |
| Chống bão alert, không gọi AI thừa, idempotent | Hàng đợi chống trùng + bộ gom + kho trạng thái | **SQS FIFO** (+DLQ) → **Correlator Lambda** → **DynamoDB** |
| AI cần **full context trong body** (xem mục 3 — `evidence_uri` chưa chạy) | Correlator **nhồi context inline** vào `/v1/triage` | Correlator đọc bundle/CloudWatch, build body đầy đủ |
| AI trả thô, CDO render Slack + tạo Jira | Lớp tích hợp ngoài, tách khỏi AI | **Integration Lambda** → Slack Block Kit + Jira |
| Canary 10/50/100, rollback RTO<60s | Phát hành từng phần + quay lui nhanh | **CodeDeploy** (blue/green cho ECS) |
| `audit_id`, log, metric | Lưu vết kiểm toán + giám sát | **S3** (audit) + **CloudWatch** |
| Datapack + bundle | Kho bằng chứng cho phần kể chuyện + tương lai proxy | **S3** (lưu 9 bundle) |

**Kết luận host:** AIO giao image → CDO chạy container → **Fargate** (đúng lựa chọn của chính AIO):
không quản cluster, AWS lo scale/vá OS, hợp 6 ngày build. Giữ y nguyên cổng/ALB/subnet/secret/header/health/canary/rollback → không phá hợp đồng.

> Nếu vẫn muốn EKS (vì contract đặt tên EKS): host **cùng image** trên EKS, mọi thứ còn lại y nguyên. Đây là đánh đổi duy nhất giữa hai hướng.

---

## 3. Sơ đồ luồng xử lý sự cố

```text
Demo app (Fargate)
   │  sinh lỗi / độ trễ cao
   ▼
CloudWatch Alarm  ──►  SNS  ──►  Ingest Lambda
   (phát hiện alert)            (xác thực + chuẩn hóa webhook)
                                     │
                                     ▼
                              SQS FIFO  (+ DLQ)
                                     │
                                     ▼
                              Correlator Lambda
                              (gom alert liên quan, kiểm tra idempotency,
                               NHỒI full context inline vào body)
                                     │  POST /v1/triage  (body đầy đủ + X-Tenant-Id, X-Correlation-Id, Bearer)
                                     ▼
                         Internal ALB  :8080
                                     │
                                     ▼
                         AI Engine (Fargate · image AIO)
                                     │  trả status + ticket_payload + evidence + audit_id
                                     ▼
                              Correlator Lambda
                                     │  (cập nhật DynamoDB, lưu audit S3)
                                     ▼
                              Integration Lambda
                              (render Slack Block Kit + tạo Jira từ ticket_payload)
                                     │
                          ┌──────────┴──────────┐
                          ▼                     ▼
                        Slack                  Jira
```

**Kho dùng chung:**

```text
DynamoDB   →  incident_state, idempotency (theo correlation_id), ID Jira/Slack, con trỏ S3
S3         →  9 evidence bundle (kho bằng chứng) + audit/replay
Secrets Manager  →  bearer token tf1/ai-engine/service-auth-token
ECR        →  image AIO (host hoặc re-tag sang ECR của CDO)
CodeDeploy →  phát hành canary
```

> **Lưu ý quan trọng về bằng chứng (W11):** engine **chưa đọc `evidence_uri`** (handoff doc ghi rõ *"not yet implemented as a bundle reader"*). Nên W11 Correlator **nhồi full context inline** vào body `/v1/triage` — các file `adapted/*.request.json` đã đúng dạng này. S3 chỉ là nơi cất 9 bundle để Correlator đọc ra rồi nhồi vào body (và để dành cho proxy `GET /v1/evidence/incidents/{id}` ở phase sau).

**Giải thích từng bước:**

1. **Demo app** (Fargate) sinh lỗi/độ trễ để có cái mà chẩn đoán.
2. **CloudWatch Alarm** bắt ngưỡng → đẩy qua **SNS**.
3. **Ingest Lambda** xác thực, chuẩn hóa, đẩy vào **SQS FIFO**.
4. **SQS FIFO** giữ alert bền vững, có thử lại + DLQ — lớp chống mất alert.
5. **Correlator Lambda** (điểm khác biệt CDO): gom alert cùng một sự cố, kiểm tra DynamoDB chống trùng, đọc bundle từ S3, **nhồi full context vào body**, rồi gọi AI khi cần.
6. **AI Engine** nhận `POST /v1/triage`, chẩn đoán, trả trạng thái + dữ liệu thô + `audit_id`.
7. **Integration Lambda** render Slack + tạo Jira từ dữ liệu thô. AI không tự đụng Slack/Jira.

---

## 4. Thông tin kết nối cho AI Engine

### 4.1 Mạng vào (inbound)

| Mục | Giá trị |
|---|---|
| Ai gọi vào | Chỉ Correlator Lambda (trong VPC) qua Internal ALB |
| Cổng | 8080, HTTP |
| Đường dẫn | `POST /v1/triage`, `GET /healthz` |
| Header | `X-Tenant-Id`, `X-Correlation-Id`, `Authorization: Bearer <token>` |
| Chặn | SG `tf1-ai-engine-sg`: chỉ cho SG của Correlator vào 8080 |

### 4.2 Mạng ra (outbound)

Chế độ skeleton **`AI_MODE=rules` — không gọi LLM**, nên engine W11 gần như tự chứa. Egress cần:

| Engine gọi tới | Khi nào | Cần |
|---|---|---|
| Secrets Manager | đọc token | VPC Endpoint Secrets Manager |
| CloudWatch Logs | ghi log | VPC Endpoint / route sẵn |
| AWS Bedrock | **chỉ khi** bật `AI_MODE=hybrid` (sau skeleton) | VPC Endpoint Bedrock / NAT — chưa cần W11 |

> W11 không bật LLM → **không cần NAT Gateway**. Chỉ cần VPC Endpoint cho Secrets Manager + CloudWatch + S3. Rẻ và an toàn.

### 4.3 Service / secret / env

| Mục | Cấu hình |
|---|---|
| Service | ECS Fargate `tf1-ai-triage-engine` sau Internal ALB |
| Secret | `SERVICE_AUTH_TOKEN` ← Secrets Manager `tf1/ai-engine/service-auth-token` |
| Env tối thiểu | `APP_ENV`, `AWS_REGION=us-east-1`, `AIOPS_LOG_LEVEL`, `AIOPS_INVESTIGATION_MODE`, `AI_MODE=rules` |
| Health | ALB target group `/healthz`, 30s, khỏe 2 / hỏng 3 |
| Scaling | desired 2, min 2, max 6, CPU 70% |
| Quyền AWS | Task role đặc quyền tối thiểu (đọc secret, ghi S3 audit, ghi CloudWatch). Không access key dài hạn |

---

## 5. Khi có sự cố: bắn Slack/Jira ngay hay đợi AI?

**Đợi AI chẩn đoán xong rồi mới đẩy** — con người nhận chẩn đoán kèm bằng chứng, không phải alert trần.

Nhưng **không để mất alert khi AI chết** (deployment §Failure Modes: *AI unavailable → CDO queue retry hoặc fallback ticket*):

| Kết quả AI | CDO làm |
|---|---|
| `DIAGNOSED` / `INVESTIGATE` | Tạo Jira + render Slack đầy đủ (luồng chính) |
| `INSUFFICIENT_CONTEXT` | Vẫn tạo ticket, gắn cờ thiếu ngữ cảnh, liệt kê field thiếu |
| `UNSAFE_SUGGESTION_BLOCKED` | Tạo ticket cảnh báo, không kèm gợi ý bị chặn |
| AI **timeout / 500 / 503** | Tạo **fallback ticket** alert thô — không nuốt im lặng |

Idempotency theo `correlation_id` (DynamoDB) → thử lại không tạo ticket trùng.

---

## 6. Khác biệt so với bản EKS-native (`02_infra_design.md`)

| Khía cạnh | Cũ (EKS-native) | Bản này (Fargate + Lambda) |
|---|---|---|
| Host AI Engine | Pod EKS | ECS Fargate task |
| Glue | Worker/pod trong cluster | Lambda |
| Trigger alert | Alertmanager | CloudWatch Alarm → SNS |
| Nguồn bằng chứng | Prometheus/Loki trực tiếp | **Inline full context (W11)**; S3 bundle / `evidence_uri` ở phase sau |
| SQS FIFO / DynamoDB / S3 | Có | **Giữ nguyên** |
| Correlator | Có | **Giữ nguyên** |
| Phát hành | ArgoCD + canary | CodeDeploy canary |
| Rollback | `kubectl rollout undo` | CodeDeploy quay task set cũ |
| Chi phí vận hành | Cao | Thấp (AWS lo) |

**Xương sống pipeline (điểm khác biệt thật của CDO) giống hệt nhau.** EKS chỉ là chi phí vận hành.

---

## 7. Soi lại với hợp đồng (recheck)

Ký hiệu: ✅ đạt · 📝 ADR nhẹ.

| Điều khoản | Yêu cầu | Đạt? | Ghi chú |
|---|---|---|---|
| ai-api · Endpoint | `/healthz`, `/v1/triage` cổng 8080 | ✅ | Giữ nguyên, ALB trỏ đúng |
| ai-api · Header/Tenant | `X-Tenant-Id` khớp body | ✅ | Correlator gắn đủ header |
| ai-api · p99<2s | Container ấm | ✅ | Fargate desired 2 |
| ai-api · Rate 60/min/tenant | 429 khi vượt | ✅ | Đếm theo tenant ở tầng gọi |
| ai-api · Payload ≤512KB | Không nhồi raw telemetry | ✅ | Context cắt mẫu theo telemetry-contract (metric 15p/5p, log ≤50) |
| ai-api · 4 status + Safety | Không auto-remediation | ✅ | CDO chỉ render, không thực thi |
| ai-api · Slack/Jira | AI thô, CDO render | ✅ | Integration Lambda render + tạo Jira |
| telemetry · Push-based | CDO phát hiện rồi gọi | ✅ | CloudWatch → SNS → ... |
| telemetry · Idempotent | Theo `correlation_id` | ✅ | DynamoDB key idempotency |
| telemetry · Evidence bounded | tenant/service/env/window | ✅ | Correlator nhồi inline đúng phạm vi (W11); proxy bounded sau |
| telemetry · Log ≤50/service | Cắt mẫu, không PII | ✅ | Bundle đã cắt sẵn |
| deployment · Port/Health/Secret | 8080, `/healthz`, Secrets Manager | ✅ | Tiêm secret vào task |
| deployment · Scaling 2–6, CPU70% | min2 max6 | ✅ | App Auto Scaling |
| deployment · Private + internal ALB | Subnet riêng | ✅ | Fargate private + internal ALB |
| deployment · Canary 10/50/100 | Phát hành từng phần | ✅ | CodeDeploy blue/green |
| deployment · Rollback RTO<60s | Quay nhanh | ✅ | CodeDeploy shift task set cũ |
| **deployment · Compute "EKS"** | Contract đặt tên EKS | **📝 ADR nhẹ** | AIO docs (02/03) tự chọn ECS Fargate + bootstrap App Runner → Fargate khớp AIO. Vẫn nên 1 ADR ngắn ghi: EKS Deployment→ECS service, namespace `tf1-aiops`→ECS, `kubectl rollout undo`→CodeDeploy, IRSA→ECS task role |

**Tóm tắt:** API + telemetry + phần lớn deployment **đạt**. Chỉ còn **1 ADR nhẹ** (host Fargate) — rủi ro thấp vì chính AIO chọn Fargate.

---

## 8. Soi lại với đề bài (requirement)

| Yêu cầu | Đạt? |
|---|---|
| Luồng: alert → ngữ cảnh → AI → Jira → Slack | ✅ (mục 3) |
| Con người trong vòng lặp, không tự sửa lỗi | ✅ AI chỉ gợi ý; gán người xác nhận tay |
| Đa tenant (`tenant-a`, `tenant-b`) | ✅ metadata: `tenant_id` là dimension CloudWatch, group SQS, PK DynamoDB, prefix S3, header `X-Tenant-Id`. Data AIO toàn `tenant-a` → relabel vài case thành `tenant-b` |
| Pipeline tin cậy, chịu bão alert | ✅ SQS FIFO + Correlator gom + idempotency |
| Giám sát pipeline | ✅ CloudWatch + S3 audit/replay |

---

## 9. Việc CDO làm tiếp

- [ ] Host image AIO lên Fargate — **re-tag** image (`589077667575…/tf1-ai-triage-engine`) vào ECR của CDO, hoặc cấu hình cross-account pull → `/healthz` xanh.
- [ ] Smoke test: `POST` 1 file `adapted/*.request.json` (inline) → nhận `200`.
- [ ] Viết **ADR ngắn**: "Host AI Engine bằng ECS Fargate" (ánh xạ rollback/namespace/IRSA như mục 7).
- [ ] Dựng Terraform: VPC + private subnet, Fargate service + Internal ALB, SQS FIFO + DLQ, DynamoDB, S3 (bundle + audit), Secrets Manager, VPC Endpoint (Secrets/CloudWatch/S3), các Lambda, CodeDeploy canary.
- [ ] Bỏ 9 evidence bundle lên S3; Correlator đọc bundle → **nhồi inline** vào body `/v1/triage`.
- [ ] Render Slack Block Kit + tạo Jira từ `ticket_payload`.
- [ ] Bộ test tải W11: 30 request/phút, kiểm p99 < 2s.
- [ ] Demo 3 scenario: latency-degradation, critical-service-down, noisy-false-alert (đúng output mong đợi ở `10_datapack_insights_for_cdo.md`).
