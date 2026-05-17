# max_locks_per_transaction 與物件數量對效能的影響

> 來源：[digoal - 数据库性能会随着对象增加而受影响吗？max_locks_per_transaction & pg_locks entrys limit (2014-10-17)](https://github.com/digoal/blog/blob/master/201410/20141017_01.md)

---

## 問題：資料庫中物件越多效能越差嗎？

核心問題：PostgreSQL 中存儲的 table / index / sequence 等物件數量越多，單表操作的效能是否會下降？

答案分兩層：對於**單表操作**（DML），效能幾乎不受影響；真正的隱患在於 **catalog metadata 的記憶體佔用**與 **shared lock table 的插槽上限**。

---

## 實驗：建立 50 萬張表

### 調整 max_locks_per_transaction

在一個 PL/pgSQL DO block 中批次建立大量 table，每次 `CREATE TABLE` 需要取得對應的 catalog lock。lock slots 公式：

```
shared lock table size = max_locks_per_transaction * (max_connections + max_prepared_transactions)
```

建表過程中 `pg_locks` 持續增長：

```
digoal=# select count(*) from pg_locks;
 count
-------
 18381
(1 row)

digoal=# select count(*) from pg_locks;
 count
-------
 33569
(1 row)

digoal=# select max(relation) from pg_locks;
   max
---------
 1190188
(1 row)
```

若 `max_locks_per_transaction` 不足，會報錯：

> `ERROR: out of shared memory`
> `HINT: You might need to increase max_locks_per_transaction.`

### 建表 PL/pgSQL

```sql
DO LANGUAGE plpgsql $$
DECLARE
  tbl name;
BEGIN
  CREATE TABLE IF NOT EXISTS tbl (id int PRIMARY KEY, info text, crt_time timestamp);
  FOR i IN 400000..500000 LOOP
    EXECUTE 'CREATE TABLE IF NOT EXISTS tbl_' || i || '(LIKE tbl INCLUDING ALL)';
  END LOOP;
END;
$$;
```

### 建完後的 Catalog 狀態

50 萬張 table 建完後，沒有任何 user data 插入，資料庫已達 **35 GB**（全為 catalog table 的 metadata）：

```
digoal=# select count(*) from pg_class;
  count
---------
 2000322
(1 row)
Time: 353.556 ms

digoal=# select count(*) from pg_class where relname ~ '^tbl';
  count
---------
 1000002
(1 row)
Time: 1917.384 ms

digoal=# select count(*) from pg_class where relname ~ '^tbl' and relkind='r';
 count
--------
 500001
(1 row)
Time: 2098.243 ms

digoal=# select count(*) from pg_attribute;
  count
----------
 10502467
(1 row)
Time: 1959.034 ms
```

| Catalog Table | Row Count |
|--------------|-----------|
| `pg_class` | 2,000,322 |
| `pg_attribute` | 10,502,467 |
| `pg_index` | 對應每個 index |
| 其他 (`pg_type`, `pg_depend` 等) | 成比例增長 |

> 補充（Senior Dev）：每個 table 在系統中不只佔 `pg_class` 一條 row，還有對應的 `pg_type`（composite type）、每個 column 一條 `pg_attribute`、每個 constraint/index 一條 `pg_index` 和 `pg_depend` entry。50 萬張表 → `pg_attribute` 超過 1000 萬條 row。catalog table 本質是普通 heap table，大量 metadata 會讓 `autovacuum` 對 catalog table 的 vacuum 成本急劇上升，這是隱性的維護開銷。

---

## 效能實測：50 萬表 vs 1 張表的單表 DML

### 測試 Function

```sql
CREATE OR REPLACE FUNCTION f_tbl(v_id int) RETURNS void AS $$
DECLARE
BEGIN
  UPDATE tbl SET info = 'test' WHERE id = v_id;
  IF NOT FOUND THEN
    INSERT INTO tbl VALUES (v_id, 'test');
  END IF;
EXCEPTION WHEN OTHERS THEN
  RETURN;
END;
$$ LANGUAGE plpgsql STRICT;
```

pgbench script：

```sql
\setrandom id 1 5000000
SELECT f_tbl(:id);
```

### 結果

**50 萬表環境下：**

```
pgbench -n -r -f ./test.sql -c 8 -j 4 -T 30
tps = 39231.529469 (excluding connections establishing)
statement latencies in milliseconds:
        0.002908        \setrandom id 1 5000000
        0.199018        select f_tbl(:id);
```

**只有 1 張表的環境下（重建 database）：**

```
pgbench -n -r -f ./test.sql -c 8 -j 4 -T 30
tps = 39714.073461 (excluding connections establishing)
statement latencies in milliseconds:
        0.002761        \setrandom id 1 5000000
        0.196848        select f_tbl(:id);
```

| 環境 | TPS | 差異 |
|------|-----|------|
| 50 萬表 | **39,231** | — |
| 1 張表 | **39,714** | +1.2% |

單表 DML 的效能差異幾乎在噪聲範圍內（~1.2%）。原因是 DML 執行時只會 lock 目標 table 對應的 catalog entry（已 cache 在 relcache 中），不會掃描整個 `pg_class`。

> 補充（Senior Dev）：這 1.2% 的微小差距可能不是來自 DML 本身，而是來自 catalog 變大後 autovacuum / background writer 的間歇性 overhead。在長連接 + 高並發場景中，差距可能因 memory pressure 而放大（見下文）。

---

## 真正的隱患：Memory 與 Lock Slot 耗盡

### Catalog Cache（syscache + relcache）記憶體佔用

PostgreSQL 在 backend 啟動後首次訪問一個 relation 時，會將該 relation 的 catalog metadata 載入 `relcache`。cache 是 per-backend（per-connection）的，不會跨連接共享。

**影響鏈：**

```
大量 table → catalog table 體積膨脹（pg_class / pg_attribute / pg_index）
         → 每個 backend 訪問多個 table 時 cache 多份 metadata
         → 長連接 → memory 不釋放 → OOM risk
```

關鍵：如果一個長連接在其生命週期中只訪問了 1 張表，那 cache 中只有 1 份 metadata。但如果連接池中的連接反覆訪問不同的表，cache 會隨著時間膨脹。

參考德哥後續文章：[PostgreSQL relcache在長連接應用中的記憶體霸佔"坑"](https://github.com/digoal/blog/blob/master/201607/20160709_01.md)

> 補充（Senior Dev）：生產環境中常見的相關問題：
> - **pg_dump / pg_restore**：在包含大量 table 的 database 中，`pg_dump` 需要掃描整個 `pg_class` 來構建 dump list，速度與 table 數線性相關。
> - **autovacuum**：autovacuum launcher 每個 `autovacuum_naptime` 周期都要掃 `pg_class` 找候選 table，table 數越多 cycle 越長。PG 13+ 對此做了一定優化（batch processing），但線性關係不變。
> - **Logical Replication / Decoding（PG 10+）**：每個 table 需要獨立的 replication slot tracking，table 數影響 startup 時間與 memory 佔用。
> - **Connection pooler（pgbouncer 等）**：transaction mode 下每個新 transaction 獲取新 backend，backend 啟動時需載入 catalog snapshot，table 數直接影響連接獲取延遲。
> - **`\dt` / `\d` 等 psql meta-command**：每次執行都查 `pg_class` + `pg_attribute`，大量 table 時這些指令明顯變慢。

### Lock Slot 耗盡機制

`max_locks_per_transaction` 控制的是 shared memory 中的 lock table slot 數量。當一個 backend 嘗試獲取 lock 但 slot 已滿時：

- **一般操作**：報錯 `ERROR: out of shared memory`，transaction abort
- **fast-path**：PG 為常見的 weak lock（AccessShareLock、RowShareLock 等）提供了 fast-path 機制，這些 lock 不佔用 shared lock table slot，直到發生衝突時才遷移到 main table

source code `src/backend/storage/lmgr/lock.c` — `LockAcquireExtended`：

```c
// 若 fast-path lock 發生衝突，需遷移到 main lock table
if (ConflictsWithRelationFastPath(locktag, lockmode))
{
    // ...
    if (!FastPathTransferRelationLocks(lockMethodTable, locktag, hashcode))
    {
        if (reportMemoryError)
            ereport(ERROR,
                    (errcode(ERRCODE_OUT_OF_MEMORY),
                     errmsg("out of shared memory"),
                     errhint("You might need to increase max_locks_per_transaction.")));
    }
}
// 建立或查找 lock + proclock entry
proclock = SetupLockInTable(lockMethodTable, MyProc, locktag, hashcode, lockmode);
if (!proclock)
{
    // 同上錯誤
}
```

> 補充（Senior Dev）：`max_locks_per_transaction` 的預設值是 64。計算公式 `64 * (100 + 0) = 6400` 在大多數場景足夠，因為 fast-path 已經承擔了大部份 AccessShareLock。真正需要調大此參數的情況：
> 1. 一個 transaction 中 DROP / CREATE 大量 table（每個需要 AccessExclusiveLock → 無法使用 fast-path → 全部佔 main table slot）
> 2. 大量 concurrent transaction 同時對大量不同 table 取強鎖
> 3. 使用 pg_dump 並行模式（`-j N`），每個 worker 可能同時鎖定大量 table
>
> 調大 `max_locks_per_transaction` 的代價是 shared memory 增加（每個 slot ~200 bytes at PG 9.x, ~240+ bytes at later versions due to 64-bit tracking）。在 `shared_buffers` 已經是 GB 級的系統上，lock table 的額外開銷通常可忽略。

### ShmemAlloc 的共享記憶體分配

source code `src/backend/storage/ipc/shmem.c` — 所有共用記憶體結構（lock table、buffer pool 等）最終都通過 `ShmemAlloc` 從 shared memory segment 中分配：

```c
void *
ShmemAlloc(Size size)
{
    Size newStart, newFree;
    void *newSpace;

    size = MAXALIGN(size);
    SpinLockAcquire(ShmemLock);
    newStart = shmemseghdr->freeoffset;
    if (size >= BLCKSZ)
        newStart = BUFFERALIGN(newStart);
    newFree = newStart + size;
    if (newFree <= shmemseghdr->totalsize)
    {
        newSpace = (void *) ((char *) ShmemBase + newStart);
        shmemseghdr->freeoffset = newFree;
    }
    else
        newSpace = NULL;
    SpinLockRelease(ShmemLock);

    if (!newSpace)
        ereport(WARNING, (errcode(ERRCODE_OUT_OF_MEMORY),
                          errmsg("out of shared memory")));
    return newSpace;
}
```

lock table 是 shared memory 中除 buffer pool 外最大的結構之一，調大 `max_locks_per_transaction` 會直接增加 shared memory 需求。

---

## Catalog 檢索：哪些操作走 Index，哪些不走？

Catalog table 在 database 啟動時會載入 shared memory（`src/backend/utils/cache`），大部分查詢走 index 且不會頻繁檢索。但以下情況可能走 Seq Scan 或大範圍 scan：

| 操作 | Catalog 掃描方式 | 大量 table 的影響 |
|------|-----------------|------------------|
| 單表 DML | Index lookup（OID / relname） | 幾乎無影響 |
| autovacuum launcher cycle | 掃 `pg_class` 找候選 | 線性增長 |
| pg_dump | 掃多張 catalog table | 線性增長 |
| `\dt` / `\d` | 掃 `pg_class` + join | 線性增長 |
| 某些 ORM / framework（如 Django migrations inspect db） | 可能全掃 catalog | **易成為瓶頸** |
| Logical replication startup | 逐表檢查 | 線性增長 |

> 補充（Senior Dev）：在實際應用中，如果發現應用框架在啟動時對 `pg_class` / `pg_attribute` 執行大量查詢（如 ActiveRecord 的 `db:schema:dump`、Django 的 `inspectdb`），可以：
> - 用 `EXPLAIN (ANALYZE, BUFFERS)` 抓出具體耗時的 catalog 查詢
> - 將這些查詢的結果緩存在應用層，避免每次連接都重複查詢
> - 確認 catalog table 的 stat 是否過時（`ANALYZE pg_class` 可能改善 planner 的 row estimate）

---

## 結論

| 場景 | 受影響程度 | 說明 |
|------|-----------|------|
| 單表 DML（OLTP） | **幾乎不受影響** | relcache 中已有 metadata，不走 catalog scan |
| 單表 DDL | **受 lock slot 限制** | 大批次 DDL 需調高 `max_locks_per_transaction` |
| 長連接 + 訪問多表 | **memory 膨脹** | relcache per-backend 不釋放 |
| autovacuum / maintenance | **線性變慢** | 需掃 `pg_class` 找候選 |
| pg_dump / pg_restore | **線性變慢** | 需掃 catalog |
| ORM / framework startup | **可能嚴重變慢** | 取決於框架是否大量查 catalog |

實務建議：盡量避免單一 database 中有數十萬張獨立 table。如需隔離資料，優先考慮 Partition（PG 10+ declarative partitioning）或 schema-per-tenant，而非 unlimited flat table namespace。

---

## 參考

1. [一個事務最多能鎖多少對象? how many objects can be locked per transaction](https://github.com/digoal/blog/blob/master/201103/20110301_01.md)
2. [PostgreSQL relcache在長連接應用中的記憶體霸佔"坑"](https://github.com/digoal/blog/blob/master/201607/20160709_01.md)
3. `src/backend/utils/cache` — catalog cache 實作
4. `src/backend/storage/lmgr/lock.c` — `LockAcquireExtended` 與 fast-path 機制
5. `src/backend/storage/ipc/shmem.c` — `ShmemAlloc` 共用記憶體分配

> [PG 版本註] 原文基於 PG 9.3.5（2014）。核心機制（fast-path lock、relcache per-backend、max_locks_per_transaction 公式）在最新版本（PG 17+）不變。主要演進：
> - PG 13+ 優化了 autovacuum 對大量 table 的批次處理
> - PG 14+ 優化了 catalog snapshot 建構速度（對大量 table 場景 startup 更快）
> - PG 15+ 進一步減少了 catalog scan 中的鎖競爭
> - 若使用 declarative partitioning（PG 10+），partition 數對 catalog 的壓力模式與獨立 table 類似，但 partitioned table 本身只佔一條 `pg_class` row
