# PostgreSQL Implicit Lock Request（隱式鎖請求）

> 來源：[digoal - 注意PostgreSQL "隐式"锁请求 (2015-11-05)](https://github.com/digoal/blog/blob/master/201511/20151105_01.md)
>
> 更新於 2026-05-17，補充 lock queue pileup 機制與現代化解方

---

## 核心發現：pg_get_indexdef 等函數請求的是 Table Lock，非 Index Lock

當你對一個表執行 DDL（如 `ALTER TABLE ... ADD COLUMN`）時，該表被加上 `AccessExclusiveLock`。此時你可能覺得只讀 index 定義不受影響——**但實際上 `pg_get_indexdef()` 會請求該 index 的**父表** `AccessShareLock`**，因此被 DDL 阻塞。

### 複現

**Session A — DDL 持有 AccessExclusiveLock：**

```sql
BEGIN;
ALTER TABLE test ADD COLUMN c1 INT;
-- AccessExclusiveLock 在 commit/rollback 前持續持有
```

**Session B — 讀取 index 定義被阻塞：**

```sql
SELECT * FROM pg_get_indexdef('test_pkey'::regclass);
-- 等待 test 表的 AccessShareLock（非 test_pkey index）
```

### Lock Debug Trace（Session B）

開啟 lock debug（參考 [lock debug 配置方法](http://blog.163.com/digoal@126/blog/static/163877040201422083228624)）：

```sql
SET client_min_messages = debug;
SET trace_locks = on;
SELECT * FROM pg_get_indexdef('test_pkey'::regclass);
```

輸出顯示 Lock 請求對象是 `(database_oid, relation_oid)` = `(13003, 42430)`，其中 42430 是 **test 表**的 OID，不是 test_pkey index 的 OID：

```
LOG:  LockAcquire: lock [13003,42430] AccessShareLock
LOG:  LockAcquire: found: lock(0x7f030ad3a750) id(13003,42430,0,0,0,1) grantMask(100)
LOG:  LockCheckConflicts: conflicting: ...
LOG:  WaitOnLock: sleeping on lock: ... type(AccessShareLock)
```

無衝突時的正常獲鎖路徑（對比）：

```
LOG:  LockAcquire: new: lock(0x...) id(13003,42430,0,0,0,1) ...
LOG:  LockCheckConflicts: no conflict: ...
LOG:  GrantLock: ...
```

---

### Lock Wait Chain 查詢

使用德哥的 lock wait query（來自 [generate_report.sh](https://github.com/digoal/pgsql_admin_script/blob/master/generate_report.sh)），可精確定位 blocked / blocking 關係：

```sql
WITH t_wait AS (
  SELECT a.mode, a.locktype, a.database, a.relation, a.page, a.tuple,
         a.classid, a.objid, a.objsubid, a.pid, a.virtualtransaction,
         a.virtualxid, a.transactionid,
         b.query, b.xact_start, b.query_start, b.usename, b.datname
  FROM pg_locks a, pg_stat_activity b
  WHERE a.pid = b.pid AND NOT a.granted
),
t_run AS (
  SELECT a.mode, a.locktype, a.database, a.relation, a.page, a.tuple,
         a.classid, a.objid, a.objsubid, a.pid, a.virtualtransaction,
         a.virtualxid, a.transactionid,
         b.query, b.xact_start, b.query_start, b.usename, b.datname
  FROM pg_locks a, pg_stat_activity b
  WHERE a.pid = b.pid AND a.granted
)
SELECT r.locktype,
       r.mode r_mode, r.usename r_user, r.datname r_db,
       r.relation::regclass, r.pid r_pid,
       r.xact_start r_xact_start, r.query_start r_query_start,
       now() - r.query_start r_locktime, r.query r_query,
       w.mode w_mode, w.pid w_pid,
       w.xact_start w_xact_start, w.query_start w_query_start,
       now() - w.query_start w_locktime, w.query w_query
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
ORDER BY f_lock_level(w.mode) + f_lock_level(r.mode) DESC, r.xact_start;
```

輸出顯示：

```
r_mode: AccessExclusiveLock   -- DDL (blocker)
r_relation: test
r_query: alter table test add column c1 int;
w_mode: AccessShareLock       -- pg_get_indexdef (blocked)
w_query: select * from pg_get_indexdef('test_pkey'::regclass);
```

> 補充（Senior Dev）：PG 9.6+ 可使用 `pg_blocking_pids(pid)` 簡化查詢，PG 14+ `pg_stat_activity` 的 `wait_event_type = 'Lock'` 可直接過濾出 lock waiting session。但上述自訂 query 仍保留價值——可看到完整的 lock 類型矩陣（relation/page/tuple/transactionid 全維度）。

---

## 受影響函數完整列表

所有 `pg_get_*def()` 系列函數都會向**父表/父物件**請求 `AccessShareLock`（而非子物件），因此都會被 DDL 阻塞：

| 函數 | 請求 Lock 的對象 | 說明 |
|------|-----------------|------|
| `pg_get_indexdef(oid)` | 父表 | index definition |
| `pg_get_constraintdef(oid)` | 父表 | constraint definition |
| `pg_get_functiondef(oid)` | 函數本身 (function oid) | function source code |
| `pg_get_function_arg_default(oid, int)` | 函數本身 | argument default |
| `pg_get_ruledef(oid)` | 父表 | rule definition |
| `pg_get_triggerdef(oid)` | 父表 | trigger definition |
| `pg_get_viewdef(oid)` | 檢視本身及其引用的**所有基礎表** | view definition |

### pg_get_ruledef 驗證

```sql
CREATE RULE r1 AS ON DELETE TO u1 DO ALSO SELECT 1;

-- Session A: ALTER TABLE u1 ADD COLUMN c2 INT;  -- AccessExclusiveLock
-- Session B: SELECT pg_get_ruledef(42458);
-- LockAcquire: lock [13003,25572] AccessShareLock  → blocked
-- 25572 = u1 表的 OID
```

### pg_get_viewdef 驗證

`pg_get_viewdef` 不僅鎖 view 本身，還會請求所有引用基礎表的 `AccessShareLock`。若其中任一基礎表被 DDL 持有 `AccessExclusiveLock`，查詢即被阻塞：

```
LockAcquire: lock [13003,42446] AccessShareLock  -- view 本身
LockAcquire: lock [13003,17229] AccessShareLock  -- 引用的基礎表之一
WaitOnLock: sleeping on lock: ... type(AccessShareLock)
```

> 補充（Senior Dev）：`pg_dump` 內部大量使用這些函數取得 schema definition（透過 `--schema-only` 或 `--table` 模式）。當你對生產庫執行 `pg_dump --schema-only` 時，這些隱式鎖會請求所有表的 `AccessShareLock`。如果此時有任何 DDL 在執行（或等待執行），pg_dump 會被阻塞——更危險的是，pg_dump 本身也會阻塞後續的 DDL（見下文 lock queue pileup 機制）。

---

## 連鎖效應：Lock Queue Pileup（鎖隊列堆積）

這是最危險的場景。原文指出了關鍵點：

> PG 對未獲得、但在等待中的鎖也在衝突列表中。

實際場景：

1. Session A 運行一個大型 `SELECT`（無鎖，但事務未結束 → `backend_xmin` 持有）
2. Session B 想對同表執行 DDL → 請求 `AccessExclusiveLock` → 因為 A 的事務未結束，B 進入 waiting queue
3. Session C 想讀取該表（`SELECT`，需要 `AccessShareLock`）→ **也被 B 阻塞**，因為 B 在 waiting queue 中，lock manager 將 B 視為 pending conflicting lock holder

結果：一個 DDL 在 waiting queue 中 = 後續所有對該表的操作全部阻塞。這不是單純的 1-to-1 衝突，而是 **1 個 pending DDL 堵死整張表**。

```
Session A: SELECT ... (長時間執行)
  → Session B: ALTER TABLE ... (Waiting on A)
    → Session C: SELECT ... (Waiting on B, 即便 A 不衝突)
      → Session D: SELECT ... (Waiting on B)
        → ...
          → 表完全無法訪問，所有 connection pool 耗盡
```

> 補充（Senior Dev）：這個問題在 PostgreSQL 社群被稱為 "lock queue pileup" 或 "ACCESS EXCLUSIVE lock starvation"。核心矛盾在於 lock manager 嚴格按 FIFO 排隊——pending 的 AccessExclusiveLock 會阻擋後續所有 AccessShareLock 請求，即使這些 SELECT 與 Session A 的 SELECT 互不衝突。這是 PG lock manager 設計權衡：避免 AccessExclusiveLock 永遠拿不到（starvation prevention），代價是偶發性 pileup。

---

## 防護措施

### 1. DDL 加 lock_timeout

```sql
SET lock_timeout = '1s';
ALTER TABLE test ADD COLUMN c1 INT;  -- 若 1s 內拿不到鎖→直接 fail，不進 waiting queue
```

### 2. DDL 加 statement_timeout（auto-commit 場景）

```sql
SET statement_timeout = '1s';
ALTER TABLE test ADD COLUMN c1 INT;
```

### 3. 監控前置：定期檢查 pending lock

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

### 4. 避免 Long Transaction（參照 Bloat 筆記）

Long transaction 持有 snapshot，DDL 等不到鎖，DDL 進 queue，堵死整張表。消除 long transaction 從根源絕殺 pileup。

> 補充（Senior Dev）：PG 15+ 的 `log_lock_waits = on` 會在 waiting > `deadlock_timeout` 時打印 lock queue 資訊到 PostgreSQL log，包含 blocked PID 和 blocking PID(s)。搭配 `pg_blocking_pids()` 和 `wait_event`，可建構自動告警系統。對於 critical table 的 DDL，強烈建議加 `SET lock_timeout = '1s'` 並配合 retry logic 在應用層而非資料庫層處理。

### 5. 使用 CONCURRENTLY 替代傳統 DDL

```sql
-- 避免 AccessExclusiveLock 長時間持有
CREATE INDEX CONCURRENTLY idx_test ON test (col);  -- ShareUpdateExclusiveLock
ALTER INDEX idx_test RENAME TO idx_test_v2;         -- ShareUpdateExclusiveLock
```

但注意：`ALTER TABLE ADD COLUMN` 至今仍需 `AccessExclusiveLock`（PG 18 仍如此），無法繞過。

---

## 原始碼參考

`src/include/storage/lock.h` 定義了 PG 的 lock mode 層級與衝突矩陣。`LockAcquire` → `LockCheckConflicts` → `WaitOnLock` 的呼叫鏈構成完整鎖機制。

## 參考

1. [lock debug 配置方法](http://blog.163.com/digoal@126/blog/static/163877040201422083228624)
2. [PG Lock Mode 矩陣](http://www.postgresql.org/docs/9.5/static/runtime-config-developer.html)
3. [generate_report.sh](https://github.com/digoal/pgsql_admin_script/blob/master/generate_report.sh)
4. `src/include/storage/lock.h`
