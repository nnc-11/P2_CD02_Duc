# Team C - Metric Flow & Evidence Flow

**Mục đích:** mô tả metric đi từ đâu đến đâu trong dự án hiện tại, Team C lấy evidence ở đâu, và cần yêu cầu Team A/B cung cấp output nào.  
**Bám source:** `contract - new 4/telemetry-contract.md`, `executor/main.py`, `executor/ai_client.py`, `executor/scenarios/tc01_service_stuck.json`, `manifests/workloads/*.yaml`, `docs/01_requirements_analysis.md`, `docs/02_infra_design.md`.  
**Lưu ý:** các doc observability cũ như `M6-IaC_Observability_v1.0.md` chỉ là reference. Không claim Prometheus/Grafana/CloudWatch đang chạy nếu chưa có output runtime mới từ Team B.

## 1. Metric Source Of Truth

Theo telemetry contract hiện tại, metric không đi thẳng vào AI theo format Prometheus thô. CDO phải chuẩn hóa thành telemetry JSON rồi đưa vào:

```text
telemetry_window[]      -> AI /v1/detect
post_telemetry_window[] -> AI /v1/verify
```

Schema mỗi telemetry point:

```json
{
  "ts": "2026-06-29T10:00:00.123Z",
  "tenant_id": "6c8b4b2b-4d45-4209-a1b4-4b532d56a31c",
  "service": "checkout-svc",
  "signal_name": "service_latency_p95",
  "value": 1850.0,
  "labels": {
    "system": "E-COMMERCE",
    "namespace": "tenant-a",
    "deployment": "cdo-sample-api"
  }
}
```

Field bắt buộc theo contract:

| Field | Bắt buộc | Ghi chú |
|---|---|---|
| `ts` | Có | RFC3339 UTC, có mili-giây |
| `tenant_id` | Có | UUID tenant CDO-02 |
| `service` | Có | Service logical, ví dụ `checkout-svc` |
| `signal_name` | Có | Một trong 12 signal enum |
| `value` | Có | number hoặc string |
| `labels.system` | Có | Ví dụ `E-COMMERCE` |
| `labels.namespace` | Nên có | Ví dụ `tenant-a` |
| `labels.deployment` | Nên có | Ví dụ `cdo-sample-api` |

## 2. Metric Flow Tổng Quát

```text
Workload / Kubernetes / Queue / DB
  -> metric source: Prometheus, CloudWatch, cAdvisor, kube-state-metrics, SQS metrics
  -> CDO preprocessor / scenario injector
  -> telemetry JSON theo telemetry-contract
  -> executor/main.py nhận telemetry_window
  -> ai_client.py POST /v1/detect
  -> AI trả anomaly_context trigger_metric/trigger_value
  -> executor action
  -> executor thu hoặc nhận post_telemetry_window
  -> ai_client.py POST /v1/verify
  -> audit.py ghi evidence theo correlation_id
```

Hiện trong `executor/main.py`, `_collect_post_telemetry()` đang trả lại chính `telemetry_window` trong mock/offline mode:

```text
Offline/Mock Mode: lấy post_telemetry_window từ dataset RE2/RE3.
```

Vì vậy với Team C, cách chắc chắn nhất hiện tại là tạo scenario JSON có `telemetry_window` đúng schema, rồi lưu output audit.

## 3. Metric Sources Theo Source Repo

### 3.1 Sample workloads

Trong `manifests/workloads/tenant-a-sample-app.yaml`:

| Namespace | Deployment | Service logical | Metrics port |
|---|---|---|---|
| `tenant-a` | `cdo-sample-api` | `checkout-svc` | container `9797`, service port `9797` |

Trong `manifests/workloads/tenant-b-sample-app.yaml`:

| Namespace | Deployment | Service logical | Metrics port |
|---|---|---|---|
| `tenant-b` | `notification-service` | `notification-service` | container `9797`, service port `9797` |

Workload dùng image:

```text
ghcr.io/stefanprodan/podinfo:6.14.0
```

`manifests/workloads/README.md` cũng nói podinfo có Prometheus metrics trên port `9797`.

