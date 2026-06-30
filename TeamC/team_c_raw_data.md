# TEAM C RAW DATA - CDO-02

File này gom dữ liệu raw liên quan đến Team C từ các tài liệu dự án. Mục đích là để đọc scope gốc trước khi tách thành checklist hoặc phân công.

Lưu ý quan trọng:

- `Role.md` và `W12_TASKS.md` là nguồn gần với scope hiện tại nhất cho Team C.
- `docs/07_test_eval_report_v1.0_Duc.md` được tạo từ giai đoạn sớm khi dự án chưa bắt đầu, nên chỉ dùng như khung tham khảo/legacy baseline. Không coi các chi tiết trong file đó là kế hoạch hiện hành nếu mâu thuẫn với thực tế build hiện tại.
- Khi làm việc thực tế, Team C nên lấy output mới từ Team A/B/AI rồi cập nhật lại test report bằng số liệu thật.

Nguồn:

- `TF3-Self-Heal-Agent-AWS/Role.md`
- `TF3-Self-Heal-Agent-AWS/W12_TASKS.md`
- `TF3-Self-Heal-Agent-AWS/docs/07_test_eval_report_v1.0_Duc.md` (legacy/reference only)
- `TF3-Self-Heal-Agent-AWS/capstone-phase2/reference/CAPSTONE_EVIDENCE_PACK_FORMAT.md`
- `TF3-Self-Heal-Agent-AWS/capstone-phase2/reference/TF3_SELFHEAL_LEARNER.md`

---

## 1. Role.md - Dòng phân team và lý do Team C tồn tại

```text
Chia theo critical path của demo, không chia theo đầu người cho đều:

- Demo sống hay chết nằm ở Executor (vòng detect→decide→safety→execute→verify→audit). Hiện chưa có code chạy thật (mới có skeleton). → đội mạnh nhất + Lead vào đây.
- Infra phải có trước thì Core và QA mới chạy được trên EKS thật. → đội Platform mở đường.
- Bằng chứng mới ra điểm (auto-resolve rate ≥60%, ≥10 scenario/4h, zero unsafe, audit query). → đội Telemetry/QA biến code thành evidence.

| Subteam | Tên gọi | Người | Vai trò cốt lõi | Thư mục sở hữu |
|---|---|:---:|---|---|
| A | Self-Heal Core (Executor & Safety) ⭐ | 4 | Trái tim runtime: vòng tự chữa lành + safety | `executor/` |
| B | Platform & Infrastructure | 3 | AWS base, EKS, GitOps, security baseline | `infra/`, `manifests/` |
| C | Telemetry, QA & Evidence | 3 | Telemetry pipeline, test/scenario, evidence, slides | `docs/`, `evidence/`, scripts test |
```

---

## 2. Role.md - SUBTEAM C raw scope

```text
## 4. SUBTEAM C — TELEMETRY, QA & EVIDENCE (3 người)

Sứ mệnh: Biến executor chạy được thành bằng chứng chấm điểm. Không có evidence = không có điểm.

### C1 — «tên» — QA Lead (Test & Eval)
- Sở hữu: `docs/07_test_eval_report_v1.0_Duc.md`, scenario simulation runner.
- Chạy 21 test case (TC-01..21), ≥10 scenario / ≥4h window → auto-resolve rate ≥60%.
- Multi-tenant isolation test (cross-tenant deny = SEV1 gate), security test, load test (k6 — defer được).
- Điền SLO measured, failure analysis (không điền số giả).

### C2 — «tên» — Telemetry Pipeline & Observability
- Sở hữu: RE2/RE3 preprocessor, inject scripts (`queue_backlog`, OOM...), SQS forwarder, Grafana/Prometheus dashboard.
- Chuẩn hóa 12 signal theo telemetry contract, scrub PII trước khi gửi AI, gắn tenant_id/correlation_id/ts.
- Cung cấp `telemetry_window` + `post_telemetry_window` cho Core/QA (Offline/Mock Mode).

### C3 — «tên» — Evidence, Docs & Demo
- Sở hữu: `docs/` (sync contract-new-4), `docs/08_adrs.md`, `evidence/`, SLIDES, demo video.
- Giữ docs khớp contract khi có thay đổi (header + I/O + SLA + error table trong cùng PR).
- Audit query (Athena/inspect) theo `correlation_id`; gói evidence pack; curveball-responses; individual-pitches.
```

---

## 3. Role.md - Thư mục Team C sở hữu

```text
## 5. Ma trận sở hữu thư mục

/executor/     → Subteam A   (review: A1 Lead)
/infra/        → Subteam B   (review: B1)
/manifests/    → Subteam B   (review: B3)
/docs/         → Subteam C   (review: C3)
/evidence/     → Subteam C
/contract*/    → A1 Lead only (FROZEN — đổi qua RFC)
/WORK_RULE.md, /Role.md → A1 Lead
```

---

## 4. Role.md - Handoff liên team liên quan Team C

