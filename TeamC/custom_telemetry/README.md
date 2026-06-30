# Team C2 - Custom Telemetry Contract Adapter

Mục tiêu: Team CDO/Team C có một lớp telemetry dùng chung cho cả hai cách chạy hiện tại:

1. **Mock/offline**: executor đọc file scenario JSON trong `executor/scenarios/*.json`.
2. **Real runtime**: CDO lấy dữ liệu từ Prometheus/K8s events/watcher rồi gọi AI `/v1/detect`.

Output của cả hai mode luôn là `telemetry_window[]` khớp contract mới nhất `TF3-Self-Heal-Agent-AWS/contract - new 4/telemetry-contract.md`: có đúng top-level field `ts`, `tenant_id`, `service`, `signal_name`, `value`, `labels`; `labels.system`; signal nằm trong enum contract; log string được scrub PII/secret.

## Flow Tích Hợp Đúng

Đây là ý đồ product của phần này:

```text
raw telemetry từ CDO
  -> custom_telemetry adapter
  -> telemetry_window[] đúng contract
  -> executor hoặc CDOps Telemetry Forwarder
  -> HTTP POST AI Engine /v1/detect
```

Adapter không để raw data đi thẳng sang AI. Nó validate trước các điểm bắt buộc trong contract new 4: top-level không có field ngoài schema, `tenant_id`, `ts`, `service`, `signal_name`, `value`, `labels.system`, signal enum và redaction PII/secret.

## Auto Mode - Tự Nhận Raw Rồi Chuyển

Nếu chưa biết raw thuộc loại nào, dùng mode `auto`:

```bash
cd /mnt/g/XBrain/CDO-02_capstone/P2_CD02_Duc/TeamC/custom_telemetry
python3 normalize_telemetry.py auto --input raw_telemetry.json --output /tmp/telemetry_window.json
```

Mode `auto` hiện tự nhận các shape phổ biến:

| Raw input | Nhận diện | Output signal |
|---|---|---|
| Scenario JSON có `telemetry_window` | file giả hiện tại của executor | Theo signal contract sau khi map |
| Prometheus HTTP API JSON | có `status=success`, `data.result` | Tự infer hoặc dùng metric label |
| K8s events JSON | có `items[].involvedObject` | `pod_oom_event`, `service_unhealthy` |
| Raw log event/list log | có `message`, `log`, `msg`, `@message` | `application_log_event` |
| CloudWatch-like logs | có `logEvents[]` | `application_log_event` |
| OTel span list | có `traceId/trace_id`, `spanId/span_id` | `distributed_trace_error_event` nếu span lỗi |
| OTLP JSON export | có `resourceSpans[]` | `distributed_trace_error_event` nếu span lỗi |

Ví dụ raw log:

```json
{
  "timestamp": 1782745200000,
  "service": "checkout-svc",
  "namespace": "tenant-a",
  "deployment": "cdo-sample-api",
  "level": "ERROR",
  "message": "checkout failed token=abc123456789 password=hunter2"
}
```

Kết quả chính:

```json
{
  "signal_name": "application_log_event",
  "value": "checkout failed token=[REDACTED] password=[REDACTED]",
  "labels": {
    "system": "E-COMMERCE",
    "namespace": "tenant-a",
    "deployment": "cdo-sample-api",
    "level": "ERROR",
    "cdo_source_mode": "real_log_event"
  }
}
```

Ví dụ raw trace span:

```json
[
  {
    "traceId": "abc",
    "spanId": "def",
    "serviceName": "checkout-svc",
    "name": "GET /checkout",
    "status": {"code": "ERROR", "message": "upstream 503"}
  }
]
```

Kết quả chính:

```json
{
  "signal_name": "distributed_trace_error_event",
  "value": "upstream 503",
  "labels": {
    "system": "E-COMMERCE",
    "trace_id": "abc",
    "span_id": "def",
    "operation": "GET /checkout",
    "cdo_source_mode": "real_otel_span"
  }
}
```

## Source Of Truth

- Contract mới nhất: `TF3-Self-Heal-Agent-AWS/contract - new 4/telemetry-contract.md`
- AI detect endpoint: `POST /v1/detect` theo `TF3-Self-Heal-Agent-AWS/contract - new 4/ai-api-contract.md`
- Executor input: `TF3-Self-Heal-Agent-AWS/executor/main.py`
- AI client payload: `TF3-Self-Heal-Agent-AWS/executor/ai_client.py`
- Runtime watcher hiện tại: `TF3-Self-Heal-Agent-AWS/executor/watcher.py`

## Cách Chạy Với File Giả

Flow mock đầy đủ:

```text
executor/scenarios/sc*.json
  -> custom_telemetry normalize scenario
  -> /tmp/<scenario>_contract.json
  -> executor/main.py
  -> mock_ai_server.py /v1/detect
  -> pre-decide gate + safety gate
  -> mock K8s execute
  -> mock_ai_server.py /v1/verify
  -> stdout audit + OUTCOME
```

### Terminal 1 - bật mock AI

```bash
cd /mnt/g/XBrain/CDO-02_capstone/TF3-Self-Heal-Agent-AWS/executor
python3 mock_ai_server.py
```

Kỳ vọng:

```text
mock AI on http://127.0.0.1:8080 (Ctrl-C to stop)
```

