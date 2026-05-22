# 追溯 PostgreSQL 慢查詢當時的狀態

> 來源：[digoal - 如何追溯 PostgreSQL 慢查询当时的状态 (2016-04-21)](https://github.com/digoal/blog/blob/master/201604/20160421_01.md)

---

## 問題：慢查詢發生後如何追溯原因？

慢查詢的原因很多：I/O 等待、CPU 繁忙、執行計劃異常、lock 等待、網路延遲等。關鍵問題是：**慢查詢發生後，如何還原當時的系統狀態**，以便定位瓶頸？

框架如下：

1. 如何監測慢查詢
2. 監測到後需要採集哪些資訊
3. 資料庫內核層面能做什麼
4. 如何分析

---

## 1. 監測慢查詢

透過 `pg_stat_activity` 找出當前執行超過閾值的 query：

```sql
SELECT
  datname, pid, usename, application_name, client_addr, client_port,
  xact_start, query_start, state_change, waiting, state, backend_xid,
  backend_xmin, query, xact_start, now() - xact_start
FROM pg_stat_activity
WHERE state <> 'idle'
  AND (backend_xid IS NOT NULL OR backend_xmin IS NOT NULL)
ORDER BY now() - xact_start;
```

| Column | 含義 |
|--------|------|
| `now() - xact_start` | transaction 截至當前已運行時間 |
| `now() - query_start` | query 截至當前已運行時間 |
| `pid` | backend process ID |
| `waiting` | 是否正在等待 lock（PG 9.5 以下；9.6+ 改為 `wait_event_type` / `wait_event`） |

> [PG 9.6+] `waiting` 欄位在 PG 9.6 被 `wait_event_type` 和 `wait_event` 取代，提供更精細的等待分類（如 `LWLock`、`Lock`、`IO`、`Client`、`CPU` 等）。
>
> [PG 13+] 新增 `leader_pid` 欄位，標識 parallel worker 所屬的 leader process，方便追蹤 parallel query 的等待鏈。
>
> [PG 14+] 新增 `query_id` 欄位（需啟用 `compute_query_id`），可直接將 `pg_stat_activity` 的 query 與 `pg_stat_statements` 的統計關聯。

---

## 2. 採集哪些資訊

當發現 query 執行時間超過設定閾值後，針對該 PID 持續採集以下資訊，直到 query 結束：

### 2.1 Process Call Stack — pstack

```bash
pstack $PID
```

採集間隔：例如每秒一次，直到 PID 執行結束。可以透過 call stack 判斷 process 卡在哪個 function（I/O、lock、CPU 計算、network send/recv）。

### 2.2 Database Lock Wait

採集間隔：每秒一次，記錄當前 lock wait chain。

參考：[PostgreSQL 鎖等待監控 — 誰堵塞了誰](https://github.com/digoal/blog/blob/master/201705/20170521_01.md)

核心查詢思路：`pg_locks` JOIN `pg_stat_activity` 找出 blocked / blocking 關係。

> 補充（Senior Dev）：production 中 lock wait 是最常見的慢查詢原因之一（僅次於 bad plan）。關鍵 script 應同時擷取：
> - blocking chain（誰鎖了誰）
> - lock mode（`AccessShareLock` vs `AccessExclusiveLock`：前者通常是讀寫並發，後者是 DDL 或 `LOCK TABLE`）
> - `pg_locks.granted` 欄位（`false` = 正在等待）

### 2.3 I/O 情況

```bash
# 整機 block device I/O
iostat -x 1

# 特定 process I/O
iotop -p $PID
```

採集間隔：每秒一次。

### 2.4 Network 情況

```bash
# 整機網路
sar -n DEV 1 1

# 特定 process / connection 的網路流量
iptraf
```

根據 `pg_stat_activity` 中的 `client_addr` 和 `client_port` 定位連線。

### 2.5 CPU 使用情況

```bash
top -p $PID
```

採集間隔：每秒一次。

### 2.6 資料庫 Wait Event 監測

`pg_stat_activity` 的 `wait_event` 欄位（PG 9.6+）記錄了 process 當前等待的具體事件。可以透過 snaphot `pg_stat_activity` 來統計時段內的 wait event 分佈。

> 補充（Senior Dev）：PG 13 以後，可以搭配 `pg_wait_sampling` extension 來做更精確的 wait event sampling（類似 Oracle ASH）。它透過 background worker 定期取樣 wait event，存入 `pg_wait_sampling_profile` 等 view，不需要頻繁 poll `pg_stat_activity`。
>
> ```sql
> CREATE EXTENSION pg_wait_sampling;
> SELECT * FROM pg_wait_sampling_profile;
> ```

### 2.7 TOP SQL 監測

透過 `pg_stat_statements` extension 做 snaphot diff：

```sql
CREATE EXTENSION pg_stat_statements;
-- snaphot 1
SELECT * FROM pg_stat_statements ORDER BY total_time DESC LIMIT 10;
-- wait N seconds
-- snaphot 2（diff = 時段內的 TOP SQL）
```

可以了解時段內的 TOP SQL 及其 `total_time`、`calls`、`shared_blks_read`、`temp_blks_read` 等開銷分佈。

### 2.8 表與索引的 stats / statsio 監測

```sql
-- 全表掃描次數、FETCH / GET TUPLE
SELECT * FROM pg_stat_user_tables;

-- 表層級物理讀區塊數
SELECT * FROM pg_statio_user_tables;

-- Index 層級物理讀區塊數
SELECT * FROM pg_statio_user_indexes;
```

Snaphot diff 可以觀察時段內哪些 table 被大量 seq scan、哪些 index 被頻繁 block read。

---

## 3. 資料庫內核層面能做什麼

### 3.1 auto_explain — 自動記錄慢查詢的 EXPLAIN 輸出

配置 `auto_explain`（shared_preload_libraries）：

```ini
# postgresql.conf
shared_preload_libraries = 'auto_explain'
auto_explain.log_min_duration = '1s'
auto_explain.log_analyze = on
auto_explain.log_buffers = on
auto_explain.log_timing = on
auto_explain.log_verbose = on
auto_explain.log_nested_statements = on
```

當 query 執行時間超過 `log_min_duration` 時，自動在 log 中輸出該 query 的 `EXPLAIN (ANALYZE, BUFFERS, TIMING, VERBOSE)` 結果，包括每個 plan node 的 actual time、buffer 用量。

參考：
- [PostgreSQL 函數調試、診斷、優化 & auto_explain](https://github.com/digoal/blog/blob/master/201611/20161121_02.md)
- [PostgreSQL 加載動態庫詳解](https://github.com/digoal/blog/blob/master/201603/20160316_01.md)

> 補充（Senior Dev）：`auto_explain.log_analyze = on` 會對被記錄的 query 實際執行 `EXPLAIN ANALYZE`，這意味著 query 本身會被執行兩次（一次正常，一次 ANALYZE）。在高吞吐 OLTP 場景下，建議只對 `log_min_duration` 較高的 query 啟用 full analyze，或使用 sampling（PG 17+ `auto_explain.log_sample_rate`）來降低 overhead。

### 3.2 log_lock_waits — 自動記錄 Lock Wait 耗時

```ini
log_lock_waits = on
deadlock_timeout = 1s
```

當 session 等待 lock 超過 `deadlock_timeout` 時，會在 log 中記錄等待資訊（waiting query、blocking query、lock mode、等待時間）。

> 補充（Senior Dev）：`deadlock_timeout` 的預設值是 1s，這是 deadlock detection 的檢查間隔。將它設為 1s 與 `log_lock_waits` 啟用搭配，可以在 lock wait > 1s 時自動記錄。注意：降低 `deadlock_timeout` 會增加 deadlock detector 的喚醒頻率（CPU 微幅上升），但對 multi-master / high concurrency 環境來說，1s 已足夠。

### 3.3 IO Timing Trace

啟用 IO timing 追蹤（`track_io_timing = on`），讓 `EXPLAIN (ANALYZE, BUFFERS)` 和 `pg_stat_statements` 中的 IO 時間與 CPU 時間分開統計。

```ini
track_io_timing = on
```

> 注意：啟用 IO timing 會帶來微小的效能開銷（每次 I/O 操作前後讀取高精度時鐘）。在高頻率 small I/O 場景下（如 random index scan on NVMe），開銷可能達到 2-5%。參考：[Linux 時鐘精度與 PostgreSQL auto_explain](https://github.com/digoal/blog/blob/master/201612/20161228_02.md)

> [PG 13+] `track_io_timing` 預設從 `off` 改為 `on`，因為現代硬體的高精度時鐘開銷已降至可忽略。

### 3.4 Network 傳輸時間

原文指出：PG 內核目前輸出的 SQL 時間包含了資料傳輸到 client 的時間，但網路傳輸的時間沒有單獨統計。這需要透過 hack kernel 來實現。

> [PG 14+] `pg_stat_statements` 新增 `wal_bytes` 欄位，可區分 WAL 寫入量；雖然仍無直接 network time，但可透過 client-side TCP 層級監控（`ss -tip` 或 `tcpdump`）來隔離網路延遲。

---

## 4. 如何分析

有了以上資訊後，可以建立一個完整的慢查詢分析框架：

```
慢查詢發生
    ↓
1. pg_stat_activity → query 執行了多久？wait event 是什麼？
    ↓
2. auto_explain log → 哪個 plan node 最慢？buffer 命中率如何？
    ↓
3. pg_stat_statements snaphot → 此時段哪些 SQL 是瓶頸？
    ↓
4. pg_locks + pg_stat_activity → 是否在等鎖？
    ↓
5. OS 層級採集（iostat / iotop / top / sar / pstack）
   → 是 I/O bound？CPU bound？network bound？
    ↓
6. pg_statio_* snaphot → 哪些 table/index 被大量物理讀？
    ↓
定位根因 → fix
```

---

## 現代化監測工具鏈補充

> 補充（Senior Dev）：原文（2016）的採集方式偏手動 script，現代 PG 生態中已有一整套工具可自動化：
>
> | 工具 | 用途 | 對應原文環節 |
> |------|------|-------------|
> | `pg_stat_statements` | TOP SQL、calls、IO time | 2.7 |
> | `auto_explain` | 慢查詢 plan 自動記錄 | 3.1 |
> | `pg_wait_sampling` (PG 13+) | Wait event sampling（類似 Oracle ASH） | 2.6 |
> | `pg_stat_kcache` (PoWA) | 每個 query 的 OS 層 read/write 量 | 2.3 / 2.8 |
> | `pgBadger` | Log 分析，自動生成慢查詢報告（含 wait event、plan、timeline） | 全環節 |
> | `pg_qualstats` (PoWA) | 追蹤 predicate 使用頻率，輔助 index 設計 | 分析環節 |
> | `pg_activity` (CLI) | `top` 風格的即時 PG activity viewer | 2.5（即時替代 pstack/top） |
> | `pgsentinel` / `pg_top` | Sampling-based activity monitoring | 2.6 |
>
> Production 建議最小組合：
> ```ini
> shared_preload_libraries = 'pg_stat_statements, auto_explain'
> track_io_timing = on
> log_lock_waits = on
> ```
> 定期將 log 餵給 `pgBadger` 生成 HTML report，即可覆蓋原文 80% 的追溯需求。

---

## 源碼與參考

1. [PostgreSQL 鎖等待監控 — 誰堵塞了誰](https://github.com/digoal/blog/blob/master/201705/20170521_01.md)
2. [PostgreSQL 函數調試、診斷、優化 & auto_explain](https://github.com/digoal/blog/blob/master/201611/20161121_02.md)
3. [PostgreSQL 加載動態庫詳解](https://github.com/digoal/blog/blob/master/201603/20160316_01.md)
4. [Linux 時鐘精度與 PostgreSQL auto_explain](https://github.com/digoal/blog/blob/master/201612/20161228_02.md)
5. [如何生成和閱讀 EnterpriseDB (PPAS) 診斷報告](https://github.com/digoal/blog/blob/master/201606/20160628_01.md)
6. `pg_stat_activity` — `src/backend/postmaster/pgstat.c`
7. `pg_stat_statements` — `contrib/pg_stat_statements/`
8. `pg_wait_sampling` — `contrib/pg_wait_sampling/`

> [PG 版本註] 原文基於 PG 9.5（2016）。核心排查框架在最新版本（PG 17+）仍然有效，主要增強：
> - PG 9.6+：`wait_event_type` / `wait_event` 取代 `waiting` 布林值
> - PG 10+：`pg_stat_activity.backend_type` 區分 backend 類型（client / autovacuum / walsender 等）
> - PG 13+：`pg_stat_activity.leader_pid`、`pg_wait_sampling` extension、`track_io_timing` 預設 on
> - PG 14+：`pg_stat_activity.query_id`、`pg_stat_statements.wal_bytes`
> - PG 16+：`pg_stat_io` 系統 view（取代 `pg_statio_*` 的手動 snaphot，提供 cluster-wide I/O 統計）
>
> 特別注意：`pg_stat_io`（PG 16+）是一次重大改進——它提供了 cluster-level 的 I/O 統計（reads、writes、extends、fsyncs、hits），按 backend type 和 context 分類，無需再對 `pg_statio_*` 做手工 snaphot diff。
