# Evidence Pack - CDO-02 Task Force 3 Self-Heal Engine

> Tài liệu hướng dẫn submission evidence cho **CDO-02** trong capstone W11-W12.
> Đề tài: **Self-Heal Agent for Kubernetes Workloads on AWS EKS**.
> Customize từ `CAPSTONE_EVIDENCE_PACK_FORMAT.md` cho project cụ thể của nhóm.

---

## 1. Cover

| Trường | Giá trị |
| --- | --- |
| **Nhóm** | CDO-02 |
| **Topic** | TF3_SELFHEAL_LEARNER |
| **Thành viên** | NGô Hữu Tài, Phạm Tùng Dương, Hà Tây Nguyên, Ka Phu Đông, Lê Văn Hải, Nguyễn Văn Toàn, Nguyễn Đỗ Hoàng Phúc, Trần Văn Đức, Nguyễn Thị Mến, Nguyễn Đức Tài |
| **Repo** |  |

---

## 1. Mục Đích & Nguyên Tắc Cốt Lõi

Document trail = công cụ chính để show **process thinking**, không chỉ "đoạn code chạy được". Reviewer chấm dựa trên **3 layer evidence**:

1. **Document quality** — thinking, trade-off analysis, justification (đây là evidence pack)
2. **Build artifacts** — code executor, infra Terraform, K8s manifests working
3. **Buổi chấm performance** — present + Q&A + individual defense

Doc artifact đóng góp **~40% điểm W11** và **~30% điểm W12** (plus là input chính để reviewer hiểu trước buổi chấm T5 W12).

### 3 Quy tắc "Sống còn" cho CDO-02

| Quy tắc | Áp dụng cụ thể cho CDO-02 |
|---|---|
| **Live in repo** | Tất cả 7 docs nằm trong `docs/`, commit lịch sử đều đặn — KHÔNG viết 1 cục cuối T6/T4. Git history là bằng chứng process. |
| **Focus on WHY** | Tại sao chọn K8s-heavy angle? Tại sao namespace-based isolation thay vì cluster-per-tenant? Tại sao CDO executor là execution boundary, không để AI gọi K8s trực tiếp? |
| **Implementable** | Doc đủ chi tiết: ai đọc xong có thể `terraform apply` + `kubectl apply` + deploy executor + chạy scenario test lại được. |

---

---

## 2. Timeline Checkpoint cho CDO-02

```
W11 T2 ─── T3 ─── T4 ─── T5 ─── T6           W12 T2 ─── T3 ─── T4
                  ▲              ▲                  ▲              ▲
              [Progress #1]  [Evidence #1]    [Progress #2]   [Evidence #2]
              light check    MAIN ⭐           light check     MAIN ⭐
```

### 4 Checkpoint

| # | Khi | Bắt buộc gì (CDO-02 cụ thể) | Scoring |
|---|---|---|---|
| **Progress #1** (light) | EOD T4 W11 | `01_requirements_analysis.md` draft + `02_infra_design.md` draft (K8s-heavy angle declared, multi-tenant approach) + `08_adrs.md` ≥2 ADRs | Sanity check, không chấm chính thức |
| **Evidence Pack #1** ⭐ | EOD T6 W11 | TẤT CẢ 6 doc Pack #1 + VPC/EKS/Observability base infra chạy được | **~40% điểm W11** |
| **Progress #2** (light) | EOD T2 W12 | AI engine integration started + tenant onboarding flow draft + docs updated | Sanity check |
| **Evidence Pack #2** ⭐ | EOD T4 W12 (code freeze 18h) | TẤT CẢ 7 doc final + `05_cost_analysis.md` measured + `07_test_eval_report.md` với chaos response evidence + git tag `final` | **~30% điểm W12** + input chính cho buổi chấm T5 |

**Light progress check**: review qua repo commits + WhatsApp standup, không cần buổi formal. Mục đích phát hiện nhóm tụt sớm.

---

---

## 3. Document Set — CDO-02 (7 Documents)

CDO-02 cần submit tổng cộng **7 files** trong thư mục `docs/`:

| # | File | Pack #1 W11 | Pack #2 W12 | Mục tiêu | Trạng thái hiện tại |
|---|---|---|---|---|---|
| 1 | `01_requirements_analysis.md` | ✓ | ✓ refined | Phân tích đề tài Self-Heal từ infra/platform perspective | Ready (~30KB, ~4000+ từ) |
| 2 | `02_infra_design.md` | ✓ draft | ✓ updated | Architecture + K8s-heavy angle + multi-tenant approach | Ready (~32KB, comprehensive) |
| 3 | `03_security_design.md` | ✓ draft | ✓ refined | IAM · RBAC · NetworkPolicy · Audit · Tenant isolation | Ready (~15KB) |
| 4 | `04_deployment_design.md` | ✓ draft | ✓ working | Terraform IaC + ArgoCD GitOps + deployment strategy | Ready (~29KB, comprehensive) |
| 5 | `05_cost_analysis.md` | (skeleton) | ✓ **measured** | Per-tenant cost model + actual AWS spend | Draft — cần actual measured data W12 |
| 6 | `07_test_eval_report.md` | - | ✓ **new** | SLO evidence + scenario test + chaos response + security test | Draft — cần execution evidence W12 |
| 7 | `08_adrs.md` | ✓ ongoing | ✓ final (≥5) | Architecture Decision Records | ✅ Ready (7 ADRs, có thể cần thêm W12) |

