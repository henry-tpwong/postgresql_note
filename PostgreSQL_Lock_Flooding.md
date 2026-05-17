# PostgreSQL Lock Flooding — 大鎖與 Long Transaction 的蝴蝶效應

> 來源：[digoal - PostgreSQL 性能优化之 - 大锁与long sql/xact的蝴蝶效应(泛洪) (2016-03-16)](https://github.com/digoal/blog/blob/master/201603/20160316_02.md)

---

## 1. 蝴蝶效應的四個因素

當以下四個因素疊加時，可能引發資料庫效能瞬間崩潰——這就是 PG 的 **lock flooding**（鎖泛洪）：

### 因素 1：Lock Queue 排隊機制

PG 的 lock queue 特性：即使 session 尚未持有 lock，只要進入 lock queue 就會產生 lock contention。鎖鏈範例：

```
Session A 持有 lock1 →  正常執行
Session B 等待 lock1 →  進入 queue，等待 A 釋放
Session C 請求 lock3 →  可能與 lock1 或 lock2 衝突（即使與已持有鎖無關，queue 前端的等待者也可能阻擋後端）
Session D 請求 lock4 →  可能與 lock1/2/3 都衝突
```

### 因素 2：Response 變慢 → Application 增加連線數

當 SQL 因鎖等待變慢，Application 層的典型反應是建立更多 connection 來處理積壓請求，試圖「補償」效能下降——這反而加劇問題。

### 因素 3：超過 Connection 閾值後效能遞減

PG 的 TPS 在 connection 數量超過一個閾值（通常是 CPU core 數的 **3 倍**）後開始顯著下降。原因包括 context switch 增加、shared_buffers contention、lock manager 的 internal spinlock 競爭。

### 因素 4：Object-Level Lock 在 Transaction 結束才釋放

即使是 `SELECT`，只要在 transaction 內（`BEGIN` 之後未 `COMMIT`），對table 持有的 `AccessShareLock` 直到 transaction 結束才釋放。這意味著一個「閒置的 long transaction」可以長時間持有 shared lock。

> 補充（Senior Dev）：因素 4 是最容易被忽略的根源。常見場景：
> - **ORM 框架**自動開啟 implicit transaction，開發者忘記 commit
> - **pgAdmin / DBeaver 查詢後未 commit**，而 auto-commit 被關閉
> - **Stored procedure** 內有 `BEGIN` 但異常路徑未 `ROLLBACK`
> - **idle in transaction** 狀態是頭號嫌疑犯（`SELECT state FROM pg_stat_activity WHERE state = 'idle in transaction'`）
>
> 檢測 idle in transaction 的簡單 threshold：
> ```sql
> SELECT pid, usename, datname, state,
>        now() - xact_start AS xact_duration,
>        query
> FROM pg_stat_activity
> WHERE state = 'idle in transaction'
>   AND now() - xact_start > interval '5 minutes';
> ```
> 建議在 production 設定 `idle_in_transaction_session_timeout`（PG 9.6+），例如 10min，自動 kill 這些 zombie transaction。

---

## 2. 模擬實驗：從正常到崩潰再到恢復的全過程

環境準備：

```sql
-- 啟用 lock wait tracking
log_lock_waits = on;
deadlock_timeout = 1s;

-- 建立測試表（1000 萬 row）
CREATE TABLE test (
    id int PRIMARY KEY,
    info text,
    crt_time timestamp
);
INSERT INTO test
    SELECT generate_series(1, 10000000),
           md5(random()::text),
           clock_timestamp();
```

測試腳本 `test1.sql`：

```sql
\setrandom id 1 10000000
UPDATE test SET info = info WHERE id = :id;
```

### Phase 1：正常狀態（10 connections）

```bash
pgbench -M prepared -n -r -P 1 -f ./test1.sql -c 10 -j 10 -T 10
```

```
progress: 2.0s, 65994.3 tps, lat 0.149 ms
progress: 3.0s, 67706.5 tps, lat 0.145 ms
progress: 4.0s, 72865.0 tps, lat 0.135 ms
progress: 5.0s, 77664.2 tps, lat 0.126 ms
progress: 6.0s, 77138.9 tps, lat 0.127 ms
progress: 7.0s, 75941.3 tps, lat 0.129 ms
progress: 8.0s, 77328.8 tps, lat 0.127 ms
```

TPS ≈ **75,000**，延遲 < 0.15ms。

### Phase 2：觸發 Long Transaction（持有 shared lock）

Session A 開啟一個長時間未結束的 transaction，對 test 表持有 `AccessShareLock`：

```sql
BEGIN;
SELECT * FROM test LIMIT 1;
--  1 | e86e219d... | 2016-03-16 16:07:49.814487
-- (不 COMMIT - 繼續持有 shared lock)
```

> 補充（Senior Dev）：除了手動開 transaction，另一個常見觸發源是 **auto-vacuum 的 `toast` table vacuum** 或 **VACUUM FULL**。`VACUUM FULL` 需要 `AccessExclusiveLock`，如果先被 long transaction 持有 shared lock 就永遠等不到。在 PG 9.6+ 可以用 `pg_blocking_pids(pid)` 函數快速定位 blocking chain：
> ```sql
> SELECT pid, pg_blocking_pids(pid) AS blocked_by,
>        wait_event_type, wait_event, query
> FROM pg_stat_activity
> WHERE wait_event_type = 'Lock';
> ```

### Phase 3：DDL 請求（觸發鎖鏈）

Session B 執行 DDL，需要 `AccessExclusiveLock`（lock level 8，最高）：

```sql
ALTER TABLE test ADD COLUMN c1 int;
-- 等待 test 的 AccessExclusiveLock → 但 Session A 仍持有 AccessShareLock
```

此 DDL 進入 lock queue **最前端**（因為 lock level 最高，PG 給更高優先級）。這瞬間堵塞所有後續 DML，因為 `UPDATE` 需要的 `RowExclusiveLock`（level 3）與 queue 中的 `AccessExclusiveLock`（level 8）衝突。

### Phase 4：TPS 歸零

正常業務（pgbench1，10 connections）瞬間 TPS 歸零：

```
progress: 53.0s, 0.0 tps
progress: 54.0s, 0.0 tps
progress: 55.0s, 0.0 tps
progress: 56.0s, 0.0 tps
progress: 57.0s, 0.0 tps
progress: 58.0s, 0.0 tps
progress: 59.0s, 0.0 tps
```

### Phase 5：Application 擴張 Connections（加劇災難）

Application 不知道 DB 堵塞了，新增 500 connections 來「緩解」：

```bash
pgbench -M prepared -n -r -P 1 -f ./test1.sql -c 500 -j 500 -T 10000
```

這 500 connections 全部進入 lock queue，狀態全部顯示 `PARSE waiting` / `BIND waiting`：

```
postgres [local] PARSE waiting
postgres [local] PARSE waiting
... x500 ...
```

> 補充（Senior Dev）：Lock queue 長度達到數百時，lock manager 的內部檢查（conflict detection）成本是 O(queue_length²)，因為每個新鎖請求都要與 queue 中所有等待者比對衝突。這解釋了為什麼不只是 DB 被阻塞——**DB process 本身也在消耗 CPU 做鎖衝突檢查**，讓情況惡性循環。

### Phase 6：Lock 釋放後的 Recovery（效能減半）

結束 long transaction 後（`COMMIT` 或 `ROLLBACK`），DDL 獲得 AccessExclusiveLock → 執行完畢 → 釋放。

所有 500+10 connections 同時開始執行積壓的 `UPDATE`：

pgbench2（500 connections，原擁塞）：

```
progress: 10.3s, 270.5 tps,  lat 1396.862 ms  ← 開始恢復
progress: 11.0s, 34443.5 tps, lat 64.132 ms
progress: 12.0s, 34986.1 tps, lat 14.229 ms
progress: 13.0s, 36645.0 tps, lat 13.661 ms
progress: 14.0s, 34570.1 tps, lat 14.463 ms
progress: 15.0s, 36435.8 tps, lat 13.752 ms
```

pgbench1（10 connections，原始業務）：

```
progress: 59.0s, 688.7 tps,  lat 340.857 ms   ← 被併發壓制
progress: 60.0s, 733.0 tps,  lat 13.659 ms
progress: 63.0s, 809.9 tps,  lat 12.370 ms
progress: 64.0s, 750.1 tps,  lat 13.338 ms
```

**TPS 只恢復到 ~36,000**（原來 75,000 的一半），因為 510 個 connections 超出了 CPU 的承載能力。

### Phase 7：釋放多餘 Connections 後完全恢復

終止額外的 500 connections 後：

```
progress: 66.0s, 1937.8 tps,  lat 5.044 ms  ← 過渡
progress: 67.0s, 64995.8 tps, lat 0.157 ms
progress: 68.0s, 73996.3 tps, lat 0.133 ms  ← 回到 ~75K TPS
progress: 69.0s, 78099.4 tps, lat 0.125 ms
```

---

## 3. Lock Log 分析

開啟 `log_lock_waits = on` 後，server log 會記錄 lock wait 事件：

```
-- DDL 等待 AccessExclusiveLock（lock queue 最前端）
process 48877 still waiting for AccessExclusiveLock on relation 61245
     after 1000.048 ms
Process holding the lock: 48557.
Wait queue: 48877, 46333, 46331, 46338, 46334, ...

-- UPDATE 等待 RowExclusiveLock（被 queue 中的 AccessExclusiveLock 阻擋）
process 46333 still waiting for RowExclusiveLock on relation 61245
     after 1000.036 ms
Process holding the lock: 48557.
Wait queue: 48877, 46333, ...

-- 新增的 500 connections 全部進 queue
process 49812 still waiting for RowExclusiveLock on relation 61245
     after 1000.207 ms
Process holding the lock: 48557.
Wait queue: 48877, 46333, ... (此處省略 500+ PIDs)

-- Lock acquired（DDL 獲得鎖，耗時 22.8 秒）
process 48877 acquired AccessExclusiveLock on relation 61245
     after 22836.312 ms

-- DDL 執行失敗（column 已存在 - 第二次執行）
process 48877 ERROR: column "c1" of relation "test" already exists

-- UPDATE connections 獲得鎖
process 49814 acquired RowExclusiveLock on relation 61245
     after 10177.162 ms
```

### Lock Chain 追蹤 SQL

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
    r.page AS r_page, r.tuple AS r_tuple,
    r.xact_start AS r_xact_start,
    r.query_start AS r_query_start,
    now() - r.query_start AS r_locktime,
    r.query AS r_query,
    w.mode AS w_mode,
    w.pid AS w_pid, w.page AS w_page,
    w.tuple AS w_tuple,
    w.xact_start AS w_xact_start,
    w.query_start AS w_query_start,
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
    (CASE w.mode
        WHEN 'INVALID' THEN 0
        WHEN 'AccessShareLock' THEN 1
        WHEN 'RowShareLock' THEN 2
        WHEN 'RowExclusiveLock' THEN 3
        WHEN 'ShareUpdateExclusiveLock' THEN 4
        WHEN 'ShareLock' THEN 5
        WHEN 'ShareRowExclusiveLock' THEN 6
        WHEN 'ExclusiveLock' THEN 7
        WHEN 'AccessExclusiveLock' THEN 8
        ELSE 0 END
    ) +
    (CASE r.mode
        WHEN 'INVALID' THEN 0
        WHEN 'AccessShareLock' THEN 1
        WHEN 'RowShareLock' THEN 2
        WHEN 'RowExclusiveLock' THEN 3
        WHEN 'ShareUpdateExclusiveLock' THEN 4
        WHEN 'ShareLock' THEN 5
        WHEN 'ShareRowExclusiveLock' THEN 6
        WHEN 'ExclusiveLock' THEN 7
        WHEN 'AccessExclusiveLock' THEN 8
        ELSE 0 END
    )
) DESC, r.xact_start;
```

> 補充（Senior Dev）：`is not distinct from` 用於處理 NULL-safe 比較（因為某些 locktype 的欄位可能是 NULL）。Lock level 的數字大小直接決定誰搶先獲得鎖——`AccessExclusiveLock (8)` 與 `RowExclusiveLock (3)` 衝突，DDL 排在 queue 前端。
>
> PG 9.6+ 簡化版的 blocking chain 查詢：
> ```sql
> SELECT blocked.pid AS blocked_pid,
>        blocked.query AS blocked_query,
>        blocking.pid AS blocking_pid,
>        blocking.query AS blocking_query,
>        now() - blocked.query_start AS wait_duration
> FROM pg_stat_activity blocked
> JOIN pg_stat_activity blocking
>   ON blocking.pid = ANY (pg_blocking_pids(blocked.pid))
> WHERE blocked.wait_event_type = 'Lock';
> ```
>
> PG 14+ 中 `pg_stat_activity` 新增了 `wait_event` 欄位的更多粒度（如 `LWLock` 的具體名稱），可用 `SELECT DISTINCT wait_event FROM pg_stat_activity WHERE wait_event_type = 'Lock'` 查看當前阻塞事件。

---

## 4. 根治方案

### 方案 1：強制 Lock Timeout

```sql
-- 全域設定
ALTER SYSTEM SET lock_timeout = '10s';
SELECT pg_reload_conf();

