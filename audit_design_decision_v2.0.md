# Feedback Audit Để Chốt Thiết Kế Cuối - TF3 Self-Heal Agent AWS / CDO-02

Ngày lập: 2026-06-26  
Repo audit: `/mnt/g/XBrain/CDO-02_capstone/TF3-Self-Heal-Agent-AWS`  
Nguồn contract chính thức: `contract - new 3/`  
Mục đích file: feedback để team cân nhắc chốt thiết kế audit cuối cùng.

---

## 1. Kết Luận Nhanh

Đề xuất trạng thái:

```text
APPROVE WITH CONDITIONS
```

Có thể chốt baseline audit theo hướng:

```text
S3 Object Lock = audit source of truth
CloudWatch Logs = runtime log/debug
Athena/S3 query = truy vấn evidence theo correlation_id
```

Ranh giới thiết kế nên tách rõ: audit core chỉ gồm nơi lưu bằng chứng và cách truy vấn bằng chứng. 

Cập nhật theo `contract - new 3`: idempotency lock chỉ áp dụng cho `/v1/decide`, `rollback_snapshot` cần được audit bằng ref/hash, và RBAC least-privilege của CDO Controller cần có evidence khi review.

Điều kiện cần bổ sung trước khi chốt final:

1. Chốt rõ **Governance Mode vs Compliance Mode**.
2. Thêm **audit event schema**.
3. Thêm **S3 key prefix convention**.
4. Thêm **audit test case/evidence checklist** gắn với TC-01 đến TC-17.
5. Thêm **redaction rule** cho PII/secret trong telemetry/log/audit.

---

## 2. Mục Đích Audit Trong Dự Án Này

Audit trong Self-Heal Engine tạo bằng chứng để trả lời:

- Incident nào đã xảy ra?
- AI đã detect/decide như thế nào?
- CDO đã kiểm tra safety gate ra sao?
- Action có được execute thật không?
- Nếu execute thì kết quả verify thế nào?
- Nếu deny/rollback/escalate thì lý do là gì?
- Có đảm bảo không làm sai tenant, không duplicate, không unsafe action không?

Nói ngắn gọn:

```text
Audit = bằng chứng bất biến cho toàn bộ quá trình self-heal.
```

Một incident hợp lệ nên có trace theo `correlation_id`:

```text
alert_received
telemetry_collected
detect_called
detect_response_received
decide_called
action_plan_received
safety_passed / safety_denied
dry_run_done
execute_done / denied / escalated
verify_called
verify_done
incident_closed / rollback_done / escalated
```

---

## 3. Scope Audit Cần Chốt

Scope audit cần bao phủ toàn bộ self-heal loop:

```text
alert/scenario
-> telemetry collected
-> AI /v1/detect
-> AI /v1/decide
-> CDO safety gate
-> dry-run
-> execute Kubernetes API hoặc GitOps/ArgoCD
-> AI /v1/verify
-> close / retry / rollback / escalate
-> audit trail
```

Nguồn xác định scope:

| Nguồn | Kết luận audit lấy từ nguồn |
|---|---|
| `contract - new 3/ai-api-contract.md` | CDO nằm giữa `/v1/decide` và `/v1/verify`, nên CDO phải audit AI decision, safety decision, execution result và verify result. New3 làm rõ `Idempotency-Key` bắt buộc ở cả 3 endpoint, nhưng lock chỉ áp dụng cho `/v1/decide`. |
| `contract - new 3/deployment-contract.md` | AI self-hosted in-cluster, AI không có Kubernetes API/kubeconfig; audit tamper-evident bằng S3 Object Lock 90 ngày; bổ sung RBAC least-privilege chi tiết cho CDO Controller. |
| `contract - new 3/telemetry-contract.md` | Telemetry cần có tenant scope, timestamp chuẩn, labels/topology và PII/secret scrubbing; audit cần ghi đủ trạng thái telemetry trong luồng self-heal. |
| `docs/01_requirements_analysis.md` | CDO scope gồm platform architecture, Kubernetes sandbox, tenant isolation, safety gate, execution layer, audit, observability, IaC, AI integration. |
| `docs/02_infra_design.md` | Luồng xử lý có urgent path bằng Kubernetes API và deferred path bằng GitOps/ArgoCD. |
| `docs/03_security_design.md` | Security goals: zero unsafe action, tenant isolation, least privilege, audit tamper-evident, trace theo `correlation_id`. |
| `docs/07_test_eval_report_v1.0_Duc.md` | Acceptance: >=10 scenarios, >=4h simulation, auto-resolve >=60%, unsafe action 0, audit coverage 100%. |
| `infra/modules/audit/main.tf` | IaC hiện tại có S3 Object Lock Governance 90 ngày, versioning, AES256 encryption và public access block. |

