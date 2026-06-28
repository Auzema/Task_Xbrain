# Hạ tầng CDO 5 (bản xét lại) — Host image của đội AI bằng Fargate + Lambda

> Tài liệu này trình bày phương án hạ tầng **Lambda + ECS Fargate** cho TF1 Triage Hub,
> được suy ra trực tiếp từ những gì đội AI (AIO) bàn giao và từ 3 bản hợp đồng đã chốt:
> `ai-api-contract.md`, `deployment-contract.md`, `telemetry-contract.md`.
>
> Bản này thay cho hướng EKS-native trong `02_infra_design.md` ở phần **host AI Engine**.
> Phần xương sống pipeline (SQS FIFO, DynamoDB, S3, Correlator) giữ nguyên — chỉ đổi cách chạy container.
>
> Cách đọc tài liệu: đi từ "đội AI giao gì" → "suy ra mình cần gì" → "vẽ flow" → "soi lại hợp đồng và đề bài".

---

## 1. Đội AI (AIO) đã bàn giao những gì

Đây là các thứ **cố định** mình không được đổi (đã ký trong hợp đồng). Mình thiết kế hạ tầng *xoay quanh* các ràng buộc này.

| Nhóm | Đội AI giao / quy định | Nguồn hợp đồng |
|---|---|---|
| **Sản phẩm bàn giao** | Một **Docker image** chứa engine triage, đẩy lên **ECR** kèm tag/digest cố định cho mỗi bản phát hành | deployment §Compute, §Ownership |
| **Kiểu dịch vụ** | HTTP API chạy trong container, lắng nghe **cổng 8080** | deployment §Runtime |
| **Hai endpoint** | `GET /healthz` (kiểm tra sống) và `POST /v1/triage` (nhận sự cố để chẩn đoán) | ai-api §Endpoint |
| **Header bắt buộc** | `X-Tenant-Id` (phải khớp `tenant_id` trong body), `X-Correlation-Id` (khớp `correlation_id`), `Authorization` | ai-api §Headers, deployment §Runtime |
| **Xác thực (demo)** | Bearer token có phạm vi, lưu ở **AWS Secrets Manager** tên `tf1/ai-engine/service-auth-token`. Bản production ưu tiên IAM SigV4 / JWT | deployment §Secrets |
| **Biến môi trường** | Tối thiểu: `APP_ENV`, `AWS_REGION=us-east-1`, `AIOPS_LOG_LEVEL`, `AIOPS_INVESTIGATION_MODE`. Tùy chọn: `PROMETHEUS_URL`, `LOKI_URL`, `JAEGER_URL`, `EVIDENCE_BUNDLE_BASE_PATH`... | deployment §Configuration |
| **Tài nguyên / pod** | Skeleton: CPU 500m–1000m, RAM 1–2 GiB. Nếu bật LLM: CPU 1000m–2000m, RAM 2–4 GiB | deployment §Compute |
| **Scaling** | Tối thiểu **2**, tối đa **6** bản chạy. Tăng theo CPU **70%** (hoặc 100 request/phút/pod). Nguội tăng 60s, nguội giảm 300s | deployment §Scaling |
| **Mạng** | Chỉ **private subnet**. Vào qua **internal ALB / private ingress**. Chỉ cho dịch vụ tích hợp của CDO gọi vào cổng 8080 | deployment §Networking |
| **Health check** | `/healthz` trả `200 {"status":"ok"}`, kiểm tra mỗi 30s, khỏe sau 2 lần 200, hỏng sau 3 lần khác 200 | deployment §Health Check |
| **Phát hành (canary)** | 10% → 50% → 100%, mỗi bước 5 phút. Hủy nếu 5xx > 1%, hoặc p99 > 2s suốt 5 phút, hoặc thiếu `audit_id` | deployment §Rollout |
| **Rollback** | Quay về bản trước, mục tiêu **RTO < 60 giây** khi chỉ đổi tag/digest image | deployment §Rollback |
| **SLA của API** | p99 **< 2 giây**, sẵn sàng **≥ 99.5%**, giới hạn **60 request/phút/tenant** (vượt trả `429`), payload **≤ 512 KB** | ai-api §SLA |
| **Bốn trạng thái trả về** | `DIAGNOSED`, `INVESTIGATE`, `INSUFFICIENT_CONTEXT`, `UNSAFE_SUGGESTION_BLOCKED` | ai-api §Status |
| **Ranh giới an toàn** | AI **không bao giờ tự sửa lỗi (auto-remediation)**. Chỉ trả gợi ý để con người duyệt | ai-api §Safety |
| **Slack/Jira** | AI **chỉ trả dữ liệu thô** (`ticket_payload`, evidence, confidence, gợi ý người nhận). **CDO tự render Slack Block Kit và tạo Jira**. Gán người phải có con người xác nhận | ai-api §Jira/Slack |
| **Cách nhận alert** | **Push-based**: CDO phát hiện alert rồi *gọi vào* AI. AI **không** poll liên tục để tự tìm alert | telemetry §Delivery |
| **Lấy thêm bằng chứng** | Sau khi có alert, AI có thể *kéo* bằng chứng giới hạn theo **tenant / service / env / khung thời gian**. Trỏ qua `evidence_uri` (S3 hoặc evidence API) | telemetry §Evidence |
| **Chống trùng** | CDO phải cấp `correlation_id`; AI trả kết quả **idempotent theo `correlation_id`** | telemetry §Delivery |
| **Giới hạn log** | Tối đa **50 dòng log / service / sự cố**, không PII, cửa sổ metric 15 phút trước + 5 phút sau alert | telemetry §Logs/Metrics |
| **Endpoint bootstrap W11** | Đội AI đang để bản demo tạm trên **AWS App Runner** (một dạng serverless container) | deployment §Per-CDO |

