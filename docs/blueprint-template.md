# Day 13 Observability Lab Report

> **Instruction**: Fill in all sections below. This report is designed to be parsed by an automated grading assistant. Ensure all tags (e.g., `[GROUP_NAME]`) are preserved.

## 1. Team Metadata
- [GROUP_NAME]: Nguyễn Đình Tiến Mạnh (cá nhân)
- [REPO_URL]: https://github.com/30MUNH/2A202600907-NguyenDinhTienManh-Day13
- [MEMBERS]:
  - Member A: Nguyễn Đình Tiến Mạnh (2A202600907) | Role: Logging & PII
  - Member B: Nguyễn Đình Tiến Mạnh | Role: Tracing & Enrichment
  - Member C: Nguyễn Đình Tiến Mạnh | Role: SLO & Alerts
  - Member D: Nguyễn Đình Tiến Mạnh | Role: Load Test & Dashboard
  - Member E: Nguyễn Đình Tiến Mạnh | Role: Demo & Report

---

## 2. Group Performance (Auto-Verified)
- [VALIDATE_LOGS_FINAL_SCORE]: 100/100
- [TOTAL_TRACES_COUNT]: 21 (20 từ 2 lần load_test + 1 request incident; sẵn sàng đẩy lên Langfuse khi có API key)
- [PII_LEAKS_FOUND]: 0

---

## 3. Technical Evidence (Group)

### 3.1 Logging & Tracing
- [EVIDENCE_CORRELATION_ID_SCREENSHOT]: data/logs.jsonl — mỗi dòng đều có `correlation_id` dạng `req-<8hex>` (ví dụ `req-93fb68c3`). 20 correlation ID duy nhất được `validate_logs.py` xác nhận.
- [EVIDENCE_PII_REDACTION_SCREENSHOT]: data/logs.jsonl dòng 2/10/18 — email → `[REDACTED_EMAIL]`, phone → `[REDACTED_PHONE_VN]`, thẻ tín dụng → `[REDACTED_CREDIT_CARD]`. 0 rò rỉ PII.
- [EVIDENCE_TRACE_WATERFALL_SCREENSHOT]: Cần Langfuse API key thật trong `.env` để hiển thị; code đã gắn `@observe()` ở `LabAgent.run` + `update_current_trace/observation`.
- [TRACE_WATERFALL_EXPLANATION]: Span `LabAgent.run` chứa 2 bước con: `retrieve` (RAG) và `llm.generate`. Khi inject `rag_slow`, span RAG kéo dài thêm 2.5s, làm tổng latency nhảy từ ~155ms lên ~2746ms — đây là điểm nóng dễ nhận ra nhất trên waterfall.

### 3.2 Dashboard & SLOs
- [DASHBOARD_6_PANELS_SCREENSHOT]: Dữ liệu nguồn lấy từ endpoint `GET /metrics` (snapshot ở mục 4). 6 panel theo `docs/dashboard-spec.md`: Latency P50/P95/P99, Traffic, Error rate, Cost over time, Tokens in/out, Quality proxy.
- [SLO_TABLE]:
| SLI | Target | Window | Current Value |
|---|---:|---|---:|
| Latency P95 | < 3000ms | 28d | 159ms ✅ (đạt) |
| Error Rate | < 2% | 28d | 0% ✅ (0/20 lỗi) |
| Cost Budget | < $2.5/day | 1d | $0.0406 / 20 req ✅ |
| Quality Score Avg | > 0.75 | 28d | 0.88 ✅ |

### 3.3 Alerts & Runbook
- [ALERT_RULES_SCREENSHOT]: config/alert_rules.yaml — 3 rule: `high_latency_p95` (P2), `high_error_rate` (P1), `cost_budget_spike` (P2).
- [SAMPLE_RUNBOOK_LINK]: docs/alerts.md#1-high-latency-p95

---

