# Kết quả và giải thích theo rubric

Số liệu trích từ output đã lưu trong [notebooks/](notebooks/), lần chạy ngày 2026-10-04
trên đường lightweight. Log bổ sung nằm trong [evidence/](evidence/).

## Part C — Reproducibility

| Kiểm tra | Kết quả | Bằng chứng |
|---|---|---|
| Smoke test | 9/9 PASS | [smoke_output.txt](evidence/smoke_output.txt) |
| Pytest | **24 passed** | [pytest_output.txt](evidence/pytest_output.txt) |
| `scripts/run_all.py` | **8/8 PASS** | [run_all_output.txt](evidence/run_all_output.txt) |

---

## NB1 — Delta basics

| Tiêu chí | Kết quả |
|---|---|
| `_delta_log/` có commit JSON | 2 commit: `00000000000000000000.json`, `00000000000000000001.json` ([listing](evidence/nb01_delta_log_listing.txt), [nội dung commit 1](evidence/nb01_commit_00000000000000000001.json)) |
| Schema enforcement | Ghi `age='thirty'` bị chặn: `Cast error: Cannot cast string 'thirty' to value of Int64 type` |
| Schema evolution | `schema_mode="merge"` thêm cột `tier`; DuckDB thấy 2 nhóm `[('premium', 1), (None, 3)]` |

**Giải thích.** Mỗi lần ghi thành công tạo đúng một file JSON trong `_delta_log/`, chứa
`commitInfo`, `metaData` (schema) và các action `add`, kèm stats min/max theo cột. Bảng là
"log + file Parquet", không chỉ là thư mục Parquet. Lần ghi sai kiểu bị chặn trước khi
có commit nên không để lại version rác. Commit 1 cho thấy `schemaString` mới có thêm `tier`.
Evolution chỉ xảy ra khi người ghi **chủ động** bật `merge`, và 3 dòng cũ đọc ra `tier = null`
mà không phải viết lại dữ liệu.

## NB2 — OPTIMIZE + Z-ORDER

| Tiêu chí | Kết quả |
|---|---|
| Small-file trước OPTIMIZE | **200 file** (≥ 100) |
| Sau OPTIMIZE + Z-ORDER | **55 file** (giảm ~4×) |
| Query point lookup | 132.2 ms → 15.3 ms, **speedup 8.6×** (≥ 3×) |
| Files-pruned | **55×**: chỉ 1/55 file có range `user_id` chứa giá trị cần tìm |

**Giải thích.** Trước OPTIMIZE, mỗi file chứa `user_id` rải rác nên range min/max của các file
chồng lấn nhau và engine phải mở cả 200 file. Z-order sắp xếp lại theo `user_id` trước khi
ghi, nên các file sau tối ưu có range liền kề, không chồng lấn (`[1,1851]`, `[1851,3696]`, …).
Stats trong log vì vậy loại được 54/55 file mà không cần đọc. Thời gian wall-clock dao động
theo máy (cache, CPU), nên rubric chấp nhận pruning ratio làm thước đo thay thế; ở đây đạt cả hai.
Delta-rs không gộp thành 1 file vì giới hạn target file size. Nếu gộp hết vào 1 file thì
không còn gì để prune.

## NB3 — Time travel, MERGE, RESTORE

| Tiêu chí | Kết quả |
|---|---|
| MERGE 100K dòng | 0.13 s; `num_source_rows=100000`, 50 000 update + 50 000 insert |
| History sau RESTORE | v0 WRITE, v1 WRITE, v2 MERGE, v3 WRITE (dữ liệu lỗi), **v4 RESTORE**; tổng 5 version |
| Dòng `score < 0` sau RESTORE | **0** |

**Giải thích.** v3 ghi 50 dòng lỗi. `RESTORE` về v2 không xóa v3 khỏi lịch sử mà tạo
**commit mới v4**: log ghi `remove` cho file lỗi và `add` lại các file của v2. Nhờ vậy vẫn
audit được sự cố (vẫn đọc được v3) trong khi version hiện tại sạch. `versionAsOf=0` vẫn trả
100 000 dòng. Time travel chỉ còn hiệu lực đến khi VACUUM xóa file cũ (xem NB6).

## NB4 — Medallion Bronze → Silver → Gold

