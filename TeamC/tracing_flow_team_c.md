# Team C - Tracing Flow & Evidence Flow

**Mục đích:** mô tả tracing đi từ đâu đến đâu trong dự án hiện tại, phần nào đã có trong source, phần nào chỉ mới là contract/design, và Team C cần hỏi ai để lấy evidence.  
**Bám source:** `contract - new 4/telemetry-contract.md`, `contract - new 4/deployment-contract.md`, `docs/01_requirements_analysis.md`, `docs/02_infra_design.md`, `docs/04_deployment_design.md`, `executor/audit.py`, `executor/main.py`, `executor/scenarios/tc01_service_stuck.json`.  

## 1. Trạng Thái Tracing Hiện Tại

Tracing trong dự án hiện tại có 2 nghĩa khác nhau:

| Loại trace | Có trong source? | Ý nghĩa |
|---|---|---|
| Incident trace theo `correlation_id` | Có trong executor/audit | Trace workflow self-heal end-to-end |
| Distributed tracing theo `trace_id`/`span_id` | Có trong telemetry contract/design | Trace request liên service; runtime collector chưa xác nhận |

Team C cần phân biệt rõ:

- `correlation_id` là audit trace của incident. Cái này dùng được ngay vì `audit.py` in stdout JSON theo `correlation_id`.
- `trace_id` và `span_id` là distributed tracing trong telemetry payload. Cái này chỉ claim nếu có OTel/X-Ray/Jaeger output hiện tại hoặc scenario JSON có trace event rõ ràng.

## 2. Source Of Truth Cho Incident Trace

Trong `executor/main.py`, mỗi incident có `correlation_id`.

Flow:

```text
scenario correlation_id
  -> Executor.handle_incident()
  -> AuditLogger(correlation_id, tenant_id)
  -> audit events stdout
  -> optional S3 audit object
```

Audit events tối thiểu để trace workflow:

```text
alert_received
detect_called
detect_response_received
pre_decide_decision
idempotency_lock_acquired
decide_called
action_plan_received
safety_passed hoặc safety_denied
rollback_snapshot_captured
execute_done hoặc execute_skipped
verify_called
verify_done
rollback_done hoặc escalated hoặc incident_closed
```

Đây là tracing evidence chắc chắn nhất hiện tại.

## 3. Source Of Truth Cho Distributed Tracing

Theo telemetry contract, telemetry point có thể chứa:

```text
labels.trace_id
labels.span_id
labels.operation
```

Signal chính cho tracing:

```text
distributed_trace_error_event
```

Payload mẫu theo schema hiện tại:

```json
{
  "ts": "2026-06-29T10:00:00.123Z",
  "tenant_id": "6c8b4b2b-4d45-4209-a1b4-4b532d56a31c",
  "service": "checkout-svc",
  "signal_name": "distributed_trace_error_event",
  "value": "downstream payment-service returned HTTP 500",
  "labels": {
    "system": "E-COMMERCE",
    "namespace": "tenant-a",
    "deployment": "cdo-sample-api",
    "trace_id": "d472bd0a6bda79d8d0b2852d8165cb97",
    "span_id": "7b8f0f8a9b1c2d3e",
    "operation": "POST /checkout"
  }
}
```

Trong source hiện tại chưa thấy manifest triển khai OTel Collector/X-Ray/Jaeger cụ thể. Vì vậy distributed tracing chỉ được claim ở 2 mức:

| Mức | Điều kiện | Claim |
|---|---|---|
| Mock/offline | Scenario JSON có `distributed_trace_error_event` + `trace_id`/`span_id` | Contract/schema tracing path |
| Real runtime | Team B cung cấp OTel/X-Ray/Jaeger output hiện tại | Real distributed tracing evidence |

## 4. Tracing Flow Tổng Quát

```text
Request lỗi trong workload
  -> app/sidecar/OTel SDK tạo span lỗi
  -> OTel Collector / X-Ray / Jaeger nhận trace
  -> CDO preprocessor lấy error span
  -> chuẩn hóa thành telemetry signal `distributed_trace_error_event`
  -> telemetry_window[]
  -> executor/main.py
  -> AI /v1/detect
  -> AI anomaly_context có thể trả trigger_metric/trace context
  -> audit.py trace toàn incident bằng correlation_id
```

Trong mock/offline mode:

```text
scenario JSON có distributed_trace_error_event
  -> executor/main.py
  -> mock/real AI
  -> audit stdout theo correlation_id
```

## 5. Correlation ID vs Trace ID

| ID | Nguồn | Dùng để làm gì | Team C lấy ở đâu |
|---|---|---|---|
| `correlation_id` | scenario/executor/AI | Trace incident self-heal từ alert đến close | audit stdout, S3 audit, run report |
| `idempotency_key` | executor/AI decide flow | Chống duplicate execution | audit event decide/lock |
| `trace_id` | distributed tracing payload | Trace request lỗi qua nhiều service | `telemetry_window[].labels.trace_id`, OTel/X-Ray/Jaeger nếu có |
| `span_id` | distributed tracing payload | Xác định span lỗi cụ thể | `telemetry_window[].labels.span_id`, OTel/X-Ray/Jaeger nếu có |

Không thay thế `correlation_id` bằng `trace_id`. Trong report, Team C nên dùng:

```text
correlation_id = trace workflow self-heal
trace_id/span_id = trace giao dịch ứng dụng nếu có
```

## 6. Tracing Flow Cho Mock/Offline Mode

