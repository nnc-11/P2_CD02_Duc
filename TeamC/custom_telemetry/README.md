# Team C - Custom Telemetry

Scope của Team C hiện tại: chỉ làm lớp `custom_telemetry`.

Team C không sửa contract, không deploy AI Engine, không sửa core executor. Phần này nhận raw telemetry từ mock scenario hoặc runtime observability, chuẩn hóa thành `telemetry_window[]` đúng contract mới nhất:

```text
TF3-Self-Heal-Agent-AWS/contract - new 4/telemetry-contract.md
```

Output sau normalize có thể đưa cho:

- `TF3-Self-Heal-Agent-AWS/executor/main.py` trong mock/offline flow.
- CDOps Telemetry Forwarder để HTTP POST sang AI Engine `/v1/detect`.
- QA/evidence để chứng minh telemetry hợp lệ theo contract.

## Flow

```text
raw telemetry
  -> normalize_telemetry.py
  -> telemetry_window[] contract new 4
  -> validate
  -> executor/main.py hoặc telemetry forwarder
```

Adapter luôn enforce các rule chính:

- Top-level field chỉ gồm `ts`, `tenant_id`, `service`, `signal_name`, `value`, `labels`.
- `tenant_id` là UUID v4.
- `ts` là RFC3339 UTC millisecond.
- `signal_name` thuộc 12 enum trong contract new 4.
- `labels.system` luôn có.
- Log value được scrub email, token, API key, password, Bearer token, connection string.

## File chính

| File | Vai trò |
|---|---|
| `telemetry_contract.py` | Logic normalize, map signal, validate, scrub secret |
| `normalize_telemetry.py` | CLI cho Team C dùng với raw input |
| `DEPLOY_HANDOFF.md` | Bàn giao cho team deploy/platform |

## Raw input hỗ trợ

| Input | CLI mode | Output signal |
|---|---|---|
| Scenario JSON có `telemetry_window` | `scenario` hoặc `auto` | Map signal nội bộ sang enum contract |
| Prometheus HTTP API JSON | `prometheus` hoặc `auto` | Metric signal theo tham số hoặc infer |
| `kubectl get events -A -o json` | `k8s-events` hoặc `auto` | `pod_oom_event`, `service_unhealthy` |
| Raw log / CloudWatch `logEvents[]` | `logs` hoặc `auto` | `application_log_event` |
| OTel span list | `otel-spans` hoặc `auto` | `distributed_trace_error_event` cho span lỗi |
| OTLP JSON export `resourceSpans[]` | `otlp-export` hoặc `auto` | `distributed_trace_error_event` cho span lỗi |
| Telemetry đã normalize | `validate` | Kiểm tra contract |

## Chạy mock scenario hiện có

```bash
cd /mnt/g/XBrain/CDO-02_capstone/P2_CD02_Duc/TeamC/custom_telemetry

python3 normalize_telemetry.py scenario \
  --input ../../../TF3-Self-Heal-Agent-AWS/executor/scenarios/sc01_oom_kill_a.json \
  --output /tmp/sc01_contract.json

python3 normalize_telemetry.py validate --input /tmp/sc01_contract.json
```

Chạy executor bằng file đã normalize:

```bash
cd /mnt/g/XBrain/CDO-02_capstone/TF3-Self-Heal-Agent-AWS/executor

CDO_K8S_MOCK=true AI_BASE_URL=http://127.0.0.1:8080 \
  python3 main.py /tmp/sc01_contract.json
```

Mock AI cần chạy ở terminal khác nếu muốn test full flow:

```bash
cd /mnt/g/XBrain/CDO-02_capstone/TF3-Self-Heal-Agent-AWS/executor
python3 mock_ai_server.py
```

## Chạy với raw telemetry thật

Prometheus query result:

```bash
python3 normalize_telemetry.py prometheus \
  --input prom_query_latency.json \
  --signal-name service_latency_p95 \
  --service checkout-svc \
  --namespace tenant-a \
  --deployment cdo-sample-api \
  --output /tmp/latency_window.json
```

K8s events:

```bash
kubectl get events -A -o json > /tmp/k8s_events.json

python3 normalize_telemetry.py k8s-events \
  --input /tmp/k8s_events.json \
  --output /tmp/k8s_event_window.json
```

Raw logs:

```bash
python3 normalize_telemetry.py logs \
  --input raw_logs.json \
  --output /tmp/log_window.json
```

OTel span list:

```bash
python3 normalize_telemetry.py otel-spans \
  --input spans.json \
  --output /tmp/trace_window.json
```

OTLP export:

```bash
python3 normalize_telemetry.py otlp-export \
  --input otlp_export.json \
  --output /tmp/otlp_trace_window.json
```

Auto-detect khi chưa biết raw shape:

```bash
python3 normalize_telemetry.py auto \
  --input raw_telemetry.json \
  --output /tmp/telemetry_window.json
```

## Validate output trước khi bàn giao

```bash
python3 normalize_telemetry.py validate --input /tmp/telemetry_window.json
```

Output kỳ vọng:

```json
{
  "valid": true,
  "points": 1
}
```

Nếu input là scenario wrapper có field `telemetry_window`, lệnh `validate` cũng đọc được:

```bash
python3 normalize_telemetry.py validate --input /tmp/sc01_contract.json
```

## Mapping chính

