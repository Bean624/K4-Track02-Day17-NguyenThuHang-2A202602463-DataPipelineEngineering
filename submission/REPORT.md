# K4-Track02-Day17 — Report cá nhân

Phần phân tích tối đa một trang, không tính output ở phần 5.
Định dạng tham chiếu và phạm vi tính trang: [SUBMISSION.md](../docs/SUBMISSION.md).

**Họ tên / MSSV:** Nguyễn Thu Hằng 

**Repo:** K4-Track02-Day17-NguyenThuHang-2A202602463-DataPipelineEngineering

**AI đã dùng và phạm vi hỗ trợ (hoặc không dùng):** Gemini, ChatGPT  

**Nguồn tham khảo khác (nếu có):**

## 1. Ba lỗi

Mỗi lỗi 4 dòng. Triệu chứng = thứ bạn *thấy* đầu tiên (check nào fail, số nào lạ,
checksum nào lệch) — không phải cách sửa.

| | Lỗi Silver | Lỗi late data | Lỗi xoá (CDC) |
|---|---|---|---|
| **Triệu chứng** |Check fail: silver_tickets có 24 dòng cho 12 vé (bị lặp). T-91 có 3 trạng thái cũ/mới thay vì chỉ 1 trạng thái mới nhất (high/closed/bug). |Check fail: LOOKBACK_DAYS=0 < 3 (P99 đo được 3 ngày). Các sự kiện ngày 08-12 của u05 đến muộn vào ngày 08-15 chỉ đếm được (2, 1, 0) thay vì (5, 3, 1). |Check fail: T-97 không thành tombstone (is_deleted vẫn False, còn email/SĐT). T-97 vẫn lọt vào training set (1 row) và RAG chunks (2 chunks). |
| **Nguyên nhân gốc** |Hàm upsert_silver_tickets dùng INSERT INTO cho mỗi batch, khiến mỗi lần vé có cập nhật lại bị chèn thêm một dòng mới thay vì ghi đè theo khoá. |LOOKBACK_DAYS = 0 ngây thơ giả định sự kiện luôn đến ngay. Khi người dùng u05 offline gửi trễ 3 ngày, pipeline không tính lại các partition ngày cũ nên bỏ sót dữ liệu. | Khi op = 'd' (xoá), Debezium trả về after = null. Staging chỉ đọc ticket_id từ after nên nhận null và bị loại bỏ bởi điều kiện WHERE ticket_id IS NOT NULL, khiến Silver không nhận được sự kiện xoá.
|
| **Cách sửa** (file, vài dòng) |Sửa file pipeline/silver.py: đổi INSERT INTO thành MERGE INTO theo khoá ticket_id, chỉ UPDATE khi source._lsn > target._lsn, nếu chưa có thì INSERT. |Sửa file pipeline/config.py: đổi LOOKBACK_DAYS = 0 thành LOOKBACK_DAYS = 3 (bao phủ P99 lateness đo được từ Bronze) để tính bù dữ liệu trễ vào đúng ngày sự kiện xảy ra. |Sửa pipeline/staging.py: dùng coalesce(after->ticket_id, before->ticket_id, key->ticket_id) để lấy ticket_id từ before hoặc key khi after bị null.
 |
| **Khái niệm trên slide** |Silver có khoá, MERGE / Upsert idempotent, CDC Log Sequence Number (_lsn). |Event time vs Ingest time, Lateness & P99, Cửa sổ lookback (lookback window), Ghi đè partition theo ngày sự kiện (idempotent overwrite). |CDC log-based, Debezium envelope (before/after/op), Xoá phải lan (Delete propagation), Tombstone.
 |


## 2. Các con số

- P99 lateness đo từ Bronze: `3` ngày → `LOOKBACK_DAYS = 3`
- `submission/checksums.txt`: PASS / FAIL — Gold checksum: `39e115c510ecdf526800eac227158a4f`
- `make parity`: PARITY 

## 3. Lựa chọn công cụ / kỹ thuật (mỗi dòng một câu "vì sao")

- MERGE theo khoá cho `silver_tickets`, overwrite-partition cho `gold_feature_daily`: MERGE theo khoá ticket_id bảo đảm tính idempotent cho bảng thực thể; overwrite-partition giúp tính lại các ngày trong cửa sổ lookback mà không ảnh hưởng ngày khác.
- Tombstone thay vì xoá hẳn hàng trong Silver: Giữ lại tombstone kèm _lsn để phân biệt giữa "chưa từng tồn tại" và "đã bị xoá", ngăn các batch cũ khi replay hồi sinh lại vé đã xoá.
- Snapshot training dựng lại từ Bronze "as of" ngày đó, không sửa snapshot cũ: Bảo đảm tính tái lập (reproducibility) cho Machine Learning; mô hình huấn luyện ở ngày nào phải nhìn thấy đúng dữ liệu tại ngày đó.
- DuckDB (lite) / dbt (track dbt) cho bài toán cỡ này, chứ không phải Spark: Dữ liệu vừa và nhỏ (medium data) xử lý trên single-node DuckDB nhanh hơn Spark hàng chục lần vì không tốn overhead mạng phân tán, zero-dependency và tiết kiệm chi phí.


