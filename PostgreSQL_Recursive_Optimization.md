# PostgreSQL Recursive CTE 優化 — GROUP BY / DISTINCT / Top-N Per Group 的 Index Skip Scan 技法

> 來源：
> - [digoal - 递归优化CASE (2012-09-14)](https://github.com/digoal/blog/blob/master/201209/20120914_01.md)
> - [digoal - distinct xx和count(distinct xx)的变态递归优化方法 - 索引收敛(skip scan)扫描 (2016-11-28)](https://github.com/digoal/blog/blob/master/201611/20161128_02.md)

---

## 1. 問題本質：全表 GROUP BY / DISTINCT 的掃描浪費

### 經典問題 SQL

```sql
SELECT a.skyid, a.giftpackageid, a.giftpackagename, a.intimestamp
FROM tbl_anc_player_win_log a
GROUP BY a.skyid, a.giftpackageid, a.giftpackagename, a.intimestamp
ORDER BY a.intimestamp DESC LIMIT 10;
```

這個查詢看似取 10 row，但 `GROUP BY` 會先對全表做排序/聚合（Seq Scan 100K row，Sort + Group），然後才取 Top 10。Planner 無法自動推斷只需要少量 row 就能滿足 `LIMIT 10`，因為它不知道 GROUP BY column 的 distinct 組合數。

### 同樣模式：COUNT(DISTINCT sex) on sparse column

```sql
SELECT COUNT(DISTINCT sex) FROM sex;  -- sex 只有 'm'/'w' 兩個值
```

在 2000 萬 row 的表上，即使建立 index，仍需掃完整個 index 才能做完 GROUP BY / DISTINCT —— PostgreSQL Index Only Scan 不支援「看到 unique value 就停」的 lazy aggregation。

> 補充（Senior Dev）：這是 B-tree index 的本質限制：B-tree 是 **range-scan oriented**，不支援直接跳到「下一個不同值」。Oracle 的 **Index Skip Scan** 能做這件事（index skip scan 在 B-tree 中跳過重複值直接定位下一個 distinct key），PostgreSQL 一直沒有實現 Index Skip Scan，直到 PG 17（2024）才加入了部分支援。本文介紹的 recursive CTE 技巧本質上是用 **application-level loop** 模擬了 Skip Scan。

---

## 2. 技法 1：Subquery 縮小 GROUP BY 範圍

**適用場景**：有 `ORDER BY` column index，數據分佈比較均勻。

```sql
SELECT * FROM (
    SELECT user_id, listid, apkid, get_time
    FROM user_download_log
    ORDER BY get_time DESC LIMIT 1000
) AS t
GROUP BY user_id, listid, apkid, get_time
ORDER BY get_time DESC LIMIT 10;
```

| 方法 | Plan | Runtime | 提速 |
|------|------|--------|:---:|
| 原始 | Seq Scan + Sort + Group | 97.7ms | 1x |
| Subquery 縮小 | Index Scan Backward + HashAgg | 2.0ms | **49x** |

```
 Limit → Sort → HashAggregate → Limit
   → Index Scan Backward using idx_time on user_download_log
```

弊端：內層 `LIMIT` 的值不好估算——太小取不滿 10 row，太大又浪費 scanning。

---

## 3. 技法 2：Cursor + Array（PL/pgSQL）

**適用場景**：複雜的 unique constraint（多 column 組合），不想每次估算 LIMIT。

```sql
CREATE TYPE typ_user_download_log AS (
    user_id int, listid int, apkid int, get_time timestamp(0)
);

CREATE OR REPLACE FUNCTION get_user_download_log(i_limit int)
RETURNS SETOF typ_user_download_log AS $$
DECLARE
    v_result typ_user_download_log;
    v_query refcursor;
    v_limit int := 0;
    v_result_array typ_user_download_log[];
BEGIN
    OPEN v_query FOR
        SELECT user_id, listid, apkid, get_time
        FROM user_download_log
        ORDER BY get_time DESC;
    LOOP
        FETCH v_query INTO v_result;
        IF (v_result = ANY(v_result_array)) THEN
            -- 已出現過的組合，跳過
        ELSE
            IF v_limit >= i_limit THEN EXIT; END IF;
            v_result_array := array_append(v_result_array, v_result);
            RETURN NEXT v_result;
            v_limit := v_limit + 1;
        END IF;
    END LOOP;
    CLOSE v_query;
    RETURN;
END;
$$ LANGUAGE plpgsql;

-- 查詢
SELECT * FROM get_user_download_log(10);
```

| 方法 | Runtime | 提速 |
|------|--------|:---:|
| 原始 | 97.7ms | 1x |
| Cursor + array | 0.81ms | **121x** |
| Cursor + temp table | 1.82ms | 54x |

**Cursor + array** 比 **Cursor + temp table** 快約 2x：array 在 memory 中比 temp table（需寫 disk buffer）快，且 `v_result = ANY(array)` 利用了 PG 的 row composite comparison。

> 補充（Senior Dev）：`ANY(array)` 對每條 fetch 做 O(n) 掃描。如果 distinct 組合極多（如數千），可改用 `hstore` 或 `jsonb` key 做 hash-based dedup。但對於 Top 10 場景，array 的 O(10) scan 完全可以接受。
>
> Cursor 的另一個陷阱：`refcursor` 在 function 返回後會被 close。如果需要跨呼叫保留 cursor state，要用 `DECLARE c CURSOR WITH HOLD`（holdable cursor），但注意它會 pin 住 backend process 的 memory 直到 cursor close。

---

## 4. 技法 3：Trigger 維護物化表

當 DQL >> DML 且需要絕對最快的查詢速度時，可以用 trigger 維護一個去重的物化表：

```sql
-- 物化表（PK 自動去重）
CREATE TABLE user_download_uk (
    user_id int, listid int, apkid int, get_time timestamp(0),
    PRIMARY KEY (user_id, listid, apkid, get_time)
);

CREATE INDEX idx_user_download_uk_time ON user_download_uk (get_time);

-- Trigger: INSERT → 僅在組合不存在時插入
CREATE FUNCTION tg_user_download_log_insert() RETURNS trigger AS $$
BEGIN
    PERFORM 1 FROM user_download_uk
    WHERE user_id = NEW.user_id AND listid = NEW.listid
      AND apkid = NEW.apkid AND get_time = NEW.get_time;
    IF NOT FOUND THEN
        INSERT INTO user_download_uk VALUES
            (NEW.user_id, NEW.listid, NEW.apkid, NEW.get_time);
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER tg_ins AFTER INSERT ON user_download_log
    FOR EACH ROW EXECUTE PROCEDURE tg_user_download_log_insert();
```

查詢直接命中物化表：

```sql
SELECT * FROM user_download_uk ORDER BY get_time DESC LIMIT 10;
-- 0.36ms（271x faster）
```

| 方法 | Runtime | 提速 | 代價 |
|------|--------|:---:|------|
| Trigger 物化 | 0.36ms | **271x** | Insert 從 ~4s 升到 ~4.9s（+23%），DML 越頻繁越重 |

Update / Delete 同理需要 trigger 維護。

> 補充（Senior Dev）：Trigger 方案的關鍵 trade-off：每個 DML 都多一次 PK lookup + possible insert。若 DML 量極大且 distinct 組合重複率高，`PERFORM 1` 的 PK lookup 會成為瓶頸。可考慮：
> - `INSERT ... ON CONFLICT DO NOTHING`（PG 9.5+）取代 `PERFORM + INSERT`，減少 round-trip
> - 若 Accept 稍許延遲，改用 async trigger（`pg_background` 或 LISTEN/NOTIFY 寫入 queue）
> - 若數據只 append 且不 update，直接用 materialized view + `REFRESH MATERIALIZED VIEW CONCURRENTLY`（PG 9.4+）週期性刷新，比 per-row trigger 更輕

---

## 5. 技法 4：Recursive CTE — 核心 Index Skip Scan 模擬

### 5.1 原理

Recursive CTE 模擬 Index Skip Scan 的關鍵思路：

```
1. Non-recursive term：用 index scan 取 min(column)（或 max）
2. Recursive term：用 index scan 取 「> 上一個值」的 min(column)
3. 每次遞迴只掃一個 index leaf → O(distinct_values) × O(log N)
4. Stop condition：WHERE column IS NOT NULL 阻斷 NULL（即是遞迴終止條件）
```

這樣 `COUNT(DISTINCT)` 的成本從 O(N)（全表掃描）降為 O(D × log N)（D = distinct value count）。

### 5.2 Sparse Column Scenario（低基數，如 sex / status）

```sql
CREATE TABLE sex (
    sex char(1),
    otherinfo text
);
INSERT INTO sex SELECT 'm', ... FROM generate_series(1, 10000000);
INSERT INTO sex SELECT 'w', ... FROM generate_series(1, 10000000);
-- 2000 萬 row，只有 2 個 distinct value
CREATE INDEX idx_sex ON sex(sex);
```

**原始 SQL（47 秒）：**

```sql
SELECT COUNT(DISTINCT sex) FROM sex;
-- 47254 ms
-- Plan: Index Only Scan → Aggregate（掃完全部 2000 萬 row）
```

**Recursive CTE 優化後（0.17ms，36,390x）：**

```sql
WITH RECURSIVE skip AS (
    (
        SELECT MIN(t.sex) AS sex
        FROM sex t
        WHERE t.sex IS NOT NULL
    )
    UNION ALL
    (
        SELECT (
            SELECT MIN(t.sex)
            FROM sex t
            WHERE t.sex > s.sex
              AND t.sex IS NOT NULL
        )
        FROM skip s
        WHERE s.sex IS NOT NULL      -- 關鍵：遞迴終止條件
    )
)
SELECT COUNT(DISTINCT sex) FROM skip;
-- 0.173 ms（原始 47254ms 的 1/273,000）
```

Execution plan：

```
 CTE skip
   ->  Recursive Union  (rows=2 loops=1)  ← 只遞迴 2 次！
         ->  Result  (rows=1)
               InitPlan: Limit → Index Only Scan using idx_sex
                     Index Cond: (sex IS NOT NULL)
         ->  WorkTable Scan on skip s  (rows=0 loops=2)
               Filter: (sex IS NOT NULL)
               SubPlan: Limit → Index Only Scan using idx_sex
                     Index Cond: ((sex > s.sex) AND (sex IS NOT NULL))
```

**關鍵：每次遞迴只掃 1 個 index leaf**，因為 `WHERE col > s.col` 配合 B-tree index 直接定位到下一個 distinct value。

### 5.3 Dense Column Scenario（高基數，不適合）

```sql
SELECT COUNT(DISTINCT user_id) FROM user_download_log;  -- 1000 萬 distinct
```

用同一 recursive 技巧：

```sql
WITH RECURSIVE skip AS (
    (SELECT MIN(t.user_id) AS user_id FROM user_download_log t
     WHERE t.user_id IS NOT NULL)
    UNION ALL
    (SELECT (SELECT MIN(t.user_id)
             FROM user_download_log t
             WHERE t.user_id > s.user_id AND t.user_id IS NOT NULL)
     FROM skip s WHERE s.user_id IS NOT NULL)
)
SELECT COUNT(DISTINCT user_id) FROM skip;
-- 186,909 ms（比原始 4008ms 慢 47x！）
```

遞迴了 **1000 萬次**。每次遞迴 overhead（CTE WorkTable Scan + SubPlan init）疊加後遠超全表 scan 成本。**Recursive CTE 只適合 distinct count < ~數百 的 sparse column。**

| Scenario | Distinct Values | Original | Recursive CTE | 適合？ |
|----------|:---:|---------|--------------|:---:|
| sex | 2 | 47,254ms | **0.17ms** | ✓ 極適合 |
| otherinfo | 1 | 6,296ms | **0.17ms** | ✓ |
| user_id | 10,000,001 | 4,009ms | **186,909ms** | ✗ 不適合 |

> 補充（Senior Dev）：Recursive CTE 的每一層 overhead 約 0.01-0.02ms（SubPlan init + WorkTable Scan + Filter）。因此 distinct count > ~50,000 時成本就會超過全表掃描。可以用這個 SQL 快速評估：
> ```sql
> SELECT COUNT(DISTINCT col) AS distinct_count,
>        reltuples::bigint AS total_rows
> FROM pg_stats WHERE tablename = 'your_table' AND attname = 'your_col';
> ```
> 若 distinct_count < 500 → recursive CTE 優化極有效；distinct_count > 100,000 → 不適用。

---

## 6. 技法 5：Recursive CTE 做 Top-N Per Group

這個技法是技法 4 的延伸變形，來自 [depesz](http://www.depesz.com/2012/10/05/getting-top-n-rows-per-group/)：

**原始 SQL（11.3 秒）：**

```sql
SELECT username, some_ts, random_value
FROM (
    SELECT row_number() OVER (PARTITION BY username ORDER BY some_ts DESC) AS rownum,
           *
    FROM test
) AS t
WHERE t.rownum < 6;
-- 200 萬 row，10 個 username，每個取 Top 5 → 共 50 row
-- 11,329 ms
```

**Recursive CTE 版本（1.9ms，5900x）：**

```sql
WITH RECURSIVE skip AS (
    (SELECT t.username FROM test t ORDER BY t.username LIMIT 1)
    UNION ALL
    (SELECT (
        SELECT MIN(t2.username)
        FROM test t2
        WHERE t2.username > s.username
    )
    FROM skip s
    WHERE s.username IS NOT NULL)
),
with_data AS (
    SELECT ARRAY(
        SELECT t
        FROM test t
        WHERE t.username = s.username
        ORDER BY t.some_ts DESC LIMIT 5
    ) AS rows
    FROM skip s
    WHERE s.username IS NOT NULL
)
SELECT (unnest(rows)).* FROM with_data;
-- 1.924 ms
```

這個技巧將問題拆成兩步：

| Step | CTE | 動作 | Index 使用 |
|------|-----|------|-----------|
| 1 | `skip` | 遞迴枚舉所有 distinct username | `(username, some_ts)` index 的 skip scan |
| 2 | `with_data` | 對每個 username 取 Top 5 | `(username, some_ts)` index 的 range scan（`LIMIT 5`） |

> 補充（Senior Dev）：在 PG 9.4+ 中，step 1 可以使用 `LATERAL` join 改寫為更簡單的形式（但 planner 不一定能正確優化）：
> ```sql
> SELECT DISTINCT ON (username) username, some_ts, random_value
> FROM test ORDER BY username, some_ts DESC;
> ```
> `DISTINCT ON` 只能取每 group 的第一條，無法取 Top 5。PG 14+ 的 `FETCH FIRST ... WITH TIES` 也只修飾最終結果。對於 Top-N per group 需求，recursive CTE 解法仍然有其不可替代的地位。
>
> 另一種等價寫法是用 `LATERAL` subquery：
> ```sql
> SELECT u.username, t.*
> FROM (SELECT DISTINCT username FROM test) u
> JOIN LATERAL (
>     SELECT * FROM test t
>     WHERE t.username = u.username
>     ORDER BY t.some_ts DESC LIMIT 5
> ) t ON true;
> ```
> 這種寫法可讀性更高，planner 也較容易生成對 `(username, some_ts)` 的 index 利用。在 PG 13+ 中 planner 對 LATERAL 的 optimization 已足夠成熟，推薦作為 recursive CTE 的首選替代。

---

## 7. Recursive CTE 的限制與注意事項

### Aggregate Function 不能在 Recursive Term 直接使用

```sql
-- 錯誤：aggregate function 不允許在 recursive query 的 recursive term
WITH RECURSIVE skip AS (
    (SELECT MIN(t.otherinfo) FROM user_download_log t WHERE t.otherinfo IS NOT NULL)
    UNION ALL
    (SELECT MIN(t.otherinfo) FROM user_download_log t, skip s
     WHERE t.otherinfo > s.otherinfo AND t.otherinfo IS NOT NULL
       AND s.otherinfo IS NOT NULL)   -- ERROR!
)
SELECT * FROM skip;
-- ERROR: aggregate functions not allowed in a recursive query's recursive term
```

必須將 aggregate 包在 scalar subquery 內：

```sql
(SELECT (
    SELECT MIN(t.otherinfo) FROM user_download_log t
    WHERE t.otherinfo > s.otherinfo AND t.otherinfo IS NOT NULL
)
FROM skip s WHERE s.otherinfo IS NOT NULL)
```

### 中止條件必須明確

Recursive CTE 依賴 `WHERE col IS NOT NULL` 在找不到下一個值時（subquery 返回 NULL）終止遞迴。若忘記加這個條件，會無限循環直到超出 `max_recursion_depth`（預設 200）：

```sql
-- 每個 recursive CTE 都要有終止保護
SET max_recursion_depth = 500;  -- PG 19 預設值，按實際 distinct count 調整
```

### Oracle 對比與 PostgreSQL 的設計差異

Oracle 在 sparse column count(distinct) 場景可利用 **Index Skip Scan** 或 **Bitmap Index**：

| DB | Sparse Column 方案 | Dense Column 方案 |
|----|-------------------|------------------|
| Oracle | Index Skip Scan / Bitmap Index | Index Fast Full Scan |
| PostgreSQL | Recursive CTE（本文）/ BRIN partial | Index Only Scan / HashAggregate |
| PG 17+ | Loose Index Scan（native skip scan） | 同上 |

Oracle Bitmap Index 的 fast bitwise AND/OR/COUNT 對 OLAP 極高效，但 DML 鎖粒度為整個 bitmap segment → OLTP 不適用。PG 社群曾討論引入 Bitmap Index 但最終選擇了其他方向（Bloom index、BRIN）。

---

## 8. 技法選擇決策

```
需要 GROUP BY + ORDER BY + LIMIT？
├─ Order by column 有 index + 數據均勻？
│   → 技法 1：Subquery LIMIT 縮小範圍（49x）
│
├─ 複雜多列 unique 組合 + 接受 PL/pgSQL？
│   → 技法 2：Cursor + Array（121x）
│
├─ DQL >> DML + 可接受 DML overhead？
│   → 技法 3：Trigger 物化表（271x）
│
├─ 單列 sparse column（distinct < 500）+ COUNT/GROUP？
│   → 技法 4：Recursive CTE Skip Scan（36,390x）
│
├─ Top-N per group（distinct group 少）？
│   → 技法 5：Recursive CTE + ARRAY unnest（5,900x）
│   或 LATERAL subquery（PG 13+ 推薦）
│
└─ Dense column（distinct > 100K）？
    → 接受全表 scan，或考慮 BRIN / 分區 / covering index
```
