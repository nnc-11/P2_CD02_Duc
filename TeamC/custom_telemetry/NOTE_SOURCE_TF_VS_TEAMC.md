# Note - Source TF vs Team C Custom Telemetry

## 1. Source TF hiện có gì?

Repo chính:

```text
TF3-Self-Heal-Agent-AWS/
```

Phần runtime chính nằm ở:

```text
TF3-Self-Heal-Agent-AWS/executor/main.py
```

`main.py` là executor self-heal. Nó chạy vòng:

```text
telemetry_window[]
  -> AI /v1/detect
  -> pre-decide gate
  -> AI /v1/decide
  -> safety gate
  -> snapshot
  -> execute
  -> AI /v1/verify
  -> audit
```

Điểm quan trọng: `main.py` **không phải telemetry preprocessor**. Nó giả định input đã là `telemetry_window[]`.

## 2. Các file scenario JSON trong source chạy cho cái gì?

Các file này nằm ở:

```text
TF3-Self-Heal-Agent-AWS/executor/scenarios/
```

Ví dụ:

```text
sc01_oom_kill_a.json
sc02_crashloop_a.json
sc03_latency_a.json
sc04_bad_deploy_a.json
sc05_memory_pressure_a.json
...
tc01_service_stuck.json
```

Chúng dùng cho **mock/offline scenario test**:

```text
scenario JSON
  -> executor/main.py
  -> mock_ai_server.py
  -> mock K8s execute
  -> stdout audit + outcome
```

Lệnh chạy 1 scenario theo source TF:

```bash
cd /mnt/g/XBrain/CDO-02_capstone/TF3-Self-Heal-Agent-AWS/executor

python3 mock_ai_server.py
```

Terminal khác:

```bash
cd /mnt/g/XBrain/CDO-02_capstone/TF3-Self-Heal-Agent-AWS/executor

CDO_K8S_MOCK=true AI_BASE_URL=http://127.0.0.1:8080 \
  python3 main.py scenarios/sc01_oom_kill_a.json
```

Các scenario này phục vụ:

- test executor flow
- test mock AI contract path
- đo auto-resolve rate bằng `run_scenarios.py`
- tạo audit evidence theo `correlation_id`

## 3. Vì sao scenario JSON chưa đủ cho Team C?

Scenario JSON hiện tại là **JSON custom để mock chạy được**, chưa phải raw telemetry product thật.

Một số signal trong scenario không nằm trong telemetry contract enum.

Ví dụ source TF có:

```json
{
  "signal_name": "pod_waiting_reason",
  "value": "OOMKilled"
}
```

Nhưng contract chính thức không có `pod_waiting_reason`. Contract yêu cầu dùng enum như:

```text
pod_oom_event
service_unhealthy
container_restart_count
container_resource_usage
service_latency_p95
application_log_event
distributed_trace_error_event
...
```

Vì vậy nếu Team C muốn claim "telemetry đúng contract", không nên đưa raw scenario thẳng vào AI thật. Cần normalize trước.

## 4. Source TF còn thiếu gì so với phần Team C cần làm?

Source TF đã có:

| Phần | Trạng thái |
|---|---|
| Executor orchestration | Có `executor/main.py` |
| Mock AI endpoint | Có `executor/mock_ai_server.py` |
| Scenario JSON giả | Có `executor/scenarios/*.json` |
| Runner đo nhiều scenario | Có `executor/run_scenarios.py` |
| Watcher K8s real mode | Có `executor/watcher.py` |

Source TF còn thiếu hoặc chưa hoàn chỉnh cho Team C:

| Gap | Vì sao quan trọng với Team C |
|---|---|
| Chưa có telemetry preprocessor contract-first | Team C cần chuẩn hóa raw telemetry trước khi gửi AI |
| Scenario signal còn có signal nội bộ ngoài enum contract | Nếu gửi AI thật có thể bị contract validation reject |
| Chưa có adapter chung cho mock và real | Mock JSON, Prometheus, K8s events, logs, trace cần ra cùng format |
| Chưa có parser log/trace raw | Contract có `application_log_event` và `distributed_trace_error_event`, source TF chưa xử lý đầy đủ |
| Watcher real mode sinh signal như `pod_waiting_reason`, `restart_count` | Cần map về `pod_oom_event`, `container_restart_count`, `service_unhealthy` |
| Chưa có redaction tập trung cho raw log | Contract yêu cầu scrub PII/secret trước khi gửi AI |

## 5. Phần Team C custom_telemetry giải quyết gì?

Folder:

```text
P2_CD02_Duc/TeamC/custom_telemetry/
```

Mục tiêu:

```text
raw telemetry hoặc scenario JSON
  -> normalize_telemetry.py
  -> telemetry_window[] đúng contract
  -> executor/main.py hoặc forwarder
  -> AI Engine /v1/detect
```

Nó xử lý cả 2 hướng:

```text
Mock/file giả:
TF3 executor/scenarios/*.json
  -> normalize
  -> file scenario contract-ready

Real/product:
Prometheus / K8s events / logs / OTel traces
  -> normalize auto
  -> telemetry_window[] contract-ready
```

## 6. Kết luận dễ nhớ

```text
TF source = có executor + mock scenario để chạy flow self-heal.

Team C custom_telemetry = lớp chuẩn hóa telemetry trước executor/AI,
để mock và real đều khớp telemetry contract.
```

Scenario JSON trong TF không phải là "raw telemetry product thật"; nó là **test input giả** cho executor. Team C dùng nó để test adapter, nhưng khi tích hợp thật thì source input sẽ là Prometheus/K8s/log/trace.