```text
## 6. Phụ thuộc & handoff (critical path)

B1 (EKS + IRSA) ──┬─► A3/A4 chạy executor thật trên cluster
B2 (S3/DDB/SQS) ──┘
B3 (ArgoCD/Kyverno/NetworkPolicy) ──► A (deferred path + 3-lớp safety)
AI team (image) ──► A4 + B3 deploy AI Engine (wave 3)   ← rủi ro #1, theo dõi sát
A (executor chạy) ──► C1 (chạy scenario) ──► C3 (evidence/slides)
C2 (telemetry/inject) ──► A (có data detect/verify) + C1 (chạy test)

Quy tắc: B không block A — nếu EKS chưa sẵn, A dùng `CDO_K8S_MOCK=true` + `mock_ai_server.py` phát triển song song.
```

---

## 5. Role.md - Lịch W12 raw cho Team C

```text
## 7. Lịch 5 ngày W12 (29/06 → 02/07, freeze 8h T5)

| Ngày | Subteam A (Core) | Subteam B (Platform) | Subteam C (Telemetry/QA) |
|---|---|---|---|
| T2 29/06 | A3 implement k8s_client restart+patch; A2 hoàn thiện gate; A1 chốt SA namespace | B1 bật EKS+IRSA; B2 apply audit infra; B3 ArgoCD+Kyverno | C2 preprocessor + inject script; C3 dựng khung evidence |
| T3 30/06 ⭐ integration | A4 audit→S3 + idempotency thật; chạy TC-01..04 trên EKS thật | B3 deferred path (hoặc báo hạ scope); B1 hỗ trợ debug | C1 bắt đầu chạy scenario; C2 dashboard |
| T4 01/07 | A xử curveball #3; vá bug; (nếu kịp) deferred TC-05/06 | B đóng gap infra, cost measured | C1 chạy đủ ≥10 scenario/4h; C3 slides draft + audit query |
| T5 02/07 sáng | 🛑 8h freeze. A1 review lần cuối, git tag `final` | B verify smoke test, đóng public endpoint | C3 demo video + curveball-responses + individual-pitches |
```

---

## 6. W12_TASKS.md - Bối cảnh Team C

```text
# 🟨 SUBTEAM C — TELEMETRY, QA & EVIDENCE

Bối cảnh: Chưa có script telemetry/inject/test. Việc tuần này là tạo dữ liệu để chạy, chạy ≥10 scenario, gom bằng chứng + slides. Phụ thuộc executor (A) chạy được — trong lúc chờ, dùng `mock_ai_server.py` + `CDO_K8S_MOCK=true` để dựng script trước.
```

---

## 7. W12_TASKS.md - C-01 đến C-16 raw task list

