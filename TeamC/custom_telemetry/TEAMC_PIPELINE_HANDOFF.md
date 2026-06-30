# Team C Custom Telemetry Pipeline & Handoff

## 1. Mục tiêu phần Team C đã làm

Team C làm lớp chuẩn hóa telemetry trước khi dữ liệu đi vào executor hoặc AI Engine.

Pipeline:

```text
raw telemetry input
  -> TeamC custom_telemetry
  -> telemetry_window[] đúng telemetry contract new 4
  -> executor/main.py hoặc telemetry forwarder
  -> AI Engine /v1/detect
```

Mục tiêu không phải thay `executor/main.py`. Mục tiêu là đảm bảo mọi nguồn telemetry, dù mock hay real, đều ra cùng format contract mới nhất tại `TF3-Self-Heal-Agent-AWS/contract - new 4/telemetry-contract.md`.

## 2. Input cần nhận

Team C adapter nhận được các nhóm input sau.

### 2.1 Mock scenario JSON

Nguồn:

```text
TF3-Self-Heal-Agent-AWS/executor/scenarios/*.json
```

Shape:

```json
{
  "scenario": "oom_kill",
  "tenant_namespace": "tenant-a",
  "correlation_id": "sc01-...",
  "telemetry_window": [
    {
      "ts": "2026-06-29T10:00:00.000Z",
      "tenant_id": "6c8b4b2b-4d45-4209-a1b4-4b532d56a31c",
      "service": "checkout-svc",
      "signal_name": "pod_waiting_reason",
      "value": "OOMKilled",
      "labels": {
        "system": "K8S_NATIVE",
        "namespace": "tenant-a",
        "deployment": "cdo-sample-api"
      }
    }
  ]
}
```

Ghi chú: đây là input giả để test executor, chưa chắc signal đã chuẩn contract.

### 2.2 Prometheus metric raw

Nguồn dự kiến: Team B/Platform hoặc Prometheus HTTP API.

Yêu cầu tối thiểu:

```text
status=success
data.result[]
metric labels có namespace/deployment/service nếu có
value hoặc values
```

Nếu không có label đầy đủ, cần truyền thêm:

```text
service
namespace
deployment
signal_name
```

### 2.3 K8s events raw

Nguồn dự kiến:

```bash
kubectl get events -A -o json
```

Adapter xử lý event như:

```text
OOMKilling
Unhealthy
BackOff
Failed
FailedScheduling
```

### 2.4 Raw logs

Nguồn dự kiến: Fluent Bit, CloudWatch Logs, app log export.

Yêu cầu tối thiểu:

```text
message/log/msg/@message
timestamp nếu có
service hoặc deployment nếu có
namespace nếu có
level/severity nếu có
```

Adapter sẽ scrub:

```text
email
token/api_key/secret
password
Bearer token
connection string
```

### 2.5 Raw traces / OTel spans

Nguồn dự kiến: OTel Collector export.

Yêu cầu tối thiểu:

```text
traceId hoặc trace_id
spanId hoặc span_id
name/operation
status code ERROR
serviceName hoặc service nếu có
```

Chỉ span lỗi mới được chuyển thành telemetry point.

## 3. Output của Team C

Output chuẩn là:

```text
telemetry_window[]
```

Mỗi point phải khớp telemetry contract:

```json
{
  "ts": "2026-06-29T10:00:00.000Z",
  "tenant_id": "6c8b4b2b-4d45-4209-a1b4-4b532d56a31c",
  "service": "checkout-svc",
  "signal_name": "pod_oom_event",
  "value": "OOMKilled",
  "labels": {
    "system": "K8S_NATIVE",
    "namespace": "tenant-a",
    "deployment": "cdo-sample-api",
    "cdo_source_mode": "mock_scenario",
    "cdo_original_signal": "pod_waiting_reason"
  }
}
```

Các rule bắt buộc:

| Field | Rule |
|---|---|
| top-level fields | Chỉ được có `ts`, `tenant_id`, `service`, `signal_name`, `value`, `labels` |
| `ts` | RFC3339 UTC, millisecond precision |
| `tenant_id` | UUID v4 |
| `service` | Không được rỗng |
| `signal_name` | Phải thuộc 12 enum trong telemetry contract |
| `value` | number hoặc string |
| `labels.system` | Bắt buộc |
| log value | Không còn PII/secret rõ ràng |

## 4. Mapping chính