| Raw/internal signal | Contract signal |
|---|---|
| `pod_waiting_reason=OOMKilled` | `pod_oom_event` |
| `exit_code_oom=137` | `pod_oom_event` |
| `restart_count` | `container_restart_count` |
| `container_memory_pct` | `container_resource_usage` |
| `readiness_fail_after_deploy` | `service_unhealthy` |
| `hpa_at_max_replicas` | `service_unhealthy` |
| `minor_blip` | `service_unhealthy` |
| raw log message | `application_log_event` |
| OTel span ERROR | `distributed_trace_error_event` |
| K8s `OOMKilling` | `pod_oom_event` |
| K8s `Unhealthy` / `BackOff` / `Failed` | `service_unhealthy` |

## Check toàn bộ scenario TF3

```bash
cd /mnt/g/XBrain/CDO-02_capstone

python3 - <<'PY'
import glob, subprocess

normalizer = "P2_CD02_Duc/TeamC/custom_telemetry/normalize_telemetry.py"
failed = []
paths = sorted(glob.glob("TF3-Self-Heal-Agent-AWS/executor/scenarios/*.json"))

for path in paths:
    result = subprocess.run(
        ["python3", normalizer, "scenario", "--input", path, "--window-only"],
        stdout=subprocess.DEVNULL,
        stderr=subprocess.PIPE,
        text=True,
    )
    if result.returncode:
        failed.append((path, result.stderr.strip()))

print("checked", len(paths))
print("all_pass" if not failed else failed)
raise SystemExit(1 if failed else 0)
PY
```

Kết quả hiện tại: `checked 15`, `all_pass`.

## Quality Quick Test

Chạy nhanh trước khi bàn giao hoặc trước khi deploy:

```bash
cd /mnt/g/XBrain/CDO-02_capstone

PYTHONPYCACHEPREFIX=/tmp/cdo_pycache \
  python3 -m py_compile \
  P2_CD02_Duc/TeamC/custom_telemetry/telemetry_contract.py \
  P2_CD02_Duc/TeamC/custom_telemetry/normalize_telemetry.py

PYTHONPYCACHEPREFIX=/tmp/cdo_pycache python3 - <<'PY'
import glob, os, subprocess

normalizer = "P2_CD02_Duc/TeamC/custom_telemetry/normalize_telemetry.py"
paths = sorted(glob.glob("TF3-Self-Heal-Agent-AWS/executor/scenarios/*.json"))
failed = []
env = {**os.environ, "PYTHONPYCACHEPREFIX": "/tmp/cdo_pycache"}

for path in paths:
    result = subprocess.run(
        ["python3", normalizer, "scenario", "--input", path, "--window-only"],
        stdout=subprocess.DEVNULL,
        stderr=subprocess.PIPE,
        text=True,
        env=env,
    )
    if result.returncode:
        failed.append((path, result.stderr.strip()))

print("checked", len(paths))
print("all_pass" if not failed else failed)
raise SystemExit(1 if failed else 0)
PY

PYTHONPYCACHEPREFIX=/tmp/cdo_pycache \
  python3 P2_CD02_Duc/TeamC/custom_telemetry/normalize_telemetry.py scenario \
  --input TF3-Self-Heal-Agent-AWS/executor/scenarios/sc01_oom_kill_a.json \
  --output /tmp/sc01_contract.json

PYTHONPYCACHEPREFIX=/tmp/cdo_pycache \
  python3 P2_CD02_Duc/TeamC/custom_telemetry/normalize_telemetry.py validate \
  --input /tmp/sc01_contract.json
```

Expected:

```text
checked 15
all_pass
{
  "valid": true,
  "points": 1
}
```

Test scrub log:

```bash
printf '%s\n' '[{"timestamp":1782745200000,"service":"checkout-svc","namespace":"tenant-a","deployment":"cdo-sample-api","level":"ERROR","message":"checkout failed token=abc123456789 password: hunter2 user=a@example.com"}]' > /tmp/raw_logs.json

PYTHONPYCACHEPREFIX=/tmp/cdo_pycache \
  python3 P2_CD02_Duc/TeamC/custom_telemetry/normalize_telemetry.py logs \
  --input /tmp/raw_logs.json \
  --output /tmp/log_window.json

PYTHONPYCACHEPREFIX=/tmp/cdo_pycache \
  python3 P2_CD02_Duc/TeamC/custom_telemetry/normalize_telemetry.py validate \
  --input /tmp/log_window.json

grep -n "REDACTED\|EMAIL" /tmp/log_window.json
```

Expected có dạng:

```text
"value": "checkout failed token=[REDACTED] password=[REDACTED] user=[EMAIL_REDACTED]"
```

Test full mock executor flow:

Terminal 1:

```bash
cd /mnt/g/XBrain/CDO-02_capstone/TF3-Self-Heal-Agent-AWS/executor
python3 mock_ai_server.py
```

Terminal 2:

```bash
cd /mnt/g/XBrain/CDO-02_capstone/TF3-Self-Heal-Agent-AWS/executor

CDO_K8S_MOCK=true AI_BASE_URL=http://127.0.0.1:8080 \
  python3 main.py /tmp/sc01_contract.json
```

Expected có các event `detect_called`, `detect_response`, `decide_called`, `verify_called`, `verify_done` và kết thúc:

```text
>>> OUTCOME: auto_resolved
```

## Evidence nên lưu

```text
evidence/custom_telemetry/
  raw_input_<source>.json
  telemetry_window_<source>.json
  validate_<source>.json
  executor_stdout_<scenario>.jsonl
```

Khi report, ghi rõ mode:

- `mock_scenario`: dùng file scenario của executor.
- `real_prometheus`: lấy từ Prometheus HTTP API.
- `real_k8s_event`: lấy từ Kubernetes events.
- `real_log_event`: lấy từ log collector/CloudWatch/Fluent Bit.
- `real_otel_span`: lấy từ OTel/OTLP trace.
