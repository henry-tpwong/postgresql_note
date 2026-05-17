# PostgreSQL 高並發全表更新：Advisory Lock vs SKIP LOCKED 消除行鎖衝突

> 來源：[digoal - PostgreSQL 使用advisory lock或skip locked消除行锁冲突, 提高几十倍并发更新效率 (2016-10-18)](https://github.com/digoal/blog/blob/master/201610/20161018_01.md)
>
> 更新於 2026-05-17，補充 CTE batch pattern / MERGE / 分區方案

---

## 場景

4GB 表，10,000 row，每 row 存大型 array（100,000 元素）。全表所有 row 需要週期性更新（如即時推薦系統的人群標記 bitmap）。

單一 transaction 更新全表 10,000 row 需 **80 秒**——無並發，不可接受。

```sql
CREATE UNLOGGED TABLE parallel_update_test (
  id INT PRIMARY KEY,
  info INT[]
);

INSERT INTO parallel_update_test
SELECT generate_series(1, 10000),
       (SELECT array_agg(id) FROM generate_series(1, 100000) t(id));

-- 單事務更新全表：80 秒
BEGIN;
UPDATE parallel_update_test SET info = array_append(info, 1);
-- UPDATE 10000, Time: 80212.641 ms
ROLLBACK;
```

目標：100 個 concurrent session 並行更新，每個 session 負責不同的 row subset，無 row lock conflict。

---

## 方法一：Advisory Lock（PG 9.4+）

每次更新前嘗試取得該 row 對應的 advisory lock，拿到鎖才更新。

```sql
CREATE OR REPLACE FUNCTION update_advisory() RETURNS void AS $$
DECLARE
  v_id INT;
BEGIN
  FOR v_id IN SELECT id FROM parallel_update_test
  LOOP
    IF pg_try_advisory_xact_lock(v_id) THEN
      UPDATE parallel_update_test
      SET info = array_append(info, 1)
      WHERE id = v_id;
    END IF;
  END LOOP;
END;
$$ LANGUAGE plpgsql STRICT;
```

核心邏輯：

```
FOR each id IN table:
  IF pg_try_advisory_xact_lock(id) THEN  ← 非阻塞 try
    UPDATE ... WHERE id = v_id            ← 拿到鎖才更新
  ELSE
    continue                              ← 這條被其他 session 拿了，跳過
```

`pg_try_advisory_xact_lock` 在 transaction level 工作，COMMIT/ROLLBACK 自動釋放鎖。100 concurrent session 各自掃描全表 10,000 row，但只對「拿到 advisory lock」的 id 執行 UPDATE。

**Benchmark（100 並行）：**

```
latency average = 4407.490 ms     ← vs 80s 單線程 → 18x 加速
tps = 22.708546
statement latencies: 3078.170 ms (select update())
```

---

## 方法二：SKIP LOCKED（PG 9.5+）

使用 `SELECT ... FOR UPDATE SKIP LOCKED` 跳過已被鎖定的 row。Iterate 模式：

```sql
CREATE OR REPLACE FUNCTION update_skip_locked() RETURNS void AS $$
DECLARE
  v_id INT;
BEGIN
  -- 獲取第一條未被鎖定的 row
  SELECT id INTO v_id FROM parallel_update_test
  ORDER BY id LIMIT 1
  FOR UPDATE SKIP LOCKED;

  UPDATE parallel_update_test
  SET info = array_append(info, 1)
  WHERE id = v_id;

  -- 從當前 id 繼續往後掃描
  LOOP
    SELECT id INTO v_id FROM parallel_update_test
    WHERE id > v_id ORDER BY id LIMIT 1
    FOR UPDATE SKIP LOCKED;

    IF FOUND THEN
      UPDATE parallel_update_test
      SET info = array_append(info, 1)
      WHERE id = v_id;
    ELSE
      RETURN;
    END IF;
  END LOOP;
END;
$$ LANGUAGE plpgsql STRICT;
```

**Benchmark（100 並行）：**

```
latency average = 4204.439 ms     ← 與 advisory lock 相當
tps = 23.813193
```

---

## 兩種方法對比

| 維度 | Advisory Lock | SKIP LOCKED |
|------|--------------|-------------|
| PG 最低版本 | 9.4 | 9.5 |
| 實現難度 | 中（需管理 lock ID 全域唯一性） | 低（純 SQL） |
| 性能（100 並行） | 4.4s | 4.2s（略快） |
| 全域 ID 衝突風險 | 有（不同業務可能共用同一 lock namespace） | 無 |
| Transaction 要求 | Transaction-level lock（commit 釋放） | Lock 在 statement 結束釋放（for update） |
| Connection Pool | 需 transaction pooling（session pooling 會殘留 lock） | 無此限制 |
| Index Scan 依賴 | 可 seq scan（FOR LOOP 逐行） | 建議有 index on `id`（`WHERE id > v_id`） |

---

## 現代化方案

### PG 15+ MERGE 替代 loop-based update

```sql
-- 原先：for loop + advisory lock / skip locked
-- 現代：MERGE 逐批處理

WITH batch AS (
  SELECT id FROM parallel_update_test
  WHERE id > :last_processed_id
  ORDER BY id
  LIMIT 100
  FOR UPDATE SKIP LOCKED
)
UPDATE parallel_update_test t
SET info = array_append(info, 1)
FROM batch b
WHERE t.id = b.id;
```

單次處理 100 row per batch，N 個 session 並行各取一批 → 消除 loop overhead。

### CTE + RETURNING（PG 12+）

```sql
WITH locked AS (
  SELECT id FROM parallel_update_test
  ORDER BY id
  LIMIT 100
  FOR UPDATE SKIP LOCKED
),
updated AS (
  UPDATE parallel_update_test t
  SET info = array_append(info, 1)
  FROM locked l
  WHERE t.id = l.id
  RETURNING t.id
)
SELECT count(*) FROM updated;
```

> 補充（Senior Dev）：loop-based function 在 100 concurrent session 各掃 10,000 row 時，總 scan 量 = 100 × 10,000 = 1,000,000 row（100 session 各掃全表）。這在更大規模（100 萬 row）時成為瓶頸。改為 batch CTE + LIMIT 100 + `WHERE id > last_id`，每個 session 只掃自己的批次範圍，總 scan 量 = 實際 row 數。

---

## 排程模式：pg_cron + 分片 ID

```sql
-- 每個 cron job instance 拿自己的 shard
SELECT mod(instance_id, 4) AS shard_id;

-- 只處理歸屬自己的 row
SELECT id FROM parallel_update_test
WHERE mod(id, 4) = :shard_id
ORDER BY id
LIMIT 100
FOR UPDATE SKIP LOCKED;
```

> 補充（Senior Dev）：分片方案讓每個 session 有固定責任範圍（`mod(id, N)`），不需掃描全表也不需 advisory lock 的 try-and-skip loop。缺點是 shard 數固定後擴縮不便。對動態變化的並發場景，advisory lock / SKIP LOCKED 的 dynamic work stealing 更靈活。

---

## 分區方案（PG 12+ 支援高效 partition pruning）

```sql
CREATE TABLE parallel_update_test (
  id INT,
  info INT[]
) PARTITION BY RANGE (id);

-- 按 1000 行為一個 partition
CREATE TABLE parallel_update_test_p0 PARTITION OF parallel_update_test
  FOR VALUES FROM (1) TO (1001);
CREATE TABLE parallel_update_test_p1 PARTITION OF parallel_update_test
  FOR VALUES FROM (1001) TO (2001);
-- ...

-- 每個 session 負責一個 partition
UPDATE parallel_update_test_p0 SET info = array_append(info, 1);
```

> 補充（Senior Dev）：100 個 partition，每個約 40MB，100 concurrent session 各自更新自己的 partition → 零 lock conflict，不需 advisory lock 也不需 SKIP LOCKED。這是最簡單的並發更新方案，但 partition 數固定，靈活性次於 dynamic work stealing。

---

## 實戰考量

1. **function IMMUTABLE trick**：若 update function 內使用 `nextval()`，需設為 `IMMUTABLE` 讓 optimizer 走 Index Scan（生產慎用）。
2. **dead tuple 風暴**：全表更新產生大量 dead tuple，vacuum 必須跟上。建議 `autovacuum_vacuum_scale_factor` 調低。
3. **HOT update**：`info = array_append(info, 1)` 會改變 tuple size → 非 HOT → index bloat。若頻繁更新大型 array，考慮用 `jsonb` + GIN（支援 in-place update）或外部存儲。
4. **work_mem**：100 concurrent session 各有自己的 function local memory，每個 `ARRAY[...]` 操作消耗 `work_mem`。

## 版本演進

| 功能 | 版本 | 說明 |
|------|------|------|
| Advisory Lock | PG 8.2+ | `pg_try_advisory_lock` / `pg_advisory_xact_lock` |
| `SKIP LOCKED` | PG 9.5 | `SELECT ... FOR UPDATE SKIP LOCKED` |
| `MERGE` | PG 15 | Upsert + conditional update，可替代部分批次邏輯 |