-- Session level（對特定 connection）
SET lock_timeout = '5s';
```

任何 SQL 等待 lock 超過此時間 → 自動 cancel（報錯 `canceling statement due to lock timeout`）。

> 補充（Senior Dev）：`lock_timeout` 與 `statement_timeout` 的區別：
> - `lock_timeout`：**只計** lock queue 中的等待時間，執行中的時間不計
> - `statement_timeout`：整條 SQL 的總限制（含 lock wait + execution）
> - 兩者應搭配使用：`lock_timeout = 5s` + `statement_timeout = 30s`

### 方案 2：Application 層自動釋放 Idle Connection

- Connection pooler（PgBouncer / Pgpool-II）設定 idle connection timeout
- ORM 層設定 connection max lifetime / max idle time
- `idle_in_transaction_session_timeout`（PG 9.6+）：自動 terminate idle-in-transaction

### 方案 3：監控 Lock Chain 與告警

- 定時執行上述 lock chain SQL，若發現 lock wait queue 長度超過閾值 → alert
- `auto_explain` 不記錄 lock wait 時間，必須依賴 server log 或上述監控 SQL 才能發現

> 補充（Senior Dev）：**DDL 在 Production 的執行規範**：
> 1. 所有 DDL 在執行前應檢查 `pg_stat_activity` 是否有 idle-in-transaction
> 2. DDL 加 `lock_timeout` 保護（如 `SET lock_timeout = '2s'; ALTER TABLE ...;`）
> 3. 可使用 `LOCK TABLE ... NOWAIT` 測試鎖是否可立即取得，失敗則 abort 排程
> 4. 大表的 DDL 考慮用 online tooling：`pg_repack`（取代 CLUSTER/VACUUM FULL）、`pg_squeeze`、`CREATE INDEX CONCURRENTLY`、`ALTER TABLE ... ADD COLUMN` 本身是 online 的（但需短暫 ACCESS EXCLUSIVE 在 catalog update 階段）
>
> 關於 Lock Manager 內部實現：PG 使用 lightweight lock (LWLock) 保護 lock table（`LockAcquire` / `LockRelease`）。當 lock queue 達到數千條時，`LockCheckConflicts` 的掃描成本成為 CPU bottleneck。這解釋了為什麼大量 connections 同時醒來會導致 CPU spike，因為每個 process 都要掃描整個 queue 確認自己是否可以獲得鎖。PG 14 對 lock manager 做了部分優化（fast-path locking 擴展），但根本上還是應該避免 queue 過長。

---

## 5. Lock Level 速查

| Level | Lock Mode | 常見觸發 | 衝突對象 |
|:-----:|------|------|------|
| 1 | AccessShareLock | `SELECT` | AccessExclusiveLock (8) |
| 2 | RowShareLock | `SELECT ... FOR UPDATE/SHARE` | ExclusiveLock, AccessExclusiveLock |
| 3 | RowExclusiveLock | `INSERT` / `UPDATE` / `DELETE` | ShareLock, ShareRowExclusiveLock, ExclusiveLock, AccessExclusiveLock |
| 4 | ShareUpdateExclusiveLock | `VACUUM`（不含 FULL）, `ANALYZE`, `CREATE INDEX CONCURRENTLY` | 同上 |
| 5 | ShareLock | `CREATE INDEX`（不含 CONCURRENTLY） | RowExclusive, ShareUpdateExclusive, ShareRowExclusive, Exclusive, AccessExclusive |
| 6 | ShareRowExclusiveLock | `CREATE TRIGGER` | RowExclusive, ShareUpdateExclusive, Share, ShareRowExclusive, Exclusive, AccessExclusive |
| 7 | ExclusiveLock | `REFRESH MATERIALIZED VIEW CONCURRENTLY` | RowShare, RowExclusive, ShareUpdateExclusive, Share, ShareRowExclusive, Exclusive, AccessExclusive |
| 8 | AccessExclusiveLock | `ALTER TABLE`, `DROP TABLE`, `TRUNCATE`, `VACUUM FULL` | **所有** lock level |