## 4. Hai câu hỏi suy ngẫm

1. Tách bạch mục đích: Snapshot cũ giữ lại để phục vụ kiểm toán (lineage/audit), nhưng đánh dấu phiên bản hết hạn. Khi huấn luyện mô hình mới, bắt buộc dùng snapshot v2026-08-16 đã loại bỏ T-97. Nếu quy định pháp lý (GDPR) bắt buộc xoá tuyệt đối ở mọi bản sao lưu, cần chạy một quy trình dọn dẹp ghi đè (purging pipeline) kèm biên bản kiểm toán xoá dữ liệu.
2. Cần đặt chốt PII ở tầng Silver bằng mô hình NER (Named Entity Recognition) hoặc công cụ như Microsoft Presidio để nhận diện tên người tiếng Việt. Đo lường hiệu quả bằng độ bao phủ (Recall - không bỏ sót tên) và tỷ lệ rò rỉ PII (PII Leakage Rate) trên tập dữ liệu kiểm thử được gán nhãn.


## 5. Output (dán nguyên văn)

```text
$ .\.venv\Scripts\python.exe -m scripts.verify
=== verify.py — Day 17 pipeline contracts ===
  [OK ] Bronze  every daily batch landed as Parquet (7 days x 3 sources)
  [OK ] Bronze  re-landing a batch is a no-op (append-only, no duplicate file)
  [OK ] Bronze  Bronze keeps the raw truth: Kafka tombstone + redelivered events are still there
  [OK ] Silver  silver_tickets has exactly one row per ticket_id
  [OK ] Silver  T-91 shows its latest state: high / closed / bug
  [OK ] Silver  deleted ticket T-97 is a tombstone: is_deleted and no personal data left
  [OK ] Silver  no email / phone number survives past Bronze
  [OK ] Silver  silver_events has one row per event_id (Kafka redeliveries removed)
  [OK ] Silver  2 malformed events quarantined with a reason; the run did not halt
  [OK ] Gold    gold_feature_daily reconciles with a full recompute from Silver
  [OK ] Gold    u05's offline events of 08-12 (arrived 08-15) are counted on 08-12
  [OK ] Gold    LOOKBACK_DAYS covers measured P99 lateness (p99=3.00 days)
  [OK ] Gold    training set uses point-in-time priority (T-91 created as 'low')
  [OK ] Gold    late feedback creates a NEW snapshot version; the old one is untouched
  [OK ] Gold    latest training snapshot excludes the deleted ticket T-97
  [OK ] Gold    deletes propagate to the RAG index: no chunk of T-97
  [OK ] Gold    gold_doc_chunks: one row per chunk, and a re-run embeds 0 new chunks
  [OK ] Rerun   re-run 2026-08-12 three times -> Gold checksum identical to a fresh build

RESULT: 18/18 checks — ALL PASS

$ .\.venv\Scripts\python.exe -m pytest
..................................                                                                               [100%]
34 passed in 2.21s

$ .\.venv\Scripts\python.exe -m scripts.rerun_check
# Lab 17 — re-run check for 2026-08-12

run                     gold_feature_daily    gold_training_set     gold_doc_chunks       gold (combined)
fresh build             8630e04a61d1          9370ca77af23          cb9ebd12fdcc          39e115c510ecdf526800eac227158a4f
re-run #1 of 2026-08-12 8630e04a61d1          9370ca77af23          cb9ebd12fdcc          39e115c510ecdf526800eac227158a4f
re-run #2 of 2026-08-12 8630e04a61d1          9370ca77af23          cb9ebd12fdcc          39e115c510ecdf526800eac227158a4f
re-run #3 of 2026-08-12 8630e04a61d1          9370ca77af23          cb9ebd12fdcc          39e115c510ecdf526800eac227158a4f

RESULT: PASS — 3 re-runs, identical checksums

$ .\.venv\Scripts\python.exe main.py --lateness
event lateness over 43 Bronze records (calendar days): p50=0.00 p95=2.90 p99=3.00 max=3
-> lookback must be >= ceil(p99) = 3 day(s); config.LOOKBACK_DAYS = 3

$ Push-Location dbt_project; try { ..\.venv\Scripts\dbt.exe build --profiles-dir . --event-time-start 2026-08-10 --event-time-end 2026-08-17 } finally { Pop-Location }
Finished running 3 incremental models, 13 data tests, 1 unit test, 2 view models in 0 hours 0 minutes and 1.52 seconds (1.52s).
Completed successfully
Done. PASS=19 WARN=0 ERROR=0 SKIP=0 NO-OP=0 REUSED=0 TOTAL=19

$ .\.venv\Scripts\python.exe -m scripts.parity
=== parity: lite pipeline vs dbt ===
  [OK ] silver_tickets       lite 3c15dfd43701  dbt 3c15dfd43701
  [OK ] gold_feature_daily   lite 8630e04a61d1  dbt 8630e04a61d1
RESULT: PARITY — both implementations agree

```

Nếu dùng PowerShell, ghi lệnh tương đương và output thực tế theo [SUBMISSION.md](../docs/SUBMISSION.md).
Nếu làm bonus, thêm output B1 hoặc đường dẫn bằng chứng B2 ở cuối phần này.