### Contracts (do AI team own, CDO review + accept)

| Contract | File | Trạng thái |
|---|---|---|
| Telemetry Contract | `new-contract/telemetry-contract.md` | ✅ Signed 2026-06-26 |
| AI API Contract | `new-contract/ai-api-contract.md` | ✅ Signed 2026-06-26 |
| Deployment Contract | `new-contract/deployment-contract.md` | ✅ Signed 2026-06-26 |

---

---

## 4. Template & Word Count — CDO-02 Specific

### Word Count Tier

| Tier | Word target | Docs CDO-02 |
|---|---|---|
| **Light** | 800-1500 từ | `01_requirements_analysis.md`, `05_cost_analysis.md`, `08_adrs.md` |
| **Medium** | 1000-2500 từ | `02_infra_design.md`, `03_security_design.md`, `04_deployment_design.md`, `07_test_eval_report.md` |

> **Cảnh báo**: `< 500 từ` trong Light/Medium = thiếu depth, không pass. `> word target × 1.5` = fluff hoặc nên split sub-doc.

### 4.1 Template: `01_requirements_analysis.md`

**Tier: Light (800-1500 từ)** — Phân tích đề tài Self-Heal Engine từ CDO/infra perspective.

```markdown
# Requirements Analysis - Task Force 3 Self-Heal Engine - CDO-02

## 1. Bối cảnh đề tài
- Client: VP Engineering, SaaS B2B, 200+ microservices trên EKS
- Problem: On-call quá tải, 2-4 page/đêm, 80% known patterns lặp lại
- Pipeline mong muốn: detect → match runbook → execute audited action → verify → escalate nếu fail

## 2. Phạm vi CDO-02 phụ trách
- Platform architecture, K8s sandbox, multi-tenant isolation
- Safety gate, execution layer, audit, observability
- AI integration: consume 3 contracts, gọi AI endpoint

## 3. Yêu cầu phi chức năng (NFR) cho infra
- Multi-tenant: ≥ 2 tenants, namespace-based isolation
- Auto-resolve rate: ≥ 60% trên ≥ 10 scenarios
- Zero unsafe action
- Audit retention: ≥ 90 ngày
- Safety checkpoints: dry-run, blast-radius, verify, rollback, circuit breaker

## 4. Hướng khác biệt (Differentiation Angle) — KEY

- Angle: K8s-heavy / Kubernetes Workflow Orchestration
- Why: TF3 là bài toán self-heal trên K8s, CDO-02 chọn thao tác trực tiếp K8s workload
- Trade-off: Chi phí + complexity > serverless-first, nhưng sát đề + dễ demo RBAC/isolation

## 5. So sánh với nhóm cùng task force
- CDO khác angle: ... → khác nhau ở chỗ ...
- Cạnh tranh trên reliability + operational control

## 6. Ngoài phạm vi
- Không build AI model
- Không cho AI gọi K8s trực tiếp
- Chỉ sandbox + synthetic workload

```

---

### 4.2 Template: `02_infra_design.md`

**Tier: Medium (1000-2500 từ)** — Architecture chi tiết + K8s-heavy angle deep-dive.

