# PostgreSQL Vacuum 原理與防止 Bloat

> 來源：[digoal - PostgreSQL 垃圾回收原理以及如何预防膨胀 (2015-04-29)](https://github.com/digoal/blog/blob/master/201504/20150429_02.md)
>
> 更新於 2026-05-17，補充 PG 12~18 新增能力

---

## Bloat 成因總覽

PostgreSQL MVCC 機制下，UPDATE / DELETE 產生 dead tuple，須由 VACUUM 回收空間。回收不及時即產生 bloat。

### 1. 未開啟 autovacuum

無 autovacuum 且無自訂 VACUUM 排程 → 必然膨脹。

### 2. autovacuum 開啟但回收不及時

#### 2.1 IO 差

大表執行 whole-table vacuum（prevent XID wraparound 或 table age > `vacuum_freeze_table_age` 時觸發全表掃描）產生大量 IO，拖慢回收速度。

#### 2.2 autovacuum 觸發閾值太晚

觸發公式（PG 原始碼）：

```
threshold = vac_base_thresh + vac_scale_factor * reltuples
```

預設值：`autovacuum_vac_thresh = 50`，`autovacuum_vac_scale = 0.2`。
→ dead tuple 達 table 的 ~20% 才觸發，回收完表已膨脹超過 20%。

> 補充（Senior Dev）：PG 13+ 針對 INSERT-only 表新增了獨立閾值：

```
insert_threshold = autovacuum_vac_insert_thresh
                  + autovacuum_vac_insert_scale_factor * reltuples
```

預設 `autovacuum_vac_insert_thresh = 1000`，`autovacuum_vac_insert_scale_factor = 0.2`，解決了過去 INSERT 多但 UPDATE/DELETE 少導致從不觸發 autovacuum 的問題——此類表雖無 dead tuple，但仍需 vacuum 來標記 frozen tuples 防止 XID wraparound。

#### 2.3 Worker 全忙

`autovacuum_max_workers` 決定最大 concurrent worker 數。當待 vacuum 的表超過 worker 數，部分表需等待空閒 worker。早前版本一個 database 同一時間只跑一個 worker，現已無此限制（同一 database 可多 worker concurrency vacuum）。

程式碼確認（`src/backend/postmaster/autovacuum.c`）：

```
 * Note that there can be more than one worker in a database concurrently.
 * They will store the table they are currently vacuuming in shared memory, so
 * that other workers avoid being blocked waiting for the vacuum lock for that
 * table.
```

每個 worker 記憶體消耗由 `autovacuum_work_mem`（預設 -1 = 沿用 `maintenance_work_mem`）控制，worker 越多記憶體需求越大。

> 補充（Senior Dev）：PG 16+ 新增 `vacuum_buffer_usage_limit`（預設 256 KB），限制每個 VACUUM/ANALYZE 可使用的 shared buffer 量，避免單一 vacuum 操作擠出 hot data。PG 18 進一步將此提升為 VACUUM 命令的 `BUFFER_USAGE_LIMIT` 選項。

#### 2.4 Long Transaction / Long SQL（最核心原因）

`backend_xid` 表示已申請 transaction ID 的事務（增刪改 DDL）。從申請 XID 開始持續到事務結束。
`backend_xmin` 表示 SQL 執行時的 snapshot 水位（可見的最大已提交 XID）。從 SQL 開始持續到 SQL 結束／游標關閉。

**關鍵機制**：VACUUM 只能回收 XID < OldestXmin 的事務產生的 dead tuple。只要存在任何持有 `backend_xid` 或 `backend_xmin` 且 XID < 新 dead tuple 的 session，這些 dead tuple 就無法被回收。

原始碼（`src/backend/utils/time/tqual.c`）：

```
/*
 * HeapTupleSatisfiesVacuum
 *      Determine the status of tuples for VACUUM purposes.
 *      OldestXmin is a cutoff XID (obtained from GetOldestXmin()).  Tuples
 *      deleted by XIDs >= OldestXmin are deemed "recently dead"; they might
 *      still be visible to some open transaction, so we can't remove them,
 *      even if we see that the deleting transaction has committed.
 */
HTSV_Result
HeapTupleSatisfiesVacuum(HeapTuple htup, TransactionId OldestXmin, Buffer buffer)
```

以下四種場景都會導致 `backend_xid` / `backend_xmin` 持續佔用：

| 場景 | 持續到何時 |
|------|-----------|
| 持有 XID 的長事務（增刪改 DDL） | 事務 COMMIT / ROLLBACK |
| 未關閉的游標（DECLARE CURSOR 後未 CLOSE） | 游標 CLOSE 或事務結束 |
| 長時間執行的查詢（`SELECT pg_sleep(1000)`） | SQL 執行結束 |
| REPEATABLE READ / SERIALIZABLE 隔離級別事務 | 事務 COMMIT / ROLLBACK |

監控 SQL：

```sql
SELECT datname, usename, query,
       xact_start, now() - xact_start AS xact_duration,
       state, backend_xid, backend_xmin
FROM pg_stat_activity
WHERE state <> 'idle'
  AND (backend_xid IS NOT NULL OR backend_xmin IS NOT NULL)
ORDER BY 4;
```

> 補充（Senior Dev）：PG 9.6 引入 `old_snapshot_threshold` 解決此問題（原文開頭提到的 "snapshot too old"），允許設定 snapshot 的生命週期上限，超過後會報錯 `snapshot too old` 而非無限阻塞 vacuum。但此參數有其 trade-off：可能導致 long-running query 被 kill，且不適用於 standby。PG 14+ 引入 `vacuum_failsafe_age`（預設 1.6 billion XID），作為 safety net：當 table age 逼近 wraparound 時，VACUUM 會跳過 index cleanup 等操作，優先完成 vacuum 以防止 shutdown，即使有 long transaction。

#### 2.5 autovacuum_vacuum_cost_delay 啟用

基於成本的垃圾回收，當達到 `autovacuum_vacuum_cost_limit` 後 sleep `autovacuum_vacuum_cost_delay` 再繼續。對 IO 充裕的系統反而拖慢回收。

成本計算由三個參數決定：

```
vacuum_cost_page_hit   = 1   (shared buffer 命中的 cost)
vacuum_cost_page_miss  = 10  (磁盤讀取的 cost)
vacuum_cost_page_dirty = 20  (弄髒 clean page 的額外 cost)
```

cost limit 在 active worker 之間按比例分配。

> 補充（Senior Dev）：PG 13+ 支援 per-table 的 `autovacuum_vacuum_cost_delay` 和 `autovacuum_vacuum_cost_limit` storage parameter，可對不同優先級的表設定不同 cost 策略。PG 14+ 新增 `maintenance_io_concurrency` 參數，控制 VACUUM 和 ANALYZE 的 prefetch 並發數。

#### 2.6 autovacuum launcher naptime 太長

Launcher process 的喚醒間隔由 `autovacuum_naptime` 決定。程式碼硬限制底限為 `MIN_AUTOVAC_SLEEPTIME = 100ms`：

```
/* the minimum allowed time between two awakenings of the launcher */
#define MIN_AUTOVAC_SLEEPTIME 100.0   /* milliseconds */
```

若設太大，launcher 睡過頭 → 有垃圾也沒人通知 fork worker。

#### 2.7 批量刪除／更新

例如 10GB 表一次 DELETE 9GB → 9GB dead tuple 必須等事務結束後才能回收，期間持續膨脹。

#### 2.8 非 HOT 更新導致 Index Bloat

非 HOT 更新產生新的 index entry，舊 index entry 需等到整個 index page 無任何引用才能回收。BTREE index 尤其容易膨脹。

---

## 測試驗證

測試環境參數：

```
autovacuum = on
log_autovacuum_min_duration = 0
autovacuum_max_workers = 10
autovacuum_naptime = 1
autovacuum_vacuum_threshold = 5
autovacuum_analyze_threshold = 5
autovacuum_vacuum_scale_factor = 0.002
autovacuum_analyze_scale_factor = 0.001
autovacuum_vacuum_cost_delay = 0
```

初始數據（200 萬 row，table 146 MB，index 43 MB）：

```sql
CREATE TABLE tbl (id INT PRIMARY KEY, info TEXT, crt_time TIMESTAMP);
INSERT INTO tbl SELECT generate_series(1,2000000), md5(random()::text), clock_timestamp();
```

測試腳本（一次更新最多 25 萬 row）：

```sql
\setrandom id 1 2000000
UPDATE tbl SET info = md5(random()::text) WHERE id BETWEEN :id-250000 AND :id+250000;
```

### Test 1：持有 XID 的長事務阻塞 Vacuum

正常壓測時 log 顯示 dead tuple 可正常回收：

```
tuples: 500001 removed, 1710872 remain, 0 are dead but not yet removable
tuples: 499647 removed, 1844149 remain, 0 are dead but not yet removable
```

開啟一個持有 transaction ID 的長事務：

```sql
-- Session A
BEGIN;
SELECT txid_current();    -- 例如 314030959
SELECT pg_backend_pid();  -- 例如 6073

-- Session B 查詢
SELECT backend_xid, backend_xmin FROM pg_stat_activity WHERE pid = 6073;
--  backend_xid  | backend_xmin
-- --------------+--------------
--   314030959   |  314030959

-- txid_current_snapshot 顯示該事務未結束
SELECT * FROM txid_current_snapshot();
-- 314030959:314030981:314030959
```

Log 立即顯示 dead tuple 無法回收，數字不斷增長：

```
tuples: 0 removed, 2391797 remain, 500001 are dead but not yet removable
tuples: 0 removed, 2459288 remain, 500001 are dead but not yet removable
tuples: 0 removed, 2713489 remain, 760235 are dead but not yet removable
tuples: 0 removed, 3023757 remain, 760235 are dead but not yet removable
tuples: 0 removed, 3135900 remain, 1137469 are dead but not yet removable
```

表與索引明顯膨脹（146 MB → 781 MB，index 43 MB → 308 MB）：

```sql
\dt+ tbl   -- 781 MB（原來 146 MB）
\di+ tbl_pkey  -- 308 MB（原來 43 MB）
```

結束該事務後，此前無法回收的垃圾全部釋放：

```sql
-- Session A
END;

-- Log 顯示回收恢復
tuples: 13629196 removed, 2515757 remain, 500001 are dead but not yet removable
tuples: 7183691 removed, 11252550 remain, 0 are dead but not yet removable
```

### Test 2：未關閉的游標阻塞 Vacuum

`backend_xmin` 在游標關閉前持續有效：

```sql
-- Session A
BEGIN;
DECLARE c1 CURSOR FOR SELECT 1 FROM pg_class;

-- Session B 查詢
SELECT backend_xid, backend_xmin FROM pg_stat_activity WHERE pid = 3823;
--  backend_xid | backend_xmin
-- -------------+--------------
--              |      5517228

-- Session A 取完數據但未關游標
FETCH ALL FROM c1;
-- 此時 backend_xmin 仍為 5517228
```

XID > 5517228 的事務產生的垃圾無法回收：

```sql
INSERT INTO t VALUES (1);
DELETE FROM t;
VACUUM VERBOSE t;
-- INFO:  "t": found 0 removable, 1 nonremovable row versions in 1 out of 1 pages
-- DETAIL:  1 dead row versions cannot be removed yet.
```

關閉游標後 `backend_xmin` 釋放，垃圾可回收：

```sql
CLOSE c1;
-- backend_xid | backend_xmin
-- -------------+--------------
--              |

VACUUM VERBOSE t;
-- INFO:  "t": removed 1 row versions in 1 pages
-- INFO:  "t": found 1 removable, 0 nonremovable row versions
```

### Test 3：長時間查詢阻塞 Vacuum

`backend_xmin` 持續到 SQL 執行結束：

```sql
BEGIN;
SELECT pg_sleep(1000);
-- backend_xmin 持續為某值直到 cancel 或執行完
-- Ctrl+C cancel 後 backend_xmin 釋放
```

### Test 4：Repeatable Read / Serializable 隔離級別

`backend_xmin` 持續到整個事務 COMMIT：

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT 1;
-- backend_xmin 持續存在直到 COMMIT
COMMIT;
-- backend_xmin 釋放
```

### Test 5：持續並發批量更新導致 Bloat

100 萬 row 的表，10 個 process 各自持續更新 10 萬 row：

```sql
-- t1.sql: UPDATE tbl SET info=info,crt_time=clock_timestamp() WHERE id >= 1 AND id < 100000;
-- t2.sql: WHERE id >= 100001 AND id < 200000;
-- ... t10.sql: WHERE id >= 900001 AND id <= 1000000;

pgbench -M prepared -n -r -f ./t1.sql -c 1 -j 1 -T 500000 &
# ... 10 個同時跑
```

結果：出現大量 `dead but not yet removable`，表從 73 MB 膨脹到 **554 MB**，index 從 21 MB 膨脹到 **114 MB**：

```
tuples: 0 removed, 2049809 remain, 999991 are dead but not yet removable
tuples: 0 removed, 2864735 remain, 1141307 are dead but not yet removable
```

三個並發問題疊加：(1) 垃圾產生速度 > 回收速度 (2) FSM 剩餘空間不足 → extend block (3) vacuum 過程中其他 process 持有排他鎖 → not yet removable。

**解法**：將單個事務的更新粒度大幅降低（改為單 row 隨機更新 + 極短事務）：

```sql
CREATE SEQUENCE seq CACHE 10;
UPDATE tbl SET info = info, crt_time = clock_timestamp()
WHERE id = mod(nextval('seq'), 2000001);

pgbench -M prepared -n -r -f ./test.sql -c 20 -j 10 -T 500000
```

結果：not yet removable 極少（< 1000），半小時後表體積穩定：

```sql
-- tbl: 75 MB（起始 73 MB，未膨脹）
-- tbl_pkey: 21 MB（起始 21 MB，未膨脹）
```

### Test 6：autovacuum_naptime 過長

將 `autovacuum_naptime = 1000` 秒後壓測：

```sql
SELECT * FROM pg_stat_all_tables WHERE relname = 'tbl';
-- n_dead_tup: 7,895,450（接近 800 萬 dead tuple）
-- n_live_tup: 2,328,301
```

表膨脹：73 MB → 393 MB，index 21 MB → 115 MB。

---

## 預防與優化措施

### 1. 務必開啟 autovacuum

```
autovacuum = on
```

### 2. 提高系統 IO

IO 越好，回收越快。

### 3. 調低觸發閾值

讓 dead tuple 少量時就觸發，而非等到 20%。對 1000 萬 row 的表希望 1 萬 dead tuple 就觸發：

```
autovacuum_vacuum_scale_factor = 0.001      # 預設 0.2
autovacuum_analyze_scale_factor = 0.0005    # 預設 0.1
```

> 補充（Senior Dev）：PG 12+ 新增 `autovacuum_vacuum_insert_threshold` 和 `autovacuum_vacuum_insert_scale_factor`，確保 INSERT-only 表（無 dead tuple）也會被 vacuum 來推進 freeze horizon。PG 13+ 所有 autovacuum 參數支援 per-table storage parameter。

### 4. 增加 Worker 數量與記憶體

```
autovacuum_max_workers = <CPU 核數>       # default 3
autovacuum_work_mem = 2GB                  # default -1 = maintenance_work_mem
```

確保系統剩餘記憶體 > `autovacuum_max_workers × autovacuum_work_mem`。

### 5. 應用層避免以下場景

| 避免 | 原因 |
|------|------|
| Long SQL（查增刪改 DDL 全部） | `backend_xmin` 長期佔用 |
| 打開游標不關閉 | `backend_xmin` 持續到游標關閉 |
| 非必要的 REPEATABLE READ / SERIALIZABLE | `backend_xmin` 持續到事務結束 |
| 大表 pg_dump（隱式 repeatable read） | 全庫備份期間的 snapshot 阻止 vacuum |
| 長時間不關閉的持有 XID 事務 | `backend_xid` 阻止 vacuum |
| 大批量 DELETE / UPDATE 在單一事務 | 大量 dead tuple 等事務結束才能回收 |

> 注意事項：standby 開啟 `hot_standby_feedback = on` 且有 long query 時，同樣會阻止 primary 的 vacuum。參考：[PostgreSQL物理"备库"的哪些操作或配置，可能影响"主库"的性能、垃圾回收、IO波动](https://github.com/digoal/blog/blob/master/201704/20170410_03.md)

### 6. IO 好的系統關閉 cost delay

```
autovacuum_vacuum_cost_delay = 0
```

### 7. 調低 autovacuum_naptime

```
autovacuum_naptime = 1s
```

若仍太長可修改原始碼底限為 1ms：

```
#define MIN_AUTOVAC_SLEEPTIME 1.0   /* milliseconds */
```

但需注意：若有 long transaction 導致垃圾無法回收，過短的 naptime 會讓 worker 不斷喚醒掃描→無法回收→再喚醒，浪費 IO/CPU。

### 8. 拆分批處理為小事務

將大範圍 UPDATE / DELETE 拆分為多個小事務，減少單次事務對 FSM 空間的壓力與鎖持有時間。

### 9. 使用較大 Block Size

現代硬體建議 32KB。`fillfactor` 不需調低（預設 100），因為表總有垃圾，每個 block 被更新後都不可能是滿的。

### 10. 已膨脹的回收方案

| 方案 | 排他鎖 | 說明 |
|------|--------|------|
| `VACUUM FULL` / `CLUSTER` | 需要 AccessExclusiveLock | table rewrite，全程鎖表 |
| `pg_repack` / `pg_reorg` | 僅最終 swap filenode 時短暫鎖 | 推薦，鎖定時間最短 |

---

## 原始碼參考

1. `src/backend/postmaster/autovacuum.c` — autovacuum daemon / worker scheduling / free worker list
2. `src/backend/utils/time/tqual.c` — `HeapTupleSatisfiesVacuum()` OldestXmin 判定邏輯
3. `src/backend/storage/ipc/sinvaladt.c` — `BackendIdGetTransactionIds()` 取得 `backend_xid` / `backend_xmin`：

```
/*
 * BackendIdGetTransactionIds
 *              Get the xid and xmin of the backend.
 */
void
BackendIdGetTransactionIds(int backendID, TransactionId *xid, TransactionId *xmin)
{
        SISeg      *segP = shmInvalBuffer;
        *xid = InvalidTransactionId;
        *xmin = InvalidTransactionId;
        LWLockAcquire(SInvalWriteLock, LW_SHARED);
        if (backendID > 0 && backendID <= segP->lastBackend)
        {
                ProcState  *stateP = &segP->procState[backendID - 1];
                PGPROC     *proc = stateP->proc;
                if (proc != NULL)
                {
                        PGXACT     *xact = &ProcGlobal->allPgXact[proc->pgprocno];
                        *xid = xact->xid;
                        *xmin = xact->xmin;
                }
        }
        LWLockRelease(SInvalWriteLock);
}
```

---

## 版本變遷速查

| 功能 | 引入版本 | 說明 |
|------|---------|------|
| `old_snapshot_threshold` | PG 9.6 | snapshot too old，限制 snapshot 生命週期上限 |
| `autovacuum_vacuum_insert_threshold` | PG 12 | INSERT-only 表 vacuum 觸發閾值 |
| `autovacuum_vacuum_insert_scale_factor` | PG 12 | INSERT-only 表 vacuum 觸發比例 |
| Per-table `autovacuum_vacuum_cost_delay` | PG 13 | 不同表可設定不同 cost 策略 |
| `vacuum_failsafe_age` | PG 14 | 防止 XID wraparound 的 safety net |
| `maintenance_io_concurrency` | PG 14 | VACUUM / ANALYZE 的 prefetch 並發數 |
| `log_autovacuum_min_duration` 支援 per-table | PG 14 | 更精細的 log 控制 |
| `autovacuum_max_workers` 上限提升 | PG 15 | 支援更多 concurrent vacuum worker |
| `vacuum_buffer_usage_limit` | PG 16 | 限制單個 vacuum 的 shared buffer 用量 |
| `VACUUM PARALLEL` | PG 18 | 並行 vacuum 加速索引 cleanup |
| `VACUUM SKIP_LOCKED` | PG 18 | 跳過無法鎖定的 relation |

## 參考

- [PostgreSQL物理"备库"的哪些操作或配置，可能影响"主库"的性能、垃圾回收、IO波动](https://github.com/digoal/blog/blob/master/201704/20170410_03.md)