---

## 4. Tiêu Chí Audit Để Chốt Final

### 4.1 Must-have

| Tiêu chí | Mức chốt |
|---|---|
| Audit source of truth | S3 Object Lock, retention >= 90 ngày. |
| Audit coverage | 100% incident/scenario có trace theo `correlation_id`. |
| Traceability | Mỗi event quan trọng có `correlation_id`, `tenant_id`, timestamp, decision, result, reason. |
| Safety evidence | Mỗi deny/execute/rollback/escalate phải có reason rõ ràng. |
| Tenant isolation | Cross-tenant target phải deny 100%, không có Kubernetes mutation. |
| Contract alignment | Dùng `contract - new 3/`; bỏ qua contract cũ. |
| PII/secret protection | Telemetry/log/audit phải redact token, password, connection string, email/PII nếu có. |
| Query evidence | Có cách query audit theo `correlation_id`, tối thiểu bằng S3 prefix hoặc Athena. |

### 4.2 Should-have

| Tiêu chí | Mức chốt |
|---|---|
| Audit event schema | Có schema/version riêng cho audit event. |
| S3 key convention | Có format key ổn định theo date/tenant/correlation/event. |
| Athena/Glue table | Có schema query audit JSON nếu kịp. |
| SSE-KMS | Tốt hơn AES256 mặc định nếu kịp. |
| Payload hash | Có `payload_hash` để tăng integrity proof nếu kịp. |

### 4.3 Ranh Giới Với Các Thành Phần Runtime

| Hạng mục | Lý do |
|---|---|
| DynamoDB | Nếu có thì là runtime/control cho idempotency, không phải audit storage. Theo new3, lock chỉ áp dụng cho `/v1/decide`; audit chỉ cần ghi `Idempotency-Key`, endpoint liên quan và kết quả duplicate/pass. |
| SQS/DLQ | Nếu có thì là telemetry buffer/malformed handling, không phải audit trail. Audit chỉ cần ghi event telemetry validation/enqueue/failure nếu liên quan incident. |
| OpenSearch | Tốt cho search dashboard nhưng cost/complexity cao, không phải WORM source of truth. |
| CloudWatch-only | Để debug, nhưng không đủ tamper-evident. |
| Database audit table only | Query tốt nhưng không WORM nếu không export sang S3 Object Lock. |

---

## 5. Feedback Về Thiết Kế Hiện Tại

### 5.1 Điểm Nên Giữ

| Hạng mục | Feedback |
|---|---|
| S3 Object Lock audit storage | Nên giữ. Đây là điểm mạnh nhất để trả lời yêu cầu tamper-evident và retention 90 ngày. |
| CDO-centric audit | Nên giữ. CDO là execution boundary nên có đầy đủ context nhất để audit. |
| CloudWatch Logs | Nên giữ làm operational log/debug, nhưng không coi là audit source of truth. |
| Safety Gate + RBAC + Kyverno | Nên giữ. Audit nên ghi rõ pass/deny của các lớp bảo vệ này. |
| RBAC least-privilege cho CDO Controller | Nên giữ và đối chiếu theo new3. Audit nên có evidence khi action bị chặn do thiếu quyền hoặc sai namespace. |
| GitOps path cho deferred action | Nên giữ. Audit cần ghi commit hash/ArgoCD sync khi action đi deferred path. |

### 5.2 Điểm Cần Sửa Trước Khi Chốt

| Ưu tiên | Vấn đề | Vì sao quan trọng | Đề xuất |
|---|---|---|---|
| P0 | Chưa có audit event schema riêng | Reviewer khó biết audit record có đủ trường hay không | Thêm `docs/06_audit_design.md` hoặc `contract/audit-event-contract.md`. |
| P0 | Chưa có S3 key prefix convention | Query/evidence theo incident sẽ rối | Chốt prefix theo date/tenant/correlation/event. |
| P0 | Governance vs Compliance Mode đang lệch contract | `contract - new 3` ghi Compliance, IaC hiện tại Governance | Ghi ADR/decision note rõ ràng. |
| P0 | Chưa có audit checklist gắn với TC-01 -> TC-17 | Khó chứng minh audit coverage 100% | Thêm bảng expected audit events cho từng test case. |
| P0 | Chưa phản ánh rõ idempotency scope theo new3 | New3 quy định lock chỉ áp dụng cho `/v1/decide`; `/detect` và `/verify` chỉ dùng key để trace audit | Sửa audit schema/test để phân biệt `idempotency_trace` và `duplicate_check` tại `/v1/decide`. |
| P0 | Chưa audit rõ `rollback_snapshot` | New3 yêu cầu `rollback_snapshot` trong `/v1/decide` để phục vụ `next_action=ROLLBACK` | Audit cần ghi `rollback_snapshot_ref` hoặc hash, không nhất thiết lưu full payload nhạy cảm. |
| P1 | Chưa formalize redaction policy | Telemetry contract yêu cầu scrub PII/secret | Thêm rule redact token/password/email/connection string. |
| P1 | Chưa có Athena/Glue schema | Query audit có thể bị manual | Nếu kịp, thêm Athena DDL cho audit JSON. |
| P1 | AES256 thay vì SSE-KMS | Chấp nhận sandbox, nhưng security reviewer có thể hỏi | Ghi rõ sandbox accepted; production nên SSE-KMS key riêng. |
| P2 | Chưa có payload hash | S3 Object Lock đã WORM, nhưng chưa có integrity hash | Thêm `payload_hash` nếu còn thời gian. |

