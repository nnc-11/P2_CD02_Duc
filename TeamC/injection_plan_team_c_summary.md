# Team C - Injection Plan Summary

**Nguồn:** `TF3-Self-Heal-Agent-AWS/injectionplan.md`  
**Mục đích:** tóm tắt phần Team C cần hiểu và làm từ injection plan.

## 1. Kết Luận Quan Trọng

`injectionplan.md` xác nhận scope Team C không chỉ là viết report. Team C cần tạo và chạy bộ injection/test input:

```text
raw/live fault hoặc synthetic payload
  -> telemetry_window[] đúng schema
  -> executor/AI detect-decide-verify
  -> audit evidence theo correlation_id
  -> run_report + auto-resolve report
```

Điểm quan trọng nhất:

```text
Mọi inject, dù LIVE hay SYNTHETIC, cuối cùng đều phải quy về telemetry_window[] đúng telemetry-contract.
```

## 2. Hai Mode Inject

| Mode | Ai phụ thuộc | Ý nghĩa | Team C làm gì |
|---|---|---|---|
| LIVE | Team B có EKS/workload, Team A executor real | Tạo lỗi thật trên podinfo/EKS | Xin command/output, gom evidence real |
| SYNTHETIC | Team C tự làm được | Tạo payload telemetry trực tiếp, không cần hạ tầng thật | Viết scenario JSON/preprocessor |

Với deadline ngắn, Team C nên làm SYNTHETIC/mock trước, LIVE làm khi Team A/B ready.

## 3. Môi Trường Theo Injection Plan

| Hạng mục | Giá trị |
|---|---|
| Cluster | `cdo-eks-cluster-dev`, `us-east-1`, EKS 1.30 |
| Tenant ID | `6c8b4b2b-4d45-4209-a1b4-4b532d56a31c` |
| tenant-a workload | `deployment/cdo-sample-api`, service `checkout-svc`, namespace `tenant-a` |
| tenant-b workload | `deployment/notification-service`, namespace `tenant-b` |
| AI endpoint | `http://ai-engine.self-heal-system.svc.cluster.local:8080` |
| system label | `E-COMMERCE` |

## 4. podinfo Có Thể Inject LIVE

Theo plan, workload podinfo hỗ trợ:

| Inject | Dùng cho |
|---|---|
| `POST /readyz/disable` | service unhealthy / stuck |
| `POST /readyz/enable` | cleanup |
| `GET /status/500` | error-rate spike |
| `GET /delay/{s}` | latency spike |
| `GET /panic` | crash |
| `:9797/metrics` | Prometheus metrics |

Team C không tự assume các endpoint này đang route được. Cần Team B/A xác nhận command chạy được trong EKS.

## 5. Scenario BUILD-REAL

| Scenario | Test | Mode | Expected outcome | Team C output |
|---|---|---|---|---|
| S-01 Service stuck tenant-a | TC-01 | LIVE/mock | `RESTART_DEPLOYMENT`, auto_resolved | scenario/evidence |
| S-02 Service stuck tenant-b | TC-02 | LIVE/mock | `RESTART_DEPLOYMENT`, auto_resolved | scenario/evidence |
| S-03 Error rate spike | TC-03 | LIVE/synthetic log | restart hoặc safe escalate | scenario/evidence |
| S-04 Memory pressure/OOM | TC-04 | LIVE/synthetic | `PATCH_MEMORY_LIMIT`, auto_resolved | scenario/evidence |
| S-05 Queue backlog | TC-05 | SYNTHETIC | `SCALE_REPLICAS` deferred | synthetic scenario |
| S-06 Secret/cert expiry | TC-06 | SYNTHETIC | `ROTATE_SECRET` deferred | synthetic scenario |
| S-07 CrashLoopBackOff | optional | LIVE | `ROLLOUT_UNDO` | P1 nếu Team A ready |

## 6. Scenario SAFETY/FAILURE

| Scenario | Test | Expected outcome |
|---|---|---|
| S-08 Cross-tenant target | TC-07 | deny `denied_cross_tenant`, 0 mutation |
| S-09 Action ngoài allow-list | TC-08 | deny `denied_action_not_allowed` |
| S-10 Blast-radius vượt ngưỡng | safety | deny `blast_radius_exceeded` hoặc Kyverno chặn |
| S-11 AI timeout/503 | TC-12 | escalate, không execute |
| S-12 Duplicate idempotency | TC-11 | chỉ 1 execute, lần 2 duplicate denied |
| S-13 Confidence thấp | TC-19/20 | discard hoặc escalate trước decide |
| S-14 Tenant mismatch | TC-18 | AI 403, CDO không retry |
| S-15 Malformed telemetry | DLQ | AI 400, route DLQ nếu có |

## 7. File Team C Nên Tạo

Trong repo TF3:

```text
TF3-Self-Heal-Agent-AWS/executor/scenarios/
  preprocess_telemetry.py
  run_all.py
  tc02_service_stuck_tenant_b.json
  tc03_error_rate_log_event.json
  tc04_oom_memory_pressure.json
  tc05_queue_backlog.json
  tc06_secret_expiry.json
  tc08_cross_tenant.json
  tc09_action_not_allowed.json
  tc10_blast_radius_exceeded.json
  tc11_duplicate_idempotency.json
  tc12_ai_503.json
  tc18_tenant_mismatch.json
  tc19_low_confidence.json
  tc20_medium_conf_high_sev.json
```

Trong TeamC:

```text
P2_CD02_Duc/TeamC/
  07_test_eval_report_current.md
  evidence/
  run_report.json
```

## 8. Output Cần Có Cho Mỗi Scenario

Theo injection plan, mỗi scenario cần:

```text
1. correlation_id
2. telemetry_window[] đã gửi
3. response /v1/detect
4. response /v1/decide
5. response /v1/verify
6. outcome cuối:
   auto_resolved | denied:<reason> | escalated:<reason> | rolled_back
7. audit trail theo correlation_id
```

## 9. Điều Team C Cần Hỏi AI Team

`injectionplan.md` yêu cầu AI team confirm:

```text
1. matched_runbook name mong đợi cho mỗi pattern.
2. confidence/severity AI trả cho từng fault.
3. secret_name allow-list cho ROTATE_SECRET.
4. Với S-03 error spike: AI muốn CDO restart hay luôn escalate?
```

## 10. Điều Team C Cần Hỏi Team A/B

Team A:

```text
1. Action nào đã real, action nào vẫn stub?
2. Executor chạy scenario command nào?
3. Có hỗ trợ bad AI response/failure injection không?
4. Audit reason code thực tế là gì?
```

Team B:

```text
1. EKS/workload podinfo đã ready chưa?
2. Các endpoint /readyz/disable, /status/500, /delay/5, /panic có chạy được trong pod không?
3. Prometheus hoặc curl /metrics có output không?
4. RBAC/Kyverno/S3/DynamoDB hiện trạng thế nào?
```