### 3.2 Contract metric sources

Theo telemetry contract, emit point dự kiến:

| Signal | Loại | Emit point theo contract | Dùng cho |
|---|---|---|---|
| `service_error_rate` | Metric | Prometheus / OTel | error-rate spike |
| `service_latency_p95` | Metric | Prometheus / OTel | service stuck / latency spike |
| `service_throughput_rps` | Metric | Prometheus / OTel | tải bất thường, scale decision |
| `container_resource_usage` | Metric | K8s Metrics Server / cAdvisor | memory pressure / OOM risk |
| `container_restart_count` | Metric | kube-state-metrics | CrashLoopBackOff / rollback |
| `queue_backlog` | Metric | SQS/RabbitMQ metrics collector | deferred scale |
| `db_connection_pool_saturation` | Metric | DB monitor / APM agent | DB saturation |

Event-like nhưng vẫn nằm trong telemetry payload:

| Signal | Loại | Emit point |
|---|---|---|
| `pod_oom_event` | Event | K8s node/container lifecycle event |
| `service_unhealthy` | Event/health | readiness/liveness/K8s event |
| `secret_expiry_warning` | Event | Secrets Manager / cert-manager event |

## 4. Metric Flow Cho Mock/Offline Mode

Đây là flow Team C có thể tự làm ngay.

```text
scenario JSON
  -> telemetry_window[]
  -> executor/main.py
  -> /v1/detect mock AI
  -> /v1/verify mock AI
  -> stdout audit JSON
```

Command:

```bash
cd TF3-Self-Heal-Agent-AWS/executor
python mock_ai_server.py
```

Terminal khác:

```bash
cd TF3-Self-Heal-Agent-AWS/executor
CDO_K8S_MOCK=true AI_BASE_URL=http://127.0.0.1:8080 python main.py scenarios/tc01_service_stuck.json
```

Evidence lưu:

```text
evidence/scenarios/tc01_service_stuck.json
evidence/audit/tc01_stdout.jsonl
evidence/run_report.json
```

## 5. Metric Flow Cho Real EKS/Prometheus Mode

Chỉ claim nếu Team B xác nhận Prometheus/kube-state-metrics đang chạy thật.

```text
podinfo workload port 9797
  -> Prometheus scrape
  -> query/rule tính signal
  -> CDO preprocessor/forwarder chuẩn hóa thành telemetry JSON
  -> executor telemetry_window
  -> AI /v1/detect
```

Output cần xin Team B:

```text
1. Namespace monitoring có chưa?
2. Prometheus pod/service đang Ready chưa?
3. kube-state-metrics đang Ready chưa?
4. Prometheus scrape được service tenant-a/tenant-b chưa?
5. Query mẫu cho podinfo metrics port 9797 là gì?
6. Screenshot Grafana hoặc Prometheus query result mới nhất.
```

Command evidence mẫu Team B có thể cung cấp:

```bash
kubectl get pods -n monitoring
kubectl get svc -n tenant-a cdo-sample-api
kubectl get svc -n tenant-b notification-service
kubectl port-forward -n tenant-a svc/cdo-sample-api 9898:80
curl http://127.0.0.1:9898/metrics
```

Nếu Prometheus chưa ready, Team C vẫn có thể dùng scenario JSON/offline telemetry và ghi mode là `Mock/offline`.

## 6. Metric Flow Cho Verify Step

Contract-new-4 yêu cầu `/v1/verify` có:

```text
post_telemetry_window[]
```

Trong real mode:

```text
executor execute action
  -> chờ verify_policy.window_seconds
  -> thu metric sau action
  -> tạo post_telemetry_window
  -> POST /v1/verify
```

Trong source hiện tại:

```text
executor/main.py::_collect_post_telemetry()
  -> TODO W12
  -> hiện trả lại telemetry_window trong Offline/Mock Mode
```

Team C phải ghi rõ:

| Mode | post_telemetry_window lấy từ đâu | Claim được gì |
|---|---|---|
| Mock/offline | scenario/dataset | verify path logic |
| Real EKS | Prometheus/CloudWatch sau action | verify metric hồi phục thật |