---

## 6. Governance Mode vs Compliance Mode

Đây là điểm cần chốt rõ nhất trước final.

| Lựa chọn | Phù hợp với | Điểm mạnh | Rủi ro |
|---|---|---|---|
| Governance Mode | Capstone/sandbox | Có Object Lock 90 ngày, admin có thể bypass khi cần cleanup | Lệch với `contract - new 3/deployment-contract.md` nếu không ghi rõ exception. |
| Compliance Mode | Production/strict compliance | Khớp contract mới, mạnh nhất về WORM | Sandbox có thể bị kẹt object/bucket, admin cũng không xóa được trước retention. |

Đề xuất chốt:

```text
Capstone sandbox:
- S3 Object Lock Governance Mode, 90-day retention.
- Lý do: theo trainer feedback và cần khả năng cleanup/unlock trong sandbox.
- Ghi rõ đây là controlled deviation so với deployment contract.

Production recommendation:
- S3 Object Lock Compliance Mode hoặc cross-account audit bucket.
- SSE-KMS key riêng.
- Bypass/delete permission tách khỏi role vận hành thông thường.
```

Nếu checklist cuối bắt buộc khớp `contract - new 3` 100%, đổi sang Compliance Mode. Nếu checklist chấp nhận trainer feedback/sandbox cleanup, giữ Governance Mode nhưng phải có decision note.

---

## 7. Phương Án Audit So Sánh

| Phương án | Đạt scope? | Đạt tamper-evident? | Cost | Complexity | Khuyến nghị |
|---|---|---:|---:|---:|---|
| A. S3 Object Lock Governance + CloudWatch/Athena | Tốt | Tốt cho sandbox | Thấp | Vừa | **Chọn cho capstone nếu có decision note** |
| B. S3 Object Lock Compliance + CloudWatch/Athena | Tốt | Mạnh nhất | Thấp | Vừa/Cao do cleanup risk | Chọn nếu cần strict contract |
| C. CloudWatch-only | Thiếu | Yếu | Trung bình | Thấp | Không chọn |
| D. Database audit table | Tốt cho query | Yếu nếu không export WORM | Thấp/Trung bình | Vừa | Không chọn làm audit chính |
| E. OpenSearch audit | Tốt cho search | Không WORM mặc định | Cao | Cao | Không chọn cho capstone |
| F. S3 Object Lock + payload hash/hash-chain | Tốt | Mạnh | Thấp | Vừa | Stretch nếu kịp |

Kết luận:

- Chốt theo capstone/sandbox: **Option A**.
- Chốt theo contract strict: **Option B**.
- Không chốt CloudWatch-only, database-only, OpenSearch-only làm audit chính.

---

## 8. Audit Event Schema Đề Xuất

Nên thêm schema tối thiểu như sau:

```json
{
  "schema_version": "audit.v1",
  "event_id": "uuid-v4",
  "event_type": "safety_passed",
  "timestamp": "2026-06-26T00:00:00.000Z",
  "correlation_id": "uuid-v4",
  "idempotency_key": "uuid-v4",
  "tenant_id": "uuid-v4",
  "namespace": "tenant-a",
  "deployment": "deployment/cdo-sample-api",
  "service": "checkout-svc",
  "actor": "cdo-self-heal-controller",
  "ai_endpoint": "/v1/decide",
  "action_type": "RESTART_DEPLOYMENT",
  "pattern_type": "urgent",
  "decision": "execute",
  "result": "success",
  "reason": "all_safety_checks_passed",
  "rollback_snapshot_ref": "s3://<audit-bucket>/.../rollback-snapshot.json",
  "dry_run_mode": false,
  "idempotency_scope": "trace_only|decide_lock",
  "safety_checks": {
    "tenant_match": "pass",
    "action_allow_list": "pass",
    "blast_radius": "pass",
    "verify_policy": "pass",
    "rollback_path": "pass",
    "duplicate_check": "pass"
  },
  "payload_hash": "sha256:<hash>",
  "redaction_applied": true
}
```

