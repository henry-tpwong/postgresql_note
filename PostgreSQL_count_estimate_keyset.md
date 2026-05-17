# PostgreSQL count(*) 替代方案與分頁 keyset 位點優化

> 來源：[digoal - 论 count 与 offset 使用不当的罪名和分页的优化 (2016-05-06)](https://github.com/digoal/blog/blob/master/201605/20160506_01.md)
>
> 相關：[PostgreSQL 分頁優化三部曲（OFFSET / Cursor / Keyset / 標記列）](PostgreSQL_分頁優化.md)

---

本文聚焦兩個分頁場景的核心效能問題：
1. **count(\*)**：計算總頁數時每次都全表掃描，與取數據重複勞動
2. **OFFSET**：跳過的 row 也要掃描，深頁效能線性下降

---

## 1. count(\*) 的替代：EXPLAIN Row Estimate

### 1.1 問題

典型分頁流程：先 `count(*)` 算總數 → 算頁數 → 再 `SELECT ... LIMIT N OFFSET M` 取數據。這等於同一批數據掃了兩遍。

### 1.2 count_estimate() 函數

從 `EXPLAIN` 輸出中擷取 planner 的 row estimate，完全避免實際掃描數據：

```sql
CREATE FUNCTION count_estimate(query text) RETURNS INTEGER AS
$func$
DECLARE
    rec   record;
    ROWS  INTEGER;
BEGIN
    FOR rec IN EXECUTE 'EXPLAIN ' || query LOOP
        ROWS := SUBSTRING(rec."QUERY PLAN" FROM ' rows=([[:digit:]]+)');
        EXIT WHEN ROWS IS NOT NULL;
    END LOOP;
    RETURN ROWS;
END
$func$ LANGUAGE plpgsql;
```

### 1.3 精度對比

```sql
-- 估算行數（0 scan）
SELECT count_estimate(
  'SELECT * FROM sbtest1 WHERE id BETWEEN 100 AND 100000'
);
-- → 102,166

-- EXPLAIN 原始輸出
EXPLAIN SELECT * FROM sbtest1 WHERE id BETWEEN 100 AND 100000;
-- Index Scan ... rows=102166 width=190

-- 實際行數（需全掃）
SELECT count(*) FROM sbtest1 WHERE id BETWEEN 100 AND 100000;
-- → 99,901
```

估算誤差 ≈ 2.3%，精度取決於 `pg_statistic` 的 histogram 品質。

### 1.4 精度依賴：autovacuum 與統計資訊

PostgreSQL 的 autovacuum 會根據表數據變化比例自動更新統計資訊，也可配置 per-table 的統計資訊收集頻率。

> 補充（Senior Dev）：
>
> **何時 count_estimate 不可靠？**
> - 最後一次 `ANALYZE` 後有大規模 INSERT/DELETE（autovacuum 觸發閾值是 `autovacuum_analyze_scale_factor = 0.1`，即 10% 變化）
> - 查詢包含複雜的 multi-column filter（planner 假設 columns 獨立，可能大幅低估 correlation）
> - `WHERE` 中有 function call（planner 使用 default selectivity = 0.005）
> - 剛 `TRUNCATE` + 重新 `INSERT` 但尚未 ANALYZE 的表
>
> **更輕量的近似 count 方案**：
> ```sql
> -- 方案 A：pg_class.reltuples（零成本，精度 ≈ 上次 ANALYZE 時的 row count）
> SELECT reltuples::bigint FROM pg_class WHERE relname = 'your_table';
>
> -- 方案 B：pg_stat_user_tables（含 live/dead tuple 統計）
> SELECT n_live_tup FROM pg_stat_user_tables WHERE relname = 'your_table';
> ```
> `reltuples` 和 `n_live_tup` 都是近似值，不掃描表。前者是 ANALYZE 取樣推算，後者是統計資訊持續累積（含 dead tuple 修正）。`count_estimate()` 的優勢是能處理 **帶 WHERE 條件** 的估算（透過 EXPLAIN 的 selectivity 計算）。
>
> **實務建議**：
> - 前端頁碼導航的總頁數不需要精確到個位數——用 `reltuples` 或 `count_estimate()`，設定 TTL 緩存（Redis 30-60s）
> - 只有需要精確合規報表（如財務對帳）才用真實 `count(*)`
> - PostgreSQL 沒有 MySQL InnoDB 的 `COUNT(*)` 快速路徑（secondary index scan），但 PG 9.2+ 有 index-only scan 優化 `count(*)`（如果 index 的 visibility map 全部標記為 all-visible）

---

## 2. OFFSET 效能退化與 CURSOR 對比

### 2.1 問題重現

```sql
-- OFFSET 0：極快
EXPLAIN ANALYZE SELECT * FROM sbtest1
WHERE id BETWEEN 100 AND 1000000
ORDER BY id OFFSET 0 LIMIT 100;
-- actual time=0.019..0.088, rows=100

-- OFFSET 900000：退化 3700x
EXPLAIN ANALYZE SELECT * FROM sbtest1
WHERE id BETWEEN 100 AND 1000000
ORDER BY id OFFSET 900000 LIMIT 100;
-- actual time=461.941..462.009, rows=100
```

Index Scan 掃了 900,100 row 才能跳過 900,000 → 丟棄 → 再取最後 100 row。

### 2.2 CURSOR 方案

```sql
BEGIN;
DECLARE cur1 CURSOR FOR
  SELECT * FROM sbtest1 WHERE id BETWEEN 100 AND 1000000 ORDER BY id;

FETCH 100 FROM cur1;    -- 取前 100
FETCH 100 FROM cur1;    -- 取下一批，速度不退化
-- 到末尾時效能保持一致
```

CURSOR 的一次性 MOVE 成本等同 OFFSET，但後續 FETCH 成本極低（O(page_size)），而 OFFSET 每次都要付 O(offset) 成本。

> 德哥提醒：CURSOR 結果集非常龐大時可能導致長 transaction，長 transaction 有諸多負面影響。

### 2.3 三種分頁方案全面對比

| 方案 | 深頁效能 | Memory | 長 Transaction | 任意跳頁 |
|------|---------|--------|---------------|---------|
| OFFSET/LIMIT | 線性退化 | 低 | 無 | 支援 |
| CURSOR (WITHOUT HOLD) | MOVE 一次成本，後續 O(1) | 低 | **嚴重**（阻塞 VACUUM） | FETCH 方向可控 |
| Keyset Pagination（位點法） | O(1) seek | 低 | 無 | **不支援** |

> 補充（Senior Dev）：CURSOR 方案在 web 請求場景基本不可行。CURSOR 綁定在 backend connection 上，HTTP 請求之間無法保持。pgbouncer transaction pooling 會在 transaction 結束後回收 connection，CURSOR 直接失效。詳見 [PostgreSQL 分頁優化三部曲](PostgreSQL_分頁優化.md)。

---

## 3. 自建位點法：當 ORDER BY column 沒有 PK/UK 時

### 3.1 問題

當排序欄位不是 PK 或 UK 時，keyset pagination 的位點無法唯一指定——同一 `crt_time` 可能對應多條 row。

原始結構：

```sql
CREATE TABLE test (
    id INT, c1 INT, c2 INT, c3 INT, c4 INT,
    crt_time TIMESTAMP
);

CREATE INDEX idx_test_1 ON test(c1, crt_time);

INSERT INTO test
SELECT id, random()*10, random()*10, random()*10, random()*10, clock_timestamp()
FROM generate_series(1, 10000000) t(id);

-- 無法用 keyset：crt_time 非唯一
SELECT * FROM test
WHERE c1 = 1 AND c2 BETWEEN 1 AND 10
ORDER BY crt_time LIMIT 10 OFFSET 100000;
```

### 3.2 解法：強制加入自增 PK 作為 tie-breaker

```sql
CREATE TABLE test1 (
    pk SERIAL8 PRIMARY KEY,        -- 新增人工 PK
    id INT, c1 INT, c2 INT, c3 INT, c4 INT,
    crt_time TIMESTAMP
);

-- Index 需包含 PK 作為最後一個 key column
CREATE INDEX idx_test1_1 ON test1(c1, crt_time, pk);

INSERT INTO test1 (id, c1, c2, c3, c4, crt_time)
SELECT id, random()*10, random()*10, random()*10, random()*10, clock_timestamp()
FROM generate_series(1, 10000000) t(id);

-- keyset pagination：用上次最後一筆的 (crt_time, pk) 作為位點
SELECT * FROM test1
WHERE c1 = 1 AND c2 BETWEEN 1 AND 10
  AND crt_time >= '2018-07-25 18:31:14.860328'  -- 上次最大 crt_time
  AND pk > 1048412                                  -- 上次最大 pk
ORDER BY crt_time, pk LIMIT 10;
```

### 3.3 效能對比

**OFFSET 方案（OFFSET 100000）：**

```
EXPLAIN ANALYZE SELECT * FROM test
WHERE c1=1 AND c2 BETWEEN 1 AND 10
ORDER BY crt_time LIMIT 10 OFFSET 100000;

 Limit  (actual time=53.373..53.379 rows=10 loops=1)
   Buffers: shared hit=8506
   ->  Index Scan using idx_test_1 on test  (actual time=0.039..45.239 rows=100010 loops=1)
         Index Cond: (c1 = 1)
         Filter: ((c2 >= 1) AND (c2 <= 10))
         Rows Removed by Filter: 5222
         Buffers: shared hit=8506
 Execution time: 53.406 ms
```

**Keyset 方案（相同邏輯位置）：**

```
EXPLAIN ANALYZE SELECT * FROM test1
WHERE c1=1 AND c2 BETWEEN 1 AND 10
  AND crt_time >= '2018-07-25 18:31:14.860328'
  AND pk > 1048412
ORDER BY crt_time, pk LIMIT 10;

 Limit  (actual time=0.042..0.049 rows=10 loops=1)
   Buffers: shared hit=9
   ->  Index Scan using idx_test1_1 on test1  (actual time=0.040..0.045 rows=10 loops=1)
         Index Cond: ((c1 = 1) AND (crt_time >= '...') AND (pk > 1048412))
         Filter: ((c2 >= 1) AND (c2 <= 10))
         Buffers: shared hit=9
 Execution time: 0.075 ms
```

| 指標 | OFFSET | Keyset | 改善 |
|------|--------|--------|------|
| Execution time | 53.4 ms | **0.075 ms** | **712x** |
| shared_buffers hit | 8,506 | **9** | 945x |
| Row scanned | 100,010 | **~10** | 10,000x |

### 3.4 設計要點

1. `ORDER BY` 必須包含 tie-breaker（PK）以確保順序唯一
2. Index key 順序必須是 `(filter_column, order_column, pk)`，讓 keyset WHERE 條件形成連續的 index scan range
3. PK 必須是 append-only（不可 UPDATE），否則位點會漂移

> 補充（Senior Dev）：
>
> **自建 PK 的生產考量**：
> - `SERIAL` vs `IDENTITY`：PG 10+ 建議用 `GENERATED ALWAYS AS IDENTITY` 替代 `SERIAL`（標準 SQL、更清晰的權限管理）
> - 現有表遷移：`ALTER TABLE test ADD COLUMN pk BIGSERIAL PRIMARY KEY` 會觸發 full table rewrite 並鎖表。大表需用 `pg_repack` 或建立新表 + 批次遷移
> - Index key 的 trade-off：把 PK 加入 index 會讓 index 微微變大（+8 bytes per entry），但換來 keyset seek 的 O(1) 效能
> - 如果原始業務表已有 business key（如 `order_id + line_item`），優先使用 business key 作為 tie-breaker，而非人為加 PK
>
> **Filter column 的選擇性陷阱**：
> 本例中 `c2 BETWEEN 1 AND 10` 是 Filter 非 Index Cond。Filter 無法用 index 加速（只能走 index scan → filter 淘汰）。如果 `c2` 的過濾淘汰率很高，即使 keyset seek 很快，仍可能在逐 row filter 時耗費 CPU。
>
> 若 `c2` 過濾率高（如 WHERE c2 = 某特定值），考慮建 `(c1, c2, crt_time, pk)` index，讓 c2 也成為 Index Cond。

---

## 參考

1. [分頁優化 — ORDER BY LIMIT x OFFSET y performance tuning (2014)](https://github.com/digoal/blog/blob/master/201402/20140211_01.md)
2. [PostgreSQL 分頁優化三部曲](PostgreSQL_分頁優化.md)
