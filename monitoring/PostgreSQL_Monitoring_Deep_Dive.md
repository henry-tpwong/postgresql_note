# PostgreSQL 監控深度分析

> 本文合併自三篇 PostgreSQL 監控相關筆記，依由淺入深的閱讀順序編排：
>
> - **第一章「慢查詢追溯」**（實戰監控框架）：從 pg_stat_activity 監測慢查詢 → 採集 OS/DB 層級數據 → 內核自動記錄（auto_explain / log_lock_waits）→ 交叉分析定位瓶頸，建立一套完整的慢查詢事後追溯體系。
> - **第二章「track_commit_timestamp」**（內部機制深入）：深入 PG 內核的 commit timestamp 機制——儲存結構（SLRU）、啟用方式、實際用途（logical replication / snapshot too old / CDC）、效能影響與 production 取捨。
>
> 建議先掌握第一章的監控實務框架，再進入第二章理解內核機制如何支撐更高階的複製與審計場景。

---

# 一、 PostgreSQL 慢查詢追溯：事後還原當時狀態

> 來源：[digoal - 如何追溯 PostgreSQL 慢查询当时的状态 (2016-04-21)](https://github.com/digoal/blog/blob/master/201604/20160421_01.md)
>
> 更新於 2026-05-17，補充 PG 14 / 15 / 16 / 17 / 18 新增能力

---

## 1. 問題：慢查詢發生後如何追溯原因？

慢查詢的原因很多：I/O 等待、CPU 繁忙、執行計劃異常、lock 等待、網路延遲等。關鍵問題是：**慢查詢發生後，如何還原當時的系統狀態**，以便定位瓶頸？

框架如下：

1. 如何監測慢查詢
2. 監測到後需要採集哪些資訊
3. 資料庫內核層面能做什麼
4. 如何分析

---

## 2. 監測慢查詢：pg_stat_activity

```sql
SELECT
  datname, pid, leader_pid, usename, application_name,
  client_addr, client_port, backend_type,
  xact_start, query_start, state_change,
  wait_event_type, wait_event,
  state, backend_xid, backend_xmin,
  query_id, query,
  now() - xact_start AS xact_duration,
  now() - query_start AS query_duration
FROM pg_stat_activity
WHERE state <> 'idle'
  AND (backend_xid IS NOT NULL OR backend_xmin IS NOT NULL)
ORDER BY now() - xact_start;
```

| 欄位 | 含義 |
|------|------|
| `now() - xact_start` | transaction 已運行時間 |
| `now() - query_start` | 當前 query 已運行時間 |
| `pid` | backend process ID |
| `leader_pid` | parallel group leader PID（PG 15+，parallel worker 可追溯到主 process） |
| `wait_event_type` / `wait_event` | 當前等待事件類型與名稱（PG 9.6+ 取代舊 `waiting` boolean） |
| `backend_type` | process 類型（`client backend` / `autovacuum worker` / `walsender` 等） |
| `query_id` | PG 14+ 內建 query identifier，對應 `pg_stat_statements.queryid` |
| `waiting` | 是否正在等待 lock（PG 9.5 以下；9.6+ 改為 `wait_event_type` / `wait_event`） |

過濾條件 `state <> 'idle'` 加上 `backend_xid IS NOT NULL OR backend_xmin IS NOT NULL` 確保只抓仍在 active transaction 中的 session。

> 補充（Senior Dev）：`backend_xmin` 非 NULL 代表該 session 持有 snapshot 正在阻止 vacuum 清理 dead tuple，即使 query 本身是 idle 狀態。在排查慢查詢時，這類 session 可能不是「慢」而是「卡住別人」，應一起納入監控。PG 14+ 可用 `wait_event` 欄位直接區分 `Lock` / `LWLock` / `BufferPin` / `IO` 等等待類型，不必再猜測瓶頸方向。

> [PG 9.6+] `waiting` 欄位在 PG 9.6 被 `wait_event_type` 和 `wait_event` 取代，提供更精細的等待分類（如 `LWLock`、`Lock`、`IO`、`Client`、`CPU` 等）。
>
> [PG 13+] 新增 `leader_pid` 欄位，標識 parallel worker 所屬的 leader process，方便追蹤 parallel query 的等待鏈。
>
> [PG 14+] 新增 `query_id` 欄位（需啟用 `compute_query_id`），可直接將 `pg_stat_activity` 的 query 與 `pg_stat_statements` 的統計關聯。

---

## 3. 發現超閾值後，採集以下資訊（持續到 PID 結束）

### I. Process Stack Trace / Call Stack

```bash
# 現代 Linux（pstack 在部分發行版已移除，改用 gdb）
gdb -p $PID -batch -ex 'thread apply all bt'

# 或傳統 pstack（如可用）
pstack $PID

# 採集間隔自訂，如每秒一次
```

還原 slow query 卡在哪個 kernel / library call。需要安裝 debuginfo package 才能看到 function name 而非 hex address。透過 call stack 可以判斷 process 卡在哪個 function（I/O、lock、CPU 計算、network send/recv）。

### II. Lock Wait 記錄

參考德哥另一篇文章：[PostgreSQL 锁等待监控 珍藏级SQL - 谁堵塞了谁](https://github.com/digoal/blog/blob/master/201705/20170521_01.md)

透過 `pg_locks` + `pg_stat_activity` JOIN 查出 blocked / blocking chain。採集間隔自訂，直到 PID 結束。核心查詢思路：`pg_locks` JOIN `pg_stat_activity` 找出 blocked / blocking 關係。

> 補充（Senior Dev）：blocking chain 可能多於兩層（A 等 B，B 等 C）。如果只查一層 `JOIN` 可能遺漏根源。建議使用 recursive CTE 追蹤完整等待鏈，或直接用 `pg_blocking_pids(pid)`（PG 9.6+）一次取得完整 blocking tree。PG 16+ 結合 `pg_stat_activity.wait_event_type = 'Lock'` 可快速過濾。

> 補充（Senior Dev）：production 中 lock wait 是最常見的慢查詢原因之一（僅次於 bad plan）。關鍵 script 應同時擷取：
> - blocking chain（誰鎖了誰）
> - lock mode（`AccessShareLock` vs `AccessExclusiveLock`：前者通常是讀寫並發，後者是 DDL 或 `LOCK TABLE`）
> - `pg_locks.granted` 欄位（`false` = 正在等待）

### III. System / Process IO

```bash
iostat -x 1              # system-level IO，每秒
iotop -p $PID            # process-level IO，針對 PID
```

判斷瓶頸是 IOPS、throughput 還是 await（磁盤響應延遲）。採集間隔：每秒一次。

> 補充（Senior Dev）：PG 16+ 新增 **`pg_stat_io`** view，提供 cluster-wide I/O 統計（含 backend type / I/O object / context / read_time / write_time / fsync_time / extends），是資料庫層面的原生 I/O 透視。可取代部分 iostat 需求：

```sql
SELECT backend_type, object, context,
       reads, read_time, writes, write_time,
       extends, fsyncs, fsync_time, hits, evictions
FROM pg_stat_io
WHERE backend_type = 'client backend';
```

> PG 16+ 的 `pg_stat_io` 提供了更細粒度的 I/O context（`normal` / `vacuum` / `bulkread` / `bulkwrite`），可區分查詢產生的 I/O vs autovacuum 產生的 I/O，這在排查「慢查詢是否被 vacuum 拖慢」時很有用。

### IV. Network

```bash
sar -n DEV 1 1           # system network throughput
iptraf                   # per-connection network（需知道 client IP + port）
```

根據 `pg_stat_activity` 中的 `client_addr` 和 `client_port` 定位連線。

### V. CPU

```bash
top -p $PID              # per-process CPU, memory, state
```

採集間隔：每秒一次。

### VI. Wait Events

PG 9.6+ 的 `pg_stat_activity` 包含 `wait_event_type` + `wait_event`。可透過 snapshot 對比時間段內的 wait event 分佈，定位瓶頸類型（Lock / IO / LWLock / BufferPin / Extension / Activity / Timeout / IPC 等）。

PG 14+ `pg_stat_activity` 新增 `wait_start` 欄位（timestamp），可直接計算**已經等了多久**，不再需要推估：

```sql
SELECT pid, wait_event_type, wait_event,
       now() - wait_start AS wait_duration
FROM pg_stat_activity
WHERE wait_event IS NOT NULL;
```

原文亦提到可透過 HOOK 機制開發類似 `pg_stat_statements` 的 wait event 統計插件，持久化存儲供事後分析。PG 18 允許 extensions 透過 `WaitEventExtensionNew()` 註冊自訂 wait event，進一步擴展監控。

> 補充（Senior Dev）：PG 13 以後，可以搭配 `pg_wait_sampling` extension 來做更精確的 wait event sampling（類似 Oracle ASH）。它透過 background worker 定期取樣 wait event，存入 `pg_wait_sampling_profile` 等 view，不需要頻繁 poll `pg_stat_activity`。
>
> ```sql
> CREATE EXTENSION pg_wait_sampling;
> SELECT * FROM pg_wait_sampling_profile;
> ```

### VII. TOP SQL（pg_stat_statements）

對 `pg_stat_statements` 做 snapshot 比對，取得時間段內的：

- 總耗時最高的 SQL
- 執行次數最多的 SQL
- mean / stddev 耗時異常的 SQL
- 命中 shared buffer 比例異常低的 SQL
- 產生 WAL 最多的 SQL（`wal_bytes`，PG 13+）
- Plan time vs Execute time 分佈（`plans` / `total_plan_time` / `total_exec_time`，PG 13+）

```sql
SELECT queryid, query,
       calls,
       total_plan_time, total_exec_time,
       mean_exec_time, stddev_exec_time,
       shared_blks_hit, shared_blks_read,
       blk_read_time, blk_write_time,
       wal_records, wal_bytes
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

也可以透過 `pg_stat_statements` extension 做 snapshot diff：

```sql
CREATE EXTENSION pg_stat_statements;
-- snapshot 1
SELECT * FROM pg_stat_statements ORDER BY total_time DESC LIMIT 10;
-- wait N seconds
-- snapshot 2（diff = 時段內的 TOP SQL）
```

可以了解時段內的 TOP SQL 及其 `total_time`、`calls`、`shared_blks_read`、`temp_blks_read` 等開銷分佈。

> 補充（Senior Dev）：PG 13 起 `pg_stat_statements.track_planning = on` 可拆分 plan / execute 時間，對排查 prepare statement plan cache 失效或 generic plan 選錯很有用。PG 14+ `compute_query_id` 讓 `query_id` 成為內建功能（不再強制依賴 `pg_stat_statements`），搭配 `log_line_prefix` 的 `%Q` 可在 log 中直接關聯。

### VIII. Table / Index Stats & IO Stats

- `pg_stat_user_tables`：`seq_scan`, `seq_tup_read`, `idx_scan`, `idx_tup_fetch`, `n_tup_ins/upd/del`
- `pg_statio_user_tables`：`heap_blks_read`（物理讀塊數）
- `pg_statio_user_indexes`：`idx_blks_read`

對這些 view 做 snapshot 比對，可以發現哪張表/哪個 index 在被大量 physical read，或是全表掃描次數突然飆升。Snaphot diff 也可以觀察時段內哪些 table 被大量 seq scan、哪些 index 被頻繁 block read。

> 補充（Senior Dev）：PG 16+ 的 `pg_stat_io` 提供了更細粒度的 I/O context（`normal` / `vacuum` / `bulkread` / `bulkwrite`），可區分查詢產生的 I/O vs autovacuum 產生的 I/O，這在排查「慢查詢是否被 vacuum 拖慢」時很有用。

---

## 4. 資料庫內核層面自動記錄

### I. auto_explain：自動記錄慢 SQL 的 EXPLAIN

參考：[PostgreSQL 函数调试、诊断、优化 & auto_explain](https://github.com/digoal/blog/blob/master/201611/20161121_02.md)

```ini
# postgresql.conf（全域）
# 或 SET 語句（per-session，不需 restart）
shared_preload_libraries = 'auto_explain'      # 全域需 restart
# session_preload_libraries = 'auto_explain'   # PG 10+，per-session 不需 restart

auto_explain.log_min_duration = '1s'           # 超過此閾值才記錄
auto_explain.log_analyze = on                   # EXPLAIN ANALYZE（含 actual time）
auto_explain.log_buffers = on                   # buffer hit/read 統計
auto_explain.log_timing = on                    # per-node timing
auto_explain.log_verbose = on                   # extra plan info
auto_explain.log_nested_statements = on         # function 內 SQL 也記錄
auto_explain.log_settings = on                  # PG 12+，記錄當時 GUC 參數
auto_explain.log_parameter_max_length = 512     # PG 17+，記錄 bind parameter 值（截斷長度）
auto_explain.log_wal = on                       # PG 18+，記錄 WAL 產生量
auto_explain.log_format = json                  # PG 18+，可選 json 格式輸出
```

當 query 執行時間超過 `log_min_duration` 時，自動在 log 中輸出該 query 的 `EXPLAIN (ANALYZE, BUFFERS, TIMING, VERBOSE)` 結果，包括每個 plan node 的 actual time、buffer 用量。記錄超過閾值的 SQL 的 execution plan 及每個 node 的 actual time，事後可直接從 PostgreSQL log 看到完整的 EXPLAIN ANALYZE 輸出。

> 補充（Senior Dev）：
> - `log_nested_statements = on` 對 function / stored procedure 內的 SQL 也生效，排查深層 SQL 至關重要。
> - `log_analyze = on` 會讓被記錄的 SQL 實際執行 EXPLAIN ANALYZE overhead，不建議在極高 QPS 環境設全域。可用 `auto_explain.log_min_duration` 設較高閾值僅捕獲真正慢的 SQL。`auto_explain.log_analyze = on` 會對被記錄的 query 實際執行 `EXPLAIN ANALYZE`，這意味著 query 本身會被執行兩次（一次正常，一次 ANALYZE）。在高吞吐 OLTP 場景下，建議只對 `log_min_duration` 較高的 query 啟用 full analyze，或使用 sampling（PG 17+ `auto_explain.log_sample_rate`）來降低 overhead。
> - PG 10+ 支援 `session_preload_libraries = 'auto_explain'`，可在 per-session 層級啟用（`SET`），不需重啟 instance。
> - PG 12+ 的 `log_settings` 記錄 optimizer 相關 GUC（`enable_seqscan`, `work_mem`, `random_page_cost` 等），對還原 execution plan 選擇原因極有幫助。
> - PG 17+ 的 `log_parameter_max_length` 解決了 `auto_explain` 長期痛點：無法知道 $1, $2 的實際值。PG 18+ 的 `log_wal` 記錄該次 query 產生的 WAL 量。

參考：
- [PostgreSQL 函數調試、診斷、優化 & auto_explain](https://github.com/digoal/blog/blob/master/201611/20161121_02.md)
- [PostgreSQL 加載動態庫詳解](https://github.com/digoal/blog/blob/master/201603/20160316_01.md)

### II. log_lock_waits：記錄 Lock Wait 耗時

```ini
# postgresql.conf
log_lock_waits = on
deadlock_timeout = 1s
```

當一個 session 等待 lock 超過 `deadlock_timeout` 時，PostgreSQL 會在 log 中寫入等待資訊（包括 blocked / blocking PID、lock type、被等待的 SQL）。PG 14+ log 中同時會打印 `wait_event` 資訊。

> 補充（Senior Dev）：`deadlock_timeout` 的預設值是 1s，這是 deadlock detection 的檢查間隔。將它設為 1s 與 `log_lock_waits` 啟用搭配，可以在 lock wait > 1s 時自動記錄。注意：降低 `deadlock_timeout` 會增加 deadlock detector 的喚醒頻率（CPU 微幅上升），但對 multi-master / high concurrency 環境來說，1s 已足夠。

### III. IO Timing Trace

PostgreSQL 可開啟 `track_io_timing = on` 來記錄每次 block read/write 的耗時。這會產生額外的 `clock_gettime()` syscall overhead。啟用 IO timing 追蹤後，讓 `EXPLAIN (ANALYZE, BUFFERS)` 和 `pg_stat_statements` 中的 IO 時間與 CPU 時間分開統計。

```ini
track_io_timing = on
```

德哥另一篇文章有詳細討論：

[Linux 时钟精度 与 PostgreSQL auto_explain (explain timing 时钟开销估算)](https://github.com/digoal/blog/blob/master/201612/20161228_02.md)

> 補充（Senior Dev）：`track_io_timing = on` 在現代 Linux kernel（5.x+）和 SSD/NVMe 上的 overhead 已大幅降低（每次 I/O 約 0.5~1μs）。PG 16+ 的 `pg_stat_io` 即使不開啟此參數也能提供 I/O 呼叫次數（`reads` / `writes`），但精確的 `read_time` / `write_time` 仍需 `track_io_timing = on`。在 `EXPLAIN (ANALYZE, BUFFERS)` 輸出中可看到每個 node 的 `I/O Timings: read=X.XXX write=Y.YYY`。
>
> 注意：啟用 IO timing 會帶來微小的效能開銷（每次 I/O 操作前後讀取高精度時鐘）。在高頻率 small I/O 場景下（如 random index scan on NVMe），開銷可能達到 2-5%。參考：[Linux 時鐘精度與 PostgreSQL auto_explain](https://github.com/digoal/blog/blob/master/201612/20161228_02.md)

> [PG 13+] `track_io_timing` 預設從 `off` 改為 `on`，因為現代硬體的高精度時鐘開銷已降至可忽略。

### IV. 網絡耗時（內核層面缺失）

PG 內核輸出的 SQL 總時間包含 network 傳輸到 client 的時間，但沒有單獨統計 network 耗時。原文提到可透過 HACK 內核來實現拆分。

> 補充（Senior Dev）：截至 PG 18，network time 仍無原生拆分。實務上可對比 `log_duration`（server side）與 client side round-trip time 來間接估算。若使用 `libpq` 的 `trace(PQtrace)` 或 pgbouncer 的 `log_pooler_errors`，可在 middleware 層取得 network 耗時。

> [PG 14+] `pg_stat_statements` 新增 `wal_bytes` 欄位，可區分 WAL 寫入量；雖然仍無直接 network time，但可透過 client-side TCP 層級監控（`ss -tip` 或 `tcpdump`）來隔離網路延遲。

---

## 5. 分析流程總結

高層次分析框架：

```
慢查詢發生
    ↓
1. pg_stat_activity → query 執行了多久？wait event 是什麼？
    ↓
2. auto_explain log → 哪個 plan node 最慢？buffer 命中率如何？
    ↓
3. pg_stat_statements snapshot → 此時段哪些 SQL 是瓶頸？
    ↓
4. pg_locks + pg_stat_activity → 是否在等鎖？
    ↓
5. OS 層級採集（iostat / iotop / top / sar / pstack）
   → 是 I/O bound？CPU bound？network bound？
    ↓
6. pg_statio_* snapshot → 哪些 table/index 被大量物理讀？
    ↓
定位根因 → fix
```

詳細 drill-down 流程：

```
慢查詢觸發閾值
  → 記錄 pg_stat_activity snapshot（誰、什麼 SQL、wait_event、跑了多久）
  → 對 PID 啟動週期性採集：
      gdb stack / lock chain / iostat / iotop / top / sar
  → pg_stat_io snapshot（PG 16+，I/O context 細分）
  → auto_explain 自動記錄 EXPLAIN plan + per-node timing + GUC settings
  → log_lock_waits 記錄 lock 等待 + wait_event
  → 事後從 pg_stat_statements snapshot 對比：
      TOP SQL / plan-vs-execute 比例 / WAL 產生量
  → 從 pg_statio_* snapshot 對比 table/index physical read 變化
  → 交叉比對所有數據，依 wait_event_type 分類定位瓶頸類型：
      Lock → lock chain 分析
      IO → iostat + pg_stat_io + pg_statio_*
      LWLock / BufferPin → pstack / gdb
      CPU → top + pg_stat_statements JIT 欄位
      Activity → 檢查是否 idle-in-transaction 阻塞 vacuum
```

---

## 6. 現代化監測工具鏈補充

> 補充（Senior Dev）：原文（2016）的採集方式偏手動 script，現代 PG 生態中已有一整套工具可自動化：
>
> | 工具 | 用途 | 對應原文環節 |
> |------|------|-------------|
> | `pg_stat_statements` | TOP SQL、calls、IO time | 3.VII |
> | `auto_explain` | 慢查詢 plan 自動記錄 | 4.I |
> | `pg_wait_sampling` (PG 13+) | Wait event sampling（類似 Oracle ASH） | 3.VI |
> | `pg_stat_kcache` (PoWA) | 每個 query 的 OS 層 read/write 量 | 3.III / 3.VIII |
> | `pgBadger` | Log 分析，自動生成慢查詢報告（含 wait event、plan、timeline） | 全環節 |
> | `pg_qualstats` (PoWA) | 追蹤 predicate 使用頻率，輔助 index 設計 | 分析環節 |
> | `pg_activity` (CLI) | `top` 風格的即時 PG activity viewer | 3.V（即時替代 pstack/top） |
> | `pgsentinel` / `pg_top` | Sampling-based activity monitoring | 3.VI |
>
> Production 建議最小組合：
> ```ini
> shared_preload_libraries = 'pg_stat_statements, auto_explain'
> track_io_timing = on
> log_lock_waits = on
> ```
> 定期將 log 餵給 `pgBadger` 生成 HTML report，即可覆蓋原文 80% 的追溯需求。

---

## 7. 版本變遷速查

| 功能 | 引入版本 | 說明 |
|------|---------|------|
| `wait_event_type` / `wait_event` | PG 9.6 | 取代 `waiting` boolean |
| `pg_blocking_pids(pid)` | PG 9.6 | 一次取得完整 blocking tree |
| `session_preload_libraries` | PG 10 | per-session 載入 `auto_explain`，不需 restart |
| `pg_stat_activity.backend_type` | PG 10 | 區分 backend 類型（client / autovacuum / walsender 等） |
| `auto_explain.log_settings` | PG 12 | 記錄 optimizer GUC |
| `pg_stat_statements.track_planning` | PG 13 | 拆分 plan time / execute time |
| `pg_stat_statements.wal_bytes` | PG 13 | WAL 產生量統計 |
| `pg_wait_sampling` extension | PG 13 | Wait event sampling（類似 Oracle ASH） |
| `track_io_timing` 預設 on | PG 13 | 現代硬體的時鐘開銷已可忽略 |
| `wait_start` in `pg_stat_activity` | PG 14 | 精確計算已等待時間 |
| `compute_query_id` | PG 14 | 內建 query_id，不強制依賴 `pg_stat_statements` |
| `leader_pid` in `pg_stat_activity` | PG 15 | parallel worker 追溯主 process |
| `pg_stat_io` | PG 16 | 原生 I/O 統計（取代部分 iostat 需求） |
| `auto_explain.log_parameter_max_length` | PG 17 | 記錄 bind parameter 實際值 |
| `auto_explain.log_sample_rate` | PG 17 | Sampling 降低 full analyze overhead |
| `auto_explain.log_wal` | PG 18 | 記錄 query WAL 產生量 |
| `auto_explain.log_format = json` | PG 18 | JSON 格式輸出 |

> [PG 版本註] 原文基於 PG 9.5（2016）。核心排查框架在最新版本（PG 17+）仍然有效，主要增強：
> - PG 9.6+：`wait_event_type` / `wait_event` 取代 `waiting` 布林值
> - PG 10+：`pg_stat_activity.backend_type` 區分 backend 類型（client / autovacuum / walsender 等）
> - PG 13+：`pg_stat_activity.leader_pid`、`pg_wait_sampling` extension、`track_io_timing` 預設 on
> - PG 14+：`pg_stat_activity.query_id`、`pg_stat_statements.wal_bytes`
> - PG 16+：`pg_stat_io` 系統 view（取代 `pg_statio_*` 的手動 snapshot，提供 cluster-wide I/O 統計）
>
> 特別注意：`pg_stat_io`（PG 16+）是一次重大改進——它提供了 cluster-level 的 I/O 統計（reads、writes、extends、fsyncs、hits），按 backend type 和 context 分類，無需再對 `pg_statio_*` 做手工 snapshot diff。

---

## 8. 參考與源碼

1. [PostgreSQL 锁等待监控 珍藏级SQL - 谁堵塞了谁](https://github.com/digoal/blog/blob/master/201705/20170521_01.md)
2. [PostgreSQL 函数调试、诊断、优化 & auto_explain](https://github.com/digoal/blog/blob/master/201611/20161121_02.md)
3. [PostgreSQL 加載動態庫詳解](https://github.com/digoal/blog/blob/master/201603/20160316_01.md)
4. [Linux 时钟精度 与 PostgreSQL auto_explain (explain timing 时钟开销估算)](https://github.com/digoal/blog/blob/master/201612/20161228_02.md)
5. [如何生成和阅读EnterpriseDB (PPAS)诊断报告](https://github.com/digoal/blog/blob/master/201606/20160628_01.md)
6. `pg_stat_activity` — `src/backend/postmaster/pgstat.c`
7. `pg_stat_statements` — `contrib/pg_stat_statements/`
8. `pg_wait_sampling` — `contrib/pg_wait_sampling/`

---

# 二、 PostgreSQL track_commit_timestamp：PG 17 視角

> 來源：[digoal - PostgreSQL 9.5 new feature - record transaction commit timestamp (2015-04-09)](https://github.com/digoal/blog/blob/master/201504/20150409_03.md)
>
> 本文基於原始 PG 9.5 文章，以 PG 17 (2026) 為基準更新：補足實際用途、效能影響、版本演進。

---

## 1. 功能概述

`track_commit_timestamp` 自 PG 9.5 引入，每筆 transaction commit 時記錄其 timestamp。儲存在 PGDATA 下 `pg_commit_ts/` 目錄，結構類似 `pg_xact`（原 `pg_clog`），以 SLRU (Simple Least Recently Used) 機制管理。

### I. 儲存結構

- 每個 page 大小 = `BLCKSZ`（預設 8KB）
- 每筆 transaction 佔 12 bytes：`TimestampTz` (8 bytes) + `CommitTsNodeId` (4 bytes)

```c
// src/backend/access/transam/commit_ts.c
typedef struct CommitTimestampEntry
{
    TimestampTz   time;
    CommitTsNodeId nodeid;
} CommitTimestampEntry;

#define SizeOfCommitTimestampEntry \
    (offsetof(CommitTimestampEntry, nodeid) + sizeof(CommitTsNodeId))

#define COMMIT_TS_XACTS_PER_PAGE \
    (BLCKSZ / SizeOfCommitTimestampEntry)
```

PGDATA 目錄結構：

```
$ cd $PGDATA
drwx------  pg_commit_ts/
```

Transaction ID 是 32-bit 會 wraparound，因此 `pg_commit_ts` 的 page / segment 編號也會對應 wraparound。截斷邏輯見 `TruncateCommitTs` → `CommitTsPagePrecedes`。

---

## 2. 啟用與確認

```ini
# postgresql.conf
track_commit_timestamp = on
```

**必須 restart**（此參數為 `postmaster` 級別，無法 reload）：

```bash
pg_ctl restart -m fast
```

確認狀態：

```bash
pg_controldata | grep track
# Current track_commit_timestamp setting: on
```

`pg_resetxlog`（PG 10+ 改名 `pg_resetwal`）重置 WAL 時需指定安全範圍：

```bash
# -c xid,xid 指定最舊 / 最新可查 commit timestamp 的 xid 範圍
# 可從 pg_commit_ts/ 目錄中最小的 file name (hex) 推算
pg_resetwal -c 0x00000100,0x0000FF00
```

> 補充（Senior Dev）：如果 `track_commit_timestamp = off` 時資料庫曾運行過，之後再開啟，中間缺失的 transaction commit timestamp 會是 NULL。`pg_xact_commit_timestamp(xid)` 對這些 xid 回傳 NULL。

---

## 3. 實際用途（PG 9.5 → PG 17 演進）

### I. 2015 年（PG 9.5，原文時期）

當時作者推測與 snapshot too old、logical replication 有關，但尚未明確。

### II. 2026 年（PG 17）

**1. 查詢 commit timestamp：`pg_xact_commit_timestamp(xid)`**

```sql
SELECT pg_xact_commit_timestamp(txid_current());
-- 2026-05-17 15:30:45.123456+08

-- 查詢特定 transaction 的 commit time
SELECT xmin, pg_xact_commit_timestamp(xmin)
FROM some_table WHERE id = 1;
```

**2. Logical Replication（PG 10+）**

Logical decoding 內部依賴 commit timestamp 來決定 transaction 的 ordering 與可見性。若未開啟 `track_commit_timestamp`，logical replication 仍可運作，但某些 replication slot 行為（如 `pg_logical_slot_get_changes` 的 `upto_lsn` / `upto_nchanges`）在跨 node consistency 場景會有限制。

**3. Snapshot too old（PG 9.6+）**

`old_snapshot_threshold` 依賴 commit timestamp 來判斷哪些 row version 過期可回收。未開啟 `track_commit_timestamp` 時此功能無法使用。

**4. 審計 / CDC 場景**

可直接透過 xmin / xmax + `pg_xact_commit_timestamp()` 還原 row-level 的變更時間線，無需額外 `updated_at` column。

**5. Extension 生態**

- `pg_ivm`（Incremental View Maintenance）：依賴 commit timestamp 追蹤增量變更
- 部分 CDC tool（如 Debezium PG connector）在 snapshot 模式下查詢 commit timestamp 做 watermark

| PG 版本 | 相關演進 |
|---------|---------|
| 9.5 | 引入 `track_commit_timestamp`、`pg_xact_commit_timestamp()` |
| 9.6 | `old_snapshot_threshold` 依賴 commit timestamp |
| 10 | Logical replication 正式 GA，重命名 `pg_resetxlog` → `pg_resetwal` |
| 14 | SLRU 效能改進，`pg_commit_ts` lookup 更快 |
| 15 | `pg_stat_reset_slru()` 可監控 `pg_commit_ts` SLRU 命中率 |

---

## 4. 效能影響與 Production 考量

> 補充（Senior Dev）：

| 面向 | 影響 |
|------|------|
| **寫入效能** | 每次 commit 多一次 SLRU write（~12 bytes），OLTP 場景 overhead 約 1-3%（視 workload，write-heavy 時更明顯） |
| **儲存空間** | 每百萬 transaction 約 12MB；需關注 SLRU truncation 是否及時（與 `vacuum_freeze_min_age` 等參數相關） |
| **讀取效能** | `pg_xact_commit_timestamp(xid)` 查詢走 SLRU buffer，hit 時幾乎零成本，miss 時觸發 page read |
| **Replication** | Logical replication 場景建議開啟，避免 corner case |

**建議：**

- **OLTP 核心庫**：若不需要審計 / logical replication / snapshot too old，關閉可節省 1-3% write overhead
- **需要 logical replication**：開啟，這是 PG 內部依賴
- **需要 row-level 時間追蹤但不想開全域**：使用 `updated_at timestamp DEFAULT now()` column，別開 `track_commit_timestamp`（更輕量、更可控）

---

## 5. 原始碼參考（2015 年原文 + 17 對應）

| 檔案路徑 | 說明 |
|---------|------|
| `src/backend/access/transam/commit_ts.c` | Commit timestamp 核心模組（PG 17 仍同路徑） |
| `src/backend/access/rmgrdesc/committsdesc.c` | WAL record descriptor for commit_ts |
| `man pg_resetwal` | PG 10+ 改名（原 `pg_resetxlog`） |
