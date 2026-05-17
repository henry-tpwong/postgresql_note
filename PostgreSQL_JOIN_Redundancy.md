# PostgreSQL JOIN 冗餘膨脹導致慢查詢優化 — Early DISTINCT 策略

> 來源：[digoal - 冗餘數據 JOIN 導致的慢 SQL 優化一例 (2016-08-17)](https://github.com/digoal/blog/blob/master/201608/20160817_03.md)

---

## 1. 問題：每表僅數千 row，4 表 JOIN 卻數十秒無結果

```sql
SELECT DISTINCT abc.pro_col1, abc.col3
FROM t0 p
INNER JOIN t1 abc ON p.id = abc.par_col2
INNER JOIN t2 s   ON s.col3 = abc.col3
INNER JOIN t3 po  ON po.id = s.col4
WHERE p.state = 2 AND po.state = 3
ORDER BY abc.pro_col1, abc.col3;
```

### 根因分析

語意上是一條取兩個欄位 distinct 值的簡單查詢。但執行時每一次 JOIN 都因為 JOIN key 的 **cardinality 過低**而產生大量 intermediate row：

```
JOIN 1: t0(p) ⋈ t1(abc) → 可能 1×N 膨脹（單個 p.id 對應多個 abc.par_col2）
JOIN 2: result ⋈ t2(s)   → 再次膨脹（abc.col3 可能對應多個 s.col3）
JOIN 3: result ⋈ t3(po)  → 再次膨脹
```

膨脹效果是乘數關係：若每層 JOIN 膨脹 10×，4 層就是 10³× = 1000×，最後才用 DISTINCT 收斂到幾條結果。中間耗費的 memory / CPU 全屬浪費。

---

## 2. 解法：Early DISTINCT — 每層 JOIN 後立即去重

```sql
SELECT DISTINCT pro_col1, col3
FROM (
    SELECT DISTINCT t1.pro_col1, t1.col3, s.col4
    FROM (
        SELECT DISTINCT abc.pro_col1, abc.col3
        FROM t1 abc
        INNER JOIN t0 p ON (p.id = abc.par_col2 AND p.state = 2)
    ) t1
    INNER JOIN t2 s ON (s.col3 = t1.col3)
) t2
INNER JOIN t3 po ON (po.id = t2.col4 AND po.state = 3)
ORDER BY t2.pro_col1, t2.col3;
```

**核心邏輯**：在每一層 JOIN 之後立刻用子查詢 + `DISTINCT` 把 intermediate result set 降到最小，阻止膨脹效應傳播到下一層 JOIN。

修改後從數十秒降至數十毫秒。

> 補充（Senior Dev）：這與 SQL 執行計劃中的「predicate pushdown」邏輯相同，只是這裡推的是 DISTINCT 而非 WHERE。PostgreSQL 的 planner 不會自動推 DISTINCT 進子查詢（因為 DISTINCT 與 JOIN 不滿足交換律：先去重再 JOIN vs 先 JOIN 再去重，結果可能不同）。只有在 query 的語意等價時才能手動改寫。判斷何時安全：
> 1. DISTINCT 的 column 集合等於或包含後續 JOIN 的 key
> 2. DISTINCT 不改變後續 JOIN 的 cardinality（即去重後 key 值不丟失）

---

## 3. 重現實驗

```sql
CREATE TABLE rt1 (id INT, info TEXT);
CREATE TABLE rt2 (id INT, info TEXT);
CREATE TABLE rt3 (id INT, info TEXT);
CREATE TABLE rt4 (id INT, info TEXT);

-- rt1: 1000 row (id = 1~1000)
INSERT INTO rt1 SELECT generate_series(1, 1000), 'test';

-- rt2/rt3/rt4: 每個 id=1 有 1000 行（全是 id=1）
INSERT INTO rt2 SELECT 1, 'test' FROM generate_series(1, 1000);
INSERT INTO rt3 SELECT 1, 'test' FROM generate_series(1, 1000);
INSERT INTO rt4 SELECT 1, 'test' FROM generate_series(1, 1000);
```

### 原寫法（不斷膨脹）

```sql
EXPLAIN SELECT DISTINCT rt1.id
FROM rt1 JOIN rt2 ON rt1.id = rt2.id
         JOIN rt3 ON rt2.id = rt3.id
         JOIN rt4 ON rt3.id = rt4.id;
```

```
HashAggregate (rows=1000)
  -> Hash Join (rows=1000)
       Hash Cond: (rt4.id = rt1.id)
       -> Seq Scan on rt4 (rows=1000)
       -> Hash (rows=1000)
            -> Hash Join (rows=1000)
                 Hash Cond: (rt3.id = rt1.id)
                 -> Seq Scan on rt3 (rows=1000)
                 -> Hash (rows=1000)
                      -> Hash Join (rows=1000)
                           Hash Cond: (rt2.id = rt1.id)
                           -> Seq Scan on rt2 (rows=1000)
                           -> Hash (rows=1000)
                                -> Seq Scan on rt1 (rows=1000)
```

關鍵觀察：每層 Hash Join 的預估 row 數都是 1000，但因為 rt2/rt3/rt4 的 id 全是 1，實際 JOIN 產生的是 **笛卡爾乘積**。最終 intermediate result 遠超 1000 row，但 DISTINCT 壓回 1 row。

### Early DISTINCT 改寫

```sql
EXPLAIN SELECT DISTINCT t2.id
FROM (
    SELECT DISTINCT t1.id
    FROM (
        SELECT DISTINCT rt1.id FROM rt1 JOIN rt2 ON rt1.id = rt2.id
    ) t1
    JOIN rt3 ON t1.id = rt3.id
) t2
JOIN rt4 ON t2.id = rt4.id;
```

```
HashAggregate (rows=1000)
  Group Key: rt1.id
  -> Hash Join (rows=1000)
       Hash Cond: (rt4.id = rt1.id)
       -> Seq Scan on rt4 (rows=1000)
       -> Hash (rows=1000)
            -> HashAggregate (rows=1000)       ← Early DISTINCT
                 Group Key: rt1.id
                 -> Hash Join (rows=1000)
                      Hash Cond: (rt3.id = rt1.id)
                      -> Seq Scan on rt3 (rows=1000)
                      -> Hash (rows=1000)
                           -> HashAggregate (rows=1000)  ← Early DISTINCT
                                Group Key: rt1.id
                                -> Hash Join (rows=1000)
                                     Hash Cond: (rt2.id = rt1.id)
                                     -> Seq Scan on rt2 (rows=1000)
                                     -> Hash (rows=1000)
                                          -> Seq Scan on rt1 (rows=1000)
```

每層 JOIN 後都插入 `HashAggregate (Group Key: rt1.id)`，確保向下傳遞的 row 數不膨脹。

```sql
-- 實際執行：2.052 ms，結果 1 row
SELECT DISTINCT t2.id FROM (
    SELECT DISTINCT t1.id FROM (
        SELECT DISTINCT rt1.id FROM rt1 JOIN rt2 ON rt1.id = rt2.id
    ) t1
    JOIN rt3 ON t1.id = rt3.id
) t2
JOIN rt4 ON t2.id = rt4.id;
-- id
-- ----
--   1
-- (1 row)
```

> 補充（Senior Dev）：
>
> **何時應該用 Early DISTINCT vs 其他方案？**
>
> | 場景 | 方案 |
> |------|------|
> | JOIN key cardinality 極低（如 status='active' 佔 80% 表） | Early DISTINCT（本文） |
> | 只需要 EXISTS 而不需要 JOIN 的 column | 改用 `WHERE EXISTS (SELECT 1 FROM t2 WHERE ...)`，徹底避免 row 膨脹 |
> | 外層 DISTINCT + ORDER BY | 考慮複合 index 讓 planner 走 Index Scan 跳過 redundant row（Index Only Scan + DISTINCT 可能直接用 index skip scan，PG 17+ `enable_indexskipscan`） |
> | 子查詢結果需要傳遞 column 到外層 | 本文的 subquery + DISTINCT |
> | PG 12+ | CTE `MATERIALIZED` 可作為 explicit optimization fence，但 planner 不會自動推 DISTINCT |
>
> **生產中如何發現此問題**：
> ```sql
> EXPLAIN (ANALYZE, BUFFERS) <your_query>;
> ```
> 觀察 `actual rows` vs `estimated rows` 的偏差。若某個 Hash Join / Nested Loop 的 `actual rows` 遠大於 `rows` estimate（如估 1000 但實際 10,000,000），就是膨脹信號。進一步檢查該 JOIN key 的 `n_distinct`（`pg_stats.n_distinct`）。
>
> **內核層改進（德哥建議）**：利用 `pg_stats.n_distinct` 統計資訊，planner 可以在 JOIN key cardinality 極低時自動推入 Early DISTINCT rewrite。目前 PostgreSQL 尚未實現，但這是 query optimizer 的合理改進方向（類似於 MySQL 8.0 的 `derived_condition_pushdown` 或 Oracle 的 cost-based query transformation）。
>
> **簡化寫法（PG 9.4+ 可用 `WITH` 結構化）**：
> ```sql
> WITH
> j1 AS (SELECT DISTINCT abc.pro_col1, abc.col3
>        FROM t1 abc JOIN t0 p ON p.id = abc.par_col2 AND p.state = 2),
> j2 AS (SELECT DISTINCT j1.pro_col1, j1.col3, s.col4
>        FROM j1 JOIN t2 s ON s.col3 = j1.col3)
> SELECT DISTINCT j2.pro_col1, j2.col3
> FROM j2 JOIN t3 po ON po.id = j2.col4 AND po.state = 3
> ORDER BY j2.pro_col1, j2.col3;
> ```
> CTE 版本與 subquery 版本在 planner 行為上等價（PG 12 之前 CTE 預設 materialized fence，12+ 預設 `NOT MATERIALIZED` 讓 planner 可推入 CTE 內），但可讀性更好。
