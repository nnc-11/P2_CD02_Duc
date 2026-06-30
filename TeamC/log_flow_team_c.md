# Team C - Log Flow & Evidence Flow

**Mục đích:** mô tả logs đi từ đâu đến đâu, Team C lấy evidence ở đâu, và cần yêu cầu Team A/B cung cấp output nào.  
**Trạng thái:** theo repo hiện tại ngày 2026-06-29.  

## 1. Source Of Truth Hiện Tại

Nguồn log chắc chắn nhất hiện tại là audit JSON được in ra stdout từ:

```text
TF3-Self-Heal-Agent-AWS/executor/audit.py
```

Trong `AuditLogger.event()`, mỗi audit event được print ra stdout:

```python
print(json.dumps(rec, ensure_ascii=False))
```

Vì vậy mỗi lần chạy executor, Team C phải capture stdout để làm evidence.

## 2. Log Flow Tổng Quát

```text
Scenario JSON / telemetry input
  -> executor/main.py xử lý incident
  -> audit.py tạo audit event
  -> stdout JSON lines
  -> [local] terminal output
  -> [EKS] container stdout
  -> [optional] kubectl logs
  -> [optional] CloudWatch Logs nếu Team B đã cấu hình log shipping
  -> [optional] S3 audit object nếu CDO_AUDIT_BUCKET đã cấu hình
```

Không được giả định CloudWatch hoặc S3 đã sẵn. Team C chỉ claim CloudWatch/S3 khi có output xác nhận từ Team B/A4.

## 3. Các Nguồn Logs Team C Có Thể Dùng

| Nguồn | Khi dùng | Ai lấy/cung cấp | Evidence lưu lại | Ghi chú |
|---|---|---|---|---|
| Local stdout | Chạy mock/offline | Team C tự lấy | `evidence/audit/<tc>_stdout.jsonl` | Dùng được ngay |
| `kubectl logs` | Executor chạy trong EKS pod | Team C lấy nếu có kubeconfig, hoặc Team A/B cung cấp | `evidence/logs/<tc>_kubectl_logs.txt` | Cần biết namespace + pod/deploy name |
| CloudWatch Logs | Team B đã bật log shipping | Team B cung cấp query/output | `evidence/logs/<tc>_cloudwatch.txt` | Không dùng doc cũ để claim |
| S3 audit object | `CDO_AUDIT_BUCKET` đã set và bucket ready | Team A/B/A4 cung cấp bucket hoặc Team C query | `evidence/audit/<tc>_s3_audit.json` | Đây là audit trail tốt nhất nếu ready |

## 4. Flow Local / Mock

Dùng khi Team A/B/AI chưa ready.

```bash
cd TF3-Self-Heal-Agent-AWS/executor
python mock_ai_server.py
```

Ở terminal khác:

```bash
cd TF3-Self-Heal-Agent-AWS/executor
CDO_K8S_MOCK=true AI_BASE_URL=http://127.0.0.1:8080 python main.py scenarios/tc01_service_stuck.json
```

Output terminal sẽ có nhiều dòng JSON, ví dụ event:

```text
alert_received
detect_called
detect_response_received
pre_decide_decision
idempotency_lock_acquired
decide_called
action_plan_received
safety_passed
rollback_snapshot_captured
execute_done
verify_called
verify_done
incident_closed
```

Team C lưu output này thành:

```text
evidence/audit/tc01_stdout.jsonl
```

## 5. Flow EKS / Pod Logs

Khi executor chạy trong EKS, stdout của container sẽ nằm trong pod logs.

Team C cần biết:

```text
executor namespace = ?
executor deployment name = ?
executor pod name = ?
```

Command mẫu:

```bash
kubectl get pods -n self-heal-system
kubectl logs -n self-heal-system deploy/<executor-deployment-name>
```

Lọc theo `correlation_id`:

```bash
kubectl logs -n self-heal-system deploy/<executor-deployment-name> | grep "tc-01-0000-0000-0000-000000000001"
```

Nếu executor nằm ở namespace khác, đổi `self-heal-system` thành namespace thực tế do Team A/B xác nhận.

Evidence lưu:

```text
evidence/logs/tc01_kubectl_logs.txt
```

## 6. Flow CloudWatch Logs

Chỉ dùng nếu Team B xác nhận log shipping hiện tại đã hoạt động.

Team C cần yêu cầu Team B cung cấp:

```text
1. CloudWatch log group của executor là gì?
2. Log stream pattern là gì?
3. Query command theo correlation_id là gì?
4. Một output query mẫu với correlation_id thật.
```

Output mong muốn từ Team B:

```text
log_group = /aws/eks/<cluster-name>/...
query = fields @timestamp, @message
        | filter @message like /<correlation_id>/
        | sort @timestamp asc
```

Evidence lưu:

```text
evidence/logs/tc01_cloudwatch_query.txt
```

Không dùng `M6-IaC_Observability_v1.0.md` để claim CloudWatch đang hoạt động. File đó chỉ là reference cũ.

## 7. Flow S3 Audit Object

Trong `audit.py`, nếu có `CDO_AUDIT_BUCKET` và `boto3`, audit sẽ flush lên S3:

```text
s3://<bucket>/audit/<tenant_id>/<correlation_id>.json
```

Command mẫu:

```bash
aws s3 cp s3://<bucket>/audit/<tenant_id>/<correlation_id>.json - | python -m json.tool
```

Team C cần hỏi Team B/A4:

```text
1. CDO_AUDIT_BUCKET hiện tại là gì?
2. Bucket đã bật Object Lock Governance chưa?
3. Executor đã set env CDO_AUDIT_BUCKET chưa?
4. Có sample object theo correlation_id nào chưa?
```

Evidence lưu:

```text
evidence/audit/tc01_s3_audit.json
```

## 8. Event Fields Bắt Buộc

Mỗi audit event nên có các field sau, tùy loại event:

| Field | Bắt buộc? | Ghi chú |
|---|---|---|
| `timestamp` | Có | RFC3339 UTC |
| `correlation_id` | Có | Dùng để trace toàn bộ incident |
| `tenant_id` | Có | Tenant CDO-02 |
| `event` | Có | Loại audit event |
| `namespace` | Khi có target namespace | Ví dụ `tenant-a` |
| `action_type` | Khi có action | Ví dụ `RESTART_DEPLOYMENT` |
| `decision` | Khi allow/deny/escalate | Ví dụ `allow`, `deny`, `escalate` |
| `result` | Khi có kết quả | Ví dụ `success`, `failed`, `auto_resolved` |
| `reason` | Khi deny/failure/escalate | Ví dụ `denied_cross_tenant` |
| `idempotency_key` | Khi vào decide/mutate flow | Dùng cho TC-11 |

## 9. Mapping Test Case -> Logs Cần Có

| Test | Log tối thiểu cần chứng minh |
|---|---|
| TC-01 auto-resolve | `safety_passed`, `execute_done`, `verify_done`, `incident_closed result=auto_resolved` |
| TC-07 cross-tenant | `safety_denied reason=denied_cross_tenant`, không có real mutation |
| TC-08 unsupported action | `safety_denied reason=denied_action_not_allowed` |
| TC-10 missing verify policy | `safety_denied reason=missing_verify_policy` |
| TC-11 duplicate idempotency | lần 1 `idempotency_lock_acquired`, lần 2 `idempotency_duplicate_denied` |
| TC-12 AI unavailable | `escalated reason=<ai_unavailable/...>`, không có `execute_done success` |
| TC-13 dry-run fail | `dry_run_failed` hoặc `execute_done result=failed`, không có mutation thật |
| TC-14 verify regression | `verify_done regression=true`, sau đó `rollback_done` hoặc `escalated` |
| TC-19 low confidence | `pre_decide_decision decision=low_confidence_discard`, không có `decide_called` |
| TC-20 medium confidence high severity | `pre_decide_decision decision=low_confidence_escalated`, sau đó `escalated` |
| TC-21 flapping | `pre_decide_decision decision=flapping_escalated`, không gọi thêm decide |

## 10. Câu Hỏi Gửi Team A

```text
Team A cho Team C xin log flow hiện tại:
1. Executor đang chạy local hay trong EKS pod?
2. Nếu trong EKS: namespace, deployment name, pod label là gì?
3. Command lấy logs theo correlation_id là gì?
4. Audit event hiện đang ra stdout đầy đủ chưa?
5. S3 audit flush đã bật chưa hay chỉ stdout?
6. Cho 1 sample log với correlation_id thật.
```

## 11. Câu Hỏi Gửi Team B

```text
Team B cho Team C xin log/observability status hiện tại:
1. Pod stdout của executor có được ship lên CloudWatch chưa?
2. CloudWatch log group/log stream là gì?
3. Query mẫu theo correlation_id là gì?
4. Prometheus/Grafana hiện có deploy thật không? Nếu có gửi screenshot/output mới.
5. Không dùng doc M6 cũ làm evidence; cần output runtime hiện tại.
```

## 12. Quy Tắc Claim Evidence

- Nếu chỉ có local stdout: ghi mode là `Mock/offline`.
- Nếu có `kubectl logs`: ghi mode là `EKS pod logs`.
- Nếu có CloudWatch query: ghi rõ log group và query.
- Nếu có S3 audit: ghi rõ bucket/key.
- Không claim CloudWatch/S3/Prometheus/Grafana nếu chỉ có tài liệu thiết kế cũ.
- Mọi evidence phải trace được bằng cùng một `correlation_id`.

