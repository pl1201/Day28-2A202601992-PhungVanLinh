# Load profile

Thời điểm chạy: 2026-09-03, Docker Compose profile `full` trên máy phát triển Windows.

```json
{
  "requests": 200,
  "workers": 8,
  "status_counts": {"200": 46, "0": 154},
  "latency_ms": {"p50": 13.259700001071906, "p95": 921.2023000000045, "p99": 3598.3348000008846}
}
```

`status 0` là exception/timeout ở client probe, không phải HTTP status. Sau tải, container vẫn chạy và `/ready` trả HTTP 200 (`degraded`). Bottleneck là readiness sâu fan-out đồng bộ đến nhiều dependency trên Docker Desktop; xem phân tích trong `SUBMISSION_REPORT.md`.