```markdown
# Infrastructure Design - Task Force 3 Self-Heal Engine - CDO-02

## 1. Architecture diagram

Mermaid hoặc PNG — CDO executor, AI Engine, EKS, Safety Gate, Audit, Observability.
Caption bắt buộc + 2-3 dòng giải thích.

## 2. Component table

| Component | Service | Vai trò | Ghi chú |
|---|---|---|---|
| EKS Cluster | Amazon EKS | Kubernetes sandbox | Target chính của self-heal |
| CDO Executor | Pod/Deployment | Orchestrate workflow | CDO own |
| Safety Gate | Module trong executor | Validate tenant, namespace, blast-radius | Chặn unsafe action |
| Kyverno | Admission Webhook | Enforce replicas ≤ 10, memory ≤ 4Gi | Layer 3 defense |
| Audit Storage | S3 Object Lock | Tamper-evident audit | Retention ≥ 90 days |
| DynamoDB | Idempotency Lock | Prevent duplicate execution | Conditional write |
| SQS | Telemetry Buffer | CDO-internal buffer | AI không pull từ SQS |

## 3. Differentiation angle deep-dive — K8s-heavy
- **Tận dụng tối đa K8s-native constructs**: Executor chạy trực tiếp dưới dạng Pod trong cluster, tương tác sát sườn với Kubernetes API để điều khiển trực tiếp workload, deployment và service.
- **Bảo mật và cô lập (Isolation) rõ ràng**: Xử lý Self-Heal trong môi trường Multi-tenant thông qua Namespace-based isolation, áp dụng chặt chẽ RBAC, NetworkPolicy và Kyverno (Safety Gate) để giới hạn Blast-Radius.
- **Khả năng kiểm toán và toàn vẹn dữ liệu**: Kết hợp S3 Object Lock (WORM) và DynamoDB (Idempotency) đảm bảo mọi hành động can thiệp của AI đều có vết, không thể xóa sửa, và không xảy ra tình trạng chạy đè kịch bản.

## 4. Multi-tenant approach
Mô hình hỗ trợ đa khách hàng (Multi-tenant) được triển khai thông qua **cơ chế cô lập theo Namespace (Namespace-based isolation)**. Trong môi trường thực tế, mỗi tenant sẽ được cấp phát một namespace riêng (ví dụ: `tenant-a`, `tenant-b`), trong khi các thành phần cốt lõi của Self-Heal Agent nằm tại `self-heal-system`.
Cấu trúc bảo mật và cô lập được thực hiện qua 3 lớp:
1. **RBAC**: Giới hạn quyền hạn của ServiceAccount, đảm bảo Executor chỉ có thể thao tác (Restart, Patch, Scale) đúng trong namespace của tenant gặp sự cố.
2. **NetworkPolicy**: Ngăn chặn tuyệt đối các luồng giao tiếp chéo giữa các tenant ở tầng network.
3. **Kyverno Admission**: Đóng vai trò như một chốt chặn cuối cùng (Safety Gate), giới hạn tài nguyên cấp phát (ví dụ: cấm vượt quá 10 Replicas hoặc 4Gi Memory) để ngăn chặn hiệu ứng "Noisy neighbor" (tenant này chiếm dụng tài nguyên làm sập tenant khác). Thêm vào đó, cơ chế rate-limit (tối đa 100 RPS cho phase detect và 10 RPS cho quyết định) giúp cân bằng tải hệ thống.

## 5. Alternatives considered (Các phương án thay thế đã cân nhắc)
Trong quá trình thiết kế kiến trúc, chúng tôi đã đánh giá các phương án thay thế sau trước khi chốt hướng đi K8s-heavy:
- **Phương án Serverless-first (dùng AWS Lambda & Step Functions)**: Phương án này có ưu điểm là giảm tải công việc vận hành (ít ops) và dễ dàng scale. Tuy nhiên, chúng tôi từ chối vì nó không bám sát vào tính chất cốt lõi của K8s workload. Việc gọi K8s API từ bên ngoài qua Lambda đòi hỏi phải quản lý Secret và kubeconfig khá phức tạp, đồng thời làm mất đi sự linh hoạt của mô hình GitOps/Operator.
- **Phương án Managed-services heavy (dùng toàn bộ giải pháp trả phí AWS)**: Rất ổn định nhưng khó thể hiện được kỹ năng thiết kế, kiểm soát tài nguyên của người kỹ sư, đồng thời chi phí duy trì hàng tháng sẽ vượt ngân sách dự kiến của bài toán Capstone.
- **Phương án Event-driven hybrid (Microservices Event Sourcing)**: Thiết kế dựa trên Kafka/EventBridge cho mọi component. Phương án này bị loại bỏ vì quá cồng kềnh (over-engineer), không phù hợp với quy mô và thời gian triển khai của dự án.

## 6. Luồng xử lý chính (Data flow)
Luồng dữ liệu và quá trình ra quyết định của Self-Heal Agent diễn ra một cách tuần tự, đảm bảo an toàn tuyệt đối qua từng chốt chặn. 
Đầu tiên, hệ thống ghi nhận **Alert** từ hạ tầng, thu thập **Telemetry** (như logs/metrics) và đưa vào **SQS buffer** để xử lý bất đồng bộ. Sau đó, dữ liệu được truyền cho AI qua endpoint `/v1/detect` để phân loại sự cố. Trước khi đi tiếp, luồng dữ liệu phải vượt qua **Pre-Decide Gate** (nhằm lọc bỏ rác và thông tin nhiễu). 
Khi AI đưa ra giải pháp khắc phục qua `/v1/decide`, lệnh này bị tạm giữ tại **Safety Gate**. Tại đây, hệ thống thực hiện `dry-run` (chạy thử nghiệm giả lập) nhằm đánh giá "blast-radius" (phạm vi ảnh hưởng). Chỉ khi hệ thống xác nhận hành động này nằm trong mức an toàn cho phép, lệnh mới được chính thức **execute** xuống cluster. Cuối cùng, hệ thống gọi `/v1/verify` để đánh giá kết quả phục hồi và đẩy toàn bộ lịch sử thao tác vào hệ thống **Audit**.

## 7. Failure modes + recovery (Xử lý sự cố và phục hồi)
Hệ thống được thiết kế với cơ chế tự bảo vệ và phục hồi linh hoạt (Resilience) trong mọi tình huống rủi ro:
- **Khi AI bị Timeout hoặc lỗi 503:** Executor sẽ bắt lỗi HTTP Timeout (thường cấu hình dưới 5s). Ngay lập tức, hệ thống hủy bỏ chu trình tự động (No execute), ghi log cảnh báo vào hệ thống Audit và leo thang (escalate) trực tiếp cho kỹ sư trực ca (On-call Engineer). Thời gian phục hồi (RTO) ở mức < 60s.
- **Khi Safety Gate từ chối hành động (Deny):** Nếu AI đề xuất một thao tác vượt quá giới hạn (ví dụ: scale lên 100 pods), Gate Check sẽ bắt được lỗi này và từ chối thực thi ngay lập tức (Immediate RTO). Hệ thống sẽ escalate sự cố và ghi nhận toàn bộ payload của AI vào Audit log phục vụ quá trình Post-mortem sau này.
- **Khi quá trình Execute bị lỗi K8s API:** Trong trường hợp thao tác thực thi bị từ chối do K8s cluster (như conflict state hoặc rớt mạng), module Circuit Breaker (cầu dao tự động) sẽ được kích hoạt. Lệnh thực thi sẽ bị ngắt hoàn toàn và hệ thống sẽ tự động gọi luồng Rollback để trả môi trường về trạng thái ổn định ban đầu (RTO < 120s).
```

