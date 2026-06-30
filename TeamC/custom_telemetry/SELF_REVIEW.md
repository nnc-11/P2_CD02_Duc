# Self Review - Custom Telemetry Adapter

## Kết luận

Phần này giải đúng nhu cầu Team C2: cùng một adapter tạo `telemetry_window[]` khớp contract cho cả chạy file giả và chạy thật. Không sửa executor core, nên rủi ro thấp và Team C có thể dùng ngay để tạo evidence.

## Review Theo Product Reality

| Tiêu chí | Kết quả |
|---|---|
| Khớp telemetry contract new 4 | Có validate required fields, UUID v4, signal enum, `labels.system`, và chặn top-level field ngoài schema |
| Dùng được với mock scenario | Có mode `scenario`, giữ nguyên wrapper scenario và thay `telemetry_window` |
| Dùng được với real telemetry | Có mode `prometheus` và `k8s-events` |
| Xử lý lệch hiện tại trong repo | Có map signal nội bộ `pod_waiting_reason`, `restart_count`, `exit_code_oom` về enum contract |
| PII/secret safety | Có scrub email, token/API key/secret, password, bearer token, connection string |
| Không claim quá mức | README ghi rõ mock vs real evidence phải tách mode |

## Gap Còn Lại

| Gap | Ảnh hưởng | Cách xử lý thực tế |
|---|---|---|
| Chưa gọi trực tiếp AI `/v1/detect` | Adapter mới chuẩn hóa data, chưa là forwarder hoàn chỉnh | Team C có thể bọc output vào scenario hoặc thêm SQS/HTTP worker HTTP POST theo contract new 4 sau |
| Prometheus query cần Team B cung cấp | Không thể tự claim dashboard/metric thật nếu cluster chưa ready | Lưu raw `prom_query_*.json` làm evidence |
| Watcher trong executor vẫn sinh signal ngoài enum | Nếu chạy `--watch` trực tiếp, telemetry có thể chưa contract-perfect | Dùng adapter này trước forwarder hoặc patch watcher ở bước sau |
| JSON Schema draft-07 chưa nhúng nguyên văn | Validation hiện là code-level, không phải jsonschema lib | Đủ offline không cần dependency; nếu repo cho phép dependency thì thêm `jsonschema` sau |

## Definition Of Done Cho Team C

- Chạy được `normalize_telemetry.py scenario` trên ít nhất 10 scenario.
- Mọi output chỉ có signal enum contract.
- Có ít nhất một sample redaction chứng minh secret/email bị ẩn.
- Nếu chạy real mode, lưu raw Prometheus/K8s input và normalized output trong evidence.
