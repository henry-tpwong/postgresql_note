# PostgreSQL Lock 機制完全指南

> 本文由 8 篇 PostgreSQL Lock 主題筆記合併整理而成，涵蓋從鎖的基礎概念、內部機制、監控除錯，到 Advisory Lock 的 4 個實戰應用場景（高並發更新、秒殺、排隊控制、無縫 ID）。
>
> **閱讀路徑**：新手建議從頭依序閱讀（第一篇 §1~§5 打基礎 → 第二篇 §1~§5 學應用）。有經驗的讀者可直接跳到需要的場景。
>
> 原始來源：[digoal（德哥）blog](https://github.com/digoal/blog) 系列文章，補充 PG 9.6~18 現代化演進與 Senior Dev 實戰註解。

---

# 一、PostgreSQL Lock 機制全景

## 1. Lock 基礎概念

### I. Lock Mode 等級與衝突矩陣

PostgreSQL 定義了 8 個鎖等級，數字越大鎖越強。每個等級與哪些操作對應、以及彼此之間的衝突關係如下：

| Level | Lock Mode | 常見觸發 | 衝突對象 |
|:-----:|------|------|------|
| 1 | AccessShareLock | `SELECT` | AccessExclusiveLock (8) |
| 2 | RowShareLock | `SELECT ... FOR UPDATE/SHARE` | ExclusiveLock, AccessExclusiveLock |
| 3 | RowExclusiveLock | `INSERT` / `UPDATE` / `DELETE` | ShareLock, ShareRowExclusiveLock, ExclusiveLock, AccessExclusiveLock |
| 4 | ShareUpdateExclusiveLock | `VACUUM`（不含 FULL）, `ANALYZE`, `CREATE INDEX CONCURRENTLY` | ShareLock, ShareRowExclusiveLock, ExclusiveLock, AccessExclusiveLock |
| 5 | ShareLock | `CREATE INDEX`（不含 CONCURRENTLY） | RowExclusive, ShareUpdateExclusive, ShareRowExclusive, Exclusive, AccessExclusive |
| 6 | ShareRowExclusiveLock | `CREATE TRIGGER` | RowExclusive, ShareUpdateExclusive, Share, ShareRowExclusive, Exclusive, AccessExclusive |
| 7 | ExclusiveLock | `REFRESH MATERIALIZED VIEW CONCURRENTLY` | RowShare, RowExclusive, ShareUpdateExclusive, Share, ShareRowExclusive, Exclusive, AccessExclusive |
| 8 | AccessExclusiveLock | `ALTER TABLE`, `DROP TABLE`, `TRUNCATE`, `VACUUM FULL` | **所有** lock level |

### II. Lock Queue 排隊機制

PG 的 lock queue 有一個關鍵特性：即使 session 尚未持有鎖，只要進入 lock queue 就會產生鎖衝突。鎖鏈範例：

```
Session A 持有 lock1 →  正常執行
Session B 等待 lock1 →  進入 queue，等待 A 釋放
Session C 請求 lock3 →  可能與 lock1 或 lock2 衝突（queue 前端的等待者也可能阻擋後端）
Session D 請求 lock4 →  可能與 lock1/2/3 都衝突
```

> PG 對未獲得、但在等待中的鎖也在衝突列表中。這意味著一個 pending 的高級別鎖（如 AccessExclusiveLock）可以堵死後續所有低級別鎖請求。

### III. Object-Level Lock 在 Transaction 結束才釋放

即使是 `SELECT`，只要在 transaction 內（`BEGIN` 之後未 `COMMIT`），對 table 持有的 `AccessShareLock` 直到 transaction 結束才釋放。這意味著一個「閒置的 long transaction」可以長時間持有 shared lock。

常見觸發源：
- **ORM 框架**自動開啟 implicit transaction，開發者忘記 commit
- **pgAdmin / DBeaver 查詢後未 commit**，而 auto-commit 被關閉
- **Stored procedure** 內有 `BEGIN` 但異常路徑未 `ROLLBACK`
- **idle in transaction** 狀態是頭號嫌疑犯

檢測 idle in transaction：

```sql
SELECT pid, usename, datname, state,
       now() - xact_start AS xact_duration,
       query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND now() - xact_start > interval '5 minutes';
```

建議在 production 設定 `idle_in_transaction_session_timeout`（PG 9.6+），例如 10min，自動 kill 這些 zombie transaction。

---

## 2. 隱式鎖請求（Implicit Lock Request）

> 來源：[digoal - 注意PostgreSQL "隐式"锁请求 (2015-11-05)](https://github.com/digoal/blog/blob/master/201511/20151105_01.md)

### I. 核心發現：pg_get_indexdef 請求的是 Table Lock，非 Index Lock

當你對一個表執行 DDL（如 `ALTER TABLE ... ADD COLUMN`）時，該表被加上 `AccessExclusiveLock`。此時你可能覺得只讀 index 定義不受影響——**但實際上 `pg_get_indexdef()` 會請求該 index 的父表 `AccessShareLock`**，因此被 DDL 阻塞。

**複現：**

Session A — DDL 持有 AccessExclusiveLock：

```sql
BEGIN;
ALTER TABLE test ADD COLUMN c1 INT;
```

Session B — 讀取 index 定義被阻塞：

```sql
SELECT * FROM pg_get_indexdef('test_pkey'::regclass);
```

### II. 受影響函數列表

所有 `pg_get_*def()` 系列函數都會向**父表/父物件**請求 `AccessShareLock`（而非子物件），因此都會被 DDL 阻塞：

| 函數 | 請求 Lock 的對象 |
|------|-----------------|
| `pg_get_indexdef(oid)` | 父表 |
| `pg_get_constraintdef(oid)` | 父表 |
| `pg_get_functiondef(oid)` | 函數本身 |
| `pg_get_ruledef(oid)` | 父表 |
| `pg_get_triggerdef(oid)` | 父表 |
| `pg_get_viewdef(oid)` | 檢視本身及其引用的所有基礎表 |

> `pg_dump` 內部大量使用這些函數。當你對生產庫執行 `pg_dump --schema-only` 時，這些隱式鎖會請求所有表的 `AccessShareLock`。如果此時有任何 DDL 在執行，pg_dump 會被阻塞——更危險的是，pg_dump 本身也會阻塞後續的 DDL（見下文 lock queue pileup 機制）。

### III. Lock Queue Pileup（鎖隊列堆積）

最危險的場景：**一個 pending DDL 堵死整張表**。

實際場景：
1. Session A 運行一個大型 `SELECT`（無鎖，但事務未結束）
2. Session B 想對同表執行 DDL → 請求 `AccessExclusiveLock` → 因為 A 的事務未結束，B 進入 waiting queue
3. Session C 想讀取該表（`SELECT`，需要 `AccessShareLock`）→ **也被 B 阻塞**，因為 B 在 waiting queue 中

```
Session A: SELECT ... (長時間執行)
  → Session B: ALTER TABLE ... (Waiting on A)
    → Session C: SELECT ... (Waiting on B, 即便 A 不衝突)
      → Session D: SELECT ... (Waiting on B)
        → ...
          → 表完全無法訪問，所有 connection pool 耗盡
```

> 這個問題在 PostgreSQL 社群被稱為 "lock queue pileup"。核心矛盾在於 lock manager 嚴格按 FIFO 排隊——pending 的 AccessExclusiveLock 會阻擋後續所有 AccessShareLock 請求，即使這些 SELECT 與 Session A 的 SELECT 互不衝突。這是 PG lock manager 設計權衡：避免 AccessExclusiveLock 永遠拿不到（starvation prevention），代價是偶發性 pileup。

### IV. 防護措施

**1. DDL 加 lock_timeout：**

```sql
SET lock_timeout = '1s';
ALTER TABLE test ADD COLUMN c1 INT;  -- 若 1s 內拿不到鎖→直接 fail，不進 waiting queue
```

**2. 監控前置：定期檢查 pending lock：**

```sql
SELECT pid, wait_event_type, wait_event,
       pg_blocking_pids(pid) blocked_by,
       now() - query_start AS wait_duration,
       query
FROM pg_stat_activity
WHERE wait_event_type = 'Lock'
  AND state <> 'idle'
ORDER BY query_start;
```

**3. 避免 Long Transaction：** 消除 long transaction 從根源絕殺 pileup。

**4. 使用 CONCURRENTLY 替代傳統 DDL：**

```sql
CREATE INDEX CONCURRENTLY idx_test ON test (col);  -- ShareUpdateExclusiveLock
```

注意：`ALTER TABLE ADD COLUMN` 至今仍需 `AccessExclusiveLock`（PG 18 仍如此），無法繞過。

---

## 3. Lock Flooding — 大鎖與 Long Transaction 的蝴蝶效應

> 來源：[digoal - PostgreSQL 大锁与long sql/xact的蝴蝶效应 (2016-03-16)](https://github.com/digoal/blog/blob/master/201603/20160316_02.md)

### I. 蝴蝶效應的四個因素

當以下四個因素疊加時，可能引發資料庫效能瞬間崩潰：

**因素 1：Lock Queue 排隊機制**（見 §2.III）

**因素 2：Response 變慢 → Application 增加連線數**

當 SQL 因鎖等待變慢，Application 層的典型反應是建立更多 connection 來處理積壓請求，試圖「補償」效能下降——這反而加劇問題。

**因素 3：超過 Connection 閾值後效能遞減**

PG 的 TPS 在 connection 數量超過 CPU core 數的 **3 倍**後開始顯著下降。原因包括 context switch 增加、shared_buffers contention、lock manager 的 internal spinlock 競爭。

**因素 4：Object-Level Lock 在 Transaction 結束才釋放**（見 §1.III）

### II. 模擬實驗：從正常到崩潰再到恢復

環境：1000 萬 row 表，10 connections baseline TPS ≈ 75,000。

**Phase 1 — 正常狀態（10 connections）：** TPS ≈ 75,000，延遲 < 0.15ms。

**Phase 2 — 觸發 Long Transaction：**

```sql
BEGIN;
SELECT * FROM test LIMIT 1;
-- 不 COMMIT - 繼續持有 AccessShareLock
```

**Phase 3 — DDL 請求觸發鎖鏈：**

```sql
ALTER TABLE test ADD COLUMN c1 int;
-- 等待 AccessExclusiveLock → 但 long transaction 持有 AccessShareLock
```

此 DDL 進入 lock queue 最前端，瞬間堵塞所有後續 DML。

**Phase 4 — TPS 歸零：**

```
progress: 53.0s, 0.0 tps
progress: 54.0s, 0.0 tps
progress: 55.0s, 0.0 tps
...
```

**Phase 5 — Application 擴張 Connections（加劇災難）：**

Application 新增 500 connections 來「緩解」——全部進入 lock queue。

> Lock queue 長度達到數百時，lock manager 的內部檢查（conflict detection）成本是 O(queue_length²)，因為每個新鎖請求都要與 queue 中所有等待者比對衝突。不只是 DB 被阻塞——DB process 本身也在消耗 CPU 做鎖衝突檢查。

**Phase 6 — Lock 釋放後的 Recovery（效能減半）：**

結束 long transaction 後，DDL 執行完畢。所有 510 connections 同時開始執行積壓的 UPDATE — **TPS 只恢復到 ~36,000**（原來 75,000 的一半），因為 510 個 connections 超出了 CPU 的承載能力。

**Phase 7 — 釋放多餘 Connections 後完全恢復：**

終止額外的 500 connections 後，TPS 回到 ~75,000。

### III. Lock 等待鏈查詢

使用以下 SQL 找出鎖等待鏈（誰阻塞了誰）：

```sql
WITH t_wait AS (
    SELECT a.mode, a.locktype, a.database, a.relation,
           a.page, a.tuple, a.classid,
           a.objid, a.objsubid, a.pid, a.virtualtransaction,
           a.virtualxid, a.transactionid,
           b.query, b.xact_start, b.query_start,
           b.usename, b.datname
    FROM pg_locks a, pg_stat_activity b
    WHERE a.pid = b.pid AND NOT a.granted
),
t_run AS (
    SELECT a.mode, a.locktype, a.database, a.relation,
           a.page, a.tuple,
           a.classid, a.objid, a.objsubid, a.pid,
           a.virtualtransaction, a.virtualxid,
           a.transactionid, b.query, b.xact_start,
           b.query_start, b.usename, b.datname
    FROM pg_locks a, pg_stat_activity b
    WHERE a.pid = b.pid AND a.granted
)
SELECT
    r.locktype, r.mode AS r_mode,
    r.usename AS r_user, r.datname AS r_db,
    r.relation::regclass, r.pid AS r_pid,
    r.xact_start AS r_xact_start,
    now() - r.query_start AS r_locktime,
    r.query AS r_query,
    w.mode AS w_mode,
    w.pid AS w_pid,
    now() - w.query_start AS w_locktime,
    w.query AS w_query
FROM t_wait w, t_run r
WHERE r.locktype IS NOT DISTINCT FROM w.locktype
  AND r.database IS NOT DISTINCT FROM w.database
  AND r.relation IS NOT DISTINCT FROM w.relation
  AND r.page IS NOT DISTINCT FROM w.page
  AND r.tuple IS NOT DISTINCT FROM w.tuple
  AND r.classid IS NOT DISTINCT FROM w.classid
  AND r.objid IS NOT DISTINCT FROM w.objid
  AND r.objsubid IS NOT DISTINCT FROM w.objsubid
  AND r.transactionid IS NOT DISTINCT FROM w.transactionid
  AND r.pid <> w.pid
ORDER BY (
    (CASE w.mode WHEN 'AccessShareLock' THEN 1 WHEN 'RowShareLock' THEN 2
     WHEN 'RowExclusiveLock' THEN 3 WHEN 'ShareUpdateExclusiveLock' THEN 4
     WHEN 'ShareLock' THEN 5 WHEN 'ShareRowExclusiveLock' THEN 6
     WHEN 'ExclusiveLock' THEN 7 WHEN 'AccessExclusiveLock' THEN 8 ELSE 0 END) +
    (CASE r.mode WHEN 'AccessShareLock' THEN 1 WHEN 'RowShareLock' THEN 2
     WHEN 'RowExclusiveLock' THEN 3 WHEN 'ShareUpdateExclusiveLock' THEN 4
     WHEN 'ShareLock' THEN 5 WHEN 'ShareRowExclusiveLock' THEN 6
     WHEN 'ExclusiveLock' THEN 7 WHEN 'AccessExclusiveLock' THEN 8 ELSE 0 END)
) DESC, r.xact_start;
```

> PG 9.6+ 簡化版：`SELECT * FROM pg_stat_activity WHERE pid = ANY(pg_blocking_pids(<blocked_pid>))`

### IV. 根治方案

| 方案 | 做法 |
|------|------|
| **強制 Lock Timeout** | `SET lock_timeout = '5s';` — 等待超時自動 cancel |
| **Connection Pool 管理** | PgBouncer 設定 idle connection timeout |
| **idle_in_transaction_session_timeout** | PG 9.6+，自動 terminate idle-in-transaction |
| **監控 Lock Chain 與告警** | 定時執行上述 lock chain SQL，queue 過長 → alert |
| **DDL 執行規範** | DDL 前檢查 `pg_stat_activity` + `lock_timeout` 保護 + 考慮 online tooling |

> **DDL 在 Production 的執行規範**：
> 1. 所有 DDL 在執行前應檢查是否有 idle-in-transaction
> 2. DDL 加 `lock_timeout` 保護（如 `SET lock_timeout = '2s'; ALTER TABLE ...;`）
> 3. 可使用 `LOCK TABLE ... NOWAIT` 測試鎖是否可立即取得
> 4. 大表的 DDL 考慮用 online tooling：`pg_repack`、`CREATE INDEX CONCURRENTLY`

---

## 4. Lock Wait 追蹤與監控

> 來源：[digoal - PostgreSQL 锁等待跟踪 (2016-03-18)](https://github.com/digoal/blog/blob/master/201603/20160318_02.md)
> 更新於 2026-05-17，補充 PG 9.6~18 現代化監控技術棧

### I. 核心問題：lock wait time 被計入 total duration 但無拆分

`auto_explain` 記錄的 total duration 包含了 lock wait time，但未單獨拆分：

```
duration: 32038.420 ms  plan:
Update on test2 (actual time=32038.418..32038.418 rows=0 loops=1)
  ->  Seq Scan on test2 (actual time=0.014..0.015 rows=1 loops=1)
```

Seq Scan 本身只花了 0.015ms，但 total duration 高達 32 秒——全數為 lock wait 耗時。

### II. 方法一：log_lock_waits（無需重編譯）

```ini
# postgresql.conf
log_lock_waits = on
deadlock_timeout = 1s
```

當 session 等待 lock 超過 `deadlock_timeout` 時，log 輸出：

```
LOG:  process 10220 still waiting for ShareLock on transaction 574614690 after 1000.036 ms
DETAIL:  Process holding the lock: 9725. Wait queue: 10220.
```

取得鎖之後：

```
LOG:  process 10220 acquired ShareLock on transaction 574614690 after 100194.020 ms
```

結合 auto_explain + log_lock_waits，可精確推算：總耗時 = lock wait + 實際運算。

> PG 13 起 `log_lock_waits` 輸出還包含 wait_event 資訊。搭配 `log_line_prefix = '%m [%p] %q%a '` 中的 `%p`（PID），可以在單條 log 中完成 PID → query → wait duration 的全鏈路追蹤。

### III. 方法二：pg_stat_activity 實時監控（現代推薦）

```sql
-- 找出所有正在等待 lock 的 session 及其等待時間
SELECT pid, usename, application_name,
       wait_event_type, wait_event,
       now() - wait_start AS wait_duration,
       pg_blocking_pids(pid) AS blocked_by,
       query
FROM pg_stat_activity
WHERE wait_event_type = 'Lock'
  AND state <> 'idle'
ORDER BY wait_start;
```

### IV. 方法三：LOCK_DEBUG 重編譯（最深粒度）

修改 `src/include/pg_config_manual.h` → `#define LOCK_DEBUG` → 重編譯 PG → 配置 `trace_locks = true`。輸出每個 lock acquire/release/grant/conflict check 的內部細節。

Lock ID 解碼格式 `id(db_oid, rel_oid, 0, block, tuple, 1)`：

| Lock Mode | Bitmask | 說明 |
|-----------|---------|------|
| AccessShareLock | `1<<0` | SELECT |
| RowShareLock | `1<<1` | SELECT FOR UPDATE / FOR SHARE |
| RowExclusiveLock | `1<<2` | INSERT / UPDATE / DELETE |
| ShareUpdateExclusiveLock | `1<<3` | VACUUM / ANALYZE / CREATE INDEX CONCURRENTLY |
| ShareLock | `1<<4` | CREATE INDEX (non-concurrent) |
| ShareRowExclusiveLock | `1<<5` | Triggers, some DDL |
| ExclusiveLock | `1<<6` | REFRESH MATERIALIZED VIEW CONCURRENTLY |
| AccessExclusiveLock | `1<<7` | ALTER TABLE / DROP TABLE / TRUNCATE |

### V. 三種方法選擇矩陣

| 方法 | 需要重編譯 | 適用場景 | 粒度 |
|------|-----------|---------|------|
| `log_lock_waits` | **否** | 生產環境常駐，日常 lock monitoring | lock type + waiting PID + duration |
| `pg_stat_activity` | **否** | 即時排查，整合 wait_event + blocking chain | 實時 snapshot |
| `LOCK_DEBUG` + `trace_locks` | **是** | 極端深度除錯、lock manager 內部行為分析 | 每個 lock acquire/release/grant/conflict check |

> 現代 PG（14+）的 `pg_stat_activity.wait_event` 已覆蓋 80% 日常需求，再加上 `log_lock_waits` 和 `pg_blocking_pids()`，幾乎不需要再重編譯。

### VI. 生產環境配置（無需重編譯）

```ini
# postgresql.conf
log_lock_waits = on
deadlock_timeout = 1s
log_line_prefix = '%m [%p] %d %u '
```

```sql
-- 定期執行監控 query（cron / pg_cron）
SELECT pid, wait_event_type, wait_event,
       now() - wait_start AS wait_duration,
       pg_blocking_pids(pid) AS blocked_by,
       query
FROM pg_stat_activity
WHERE wait_event_type = 'Lock'
  AND now() - wait_start > '5s'::interval;
```

| 功能 | 引入版本 |
|------|---------|
| `wait_event_type` / `wait_event` | PG 9.6 |
| `pg_blocking_pids(pid)` | PG 9.6 |
| `log_lock_waits` 增強（附加 wait_event） | PG 13 |
| `wait_start` | PG 14 |
| `pg_stat_activity.wait_for` | PG 17 |

---

## 5. max_locks_per_transaction 與物件數量對效能的影響

> 來源：[digoal - max_locks_per_transaction & pg_locks entrys limit (2014-10-17)](https://github.com/digoal/blog/blob/master/201410/20141017_01.md)

### I. 問題：資料庫中物件越多效能越差嗎？

核心問題：PostgreSQL 中 table / index / sequence 等物件數量越多，單表操作的效能是否會下降？

答案分兩層：對於**單表操作**（DML），效能幾乎不受影響；真正的隱患在於 **catalog metadata 的記憶體佔用**與 **shared lock table 的插槽上限**。

### II. 實驗：50 萬表 vs 1 張表的單表 DML

| 環境 | TPS | 差異 |
|------|-----|------|
| 50 萬表 | **39,231** | — |
| 1 張表 | **39,714** | +1.2% |

單表 DML 的效能差異幾乎在噪聲範圍內（~1.2%）。原因是 DML 執行時只會 lock 目標 table 對應的 catalog entry（已 cache 在 relcache 中），不會掃描整個 `pg_class`。

但建完 50 萬張表後，資料庫已達 **35 GB**（全為 catalog table 的 metadata）：

| Catalog Table | Row Count |
|--------------|-----------|
| `pg_class` | 2,000,322 |
| `pg_attribute` | 10,502,467 |

### III. 真正的隱患：Memory 與 Lock Slot 耗盡

**Catalog Cache（syscache + relcache）記憶體佔用：**

PostgreSQL 在 backend 啟動後首次訪問一個 relation 時，會將該 relation 的 catalog metadata 載入 `relcache`。cache 是 per-backend（per-connection）的，不會跨連接共享。

```
大量 table → catalog table 體積膨脹
         → 每個 backend 訪問多個 table 時 cache 多份 metadata
         → 長連接 → memory 不釋放 → OOM risk
```

**Lock Slot 耗盡機制：**

`max_locks_per_transaction` 控制 shared memory 中的 lock table slot 數量。公式：

```
shared lock table size = max_locks_per_transaction * (max_connections + max_prepared_transactions)
```

若不足，報錯：`ERROR: out of shared memory` / `HINT: You might need to increase max_locks_per_transaction.`

> PG 為常見的 weak lock（AccessShareLock、RowShareLock 等）提供了 fast-path 機制，這些 lock 不佔用 shared lock table slot，直到發生衝突時才遷移到 main table。因此 `max_locks_per_transaction = 64` 在大多數場景足夠。真正需要調大的情況：大批次 DDL、大量 concurrent transaction 同時對大量不同 table 取強鎖、pg_dump 並行模式。

### IV. Catalog 掃描影響

| 操作 | Catalog 掃描方式 | 大量 table 的影響 |
|------|-----------------|------------------|
| 單表 DML | Index lookup（OID / relname） | 幾乎無影響 |
| autovacuum launcher cycle | 掃 `pg_class` 找候選 | 線性增長 |
| pg_dump | 掃多張 catalog table | 線性增長 |
| `\dt` / `\d` | 掃 `pg_class` + join | 線性增長 |
| ORM / framework startup | 可能全掃 catalog | 易成為瓶頸 |
| Logical replication startup | 逐表檢查 | 線性增長 |

> 實務建議：盡量避免單一 database 中有數十萬張獨立 table。如需隔離資料，優先考慮 Partition（PG 10+ declarative partitioning）或 schema-per-tenant。

---

# 二、Advisory Lock 應用場景

## 1. Advisory Lock API 概述

PostgreSQL 的 Advisory Lock 是 application-level lock，不與 table/row 綁定，只認一個 `bigint key`（或兩個 `int key`）。適合跨多個 transaction 的協調、或需要自訂鎖語意的場景。

### I. Session-Level Lock（會話鎖）

| 函數 | 說明 |
|------|------|
| `pg_advisory_lock(key)` | 獲取排他 session lock（blocking） |
| `pg_advisory_lock_shared(key)` | 獲取共享 session lock（blocking） |
| `pg_try_advisory_lock(key)` | 嘗試獲取排他 session lock（non-blocking） |
| `pg_try_advisory_lock_shared(key)` | 嘗試獲取共享 session lock（non-blocking） |
| `pg_advisory_unlock(key)` | 釋放排他 session lock |
| `pg_advisory_unlock_shared(key)` | 釋放共享 session lock |
| `pg_advisory_unlock_all()` | 釋放當前 session 的所有 advisory lock |

### II. Transaction-Level Lock（交易鎖）

| 函數 | 說明 |
|------|------|
| `pg_advisory_xact_lock(key)` | 獲取排他 transaction lock（blocking，transaction 結束自動釋放） |
| `pg_advisory_xact_lock_shared(key)` | 獲取共享 transaction lock（blocking） |
| `pg_try_advisory_xact_lock(key)` | 嘗試獲取排他 transaction lock（non-blocking） |
| `pg_try_advisory_xact_lock_shared(key)` | 嘗試獲取共享 transaction lock（non-blocking） |

關鍵區別：
- **Session lock**：需手動釋放，或 session 斷開時自動釋放。適合跨多個 transaction 的協調。
- **Transaction lock**：transaction commit/rollback 時自動釋放，不需手動管理。

> Lock 數量無上限（不佔 `max_locks_per_transaction` slot，advisory lock 使用獨立的 shared memory hash table）。PG 9.6+ 可透過 `pg_locks` 查看：`SELECT * FROM pg_locks WHERE locktype = 'advisory'`。

---

## 2. 高並發全表更新：Advisory Lock vs SKIP LOCKED

> 來源：[digoal - PostgreSQL 使用advisory lock或skip locked消除行锁冲突 (2016-10-18)](https://github.com/digoal/blog/blob/master/201610/20161018_01.md)

### I. 場景

4GB 表，10,000 row，每 row 存大型 array。全表所有 row 需要週期性更新。單一 transaction 更新全表需 **80 秒**。目標：100 個 concurrent session 並行更新，每個 session 負責不同的 row subset，無 row lock conflict。

### II. 方法一：Advisory Lock

每次更新前嘗試取得該 row 對應的 advisory lock，拿到鎖才更新：

```sql
FOR v_id IN SELECT id FROM table LOOP
    IF pg_try_advisory_xact_lock(v_id) THEN
        UPDATE ... WHERE id = v_id;
    END IF;
END LOOP;
```

Benchmark（100 並行）：latency average = 4,407ms（vs 80s 單線程 → 18x 加速）。

### III. 方法二：SKIP LOCKED（PG 9.5+）

```sql
SELECT id INTO v_id FROM table
ORDER BY id LIMIT 1
FOR UPDATE SKIP LOCKED;
```

Benchmark（100 並行）：latency average = 4,204ms（與 advisory lock 相當）。

### IV. 兩種方法對比

| 維度 | Advisory Lock | SKIP LOCKED |
|------|--------------|-------------|
| PG 最低版本 | 9.4 | 9.5 |
| 實現難度 | 中（需管理 lock ID 全域唯一性） | 低（純 SQL） |
| 全域 ID 衝突風險 | 有 | 無 |
| Transaction 要求 | Transaction-level | Statement 結束釋放 |
| Connection Pool | 需 transaction pooling | 無此限制 |

### V. 現代化方案

**PG 15+ MERGE 批次處理：**

```sql
WITH batch AS (
    SELECT id FROM table
    WHERE id > :last_id ORDER BY id LIMIT 100
    FOR UPDATE SKIP LOCKED
)
UPDATE table t SET info = array_append(info, 1)
FROM batch b WHERE t.id = b.id;
```

**分片方案（無需 advisory lock 也不需 SKIP LOCKED）：**

```sql
SELECT id FROM table
WHERE mod(id, 4) = :shard_id
ORDER BY id LIMIT 100
FOR UPDATE SKIP LOCKED;
```

每個 session 有固定責任範圍（`mod(id, N)`），不需掃描全表。

---

## 3. 秒殺場景優化：Advisory Lock vs FOR UPDATE NOWAIT

> 來源：[digoal - PostgreSQL 秒杀场景优化 (2015-09-14)](https://github.com/digoal/blog/blob/master/201509/20150914_01.md)

### I. 核心問題

秒殺場景典型瓶頸：多個 concurrent session 同時更新同一條 record，獲得 row lock 的 session 處理期間，其他 session 全部等待——對於沒搶到鎖的用戶，等待毫無意義。

### II. 四種方案 Benchmark

**方案 A：無優化 Baseline**

```sql
UPDATE t1 SET info = now()::text WHERE id = :id;
```

128 concurrent, 100s: TPS = **2,855**。

**方案 B：FOR UPDATE NOWAIT**

```sql
PERFORM 1 FROM t1 WHERE id = i_id FOR UPDATE NOWAIT;
UPDATE t1 SET info = now()::text WHERE id = i_id;
```

128 concurrent, 100s: TPS = **66,623**（提升 ~23 倍）。無法即刻獲得 row lock 的 session 直接返回 error，不等待。

**方案 C：pg_try_advisory_xact_lock（Direct UPDATE）**

```sql
UPDATE t1 SET info = now()::text
WHERE id = :id AND pg_try_advisory_xact_lock(:id);
```

20 concurrent, 60s: TPS = **23,194**（比 FOR UPDATE NOWAIT 提升 ~75%）。

**方案 D：Advisory Lock + Function 包裝（消除無用掃描）**

先純 CPU 判斷 advisory lock 是否取得，拿到才做 UPDATE：

80 concurrent, 60s: TPS = **231,197**（對比 baseline 提升 ~81 倍）。

### III. 總結對比

| 方案 | TPS (20C) | TPS (128C/80C) | 無用掃描 | 說明 |
|------|-----------|----------------|---------|------|
| 無優化 | — | 2,855 | — | row lock 排隊等待 |
| FOR UPDATE NOWAIT | 13,196 | 66,623 | 大量 | 即刻失敗不等待 |
| Advisory Lock (direct) | 23,194 | — | 大量 | TPS 高但大量無用 scan |
| Advisory Lock + 先判斷 | 20,283 | 231,197 | 極少 | 消除無用 scan |

### IV. SKIP LOCKED 現代模式（PG 9.5+）

```sql
WITH locked AS (
    SELECT id FROM tbl
    WHERE upd_cnt + 1 <= 5
    ORDER BY id LIMIT 1
    FOR UPDATE SKIP LOCKED
)
UPDATE tbl SET info = now(), upd_cnt = upd_cnt + 1
FROM locked WHERE tbl.id = locked.id;
```

既消除了 advisory lock 的全局 ID 管理問題，又做到了不等待、不無用掃描。

> 實戰注意：Advisory lock ID 必須全局唯一（不同業務可能共用同一 lock namespace）；分段秒殺：庫存打散成多條 record 進一步提高並發；Pool 模式：必須用 transaction pooling（非 session pooling）。

---

## 4. OLTP 高並發排隊：Advisory Lock Admission Control

> 來源：[digoal - PostgreSQL OLTP 高並發請求性能優化 (2015-10-08)](https://github.com/digoal/blog/blob/master/201510/20151008_01.md)

### I. 核心問題：並發數超過 CPU 核數後 TPS 急降

在多核系統中，當 concurrent 數超過 CPU 核數的 2~3 倍後，效能開始下降。原因疊加：
1. Context switching 開銷：backend process 數量遠超 CPU 核數
2. Snapshot 管理成本：每個 active backend 都需建立 snapshot（ProcArrayLock contention）
3. Lock contention：大量 backend 同時競爭 row-level lock
4. Memory 壓力：每個 backend 消耗 ~5-10 MB

### II. 實驗：8000 active vs idle

**8000 active backend 全部執行 UPDATE：** TPS ≈ **9,124**（大部分時間消耗在 CPU scheduling）

**10 active + 7990 idle backend：** TPS ≈ **29,000**（idle connection 對效能幾乎無影響）

> 關鍵結論：只有真正執行 SQL 的 backend 才會參與 CPU 競爭。8000 idle 連接只佔用 memory（約 40-80GB），不參與 ProcArray lock / snapshot 競爭。

### III. Advisory Lock 排隊方案（Admission Control）

限制真正並行處理的 backend 數量，多出限制的連線等待：

```sql
CREATE OR REPLACE FUNCTION upd(l INT, v_id INT) RETURNS void AS $$
BEGIN
    LOOP
        IF pg_try_advisory_xact_lock(l) THEN
            UPDATE test_8000 SET cnt = cnt + 1 WHERE id = v_id;
            UPDATE test_8000 SET cnt = cnt + 2 WHERE id = v_id;
            RETURN;
        ELSE
            PERFORM pg_sleep(30 * random());
        END IF;
    END LOOP;
END;
$$ LANGUAGE plpgsql STRICT;
```

**實驗三：Advisory Lock 排隊（8000 backend，最多 10 個同時執行）**

TPS ≈ **26,774**，相比不排隊（9,124）提升約 **2.9 倍**。CPU idle 仍高達 93.9%——大部分 backend 處於 `pg_sleep()` 狀態。

### IV. 現代解決方案對比

| 方案 | 優點 | 缺點 |
|------|------|------|
| Advisory Lock 排隊 | 應用層可控、不需外部組件 | 仍佔用 8000 backend process（memory） |
| pgbouncer transaction pooling | 成熟穩定、極低 overhead | 不支援 prepared statement 跨 transaction |
| PG 14+ `idle_session_timeout` | 無需外部組件 | 不如 pgbouncer 精細 |
| `SKIP LOCKED`（PG 9.5+） | 天然支援工作佇列模式 | 僅適用於 queue-like table design |

> **現代最佳實踐**：pgbouncer transaction pooling + `max_client_conn = 10000` / `default_pool_size = 200` + PG 端 `max_connections = 200`。把 10,000 個 client 連線收斂到 200 個真正的 PG backend。

---

## 5. 無縫自增 ID：Advisory Lock Generator

> 來源：[digoal - PostgreSQL 无缝自增ID的实现 - by advisory lock (2016-10-20)](https://github.com/digoal/blog/blob/master/201610/20161020_02.md)

### I. SEQUENCE 的空洞問題

PostgreSQL 的 `SERIAL` / `SEQUENCE` 提供遞增數值，但 rollback 時序列值已被取走，產生空洞：

```sql
BEGIN;
INSERT INTO seq_test (info) VALUES ('test');  -- id=2
ROLLBACK;                                      -- id=2 永不復用
INSERT INTO seq_test (info) VALUES ('test');  -- id=3
```

這是 SEQUENCE 的設計取捨：為效能最大化（無 lock contention），犧牲連續性。

### II. 何時需要無縫 ID？

| 場景 | 是否需要無縫 | 原因 |
|------|------------|------|
| 發票號碼 (invoice number) | **是** | 法規 / 審計要求連續 |
| 銀行流水號 | **是** | 監管要求不可跳號 |
| PK (Primary Key) | 否 | 空洞不影響功能 |
| 對外 API 的 ID | 否 | 用 UUID 更好 |

> 即使業務需要無縫 ID，也應分離 PK 與業務 ID。PK 用 `SERIAL` / `BIGSERIAL` 確保寫入效能；業務 ID 用本文方案單獨生成。

### III. Advisory Lock Generator 實作

核心邏輯：以 `id` 值本身作為 advisory lock key——若 `pg_try_advisory_xact_lock(N)` 成功，表示當前 transaction 獨佔了 ID=N。

```sql
CREATE TABLE uniq_test (id int PRIMARY KEY, info text);

CREATE OR REPLACE FUNCTION f_uniq(i_info text) RETURNS int AS $$
DECLARE
    newid int;
    i int := 0;
    res int;
BEGIN
    LOOP
        IF i > 0 THEN
            PERFORM pg_sleep(0.2 * random());  -- 衝突時隨機退縮
        ELSE
            i := i + 1;
        END IF;

        SELECT max(id) + 1 INTO newid FROM uniq_test;

        IF newid IS NOT NULL THEN
            IF pg_try_advisory_xact_lock(newid) THEN
                INSERT INTO uniq_test (id, info) VALUES (newid, i_info);
                RETURN newid;
            ELSE
                CONTINUE;  -- 鎖被搶走，重試下一個 ID
            END IF;
        ELSE
            IF pg_try_advisory_xact_lock(1) THEN
                INSERT INTO uniq_test (id, info) VALUES (1, i_info);
                RETURN 1;
            END IF;
        END IF;
    END LOOP;

    EXCEPTION WHEN OTHERS THEN
        SELECT f_uniq(i_info) INTO res;
        RETURN res;
END;
$$ LANGUAGE plpgsql STRICT;
```

### IV. 效能實測

| 指標 | 數值 |
|------|------|
| 並發數 | 164 clients |
| TPS | **11,730** |
| Avg Latency | 13.79 ms |
| 結果 | count = max = 119,565（完美無空洞） |

### V. 與原生 SEQUENCE 對比

| 維度 | SERIAL / SEQUENCE | Advisory Lock 方案 |
|------|-------------------|-------------------|
| TPS (單表) | ~50,000-100,000+ | ~12,000 |
| 空洞 | 有 | 無 |
| Lock contention | 極低 | 高（每個 ID 需 try lock） |
| `SELECT max(id)` | 不需要 | 每次都要（隨 table 增大而變慢） |
| 適用場景 | 99% 的 OLTP | 業務 ID（invoice no. 等） |

### VI. 優化方向

**`SELECT max(id)` 的瓶頸：** 隨著 table 增大，每次呼叫都需 Index Only Scan on PK。解法：用 counter table 替代：

```sql
CREATE TABLE id_counter (current_max int);
INSERT INTO id_counter VALUES (0);

CREATE OR REPLACE FUNCTION f_uniq_v2(i_info text) RETURNS int AS $$
DECLARE newid int;
BEGIN
    UPDATE id_counter SET current_max = current_max + 1
        RETURNING current_max INTO newid;
    INSERT INTO uniq_test (id, info) VALUES (newid, i_info);
    RETURN newid;
END;
$$ LANGUAGE plpgsql;
```

但 `UPDATE id_counter` 變成 single-row hotspot（所有 session 排隊等同一行 lock），TPS 會驟降到 ~2,000。需按場景取捨。

> **生產最佳實踐**：PK 用 SERIAL，業務 ID 用本文方案分開管理。替代方案：counter table、外部 ID service（Redis INCR、Snowflake）、UUID v7（sortable、no gap concern）。

---

## 參考來源

1. [digoal - 注意PostgreSQL "隐式"锁请求](https://github.com/digoal/blog/blob/master/201511/20151105_01.md)
2. [digoal - PostgreSQL 大锁与long sql/xact的蝴蝶效应](https://github.com/digoal/blog/blob/master/201603/20160316_02.md)
3. [digoal - PostgreSQL 锁等待跟踪](https://github.com/digoal/blog/blob/master/201603/20160318_02.md)
4. [digoal - max_locks_per_transaction & pg_locks entrys limit](https://github.com/digoal/blog/blob/master/201410/20141017_01.md)
5. [digoal - PostgreSQL 使用advisory lock或skip locked消除行锁冲突](https://github.com/digoal/blog/blob/master/201610/20161018_01.md)
6. [digoal - PostgreSQL 秒杀场景优化](https://github.com/digoal/blog/blob/master/201509/20150914_01.md)
7. [digoal - PostgreSQL OLTP 高並發請求性能優化](https://github.com/digoal/blog/blob/master/201510/20151008_01.md)
8. [digoal - PostgreSQL 无缝自增ID的实现 - by advisory lock](https://github.com/digoal/blog/blob/master/201610/20161020_02.md)
9. [PostgreSQL Advisory Lock 官方文檔](https://www.postgresql.org/docs/9.6/static/functions-admin.html#FUNCTIONS-ADVISORY-LOCKS)
10. [generate_report.sh](https://github.com/digoal/pgsql_admin_script/blob/master/generate_report.sh)
11. `src/include/storage/lock.h` — lock mode 定義與衝突矩陣
12. `src/backend/storage/lmgr/lock.c` — lock manager 實作