---

### 4.3 Template: `03_security_design.md`

**Tier: Medium (1000-2500 từ)** — hoặc Heavy nếu đủ depth.

```markdown
# Security Design - Task Force 3 Self-Heal Engine - CDO-02

## 1. IAM model

- IRSA (IAM Roles for Service Accounts) cho executor + AI Engine
- Least privilege: executor chỉ có quyền restart/scale/patch trong allowed namespaces
- Per-tenant RBAC Role/RoleBinding
- Kubernetes RBAC:
  - Executor ServiceAccount trong namespace `self-heal-system`
  - Role: chỉ có verbs cần thiết (get, list, patch, delete pods)
  - RoleBinding scoped theo tenant namespace (`tenant-a`, `tenant-b`)
  - ClusterRole không dùng — tránh quyền quá rộng

## 2. Secrets management
- AWS Secrets Manager / K8s Secrets cho API keys, DB credentials
- Rotation policy: manual rotation cho sandbox, target auto-rotation production
- Không lưu kubeconfig/secret trong repo, log hoặc container image
- K8s Secrets mount qua volume, không qua env vars (giảm exposure risk)
- `.gitignore` + `gitleaks` scan để chặn secret commit

## 3. Network policy

- VPC topology: private subnets, NAT gateway, VPC endpoints (S3/DynamoDB)
- NetworkPolicy chặn **inter-tenant communication** (tenant-a ↔ tenant-b blocked)
- AI Engine chỉ nhận traffic từ executor (namespace `self-heal-system`)
- Security groups: EKS node SG chỉ allow internal cluster traffic
- WAF/Shield: không implement trong sandbox, documented cho production consideration

## 4. Audit trail

- Format: JSON schema, keyed by `correlation_id`
- Storage: S3 Object Lock (Governance mode), retention ≥ 90 ngày
- Query: CloudWatch Logs Insights / Athena
- Mọi action (detect/decide/execute/verify/escalate/deny) đều ghi audit record

## 5. Compliance touch
- **SOC2 controls touched:**
  - CC6.1 (Logical access): RBAC least privilege + IRSA + namespace isolation
  - CC7.2 (System monitoring): CloudWatch Logs + Container Insights + audit trail
  - CC8.1 (Change management): ArgoCD GitOps + git history = audit trail cho mọi thay đổi
- **Data residency:** Tất cả data trong `us-east-1`, không cross-region replication trong sandbox
- **GDPR-style tenant data deletion + retention:**
  - Tenant data isolated theo namespace → xóa namespace = xóa toàn bộ workload data
  - Audit logs giữ ≥ 90 ngày (S3 Object Lock Governance) → sau retention period có thể delete
  - DynamoDB idempotency records TTL auto-delete sau 24 giờ
  - Không lưu PII trong audit log (chỉ `tenant_id`, `correlation_id`, action metadata)

## 6. Safety Gate (app-level security — CDO-02 specific)

- Validate: `tenant_id` match, namespace trong `allowed_namespaces`
- Blast-radius check: replicas ≤ 10, memory ≤ 4Gi
- Verify: local rollback path + `verify_policy` bắt buộc trước execute
- Kyverno admission webhook: layer 3 defense (cluster-level, independent từ executor code)

## 7. Threat model (STRIDE)
| Threat | Component | Mitigation |
|---|---|---|
| Spoofing | AI endpoint | K8s NetworkPolicy + ServiceAccount |
| Tampering | Audit log | S3 Object Lock (Governance mode) |
| Repudiation | Execute action | `correlation_id` trace end-to-end |
| Information Disclosure | Cross-tenant | RBAC + namespace isolation + NetworkPolicy |
| Denial of Service | Executor | Circuit breaker + rate limit per tenant |
| Elevation of Privilege | K8s RBAC | Least privilege + Kyverno admission |

## 8. Incident response runbook (high-level)
- **Detection**: CloudWatch Alarms + executor error logs + Kyverno deny events
- **Containment**: Circuit breaker halt executor + isolate affected namespace
- **Eradication**: Identify root cause via `correlation_id` trace + audit logs
- **Recovery**: ArgoCD rollback hoặc manual kubectl restore
- **Post-mortem**: Document trong ADR nếu cần design change
```