| Tiêu chí | Kết quả |
|---|---|
| Ba lớp trên storage | `_lakehouse/bronze/llm_calls_raw`, `_lakehouse/silver/llm_calls`, `_lakehouse/gold/llm_daily_metrics` |
| Silver < Bronze | 200 000 → **190 052** (dedup bỏ 9 948 dòng trùng `request_id`) |
| Gold | 24 dòng = **8 ngày × 3 model** (haiku-4-5, sonnet-4-6, opus-4-7) |
| Chất lượng Gold | p50 ≤ p95 mọi dòng; `cost_usd` > 0; `error_rate` ∈ [0.042, 0.062] ([kiểm tra](evidence/nb04_gold_check.txt)) |

**Giải thích.** Bronze giữ nguyên payload `raw_json` để có thể replay. Silver parse JSON thành
cột có kiểu và dedup theo `request_id`; số dòng bị bỏ (9 948) khớp với số bản trùng do
generator cố ý tạo. Gold trả lời câu hỏi vận hành (latency, lỗi, chi phí theo ngày × model).
Ví dụ ngày 2026-04-02, opus có p50 3 004 ms so với haiku 565 ms, và chi phí cao hơn khoảng 6×
dù số token ít hơn, đúng với giá minh họa của lab. Notebook không assert đầy đủ yêu cầu Gold,
nên mình kiểm tra riêng trong `nb04_gold_check.txt`.

## NB5 — Iceberg + catalog

| Tiêu chí | Kết quả |
|---|---|
| Tạo qua catalog, spec `day(ts)` | `SqlCatalog` → `('lake','llm_events')`, spec `1000: ts_day: day(2)` |
| Hidden-partition pruning | Không lọc: 10 file; lọc một ngày **trên `ts`**: 1 file → **10×** (≥ 5×) |
| Metadata 3 tầng | metadata.json → 10 manifest list → 10 manifest → 10 data file; metadata 134.2 KB so với data 47.3 KB (**283.9%**) |
| Rename giữ field_id | `latency_ms` → `latency_millis` vẫn **field_id=4** |
| ≥ 2 partition spec | Spec ID đang dùng `[1, 2]`; đọc được đủ **5 500** dòng |

**Giải thích.** Filter viết trên `ts`, nhưng Iceberg lưu transform `day(ts)` trong spec nên
planner tự suy ra `ts_day` và loại 9/10 file. Người dùng không cần biết cột partition. Với
Hive-style, quên `WHERE dt=...` sẽ đọc cả 10 file; notebook ước tính ~$220/ngày ở 10K
query/ngày (512 MB/file, $5/TB). Tỷ lệ metadata 284% là do file quá nhỏ (10 dòng/file). Ở
512 MB/file tỷ lệ này chỉ khoảng 0.1%: small files tốn kém hai lần, ở cả data lẫn metadata.
Rename chỉ đổi tên trong schema vì cột được định danh bằng field ID, nên không file Parquet
nào bị viết lại. Partition evolution cho phép file cũ (spec 1) và file mới (spec 2) cùng tồn
tại, query vẫn đọc được tất cả.

## NB6 — Maintenance

| Job | Trước → Sau |
|---|---|
| 1. Compaction | 200 → **11 file (18×)**, 100 000 dòng giữ nguyên |
| 2. Clustering | Point query `user_id=12345`: 11/11 → **1/10 file**, skip **90%** |
| 3. Expiry | Delta VACUUM thu hồi **16.1 MB** (211 file tombstone); Iceberg 20 → **3 snapshot** |
| 4. Orphans | Tìm và xóa **3** Delta orphan (`part-9999x-crashed-writer`); quét **17** Iceberg manifest list bị bỏ lại (36.8 KB) |
| 5. Checkpoint | `00000000000000000099.checkpoint.parquet` + `_last_checkpoint` |

**Giải thích.**
- Compaction làm dung lượng data **tạm tăng** (10.1 → 16.1 MB) vì file mới được ghi trước,
  file cũ chỉ bị tombstone. Chỉ sau VACUUM dung lượng mới giảm xuống 6.2 MB.
- Clustering có hiệu quả vì stats min/max chỉ hữu ích khi các range không chồng lấn.
- **Hành vi đo được 1:** `deltalake` 1.6.6 VACUUM chỉ xóa file đã bị tombstone trong log. File
  do writer crash để lại chưa từng được commit, nên dry-run VACUUM không thấy (5 file trên đĩa
  không có trong log). Phải tự tính hiệu tập hợp "file trên đĩa − file trong log", có age guard
  để tránh xóa file của writer đang chạy.
