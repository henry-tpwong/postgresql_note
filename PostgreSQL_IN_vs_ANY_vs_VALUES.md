# PostgreSQL IN / =ANY(ARRAY) / =ANY(VALUES) / JOIN VALUES 效能對決

> 來源：[digoal - PostgreSQL in (...|values()) , = any (values|array) SQL 优化 (2014-10-16)](https://github.com/digoal/blog/blob/master/201410/20141016_01.md)
> 案例：[Datadog: 100x faster Postgres performance by changing 1 line](https://www.datadoghq.com/2013/08/100x-faster-postgres-performance-by-changing-1-line/)
>
> 更新於 2026-05-17，補充 PG 14~18 query rewrite 演進

---

## 四種寫法與 Execution Plan

```sql
-- 1) IN 列表
SELECT * FROM t2 WHERE id IN (1,2,3,100,1000,0);

-- 2) =ANY(ARRAY) → optimizer 內部改寫為 ScalarArrayOpExpr
SELECT * FROM t2 WHERE id = ANY ('{1,2,3,100,1000,0}'::integer[]);

-- 3) =ANY(VALUES) → 產生 HashAggregate + Nested Loop
SELECT * FROM t2 WHERE id = ANY (VALUES (1),(2),(3),(100),(1000),(0));

-- 4) JOIN VALUES → Nested Loop（不產生 HashAggregate）
SELECT t2.* FROM t2 JOIN (VALUES (1),(2),(3),(100),(1000),(0)) AS t(id) ON t2.id = t.id;
```

Execution plan 示範（PG 9.x）：

```
-- IN / =ANY(ARRAY)：被 optimizer 重寫為 ScalarArrayOpExpr
-- Index Scan using t2_pkey on t2 (cost=0.29..25.82 rows=6 width=37)
--   Index Cond: (id = ANY ('{1,2,3,100,1000,0}'::integer[]))

-- =ANY(VALUES)：產生 HashAggregate 去重
-- Nested Loop  (cost=0.38..46.04 rows=6 width=37)
--   ->  HashAggregate  (cost=0.09..0.15 rows=6 width=4)
--         ->  Values Scan on "*VALUES*"
--   ->  Index Scan using t2_pkey on t2
--         Index Cond: (id = "*VALUES*".column1)

-- JOIN VALUES：不產生 HashAggregate，直接 Nested Loop
-- Nested Loop  (cost=0.29..45.96 rows=6 width=37)
--   ->  Values Scan on "*VALUES*"
--   ->  Index Scan using t2_pkey on t2
--         Index Cond: (id = "*VALUES*".column1)
```

---

## Datadog 實戰案例：22s → 263ms（100x 提速）

### 原始問題

15 百萬 row 的表，查詢 11,000 個 primary key 時花了 22 秒。執行計劃核心節點是 **Bitmap Heap Scan**：

```
Bitmap Heap Scan on context c  (actual time=17128..22031 rows=10858 loops=1)
   Recheck Cond: ((tags @> '{blah}'::text[]) AND (x_key = 1))
   Filter: (key = ANY ('{15368196, ...11000 keys...}'::integer[]))
   Buffers: shared hit=50919
```

### 根因

大陣列 `=ANY(ARRAY[...])` 讓 optimizer 選擇 **Bitmap Heap Scan + Filter** 而非 **Index Scan**。Filter 逐行檢查 11,000 個 key 的 membership，即使有 primary key index 也用不上。

Bitmap Heap Scan 將所有符合 `tags` 和 `x_key` 條件的 page 讀出，然後在 Filter 階段逐行比對 key ∈ array → 產生了巨大的 Recheck overhead。

### 解法（一行差異）

```sql
-- Before: 22s
WHERE c.key = ANY (ARRAY[15368196, -- 11,000 other keys --])
-- After: 263ms (100x faster)
WHERE c.key = ANY (VALUES (15368196), -- 11,000 other keys --)
```

新計劃使用 **Nested Loop Index Scan**（一次性逐 key 精確查表）：

```
Nested Loop  (actual time=22.060..242.406 rows=10858)
   ->  HashAggregate  (actual time=21.529..32.820 rows=11215)
         ->  Values Scan on "*VALUES*"
   ->  Index Scan using context_pkey on context c  (actual rows=1 loops=11215)
         Index Cond: (c.key = "*VALUES*".column1)
         Filter: ((c.tags @> '{blah}') AND (c.x_id = 1))
```

關鍵差異：**Bitmap Heap Scan 遍歷大量 page → Index Scan 精確走 BTREE → 103x speedup**。

### Perf 視角

原始 22s 中 CPU 90/10 分配在 PostgreSQL / OS，幾乎無 disk I/O。說明時間不是花在打磁盤，而是 **Recheck Cond（逐行 filter + 陣列 membership 檢查）** 和 **BitmapAnd（兩個 bitmap index 合併）**。`rows_fetched` 指標顯示 Postgres 在大量「讀了→檢查→丟棄」的循環。

---

## PG 14 實測：work_mem 對 VALUES 策略的影響

測試環境：100 萬 row 的 t1（PK 為 id），t2 存有 10,000 個 id 的陣列。

```sql
CREATE UNLOGGED TABLE t1 (id INT PRIMARY KEY, info TEXT, crt_time TIMESTAMP);
INSERT INTO t1 SELECT generate_series(0, 1000000), md5(random()::text), clock_timestamp();

CREATE UNLOGGED TABLE t2 (id INT PRIMARY KEY, ids INT[]);

CREATE OR REPLACE FUNCTION gen_rands_arr(INT, INT) RETURNS INT[] AS $$
  SELECT array(SELECT (random() * $1)::INT FROM generate_series(1, $2));
$$ LANGUAGE SQL STRICT;

INSERT INTO t2 SELECT i, gen_rands_arr(1000000, 10000) FROM generate_series(1, 100) i;
```

動態產生 VALUES 語法進行對比：

```sql
DO LANGUAGE plpgsql $$
DECLARE
  v_ids TEXT;
BEGIN
  SELECT rtrim(ltrim(ids::TEXT, '{'), '}') INTO v_ids FROM t2 WHERE id = 1;
  EXECUTE format('SELECT * FROM t1 WHERE id IN (%s)', v_ids);
END;
$$;
-- → Index Scan using t1_pkey (actual rows=9958 loops=1)
--   Index Cond: (id = ANY ('{...}'::integer[]))
-- Duration: 23 ms
```

```sql
-- work_mem = 256kB → Sort (external merge Disk)
-- Nested Loop
--   ->  Unique  (Sort: external merge Disk: 160kB)
--         Sort Key: "*VALUES*".column1
--   ->  Index Scan using t1_pkey
-- Duration: 32 ms

-- work_mem = 4MB → HashAggregate (in-memory)
-- Nested Loop
--   ->  HashAggregate (Batches: 1  Memory Usage: 913kB)
--   ->  Index Scan using t1_pkey
-- Duration: 40 ms
```

`=ANY(VALUES(...))` 的效能受 **work_mem** 影響：低 work_mem 時 VALUES 去重用 external sort → disk I/O，高 work_mem 時用 in-memory HashAggregate。但即使 external sort，仍遠快於 Bitmap Heap Scan + Filter 方案（對比 Datadog 案例）。

> 補充（Senior Dev）：PG 14 起引入 `MIN_ARRAY_SIZE_FOR_HASHED_SAOP`（預設 9），當 `IN` 列表超過 9 個元素時，optimizer 自動將 `INDEX = ANY(ARRAY[...])` 從 sequential scan 切換為 hash table probe，大幅改善中等大小 IN 列表的效能。這使得 2014 年需要手動改寫成 `=ANY(VALUES)` 的場景在 PG 14+ 中愈來愈少。

---

## 效能矩陣：何時用哪種寫法

| 寫法 | 適合場景 | 效能特徵 | 注意 |
|------|---------|---------|------|
| `IN (1,2,3,4,5)` | ≤ 9 個元素（PG 14+ 自動 hash） | 內聯常數，最快 | 不適合應用層動態拼接 |
| `=ANY(ARRAY[...])` | ≤ 數百個元素 | optimizer 用 ScalarArrayOpExpr，可能走 Index Scan 或 Bitmap Scan | 大陣列 (> 數百) optimizer 可能選 Bitmap Heap Scan + Filter |
| `=ANY(VALUES(...))` | 數百~數萬個元素 | 產生 Values Scan + De-duplication + Nested Loop Index Scan | 比 =ANY(ARRAY) 慢於小列表，但大列表穩定性極佳 |
| `JOIN (VALUES(...))` | 需保留排序 | 直接 Nested Loop（不 HashAgg），結果順序 = VALUES 順序 | 不保證同一個值只回傳一次（若 t1 無 unique） |

### 選擇決策

```
元素數 ≤ 9     → IN (...)（PG 14+ 自動最佳化）
元素數 10~500  → =ANY(ARRAY[...])（optimizer 可選 Index Scan 或 Hash SAOP）
元素數 500~50K → =ANY(VALUES(...))（強制 Nested Loop Index Scan，避免 Bitmap Heap Scan 退化）
元素數 > 50K   → 考慮 TEMPORARY TABLE + JOIN + ANALYZE（減少 planning overhead）
```

> 補充（Senior Dev）：自 PG 14 `MIN_ARRAY_SIZE_FOR_HASHED_SAOP` 後，optimizer 在 `IN(...)` / `=ANY(ARRAY[...])` 超過 9 個元素時自動內部轉換為 hash table lookup，對小到中型陣列（數十至數百）幾乎消滅了曾需要手寫 `=ANY(VALUES)` 的場景。但在以下情境手動改寫仍有價值：
> 1. **超大列表**（> 數千元素）：避免 Bitmap Heap Scan 退化
> 2. **需要保留輸入順序**：`JOIN VALUES` 保留 VALUES 輸入順序（vs `=ANY(VALUES)` 經 HashAgg 隨機化）
> 3. **Multi-column IN**：`=ANY(VALUES(...))` 可處理多欄複合匹配，`=ANY(ARRAY)` 不行
> 
> PG 17 進一步引入 `enable_presorted_aggs`（預設 on），讓 optimizer 在某些情況下跳過 explicit sort 直接用 pre-sorted input，使 `=ANY(VALUES)` 在小 `work_mem` 時避免 external sort 懲罰。

---

## Multi-Column IN 的寫法

```sql
-- 單列 IN（四種寫法都可用）
WHERE id IN (1,2,3)
WHERE id = ANY(ARRAY[1,2,3])
WHERE id = ANY(VALUES(1),(2),(3))

-- 多列 IN：只有 =ANY(VALUES) 可用（ARRAY 只能處理單列）
WHERE (col_a, col_b) = ANY(VALUES (1,'x'), (2,'y'), (3,'z'))
-- 或
FROM t JOIN (VALUES (1,'x'),(2,'y'),(3,'z')) v(a,b) ON t.col_a=v.a AND t.col_b=v.b
```

`=ANY(ARRAY)` 無法處理複合 type（除非手動 CAST composite type）。`=ANY(VALUES)` 是原生支援多列比對的唯一簡潔寫法。

---

## 原始碼參考

1. `src/backend/parser/parse_relation.c` — `RTE_VALUES` 的 type inference（從 VALUES 第一列推斷 column type）：

```c
case RTE_VALUES:
{
    List *collist = (List *) linitial(rte->values_lists);
    Node *col;
    if (attnum < 1 || attnum > list_length(collist))
        elog(ERROR, "values list %s does not have attribute %d",
             rte->eref->aliasname, attnum);
    col = (Node *) list_nth(collist, attnum - 1);
    *vartype = exprType(col);
    *vartypmod = exprTypmod(col);
}
```

2. ScalarArrayOpExpr 機制（`src/backend/access/index/indexam.c`）：

```
 * 4. ScalarArrayOpExpr ("indexkey op ANY (array-expression)").
 * If the index has rd_am->amsearcharray, we handle these the same as simple
 * operators, setting the SK_SEARCHARRAY flag to tell the AM to handle them.
 * Otherwise, we create a ScanKey with everything filled in except the
 * comparison value, and set up an IndexArrayKeyInfo struct to drive
 * processing of the qual.
```

---

## 版本演進

| 功能 | 版本 | 說明 |
|------|------|------|
| `=ANY(VALUES)` 基本支援 | PG 8.4+ | 初代 VALUES 用法 |
| `plan_cache_mode = force_custom_plan` | PG 12 | 避免 generic plan 對 large IN list 選擇錯誤 |
| `MIN_ARRAY_SIZE_FOR_HASHED_SAOP` | PG 14 | IN 列表 ≥ 9 自動切換 hash table probe |
| `enable_presorted_aggs` | PG 17 | 減少 VALUES HashAgg 前的 explicit sort |

## 參考

1. [Datadog: 100x faster Postgres performance by changing 1 line](https://www.datadoghq.com/2013/08/100x-faster-postgres-performance-by-changing-1-line/)
2. 德哥相關：[PG 14 large IN list hash optimization](https://github.com/digoal/blog/blob/master/202105/20210519_02.md)
3. 德哥相關：[PostgreSQL 优化器逻辑推理能力 源码解析](https://github.com/digoal/blog/blob/master/201602/20160225_01.md)