---

### 4.4 Template: `04_deployment_design.md`

**Tier: Medium (1000-2500 từ)** — IaC + CI/CD + GitOps.

```markdown
# Deployment & CI/CD Design - Task Force 3 Self-Heal Engine - CDO-02

## 1. IaC strategy

- Tool: Terraform >= 1.10
- Module structure: vpc/, eks/, iam/, observability/, audit/, kyverno/, argocd/
- State: S3 backend (target), hiện đang local state (known gap)

## 2. CI/CD pipeline

- GitHub Actions: lint → test → build → scan → deploy
- Quality gates: Terraform plan review, container image scan

## 3. GitOps — ArgoCD

- ArgoCD sync manifests/ → EKS cluster
- Sync waves: namespaces → RBAC → workloads → executor → AI Engine
- Drift detection: ArgoCD auto-sync

## 4. Deployment strategy

- Deferred actions: Git commit → ArgoCD sync (GitOps path)
- Urgent actions: Direct K8s API (RTO < 60s)
- Rollback: ArgoCD rollback hoặc executor local rollback

## 5. Environment separation
- Sandbox (duy nhất trong capstone)
- Production considerations documented nhưng không implement

## 6. Secrets in pipeline
- IRSA: không cần AWS credentials trong pipeline
- K8s secrets: managed qua ArgoCD sealed secrets hoặc External Secrets
```

---

### 4.5 Template: `05_cost_analysis.md`

**Tier: Light (800-1500 từ)** — Pack #2 phải có **measured actual**.

```markdown
# Cost Analysis - Task Force 3 Self-Heal Engine - CDO-02

## 1. Cost model per component

| Component | Unit cost | Usage (sandbox 10 days) | Total |
|---|---|---|---|
| EKS Cluster | $0.10/h | 240h | $24.00 |
| EC2 t3.medium × 2 | $0.0416/h/node | 240h | $20.00 |
| NAT Gateway | $0.045/h | 240h | $10.80 |
| S3 Audit | $0.023/GB | ~500MB | $0.04 |
| DynamoDB | On-Demand | ~5K WCU/day | $0.06 |
| SQS | Free tier | ~100K msg/day | $0.00 |
| CloudWatch | $0.50/GB ingested | ~5GB | $2.50 |
| **Total sandbox** | | | **~$80-90** |

## 2. Cost per tenant (production forecast)
| Tenants | Monthly estimate | Per-tenant |
|---|---|---|
| 2 | ~$160 | ~$80 |
| 10 | ~$400 | ~$40 |
| 50 | ~$1200 | ~$24 |

## 3. Cost optimization applied
- Single NAT Gateway (thay vì per-AZ) cho sandbox
- VPC Gateway Endpoints cho S3/DynamoDB (miễn phí)
- DynamoDB On-Demand + TTL auto-delete 24h
- SQS Free Tier

## 4. Cost vs alternatives (cùng task force)
- CDO khác: ước tính $X/tenant
- CDO-02: ước tính ~$80/tenant/sandbox — trade-off: EKS cluster cost cao nhưng sát đề K8s-heavy

## 5. Measured actual (capstone) — W12 REQUIRED

- AWS Cost Explorer data split by service
- Screenshot hoặc CSV từ AWS Billing
```

---

### 4.6 Template: `07_test_eval_report.md`

**Tier: Medium (1000-2500 từ)** — Pack #2 mới bắt buộc.