**Hai điểm quan trọng rút ra ngay từ bảng trên:**

1. **Hợp đồng giao một image, không giao mã nguồn.** Việc của CDO là *chạy image đó cho tốt*, không phải viết logic RCA.
2. **Runtime do CDO tự chọn.** Hợp đồng có ghi tên "EKS" nhưng nói rõ *"CDO sở hữu cluster/service runtime"*, cho phép *"hoặc namespace/dịch vụ tương đương của CDO"*, và bản demo của chính đội AI đang chạy trên **App Runner** (không phải EKS). Nghĩa là **chạy bằng Fargate là hợp lệ** — xem mục 7 về thủ tục ADR.

---

## 2. Suy ra hạ tầng từ những ràng buộc trên

Đọc theo kiểu: *"AI yêu cầu X → mình phải có Y"*.

| Đội AI yêu cầu | Suy ra CDO cần | Dịch vụ AWS chọn |
|---|---|---|
| Giao **image** chạy HTTP API cổng 8080 | Một nơi **chạy container** có sẵn, không phải tự dựng cluster | **ECS Fargate** (chạy container không cần quản máy chủ) |
| p99 < 2s, tối thiểu 2 bản luôn chạy | Container phải **luôn ấm**, không được khởi động nguội mỗi request | Fargate service `desired = 2` (không dùng Lambda thuần cho engine vì cold-start phá p99) |
| Đọc token từ Secrets Manager `tf1/ai-engine/service-auth-token` | CDO tạo secret và **tiêm vào container** | **Secrets Manager** + task role đọc secret |
| Cần `APP_ENV`, `AWS_REGION`, `AIOPS_*`... | Khai báo biến môi trường khi chạy | Task definition (env) / SSM Parameter Store |
| Chỉ private subnet, vào qua internal ALB, cổng 8080 | Đặt container trong subnet riêng, sau **ALB nội bộ**, chặn nguồn gọi | VPC private subnet + **Internal ALB** + Security Group |
| Health `/healthz` 30s | ALB phải health-check đúng đường dẫn này | ALB Target Group health check `/healthz` |
| Scaling 2→6 theo CPU 70% | Bật auto-scaling cho service | **Application Auto Scaling** cho ECS |
| Nhận alert kiểu **push** (CDO phát hiện rồi gọi AI) | Cần bộ **phát hiện alert** + cửa tiếp nhận | Demo app (Fargate) → **CloudWatch Alarm** → **SNS** → **Ingest Lambda** |
| Chống bão alert, **không gọi AI thừa**, idempotent theo `correlation_id` | Cần hàng đợi chống trùng + bộ gom + kho trạng thái | **SQS FIFO** (+ DLQ) → **Correlator Lambda** → **DynamoDB** (state + idempotency) |
| Bằng chứng giới hạn, trỏ qua `evidence_uri` | CDO **dựng sẵn gói bằng chứng** và đưa đường dẫn | **S3** evidence bundle + nhãn `evidence_uri` |
| AI trả dữ liệu thô, **CDO tự render Slack + tạo Jira** | Cần lớp tích hợp ngoài, tách khỏi AI | **Integration Lambda** → Slack Block Kit + Jira |
| Canary 10/50/100, rollback RTO < 60s | Cần cơ chế phát hành từng phần + quay lui nhanh | **CodeDeploy** (blue/green cho ECS) |
| `audit_id` mỗi lần thành công, log, metric | Cần nơi lưu vết kiểm toán + giám sát | **S3** (audit) + **CloudWatch** (log/metric) |
| Kéo image từ **ECR của đội AI** | CDO phải **pull được qua tài khoản khác** (cross-account) | Quyền cross-account pull trên ECR (**việc cần chốt với AI — mục 8**) |