Event types nên có:

```text
alert_received
telemetry_collected
telemetry_validation_failed
detect_called
detect_response_received
decide_called
action_plan_received
rollback_snapshot_recorded
duplicate_check_passed
duplicate_request_denied
safety_passed
safety_denied
dry_run_started
dry_run_done
execute_started
execute_done
deferred_gitops_commit_created
argocd_sync_observed
verify_called
verify_done
verify_next_action_done
verify_next_action_retry
verify_next_action_rollback
verify_next_action_escalate
rollback_started
rollback_done
escalated
static_runbook_fallback_selected
incident_closed
cost_cap_exceeded_warning
circuit_breaker_opened
```

S3 key prefix đề xuất:

```text
audit/year=YYYY/month=MM/day=DD/tenant_id=<tenant_id>/correlation_id=<correlation_id>/<seq>-<event_type>.json
```

Ví dụ:

```text
audit/year=2026/month=06/day=26/tenant_id=6c8b.../correlation_id=9b1.../007-safety_passed.json
```

---

## 9. Test Case Cho Audit

Mục này không thêm thiết kế mới. Đây là các test/evidence cần có để chứng minh thiết kế audit hoạt động đúng khi reviewer kiểm tra.

| Test / Evidence | Bắt buộc? | Mục đích |
|---|---|---|
| Terraform output/config S3 Object Lock 90 ngày | Có | Chứng minh audit storage. |
| S3 object sample cho 1 incident success | Có | Chứng minh audit event thật. |
| Query theo `correlation_id` | Có | Chứng minh traceability. |
| Cross-tenant deny audit | Có | Chứng minh tenant isolation. |
| Duplicate request audit tại `/v1/decide` | Có | Chứng minh request duplicate được ghi nhận và không tạo thêm unsafe action. |
| Rollback snapshot audit | Có | Chứng minh CDO có điểm revert trước khi execute và có thể xử lý `next_action=ROLLBACK`. |
| AI timeout/503 fallback/escalation audit | Có | Chứng minh fail-safe: hoặc chọn static runbook fallback đã được thống nhất, hoặc escalate. |
| Dry-run fail -> no execute audit | Có | Chứng minh safety gate. |
| Deferred GitOps audit có commit hash/ArgoCD sync | Có nếu demo TC-05/TC-06 | Chứng minh deferred path không direct mutate K8s. |
| Redaction test | Nên có | Khớp PII/secret requirement. |
| Athena query output | Nên có | Chứng minh audit query tốt hơn inspect thủ công. |

---

## 10. Final Recommendation Cho Team

Đề xuất chốt thiết kế audit:

1. **Audit source of truth:** S3 Object Lock 90 ngày.
2. **Sandbox mode:** Governance Mode, nhưng phải ghi rõ controlled deviation so với `contract - new 3/deployment-contract.md`.
3. **Production mode:** Compliance Mode hoặc cross-account audit bucket nếu client yêu cầu strict SOC2/WORM.
4. **Runtime log:** CloudWatch Logs, query/debug theo `correlation_id`, không thay S3 audit.
5. **Query/evidence:** S3 prefix + Athena nếu kịp.
6. **Schema:** Thêm audit event schema/version trước khi chốt final.
7. **Evidence:** Gắn TC-01 -> TC-17 với audit event expected và thu evidence tối thiểu cho success, deny, timeout, duplicate tại `/v1/decide`, rollback/escalate.

Trạng thái khuyến nghị:

```text
APPROVE WITH CONDITIONS

Approved baseline:
- S3 Object Lock + CloudWatch Logs + S3/Athena query.

Conditions before final submission:
- Add audit schema.
- Add S3 key prefix convention.
- Add decision note for Governance vs Compliance Mode.
- Add audit test case/evidence checklist mapped to TC-01..TC-17.
- Add redaction rule for PII/secret.
```

---

## 11. Câu Hỏi Cần Checklist Cuối Xác Nhận

1. Checklist cuối có bắt buộc khớp `contract - new 3` về **Compliance Mode** hay cho phép Governance Mode theo trainer feedback?
2. Có bắt buộc demo audit query bằng Athena hay inspect S3 object là đủ?
3. Có bắt buộc có evidence full 4h simulation, hay chấp nhận representative scenarios?
4. Có bắt buộc có SSE-KMS key riêng cho audit bucket không?
5. Có bắt buộc payload hash/hash-chain không, hay S3 Object Lock đủ cho tamper-evident?
6. Có bắt buộc audit `rollback_snapshot` dưới dạng full object, hay chỉ cần `rollback_snapshot_ref`/hash để tránh lưu payload nhạy cảm?

Nếu 6 câu hỏi này được confirm, team có thể đóng thiết kế audit cuối cùng.