```markdown
# Test & Eval Report - Task Force 3 Self-Heal Engine - CDO-02

## 1. Test coverage

| Type | Tool | Scope | Status |
|---|---|---|---|
| Contract test | JSON schema | /v1/detect, /v1/decide, /v1/verify | Planned → Done |
| Safety unit test | pytest | allow-list, tenant match, blast-radius | Planned → Done |
| Integration test | Mock AI + executor | Full workflow | Planned → Done |
| E2E scenario test | Scenario injector | ≥10 scenarios, ≥4h window | Planned → Done |
| Load test | k6/Locust | Sustained flow | Planned → Done |
| Security test | RBAC abuse cases | Cross-tenant deny | Planned → Done |

## 2. Test Case Matrix (≥10 scenarios)

| ID | Scenario | Tenant | Expected result |
|---|---|---|---|
| TC-01 | Service stuck / latency spike | tenant-a | Auto-resolved via RESTART_DEPLOYMENT |
| TC-02 | Error rate spike | tenant-a | Auto-resolved via RESTART_DEPLOYMENT |
| TC-03 | OOM / memory pressure | tenant-b | Auto-resolved via PATCH_MEMORY_LIMIT |
| TC-04 | Secret/cert expiry | tenant-a | Deferred via ROTATE_SECRET (GitOps) |
| TC-05 | Queue backlog (synthetic) | tenant-b | Deferred via SCALE_REPLICAS (GitOps) |
| TC-06 | Duplicate scenario | tenant-a | Deduplicated via idempotency |
| TC-07 | Cross-tenant attempt | tenant-b→tenant-a | DENIED by safety gate |
| TC-08 | AI timeout/503 | tenant-a | No execute, escalate + audit |
| TC-09 | Low confidence response | tenant-b | No action, log warning |
| TC-10 | Disallowed namespace | tenant-a | DENIED by safety gate |

## 3. SLO Evidence

| SLO | Target | Measured | Pass/Fail |
|---|---|---|---|
| Executor availability | ≥ 99.5% | TBD | TBD |
| AI detect p99 | < 300ms | TBD | TBD |
| AI decide p99 | < 3000ms (LLM) | TBD | TBD |
| E2E auto-heal latency | < 5 min | TBD | TBD |
| Unsafe action rate | 0% | TBD | TBD |
| Auto-resolve rate | ≥ 60% | TBD | TBD |
| Audit coverage | 100% | TBD | TBD |

## 4. Chaos test results (Curveball — W12)

- Curveball #1 (small): ... response + outcome
- Curveball #2 (medium): ... response + outcome
- Curveball #3 (chaos): ... response + outcome

## 5. Security test

- Cross-tenant deny: confirmed via TC-07
- RBAC least privilege: verified
- Secret exposure check: passed

## 6. Failure analysis
| Failure | Root cause | Fix | Final status |
|---|---|---|---|
| TBD | TBD | TBD | TBD |

## 7. Load test results

- Tool: k6/Locust
- Synthetic load: X concurrent scenarios
- Observed behavior: ...
```

---

### 4.7 Template: `08_adrs.md`

**Tier: Light (800-1500 từ)** — Append-only, ≥3 ADRs cho Pack #1, ≥5 ADRs cho Pack #2.

```markdown
# Architecture Decision Records - CDO-02

> **Hướng dẫn Evidence:** Chứng minh quá trình ra quyết định.
> Lệnh: `git log --pretty=format:"%h %ad %s" --date=short -- docs/08_adrs.md`
> Paste git log vào để chứng minh ADR được update theo time.

## ADR-NNN - <Short title>
- **Status**: Proposed | Accepted | Superseded | Rejected
- **Date**: YYYY-MM-DD
- **Context**: 1-3 câu tại sao có decision này
- **Decision**: chốt cụ thể gì
- **Consequences**: trade-off + impact downstream
- **Alternatives considered**: bullet list với pros/cons từng option

| ADR | Title | Status |
|---|---|---|
| ADR-001 | Chọn K8s-heavy / Kubernetes Workflow Orchestration | Accepted |
| ADR-002 | AI là decision service, CDO executor là execution boundary | Accepted |
| ADR-003 | Namespace-based tenant isolation + RBAC least privilege | Accepted |
| ADR-004 | CDO self-host AI Engine container trong EKS | Accepted |
| ADR-005 | Kyverno admission webhook thay vì OPA Gatekeeper | Accepted |
| ADR-006 | S3 Object Lock Governance mode thay vì Compliance mode | Accepted |
| ADR-007 | SQS làm CDO-internal telemetry buffer | Accepted |
```

---

## 5. Format Conventions — CDO-02

### File format
- **Markdown only** (`.md`) — không Google Doc / PDF rời rạc
- File live trong repo `docs/`, không attach binary
- Char encoding UTF-8

### Diagrams
- **Mermaid** preferred — embed inline, version-controllable
- PNG/draw.io export OK → place trong `docs/assets/` hoặc `docs/docs_ObservabilityStack/picture/`
- **BẮT BUỘC**: mọi diagram phải có caption + 2-3 dòng giải thích bên dưới

