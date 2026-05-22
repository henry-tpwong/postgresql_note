# PostgreSQL 秒殺場景優化：Advisory Lock vs FOR UPDATE NOWAIT

> 來源：[digoal - PostgreSQL 秒杀场景优化 (2015-09-14)](https://github.com/digoal/blog/blob/master/201509/20150914_01.md)
>
> 更新於 2026-05-17，補充 SKIP LOCKED 演進

---

## 核心問題：單行鎖成為瓶頸

秒殺場景典型瓶頸：多個 concurrent session 同時更新同一條 record，但只有少數成功（庫存有限）。

原始做法（無優化）：

```sql
UPDATE tbl SET xxx = xxx, upd_cnt = upd_cnt + 1
WHERE id = pk AND upd_cnt + 1 <= 5;  -- 假設 5 台庫存
```

獲得 row lock 的 session 處理期間，其他 session 全部等待。等待是浪費——對於沒搶到鎖的用戶，等待毫無意義。

---

## 方案對比與 Benchmark

測試表（單條 record，持續被更新）：

```sql
CREATE TABLE t1 (id INT PRIMARY KEY, info TEXT);
INSERT INTO t1 VALUES (1, now()::text);
```

### 方案 A：無優化 Baseline

```sql
UPDATE t1 SET info = now()::text WHERE id = :id;
```

128 concurrent client，100 秒：

```
tps = 2,855  (including connections establishing)
latency average: 44.806 ms
latency stddev: 45.751 ms
```

### 方案 B：FOR UPDATE NOWAIT

```sql
CREATE OR REPLACE FUNCTION f1(i_id INT) RETURNS void AS $$
DECLARE
BEGIN
  PERFORM 1 FROM t1 WHERE id = i_id FOR UPDATE NOWAIT;
  UPDATE t1 SET info = now()::text WHERE id = i_id;
  EXCEPTION WHEN OTHERS THEN
    RETURN;
END;
$$ LANGUAGE plpgsql;
```

無法即刻獲得 row lock 的 session 直接返回 error，事務 rollback，不等待。

**20 concurrent client，60 秒：**

```
tps = 13,196
latency average: 1.505 ms
idx_scan / idx_tup_fetch: 896,963   -- 大量無用功掃描
n_tup_upd: 41,775                   -- 實際成功更新
```

**128 concurrent client，100 秒：**

```
tps = 66,623
latency average: 1.919 ms
```

對比 baseline 提升 ~23 倍。

### 方案 C：pg_try_advisory_xact_lock（Direct UPDATE）

```sql
UPDATE t1 SET info = now()::text
WHERE id = :id AND pg_try_advisory_xact_lock(:id);
```

Advisory lock 比 row lock 更 light-weight（不需 engine lock manager）。`pg_try_advisory_xact_lock` 在 transaction level 工作，commit/rollback 時自動釋放。返回值為 boolean：取得鎖回 true，否則 false。

注意：advisory lock ID 在單個 database 內須保持 global unique，否則不同業務的 advisory lock 互相干擾。

**20 concurrent client，60 秒：**

```
tps = 23,194
latency average: 0.851 ms
idx_scan / idx_tup_fetch: 1,368,933  -- 無用功掃描仍極多
n_tup_upd: 54,957                     -- 實際成功更新
```

對比 FOR UPDATE NOWAIT 提升 ~75%。

### 方案 D：Advisory Lock + Function 包裝（消除無用掃描）

```sql
CREATE OR REPLACE FUNCTION f(i_id INT) RETURNS void AS $$
DECLARE
  a_lock BOOLEAN := false;
BEGIN
  SELECT pg_try_advisory_xact_lock(i_id) INTO a_lock;
  IF a_lock THEN
    UPDATE t1 SET info = now()::text WHERE id = i_id;
  END IF;
  EXCEPTION WHEN OTHERS THEN
    RETURN;
END;
$$ LANGUAGE plpgsql;
```

關鍵：先純 CPU 判斷 advisory lock 是否取得，拿到才做 UPDATE。沒拿到的 session 不做任何 table/index scan。

**20 concurrent client，60 秒：**

```
tps = 20,283
latency average: 0.973 ms
idx_scan / idx_tup_fetch: 75,927     -- 無用功掃描大幅下降
n_tup_upd: 75,927                    -- 每次 index scan 都轉化為有效更新
```

雖然 TPS 略低於方案 C（function call overhead），但無用功掃描從 1,368,933 降到 75,927（~18x 減少），實際有效更新數反而增加（54,957 → 75,927）。

### 物理機終極壓測（Advisory Lock）

**80 concurrent client，60 秒：**

```
tps = 231,197
latency average: 0.344 ms
latency stddev: 0.535 ms
```

對比 baseline 提升 ~81 倍，對比 FOR UPDATE NOWAIT 提升 ~2.5 倍。

### Perf Top（231K TPS 時）

```
GetSnapshotData       3.5%    postgres
AllocSetAlloc         2.6%    postgres
_int_malloc           1.6%    libc
LWLockAcquire         1.4%    postgres
hash_search_with_hash 1.3%    postgres
PostgresMain          1.1%    postgres
```

hot path 集中在 Snapshot 獲取、Memory Alloc、LWLock——都是 PG engine 層面的瓶頸，而非業務邏輯。

---

## 總結對比

| 方案 | TPS (20C) | TPS (128C/80C) | Index Scan 無用功 | 說明 |
|------|-----------|----------------|-------------------|------|
| 無優化 | — | 2,855 | — | row lock 排隊等待 |
| FOR UPDATE NOWAIT | 13,196 | 66,623 | 大量 | 即刻失敗不等待，但仍有大量 scan |
| Advisory Lock (direct) | 23,194 | — | 大量 | TPS 最高但大量無用 scan |
| Advisory Lock + 先判斷 | 20,283 | 231,197 | 極少 | 消除無用 scan，有效更新率最高 |

---

## SKIP LOCKED（PG 9.5+）

原文撰寫時 PG 9.5 即將發布，提到 `SELECT ... FOR UPDATE SKIP LOCKED` 新特性：

```sql
SELECT * FROM tbl
WHERE id = pk AND upd_cnt + 1 <= 5
FOR UPDATE SKIP LOCKED;
```

跳過已被鎖定的 row，不等待也不報錯。如果鎖最終在 `UPDATE` 層面也支援（即 `UPDATE ... SKIP LOCKED`），就不需要 advisory lock。但截至 PG 18，`SKIP LOCKED` 仍只支援 `SELECT ... FOR UPDATE`。

> 補充（Senior Dev）：實務做法是在 CTE 或 subquery 中用 `SELECT ... FOR UPDATE SKIP LOCKED` 鎖住可用 row，然後在外層 UPDATE 已鎖定的 row：

```sql
WITH locked AS (
  SELECT id FROM tbl
  WHERE upd_cnt + 1 <= 5
  ORDER BY id
  LIMIT 1
  FOR UPDATE SKIP LOCKED
)
UPDATE tbl SET info = now(), upd_cnt = upd_cnt + 1
FROM locked WHERE tbl.id = locked.id;
```

這樣既消除了 advisory lock 的全局 ID 管理問題，又做到了不等待、不無用掃描。

---

## 實戰考量

1. **Advisory lock ID 必須全局唯一**：所有秒殺商品要分配不同 advisory lock ID，否則不同商品之間互相阻塞。
2. **分段秒殺**：庫存打散成多條 record（如 100 台 → 10 條 record 各存 10 台），進一步提高並發。
3. **Pool 模式**：connection pool（pgbouncer）必須用 transaction pooling（非 session pooling），否則 advisory lock 可能在 session 重複使用時殘留。
4. **與 auto_explain 的互動**：高 TPS 場景下若啟用 `auto_explain.log_analyze = on`，每個 failed tx 也會被 EXPLAIN ANALYZE，overhead 顯著。

## 參考

- [PostgreSQL 9.5 Advisory Lock Functions](http://www.postgresql.org/docs/9.5/static/functions-admin.html#FUNCTIONS-ADVISORY-LOCKS)
- [SELECT ... FOR UPDATE SKIP LOCKED](http://blog.163.com/digoal@126/blog/static/163877040201551552017215/)
