# PostgreSQL 分組 TOP-N 效能優化：CTE Recursive + Index Loop 替代 Window Function（44x 加速）

> 來源：[digoal - PostgreSQL 雕蟲小技 CTE 遞歸查詢，分組TOP性能提升44倍 (2016-08-15)](https://github.com/digoal/blog/blob/master/201608/20160815_04.md)
>
> 相關：[digoal - 用遞歸查詢優化 count distinct](https://yq.aliyun.com/articles/39689)

---

## 1. 問題：Window Function 全表掃描

業務需求：每個 group 取 TOP N。例如每位歌手下載量 TOP 10、每個城市納稅前 10 名。

### 傳統寫法

```sql
SELECT * FROM (
  SELECT row_number() OVER (PARTITION BY c1 ORDER BY c2) AS rn, *
  FROM tbl
) t
WHERE t.rn <= 10;
```

### 實驗：10,000 groups / 10M rows

```sql
CREATE TABLE tbl (c1 INT, c2 INT, c3 INT);
CREATE INDEX idx1 ON tbl (c1, c2);

INSERT INTO tbl
SELECT mod(trunc(random() * 10000)::int, 10000),
       trunc(random() * 10000000)
FROM generate_series(1, 10000000);
```

### Window Function 執行計劃與效能

```sql
EXPLAIN (ANALYZE, VERBOSE, COSTS, TIMING, BUFFERS)
SELECT * FROM (
  SELECT row_number() OVER (PARTITION BY c1 ORDER BY c2) AS rn, *
  FROM tbl
) t
WHERE t.rn <= 10;
```

```
 Subquery Scan on t  (actual time=0.040..20813.469 rows=100000 loops=1)
   Filter: (t.rn <= 10)
   Rows Removed by Filter: 9900000
   Buffers: shared hit=10035535
   ->  WindowAgg  (actual time=0.035..18268.027 rows=10000000 loops=1)
         ->  Index Scan using idx1 on tbl  (actual time=0.026..11913.677 rows=10000000 loops=1)
               Buffers: shared hit=10035535
 Execution time: 20833.747 ms
```

**WindowAgg 掃描了全部 10M row**，僅為了每個 group 取前 10 條 → 淘汰 99%（9.9M row 被 `Rows Removed by Filter` 丟棄）。耗時 **20.8 秒**。

> 補充（Senior Dev）：WindowAgg 是 blocking operator——必須把整個 partition 的 row 讀完才能確定 `row_number()` 排序。即使 index 提供排序，`PARTITION BY c1` 的 boundary 跨越讓 planner 無法做 early termination。這與普通 `ORDER BY ... LIMIT` 的 Index Scan 能 early stop 的本質不同。

---

## 2. 優化方案：Recursive CTE 取 Distinct Group ID + Per-Group Index Seek

### 2.1 核心思路

1. 用 recursive CTE 取出所有 distinct `c1` 值（利用 index skip scan 行為）
2. 對每個 `c1` 執行 `SELECT * FROM tbl WHERE c1 = v ORDER BY c2 LIMIT 10`
3. 每個 group 的查詢只需要讀取 index 中的 10 條 entry（non-blocking）

### 2.2 Recursive CTE 取出所有 Group ID

```sql
WITH RECURSIVE t1 AS (
  (SELECT min(c1) c1 FROM tbl)
  UNION ALL
  (SELECT (SELECT min(tbl.c1) c1 FROM tbl WHERE tbl.c1 > t.c1) c1
   FROM t1 t WHERE t.c1 IS NOT NULL)
)
SELECT * FROM t1;
```

這種寫法利用 `min(c1)` + `WHERE c1 > t.c1` 在 `idx1 (c1,c2)` 上執行 index skip scan——每次取 current min 的下一組 min，不掃描 group 內全部 row。

### 2.3 PL/pgSQL Function 封裝（PG 9.5 限制）

由於當時 recursive CTE 不支援 `ORDER BY` 在啟動表內，需用 function 封裝循環：

```sql
CREATE OR REPLACE FUNCTION f() RETURNS SETOF tbl AS $$
DECLARE
  v INT;
BEGIN
  FOR v IN
    WITH RECURSIVE t1 AS (
      (SELECT min(c1) c1 FROM tbl)
      UNION ALL
      (SELECT (SELECT min(tbl.c1) c1 FROM tbl WHERE tbl.c1 > t.c1) c1
       FROM t1 t WHERE t.c1 IS NOT NULL)
    )
    SELECT * FROM t1
  LOOP
    RETURN QUERY SELECT * FROM tbl WHERE c1 = v ORDER BY c2 LIMIT 10;
  END LOOP;
  RETURN;
END;
$$ LANGUAGE plpgsql STRICT;
```

### 2.4 效能結果

```sql
EXPLAIN (ANALYZE, VERBOSE, TIMING, COSTS, BUFFERS) SELECT * FROM f();
```

```
 Function Scan on f  (actual time=419.218..444.810 rows=100000 loops=1)
   Function Call: f()
   Buffers: shared hit=170407, temp read=221 written=220
 Execution time: 464.257 ms
```

**20.8s → 464ms，44x 加速。**

| 指標 | Window Function | Recursive CTE + Function |
|------|----------------|--------------------------|
| Execution time | 20,834 ms | **464 ms** |
| shared_buffers hit | 10,035,535 | 170,407 (98% 減少) |
| Row scanned | 10,000,000 | ~10 per group × 10,000 = ~100,000 |

> 補充（Senior Dev）：
>
> **此方案成立的前提條件**：
> 1. **Group 數量 << Total row 數**。本例 10K groups / 10M rows = 0.1%。若 group 數接近總 row 數（如每個 group 只有 1-2 row），recursive CTE 的 per-group subquery overhead 反噬。
> 2. **Index 必須是 `(group_col, order_col)`**。本例 `idx1 (c1, c2)` 讓每個 `WHERE c1 = v ORDER BY c2 LIMIT 10` 能用 index 直接定位，無 sort。
> 3. **TOP N 遠小於 group 內 row 數**。N=10 時收益最大。若 N 接近 group 平均大小（如 TOP 500），則 window function 可能更優。
>
> **損益平衡點估算**：
> ```sql
> -- Group 數 = G, 每 group 平均 row = R
> -- Window: O(G × R)
> -- Recursive: O(G × N × log R) (N 次 index seek per group)
> -- 當 N < R 時遞歸方案開始有利；N << R 時優勢極大
> ```

---

## 3. 現代替代方案（PG 9.3+ / PG 11+ / PG 14+）

### 3.1 LATERAL + DISTINCT（PG 9.3+，無需 function）

```sql
SELECT t.*
FROM (
  SELECT DISTINCT c1 FROM tbl
) groups,
LATERAL (
  SELECT * FROM tbl
  WHERE tbl.c1 = groups.c1
  ORDER BY c2 LIMIT 10
) t;
```

`SELECT DISTINCT c1` 可用 `(c1, c2)` index 做 skip scan（planner 會選 Index Only Scan + `GROUP BY` 或 `DISTINCT`）。LATERAL 確保每個 group 只 access 10 row。

> 補充（Senior Dev）：`LATERAL` + `DISTINCT` 方案在 Planner 行為上與 recursive CTE 略有不同：`DISTINCT` 可能觸發 HashAgg 而非 Index Skip Scan（取決於 `c1` distinct cardinality estimate）。若 distinct value 極少，HashAgg 更優；但本例 10K 個 distinct group 時 Index Only Scan 的 skip 行為更有利。可用 `SET enable_hashagg = off` 強制 planner 選 index path。

### 3.2 CTE MATERIALIZED 控制（PG 12+）

```sql
WITH groups AS MATERIALIZED (
  SELECT DISTINCT c1 FROM tbl
)
SELECT t.*
FROM groups,
LATERAL (
  SELECT * FROM tbl WHERE tbl.c1 = groups.c1 ORDER BY c2 LIMIT 10
) t;
```

`MATERIALIZED` 強制 groups CTE 先計算並暫存，避免 planner 將其重寫回主查詢（`NOT MATERIALIZED`，PG 12 預設），讓 per-group 查詢穩定使用 index。

### 3.3 方案對比總結

| 方案 | PG 版本 | 優點 | 劣勢 |
|------|---------|------|------|
| Window Function | 8.4+ | 寫法簡單，適用於 TOP N 很大的場景 | 全表掃描，O(G×R) |
| Recursive CTE + Function | 9.0+ | N << R 時效能極佳 | 需 function 封裝、PL/pgSQL overhead |
| LATERAL + DISTINCT | 9.3+ | 純 SQL、無 function | Planner 可能選 HashAgg 而非 Index Skip |
| CTE MATERIALIZED + LATERAL | 12+ | Planner 行為可控 | 需暫時存儲 distinct group 結果 |
| `SELECT DISTINCT ON` | 8.4+ | 寫法最簡 | 不支援 LIMIT per group，只取 row_number=1 |

> 補充（Senior Dev）：
> - **實際生產選擇**：9.3+ 用 `LATERAL + DISTINCT`，12+ 加 `MATERIALIZED`，14+ planner 已對這種 pattern 有更好的 skip scan heuristic
> - **index 設計**：`(c1, c2)` 是這種 pattern 的黃金組合——`c1` 用於 group filter，`c2` 用於 order + limit。如果查詢還需要 `WHERE` 其他 column，考慮加到 index key 中讓 Filter 變成 Index Cond
> - **陷阱**：10K group × 10 top = 100K row 輸出看似不多，但 Planner 可能因 `LIMIT` 不像 `row_number() <= N` 有統計資訊而誤判 row estimate。EXPLAIN 中對比 `rows` estimate vs actual 是判斷 query plan 是否 optimal 的關鍵

---

## 參考

1. [用遞歸查詢優化 count distinct](https://yq.aliyun.com/articles/39689)