### Code blocks
- Dùng fenced code blocks với language tag: ` ```yaml `, ` ```python `, ` ```hcl `
- Doc chỉ chứa pseudo-code/snippet; full implementation ở `executor/` hoặc `infra/`

### ADR format (strict)
```markdown
## ADR-NNN - <Short title>
- **Status**: Proposed | Accepted | Superseded | Rejected
- **Context**: 1-3 câu tại sao
- **Decision**: chốt cụ thể gì
- **Consequence**: trade-off + impact downstream
- **Alternatives considered**: bullet list
```

### Cross-references
- Refer docs qua relative path: `[xem 02_infra_design.md](02_infra_design.md)`
- Refer ADR: `ADR-005`
- Refer contracts: `[AI API Contract](../new-contract/ai-api-contract.md)`

---

---

## 6. Repo Structure — CDO-02 Actual

```
TF3-Self-Heal-Agent-AWS/
├── docs/                                    # Tất cả 7 evidence docs
│   ├── 01_requirements_analysis.md          # Ready
│   ├── 02_infra_design.md                   # Ready
│   ├── 03_security_design.md                # Ready
│   ├── 04_deployment_design.md              # Ready
│   ├── 05_cost_analysis.md                  # Draft → W12 measured
│   ├── 07_test_eval_report_v2.0.md      # Draft → W12 evidence
│   ├── 08_adrs.md                           # Ready (7 ADRs)
│   ├── standup_notes.md                     # Standup notes
│   └── docs_ObservabilityStack/             # Observability docs + diagrams
│
├── executor/                                # CDO Self-Heal Executor (Python)
│   ├── main.py                              # Main orchestrator
│   ├── ai_client.py                         # AI endpoint client
│   ├── safety_gate.py                       # Safety validation
│   ├── pre_decide_gate.py                   # Pre-decide filtering
│   ├── audit.py                             # Audit logging
│   ├── k8s_client.py                        # Kubernetes client
│   ├── circuit_breaker.py                   # Circuit breaker
│   ├── idempotency.py                       # Idempotency lock
│   ├── escalation.py                        # Escalation logic
│   ├── executors/                           # Action executors
│   ├── scenarios/                           # Test scenarios
│   ├── tests/                               # Unit tests
│   └── mock_ai_server.py                    # Mock AI for testing
│
├── infra/                                   # Terraform IaC
│   ├── modules/
│   │   ├── vpc/                             # VPC, subnets, route tables
│   │   ├── eks/                             # EKS cluster, node groups
│   │   ├── iam/                             # IRSA, policies
│   │   ├── observability/                   # CloudWatch, alarms
│   │   ├── audit/                           # S3 Object Lock, DynamoDB, SQS
│   │   ├── kyverno/                         # Kyverno Helm release
│   │   ├── argocd/                          # ArgoCD Helm release
│   │   ├── ecr/                             # ECR registry
│   │   └── secrets/                         # Secrets management
│   ├── envs/dev/                            # Sandbox environment wiring
│   └── bootstrap/                           # Bootstrap scripts
│
├── manifests/                               # K8s Manifests (ArgoCD managed)
│   ├── namespaces/                          # tenant-a, tenant-b, self-heal-system
│   ├── rbac/                                # RBAC roles/bindings
│   ├── networkpolicies/                     # Inter-tenant block
│   ├── workloads/                           # Sample workloads (Online Boutique)
│   ├── executor/                            # CDO executor deployment
│   ├── ai-engine/                           # AI Engine deployment
│   ├── kyverno/                             # Kyverno policies
│   └── argocd/                              # ArgoCD configs
│
├── new-contract/                            # AI Contracts (signed)
│   ├── telemetry-contract.md
│   ├── ai-api-contract.md
│   └── deployment-contract.md
│
├── CAPSTONE_GUIDE/                          # Templates & references
│   ├── CAPSTONE_EVIDENCE_PACK_FORMAT.md     # File gốc (template)
│   └── TF3_SELFHEAL_LEARNER.md              # Đề tài chi tiết
    ├── templates/                            # Doc templates
    └── EVIDENCE_PACK_CDO02_TF3.md           # FILE NÀY