Team C có thể tự tạo scenario trace event:

```json
{
  "ts": "2026-06-29T10:00:00.123Z",
  "tenant_id": "6c8b4b2b-4d45-4209-a1b4-4b532d56a31c",
  "service": "checkout-svc",
  "signal_name": "distributed_trace_error_event",
  "value": "checkout span failed because payment-service returned 500",
  "labels": {
    "system": "E-COMMERCE",
    "namespace": "tenant-a",
    "deployment": "cdo-sample-api",
    "trace_id": "trace-tc03-001",
    "span_id": "span-payment-001",
    "operation": "POST /checkout"
  }
}
```

Evidence cần lưu:

```text
evidence/scenarios/tc03_trace_error.json
evidence/audit/tc03_trace_error_stdout.jsonl
evidence/run_report.json
```

Claim đúng:

```text
Team C validated distributed_trace_error_event schema and incident correlation_id audit path in mock/offline mode.
```

Không claim:

```text
OTel/X-Ray/Jaeger tracing is deployed and working.
```

nếu chưa có output runtime.

## 7. Tracing Flow Cho Real Runtime

Chỉ dùng nếu Team B xác nhận có OTel/X-Ray/Jaeger thật.

```text
Workload request
  -> OpenTelemetry SDK / auto instrumentation
  -> OTel Collector
  -> Jaeger hoặc AWS X-Ray
  -> Team C query trace_id/span_id
  -> CDO preprocessor tạo distributed_trace_error_event
  -> executor/AI/audit
```

Output cần xin Team B:

```text
1. OTel Collector có deploy thật chưa?
2. Namespace/pod/service của OTel Collector là gì?
3. Backend tracing là Jaeger hay AWS X-Ray?
4. Có trace_id mẫu từ tenant-a/tenant-b không?
5. Query/screenshot trace lỗi hiện tại ở đâu?
6. Có mapping trace_id -> service/namespace/deployment không?
```

Nếu không có output này, Team C chỉ ghi tracing là `Mock/offline contract evidence`.

## 8. Mapping Tracing -> Test Case

| Test | Trace/ID cần có | Evidence |
|---|---|---|
| TC-03 Error-rate/code fault | optional `distributed_trace_error_event`, `trace_id`, `span_id`, `operation` | scenario JSON + audit by `correlation_id` |
| TC-07 Cross-tenant | `correlation_id` đủ; không cần distributed trace | safety audit |
| TC-12 AI unavailable | `correlation_id` đủ | audit escalated |
| TC-14 Verify regression | `correlation_id`, optional trace context trong escalation bundle | verify/rollback/escalate audit |
| All scenarios | `correlation_id` bắt buộc | run report + audit chain |

## 9. Tracing Evidence Team C Cần Gom

| Evidence | Nguồn | Bắt buộc? | Ghi chú |
|---|---|---|---|
| Audit stdout theo `correlation_id` | executor/audit.py | Có | Incident trace chính |
| S3 audit object theo `correlation_id` | audit.py + S3 | P1 | Nếu bucket ready |
| Scenario JSON có `trace_id`/`span_id` | Team C | P1 | Dùng cho mock distributed trace |
| OTel Collector status | Team B | P1 | Chỉ nếu real tracing |
| Jaeger/X-Ray screenshot/query | Team B | P1 | Chỉ nếu real tracing |
| Mapping trace -> telemetry signal | Team C/B | P1 | Chứng minh `distributed_trace_error_event` |

## 10. Input/Output Cần Yêu Cầu Team A

```text
Team A cho Team C xin tracing info:
1. Executor audit hiện có đủ correlation_id qua toàn bộ flow chưa?
2. AI response có echo correlation_id ổn định không?
3. Khi escalate, escalation_bundle có include logs/metrics/trace context không?
4. Cho 1 stdout audit sample đầy đủ từ alert_received đến incident_closed/escalated.
```

## 11. Input/Output Cần Yêu Cầu Team B

```text
Team B cho Team C xin tracing runtime status:
1. OpenTelemetry Collector có deploy thật không?
2. Backend tracing là Jaeger hay AWS X-Ray?
3. Namespace/pod/service của collector/backend là gì?
4. Có trace_id mẫu từ workload tenant-a/tenant-b không?
5. Gửi screenshot hoặc query output cho một trace lỗi.
6. Nếu tracing chưa deploy, confirm để Team C chỉ ghi Mock/offline tracing evidence.
```

## 12. Input/Output Cần Yêu Cầu AI/A4

```text
AI/A4 cho Team C xin:
1. AI có sử dụng `labels.trace_id`/`labels.span_id` trong telemetry_window không?
2. Với `distributed_trace_error_event`, AI expected response là gì?
3. Có sample detect response cho trace-based error không?
4. AI có echo correlation_id từ detect -> decide -> verify không?
```

## 13. Quy Tắc Claim Tracing Evidence

- Luôn dùng `correlation_id` làm trace chính cho incident self-heal.
- Chỉ claim distributed tracing thật nếu có OTel/X-Ray/Jaeger runtime output hiện tại.
- Nếu chỉ có scenario JSON chứa `trace_id`, ghi là `Mock/offline distributed trace payload`.
- Không dùng `M6-IaC_Observability_v1.0.md` hoặc observability doc cũ để claim tracing đã deploy.
- Mọi screenshot/query trace phải link được về scenario hoặc audit bằng `correlation_id` hoặc `trace_id`.