| Input raw/internal | Output contract |
|---|---|
| `pod_waiting_reason=OOMKilled` | `pod_oom_event` |
| `exit_code_oom=137` | `pod_oom_event` |
| `restart_count` | `container_restart_count` |
| `container_memory_pct` | `container_resource_usage` |
| `readiness_fail_after_deploy` | `service_unhealthy` |
| `hpa_at_max_replicas` | `service_unhealthy` |
| raw log message | `application_log_event` |
| OTel span ERROR | `distributed_trace_error_event` |
| K8s `Unhealthy` / `BackOff` | `service_unhealthy` |

## 5. Cách chạy

Auto detect raw input:

```bash
cd /mnt/g/XBrain/CDO-02_capstone/P2_CD02_Duc/TeamC/custom_telemetry

python3 normalize_telemetry.py auto \
  --input raw_telemetry.json \
  --output /tmp/telemetry_window.json
```

Normalize scenario giả:

```bash
python3 normalize_telemetry.py scenario \
  --input ../../../TF3-Self-Heal-Agent-AWS/executor/scenarios/sc01_oom_kill_a.json \
  --output /tmp/sc01_contract.json
```

Chạy qua executor:

```bash
cd /mnt/g/XBrain/CDO-02_capstone/TF3-Self-Heal-Agent-AWS/executor

CDO_K8S_MOCK=true AI_BASE_URL=http://127.0.0.1:8080 \
  python3 main.py /tmp/sc01_contract.json
```

## 6. Bàn giao cho bên nào?

### Bàn giao cho Team A / Executor

Team A cần nhận:

```text
telemetry_window[] đúng contract
```

Mục đích:

```text
main.py dùng telemetry_window[] để gọi AI /v1/detect
```

Team A không cần biết raw input ban đầu là Prometheus, log hay scenario.

### Bàn giao cho Team B / Platform

Team B cần biết input raw nào nên xuất ra:

```text
Prometheus query JSON
kubectl events JSON
CloudWatch/Fluent Bit log JSON
OTel span export
```

Team B cũng cần cung cấp metadata đủ:

```text
namespace
deployment
service
pod/container nếu có
timestamp
```

### Bàn giao cho AI team

AI team nhận gián tiếp qua endpoint:

```text
POST /v1/detect
```

Payload chứa:

```text
idempotency_key
dry_run_mode
telemetry_window[]
```

Team C cần đảm bảo trước khi gửi AI:

```text
signal_name thuộc enum contract
tenant_id đúng UUID
không còn PII/secret trong log
labels đủ system/namespace/deployment nếu có
```

### Bàn giao cho QA/Evidence

QA cần lưu:

```text
raw input sample
normalized output sample
executor stdout/audit
correlation_id
mode: mock_scenario / real_prometheus / real_k8s_event / real_log_event / real_otel_span
```

## 7. Nghiệm thu

### Nghiệm thu tối thiểu

| Check | Pass criteria |
|---|---|
| Scenario JSON hiện có | 100% normalize pass |
| Contract enum | 100% output `signal_name` thuộc 12 enum |
| Required fields | 100% point có `ts`, `tenant_id`, `service`, `signal_name`, `value` |
| Additional properties | 100% point không có top-level field ngoài schema new 4 |
| `labels.system` | 100% có |
| PII/secret scrub | sample log không còn email/token/password/connection string |
| Executor compatibility | normalized scenario chạy được với `main.py` mock mode |

### Nghiệm thu tích hợp thật

| Check | Pass criteria |
|---|---|
| Prometheus input thật | convert được thành metric signal contract |
| K8s event thật | convert được OOM/unhealthy/backoff event |
| Log thật | convert được `application_log_event` và scrub secret |
| Trace thật | convert được span lỗi thành `distributed_trace_error_event` |
| AI `/v1/detect` | nhận payload không trả `400 Bad Request` |
| Audit | có `correlation_id` xuyên suốt từ input đến outcome |

## 8. Definition of Done

Phần Team C custom telemetry được coi là xong khi:

```text
1. Có README hướng dẫn mock và real flow.
2. Có note phân biệt source TF executor với Team C adapter.
3. Có code normalize scenario/Prometheus/K8s/log/trace.
4. Có validation contract và redaction.
5. Chạy pass toàn bộ scenario JSON hiện có.
6. Có sample output để Team A/AI/QA dùng nghiệm thu.
```