## 7. Mapping Metric -> Test Case

| Test | Metric/signal chính | Labels cần có | Evidence cần lưu |
|---|---|---|---|
| TC-01 | `service_latency_p95`, `service_unhealthy` | `namespace=tenant-a`, `deployment=cdo-sample-api` | scenario JSON + audit `auto_resolved` |
| TC-02 | `service_latency_p95`, `service_unhealthy` | `namespace=tenant-b`, `deployment=notification-service` | scenario JSON + audit |
| TC-03 | `service_error_rate`, `application_log_event` | endpoint/deployment nếu có | scenario JSON + audit |
| TC-04 | `container_resource_usage`, `pod_oom_event` | pod/container/deployment | before/after memory nếu real |
| TC-05 | `queue_backlog`, optional `service_throughput_rps` | `namespace=tenant-b`, queue/service labels | deferred scale evidence nếu ready |
| TC-11 | không phải metric chính | idempotency evidence | audit duplicate |
| TC-13 | metric không chính | dry-run/action evidence | audit dry-run fail |
| TC-14 | post-action metrics xấu hơn | same namespace/deployment | `verify_done`, rollback/escalate |

## 8. Metric Evidence Team C Cần Gom

| Evidence | Nguồn | Bắt buộc? | Ghi chú |
|---|---|---|---|
| scenario JSON có `telemetry_window` | Team C | Có | Dùng được ngay |
| stdout audit có `correlation_id` | Team C | Có | Chứng minh executor nhận metric |
| `post_telemetry_window` sample | Team C/A | Có cho verify | Mock hoặc real |
| Prometheus query result | Team B | P1 | Chỉ nếu Prometheus ready |
| Grafana screenshot | Team B/C | P1 | Không dùng doc M6 cũ |
| `curl /metrics` của podinfo | Team B/C | P1 | Chứng minh workload expose metrics |
| CloudWatch metric output | Team B | P1 | Chỉ nếu đang dùng CloudWatch |

## 9. Input/Output Cần Yêu Cầu Team A

```text
Team A cho Team C xin:
1. Executor hiện lấy `telemetry_window` từ scenario JSON hay có ingestion endpoint riêng?
2. `_collect_post_telemetry()` đã implement real metric collection chưa, hay vẫn trả lại input window?
3. Khi verify, cần Team C cung cấp `post_telemetry_window` trong scenario hay executor tự collect?
4. Output audit event nào chứng minh metric đã vào detect/verify?
5. Cho 1 run log có `detect_called`, `verify_called`, `verify_done` theo cùng correlation_id.
```

## 10. Input/Output Cần Yêu Cầu Team B

```text
Team B cho Team C xin metric runtime status:
1. Prometheus/kube-state-metrics/node-exporter/Grafana hiện có deploy thật không?
2. Namespace/service/pod name của Prometheus là gì?
3. Prometheus có scrape được `tenant-a/cdo-sample-api` port 9797 không?
4. Prometheus có scrape được `tenant-b/notification-service` port 9797 không?
5. Gửi output `kubectl get pods -n monitoring`.
6. Gửi output `curl /metrics` hoặc Prometheus query cho sample workload.
7. Nếu dùng CloudWatch metrics thay Prometheus, gửi metric namespace/query cụ thể.
8. Không dùng M6 doc cũ làm evidence; cần output runtime hiện tại.
```

## 11. Quy Tắc Claim Metric Evidence

- Nếu metric nằm trong scenario JSON: claim là `Mock/offline telemetry`.
- Nếu metric lấy từ `/metrics` podinfo: claim là `workload metrics exposed`.
- Nếu metric query từ Prometheus: claim là `Prometheus runtime evidence`.
- Nếu metric query từ CloudWatch: claim là `CloudWatch runtime evidence`.
- Không claim real metric pipeline nếu chưa có output hiện tại từ Team B.
- Mọi metric evidence phải gắn được với `correlation_id` trong audit hoặc scenario run report.