```text
### C-01 — Preprocessor RE2/RE3 (CSV→telemetry JSON) · MUST · C2 · T2
🎯 Biến dataset CSV thành telemetry đúng schema (12 signal), scrub PII.
🔧 Tạo `executor/scenarios/preprocess.py`:
   import csv, json, uuid, sys
   TENANT = "6c8b4b2b-4d45-4209-a1b4-4b532d56a31c"
   def row_to_signal(r):
       return {"ts": r["ts"], "tenant_id": TENANT, "service": r["service"],
               "signal_name": r["signal_name"], "value": _num(r["value"]),
               "labels": {"system": "E-COMMERCE", "namespace": r["namespace"],
                          "deployment": r["deployment"]}}
   def _num(v):
       try: return float(v)
       except: return v
   # đọc metrics.csv → list signal → ghi telemetry_window JSON

2. Scrub PII: regex xóa email/token/connection-string khỏi `application_log_event.value`.
✅ Kiểm tra: output validate được bằng schema telemetry-contract; không còn PII.
📎 Evidence: 1 file telemetry JSON mẫu.

### C-02 — Inject scripts cho từng pattern · MUST · C2 · T2–T3
🎯 Script bắn telemetry để trigger từng pattern. Đã có mẫu `scenarios/tc01_service_stuck.json`.
🔧 Tạo thêm các scenario JSON (copy mẫu, đổi `signal_name`/`value`/`tenant_namespace`):
   - `tc04_oom.json`: `container_resource_usage` value cao + `pod_oom_event`.
   - `tc05_queue.json`: `queue_backlog` value 15000, tenant-b.
   - `tc06_secret.json`: `secret_expiry_warning` value 7.
   - `tc07_cross_tenant.json`: incident tenant-a (để test deny — dùng mock AI trả target tenant-b).
2. Cách chạy 1 scenario: `python main.py scenarios/<file>.json`.
✅ Kiểm tra: mỗi script chạy ra đúng pattern_type/action mong đợi.
📎 Evidence: log mỗi scenario.

### C-03 — SQS forwarder/worker · SHOULD · C2 · T3
🎯 Đẩy telemetry qua SQS buffer rồi forward sang `/v1/detect` (chứng minh backpressure ≤100 RPS).
🔧 Viết worker: đọc SQS (queue từ B-03) → batch → POST `/v1/detect`. Dùng boto3 `receive_message`.
✅ Kiểm tra: bắn 200 message → worker giữ nhịp ≤100 RPS, không drop.
📎 Evidence: log throughput.

### C-04 — Dashboard Grafana/Prometheus · SHOULD · C2 · T3
🔧 Cài kube-prometheus-stack qua Helm; import dashboard latency/error/memory/restart cho tenant-a/b.
✅ Kiểm tra: dashboard hiện metric thật từ podinfo.
📎 Evidence: screenshot dashboard.

### C-05 — Scenario simulation runner (≥10 scenario, ≥4h) · MUST · C1 · T3–T4
🎯 Chạy đủ ≥10 scenario trong cửa sổ ≥4h → tính auto-resolve rate.
🔧 Tạo `executor/scenarios/run_all.py`:
   import glob, subprocess, time, json
   results = []
   for f in sorted(glob.glob("scenarios/tc*.json")):
       out = subprocess.run(["python","main.py",f], capture_output=True, text=True)
       outcome = out.stdout.strip().splitlines()[-1]
       results.append({"scenario": f, "outcome": outcome,
                       "ts": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime())})
       time.sleep(5)   # giãn cho đủ window; lặp lại set để đạt 4h
   json.dump(results, open("scenarios/run_report.json","w"), indent=2)
2. Để đạt 4h: lặp set scenario nhiều vòng (cron/loop), ghi timestamp đầu–cuối.
✅ Kiểm tra: `run_report.json` có ≥10 dòng, timestamp cách nhau ≥4h.
📎 Evidence: run_report.json.

### C-06 — Chạy TC-01..06 + auto-resolve rate · MUST · C1 · T3–T4
🎯 Đo `auto_resolved_count / total ≥ 60%`.
🔧 Từ `run_report.json`, đếm outcome `auto_resolved`. Điền bảng "Tóm tắt kết quả" trong `docs/07_test_eval_report_v1.0_Duc.md`.
✅ Kiểm tra: rate ≥60%, có bảng số thật.
📎 Evidence: bảng auto-resolve + run_report.

### C-07 — Chạy TC-07..21 (safety/failure) · MUST · C1 · T4
🎯 Chứng minh deny đúng từng case (cross-tenant, action lạ, thiếu verify_policy, AI 503...).
🔧 Mỗi TC: dùng mock AI trả response "xấu" tương ứng (sửa `mock_ai_server.py` trả target sai/action lạ), chạy, kiểm audit reason.
✅ Kiểm tra: mỗi TC ra đúng audit reason (vd `denied_cross_tenant`), 0 mutation sai.
📎 Evidence: log audit từng TC.

### C-08 — Multi-tenant isolation suite · MUST · C1 · T3
🎯 Cross-tenant = SEV1. Chứng minh deny ở mọi lớp.
🔧 Chạy: (1) safety gate test cross-tenant (đã có trong `test_safety_gate.py`); (2) RBAC `kubectl auth can-i` (B-11); (3) ArgoCD AppProject chặn sync sai namespace.
✅ Kiểm tra: 100% cross-tenant bị deny, 0 mutation thành công.
📎 Evidence: output 3 lớp.

### C-09 — Load test k6 · CUT · C1 · T4
🔧 k6 script bắn 100 events/phút trong 10 phút vào ingestion; ghi p99, error rate. Nếu trễ → để TBD.
📎 Evidence: k6 summary.

### C-10 — Audit query theo correlation_id · MUST · C3 · T4
🎯 Chứng minh audit query được (rubric).
🔧 Cách nhanh: `aws s3 cp s3://<bucket>/audit/<tenant>/<cid>.json - | python -m json.tool`. Nâng cao: tạo Athena table trỏ vào bucket, query `WHERE correlation_id=...`.
✅ Kiểm tra: trả full chuỗi event 1 incident.
📎 Evidence: output query.

### C-11 — Điền 07_test_eval (measured) · MUST · C1+C3 · T4
🔧 Điền cột Measured/Actual trong các bảng SLO + Failure Analysis của `07_test_eval` bằng số thật từ C-05..08. Không điền số giả.
✅ Kiểm tra: không còn TBD ở phần đã chạy.
📎 Evidence: commit doc.

### C-12 — Gói evidence pack · MUST · C3 · T4–T5
🔧 Sắp xếp `evidence/` đủ: run_report, audit samples, kubectl outputs, screenshots, cost. Theo `capstone-phase2/reference/CAPSTONE_EVIDENCE_PACK_FORMAT.md`.
✅ Kiểm tra: đủ artifact theo checklist Pack #2.
📎 Evidence: cây thư mục `evidence/`.

### C-13 — SLIDES.pdf + demo video · MUST · C3 · T4–T5
🎯 Phần present ăn điểm lớn. Demo lõi = A-14 (auto-resolve thật) + cross-tenant deny + Kyverno block.
🔧 Slide: kiến trúc, differentiation (3-lớp safety, Pre-Decide Gate, CDO self-capture snapshot), số liệu auto-resolve/cost. Video: quay E2E auto_resolved + 1 deny.
✅ Kiểm tra: video chạy được, slide ≤ thời lượng pitch.
📎 Evidence: SLIDES.pdf + demo-video.mp4.

### C-14 — curveball-responses.md · MUST · C3 · T2 & T4
🔧 Ngay sau curveball #2 (T2 16h) và #3 (T4 14h), ghi: đề bài, cách CDO-02 phản ứng, file/đổi gì. Mẫu mỗi mục: Context · Quyết định · Thay đổi code/doc · Kết quả.
✅ Kiểm tra: 2 mục ghi trong ngày curveball.
📎 Evidence: commit curveball-responses.md.