**Kết luận lựa chọn host:** đội AI giao image → CDO cần chạy container → chọn **Fargate** thay vì EKS, vì:

- Không phải vận hành cluster (không patch node, không quản control plane) — hợp với 6 ngày build.
- AWS lo scale, lo vá hệ điều hành.
- Vẫn giữ y nguyên: cổng 8080, internal ALB, private subnet, secret, header, health check, canary, rollback < 60s → **không phá hợp đồng** (chi tiết mục 7).

> Nếu muốn tránh thủ tục ADR (mục 7), có thể host **cùng image đó trên EKS** — mọi thứ còn lại trong tài liệu này giữ nguyên. Đây là điểm đánh đổi duy nhất giữa hai hướng.

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
                              (đệm bền vững, chống trùng theo nhóm)
                                     │
                                     ▼
                              Correlator Lambda
                              (gom alert liên quan, kiểm tra idempotency,
                               dựng evidence bundle, quyết định có gọi AI không)
                                     │  POST /v1/triage  (kèm X-Tenant-Id, X-Correlation-Id, Bearer token)
                                     ▼
                         Internal ALB  :8080
                                     │
                                     ▼
                         AI Engine (Fargate · image của AIO)
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

**Kho dùng chung (không nằm thẳng trên luồng, nhiều thành phần đọc/ghi):**

```text
DynamoDB   →  incident_state, idempotency (theo correlation_id), ID Jira/Slack, con trỏ S3
S3         →  evidence bundle (cho AI kéo) + audit/replay (payload, request/response AI, RCA)
Secrets Manager  →  bearer token tf1/ai-engine/service-auth-token
ECR        →  image của đội AI (pull cross-account)
CodeDeploy →  phát hành canary cho AI Engine
```

**Giải thích từng bước (ngắn gọn):**

1. **Demo app** chạy trên Fargate, cố tình tạo lỗi/độ trễ để có cái mà chẩn đoán.
2. **CloudWatch Alarm** bắt ngưỡng bất thường → đẩy thông báo qua **SNS**.
3. **Ingest Lambda** nhận thông báo, kiểm tra đủ trường bắt buộc, chuẩn hóa rồi đẩy vào **SQS FIFO**.
4. **SQS FIFO** giữ alert bền vững, có cơ chế thử lại và hàng đợi lỗi (DLQ). Đây là lớp chống mất alert.
5. **Correlator Lambda** là điểm khác biệt của CDO: gom nhiều alert của *cùng một sự cố* thành một, kiểm tra DynamoDB để không xử lý trùng, dựng **gói bằng chứng** lên S3, rồi **chỉ gọi AI khi thật sự cần**.
6. **AI Engine** (image của đội AI, chạy Fargate) nhận `POST /v1/triage`, chẩn đoán, trả về trạng thái + dữ liệu thô + `audit_id`.
7. **Integration Lambda** lấy dữ liệu thô đó render Slack và tạo Jira. **AI không tự đụng vào Slack/Jira.**