### Terminal 2 - normalize file giả theo contract

```bash
cd /mnt/g/XBrain/CDO-02_capstone/P2_CD02_Duc/TeamC/custom_telemetry
python normalize_telemetry.py scenario \
  --input ../../../TF3-Self-Heal-Agent-AWS/executor/scenarios/sc01_oom_kill_a.json \
  --output /tmp/sc01_contract.json
```

Kiểm tra nhanh telemetry đã đổi từ signal nội bộ sang contract enum:

```bash
python normalize_telemetry.py scenario \
  --input ../../../TF3-Self-Heal-Agent-AWS/executor/scenarios/sc01_oom_kill_a.json \
  --window-only
```

Kỳ vọng trong output:

```json
{
  "signal_name": "pod_oom_event",
  "labels": {
    "cdo_source_mode": "mock_scenario",
    "cdo_original_signal": "pod_waiting_reason"
  }
}
```

### Terminal 2 - chạy executor bằng file đã normalize

```bash
cd /mnt/g/XBrain/CDO-02_capstone/TF3-Self-Heal-Agent-AWS/executor
CDO_K8S_MOCK=true AI_BASE_URL=http://127.0.0.1:8080 \
  python3 main.py /tmp/sc01_contract.json
```

Kỳ vọng flow audit trên stdout có các event chính:

```text
alert_received
detect_called
detect_response
predecide
lock_acquired
decide_called
action_plan
safety_passed
snapshot_captured
execute_done
verify_called
verify_done
incident_closed

>>> OUTCOME: auto_resolved
```

Nếu thấy `OUTCOME: escalated:ai_unavailable_escalated`, nghĩa là executor không gọi được mock AI tại `127.0.0.1:8080`. Kiểm tra lại terminal 1 còn chạy không và biến `AI_BASE_URL` có đúng không.

### Chạy toàn bộ scenario hiện có qua normalizer

```bash
cd /mnt/g/XBrain/CDO-02_capstone
python3 - <<'PY'
import glob, subprocess
normalizer = "P2_CD02_Duc/TeamC/custom_telemetry/normalize_telemetry.py"
failed = []
for path in sorted(glob.glob("TF3-Self-Heal-Agent-AWS/executor/scenarios/*.json")):
    r = subprocess.run(
        ["python3", normalizer, "scenario", "--input", path, "--window-only"],
        stdout=subprocess.DEVNULL,
        stderr=subprocess.PIPE,
        text=True,
    )
    if r.returncode:
        failed.append((path, r.stderr.strip()))
print("checked", len(glob.glob("TF3-Self-Heal-Agent-AWS/executor/scenarios/*.json")))
print("all_pass" if not failed else failed)
PY
```

## Cách Chạy Với Prometheus Thật

Prometheus HTTP API trả JSON dạng `/api/v1/query`. Team C map query result sang một signal contract:

```bash
python normalize_telemetry.py prometheus \
  --input prom_query_latency.json \
  --signal-name service_latency_p95 \
  --service checkout-svc \
  --namespace tenant-a \
  --deployment cdo-sample-api \
  --output /tmp/latency_window.json
```

`/tmp/latency_window.json` là list telemetry point có thể đưa vào scenario wrapper hoặc forwarder gửi AI.

## Cách Chạy Với K8s Event Thật

```bash
kubectl get events -A -o json > /tmp/k8s_events.json

python normalize_telemetry.py k8s-events \
  --input /tmp/k8s_events.json \
  --output /tmp/k8s_event_window.json
```

Adapter chỉ nhận event có ý nghĩa self-heal như `OOMKilling`, `Unhealthy`, `BackOff`, `Failed`, `FailedScheduling`; event khác bị bỏ qua để tránh nhiễu.

## Mapping Quan Trọng

| Runtime/source signal | Contract signal | Lý do |
|---|---|---|
| `pod_waiting_reason=OOMKilled` | `pod_oom_event` | Contract không có `pod_waiting_reason` |
| `exit_code_oom=137` | `pod_oom_event` | OOM sau restart vẫn cần detect |
| `restart_count` | `container_restart_count` | Khớp enum contract |
| `pod_waiting_reason=CrashLoopBackOff` | `service_unhealthy` | Phản ánh trạng thái service cần heal |
| `readiness_fail_after_deploy` | `service_unhealthy` | Bad deploy làm service không ready |
| `container_memory_pct` | `container_resource_usage` | Memory pressure là resource signal |
| `hpa_at_max_replicas` | `service_unhealthy` | Capacity không đủ làm service degraded |
| `minor_blip` | `service_unhealthy` | Low-confidence/no-action scenario vẫn phải dùng enum contract |
| Prometheus latency query | `service_latency_p95` | Metric app/service layer |
| Prometheus error ratio query | `service_error_rate` | Metric app/service layer |

## Evidence Team C Nên Lưu

```text
evidence/custom_telemetry/
  input_mock_scenario.json
  output_contract_window.json
  input_prometheus_query.json
  output_prometheus_contract_window.json
  input_k8s_events.json
  output_k8s_event_contract_window.json
  executor_stdout_by_correlation_id.jsonl
```

Rule khi chấm: nếu claim real mode thì phải có input thật từ Prometheus/K8s kèm output đã normalize. Nếu chỉ dùng scenario JSON thì ghi mode là `mock_scenario`.