### C-15 — individual-pitches.md · MUST · cả team · T4
🎯 Chống free-rider: mỗi người walk-through được commit của mình.
🔧 Mỗi thành viên viết 3–5 câu: tôi làm task gì, commit nào, quyết định kỹ thuật + trade-off.
✅ Kiểm tra: 10 mục, mỗi mục có commit SHA.
📎 Evidence: file + SHA.

### C-16 — Giữ docs sync nếu contract đổi · MUST · C3 · ongoing
🔧 Nếu curveball đổi contract: cập nhật header "sync contract-new-X" + bảng I/O + SLA + error table trong cùng PR với code. Grep tìm reference cũ: `grep -rn "contract-new" docs/`.
✅ Kiểm tra: `grep` không còn version cũ lẫn lộn.
📎 Evidence: commit doc sync.
```

---

## 8. W12_TASKS.md - Critical path và quy tắc cắt scope

```text
# 🔗 Critical path & quy tắc cắt scope (A1 quyết)

B-01 EKS ─┬─► A-02/03/04 (K8s thật) ─┐
B-03 audit─┘                          ├─► A-14 E2E thật ─► C-05/06/07 scenario ─► C-11/12/13 evidence+slide
AI image ─► B-10 ─► A-08 integrate ──┘

- B không block A: chưa có cluster → A dev tiếp bằng `CDO_K8S_MOCK=true` + `mock_ai_server.py`.
- Thứ tự cắt khi trễ: C-09 load test → A-13 multi-step → A-12 circuit breaker → B-09 deferred (hạ queue/secret về designed-only; vẫn đạt ≥3 build + ≥2 design).
- KHÔNG bao giờ cắt: safety gate, cross-tenant deny (C-08), audit trail (A-07/C-10), ≥10 scenario + auto-resolve (C-05/06), slides+demo (C-13).
```

---

## 9. W12_TASKS.md - Nghi thức hằng ngày liên quan Team C

```text
# 📌 Nghi thức hằng ngày (mọi người)
- Sáng: pull 1 Jira task → In Progress. Trong ngày: comment ≥1 lần (progress/blocker/ETA).
- Close task bắt buộc gắn evidence link. Không backdate, không batch 5 task/1 phút.
- 14h standup ≤30s/người: Done / Doing / Blocker.
- Escalate mentor ngay khi: 2 ngày cùng blocker · AI–CDO lệch contract · build <50% giữa tuần.
```

---

## 10. 07_test_eval_report - LEGACY/reference only - Mục tiêu test/evidence

Ghi chú: phần này lấy từ file test report cũ, tạo trước khi dự án thật sự bắt đầu. Chỉ dùng để hiểu intent ban đầu và các loại evidence cần có. Cần validate lại với trạng thái executor/infra/AI hiện tại.

```text
Tài liệu này mô tả test cases, phạm vi test và evidence cần thu cho platform CDO-02 của TF3 Self-Heal Engine. Mục tiêu không chỉ là chứng minh một happy path chạy được, mà là chứng minh CDO executor có thể tự động xử lý known patterns trong giới hạn an toàn.

Luồng test chính:

alert -> telemetry -> AI detect -> AI decide -> CDO safety gate -> dry-run -> execute/mock execute -> verify -> audit -> close/rollback/escalate

Boundary dùng cho toàn bộ test:

- AI Engine là decision service qua `/v1/detect`, `/v1/decide`, `/v1/verify`.
- CDO executor là execution boundary duy nhất trước khi có Kubernetes mutation.
- Mọi action phải qua safety gate, dry-run, blast-radius, verify, rollback/circuit breaker và audit.
- Multi-tenant isolation giữa `tenant-a` và `tenant-b` là hard gate. Nếu có cross-tenant action được execute, toàn bộ test suite fail.
```

---

## 11. 07_test_eval_report - LEGACY/reference only - Test coverage raw

```text
| Test type | Tool / Method | Phạm vi | Trạng thái |
|---|---|---|---|
| Contract test | JSON schema + signed AI contract | Validate request/response cho `/v1/detect`, `/v1/decide`, `/v1/verify` | Planned |
| Safety unit test | pytest hoặc module test tương đương | Validate allow-list, tenant match, blast-radius, local rollback/runbook path, `verify_policy`, idempotency | Planned |
| Integration test | Mock AI endpoint + CDO executor | Alert payload -> AI decision -> safety decision -> audit record | Planned |
| Kubernetes action test | EKS/Kubernetes sandbox + server-side dry-run | Restart deployment, scale worker, deny unsafe namespace/action | Planned |
| E2E scenario test | Scenario injector + Prometheus/Alertmanager hoặc offline RE2/RE3 preprocessor | >= 10 injected scenarios trong >= 4h simulation window | Planned |
| Load test | k6/Locust hoặc scenario replay runner | Sustained telemetry/API flow và executor queue behavior | Planned |
| Security test | Manual abuse cases + RBAC checks | Cross-tenant deny, secret/log exposure, IAM/RBAC least privilege | Planned |
| Audit evidence test | S3 Object Lock hoặc append-only audit target + CloudWatch logs | Query toàn bộ events theo `correlation_id`, retention target >= 90 days | Planned |
```

---

## 12. 07_test_eval_report - LEGACY/reference only - Acceptance criteria raw

```text
| Requirement | Target | Evidence source | Trạng thái |
|---|---:|---|---|
| Scenario count | >= 10 injected scenarios | Scenario run log | Pending W12 evidence |
| Simulation window | >= 4 hours | Start/end timestamps trong audit/logs | Pending W12 evidence |
| Auto-resolve rate | >= 60% | Scenario summary table | Pending W12 evidence |
| Unsafe action count | 0 | Safety/audit records | Pending W12 evidence |
| Multi-tenant isolation | 100% deny cho cross-tenant attempts | RBAC + safety tests | Pending W12 evidence |
| Audit trail | 100% scenarios có trace theo `correlation_id` | Audit query output | Pending W12 evidence |
| Safety checkpoints | 5/5 enforced cho mutating actions | Safety gate test output | Pending W12 evidence |
```

---

## 13. 07_test_eval_report - LEGACY/reference only - SLO evidence raw

```text
Các SLO dưới đây là target cho W12 Pack #2. Cột measured chỉ được điền sau khi chạy test thật, không điền số giả định.