---

## 4. Thông tin kết nối cho AI Engine (in/out, service, secret)

Đây là phần trả lời trực tiếp câu hỏi: *deploy xong thì engine kết nối ra/vào đâu, cấu hình service/API thế nào.*

### 4.1 Mạng vào (inbound)

| Mục | Giá trị |
|---|---|
| Ai được gọi vào | **Chỉ Correlator Lambda** (nằm trong VPC), đi qua Internal ALB |
| Cổng | **8080**, giao thức HTTP |
| Đường dẫn | `POST /v1/triage`, `GET /healthz` |
| Header kèm theo | `X-Tenant-Id`, `X-Correlation-Id`, `Authorization: Bearer <token>` |
| Chặn truy cập | Security Group `tf1-ai-engine-sg`: chỉ cho SG của Correlator vào cổng 8080; chặn hết phần còn lại |

### 4.2 Mạng ra (outbound) — **cần hỏi đội AI để chốt**

Engine có thể cần gọi ra ngoài. Đây là câu hỏi quan trọng nhất quyết định có cần NAT Gateway hay không:

| Engine có gọi tới...? | Nếu CÓ thì cần |
|---|---|
| Secrets Manager (đọc token) | VPC Endpoint cho Secrets Manager |
| Evidence API / S3 (kéo bằng chứng) | VPC Endpoint cho S3 |
| AWS Bedrock (nếu bật LLM) | VPC Endpoint Bedrock hoặc NAT |
| Prometheus / Loki (nếu dùng context tool trực tiếp) | Đường mạng nội bộ tới observability |

> Nếu image **tự chứa, không gọi ra ngoài** → **không cần NAT Gateway** → rẻ hơn và an toàn hơn. Phải hỏi rõ đội AI.

### 4.3 Cấu hình service / secret / env

| Mục | Cấu hình |
|---|---|
| Service | ECS Fargate service `tf1-ai-triage-engine`, sau Internal ALB |
| Secret | Token lưu ở Secrets Manager `tf1/ai-engine/service-auth-token`, tiêm vào container. **Hỏi AI: image đọc token từ biến nào** (hợp đồng ghi `SERVICE_AUTH_TOKEN`) |
| Env tối thiểu | `APP_ENV`, `AWS_REGION=us-east-1`, `AIOPS_LOG_LEVEL`, `AIOPS_INVESTIGATION_MODE` |
| Health check | ALB target group trỏ `/healthz`, 30s, khỏe sau 2, hỏng sau 3 |
| Scaling | desired 2, min 2, max 6, scale theo CPU 70% (+ tùy chọn request/phút) |
| Quyền AWS | Task role theo nguyên tắc đặc quyền tối thiểu (đọc secret, ghi S3 audit, ghi CloudWatch). Không dùng access key dài hạn |

---

## 5. Khi có sự cố: bắn Slack/Jira ngay hay đợi AI chẩn đoán xong?

**Nguyên tắc: đợi AI chẩn đoán xong rồi mới đẩy Slack/Jira.** Đó là toàn bộ giá trị sản phẩm — con người nhận **chẩn đoán kèm bằng chứng**, không phải alert trần.

Nhưng **không được để mất alert khi AI chết**. Hợp đồng (deployment §Failure Modes) đã quy định: *AI unavailable → CDO queue retry hoặc tạo fallback ticket.* Vậy xử lý theo bảng:

| Kết quả từ AI | CDO làm gì |
|---|---|
| `DIAGNOSED` / `INVESTIGATE` | Tạo Jira + render Slack đầy đủ chẩn đoán (luồng chính) |
| `INSUFFICIENT_CONTEXT` | Vẫn tạo ticket, gắn cờ "thiếu ngữ cảnh", liệt kê field còn thiếu |
| `UNSAFE_SUGGESTION_BLOCKED` | Tạo ticket cảnh báo, **không** kèm gợi ý bị chặn |
| AI **timeout / 500 / 503** | Tạo **fallback ticket** với alert thô ("AI engine unavailable") — tuyệt đối không nuốt im lặng |

