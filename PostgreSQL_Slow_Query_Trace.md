# PostgreSQL 慢查詢追溯：事後還原當時狀態

> 來源：[digoal - 如何追溯 PostgreSQL 慢查询当时的状态 (2016-04-21)](https://github.com/digoal/blog/blob/master/201604/20160421_01.md)
>
> 更新於 2026-05-17，補充 PG 14 / 15 / 16 / 17 / 18 新增能力

---

## 監測與數據採集體系

### 1. 監測慢查詢：pg_stat_activity

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

過濾條件 `state <> 'idle'` 加上 `backend_xid IS NOT NULL OR backend_xmin IS NOT NULL` 確保只抓仍在 active transaction 中的 session。

> 補充（Senior Dev）：`backend_xmin` 非 NULL 代表該 session 持有 snapshot 正在阻止 vacuum 清理 dead tuple，即使 query 本身是 idle 狀態。在排查慢查詢時，這類 session 可能不是「慢」而是「卡住別人」，應一起納入監控。PG 14+ 可用 `wait_event` 欄位直接區分 `Lock` / `LWLock` / `BufferPin` / `IO` 等等待類型，不必再猜測瓶頸方向。

---

### 2. 發現超閾值後，採集以下資訊（持續到 PID 結束）

#### 2.1 Process Stack Trace

```bash
# 現代 Linux（pstack 在部分發行版已移除，改用 gdb）
gdb -p $PID -batch -ex 'thread apply all bt'

# 採集間隔自訂，如每秒一次
```

還原 slow query 卡在哪個 kernel / library call。需要安裝 debuginfo package 才能看到 function name 而非 hex address。

#### 2.2 Lock Wait 記錄

參考德哥另一篇文章：[PostgreSQL 锁等待监控 珍藏级SQL - 谁堵塞了谁](https://github.com/digoal/blog/blob/master/201705/20170521_01.md)

透過 `pg_locks` + `pg_stat_activity` JOIN 查出 blocked / blocking chain。採集間隔自訂，直到 PID 結束。

> 補充（Senior Dev）：blocking chain 可能多於兩層（A 等 B，B 等 C）。如果只查一層 `JOIN` 可能遺漏根源。建議使用 recursive CTE 追蹤完整等待鏈，或直接用 `pg_blocking_pids(pid)`（PG 9.6+）一次取得完整 blocking tree。PG 16+ 結合 `pg_stat_activity.wait_event_type = 'Lock'` 可快速過濾。

#### 2.3 System / Process IO

```bash
iostat -x 1              # system-level IO，每秒
iotop -p $PID            # process-level IO，針對 PID
```

判斷瓶頸是 IOPS、throughput 還是 await（磁盤響應延遲）。

> 補充（Senior Dev）：PG 16+ 新增 **`pg_stat_io`** view，提供 cluster-wide I/O 統計（含 backend type / I/O object / context / read_time / write_time / fsync_time / extends），是資料庫層面的原生 I/O 透視。可取代部分 iostat 需求：

```sql
SELECT backend_type, object, context,
       reads, read_time, writes, write_time,
       extends, fsyncs, fsync_time, hits, evictions
FROM pg_stat_io
WHERE backend_type = 'client backend';
```

#### 2.4 Network

```bash
sar -n DEV 1 1           # system network throughput
iptraf                   # per-connection network（需知道 client IP + port）
```

#### 2.5 CPU

```bash
top -p $PID              # per-process CPU, memory, state
```

#### 2.6 Wait Events

PG 9.6+ 的 `pg_stat_activity` 包含 `wait_event_type` + `wait_event`。可透過 snapshot 對比時間段內的 wait event 分佈，定位瓶頸類型（Lock / IO / LWLock / BufferPin / Extension / Activity / Timeout / IPC 等）。

PG 14+ `pg_stat_activity` 新增 `wait_start` 欄位（timestamp），可直接計算**已經等了多久**，不再需要推估：

```sql
SELECT pid, wait_event_type, wait_event,
       now() - wait_start AS wait_duration
FROM pg_stat_activity
WHERE wait_event IS NOT NULL;
```

原文亦提到可透過 HOOK 機制開發類似 `pg_stat_statements` 的 wait event 統計插件，持久化存儲供事後分析。PG 18 允許 extensions 透過 `WaitEventExtensionNew()` 註冊自訂 wait event，進一步擴展監控。

#### 2.7 TOP SQL（pg_stat_statements）

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

> 補充（Senior Dev）：PG 13 起 `pg_stat_statements.track_planning = on` 可拆分 plan / execute 時間，對排查 prepare statement plan cache 失效或 generic plan 選錯很有用。PG 14+ `compute_query_id` 讓 `query_id` 成為內建功能（不再強制依賴 `pg_stat_statements`），搭配 `log_line_prefix` 的 `%Q` 可在 log 中直接關聯。

#### 2.8 Table / Index Stats & IO Stats

- `pg_stat_user_tables`：`seq_scan`, `seq_tup_read`, `idx_scan`, `idx_tup_fetch`, `n_tup_ins/upd/del`
- `pg_statio_user_tables`：`heap_blks_read`（物理讀塊數）
- `pg_statio_user_indexes`：`idx_blks_read`

對這些 view 做 snapshot 比對，可以發現哪張表/哪個 index 在被大量 physical read，或是全表掃描次數突然飆升。

> 補充（Senior Dev）：PG 16+ 的 `pg_stat_io` 提供了更細粒度的 I/O context（`normal` / `vacuum` / `bulkread` / `bulkwrite`），可區分查詢產生的 I/O vs autovacuum 產生的 I/O，這在排查「慢查詢是否被 vacuum 拖慢」時很有用。

---

### 3. 資料庫內核層面自動記錄

#### 3.1 auto_explain：自動記錄慢 SQL 的 EXPLAIN

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
auto_explain.log_nested_statements = on         # function 內 SQL 也記錄
auto_explain.log_settings = on                  # PG 12+，記錄當時 GUC 參數
auto_explain.log_parameter_max_length = 512     # PG 17+，記錄 bind parameter 值（截斷長度）
auto_explain.log_wal = on                       # PG 18+，記錄 WAL 產生量
auto_explain.log_format = json                  # PG 18+，可選 json 格式輸出
```

記錄超過閾值的 SQL 的 execution plan 及每個 node 的 actual time，事後可直接從 PostgreSQL log 看到完整的 EXPLAIN ANALYZE 輸出。

> 補充（Senior Dev）：
> - `log_nested_statements = on` 對 function / stored procedure 內的 SQL 也生效，排查深層 SQL 至關重要。
> - `log_analyze = on` 會讓被記錄的 SQL 實際執行 EXPLAIN ANALYZE overhead，不建議在極高 QPS 環境設全域。可用 `auto_explain.log_min_duration` 設較高閾值僅捕獲真正慢的 SQL。
> - PG 10+ 支援 `session_preload_libraries = 'auto_explain'`，可在 per-session 層級啟用（`SET`），不需重啟 instance。
> - PG 12+ 的 `log_settings` 記錄 optimizer 相關 GUC（`enable_seqscan`, `work_mem`, `random_page_cost` 等），對還原 execution plan 選擇原因極有幫助。
> - PG 17+ 的 `log_parameter_max_length` 解決了 `auto_explain` 長期痛點：無法知道 $1, $2 的實際值。PG 18+ 的 `log_wal` 記錄該次 query 產生的 WAL 量。

#### 3.2 log_lock_waits：記錄 Lock Wait 耗時

```ini
# postgresql.conf
log_lock_waits = on
deadlock_timeout = 1s
```

當一個 session 等待 lock 超過 `deadlock_timeout` 時，PostgreSQL 會在 log 中寫入等待資訊（包括 blocked / blocking PID、lock type、被等待的 SQL）。PG 14+ log 中同時會打印 `wait_event` 資訊。

#### 3.3 IO Timing Trace

PostgreSQL 可開啟 `track_io_timing = on` 來記錄每次 block read/write 的耗時。這會產生額外的 `clock_gettime()` syscall overhead。德哥另一篇文章有詳細討論：

[Linux 时钟精度 与 PostgreSQL auto_explain (explain timing 时钟开销估算)](https://github.com/digoal/blog/blob/master/201612/20161228_02.md)

> 補充（Senior Dev）：`track_io_timing = on` 在現代 Linux kernel（5.x+）和 SSD/NVMe 上的 overhead 已大幅降低（每次 I/O 約 0.5~1μs）。PG 16+ 的 `pg_stat_io` 即使不開啟此參數也能提供 I/O 呼叫次數（`reads` / `writes`），但精確的 `read_time` / `write_time` 仍需 `track_io_timing = on`。在 `EXPLAIN (ANALYZE, BUFFERS)` 輸出中可看到每個 node 的 `I/O Timings: read=X.XXX write=Y.YYY`。

#### 3.4 網絡耗時（內核層面缺失）

PG 內核輸出的 SQL 總時間包含 network 傳輸到 client 的時間，但沒有單獨統計 network 耗時。原文提到可透過 HACK 內核來實現拆分。

> 補充（Senior Dev）：截至 PG 18，network time 仍無原生拆分。實務上可對比 `log_duration`（server side）與 client side round-trip time 來間接估算。若使用 `libpq` 的 `trace(PQtrace)` 或 pgbouncer 的 `log_pooler_errors`，可在 middleware 層取得 network 耗時。

---

## 分析流程總結

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

## 版本變遷速查

| 功能 | 引入版本 | 說明 |
|------|---------|------|
| `wait_event_type` / `wait_event` | PG 9.6 | 取代 `waiting` boolean |
| `pg_blocking_pids(pid)` | PG 9.6 | 一次取得完整 blocking tree |
| `session_preload_libraries` | PG 10 | per-session 載入 `auto_explain`，不需 restart |
| `auto_explain.log_settings` | PG 12 | 記錄 optimizer GUC |
| `pg_stat_statements.track_planning` | PG 13 | 拆分 plan time / execute time |
| `pg_stat_statements.wal_bytes` | PG 13 | WAL 產生量統計 |
| `wait_start` in `pg_stat_activity` | PG 14 | 精確計算已等待時間 |
| `compute_query_id` | PG 14 | 內建 query_id，不強制依賴 `pg_stat_statements` |
| `leader_pid` in `pg_stat_activity` | PG 15 | parallel worker 追溯主 process |
| `pg_stat_io` | PG 16 | 原生 I/O 統計（取代部分 iostat 需求） |
| `auto_explain.log_parameter_max_length` | PG 17 | 記錄 bind parameter 實際值 |
| `auto_explain.log_wal` | PG 18 | 記錄 query WAL 產生量 |
| `auto_explain.log_format = json` | PG 18 | JSON 格式輸出 |

## 參考

- [如何生成和阅读EnterpriseDB (PPAS)诊断报告](https://github.com/digoal/blog/blob/master/201606/20160628_01.md)