| SLO | Target | Measured | Window | Pass/Fail |
|---|---:|---:|---|---|
| CDO executor availability trong simulation | >= 99.5% | TBD | >= 4h scenario window | TBD |
| AI API call p99 do CDO observe | detect < 300ms; decide < 3000ms (LLM) / < 500ms (fallback); verify < 500ms | TBD | Scenario run | TBD |
| AI API abort threshold (contract-new-4 §6.B) | detect p99 ≤ 800ms; decide p99 ≤ 3000ms; verify p99 ≤ 1000ms; 5xx ≤ 1% | TBD | 5-min measurement window | TBD |
| End-to-end auto-heal latency | < 5 min cho safe restart/scale cases | TBD | Successful auto-resolve cases | TBD |
| Audit write coverage | 100% incidents | TBD | Scenario run | TBD |
| Unsafe action rate | 0% | TBD | Scenario run | TBD |
| Tenant onboarding smoke test | < 30 min cho 2 tenants | TBD | `tenant-a`, `tenant-b` setup | TBD |
| Rate limit compliance | detect <= 100 RPS; decide/verify <= 10 RPS per tenant | TBD | Scenario run | TBD |
```

---

## 14. 07_test_eval_report - LEGACY/reference only - Known pattern scenarios TC-01..TC-06

```text
Ghi chú: TC-01 đến TC-06 là build-real — bắt buộc chạy và tính vào auto-resolve rate. TC-05 dùng synthetic signal injection (không cần real queue infrastructure) — xem TC-05 detailed section.

| ID | Scenario | Tenant | Signal source | Expected AI decision | Expected CDO action | Expected result |
|---|---|---|---|---|---|---|
| TC-01 | Service stuck / latency spike | `tenant-a` | `service_latency_p95` | `service_stuck` | `RESTART_DEPLOYMENT` sau khi safety pass | Auto-resolved |
| TC-02 | Service stuck / latency spike | `tenant-b` | `service_latency_p95` | `service_stuck` | `RESTART_DEPLOYMENT` sau khi safety pass | Auto-resolved |
| TC-03 | Error rate spike | `tenant-a` | `service_error_rate`, app logs | `error_rate_spike` | Restart nếu confidence/safety pass, nếu không thì escalate | Auto-resolved hoặc escalated safely |
| TC-04 | Memory pressure / OOM prevention | `tenant-a` | `container_resource_usage` | `memory_pressure` | `PATCH_MEMORY_LIMIT` chỉ khi có verify_policy và local rollback/runbook path | Auto-resolved hoặc denied safely |
| TC-05 | Queue/backpressure | `tenant-b` | Synthetic inject script (`signal_name: queue_backlog, value: 15000`) | `queue_backlog` | `SCALE_REPLICAS` via deferred GitOps path (Git commit → ArgoCD sync) trong giới hạn blast-radius | Auto-resolved via ArgoCD sync |
| TC-06 | Secret/cert expiry | `tenant-a` | `secret_expiry_warning` | `secret_expiry` | `ROTATE_SECRET` via deferred GitOps path (safety gate: allow-list + verify_policy bắt buộc) | Auto-resolved via ArgoCD sync |
```

---

## 15. 07_test_eval_report - LEGACY/reference only - Safety/failure scenarios TC-07..TC-21

```text
| ID | Scenario | Injected fault | Expected CDO decision | Pass condition |
|---|---|---|---|---|
| TC-07 | Cross-tenant target | Incident tenant là `tenant-a`, AI action target `tenant-b` | Deny action | Không có Kubernetes mutation, audit reason `denied_cross_tenant` |
| TC-08 | Action ngoài allow-list | AI trả `DELETE_NAMESPACE` | Deny action | Không có Kubernetes mutation, audit reason `denied_action_not_allowed` |
| TC-09 | Thiếu local rollback/runbook path | Mutating action không có fallback path cục bộ | Deny action | Audit ghi `missing_rollback_path` |
| TC-10 | Thiếu `verify_policy` | Mutating action không có `verify_policy.window_seconds` | Deny action | Audit ghi `missing_verify_policy` |
| TC-11 | Duplicate idempotency key | Retry cùng action với cùng `Idempotency-Key` | Deny duplicate execute | Chỉ có 1 execute event cho key đó |
| TC-12 | AI timeout/503 | `/v1/decide` timeout hoặc trả 503 | Escalate, không execute | Audit ghi `ai_unavailable_escalated` |
| TC-13 | Dry-run failure | Kubernetes server-side dry-run fail | Deny execute | Không có real action sau dry-run fail |
| TC-14 | Verify regression | Post-action metrics xấu hơn | Rollback hoặc escalate | Audit có rollback/escalation event |
| TC-15 | Circuit breaker | Quá nhiều action fail trong thời gian ngắn | Stop automation | Các action tiếp theo bị deny tới khi breaker reset |
| TC-16 | `pattern_type: deferred` routing | AI trả `pattern_type: "deferred"` (e.g. SCALE_REPLICAS) | CDO tạo Git commit/PR, không gọi K8s API trực tiếp | Không có K8s mutation, có Git commit/PR evidence, audit ghi `deferred_gitops_path` |
| TC-17 | `cost_cap_exceeded: true` handling | AI `/v1/decide` trả `cost_cap_exceeded: true` | CDO log warning + notify, vẫn execute action plan theo safety gate | Audit ghi `cost_cap_exceeded_warning`, action executed bình thường |
| TC-18 | 403 Tenant mismatch | `X-Tenant-Id` header ≠ `tenant_id` trong payload | CDO nhận 403, không retry, ghi audit | Audit ghi `tenant_mismatch`, không có action execute |
| TC-19 | Pre-Decide: confidence quá thấp | AI `/v1/detect` trả `confidence=0.40`, `anomaly_detected=true` | CDO discard — không gọi `/v1/decide` | Không có decide call; audit ghi `low_confidence_discard` |
| TC-20 | Pre-Decide: confidence trung bình + severity cao | AI detect trả `confidence=0.65`, `severity=HIGH` | CDO escalate ngay, không gọi `/v1/decide` | Không có decide call; audit ghi `low_confidence_escalated` |
| TC-21 | Pre-Decide: flapping detection | Cùng service bị detect lần 3 trong vòng 10 phút | CDO escalate ở lần 3, không gọi `/v1/decide` thêm | Audit ghi `flapping_escalated`; chỉ có tối đa 2 decide calls trong window |
```

---

## 16. 07_test_eval_report - LEGACY/reference only - Detailed test cases raw excerpts

```text
### TC-01 - Service Stuck Auto-Restart