Idempotency theo `correlation_id` trong DynamoDB bảo đảm: thử lại nhiều lần **không tạo ticket trùng**.

---

## 6. Khác biệt so với bản EKS-native (`02_infra_design.md`)

| Khía cạnh | Bản cũ (EKS-native) | Bản này (Fargate + Lambda) |
|---|---|---|
| Host AI Engine | Pod trên EKS | ECS Fargate task |
| Glue (ingest/correlate/integrate) | Worker/pod trong cluster | Lambda |
| Trigger alert | Alertmanager | CloudWatch Alarm → SNS |
| Nguồn bằng chứng cho AI | Prometheus/Loki trực tiếp | **Gói bằng chứng S3 qua `evidence_uri`** (hợp đồng W11 cho phép) |
| SQS FIFO / DynamoDB / S3 | Có | **Giữ nguyên** |
| Correlator (chống bão alert) | Có | **Giữ nguyên** |
| Phát hành | ArgoCD + canary | CodeDeploy canary |
| Rollback | `kubectl rollout undo` | CodeDeploy quay về task set cũ |
| Chi phí vận hành | Cao (quản cluster, node, control plane) | Thấp (AWS lo) |

**Xương sống pipeline (điểm khác biệt thật của CDO) giống hệt nhau.** EKS không phải điểm khác biệt — nó chỉ là chi phí vận hành.

---

## 7. Soi lại với hợp đồng (recheck)

Ký hiệu: ✅ đạt · 📝 cần ADR · ⚠️ việc cần làm/chốt.

| Điều khoản hợp đồng | Yêu cầu | Thiết kế đáp ứng? | Ghi chú |
|---|---|---|---|
| ai-api · Endpoint | `/healthz`, `/v1/triage` cổng 8080 | ✅ | Giữ nguyên trong container, ALB trỏ đúng |
| ai-api · Header/Tenant | `X-Tenant-Id` khớp body, `X-Correlation-Id` | ✅ | Correlator gắn đủ header khi gọi |
| ai-api · SLA p99<2s | Container luôn ấm | ✅ | Fargate desired 2, không cold-start |
| ai-api · Rate 60/min/tenant | Trả 429 khi vượt | ✅ | Đo theo tenant ở tầng gọi/đếm |
| ai-api · Payload ≤512KB | Không nhồi raw telemetry | ✅ | Bằng chứng lớn để ở S3, chỉ truyền `evidence_uri` |
| ai-api · 4 status + Safety | Không auto-remediation, advisory | ✅ | CDO không thực thi hành động; chỉ render |
| ai-api · Slack/Jira | AI trả thô, CDO render | ✅ | Integration Lambda render Block Kit + tạo Jira |
| telemetry · Push-based | CDO phát hiện rồi gọi AI | ✅ | CloudWatch → SNS → Ingest → ... → gọi AI |
| telemetry · Idempotent | Theo `correlation_id` | ✅ | DynamoDB giữ key idempotency |
| telemetry · Evidence bounded | tenant/service/env/window, qua `evidence_uri` | ✅ | Gói bằng chứng S3 (đúng quyết định W11 evidence-bundles) |
| telemetry · Log ≤50/service | Cắt mẫu, không PII | ✅ | Correlator/evidence builder cắt đúng giới hạn |
| deployment · Port/Health/Secret | 8080, `/healthz`, Secrets Manager | ✅ | Giữ nguyên, tiêm secret vào task |
| deployment · Scaling 2–6, CPU70% | min2 max6 | ✅ | App Auto Scaling cho ECS |
| deployment · Private + internal ALB | Chỉ subnet riêng | ✅ | Fargate trong private subnet sau internal ALB |
| deployment · Canary 10/50/100 | Phát hành từng phần | ✅ | CodeDeploy blue/green cho ECS |
| deployment · Rollback RTO<60s | Quay về bản trước nhanh | ✅ | CodeDeploy shift về task set cũ < 60s |
| **deployment · Compute "EKS Deployment"** | Hợp đồng đặt tên EKS | **📝 cần ADR** | Đổi sang Fargate đụng clause *"changes to endpoint hosting + rollout/rollback behavior require a formal ADR"*. Hợp lệ vì hợp đồng cho *"CDO platform equivalent"* + bootstrap đang là App Runner. **Phải viết 1 ADR** ghi: EKS Deployment → ECS service, namespace `tf1-aiops` → ECS/CloudMap, `kubectl rollout undo` → CodeDeploy, IRSA → ECS task role |
| deployment · ECR image | Pull image của đội AI | **⚠️ cần chốt** | Cần quyền **cross-account pull** từ ECR của AI sang tài khoản CDO |
| deployment · Egress endpoints | Engine gọi Secrets/Bedrock/evidence | **⚠️ cần chốt** | Hỏi AI image gọi ra đâu → quyết định NAT vs VPC Endpoint |
| Observability stack (Prom/Loki/Grafana) | Hợp đồng giả định stack này trong EKS | **📝 lưu ý** | Bản serverless dùng **CloudWatch** cho demo app + **S3 evidence bundle** thay nguồn live Prom/Loki. Hợp đồng cho phép qua `evidence_uri`; ghi rõ trong ADR |

