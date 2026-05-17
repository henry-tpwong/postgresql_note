# PostgreSQL 分頁優化 — OFFSET / Cursor / Keyset / 標記列

> 來源（德哥 digoal 分頁優化三部曲）：
> - [PostgreSQL's Cursor USAGE with SQL MODE — 分頁優化 (2011-02-16)](https://github.com/digoal/blog/blob/master/201102/20110216_02.md)
> - [分頁優化 — add max_tag column speedup Query (2012-06-20)](https://github.com/digoal/blog/blob/master/201206/20120620_01.md)
> - [分頁優化 — ORDER BY LIMIT x OFFSET y performance tuning (2014-02-11)](https://github.com/digoal/blog/blob/master/201402/20140211_01.md)

---

## 1. OFFSET/LIMIT 的根本問題

### 實驗：OFFSET 0 vs OFFSET 10，執行計劃相同但效能天差地遠

```sql
-- 快速（~100ms）：OFFSET 0
SELECT * FROM tbl
WHERE create_time >= '2014-02-08' AND create_time < '2014-02-11'
  AND x = 3
  AND id != '123'
  AND id != '321'
  AND y > 0
ORDER BY create_time LIMIT 1 OFFSET 0;

-- 慢（數十秒）：OFFSET 10（只改了兩個 id 值和 OFFSET）
SELECT * FROM tbl
WHERE create_time >= '2014-02-08' AND create_time < '2014-02-11'
  AND x = 3
  AND id != '11622'
  AND id != '13042'
  AND y > 0
ORDER BY create_time LIMIT 1 OFFSET 10;
```

兩個 SQL 的 execution plan 完全一樣（都用 `Index Scan using idx on tbl` + `Filter`），但掃描 row 數相差懸殊。

### 原因分析

第二個查詢 OFFSET 10 後取到的那條 record 的 `create_time` 值是：

```
'2014-02-08 18:38:35.79'
```

這意味著為了跳過 10 條滿足所有 Filter 條件的 record，Index Scan 必須沿 `create_time` 索引從 00:00:00 一直掃描到 18:38:35.79，把中途所有 row 都讀出來用 Filter 檢查：

```sql
SELECT count(*) FROM tbl
WHERE create_time <= '2014-02-08 18:38:35.79'
  AND create_time >= '2014-02-08';
-- count: 1,448,081
-- 掃描了 144 萬條 record 才找到 OFFSET 10 需要的 10 條！
```

根本矛盾：**WHERE 條件中的 `id !=` 和 `x = 3` 等 Filter 無法被 `create_time` 索引加速**，Index Scan 只能在 `create_time` 維度快速定位，但 Filter 淘汰率高 → OFFSET 越大，需要掃描的 row 越多。

> 補充（Senior Dev）：這是所有 SQL 資料庫的共同問題，不是 PostgreSQL 特有。`OFFSET N` 的複雜度是 O(N) 而非 O(1)。在實務中，當 Filter 條件多且選擇性低時，即使 N=10 也可能極慢。EXPLAIN 的 `rows` estimate 只是統計估算的「符合所有條件的總行數」（本例 4710），並不能反映「要掃描多少行才能找到 N 條」的實際成本。要準確估算，需要用 `WHERE create_time <= <last_fetched_time>` 的方式手算。

---

## 2. 解法一：Keyset Pagination（Seek Method）

德哥的建議（不需要新增 index）：

每次記錄最後一筆資料的 `create_time` 最大值和對應的 PK，下一頁查詢時加上這兩個條件，**完全不用 OFFSET**：

```sql
-- 第一頁
SELECT * FROM tbl
WHERE create_time >= '2014-02-08' AND create_time < '2014-02-11'
  AND x = 3
  AND id != '11622' AND id != '13042'
  AND y > 0
ORDER BY create_time, pk LIMIT 10 OFFSET 0;
-- 假設最後一筆: create_time = '2014-02-08 18:38:35.79', pk = 56789

-- 第二頁（不依賴 OFFSET）
SELECT * FROM tbl
WHERE create_time >= '2014-02-08' AND create_time < '2014-02-11'
  AND x = 3
  AND id != '11622' AND id != '13042'
  AND y > 0
  AND pk NOT IN (?)  -- 防止與上一頁最後一筆 time 相同時重複
  AND create_time >= '2014-02-08 18:38:35.79'
ORDER BY create_time LIMIT ? OFFSET 0;
```

**核心思想**：把「跳過前 N 條」轉化為 `WHERE create_time > last_fetched_time`，讓 Index Scan 直接 seek 到上次結束的位置。

> 補充（Senior Dev）：這是業界稱為 **Keyset Pagination** 或 **Cursor-based Pagination** 的標準做法（如 GraphQL Relay 的 `after`/`before` cursor）。
>
> **實務注意事項**：
> 1. `ORDER BY` 的 column 必須形成 unique key。只用 `create_time` 不夠（可能有相同 time 的多條 record），必須加上 PK 作為 tie-breaker：`ORDER BY create_time, pk`。否則同一時間點的 record 可能在兩頁間重複或遺漏。
> 2. WHERE 條件必須使用 `>=` + `PK NOT IN (...)` 而非 `>`。若 `create_time` 有重複值，單用 `>` 會跳過同一 time 內的後續 record。
> 3. 雙向分頁（上一頁）的實現比下一頁複雜，因為需要 reverse ORDER BY 和 WHERE 方向。
> 4. **不支援任意跳頁**（「跳到第 10 頁」）。這是 keyset pagination 的先天限制——若需要任意跳頁（如論壇的頁碼導航），可考慮：
>    - 讓 client 緩存前幾頁的 cursor 位置
>    - 接受前幾頁使用 OFFSET（offset 小時成本可接受），深頁使用 keyset
>    - 估算總頁數（`count(*)` 結果緩存），只渲染鄰近頁碼

---

## 3. 解法二：消除昂貴子查詢 — ismax 標記列

### 原始問題（2012 場景）

```sql
SELECT t.APP_ID, t.APP_VER, t.CN_NAME, ...
FROM
  (SELECT APP_ID, max(APP_VER) APP_VER
   FROM test1 GROUP BY APP_ID) s
JOIN test1 t
  ON s.APP_ID = t.APP_ID AND s.APP_VER = t.APP_VER AND t.DELETED = 0
LEFT OUTER JOIN test2 at ON t.APP_ID = at.APP_ID
LEFT OUTER JOIN test3 h  ON t.APP_ID = h.APP_ID
LIMIT 24 OFFSET 0;
```

Execution plan (`OFFSET 0` → fast, `OFFSET 100000` → 1,060ms)：

```
-- OFFSET 0: total 0.565 ms
GroupAggregate → Index Scan using idx_test1_2 (62 rows → 25 groups)
Merge Join → Index Scan on t (60 rows)
Merge Left Join → Index Scan on at (6 rows)
Nested Loop → Index Scan on h (1 row × 24 loops)

-- OFFSET 100000: total 1,060.683 ms — 注意 loops=92075
GroupAggregate → Index Scan using idx_test1_2 (110,646 rows → 92,796 groups)
Merge Join → Index Scan on t (109,855 rows)
Merge Left Join → Index Scan on at (476 rows)
Nested Loop → Index Scan on h (1 row × 92,075 loops)  -- 91K 次回表！
```

瓶頸是本應 LIMIT 24 才需要的 row，但因為 OFFSET 100000 導致 planner 被迫全量 JOIN 再丟棄 100,000 條。

### 優化 1：用 ismax 標記列消除子查詢的 GROUP BY

```sql
ALTER TABLE test1 ADD COLUMN ismax BOOLEAN;

UPDATE test1 SET ismax = TRUE
WHERE (app_id, app_ver) IN (
    SELECT app_id, max(app_ver) FROM test1 GROUP BY app_id
);

-- Partial Index：只索引 ismax = TRUE 的 row，且對應 deleted = 0
CREATE INDEX idx_test1_1 ON test1(app_id) WHERE ismax IS TRUE AND deleted = 0;
```

重寫查詢，消除子查詢：

```sql
SELECT t.APP_ID, t.APP_VER, t.CN_NAME, ...
FROM test1 t
LEFT OUTER JOIN test2 at
  ON (t.APP_ID = at.APP_ID AND t.DELETED = 0 AND t.ismax IS TRUE)
LEFT OUTER JOIN test3 h
  ON (t.APP_ID = h.APP_ID)
LIMIT 24 OFFSET 0;
```

Execution plan 對比（OFFSET 100000 的最差情況）：

| 指標 | 原 SQL | ismax 優化 | 改善 |
|------|--------|-----------|------|
| Total runtime | 1,060 ms | 584 ms | ~1.8× |
| Nested Loop loops | 92,075 | 110,646 | 仍舊 linear scan |

性能大約 1 倍提升，但到深頁仍然隨 OFFSET 變慢。

### 優化 2：加大每頁數量 + 應用層緩存

實驗：用 function 模擬完整翻完所有頁面所需時間：

```sql
CREATE OR REPLACE FUNCTION f_test1(i_limit INT) RETURNS INT AS $$
DECLARE
  v_work INT; v_count INT := 0; v_offset INT := 0; i INT := 1;
BEGIN
  RAISE NOTICE 'start time: %', clock_timestamp();
  LOOP
    SELECT count(*) INTO v_work FROM (
      SELECT 1 FROM test1 t
      LEFT OUTER JOIN test2 at
        ON (t.APP_ID = at.APP_ID AND t.DELETED = 0 AND t.ismax IS TRUE)
      LEFT OUTER JOIN test3 h ON (t.APP_ID = h.APP_ID)
      LIMIT i_limit OFFSET v_offset
    ) t;
    IF v_work = 0 THEN EXIT; END IF;
    v_offset := i * i_limit;
    v_count := v_count + v_work;
    i := i + 1;
  END LOOP;
  RAISE NOTICE 'end time: %', clock_timestamp();
  RETURN v_count;
END;
$$ LANGUAGE plpgsql;
```

```sql
SELECT * FROM f_test1(10000);  -- 每頁 10,000 條 → 2 秒翻完
SELECT * FROM f_test1(100);    -- 每頁 100 條    → 128 秒翻完
```

結論：**每頁取的數量越多，總翻頁時間越短**。理想情況下一次性取全部數據最快。

> 德哥指出：原 SQL 沒有 `ORDER BY`，無法保證返回順序，這是業務層 BUG。

> 補充（Senior Dev）：
>
> **ismax 方案的生產維護考量**：
> - 新增 `app_ver` 時需同時更新 ismax：新 row 的 ismax = TRUE，同一 `app_id` 的舊 row 需要設為 FALSE。可以用 trigger `BEFORE INSERT ... UPDATE test1 SET ismax = FALSE WHERE app_id = NEW.app_id AND app_ver < NEW.app_ver`。
> - **並發問題**：兩個 transaction 同時插入同一 `app_id` 的不同 `app_ver`，各自都可能認為自己的 row 是 max。解法：
>   1. 使用 `SERIALIZABLE` isolation level 觸發 serialization failure 重試
>   2. 使用 advisory lock on app_id
>   3. 定期用 `UPDATE ... FROM (SELECT app_id, max(app_ver) ...)` 修正（cron job）
> - **Partial Index 的維護**：`WHERE ismax IS TRUE AND deleted = 0` 條件變更時（如 `deleted` 從 0 變非 0），row 會從 partial index 中被移除，VACUUM 會清理 dead tuple。但前提是 VACUUM 能及時觸發——大量 UPDATE ismax/deleted 後應手動 VACUUM。
> - **現代替代方案**：PostgreSQL 的 `DISTINCT ON` + `ORDER BY app_id, app_ver DESC` 可以取代 GROUP BY 的 `max(app_ver)`，在某些場景比 ismax 標記列更簡潔（不需要 trigger），但無法利用 partial index 過濾。
>
> **OFFSET 與 cursor 的數學本質**：
> - OFFSET 的總翻頁成本是 O(n²)：每翻一頁都要從頭掃描到上次的位置
> - CURSOR 的總翻頁成本是 O(n)：MOVE 一次成本 = OFFSET 一次成本，但後續 FETCH 成本極低（O(page_size)）
> - keyset pagination 的總翻頁成本也是 O(n)：每次 seek 直接定位，不需要從頭掃描

---

## 4. 解法三：CURSOR（DB Cursor）

### 三種大批量數據返回方式對比

環境：PostgreSQL 9.0.2，表大小 588,240 KB（約 10M row）。

**方式一：SELECT \*（全部載入 client memory）**

```sql
SELECT * FROM tbl_user;
```

| 階段 | Backend memory 佔用 |
|------|-------------------|
| 連線 idle | 3,512 KB |
| 執行中 | 409,764 KB → 721,852 KB |
| 結果傳完 idle | 721,852 KB |

所有數據載入 backend process memory，適合小結果集。

**方式二：CURSOR WITH HOLD（session-level cursor）**

```sql
BEGIN;
DECLARE cur_test SCROLL CURSOR WITH HOLD FOR SELECT * FROM tbl_user;
FETCH 10 FROM cur_test;
```

| 操作 | Memory 佔用 |
|------|------------|
| 連線 idle | 3,508 KB |
| DECLARE CURSOR 後 | 709,332 KB |
| FETCH 1 | ~5,128 KB |
| FETCH LAST | 703,972 KB |

關鍵觀察：DECLARE CURSOR 時數據已全部 materialize（載入 memory 或 temporary file），memory 佔用接近全表大小。後續 FETCH 操作幾乎零成本（0.2-0.3ms）。

> 補充（Senior Dev）：`WITH HOLD` cursor 的 materialize 行為：當結果集超過 `work_mem` 時，數據寫入 temporary file（`$PGDATA/base/pgsql_tmp/`）。這解釋了為什麼 memory 佔用可能低於表大小——超出 work_mem 的部分在 disk 上。但 `WITH HOLD` cursor 的實務問題是：
> 1. 持有 cursor 的 transaction 即使 idle，也無法被 pgbouncer transaction pooling 復用（因為 cursor 狀態跨越 transaction boundary）
> 2. 多個 concurrent cursor 可能耗盡 temporary file space
> 3. 3rd-party connection pooler 多數不支援 cursor（如 HikariCP、node-postgres pool）

**Cursor 的 INSENSITIVE 語義**：定義後的 DML 對 FETCH 不可見。即使其他 session DELETE 了所有數據，cursor 仍能 FETCH 到定義時的 snapshot。

```sql
-- SESSION A: 定義 cursor WITH HOLD
DECLARE cur_test SCROLL CURSOR WITH HOLD FOR SELECT * FROM tbl_user;

-- SESSION B: DELETE 全部
DELETE FROM tbl_user;

-- SESSION A: 仍然可 FETCH
FETCH FIRST FROM cur_test;  -- 返回原始數據
FETCH 20 FROM cur_test;     -- 9 rows（與定義時一致）
```

**CURSOR 生命週期**：
- `WITHOUT HOLD`（預設）：transaction 結束時自動釋放
- `WITH HOLD`：session 結束時釋放，或定義 cursor 的 transaction 被 ABORT 時釋放

```sql
BEGIN;
DECLARE cur_test SCROLL CURSOR WITH HOLD FOR SELECT * FROM tbl_user;
ABORT;  -- cursor 被釋放
FETCH LAST FROM cur_test;
-- ERROR: cursor "cur_test" does not exist
```

**方式三：OFFSET/LIMIT**

```sql
SELECT * FROM tbl_user ORDER BY id OFFSET 0 LIMIT 1;       -- 1.354 ms,  5,256 KB
SELECT * FROM tbl_user ORDER BY id OFFSET 99999 LIMIT 1;    -- 15.248 ms, 832,020 KB
SELECT * FROM tbl_user ORDER BY id OFFSET 9999999 LIMIT 1;  -- 1,679 ms, 832,020 KB
```

OFFSET 越大越慢，因為必須從頭掃描到 OFFSET 位置。Memory 佔用取決於 scanned data 量（Index Scan 走 `id` index，page 載入 shared_buffers）。

### CURSOR MOVE vs OFFSET 成本對比

```sql
-- CURSOR: 一次性 MOVE 成本，後續 FETCH 幾無成本
BEGIN;
DECLARE cur_test NO SCROLL CURSOR WITHOUT HOLD FOR
  SELECT * FROM tbl_user ORDER BY id;
MOVE 100000 FROM cur_test;   -- 18.638 ms（一次性）
FETCH 10 FROM cur_test;      -- 0.317 ms
FETCH 10 FROM cur_test;      -- 0.223 ms

-- OFFSET: 每次都要付成本
SELECT * FROM tbl_user ORDER BY id OFFSET 100000 LIMIT 10;  -- 15.432 ms
SELECT * FROM tbl_user ORDER BY id OFFSET 100010 LIMIT 10;  -- 15.170 ms
```

> CURSOR 的總成本是 O(n)，OFFSET 翻頁的總成本是 O(n²)。

### CURSOR 使用注意事項（德哥原文）

1. CURSOR 用完一定要 `CLOSE` 關掉
2. 盡量避免在 large result set 下使用 `WITH HOLD` cursor——需要將數據預載入 memory 或 temporary file，concurrent 高 + result set 大時可能 memory 耗盡
3. 盡量避免長 transaction（如等待用戶翻頁）。**`WITHOUT HOLD` cursor 的 transaction 會影響 VACUUM 回收空間**——這是非常嚴重的問題

> 補充（Senior Dev）：第 3 點是生產中 CURSOR 翻頁方案的最大殺手。一個用戶打開了 cursor 然後去吃飯，這個 backend 的 transaction 就一直 open → `xmin` horizon 無法推進 → VACUUM 無法回收該 transaction 開始後產生的 dead tuple → table bloat。解決方案：
> - `idle_in_transaction_session_timeout`（PG 9.6+）：強制殺掉停在 idle in transaction 的 connection
> - 如果必須用 CURSOR，使用 `WITH HOLD` + 在 FETCH 之間 `COMMIT`（但 `WITH HOLD` 本身消耗 memory/temp file）
> - **Web 場景不要用 DB CURSOR**：HTTP request 之間無法保持 DB connection/cursor 狀態。應用層實現 keyset pagination 才是正解

### 查詢 CURSOR 狀態

```sql
SELECT * FROM pg_cursors;
-- name | statement | is_holdable | is_binary | is_scrollable | creation_time
```

---

## 5. 四種分頁方案全面對比

| 方案 | 總翻頁成本 | Memory | VACUUM 影響 | 任意跳頁 | 適用場景 |
|------|----------|--------|------------|---------|---------|
| OFFSET/LIMIT | O(n²) | 低 | 無 | 支援 | 極小 offset（<100）/ 總行數少 |
| Keyset Pagination | O(n) | 低 | 無 | 不支援 | Web API（無限滾動 / 下一頁）/ app 列表 |
| CURSOR WITHOUT HOLD | O(n) | 低（遞增） | **嚴重**（阻塞 VACUUM） | 支援（SCROLL） | 內部批次處理 / ETL |
| CURSOR WITH HOLD | O(n) | **高**（全量 materialize） | 輕微（commit 後釋放） | 支援（SCROLL） | 需跨 transaction 的內部工具 |
| 加大 page size + 應用層緩存 | O(n)（減少 DB round-trip） | 應用層 | 無 | 客戶端模擬 | 總行數可控的列表 |

> 補充（Senior Dev）：
>
> **Web 場景最終結論**：
> 1. **第一選擇**：Keyset pagination（`WHERE id > last_id ORDER BY id LIMIT N`）
> 2. 若需要頁碼導航：小於 ~100 頁用 OFFSET，深頁轉 keyset 或提供搜尋/篩選縮小範圍
> 3. 若需要總頁數顯示，`count(*)` 結果應緩存（如 Redis，TTL 30-60s），不要每頁都 count
> 4. **絕對不要**在 web request handler 中使用 DB CURSOR——HTTP 是 stateless，無法跨 request 保持 cursor
>
> **Keyset pagination 的 SQL 模板（PostgreSQL）**：
> ```sql
> -- Forward (下一頁)
> SELECT * FROM t
> WHERE (order_col, pk_col) > (:last_order_val, :last_pk_val)
> ORDER BY order_col, pk_col
> LIMIT :page_size;
>
> -- Backward (上一頁)
> SELECT * FROM (
>   SELECT * FROM t
>   WHERE (order_col, pk_col) < (:first_order_val, :first_pk_val)
>   ORDER BY order_col DESC, pk_col DESC
>   LIMIT :page_size
> ) sub
> ORDER BY order_col, pk_col;
> ```
> PostgreSQL 的 row value comparison `(a, b) > (x, y)` 讓 keyset WHERE 變得非常簡潔，這是 PG 相對於 MySQL 的優勢（MySQL 5.7 前不支援 row value 比較）。