Goal: Chứng minh CDO có thể auto-resolve một latency spike an toàn bằng cách restart đúng một deployment trong đúng namespace.

Preconditions:
- Namespace `tenant-a` đã tồn tại.
- Target deployment có label `tenant_id=tenant-a`.
- AI mock/real endpoint có thể trả `RESTART_DEPLOYMENT`.
- Audit sink đang available.

Steps:
1. Inject alert với `correlation_id=tc-01-*`, tenant `tenant-a`, namespace `tenant-a`.
2. Cung cấp telemetry cho thấy `service_latency_p95` cao bất thường.
3. CDO gọi `/v1/detect`; lưu `anomaly_context` từ response. CDO gọi `/v1/decide` với body bao gồm `anomaly_context` (bắt buộc theo contract-new-4).
4. AI trả `RESTART_DEPLOYMENT` cho một deployment, có `verify_policy`, `matched_runbook` — CDO capture rollback snapshot TRƯỚC khi execute: đọc K8s API lấy current state (`memory_limit`, `replica_count`, `image_tag`) → lưu vào audit log (AI không trả `rollback_snapshot` theo contract-new-4).
5. Safety gate validate tenant, allow-list, blast-radius, rollback/runbook path, `verify_policy` và idempotency (DynamoDB lock cho `/v1/decide`).
6. CDO chạy server-side dry-run.
7. CDO execute hoặc mock-execute restart. Ghi lại: action, target (string "deployment/<name>"), status (COMPLETED|FAILED).
8. CDO gọi `/v1/verify` với `action_executed: { action, target, status, execution_time_seconds }` và `post_telemetry_window` (required). Xử lý `next_action`: DONE → close; RETRY → retry; ROLLBACK → restore CDO self-captured snapshot từ audit log (kubectl rollout undo hoặc revert manifest); ESCALATE → gửi `escalation_bundle`.
9. CDO ghi full audit trail.

Expected result: Incident được close dưới trạng thái auto-resolved. Audit có `safety_passed`, `dry_run_done`, `execute_done`, `verify_done`, `incident_closed`.

### TC-05 - Queue Backpressure Scale-Out

Goal: Chứng minh CDO có thể auto-scale workload khi queue backlog cao, qua deferred GitOps path — không direct mutate K8s API.

Approach: Synthetic signal injection — inject telemetry payload với `signal_name: "queue_backlog"` và `value: 15000` để trigger AI decision mà không cần real queue infrastructure.

Expected result: Incident được close dưới trạng thái auto-resolved. Audit có `safety_passed`, `deferred_gitops_path`, Git commit hash, ArgoCD sync event, `verify_done`, `incident_closed`. Không có direct K8s mutation nào từ CDO executor (TC-16 cross-check).

### TC-07 - Cross-Tenant Action Deny

Goal: Chứng minh CDO chặn tenant isolation violation ngay cả khi AI trả sai target.