**Tóm tắt recheck:** Toàn bộ API + telemetry + phần lớn deployment **đạt**. Chỉ có **một việc cần ADR**: đổi host EKS→Fargate (kéo theo cách rollback và namespace). Đây là *thủ tục được hợp đồng cho phép*, không phải vi phạm. Cộng **hai việc cần chốt với đội AI** (cross-account ECR, egress của image).

---

## 8. Soi lại với đề bài (requirement)

| Yêu cầu đề bài | Thiết kế đáp ứng? |
|---|---|
| Luồng: alert → ngữ cảnh → AI chẩn đoán → Jira → Slack | ✅ Đúng nguyên luồng (mục 3) |
| Có con người trong vòng lặp, không tự sửa lỗi | ✅ AI chỉ gợi ý; gán người phải xác nhận tay |
| Đa tenant (demo `tenant-a`, `tenant-b`) | ✅ Phân tách bằng metadata: `tenant_id` là dimension CloudWatch, message group SQS, partition key DynamoDB, prefix S3, header `X-Tenant-Id` |
| Pipeline tin cậy, chịu được bão alert | ✅ SQS FIFO + Correlator gom + idempotency (điểm khác biệt CDO) |
| Giám sát được toàn pipeline | ✅ CloudWatch cho Lambda/SQS/DynamoDB/Fargate + S3 audit/replay |

---

## 9. Việc cần chốt với đội AI trước W11

Một câu hỏi gọn gửi đội AI, gồm 4 ý:

1. **Cross-account ECR:** cho CDO **pull image** từ ECR của tụi bay qua tài khoản khác — cần ARN repo + policy.
2. **Egress:** image **gọi ra ngoài** những đâu (Secrets Manager / Bedrock / evidence API / Loki)? Hay self-contained? → để CDO quyết NAT vs VPC Endpoint.
3. **Token:** image đọc bearer token từ **biến môi trường nào** (mặc định hợp đồng là `SERVICE_AUTH_TOKEN`)? Đọc từ env hay file mount?
4. **Env/secret bắt buộc:** liệt kê đủ biến môi trường + secret cần cấp (model name, deadline, evidence base path...).

---

## 10. Việc CDO phải làm tiếp

- [ ] Viết **ADR**: "Host AI Engine bằng ECS Fargate thay vì EKS" (ghi rõ ánh xạ rollback/namespace/IRSA như mục 7).
- [ ] Chốt 4 câu hỏi mục 9 với đội AI.
- [ ] Dựng Terraform: VPC + private subnet, Fargate service + Internal ALB, SQS FIFO + DLQ, DynamoDB, S3 (evidence + audit), Secrets Manager, các Lambda, CodeDeploy canary.
- [ ] Định nghĩa **schema gói bằng chứng** trên S3 + cách Correlator sinh `evidence_uri`.
- [ ] Bộ test tải W11: 30 request/phút, kiểm p99 < 2s.
