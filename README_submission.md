# Báo Cáo Nộp Bài Lab 17 - Multi-Memory Agent với Zep

## 1. Trả Lời Các Câu Hỏi Bắt Buộc (README_submission)

### Câu 1: Layer quan trọng nhất trong bộ test này và case minh họa

- **Layer quan trọng nhất:** `Long-term (Declarative) Memory`.
- **Lý do & Case minh họa:** Bộ test đánh giá khả năng duy trì preferences, decisions và open loops qua nhiều session khác nhau. Ví dụ case **E02** (`Python preference`), **E03** (`open loop deadline 16:00`), **E08** (`recency: BLUEBIRD-42 chuyển sang TypeScript/NestJS`), và **E09** (`user isolation`). Nếu không có long-term memory, agent hoàn toàn mất ngữ cảnh user giữa các session.

### Câu 2: Trade-off giữa Zep Context Block và Redis + Qdrant tự xây dựng

- **Zep Context Block (Managed Memory):** Tự động tổng hợp facts, entities, thread summary và relevance graph thành một Context Block có sẵn. Tiết kiệm thời gian phát triển, tự xử lý async ingestion, recency ranking và privacy consent. Tuy nhiên phụ thuộc vào cloud service/API.
- **Redis + Qdrant (Self-hosted/Manual):** Cho phép kiểm soát hoàn toàn dữ liệu local, tùy chỉnh vector index và caching strategy. Tuy nhiên phải tự xây dựng toàn bộ pipeline trích xuất facts, xử lý graph relational queries, compaction và resolution conflict khi có thông tin mới/cũ.

### Câu 3: Guardrail chống Memory Poisoning (Nhiễm độc bộ nhớ)

- **Kiểm soát Consent & PII Redaction:** Áp dụng `privacy_guard.py` để redact email, số điện thoại và kiểm tra opt-in trước khi nạp vào durable memory.
- **User Scoping & Isolation:** Bắt buộc phân tách dữ liệu theo `user_id` và `thread_id`. Không cho phép thông tin từ user này trôi sang user khác (như kiểm chứng ở case **E09**).
- **Validation & Schema Policy:** Chỉ chấp nhận thông tin hợp lệ qua quy trình kiểm duyệt (như `heartbeat` dry-run), loại bỏ các prompt injection cố tình ghi đè instruction hệ thống vào bộ nhớ dài hạn.

---

## 2. Phân Tích Kết Quả Benchmark

### Phân tích 4 câu benchmark:

1. **Layer nào có hit rate thấp nhất?**
   - Ở bản **No-memory baseline**, các layer `long_term`, `episodic`, và `semantic` đạt hit rate **0%** (2/11 pass nhờ `short_term` local).
   - Khi bật **Student Memory (Zep)**, tất cả các layer đều đạt hit rate **100%** (11/11 PASS).
2. **Query nào retrieve nhiều token nhất?**
   - Query **E07 (mixed)** và **E03 (long-term open loop)** truy xuất lượng token cao nhất do phải kết hợp nhiều thông tin từ Context Block, edge facts và semantic documents.
3. **Case mixed (E07) cần kết hợp memory nào? Evidence nào bắt buộc?**
   - E07 kết hợp **Long-term Memory** (lấy preference lập trình của Minh là `Python`) và **Semantic Memory** (lấy quy tắc retry `Idempotency-Key`).
   - Evidence bắt buộc: `Python` và `Idempotency-Key`.
4. **Token reduction và lý do No-memory có reduction cao nhưng hit rate thấp:**
   - No-memory đạt token reduction **81.8%** vì nó không truy xuất bất kỳ bộ nhớ dài hạn nào (chỉ giữ ngắn hạn). Việc không load context giúp giảm bớt token nhưng làm giảm hit rate xuống chỉ còn **18.2%**.
   - Student Memory đạt hit rate **100.0%** với token reduction hợp lý **14.2%** nhờ cơ chế trimming 10/4/3/3 của `ContextBudgetManager`.

---

## 3. Ghi Chú Cơ Chế Đặc Thụ

- **E08 Recency:** Khi user cập nhật preference từ Python sang TypeScript/NestJS cho dự án `BLUEBIRD-42`, Zep đánh dấu khoảng thời gian hiệu lực (`valid_at`, `invalid_at`) trên graph edges, giúp truy xuất đúng thông tin mới nhất.
- **E10 Compaction:** Cấu hình `ShortTermMemory` nén các turn cũ thành durable notes (ví dụ: `REVIEW-DEADLINE-1600`), giúp giữ lại mốc thời gian quan trọng ngay cả khi giảm `max_recent_messages` xuống 4.