Steps:
1. Inject incident với `tenant_id=tenant-a`.
2. Mock response của AI `/v1/decide` target namespace `tenant-b`.
3. Chạy CDO safety gate.

Expected result: CDO deny trước dry-run/execute. Audit có `safety_denied` với reason `denied_cross_tenant`. Kubernetes mutation count bằng 0.

### TC-12 - AI Timeout Escalation

Goal: Chứng minh platform fail-safe khi AI unavailable.

Steps:
1. Inject một valid incident.
2. Force `/v1/decide` timeout hoặc HTTP 503.
3. Quan sát fallback behavior của CDO.

Expected result: CDO không tự execute static action theo default. Incident được escalate kèm context bundle và audit reason `ai_unavailable_escalated`.

### TC-14 - Verify Regression Rollback/Escalation

Goal: Chứng minh workflow không đánh dấu remediation fail thành success.

Expected result: CDO dùng snapshot đã capture trước execute (lưu trong audit log) để restore trạng thái trước action (kubectl rollout undo / revert manifest về state snapshot). Nếu `next_action=ESCALATE`, CDO gửi `escalation_bundle` {reason, logs, metrics} lên channel cảnh báo — không execute thêm action. Audit có `verify_regression` và `rollback_done` hoặc `escalated`.

### TC-18 - 403 Tenant Mismatch Handling

Goal: Chứng minh CDO xử lý đúng khi `X-Tenant-Id` header không khớp `tenant_id` trong payload.

Expected result: AI trả `403 Forbidden`. CDO không retry, ghi audit `tenant_mismatch`, kiểm tra lại header config trước khi gửi tiếp. Không có action nào được execute.
```

---

## 17. 07_test_eval_report - LEGACY/reference only - Scenario simulation plan raw

```text
W12 evidence run phải inject ít nhất 10 scenarios trong tối thiểu 4 giờ.

| Phase | Duration | Activity | Evidence |
|---|---:|---|---|
| Warm-up | 15 min | Verify namespaces, AI endpoint, audit sink, telemetry path | Readiness log |
| Baseline | 30 min | Chạy workload không inject fault | Baseline metrics |
| Scenario wave 1 | 90 min | TC-01 tới TC-06 cho known pattern cases | Scenario logs + audit |
| Scenario wave 2 | 90 min | TC-07 tới TC-15 cho safety/failure cases | Safety deny/escalation audit |
| Cooldown | 15 min | Verify không còn stuck incident, tổng hợp kết quả | Final report |

Các metric tối thiểu cần thu:

| Metric | Formula |
|---|---|
| Auto-resolve rate | `auto_resolved_count / total_injected_scenarios` |
| Unsafe action count | Số real Kubernetes mutations vi phạm tenant/action/blast-radius rules |
| Audit coverage | `scenarios_with_complete_audit / total_injected_scenarios` |
| Escalation quality | Escalated incidents có logs, metrics, deploy history và attempted actions đi kèm |
```

---

## 18. 07_test_eval_report - LEGACY/reference only - Load/security/multi-tenant raw

```text
## Load Test Results

Planned load profile:
- Ramp-up: 0 -> 100 simulated alert/API events mỗi phút trong 5 phút.
- Sustained: 100 events mỗi phút trong 10 phút.
- Tenants simulated: namespaces `tenant-a`, `tenant-b`; request header dùng CDO-02 tenant UUID `6c8b4b2b-4d45-4209-a1b4-4b532d56a31c` (confirmed deployment contract 2026-06-25).
- Tool: k6, Locust hoặc scenario replay runner.

## Security Test

| Check | Method | Expected result | Status |
|---|---|---|---|
| API auth bypass attempt | Gọi AI/CDO endpoint không có required auth/header | 401/403 hoặc request bị reject | Pending |
| Cross-tenant data/action leak | Dùng incident `tenant-a` để target namespace `tenant-b` | Safety deny + no mutation | Pending |
| Action allow-list bypass | Cho AI trả unsupported action | Safety deny | Pending |
| IAM/K8s privilege escalation | Thử delete namespace hoặc cluster-wide mutation | RBAC deny | Pending |
| Secret exposure via logs | Inspect logs tìm token/SigV4/kube token | Không log sensitive value | Pending |

## Multi-Tenant Isolation Test

Toàn bộ test trong section này phải pass. Nếu có bất kỳ cross-tenant mutation nào execute thành công, đây là SEV1 failure cho capstone.

| Test | Method | Expected result |
|---|---|---|
| Tenant A đọc action context của Tenant B | Inject A token/header nhưng request B target | Request bị deny hoặc context bị omit |
| Tenant A action target Tenant B namespace | Mock AI action target namespace `tenant-b` | Safety deny trước dry-run |
| Tenant A ServiceAccount mutate Tenant B workload | Thử Kubernetes patch bằng A-scoped identity | RBAC deny |
| Cross-tenant queue contamination | Queue message có `tenant_id` và namespace không khớp | Message bị reject hoặc safety denied |
| Audit query by tenant | Query audit cho `tenant-a` | Không expose payload content của `tenant-b` |
```

---

## 19. 07_test_eval_report - LEGACY/reference only - Audit evidence requirements raw

```text
Mọi scenario phải query được bằng `correlation_id`. Audit trail tối thiểu cần có:

alert_received
telemetry_collected
detect_called
detect_response_received
decide_called
action_plan_received
idempotency_lock_acquired or idempotency_duplicate_denied
safety_passed or safety_denied
dry_run_done or dry_run_failed
execute_done or execute_skipped
verify_called
verify_done
rollback_done or escalated or incident_closed

Audit fields tối thiểu:

| Field | Required | Notes |
|---|---|---|
| `timestamp` | Yes | RFC3339 UTC |
| `correlation_id` | Yes | Stable trong toàn workflow |
| `tenant_id` | Yes | Phải khớp namespace mapping |
| `namespace` | Yes | Kubernetes target namespace |
| `action_type` | Yes cho decision/action events | Lấy từ allow-list |
| `decision` | Yes | execute, deny, escalate, rollback, close |
| `result` | Yes | success, failure, denied, skipped |
| `reason` | Yes cho deny/failure | Machine-readable reason |
| `idempotency_key` | Yes cho mutating workflow | Dùng để chống duplicate execution |
```

---

## 20. 07_test_eval_report - LEGACY/reference only - Test gaps acknowledged raw

```text
Các gap đã biết trước W12 execution:

- Real production traffic nằm ngoài scope; test dùng sandbox workload và/hoặc RE2/RE3 offline simulation.
- Cross-region audit replication nằm ngoài scope.
- Full long-term Prometheus retention với Thanos/Cortex/Mimir nằm ngoài scope.
- Real PagerDuty/OpsGenie integration nằm ngoài scope; Slack/mock pager escalation là đủ cho capstone.
- Nếu AWS/EKS quota hoặc account access chặn full deployment, evidence phải ghi rõ test nào chạy trên Kubernetes sandbox và test nào chạy ở offline/mock mode.
```

---

## 21. 07_test_eval_report - LEGACY/reference only - Final summary raw

```text
Điền bảng này sau W12 evidence run.

| Summary metric | Target | Actual | Pass/Fail |
|---|---:|---:|---|
| Total scenarios injected | >= 10 | TBD | TBD |
| Scenario window | >= 4h | TBD | TBD |
| Auto-resolve rate | >= 60% | TBD | TBD |
| Unsafe actions | 0 | TBD | TBD |
| Cross-tenant leaks | 0 | TBD | TBD |
| Complete audit coverage | 100% | TBD | TBD |
| Critical security findings | 0 | TBD | TBD |
```

---

## 22. CAPSTONE_EVIDENCE_PACK_FORMAT.md - Evidence pack raw excerpts

```text
Document trail của bạn = công cụ chính để show process thinking, không chỉ "đoạn code chạy được". Reviewer chấm dựa trên 3 layer evidence:

1. Document quality - thinking, trade-off analysis, justification (đây là evidence pack)
2. Working artifact - repo, code, infra, pipeline, test outputs
3. Demo/presentation - ability to explain and defend

Doc viết live trong repo (markdown), git history = evidence cho process - không phải "viết một phát cuối".

Evidence Pack #2 ⭐ | EOD T4 W12 (cùng code freeze 18h) | TẤT CẢ doc final + test + eval + cost + ops | ~30% điểm W12 + input chính cho buổi chấm T5.

`07_test_eval_report.md` | new | SLO evidence + load test + multi-tenant isolation

Test scenarios:
List ≥10 scenarios cover happy path + edge cases.

Scenarios fail → root cause → fix attempted → result.

Evidence Pack #2 (EOD T4 W12) - MAIN ⭐ + code freeze 18h:
- [ ] `07_test_eval_report.md` new với chaos response evidence
```

---

## 23. TF3_SELFHEAL_LEARNER.md - Rubric/target raw excerpts

```text
Không phải research project - artefact demo trên sandbox cluster cuối W12, evidence rõ ràng.

- Auto-resolve rate ≥60% trên ≥10 scenarios injected.
- Scenario simulation ≥4h test window (KHÔNG yêu cầu 1-week real observation).
- 1-week real observation (scenario simulation ≥4h đủ).
- Mandatory request fields: `idempotency_key`, `dry_run_mode`, `correlation_id`.
- Scenario simulation ≥4h: ≥10 scenarios, auto-resolve rate report.
- Action semantics: "auto-resolved" định nghĩa gì - execute success hay metric returns normal.
```

---

## 24. Raw interpretation note - Team C can work independently on these

```text
Team C có thể tự làm độc lập:
- scenario JSON
- preprocessor telemetry
- run_all.py / run_report.json
- mock/offline test flow
- test report skeleton and measured fields after run
- evidence folder structure
- audit query commands/templates
- slides/demo script
- curveball-responses.md
- individual-pitches.md

Team C cần input/xác nhận từ Team A:
- executor chạy mock/real như thế nào
- action nào đã implement
- audit reason code thực tế
- log/audit output format

Team C cần input/xác nhận từ Team B:
- namespace/RBAC/Kyverno/EKS readiness
- S3 audit bucket, DynamoDB, SQS info
- kubectl/RBAC/Kyverno evidence

Team C cần input/xác nhận từ AI/A4:
- AI endpoint mock/real
- schema compatible contract-new-4
- cách simulate failure response cho TC-07..TC-21
```