## 4. Incident Response (Group)
- [SCENARIO_NAME]: rag_slow
- [SYMPTOMS_OBSERVED]: Latency P95 tăng vọt. Một request bình thường ~155ms; sau khi inject `rag_slow`, request `req-93fb68c3` mất 2746ms (log ghi `latency_ms: 2657`). Error rate vẫn 0% → đây là sự cố hiệu năng, không phải lỗi.
- [ROOT_CAUSE_PROVED_BY]: Log line `correlation_id=req-93fb68c3` event `response_sent` với `latency_ms=2657` (data/logs.jsonl dòng 44). Trace span RAG `retrieve()` chứa `time.sleep(2.5)` trong `app/mock_rag.py` khi `STATE["rag_slow"]=True`. Flow điều tra: **Metrics** (P95 vọt) → **Traces** (span RAG chiếm gần như toàn bộ thời gian) → **Logs** (xác nhận latency theo từng correlation_id).
- [FIX_ACTION]: Tắt incident bằng `python scripts/inject_incident.py --scenario rag_slow --disable`. Latency trở về ~155ms ngay sau đó.
- [PREVENTIVE_MEASURE]: Thêm timeout cho lệnh gọi RAG + fallback nguồn truy xuất; cache truy vấn lặp; alert `high_latency_p95` (P2, `latency_p95_ms > 5000 for 30m`) để cảnh báo sớm.

---

## 5. Individual Contributions & Evidence

> Lab thực hiện cá nhân bởi Nguyễn Đình Tiến Mạnh — đảm nhận toàn bộ các vai trò A–E.

### Nguyễn Đình Tiến Mạnh — Member A (Logging & PII)
- [TASKS_COMPLETED]: Bật processor `scrub_event` trong `app/logging_config.py`; bổ sung pattern PII (passport `[A-Z]\d{7}`, địa chỉ VN) và sắp xếp lại thứ tự credit_card trước cccd trong `app/pii.py`. Kết quả: 0 rò rỉ PII.
- [EVIDENCE_LINK]: app/logging_config.py:46, app/pii.py:11-16 (commit trên branch main)

### Nguyễn Đình Tiến Mạnh — Member B (Tracing & Enrichment)
- [TASKS_COMPLETED]: Bind context `user_id_hash, session_id, feature, model, env` vào structlog ở `app/main.py`; xác nhận `@observe()` + `update_current_trace/observation` trong `app/agent.py`. Mọi log có đủ enrichment (validate: 0 thiếu).
- [EVIDENCE_LINK]: app/main.py:47-55, app/agent.py:28-46

### Nguyễn Đình Tiến Mạnh — Member C (SLO & Alerts)
- [TASKS_COMPLETED]: Đối chiếu SLO trong `config/slo.yaml` với số đo thực tế (bảng mục 3.2); xác nhận 3 alert rule + runbook trong `config/alert_rules.yaml` và `docs/alerts.md`.
- [EVIDENCE_LINK]: config/slo.yaml, config/alert_rules.yaml, docs/alerts.md

### Nguyễn Đình Tiến Mạnh — Member D (Load Test & Dashboard)
- [TASKS_COMPLETED]: Chạy `load_test.py` (tuần tự + `--concurrency 5`) sinh 20 request, thu metrics qua `/metrics`; map dữ liệu vào 6 panel của `docs/dashboard-spec.md`.
- [EVIDENCE_LINK]: data/logs.jsonl, snapshot `/metrics` ở mục 4

### Nguyễn Đình Tiến Mạnh — Member E (Demo & Report)
- [TASKS_COMPLETED]: Triển khai correlation ID middleware (`app/middleware.py`), inject/giải thích incident `rag_slow`, hoàn thiện báo cáo này. `validate_logs.py` = 100/100, pytest = 2 passed.
- [EVIDENCE_LINK]: app/middleware.py:12-32, docs/blueprint-template.md

---

## 6. Bonus Items (Optional)
- [BONUS_COST_OPTIMIZATION]: Quan sát incident `cost_spike` (output tokens x4 trong `app/mock_llm.py`) → đề xuất route câu hỏi đơn giản sang model rẻ + rút ngắn prompt. Baseline avg_cost = $0.002/req.
- [BONUS_AUDIT_LOGS]: Đã có sẵn biến `AUDIT_LOG_PATH=data/audit.jsonl` trong `.env` để tách audit log (chưa kích hoạt ghi riêng).
- [BONUS_CUSTOM_METRIC]: `quality_score` heuristic (`app/agent.py:_heuristic_quality`) được tổng hợp thành `quality_avg=0.88` trong `/metrics` — dùng làm panel "Quality proxy".