```

---

---

## 7. Checklist Submission — CDO-02 Specific

### Progress #1 (EOD T4 W11) — light

- [x] `01_requirements_analysis.md` (draft) — K8s-heavy angle declared
- [x] `02_infra_design.md` (draft + angle declared + multi-tenant approach)
- [x] `08_adrs.md` (≥2 ADRs) — hiện có 7 ADRs

### Evidence Pack #1 (EOD T6 W11) — MAIN 

- [x] `01_requirements_analysis.md` — ready, comprehensive
- [x] `02_infra_design.md` (with multi-tenant approach) — ready
- [x] `03_security_design.md` (draft) — ready
- [x] `04_deployment_design.md` (draft) — ready
- [x] `05_cost_analysis.md` (skeleton) — draft available
- [x] `08_adrs.md` (≥3 ADRs) — 7 ADRs available
- [x] Base infra (VPC + EKS + Observability) chạy được — Terraform modules exist

### Progress #2 (EOD T2 W12) — light

- [ ] AI engine integration started — deploy AI container vào EKS
- [ ] Tenant onboarding flow draft — `tenant-a`, `tenant-b` namespaces + RBAC
- [ ] Docs updated với progress notes

### Evidence Pack #2 (EOD T4 W12) — MAIN + code freeze 18h

- [ ] All 7 docs **final**
- [ ] `05_cost_analysis.md` **measured** — AWS Cost Explorer actual data
- [ ] `07_test_eval_report.md` **new** với:
  - [ ] ≥10 scenario test results
  - [ ] ≥4h simulation window evidence
  - [ ] ≥60% auto-resolve rate
  - [ ] 0 unsafe action
  - [ ] SLO measured values (không còn TBD)
  - [ ] Chaos/curveball response evidence
  - [ ] Failure analysis cho scenarios fail
- [ ] `08_adrs.md` final (≥5 ADRs) — hiện có 7, cần thêm curveball ADRs
- [ ] Platform infra deployed + integrated với AI engine
- [ ] E2E demo evidence: screenshot/log/video
- [ ] `git tag final` trên repo

---

---

## 8. Bảng theo dõi tiến độ & Checklist cho CDO-02 (Tracking Matrix)

Hãy đánh dấu `[x]` vào cột cuối cùng khi hoàn thành.

| Mốc thời gian | Tài liệu / Nhiệm vụ (Files & Tasks) | Trạng thái yêu cầu | Ghi chú & Trọng tâm cho CDO-02 (Focus Angle) | Done |
| :--- | :--- | :--- | :--- | :---: |
| **Progress #1**<br>*(EOD T4 W11)* | `01_requirements_analysis.md` | Draft | Làm rõ Scope và cơ chế Safety Gate. | [ ] |
| *(Checkpoint nhẹ)* | `02_infra_design.md` | Draft | Khai báo hướng đi "K8s-heavy Orchestration" & Multi-tenant. | [ ] |
| | `08_adrs.md` | Draft | Ghi nhận ít nhất **≥2 ADRs** ban đầu. | [ ] |
| | Hợp đồng (Contracts) | Reviewing | Tham gia cùng team AI chốt nháp 3 hợp đồng (API, Telemetry, Deploy). | [ ] |
| | | | | |
| **Evidence #1** ⭐<br>*(EOD T6 W11)* | `01_requirements_analysis.md` | **Final** | Hoàn thiện bản cuối cùng. | [ ] |
| *(Cột mốc chính 1)* | `02_infra_design.md` | **Final** | Chốt sơ đồ kiến trúc K8s, IAM, Network (kèm hình ảnh). | [ ] |
| | `03_security_design.md` | Draft | Phải có cấu trúc chuẩn SOC2, S3 Object Lock (WORM). | [ ] |
| | `04_deployment_design.md` | Draft | Phác thảo luồng CI/CD và chiến lược Terraform (IaC). | [ ] |
| | `05_cost_analysis.md` | Skeleton | Dựng khung sườn các mục chi phí cần đo lường. | [ ] |
| | `08_adrs.md` | Draft | Ghi nhận ít nhất **≥3 ADRs**. | [ ] |
| | Hợp đồng (Contracts) | **Signed** | 3 hợp đồng đã ký (từ T5) và đóng băng (Freeze). | [ ] |
| | **Thực hành (Code)** | **Working** | Base infra (VPC + EKS Cluster + Observability) deploy thành công. | [ ] |
| | | | | |
| **Progress #2**<br>*(EOD T2 W12)* | `07_test_eval_report.md` | Skeleton | Lên khung các kịch bản test (Chaos test, Load test). | [ ] |
| *(Checkpoint nhẹ)* | Luồng Onboarding | Draft | Phác thảo luồng thêm Tenant mới. | [ ] |
| | Tích hợp hệ thống | Doing | Bắt đầu ráp nối gọi API thực tế vào endpoint của AI Engine. | [ ] |
| | | | | |
| **Evidence #2** ⭐<br>*(EOD T4 W12)* | **TẤT CẢ TÀI LIỆU (01 → 08)**| **Final** | **Đã review chéo, đúng Word count và không có lỗi chính tả.** | [ ] |
| *(Cột mốc chính 2)* | `05_cost_analysis.md` | **Final** | Chi phí đo lường thật (Measured từ AWS), KHÔNG đoán mò. | [ ] |
| | `07_test_eval_report.md` | **Final** | Chứa evidence (log/hình ảnh) test chịu lỗi & Fallback khi AI sập. | [ ] |
| | `08_adrs.md` | **Final** | Chốt ít nhất **≥5 ADRs**. | [ ] |
| | **Thực hành (Code)** | **E2E Done**| Platform hoàn chỉnh + Tích hợp thành công từ đầu đến cuối với AI. | [ ] |
| *(18h00)* | **Git Versioning** | **`final`** | Đóng băng code (Code freeze), gắn tag `final` trên repo. | [ ] |

---

---

## Appendix: Quick Reference Links

| Resource | Path |
|---|---|
| Template gốc | `CAPSTONE_GUIDE/CAPSTONE_EVIDENCE_PACK_FORMAT.md` |
| Đề tài chi tiết | `CAPSTONE_GUIDE/TF3_SELFHEAL_LEARNER.md` |
| CDO Templates | `capstone-phase2/templates/cdo/docs/` |
| AI Contracts | `new-contract/` |
| Executor Code | `executor/` |
| Terraform IaC | `infra/` |
| K8s Manifests | `k8s/` |
| Existing Docs | `docs/` |
| W12 Tasks | `W12_TASKS.md` |

---

> Note **File này là bản customize cho CDO-02 TF3**. Khi chỉnh sửa docs, luôn đối chiếu với template gốc tại `CAPSTONE_GUIDE/CAPSTONE_EVIDENCE_PACK_FORMAT.md` để đảm bảo không thiếu section nào.