- **Hành vi đo được 2:** `expire_snapshots` của PyIceberg 0.12.0 giảm snapshot 20 → 3 nhưng
  avro trên đĩa vẫn là 40 → 40, metadata bytes còn tăng nhẹ. Phải chạy thêm bước orphan sweep
  thì mới thu hồi được 36.8 KB. Job 3 và Job 4 phải đi cặp. Đây là hành vi của phiên bản
  thư viện trong lab, không phải kết luận chung cho mọi engine.
- Checkpoint giúp reader mới chỉ cần đọc 1 file Parquet cộng vài JSON phía sau, thay vì replay
  204 JSON.

## NB7 — Vectors và multimodal

| Tiêu chí | Kết quả |
|---|---|
| Random-access amplification | Lấy 1 frame (64 KB) từ bảng inline phải đọc cả row group 12.5 MB → **200×** (≥ 5×) |
| Projection scan | `GROUP BY topic` chỉ đọc 1.2 KB ở cả hai layout |
| int8 vs float32 trên đĩa | 2.6 MB → 451.9 KB, **5.8× nhỏ hơn** (≥ 3×) |
| recall@10 / topic fidelity | **0.904** / **1.000** (≥ 0.80 / ≥ 0.95) |
| SQL semantic search | `array_cosine_similarity` trong DuckDB; top-5 của `storage-note-00007` đều thuộc topic `storage` |
| Lifecycle bug | Xóa 8 doc của `user_042`: trong bảng **0** hit, external index cũ **8** hit |

**Giải thích.** Parquet đọc theo cột nên scan phân tích bỏ qua cột blob (projection pushdown
hoạt động tốt). Nhưng đơn vị đọc nhỏ nhất của một cột là **row group**: cả 200 frame nằm
trong 1 row group nên lấy 1 frame phải đọc 12.5 MB, khuếch đại 200×. Đó là lý do nên lưu
pointer cho truy cập ngẫu nhiên, hoặc dùng format như Lance. int8 giảm 4× theo lý thuyết; trên
đĩa đạt 5.8× vì int8 nén tốt hơn float32. recall exact-ID 0.904 thấp hơn topic fidelity 1.0 vì
các "miss" là hoán đổi giữa những hàng xóm gần tương đương, nên với RAG chất lượng gần như
không đổi. Lifecycle bug: index ngoài chỉ được sync một chiều bằng upsert nên không bao giờ
nhận lệnh delete. Change Data Feed phát ra 8 sự kiện delete để index đăng ký xử lý; cách tốt
nhất là giữ vector ngay trong row.

## NB8 — Agents và provenance

| Tiêu chí | Kết quả |
|---|---|
| Silver partition `agent_version` | `agent_version=policy-v2`, `agent_version=policy-v3`; 1 578 step |
| Gold 2 policy | v2: success 0.76, v3: 0.753 (mỗi policy 150 trajectory) |
| Pin version | Training pin `table_version=0` (1 578 step); sau khi thêm dữ liệu bảng ở v1 có 1 978 step; replay v0 = **1 578**, khớp |
| Lớp MCP mô phỏng | 5 lượt `list_tables` → **1** lần đọc catalog; destructive call trả `input_required`; task `working → completed` |
| Provenance | 4 bucket (`licensed`, `public_domain`, `synthetic`, `scraped_optout_checked`) + `UNCLASSIFIED` (334 dòng) là partition; tập train 1 666/2 000, loại 334 |

**Giải thích.** Pin `table_version` là một số nguyên nhưng giúp training tái lập được: dữ liệu
mới đến sau không làm lệch replay. Replay hiện chỉ so **số bước**, chưa so nội dung. Lớp MCP là
mô phỏng offline: cache đo trên `list_tables` (không phải `tools/list`), và cờ `confirmed` do
bên gọi truyền vào nên **không** phải ranh giới phân quyền thật. Bucket provenance là quy tắc
minh họa của lab: mapping gán CC-BY-4.0 vào `public_domain` là chưa chính xác vì CC-BY yêu cầu
ghi công, nên không dùng để kết luận pháp lý. Xóa `user_007` làm bảng lên v1 với 0 dòng còn lại,
nhưng v0 **vẫn chứa** dữ liệu đó cho đến khi retention/VACUUM xóa file cũ. Đây là mâu thuẫn
giữa time travel và quyền xóa dữ liệu.
