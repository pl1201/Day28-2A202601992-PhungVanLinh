# Reflection — Day 28 Track 2

## Phạm vi và đóng góp

Bài được thực hiện cá nhân bởi Phùng Văn Linh. Phần triển khai trực tiếp tập trung vào
bốn boundary còn thiếu trong starter repository:

- IP01/IP10: bảo toàn `traceparent` và `idempotency-key` qua Kafka header;
- IP03: loại bản ghi replay trước Delta MERGE bằng khóa ổn định;
- IP04: giữ đúng contract giữa API và Feast feature registry;
- IP07/IP08: phân biệt `ready`, `degraded` và `not_ready` theo mức bắt buộc của dependency.

Ngoài phần mã nguồn, bài kiểm tra contract, portability, Kubernetes/GitOps manifest và
quy tắc không đưa secret/runtime state vào Git cũng được thực hiện.

## Các lựa chọn kỹ thuật và trade-off

### Idempotency và thứ tự sự kiện

`idempotency_key` được dùng làm đơn vị duy nhất của một fact, thay vì `event_id` vì mỗi
lần delivery có thể có event ID khác nhau. Khi nhiều event có cùng khóa, winner được
chọn theo `(occurred_at, event_id)`. `occurred_at` biểu diễn phiên bản nghiệp vụ mới
nhất; `event_id` là tie-breaker ổn định để kết quả không phụ thuộc thứ tự Kafka giao
message. Sau đó danh sách được sắp xếp theo khóa để Spark luôn nhận nguồn MERGE có thứ
tự xác định.

Trade-off là một correction cũ nhưng đến muộn sẽ không ghi đè correction mới. Nếu
production cần xử lý event-time phức tạp hơn, cần watermark, chính sách late event và
quarantine thay vì chỉ so timestamp.

### Trace context

Kafka header luôn mang `idempotency-key` dưới dạng bytes. `traceparent` chỉ được gửi
khi tồn tại; gửi chuỗi rỗng sẽ tạo W3C context không hợp lệ và làm trace bị tách. Thiết
kế hiện chỉ truyền trace context tối thiểu. Production nên cân nhắc thêm `tracestate`,
baggage có allowlist và giới hạn kích thước header.

### Feast contract

Client gọi feature view `asker_activity_v1` bằng entity `asker_id` và yêu cầu rõ bốn
feature. `full_feature_names=false` giữ response đơn giản cho serving code. Việc ghi rõ
danh sách feature giúp phát hiện registry drift sớm, nhưng mỗi lần đổi feature view cần
phát hành đồng bộ producer, materialization và serving client.

### Readiness và degraded mode

Dependency bắt buộc lỗi dẫn đến `not_ready`; dependency tùy chọn lỗi dẫn đến
`degraded`; tất cả probe khỏe là `ready`. Qdrant/retrieval là bắt buộc vì không có nguồn
grounding thì không nên trả lời. Feast và vLLM có thể tùy chính sách cho phép degraded,
nhưng response phải công khai lý do và không được giả vờ là kết quả đầy đủ.

Trade-off của degraded mode là tăng availability nhưng làm chất lượng câu trả lời không
đồng nhất. Production cần SLO riêng cho full-quality và degraded responses.

### Model release và rollback

MLflow alias `champion` tách quyết định vận hành khỏi mã nguồn. Release gắn prompt,
retrieval configuration, model ID, embedding ID và Delta version để rollback tái lập
được. Alias rollback nhanh hơn build/deploy lại image, nhưng cần audit log, quyền thay
alias và compatibility gate giữa model, prompt, embedding và schema.

### GitOps

Desired state nằm trong Git và Argo CD thực hiện sync/self-heal. Rollback phải revert
revision hoặc image tag bất biến, không live-edit cluster. Cách này tạo audit trail rõ
ràng nhưng phụ thuộc quy trình review, image provenance và khả năng phục hồi Git/registry.

## Production gaps

- Chưa có authentication/authorization hoàn chỉnh cho API, Airflow, MLflow và dashboard.
- Secret cần chuyển sang secret manager; không đặt trong Git, ConfigMap hoặc evidence.
- Kafka cần multi-broker, replication, ACL, TLS, schema registry và chiến lược partition.
- Delta cần object storage bền vững, catalog, retention/vacuum policy và disaster recovery.
- Feast/Qdrant/MLflow cần HA, backup, restore test và capacity planning.
- Cần kiểm soát PII, data retention, prompt injection, output safety và audit access.
- Cần SLO/error budget, paging route, dashboard ownership và runbook được diễn tập.
- Cần canary/shadow evaluation trước promotion cùng tiêu chí rollback tự động.
- Cần SBOM, image signing, vulnerability scanning và admission policy.
- Kết quả load test trên laptop không đại diện production; benchmark phải ghi phần cứng,
  model, dataset, warm-up, concurrency và degraded policy.

## Kết luận readiness

Mã nguồn và contract tĩnh chỉ chứng minh tính đúng của implementation. Kết luận
production readiness chỉ được đưa ra sau khi full stack chạy, các journey J1–J5 đạt,
failure/recovery chứng minh không mất dữ liệu, và evidence có timestamp cùng run ID,
trace ID, Delta version và MLflow version. IP07 chỉ được xác minh khi endpoint chứng minh
đó là vLLM thật qua `/version`, `/v1/models` và metric `vllm:`; nếu không có GPU hoặc
credential, trạng thái đúng là `UNVERIFIED`, không dùng mock để thay thế.
